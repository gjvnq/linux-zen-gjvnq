# Maintainer: Shayne Hartford<shayneehartford@gmail.com>

pkgbase=linux-zen-gjvnq
pkgver=5.7.zen1
pkgrel=6
pkgdesc='Linux ZEN with VFIO patches and https://clbin.com/VCiYJ patch by u/Aiberia (Compiled for AMD ZEN 2)'
_srctag=v${pkgver%.*}-${pkgver##*.}
url="https://github.com/zen-kernel/zen-kernel/commits/$_srctag"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc kmod libelf
  xmlto python-sphinx python-sphinx_rtd_theme graphviz imagemagick
  git
)
options=('!strip')
_srcname=zen-kernel
source=(
  #"$_srcname::git+https://github.com/zen-kernel/zen-kernel?signed#tag=$_srctag"
  config         # the main kernel config file
  add-acs-overrides.patch # patch for acs overrides
  aiberia.patch           # patch for 'PCI header type 127' error
  sphinx-workaround.patch
)
validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
  '8218F88849AAC522E94CF470A5E9288C4FA415FA'  # Jan Alexander Steffens (heftig)
  'FEDB7014D7D90ADE3B74B54B96E477D4BFC6B8D9'  # gabrieljvnq@gmail.com
)
sha256sums=('fa364798c588add159938fcbcc1b870950b1163c9c778b383088e39bfae12468'
            '0352f4a52166bef96ac5b4ff1d2bcb61efd9580803af57ce0f3019565daa0bc2'
            '130c58ab30968b02087c54c9587683c1c7baebf7eaf6b128ca0789615d225775'
            '8cb21e0b3411327b627a9dd15b8eb773295a0d2782b1a41b2a8839d1b2f5778c')

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

# Uncomment line below if your signing_key.pem is password encrypted
#export KBUILD_SIGN_PIN=1234

prepare() {
  NC='\033[0m'
  Blue='\033[0;34m'
  Bold='\033[1m'
  MSG_HEAD="$NC$Blue  -> $NC$Bold"

  if [ ! -d $_srcname ]; then
    echo -e $MSG_HEAD"Downloading zen kernel..."$NC
    git clone --depth=1 --branch=$_srctag "https://github.com/zen-kernel/zen-kernel" $_srcname
  fi

  if [ -f "../signing_key.pem" ]; then
    echo -e $MSG_HEAD"Found signing_key.pem"$NC
    echo "Using $(pwd)/../signing_key.pem for module signing"
    cp ../signing_key.pem $_srcname/certs/signing_key.pem
  elif [ -f "../x509.genkey" ]; then
    echo -e $MSG_HEAD"Found x509.genkey"$NC
    echo "Using $(pwd)/../x509.genkey to generate a key for module signing"
    cp ../x509.genkey $_srcname/certs/x509.genkey
  else
    echo -e $MSG_HEAD"Found neither signing_key.pem nor x509.genkey"$NC
    echo "Will auto generate module singing key with default config"
    echo "Consider setting placing either signing_key.pem or x509.genkey on: $(pwd)/../"
  fi

  cd $_srcname

  echo -e $MSG_HEAD"Setting version..."$NC
  git reset --hard $_srctag
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    echo -e $MSG_HEAD"Applying patch $src..."$NC
    patch -Np1 < "../$src"
  done

  echo -e $MSG_HEAD"Setting config..."$NC
  cp ../config .config
  make olddefconfig

  make -s kernelrelease > version
  echo -e $MSG_HEAD"Prepared $pkgbase version $(<version)"$NC
}

build() {
  cd $_srcname
  make all
  make htmldocs
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
 
