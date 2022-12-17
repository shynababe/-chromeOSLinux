![Screenshot_2022-11-30_03-07-32](https://user-images.githubusercontent.com/120763310/208248341-17c89809-ce25-4818-9898-d5b08ed5a378.png)
# -chromeOSLinux
my attempt at RPM chromebook install with Cras and Xorg with new kernels

### Append to prepare.sh:

# Fetch CRAS source

CRASBUILDTMP="`mktemp -d chromeOSLinux-cras.XXXXXX --tmpdir=/usr/local/tmp`" # need to create dir moved from /tmp

addtrap "rm -rf --one-file-system '$CRASBUILDTMP'"

# ADHD_HEAD is normally autodetected, but it can be set manually using a custom
# target sourced before this one (used for automated testing)
# The custom target also needs to set CROS_VER_1
if [ -z "$ADHD_HEAD" ]; then
    # Chrome OS version (e.g. 4100.86.0)
    CROS_VER="`sed -n 's/^CHROMEOS_RELEASE_VERSION=//p' /var/host/lsb-release`"

    # Set CROS_VER_1 to the major version number (e.g. 4100), used later on to
    # selectively apply patches.
    CROS_VER_1="${CROS_VER%%.*}"

    cras_version="`cat /var/host/cras-version 2>/dev/null || true`"
    ADHD_HEAD="${cras_version##*-}"
    # Make sure ADHD_HEAD looks like a commit id
    if [ "${#ADHD_HEAD}" -ne 40 -o "${head%[^0-9a-f]*}" != "$head" ]; then
        echo "Empty or invalid cras-version (${cras_version})." 1>&2
        ADHD_HEAD=""
    fi
fi

echo "Fetching CRAS (branch $ADHD_HEAD)..." 1>&2

# Try to download the CRAS commit id first, and fall back on master if that is
# not found (see crbug.com/417820).

archive="$CRASBUILDTMP/adhd.tar.gz"
log="$CRASBUILDTMP/wget.log"
urlbase="https://chromium.googlesource.com/chromiumos/third_party/adhd/+archive"
( wget -O "$archive" "$urlbase/$ADHD_HEAD.tar.gz" 2>&1 \
                                    || echo "Error fetching CRAS" ) | tee "$log"
if tail -n 1 "$log" | grep -q "^Error"; then
    if grep -q "404 Not Found" "$log" 2>/dev/null; then
        echo "Branch not found, falling back on master (audio might not work)..." 1>&2
        ADHD_HEAD="master"
        wget -O "$archive" "$urlbase/$ADHD_HEAD.tar.gz"
    else
        # wget already printed an explicit error
        exit 1
    fi
fi

# Build CRAS ALSA plugin for the given architecture ($1)
# A blank parameter means we are building for the native architecture.
build_cras() {
    local cras_arch="$1"
    local pkgsuffix=''
    local pkgdepextra=''
    local archextrapath=''
    local pkgconfigpath=''
    local archgccflags=''
    if [ -n "$cras_arch" ]; then
        pkgsuffix=":$cras_arch"
        pkgdepextra='gcc-multilib'
        archextrapath="/$cras_arch-linux-gnu"
        pkgconfigpath="/usr/lib$archextrapath/pkgconfig"
        archgccflags='-m32'

        # Add foreign architecture, if necessary
        if ! dpkg --print-foreign-architectures | grep -q "^$cras_arch$"; then
            echo "Adding foreign architecture $cras_arch to dpkg..." 1>&2
            dpkg --add-architecture "$cras_arch"
            apt-get update || true
        fi
    fi

    # Install CRAS dependencies
    install --minimal alsa-utils \
        libasound2$pkgsuffix \
        libasound2-plugins$pkgsuffix \
        libspeexdsp1$pkgsuffix

    install --minimal --asdeps gcc $pkgdepextra libc6-dev$pkgsuffix \
        pkg-config libspeexdsp-dev$pkgsuffix

    # precise does not allow libasound2-dev and libasound2-dev:i386 to be
    # installed simultaneously
    if release -le precise && [ -n "$cras_arch" ]; then
        install --minimal --asdeps libasound2-dev
        # Manually link .so file
        libasoundso="/usr/lib$archextrapath/libasound.so"
        if [ ! -f "$libasoundso" ]; then
            addtrap "rm -f '$libasoundso' 2>/dev/null"
            ln -sfT libasound.so.2 "$libasoundso"
        fi
        ALSALIBDIR="/usr/lib$archextrapath/alsa-lib"
    else
        install --minimal --asdeps libasound2-dev$pkgsuffix
        ALSALIBDIR="`PKG_CONFIG_PATH="$pkgconfigpath" \
                                pkg-config --variable=libdir alsa`/alsa-lib"
    fi

    # Start subshell for compilation
    (
        cd "$CRASBUILDTMP"

        # Make sure we start fresh
        rm -rf --one-file-system cras

        # -m prevents "time stamp is in the future" spam
        tar -xmf adhd.tar.gz cras/src

        cd cras/src

        # Create version file
        echo '#define VCSID "chromeOSLinux-'"$ADHD_HEAD"'"' > common/cras_version.h

        install --minimal --asdeps patch

        # Remove SBC dependency
        sed -e 's/#include <sbc.*//' -i common/cras_sbc_codec.h
        cat > common/cras_sbc_codec.c <<END
#include <stdint.h>
#include <stdlib.h>
#include "cras_audio_codec.h"
struct cras_audio_codec *cras_sbc_codec_create(uint8_t freq,
		   uint8_t mode, uint8_t subbands, uint8_t alloc,
		   uint8_t blocks, uint8_t bitpool) {
    abort();
}
void cras_sbc_codec_destroy(struct cras_audio_codec *codec) {
    abort();
}
END

        # Find the cras test client
        cras_test_client='tools/cras_test_client/cras_test_client.c'
        # FIXME: remove when 12334.0.0 (R77) becomes minimum target
        if [ ! -f "$cras_test_client" ]; then
            cras_test_client='tests/cras_test_client.c'
        fi

        # Drop SBC constants
        sed -e 's/SBC_[A-Z0-9_]*/0/g' -i "$cras_test_client"

	# FIXME(crbug.com/864815) Remove when fixed upstream
	if [ "$cras_arch" = "i386" ]; then
	    sed -i -e 's/Dumping AEC info to %s, stream %lu/Dumping AEC info to %s, stream %llu/' \
		"$cras_test_client"
	fi
	sed -i -e 's/cras_stream_id_t stream_id;/cras_stream_id_t stream_id = 0;/' \
	    "$cras_test_client"

	# FIXME(http://crrev.com/c/1759370): Remove when R78 becomes minimum target
	sed -i -e 's/\(strncpy.*sizeof(info.name)\));/\1 - 1); info.name[sizeof(info.name) - 1] = '"'"'\\0'"'"';/' \
		common/cras_shm.c
	sed -i -e 's/^struct \([^_]\)/struct __attribute__((__packed__)) \1/' \
		common/cras_messages.h

        # Directory to install CRAS library/binaries
        CRASLIBDIR="/usr/local$archextrapath/lib"
        CRASBINDIR="/usr/local$archextrapath/bin"

        echo "Compiling CRAS (${cras_arch:-native})..." 1>&2
        # Convert Makefile.am to a shell script, and run it.
        {
            convert_automake

            echo '
                CFLAGS="$CFLAGS -DCRAS_SOCKET_FILE_DIR=\"/var/run/cras\" -Wno-error"
                buildlib libcras
                # Pass -rpath=$CRASLIBDIR to linker, so we do not need to add
                # the directory to ldconfig search path (some distributions do
                # not include /usr/local/lib in /etc/ld.so.conf).
                # We also need to add "-L." as we are not using .la files.
                extraflags="-Wl,-rpath='"$CRASLIBDIR"' -L."
                buildlib libasound_module_pcm_cras "$extraflags"
                buildlib libasound_module_ctl_cras "$extraflags"
                buildexe cras_test_client "$extraflags"
            '
        } | sh -s -e $SETOPTIONS

        echo "Installing CRAS..." 1>&2

        mkdir -p "$CRASBINDIR/" "$CRASLIBDIR/" "$ALSALIBDIR/"
        # Only install libcras.so.X.Y.Z
        /usr/bin/install -s libcras.so.*.* "$CRASLIBDIR/"
        # Generate symbolic link to libcras.so.X
        ldconfig -l "$CRASLIBDIR"/libcras.so.*.*
        /usr/bin/install -s libasound_module_pcm_cras.so "$ALSALIBDIR/"
        /usr/bin/install -s libasound_module_ctl_cras.so "$ALSALIBDIR/"
        /usr/bin/install -s cras_test_client "$CRASBINDIR/"
    ) # End compilation subshell
}

# On x86_64, the ALSA plugin needs to be compiled for both 32-bit and 64-bit
# to allow audio playback using 32-bit applications.
if [ "$ARCH" = 'amd64' ]; then
    build_cras 'i386'
fi

