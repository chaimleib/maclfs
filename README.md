# Mac LFS

The goal of this project is to cross compile Linux on a Mac host. The only requirements are:

* Xcode command line tools
* [Homebrew](https://brew.sh)
* [Paragon extFS for Mac](https://www.paragon-software.com/home/extfs-mac/)
  ($39.95), or some other method of mounting and modifying an ext4 volume

This guide is based on [Linux from
Scratch](https://linuxfromscratch.org/view/lfs/), [Cross LFS - Embedded
x86](https://clfs.org/view/clfs-embedded/x86/) and [dslm4515's
Musl-LFS](https://github.com/dslm4515/Musl-LFS/blob/master/doc). I updated to
the latest packages as of 2021-01-08. The disk formatting tools were updated
2021-09-24.

# Set up environment

> `Parted` depends on glibc, which does not compile under macOS (dunno about
> musl). Compiling `gdisk` and `sgdisk` from source requires the brew packages
> `ossp-uuid` and `popt`.  However, rather than descend into the dependencies
> and make the project independent of `brew`, I favored speed at this stage,
> since the goal is to learn about Linux, rather than macOS.

```bash
git clone https://github.com/chaimleib/maclfs
cd maclfs
```

To create the boot disk, we need `gptfdisk` and `e2fsprogs`. Building the
header files will require GNU's version of sed.

```bash
brew install gptfdisk e2fsprogs gsed
```


# Creating the target disk

```bash
# create a blank 15GB file
dd bs=4096 if=/dev/zero of=lfs.iso count=15728540

# create loopback device from blank file, and remember the device name
# In case you need to run this command again after a reboot and other
# partitions exist already, the `; exit` to awk behaves like `head -n1` to grab
# just the disk device name and not partition device names.
lfsDev=$(hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount lfs.iso |
  awk '{print $1; exit}')

# partition the disk
# -g                   Use GPT partitioning
# -n 1::+200M -t ef00  First partition should be 200MB EFI
# -n 2:: -t 8304       Rest should be ext4
# -c 2:'LFS Disk'      Name the main partition
sgdisk $lfsDev -g -n 1::+200M -t ef00 -n 2:: -t 8304
```

Now you have the necessary partition locations, but they aren't formatted yet.

```bash
# if you wish, verify the partitions.
sgdisk -p $lfsDev

Disk /dev/disk2: 31457080 sectors, 15.0 GiB
Sector size (logical): 512 bytes
Disk identifier (GUID): 87317754-AF95-4293-83D0-1F9F128FAA71
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 31457046
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          411647   200.0 MiB   EF00
   2          411648        31457046   14.8 GiB    8304  LFS Disk
```

Now, format the main partition.

```bash
mke2fs -t ext4 -L lfsdisk ${lfsDev}s2
```

Install extFS for Mac by Paragon Software ($39.95 or Free 10-day trial) to
mount the disk image.

https://www.paragon-software.com/home/extfs-mac/

If you have to reboot, use the following command to create the loopback device
again:

```bash
lfsDev=$( \
  hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount lfs.iso |
  awk '{print $1; exit}')
```

Then, extFS should be able to mount it.


# Build directory, Packages and Patches

Give all users permissions to modify the disk:

```bash
LFS=/Volumes/lfsdisk
sudo chmod 777 $LFS
```

Create a directory that will hold the tools we need before we chroot or boot
the new linux system. In other words, the tools in this folder will be used
from macOS.

```bash
mkdir -pv $LFS/cross-tools/{bin,lib/pkgconfig}
```

Remember installing `gsed` a while back? Make a link to it named `sed` in our
cross-tools, so that it is available when we isolate our PATH variable:

```bash
ln -s /usr/local/bin/gsed $LFS/cross-tools/bin/sed
```

Create a bashrc:

```bash
cat << "EOF" > $MLFS/bashrc
set +h
umask 022
MLFS=$HOME/maclfs
LFS=/Volumes/lfsdisk
LC_ALL=POSIX
PATH=$LFS/cross-tools/bin:$LFS/cross-tools/$LFS_TGT/bin:/bin:/usr/bin
MANPATH=$LFS/cross-tools/share/man:$MANPATH
PKG_CONFIG_PATH=$LFS/cross-tools/lib/pkgconfig
export MLFS LFS LC_ALL PATH MANPATH PKG_CONFIG_PATH
unset CFLAGS

export LFS_HOST="$MACHTYPE-cross-darwin"
export LFS_TGT=x86_64-mlfs-linux-musleabihf
export BUILD64=-m64
export LFS_CPU=k8-sse3
export LFS_ARCH=x86_64
EOF
```

Then create and run a script to isolate your environment variables and load
your bashrc for you, make it executable, and run it. This reduces the chances
that variables from your host environment can affect your build.

```bash
cat << "EOF" > $MLFS/lfs.env
#!/bin/bash
exec env -i \
  HOME=$HOME \
  TERM=$TERM \
  PS1='\u:\w\$ ' \
  /bin/bash --rcfile bashrc
EOF
chmod +x $MLFS/lfs.env
$MLFS/lfs.env
```

If you ever need to exit the new environment, type `exit`. To re-enter it, run `$MLFS/lfs.env`.

# Cross toolchain

## Prereqs for linux headers

Instead of `wget`, use `curl -LJO` to download each URL in packages.txt and patches.txt.

> On 2021-09-24, I had HTTPS certificate issues with kernel.org and other
> source mirrors. For now, using `curl` works around it. Otherwise, I would
> have installed `wget` earlier, linked that and used it instead.
>
> `-LJO` must precede each URL sent to `curl`. `L` means to follow redirects,
> and `O` means to save a file with an automatically-determined name. However,
> the default is to figure out the name from the URL which is not always the
> original file name.  `J` says to use the name provided in the response
> headers.

The following commands assume that you have downloaded, decompressed and cd-ed
into the packages listed in packages.txt. Some packages also require
corresponding patches in patches.txt.

## Linux (headers only)
Now make the linux headers.

```bash
make mrproper
make ARCH=$LFS_ARCH headers
make ARCH=$LFS_ARCH INSTALL_HDR_PATH=$LFS/cross-tools/$LFS_TGT headers_install
```

> The above command failed to work on an HFS+ disk, and only succeeded inside the ext4 disk image.
>
> Since kernel version 5.5, `make headers_check` has been a no-op.

## Binutils

```bash
mkdir -v build
cd build
../configure \
   --prefix=$LFS/cross-tools/$LFS_TGT \
   --target=$LFS_TGT \
   --with-sysroot=$LFS/cross-tools/$LFS_TGT \
   --disable-nls \
   --disable-multilib
make configure-host
make
make install
```

### Musl (headers only)

```bash
./configure --prefix=$LFS/cross-tools
make install-headers
cp -r $LFS/cross-tools/include/{bits,{byteswap,elf,endian,features}.h} /usr/local/include

# avoid "typedef redefinition with different types" error with macOS headers
sed -i '/typedef unsigned _Int64 uint64_t/d' /usr/local/include/bits/alltypes.h
```

## GCC (static)

```bash
for p in mpfr gmp mpc; do
  tar -xf ../$p-*.tar.*
  mv -v $p-* $p
done
mkdir -v build
cd build
../configure \
  --prefix=$LFS/cross-tools \
  --build=$LFS_HOST \
  --host=$LFS_HOST \
  --target=$LFS_TGT \
  --with-sysroot=$LFS/cross-tools/$LFS_TGT \
  --disable-nls  \
  --disable-shared \
  --without-headers \
  --with-newlib \
  --disable-decimal-float \
  --disable-libgomp \
  --disable-libmudflap \
  --disable-libssp \
  --disable-libatomic \
  --disable-libquadmath \
  --disable-threads \
  --enable-languages=c \
  --disable-multilib \
  --with-mpfr-include=$PWD/../mpfr/src \
  --with-mpfr-lib=$PWD/mpfr/src/.libs \
  --with-arch=$LFS_CPU

# remember -j8 as applicable
make all-gcc all-target-libgcc
make install-gcc install-target-libgcc
```

> Attempting to include isl here will cause install to fail.

## musl

```bash
patch -Np1 -i ../wcsnrtombs_cve_2020_28928_diff.txt
./configure \
  CROSS_COMPILE=$LFS_TGT- \
  --prefix=/ \
  --target=$LFS_TGT
make
DESTDIR=$LFS/cross-tools/$LFS_TGT make install
```

## GCC (final cross-compiler)

```bash
for p in mpfr gmp mpc; do
  tar -xf ../$p-*.tar.*
  mv -v $p-* $p
done
```

Now prepare the build directory and make a symlink to enable POSIX threads.

```bash
mkdir -v build
cd build
mkdir -pv $LFS_TGT/libgcc
ln -s ../../../libgcc/gthr-posix.h $LFS_TGT/libgcc/gthr-default.h
```

```bash
../configure \
  --prefix=$LFS/cross-tools \
  --build=$LFS_HOST \
  --host=$LFS_HOST \
  --target=$LFS_TGT \
  --with-sysroot=$LFS/cross-tools/$LFS_TGT \
  --disable-nls \
  --enable-languages=c \
  --enable-c99 \
  --enable-long-long \
  --disable-libmudflap \
  --disable-multilib \
  --with-arch=$LFS_CPU
make
make install
```

> Do not attempt to also compile the c++ compiler at this stage. This leads to
> a dependency hell for files normally provided by glibc, but glibc has not
> been ported to work on macOS.

## Findutils

The busybox Makefile requires us to have the GNU version of xargs, which is
provided by GNU Findutils.

```bash
./configure --prefix=$LFS/cross-tools
make
make install
```

## Pkg-config

To allow the menuconfig of the linux kernel to find our Ncurses library,
Pkg-config needs to be present to point the linker in the right direction.

```bash
./configure --prefix=$LFS/cross-tools  \
            --with-internal-glib       \
            --disable-host-tool        \
            --docdir=$LFS/cross-tools/share/doc/pkg-config-0.29.2
make
make install
```

## Ncurses

When compiling the linux kernel, the menuconfig step requires Ncurses.

```bash
sed -i '/LIBTOOL_INSTALL/d' c++/Makefile.in
sed -i s/mawk// configure
```

To bootstrap the compilation, we need to build the tic program first.

```bash
mkdir build
cd build
../configure
make -C include
make -C progs tic
cd ..
```

Now build the rest of Ncurses.

```bash
./configure --prefix=$LFS/cross-tools                \
            --mandir=$LFS/cross-tools/share/man      \
            --with-manpage-format=normal             \
            --with-shared                            \
            --without-debug                          \
            --without-ada                            \
            --without-normal                         \
            --enable-pc-files                        \
            --enable-widec
make
make TIC_PATH=$PWD/build/progs/tic install
echo "INPUT(-lncursesw)" > $LFS/cross-tools/lib/libncurses.so
```

In case there are libraries expecting the non-wide character libs, link them to
the wide character versions.

```bash
for lib in ncurses form panel menu ; do
    rm -vf                    $LFS/cross-tools/lib/lib${lib}.so
    echo "INPUT(-l${lib}w)" > $LFS/cross-tools/lib/lib${lib}.so
    ln -sfv ${lib}w.pc        $LFS/cross-tools/lib/pkgconfig/${lib}.pc
done
```

> Also, make sure that older programs looking for curses will be redirected to ncurses. Such programs would also require version 5, so we have to build again.

> ```bash
> rm -vf                     $LFS/cross-tools/lib/libcursesw.so
> echo "INPUT(-lncursesw)" > $LFS/cross-tools/lib/libcursesw.so
> ln -sfv libncurses.so      $LFS/cross-tools/lib/libcurses.so
> make distclean
> ./configure --prefix=$LFS/cross-tools  \
>             --with-shared    \
>             --without-normal \
>             --without-debug  \
>             --without-cxx-binding \
>             --with-abi-version=5 
> make sources libs
> cp -av lib/lib*.so.5* $LFS/cross-tools/lib
> ```

# Building the LFS system

## Build variables

```bash
cat << EOF >> $MLFS/bashrc

export CC="${LFS_TGT}-gcc --sysroot=${LFS}"
export CXX="${LFS_TGT}-g++ --sysroot=${LFS}"
export AR="${LFS_TGT}-ar"
export AS="${LFS_TGT}-as"
export LD="${LFS_TGT}-ld --sysroot=${LFS}"
export RANLIB="${LFS_TGT}-ranlib"
export READELF="${LFS_TGT}-readelf"
export STRIP="${LFS_TGT}-strip"
EOF
source $MLFS/bashrc
```

## LSB folder tree

Create the LSB (Linux Standard Base) folder structure.

```bash
cd $LFS
mkdir -pv bin boot dev etc home lib/{firmware,modules}
mkdir -pv mnt opt proc sbin srv sys
mkdir -pv var/{cache,lib,local,lock,log,opt,run,spool}
install -dv -m 0750 root
install -dv -m 1777 {var/,}tmp
mkdir -pv usr/{,local/}{bin,include,lib,lib64,sbin,share,src}
```

## Config files

Set up config files.

List of mounted files.

```bash
ln -svf ../proc/mounts etc/mtab
```

```bash
cat > etc/passwd << "EOF"
root::0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/bin/false
daemon:x:2:6:daemon:/sbin:/bin/false
adm:x:3:16:adm:/var/adm:/bin/false
lp:x:10:9:lp:/var/spool/lp:/bin/false
mail:x:30:30:mail:/var/mail:/bin/false
news:x:31:31:news:/var/spool/news:/bin/false
uucp:x:32:32:uucp:/var/spool/uucp:/bin/false
operator:x:50:0:operator:/root:/bin/ash
postmaster:x:51:30:postmaster:/var/spool/mail:/bin/false
nobody:x:65534:65534:nobody:/:/bin/false
EOF
```

```bash
cat > etc/group << "EOF"
root:x:0:
bin:x:1:
sys:x:2:
kmem:x:3:
tty:x:4:
tape:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
EOF
```

Remember who last logged on.

```bash
touch var/log/lastlog
chmod -v 664 var/log/lastlog
```

## Libgcc

Install a copy of libgcc, and strip it to make it smaller.

```bash
if [ "$LFS_ARCH" = 'x86_64' ]; then
  cp -v cross-tools/$LFS_TGT/lib64/libgcc_s.so.1 lib64
  $LFS_TGT-strip lib64/libgcc_s.so.1
else
  cp -v cross-tools/$LFS_TGT/lib/libgcc_s.so.1 lib
  $LFS_TGT-strip lib/libgcc_s.so.1
fi
```

## Musl

```bash
patch -Np1 -i ../wcsnrtombs_cve_2020_28928_diff.txt
./configure \
  CROSS_COMPILE=$LFS_TGT- \
  --prefix=/ \
  --disable-static \
  --target=$LFS_TGT
make -j8
DESTDIR=$LFS make install-libs
```

## Busybox

```bash
make distclean
make ARCH=$LFS_ARCH defconfig
```

Modify the default config to disable features that will not build under musl:

```bash
sed -i 's/\(CONFIG_\)\(.*\)\(INETD\)\(.*\)=y/# \1\2\3\4 is not set/g' .config
sed -i 's/\(CONFIG_IFPLUGD\)=y/# \1 is not set/' .config
sed -i 's/\(CONFIG_FEATURE_WTMP\)=y/# \1 is not set/' .config
sed -i 's/\(CONFIG_FEATURE_UTMP\)=y/# \1 is not set/' .config
sed -i 's/\(CONFIG_UDPSVD\)=y/# \1 is not set/' .config
sed -i 's/\(CONFIG_TCPSVD\)=y/# \1 is not set/' .config
```

The make script failed for me while trying to remove the contents of
`.tmp_versions/`. To let that command succeed, add a dummy file there for it to
remove, and then compile the package.

```bash
touch .tmp_versions/1
make ARCH=$LFS_ARCH CROSS_COMPILE=$LFS_TGT-
make ARCH=$LFS_ARCH CROSS_COMPILE=$LFS_TGT- CONFIG_PREFIX=$LFS install
cp -v examples/depmod.pl $LFS/cross-tools/bin
chmod -v 755 $LFS/cross-tools/bin/depmod.pl
```

## Iana-etc

```bash
cp services protocols $LFS/etc
```


# Make the system bootable

## Create /etc/fstab

```bash
cat > $LFS/etc/fstab << "EOF"
# file-system  mount-point  type   options          dump  fsck
EOF
```

## Linux kernel

```bash
make mrproper
make ARCH=$LFS_ARCH CROSS_COMPILE=$LFS_TGT- menuconfig
```
