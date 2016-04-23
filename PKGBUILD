# Maintainer: Jiachen Yang <farseerfc@gmail.com>
pkgname=systemd-shutdown-diagnose
pkgver=1
pkgrel=1
pkgdesc="help to diagnose shutdown sequence for systemd"
arch=(any)
url="http://github.com/farseerfc/systemd-shutdown-diagnose"
license=('GPL')
depends=()
source=('analyze-shutdown'
    'diagnose.shutdown'
    'shutdown-diagnose.service'
    'start-diagnose-shutdown'
)
sha512sums=('883b12b23cad21f620fb10e6f682390d155da65f41517701296793e7ad4ca16e55547b78d3fb06dd697a261e5f236e5d96e6d6c539d16ed3a7b52f890066ac59'
            'ab00ff35e6e74f90589587c44d3efc4584f0d5d40e7767ee2fc982f2b20f0656dfbb58b3c3a8103a32538a28bb08dd15087114aae0b1f6f73a12b6073d32c99f'
            'db952a867bb5d7ba6283c5a816b9ae3736dfcfbe04ce67d1cef96049ad0951e05273cdb9115ba6d4e16f12c470f0e2dc8e4a9f1c9741a6cbda18091222e18d3c'
            '2ee770dce7cbf95d07b49e24522c6be2d7dd8f3cbe88097d6b27a9ea05052525ef58ef6c2f95bf09bcbc38db46918098faf13655c0f6853b3057dac9ec10e6bb')

package() {
    install -dm755 diagnose.shutdown /usr/lib/systemd/system-shutdown/
    install -dm755 start-diagnose-shutdown /usr/bin/
    install -dm755 analyze-shutdown /usr/bin/
    install -dm644 shutdown-diagnose.service /usr/lib/systemd/system/
}
