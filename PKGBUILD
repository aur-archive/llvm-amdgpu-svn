# Maintainer: Lone_Wolf < lonewolf at xs4all dot nl>
# Contributor: Laurent Carlier <lordheavym@gmail.com>
# Contributor: Thomas Dziedzic < gostrc at gmail >
# Contributor: Roberto Alsina <ralsina@kde.org>
# Contributor: Tomas Lindquist Olsen <tomas@famolsen.dk>
# Contributor: Anders Bergh <anders@archlinuxppc.org>
# Contributor: Tomas Wilhelmsson <tomas.wilhelmsson@gmail.com>
# Contributor: Evangelos Foutras <evangelos@foutrelis.com>
# Contributor: Jan "heftig" Steffens <jan.steffens@gmail.com>
# Contributor: Sebastian Nowicki <sebnow@gmail.com>
# Contributor: Devin Cofer <ranguvar{AT]archlinux[DOT}us>
# Contributor: Tobias Kieslich <tobias@justdreams.de>
# Contributor: Geoffroy Carrier <geoffroy.carrier@aur.archlinux.org>
# Contributor: Gerardo Exequiel Pozzi <vmlinuz386@yahoo.com.ar>

pkgname=llvm-amdgpu-svn
pkgver=179090
pkgrel=1
pkgdesc='Low Level Virtual Machine with AMDGPU enabled, build from upstream svn trunk version'
arch=('i686' 'x86_64')
url="http://llvm.org"
license=('custom:University of Illinois/NCSA')
depends=('python2' 'ocaml')
makedepends=('subversion' 'python-sphinx')
source=("$pkgname::svn+http://llvm.org/svn/llvm-project/llvm/trunk"
        'llvm-Config-config.h'
        'llvm-Config-llvm-config.h')
conflicts=('llvm')
provides=('llvm')
md5sums=('SKIP'
         'e4f9c0c37d6858baf2a1099a73db0f6e'
         '295c343dcd457dc534662f011d7cff1a')

pkgver() {
  cd "$SRCDEST/$pkgname"
  svnversion | tr -d [A-z]
}

prepare() {

  cd "$pkgname"

  # Fix symbolic links from OCaml bindings to LLVM libraries
  sed -i 's:\$(PROJ_libdir):/usr/lib/llvm:' bindings/ocaml/Makefile.ocaml

  # Fix installation directories, ./configure doesn't seem to set them right
  sed -i -e 's:\$(PROJ_prefix)/etc/llvm:/etc/llvm:' \
         -e 's:\$(PROJ_prefix)/lib:$(PROJ_prefix)/lib/llvm:' \
         -e 's:\$(PROJ_prefix)/docs/llvm:$(PROJ_prefix)/share/doc/llvm:' \
    Makefile.config.in
  sed -i '/ActiveLibDir = ActivePrefix/s:lib:lib/llvm:' \
    tools/llvm-config/llvm-config.cpp
  sed -i 's:LLVM_LIBDIR="${prefix}/lib":LLVM_LIBDIR="${prefix}/lib/llvm":' \
    autoconf/configure.ac \
    configure

  # Fix insecure rpath (http://bugs.archlinux.org/task/14017)
  sed -i 's:$(RPATH) -Wl,$(\(ToolDir\|LibDir\|ExmplDir\))::g' Makefile.rules
}

build() {
  cd "$pkgname"

  # Apply strip option to configure
  _optimized_switch="enable"
  [[ $(check_option strip) == n ]] && _optimized_switch="disable"

  # Include location of libffi headers in CPPFLAGS
  export CPPFLAGS="$CPPFLAGS $(pkg-config --cflags libffi)"

  # Force the use of GCC instead of clang
  CC=gcc CXX=g++ \
  ./configure \
    --with-python=/usr/bin/python2 \
    --prefix=/usr \
    --libdir=/usr/lib/llvm \
    --sysconfdir=/etc \
    --enable-shared \
    --enable-libffi \
    --enable-targets=all \
    --enable-experimental-targets=R600 \
    --disable-expensive-checks \
    --disable-debug-runtime \
    --disable-assertions \
    --with-binutils-include=/usr/include \
    --$_optimized_switch-optimized

  make REQUIRES_RTTI=1
  make -C docs -f Makefile.sphinx man
  make -C docs -f Makefile.sphinx html
}

package() {
  cd "$pkgname"

  make DESTDIR="$pkgdir" install
  # OCaml bindings go to a separate package
  rm -rf "$srcdir"/{ocaml,ocamldoc}
  mv "$pkgdir"/usr/{lib/ocaml,share/doc/llvm/ocamldoc} "$srcdir"

  # Remove duplicate files installed by the OCaml bindings
  rm "$pkgdir"/usr/{lib/llvm/libllvm*,share/doc/llvm/ocamldoc.tar.gz}

  # Fix permissions of static libs
  chmod -x "$pkgdir"/usr/lib/llvm/*.a

  # Get rid of example Hello transformation
  rm "$pkgdir"/usr/lib/llvm/*LLVMHello.*

  # Add ld.so.conf.d entry
  install -d "$pkgdir/etc/ld.so.conf.d"
  echo /usr/lib/llvm >"$pkgdir/etc/ld.so.conf.d/llvm.conf"

  # Symlink LLVMgold.so into /usr/lib/bfd-plugins
  # (https://bugs.archlinux.org/task/28479)
  install -d "$pkgdir/usr/lib/bfd-plugins"
  ln -s ../llvm/LLVMgold.so "$pkgdir/usr/lib/bfd-plugins/LLVMgold.so"

  if [[ $CARCH == x86_64 ]]; then
    # Needed for multilib (https://bugs.archlinux.org/task/29951)
    # Header stubs are taken from Fedora
    for _header in config llvm-config; do
      mv "$pkgdir/usr/include/llvm/Config/$_header"{,-64}.h
      cp "$srcdir/llvm-Config-$_header.h" \
        "$pkgdir/usr/include/llvm/Config/$_header.h"
    done
  fi

  # Install man pages
  install -d "$pkgdir/usr/share/man/man1"
  cp docs/_build/man/*.1 "$pkgdir/usr/share/man/man1/"

  # Install html docs
  cp -r docs/_build/html/* "$pkgdir/usr/share/doc/llvm/html/"
  rm -r "$pkgdir/usr/share/doc/llvm/html/_sources"

  install -Dm644 LICENSE.TXT "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  find "$pkgdir" -type d -name .svn -exec rm -r '{}' +
}