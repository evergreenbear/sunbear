# sunbear linux notes
## methodology
This is the initial prototyping stage of a linux distribution with pkgin as its sole package manager and with every system file belonging to a pkgin package.  
Currently, sunbear is far from this goal. For this first pass, the approach is to copy the kernel, toolchain, and other necessary items which pkgsrc cannot provide from a Void Linux tarball and then use pkgsrc to install packages from that point, keeping only the minimum necessary Void artifacts to be replaced by sunbear tooling at a later point in time.   
At such a point I will need to have my own build system to compile pkgsrc packages for a Linux target - there is no actively maintained repo that does this anymore, but as of 2024 Joyent was packaging linux binaries for SmartOS at https://pkgsrc.smartos.org/packages/Linux/el7/trunk/x86_64/All/. This package list could be used to create a suitable set of base packages for the sunbear repository.
## bootstrapping

### void linux base
Using a minimal Void Linux tarball, we will build pkgsrc and configure it to build to a clean directory that will act as the root directory for sunbear. The directory structure I am using is as follows:
```
$ ls /media/ssd/sunbear/
root sources void-root
```
`root`: root of sunbear system where pkgsrc will compile to  
`void-root`: location of extracted void linux tarball  
`sources`: location of prerequisite files used (e.g. void tarball archive)  
  
Extract the tarball:
```
cd void-root
tar xvf ../sources/void-x86_64-ROOTFS-[version].tar.xz
```
Set a few environment variables to make chrooting easier:
```
export VOIDROOT="/media/ssd/sunbear/void-root" 
mount --bind /proc $VOIDROOT/proc
mount --bind /sys $VOIDROOT/sys
mount --bind /dev $VOIDROOT/dev
mount --bind /dev/pts $VOIDROOT/dev/pts
mount --bind /media/ssd/sunbear/root /media/ssd/sunbear/void-root/mnt
chroot $VOIDROOT /bin/bash --login
```
Ensure the chroot was successful:
```
# cat /etc/os-release
NAME="Void"
ID="void"
PRETTY_NAME="Void Linux"
[...]
```
### pkgsrc

Pkgsrc requires the host system to provide a toolchain. Void does not release images frequently, so we'll also make sure everything is up-to-date before we get to work:
```
xbps-install -Su xbps
xbps-install -Su
xbps-install -S gcc make automake autoconf patch curl tar gzip bzip2 xz git vim
exit
chroot $VOIDROOT /bin/bash --login
```
You will need an internet connection from within the chroot to be able to install these packages. On my ethernet-connected system, this did not require any manual configuration.
Now we can acquire pkgsrc and start bootstrapping to /mnt:
```
cd /tmp
curl -fLO https://cdn.netbsd.org/pub/pkgsrc/pkgsrc-2026Q1/pkgsrc-2026Q1.tar.bz2
tar xvf pkgsrc-2026Q1.tar.bz2 -C /usr
```
```
# Required or build fails
cp /usr/share/automake-*/install-sh /usr/pkgsrc/archivers/libarchive/files/
cp /usr/share/automake-*/install-sh /usr/pkgsrc/archivers/libarchive/files/build/autoconf/

cd /usr/pkgsrc/bootstrap/
./bootstrap \
  --prefix=/usr/pkg \
  --sysconfdir=/etc \
  --varbase=/var \
  --make-jobs=16 \
  --abi=64 \
  --pkgdbdir=/var/db/pkg
```
You will see something like this when the bootstrap is completed:
```  
===========================================================================

Please remember to add /usr/pkg/bin to your PATH environment variable
and /usr/pkg/man to your MANPATH environment variable, if necessary.

An example mk.conf file with the settings you provided to "bootstrap"
has been created for you. It can be found in:

      /etc/mk.conf

You can find extensive documentation of the NetBSD Packages Collection
in /usr/pkgsrc/doc/pkgsrc.txt.

Thank you for using pkgsrc!

===========================================================================

===> bootstrap started: Fri Jun 19 09:48:28 PM UTC 2026
===> bootstrap ended:   Fri Jun 19 09:49:46 PM UTC 2026
```
We'll finish the setup as directed:
```
# add paths to /etc/profile via sed
sed -i "/^unset appendpath/i \\
appendpath '/usr/pkg/bin'\\
appendpath '/usr/pkg/sbin'\\
export MANPATH=\"/usr/pkg/man:\$MANPATH\"" /etc/profile

mkdir -p /var/distfiles /var/packages /var/obj
```
Setup an `/etc/mk.conf`:
```
# /etc/mk.conf - void linux build tarball
# Fri Jun 19 09:48:29 PM UTC 2026

.ifdef BSD_PKG_MK       # begin pkgsrc settings

ABI=                    64

PKG_DBDIR=              /var/db/pkg
LOCALBASE=              /usr/pkg
VARBASE=                /var
PKG_SYSCONFBASE=        /etc
PKG_TOOLS_BIN=          /usr/pkg/sbin
PKGINFODIR=             info
PKGMANDIR=              man

# Parallelism
MAKE_JOBS=              16

# Fetch
FETCH_USING=            curl
DISTDIR=                /var/distfiles
PACKAGES=               /var/packages
WRKOBJDIR=              /var/obj

PREFER_PKGSRC=          yes

.endif                  # end pkgsrc settings
```
Packages are built on the void tarball and copied over upon completion.

### sunbear root
-- DEPRECATED -------------------------------------------------------------------------------------------------------  
We'll go ahead and install `openssl` and `pkgin` on the Void tarball so that they are copied to the sunbear root with everything else:
```
cd /usr/pkgsrc/security/openssl
bmake install clean

cd /usr/pkgsrc/pkgtools/pkgin
bmake install clean
```
--END DEPRECATED--------------------------------------------------------------------------------------------------------------------------------  
  
Now begin to fill out /mnt with the necessary libraries from Void:
```
# Set up directory structure in /mnt
mkdir -p /mnt/usr/lib
mkdir -p /mnt/usr/lib64
mkdir -p /mnt/var/db/pkg
ln -s usr/lib /mnt/lib
ln -s usr/lib64 /mnt/lib64

# Copy essential runtime libs from Void to act as vestigial base
for lib in \
  libc.so.6 \
  libm.so.6 \
  libdl.so.2 \
  libpthread.so.0 \
  libresolv.so.2 \
  libgcc_s.so.1 \
  libstdc++.so.6; do
    cp -a /usr/lib/${lib}* /mnt/usr/lib/ 2>/dev/null
done

# lib64 needs ld-linux and libc specifically
cp -a /usr/lib/ld-linux-x86-64.so.2 /mnt/usr/lib64/
cp -a /usr/lib/libc.so.6 /mnt/usr/lib64/
```
You could use pkgsrc to build the core packaging tools + pkgin, but it's easier to copy the built versions over from the Void tarball:
```
mkdir -p /mnt/usr/pkg
cp -a /usr/pkg/* /mnt/usr/pkg/

mkdir -p /mnt/var/db/pkg
cp -a /var/db/pkg/* /mnt/var/db/pkg/

chroot /mnt /usr/pkg/sbin/pkg_info -K /var/db/pkg
```
If successful you should see a small list of packages:
```
cwrappers-20220403  pkgsrc compiler wrappers
bootstrap-mk-files-20250601 *.mk files for the bootstrap bmake utility
pkg_install-20260227 Package management and administration tools for pkgsrc
mktools-20250213    Collection of pkgsrc mk infrastructure tools
bmake-20240909      Portable (autoconf) version of NetBSD 'make' utility
libnbcompat-20251029 Portable NetBSD compatibility library
[...]
```
We'll go ahead and make a `/mnt/etc/mk.conf` for sunbear:
```
# /etc/mk.conf - sunbear linux

.ifdef BSD_PKG_MK       # begin pkgsrc settings

ABI=                    64
PKG_DBDIR=              /var/db/pkg
LOCALBASE=              /usr/pkg
VARBASE=                /var
PKG_SYSCONFBASE=        /etc
PKG_TOOLS_BIN=          /usr/pkg/sbin
PKGINFODIR=             info
PKGMANDIR=              man

PKG_DEFAULT_OPTIONS+=   openssl
PKG_OPTIONS.libfetch+=  openssl

PREFER_PKGSRC=          yes

.endif                  # end pkgsrc settings
```
However we will continue building packages from the Void tarball and copy them over after. We'll build the minimum packages necessary for a functional chroot:

```bash
cd /usr/pkgsrc/shells/bash && bmake install clean
cd /usr/pkgsrc/sysutils/coreutils && bmake install clean
cd /usr/pkgsrc/editors/vim && bmake install clean

cp -a /usr/pkg/* /mnt/usr/pkg/
cp -a /var/db/pkg/* /mnt/var/db/pkg/
```
Then we'll fill in the fundamental *nix configuration:
```
mkdir -p /mnt/etc /mnt/root /mnt/tmp /mnt/proc /mnt/sys /mnt/dev
chmod 1777 /mnt/tmp
echo "root:x:0:0:root:/root:/usr/pkg/bin/bash" > /mnt/etc/passwd
echo "root:x:0:" > /mnt/etc/group
```
At this stage, the Void environment can be switched out for sunbear:
```
chroot /media/ssd/sunbear/root /usr/pkg/bin/bash --login
```