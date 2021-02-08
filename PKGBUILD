# Maintainer: Joan Figueras <ffigue at gmail dot com>
# Contributor: Jacek Szafarkiewicz <szafar@linux.pl>
# Contributor: Maxim Baz <$pkgname at maximbaz dot com>

##
## The following variables can be customized at build time. Use env or export to change at your wish
##
##   Example: env USE_SCCACHE=1 COMPONENT=1 makepkg -sc
##
## sccache for faster builds - https://github.com/brave/brave-browser/wiki/sccache-for-faster-builds
## Valid numbers between: 0 and 1
## Default is: 0 => not use sccache
if [ -z ${USE_SCCACHE+x} ]; then
  USE_SCCACHE=0
fi
##
## COMPONENT variable
## 0 -> build normal (with debug symbols)
## 1 -> release (default)
## 2 -> static
## 3 -> debug
## https://github.com/brave/brave-browser/wiki#clone-and-initialize-the-repo
if [ -z ${COMPONENT+x} ]; then
  COMPONENT=1
fi
##

pkgname=brave-nightly
pkgver=1.22.9
pkgrel=1
pkgdesc='A web browser that stops ads and trackers by default. Nightly Channel'
arch=('x86_64')
url='https://www.brave.com/download'
license=('custom')
depends=('gtk3' 'nss' 'alsa-lib' 'libxss' 'ttf-font' 'libva' 'json-glib')
makedepends=('git' 'npm<7.0.0' 'python' 'python2' 'icu' 'glibc' 'gperf' 'java-runtime-headless' 'clang' 'python2-setuptools')
optdepends=('cups: Printer support'
            'libpipewire02: WebRTC desktop sharing under Wayland'
            'org.freedesktop.secrets: password storage backend on GNOME / Xfce'
            'kwallet: for storing passwords in KWallet on KDE desktops'
            'sccache: For faster builds')
chromium_base_ver="89"
patchset="6"
patchset_name="chromium-${chromium_base_ver}-patchset-${patchset}"
_launcher_ver=6
source=("https://github.com/brave/brave-browser/archive/v${pkgver}.tar.gz"
        'brave-launcher'
        'brave-browser-nightly.desktop'
        "chromium-launcher-$_launcher_ver.tar.gz::https://github.com/foutrelis/chromium-launcher/archive/v$_launcher_ver.tar.gz"
        "https://github.com/stha09/chromium-patches/releases/download/${patchset_name}/${patchset_name}.tar.xz")
arch_revision=4332a9b5a5f7e1d5ec8e95ee51581c3e55450f41
for Patches in \
	subpixel-anti-aliasing-in-FreeType-2.8.1.patch
do
  source+=("${Patches}::https://git.archlinux.org/svntogit/packages.git/plain/trunk/${Patches}?h=packages/chromium&id=${arch_revision}")
done

sha256sums=('19b9ecfbf6f5e91417fe4a2a345a8cd777602ea0e5680df2a747461babab07e6'
            '8c2ac6247ea030606ae43aa486531c13d1e101a8564ffae65a337a8037be142b'
            '58ead81fdd519b64c8d2f7776e54a0860104d39fcc6b4858e136c6fa8061b50f'
            '04917e3cd4307d8e31bfb0027a5dce6d086edb10ff8a716024fbb8bb0c7dccf1'
            '359d2847e775d8cf6f4e0b12c94c8f2718f0fd562427859c596ce1c3711dbd8e'
            '1e2913e21c491d546e05f9b4edf5a6c7a22d89ed0b36ef692ca6272bcd5faec6')

prepare() {
  cd "brave-browser-${pkgver}"

  # Hack to prioritize python2 in PATH
  mkdir -p "${srcdir}/bin"
  ln -sf /usr/bin/python2 "${srcdir}/bin/python"
  ln -sf /usr/bin/python2-config "${srcdir}/bin/python-config"
  export PATH="${srcdir}/bin:${PATH}"

  msg2 "Prepare the environment..."
  npm install
  if [ -d src/out/Release ]; then
    npm run sync -- --force
  else
    npm run init
  fi

  msg2 "Apply Chromium patches..."
  cd src/

  # https://crbug.com/893950
  sed -i -e 's/\<xmlMalloc\>/malloc/' -e 's/\<xmlFree\>/free/' \
    third_party/blink/renderer/core/xml/*.cc \
    third_party/blink/renderer/core/xml/parser/xml_document_parser.cc \
    third_party/libxml/chromium/*.cc

  # Upstream fixes
  patch -Np1 -d third_party/skia <../../subpixel-anti-aliasing-in-FreeType-2.8.1.patch

  # Fixes for building with libstdc++ instead of libc++
  patch -Np1 -i ../../patches/chromium-87-openscreen-include.patch
  patch -Np1 -i ../../patches/chromium-88-CompositorFrameReporter-dcheck.patch
  patch -Np1 -i ../../patches/chromium-88-ideographicSpaceCharacter.patch
  patch -Np1 -i ../../patches/chromium-88-AXTreeFormatter-include.patch

  # Force script incompatible with Python 3 to use /usr/bin/python2
  sed -i '1s|python$|&2|' third_party/dom_distiller_js/protoc_plugins/*.py

  # Hacky patching
  sed -e 's/enable_distro_version_check = true/enable_distro_version_check = false/g' -i chrome/installer/linux/BUILD.gn
}

build() {
  cd "brave-browser-${pkgver}"

  if check_buildoption ccache y; then
    # Avoid falling back to preprocessor mode when sources contain time macros
    export CCACHE_SLOPPINESS=time_macros
  fi

  export CC=clang
  export CXX=clang++
  export AR=ar
  export NM=nm

  # Hack to prioritize python2 in PATH
  mkdir -p "${srcdir}/bin"
  ln -sf /usr/bin/python2 "${srcdir}/bin/python"
  ln -sf /usr/bin/python2-config "${srcdir}/bin/python-config"
  export PATH="${srcdir}/bin:${PATH}"

  if [ "$USE_SCCACHE" -eq "1" ]; then
    echo "sccache = /usr/bin/sccache" >> .npmrc
  fi

  echo 'brave_variations_server_url = https://variations.brave.com/seed' >> .npmrc
  echo 'brave_stats_updater_url = https://laptop-updates.brave.com' >> .npmrc
  echo 'brave_stats_api_key = fe033168-0ff8-4af6-9a7f-95e2cbfc' >> .npmrc
  echo 'brave_sync_endpoint = https://sync-v2.brave.com/v2' >> .npmrc

  ## See explanation on top to select your build
  case ${COMPONENT} in
    0)
       msg2 "Normal build (with debug)"
       npm run build
       ;;
    2)
       msg2 "Static build"
       npm run build -- Static
       ;;
    3)
       msg2 "Debug build"
       npm run build -- Debug
       ;;
    *)
       msg2 "Release build"
       npm run build Release
       ;;
  esac
}

package() {
  install -d -m0755 "${pkgdir}/usr/lib/${pkgname}/"{,swiftshader,locales,resources}

  # Copy necessary release files
  cd "brave-browser-${pkgver}/src/out/Release"
  cp -a --reflink=auto \
    WidevineCdm \
    MEIPreload \
    brave \
    brave_*.pak \
    chrome_*.pak \
    resources.pak \
    v8_context_snapshot.bin \
    libGLESv2.so \
    libEGL.so \
    "${pkgdir}/usr/lib/${pkgname}/"
  cp -a --reflink=auto \
    swiftshader/libGLESv2.so \
    swiftshader/libEGL.so \
    "${pkgdir}/usr/lib/${pkgname}/swiftshader/"
  cp -a --reflink=auto \
    locales/*.pak \
    "${pkgdir}/usr/lib/${pkgname}/locales/"
  cp -a --reflink=auto \
    resources/brave_extension \
    resources/brave_rewards \
    "${pkgdir}/usr/lib/${pkgname}/resources/"

  cd "${srcdir}"
  install -Dm0755 brave-launcher "${pkgdir}/usr/bin/${pkgname}"
  install -Dm0644 -t "${pkgdir}/usr/share/applications/" brave-browser-nightly.desktop
  install -Dm0644 "brave-browser-${pkgver}/src/brave/app/theme/brave/product_logo_128.png" "${pkgdir}/usr/share/pixmaps/${pkgname}.png"
  install -Dm0644 -t "${pkgdir}/usr/share/licenses/${pkgname}" "brave-browser-${pkgver}/LICENSE"
}

# vim:set ts=4 sw=4 et:
