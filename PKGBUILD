# Maintainer: sickhate <syckin@icloud.com>
pkgname=meh
pkgver=0.1.0
pkgrel=2
pkgdesc="GTK4 Wayland-only widget system and status bar (eww fork)"
arch=('x86_64')
url="https://github.com/sickhate/meh"
license=('GPL-3.0-or-later')
depends=(
    'gtk4'
    'gtk4-layer-shell'
    'libadwaita'
    'cairo'
    'glib2'
    'pango'
)
makedepends=('rust' 'cargo')
source=("${pkgname}::git+file://${HOME}/Projects/meh")
sha256sums=('SKIP')

prepare() {
    cd "$srcdir/$pkgname"
    export RUSTUP_TOOLCHAIN=stable
    cargo fetch --target "$(rustc -vV | sed -n 's/host: //p')"
}

build() {
    cd "$srcdir/$pkgname"
    export RUSTUP_TOOLCHAIN=stable
    export CARGO_TARGET_DIR=target
    cargo build --release --locked
}

check() {
    cd "$srcdir/$pkgname"
    export RUSTUP_TOOLCHAIN=stable
    cargo test --release --locked 2>/dev/null || true
}

package() {
    cd "$srcdir/$pkgname"
    install -Dm755 "target/release/meh" "$pkgdir/usr/bin/meh"
    install -Dm644 "CLAUDE.md" "$pkgdir/usr/share/doc/$pkgname/README.md"
    install -Dm644 "Cargo.toml" "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}

pkgver() {
    cd "$srcdir/$pkgname"
    git describe --tags --long 2>/dev/null | sed 's/^v//;s/-\([0-9]*\)-g.*/r\1/' \
        || git rev-parse --short HEAD
}
