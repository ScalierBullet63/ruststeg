# RustSteg

> Hide encrypted messages inside images — a steganography CLI written in Rust.

[![crates.io][crates-badge]][crates-url]
[![CI][ci-badge]][ci-url]
[![License: GPL-3.0-only][license-badge]][license-url]
[![Rust 1.85+][rust-badge]][rust-url]

RustSteg embeds arbitrary text messages into the least-significant bits of an
image's pixels, optionally encrypting them first with a password-based,
authenticated cipher. The result looks like an ordinary image, but it carries a
hidden payload that only the intended recipient can read.

> **Status:** pre-1.0 / work in progress. The payload format is versioned and
> stable within a major version, but breaking changes may still happen before
> `1.0.0`.

---

## Features

- **LSB image steganography** — payload bits are embedded into the low bit of
  the R, G and B channels of each pixel, leaving the image visually unchanged.
- **Optional encryption** — pass `--encrypt` to protect the message with
  **XChaCha20Poly1305** (authenticated encryption). Keys are derived from a
  password with **Argon2**, using a random per-message salt and nonce.
- **Tamper detection** — the AEAD auth tag means a wrong password or a modified
  image is detected instead of silently returning garbage.
- **Self-describing payload** — a `RUSTSTEG` magic header, version byte and flag
  bits let the decoder recognise its own files and reject foreign ones.
- **Password hygiene** — passwords are read without echoing to the terminal and
  zeroized from memory after use.
- **Safe output handling** — never overwrites an existing file without asking,
  and validates carrier formats up front.

## Installation

### From crates.io (recommended)

```bash
cargo install ruststeg
```

### Prebuilt binaries

Ready-to-run binaries for Linux and Windows (x86_64 and ARM64) are published on
the [GitHub Releases][releases-url] page, built with
[cargo-dist](https://github.com/axodotdev/cargo-dist). Install the latest
release with the generated installer:

```bash
# Linux / macOS
curl --proto '=https' --tlsv1.2 -sSfL https://github.com/ScalierBullet63/ruststeg/releases/latest/download/ruststeg-x86_64-unknown-linux-gnu-installer.sh | sh

# Windows (PowerShell)
irm https://github.com/ScalierBullet63/ruststeg/releases/latest/download/ruststeg-x86_64-pc-windows-msvc-installer.ps1 | iex
```

### Build from source

```bash
git clone https://github.com/ScalierBullet63/ruststeg.git
cd ruststeg
cargo build --release
# binary at target/release/ruststeg
```

Requires a Rust toolchain for edition 2024 (Rust 1.85 or newer).

## Quick start

```bash
# Hide a plain message. Writes image_steg.png next to the original by default.
ruststeg encode --target-file image.png --msg "meet me at midnight"

# Hide an encrypted message. You will be prompted for a password.
ruststeg encode --target-file image.png --msg "meet me at midnight" --encrypt

# Choose an explicit output path.
ruststeg encode -t image.png -m "secret" -o out.png --encrypt

# Recover the message (you will be prompted for the password if it was encrypted).
ruststeg decode --target-file image_steg.png
```

Example session:

```text
$ ruststeg encode -t photo.png -m "the eagle flies at dawn" --encrypt
Enter the password: ********

$ ruststeg decode -t photo_steg.png
Enter the password: ********
Hidden message: the eagle flies at dawn
```

## Command reference

RustSteg exposes two subcommands. Run `ruststeg --help` or
`ruststeg <subcommand> --help` for the full, always-current list.

### `encode` — hide a message in an image

| Option | Short | Description |
| --- | --- | --- |
| `--target-file <PATH>` | `-t` | Input image to use as the carrier (required) |
| `--msg <TEXT>` | `-m` | Message to hide (required) |
| `--output-path <PATH>` | `-o` | Where to write the stego image. Defaults to `<name>_steg.<ext>` |
| `--encrypt` | | Encrypt the message; prompts for a password |

### `decode` — recover a message from an image

| Option | Short | Description |
| --- | --- | --- |
| `--target-file <PATH>` | `-t` | Image to read the hidden message from (required) |

If the payload was encrypted, `decode` prompts for the password automatically.

## Supported formats

| Carrier | Status |
| --- | --- |
| PNG | ✅ Supported |
| BMP | ✅ Supported |
| Audio / Video | 🚧 Planned |

Use a lossless carrier. Lossy formats (e.g. JPEG) re-encode pixel values and
destroy the embedded bits, so they are intentionally rejected.

## How it works

### Embedding

The message is packed into a binary payload, serialised MSB-first, and each bit
is written into the least-significant bit of a pixel channel (R, then G, then B;
the alpha channel is left untouched). A pixel channel is nudged by ±1 to force
the desired parity, so the change is imperceptible. A `width × height × 3`
carrier can hold up to `width × height × 3` bits; if the payload does not fit,
RustSteg reports `Not enough bits in the image`.

### Payload format

The payload is self-describing and versioned:

| Field | Size | Notes |
| --- | --- | --- |
| Magic | 8 bytes | ASCII `RUSTSTEG` |
| Version | 1 byte | Currently `1` |
| Flags | 1 byte | bit 0 = encrypted |
| Salt | 16 bytes | present only when encrypted |
| Nonce | 24 bytes | present only when encrypted |
| Length | 4 bytes | message length, big-endian `u32` |
| Message | *Length* bytes | plaintext, or ciphertext when encrypted |
| Auth tag | 16 bytes | present only when encrypted |

### Encryption

When `--encrypt` is set:

- A 32-byte key is derived from the password with **Argon2** and a random 16-byte
  salt.
- The message is sealed with **XChaCha20Poly1305** using a random 24-byte nonce.
- The resulting ciphertext and 16-byte auth tag are embedded; the salt and nonce
  travel in the header so the recipient only needs the password.

The password and derived key are zeroized from memory as soon as they are no
longer needed.

## Security notes & limitations

- Steganography hides the *existence* of a message; it is not a substitute for
  secure transport. Treat RustSteg as a learning and privacy tool, not a
  hardened covert channel.
- The password is the only secret when encrypted — a weak password undermines
  the Argon2 + XChaCha20Poly1305 construction.
- LSB embedding is fragile: recompression, resizing, screenshots or format
  conversion will destroy the payload. Keep the original lossless file.
- This project has not been independently audited. See
  [SECURITY.md](SECURITY.md) before reporting a vulnerability.

## Development

```bash
cargo build            # debug build
cargo test             # run tests
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
```

A [`bacon.toml`](bacon.toml) is included for fast, watch-based feedback during
development. Continuous integration builds and tests on Linux and Windows, and
enforces `cargo fmt` and `cargo clippy` on every push and pull request.

Contributions are welcome — please read [CONTRIBUTING.md](CONTRIBUTING.md) for
the branching, commit and CI requirements.

## Release & versioning

Releases are automated with [release-plz](https://release-plz.ieni.dev/) and
published to both crates.io and GitHub Releases. Notable changes are tracked in
[CHANGELOG.md](CHANGELOG.md), which follows
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and
[Semantic Versioning](https://semver.org/).

## License

RustSteg is licensed under the **GNU General Public License v3.0 only**
(`GPL-3.0-only`). See [LICENSE](LICENSE) for details.

<!-- Badges -->
[crates-badge]: https://img.shields.io/crates/v/ruststeg.svg
[crates-url]: https://crates.io/crates/ruststeg
[ci-badge]: https://github.com/ScalierBullet63/ruststeg/actions/workflows/rust.yml/badge.svg
[ci-url]: https://github.com/ScalierBullet63/ruststeg/actions/workflows/rust.yml
[license-badge]: https://img.shields.io/badge/license-GPL--3.0--only-blue.svg
[license-url]: https://github.com/ScalierBullet63/ruststeg/blob/main/LICENSE
[rust-badge]: https://img.shields.io/badge/rust-1.85%2B-orange.svg
[rust-url]: https://www.rust-lang.org/
[releases-url]: https://github.com/ScalierBullet63/ruststeg/releases
