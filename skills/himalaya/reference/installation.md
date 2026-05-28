# Installing himalaya

`himalaya` is a single Rust binary. The fastest install depends on what package manager the host has.

## Verify

After any install, confirm the binary works:

```bash
himalaya --version
```

If this prints a version string, you're done — proceed to [configuration.md](configuration.md).

## Linux

### Arch Linux / Manjaro

```bash
sudo pacman -S himalaya
```

### Debian / Ubuntu

The Debian repos are usually out of date. Two reliable paths:

**Prebuilt binary from GitHub releases:**

```bash
curl -L -o /tmp/himalaya.tar.gz \
  https://github.com/pimalaya/himalaya/releases/latest/download/himalaya.linux.tar.gz
tar -xzf /tmp/himalaya.tar.gz -C /tmp
sudo install /tmp/himalaya /usr/local/bin/himalaya
```

**Or `cargo install`** (if you have Rust):

```bash
cargo install himalaya
```

### Fedora / RHEL

```bash
cargo install himalaya
```

(There's no first-party RPM; Cargo is the simplest path.)

### Nix / NixOS

```bash
nix profile install nixpkgs#himalaya
```

### Any Linux with Cargo

```bash
cargo install himalaya
```

The binary lands in `~/.cargo/bin/himalaya`; make sure that's on your `$PATH`.

## macOS

### Homebrew

```bash
brew install himalaya
```

### MacPorts

```bash
sudo port install himalaya
```

### Cargo

```bash
cargo install himalaya
```

## Windows

```bash
scoop install himalaya
```

…or download a prebuilt `.exe` from the [releases page](https://github.com/pimalaya/himalaya/releases).

## Docker / container hosts (no package manager)

For headless servers or minimal containers, the prebuilt static binary is the path of least resistance:

```bash
ARCH=$(uname -m)  # x86_64 or aarch64
curl -L -o /tmp/himalaya.tar.gz \
  "https://github.com/pimalaya/himalaya/releases/latest/download/himalaya.${ARCH}-unknown-linux-musl.tar.gz"
tar -xzf /tmp/himalaya.tar.gz -C /tmp
sudo install /tmp/himalaya /usr/local/bin/himalaya
```

## Optional companions

These are NOT required but commonly paired with himalaya:

- **`pass`** — standard UNIX password store. Stores IMAP/SMTP passwords encrypted with GPG; `himalaya` reads them via `auth.cmd = "pass show …"`.
- **`gpg`** — required by `pass`.
- **A `$EDITOR`** (`vim`, `nano`, `nvim`, `code -w`, etc.) — for the interactive compose flow.

## Source build

If you need a feature only on `main`:

```bash
git clone https://github.com/pimalaya/himalaya.git
cd himalaya
cargo build --release
sudo install target/release/himalaya /usr/local/bin/himalaya
```

## After installing

Run the interactive wizard once to create an account:

```bash
himalaya account configure
```

…or hand-write a `~/.config/himalaya/config.toml` — see [configuration.md](configuration.md) for layouts per provider.
