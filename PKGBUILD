# Maintainer: Estela ad Astra <i@estela.cn>

pkgbase=linux-515-starfive-visionfive2
pkgver=5.15.96
pkgrel=1
_tag=VF2_v2.6.0
_srcname=linux-$_tag

_desc='Linux 5.15 for StarFive RISC-V VisionFive 2 Board'
url="https://github.com/starfive-tech/linux/"
arch=(riscv64)
license=(GPL2)
makedepends=(bc libelf pahole cpio perl tar xz)
options=('!strip')
source=("https://github.com/starfive-tech/linux/archive/refs/tags/${_tag}.tar.gz"
  "https://cdn.kernel.org/pub/linux/kernel/v5.x/patch-${pkgver}.xz"
  'config'
  'linux.preset'
  '90-linux.hook')

sha256sums=('a56fd7a684998f13667253b5a0a7af073322f87ad553c9498700013898040ee9'
            '23726cf33834f5d49ce31837ec552e1aff6c2833d7c1a93c15f6f9f83933dccb'
            'e639088bfc6c09e5af0e4cd5f9cbc9463bdce0355a6d84e968383316494bcae8'
            '57acae869144508c5600d6c8f41664f073f731c40cad2c58d2a1d55240495ddb'
            '5308a6dcabff290c627cab5c9db23c739eddbf7aa8a4984468ed59e6a5250702')

if [ $(uname -m) != riscv64 ]; then
  shopt -s expand_aliases
  alias make="make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu-"
  alias strip=riscv64-linux-gnu-strip
fi

prepare() {
  cd $_srcname

  echo "Applying patch $src..."
  patch -Np1 <"../patch-${pkgver}" || echo fuck

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" >localversion.10-pkgrel
  echo "${pkgbase#linux}" >localversion.20-pkgname

  echo "Setting config..."
  cp ../config .config
  make olddefconfig
  #make starfive_visionfive2_defconfig
  cp .config ../../config.new

  make -s kernelrelease >version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
}

_package() {
  pkgdesc="The $_desc kernel and modules"
  depends=(coreutils kmod mkinitcpio)
  optdepends=('wireless-regdb: to set the correct wireless channels of your country'
    'linux-firmware: firmware images needed for some devices')
  provides=("linux=${pkgver}" "WIREGUARD-MODULE")
  conflicts=('linux')

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  install -Dm644 "arch/riscv/boot/Image.gz" "$modulesdir/vmlinuz"
  install -Dm644 "arch/riscv/boot/Image.gz" "$pkgdir/boot/vmlinuz"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  echo "Installing dtbs..."
  make INSTALL_DTBS_PATH="$pkgdir/usr/share/dtbs/$kernver" dtbs_install
  make INSTALL_DTBS_PATH="$pkgdir/boot/dtbs/" dtbs_install

  # remove build links
  rm "$modulesdir"/build

  install -Dm644 ../linux.preset "${pkgdir}/etc/mkinitcpio.d/linux.preset"
  install -Dm644 ../90-linux.hook "${pkgdir}/usr/share/libalpm/hooks/90-linux.hook"

}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $_desc kernel"
  depends=(pahole)
  provides=("linux-headers=${pkgver}")
  conflicts=('linux-headers')

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/riscv" -m644 arch/riscv/Makefile
  cp -t "$builddir" -a scripts

  # required when DEBUG_INFO_BTF_MODULES is enabled
  install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/*

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/riscv" -a arch/riscv/include
  install -Dt "$builddir/arch/riscv/kernel" -m644 arch/riscv/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */riscv/ ]] && continue
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
    application/x-sharedlib\;*) # Libraries (.so)
      strip -v $STRIP_SHARED "$file" ;;
    application/x-archive\;*) # Libraries (.a)
      strip -v $STRIP_STATIC "$file" ;;
    application/x-executable\;*) # Binaries
      strip -v $STRIP_BINARIES "$file" ;;
    application/x-pie-executable\;*) # Relocatable binaries
      strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
