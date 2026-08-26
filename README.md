# OpenWrt Packages — SnapRAID & Cosmos Cloud

Prebuilt **[SnapRAID](https://www.snapraid.it/)** and **[Cosmos Cloud](https://cosmos-cloud.io/)**
OpenWrt packages (release 25.12.5), cross-compiled / assembled for **every supported
OpenWrt architecture** and published automatically as an **apk package repository**.

- ⚙️ SnapRAID is built from source with the official OpenWrt SDK (musl libc).
- ☁️ Cosmos Cloud is shipped from its **official prebuilt binaries**
  (statically-linked Go) + web assets — for the architectures the upstream
  publishes (amd64, arm64, 386, armv6, armv7, riscv64).
- 🗂️ apk format (OpenWrt **25.1+**); one directory per architecture.
- 🤖 Auto-built & re-published on push / schedule / manual dispatch.

---

## Add the repo & trust key (OpenWrt 25.1+, apk)

The repository is served from the `gh-pages` branch:
`https://aseracorp.github.io/openwrt-packages/<arch>/packages.adb`

`<arch>` is your device's OpenWrt package architecture (e.g. `aarch64_cortex-a72`,
`x86_64`, `mipsel_24kc`, ...). Find it with `apk arch` or the fw-selector.

**One-time setup** (replace `<arch>` with the real value):

```sh
# 1. Install the repository verification public key (required for trust)
mkdir -p /etc/apk/keys
wget -O /etc/apk/keys/aede713bafd53a86.pub \
  https://raw.githubusercontent.com/aseracorp/openwrt-packages/main/keys/aede713bafd53a86.pub

# 2. Add the repository (URL must end in /packages.adb and match YOUR arch)
echo "https://aseracorp.github.io/openwrt-packages/aarch64_cortex-a72/packages.adb" \
  > /etc/apk/repositories.d/customfeeds.list

# 3. Refresh the index and install
apk update
apk add snapraid cosmoscloud
```

> **Important:** include the trailing `/packages.adb` and use your exact
> architecture. If you point apk at a bare directory it falls back to Alpine's
> `APKINDEX.tar.gz` layout against the wrong `/<arch>/` path and fails.

---

## Packages

### SnapRAID (`snapraid`)
Prebuilt 14.9 for every OpenWrt architecture, includes man pages.

```sh
apk add snapraid
snapraid --version
```

### Cosmos Cloud (`cosmoscloud`)

Prebuilt Cosmos Cloud (statically-linked Go binaries + web assets) for the
architectures that have an official archive:

| OpenWrt arch | Cosmos binary |
|---|---|
| `x86_64` | amd64 |
| `i386_pentium4` / `i386_pentium-mmx` | 386 |
| `aarch64_*` (cortex-a53/a72/a76/generic) | arm64 |
| `arm_cortex-a7` / `-a9` / `-a15` (armv7) | armv7 |
| `arm_arm1176jzf-s_vfp` (RPi Zero) | armv6 |
| `riscv64_generic` | riscv64 |

```sh
apk add cosmoscloud
# service
/etc/init.d/CosmosCloud enable
/etc/init.d/CosmosCloud start
```

Cosmos Cloud stores its config in `/etc/cosmos/` (set via `COSMOS_CONFIG_FOLDER`).
The runtime tree (binaries `cosmos`, `cosmos-launcher`, `nebula`, `restic`, web
assets, GeoLite DB) is installed under `/opt/cosmos/`.

> The mips/mipsel/powerpc/loongarch/etc architectures have **no** upstream
> prebuilt Cosmos Cloud binary, so they ship SnapRAID only (the build matrix
> simply doesn't package cosmoscloud for them).

---

## If the package is unsigned

If `apk` reports `UNTRUSTED signature`, the publishing signing key wasn't
configured (i.e. the `APK_SIGNING_KEY` secret is missing). Install with trust
disabled:

```sh
apk add --allow-untrusted --repository \
  https://aseracorp.github.io/openwrt-packages/<arch>/packages.adb snapraid
```

---

## Build it yourself

Via `feeds.conf`:

```
src-git packages https://github.com/aseracorp/openwrt-packages.git
```

```sh
./scripts/feeds update -a
./scripts/feeds install snapraid cosmoscloud
make menuconfig     # select Network > cosmoscloud, Utilities > snapraid
make package/snapraid/compile package/cosmoscloud/compile
```

> `cosmoscloud` requires the prebuilt release archive to be present at
> `package/cosmoscloud/prebuilt/` before compiling — the CI workflow fetches it
> automatically.

---

## CI / release workflow

`.github/workflows/build.yml`:
- Builds SnapRAID from source and repackages Cosmos Cloud from its prebuilt
  archives, for every supported architecture, with the matching OpenWrt
  **25.12.5** SDK.
- **Storage split (keeps gh-pages tiny):**
  - The large `.apk` binaries are uploaded to a GitHub **Release**
    (`https://github.com/aseracorp/openwrt-packages/releases/download/<tag>/`),
    one asset per package+arch: `<name>_<version>_<arch>.apk`.
  - The per-architecture index `packages.adb` (just a few KB) is generated with
    an **absolute pkgname-spec** pointing at that release, so `apk` fetches the
    packages straight from the release. Only the tiny index + public key are
    committed to `gh-pages` — no package blobs live in the git repo.
- The whole feed is **signed** (apk EC P-256), so installs verify without
  `--allow-untrusted`.
- Cosmos version and the release tag are configurable via `workflow_dispatch`
  (defaults `0.23.0-unstable013` / `feed-v1`).

### Setting up package signing (OpenWrt apk uses an **EC P-256** key, not usign)

```sh
openssl ecparam -name prime256v1 -genkey -noout -out private.key
openssl ec -in private.key -pubout > <KEYID>.pub   # commit as keys/<KEYID>.pub
```
`KEYID` = the first 16 hex characters of the SHA-512 of the SEC1 uncompressed
EC public point (`0x04||X||Y`). Derive it with:
```sh
openssl pkey -pubin -in <KEYID>.pub -pubout -outform DER | \
  tail -c 65 | openssl dgst -sha512 | head -c 16
```
1. Commit `keys/<KEYID>.pub`.
2. Set `KEYID` in the workflow `env` (already configured).
3. Add `base64 private.key` as the repo secret **`APK_SIGNING_KEY`**.

When the secret is present, the SDK signs each index (`apk mkndx --sign`);
otherwise packages are published unsigned (install with `--allow-untrusted`).

---

## License

- SnapRAID: GPL-3.0-or-later (© Andrea Mazzoleni, https://www.snapraid.it/)
- Cosmos Cloud: Apache-2.0 with Commons Clause (© azukaar / Cosmos)
- This OpenWrt feed packaging: GPL-2.0 (OpenWrt standard)