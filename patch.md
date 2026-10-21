# Upstream sync patch

- Generated: 2026-10-21T16:01:38Z
- Previous fork tip: backup-main-before-sync (42fbe32)
- New base: upstream/main (a4d36e9)
- Merge-base: 9e1c09610cdcb073d16559713dee6409d1a08b20
- Upstream commits included: 18
- Local commits kept on top: 2 (CMAF fix a3abe21, private ghcr workflow 4dabe3a)
- Dropped (superseded by wreq): hyper-tls/openssl TLS work (43fe358, 42fe41b, cdaad63, merges 73b01a6/42fbe32)

## Highlights

- RSS: image enclosure in feed items (1377859), index-out-of-bounds guard (726c343)
- HTTP client switch: hyper-rustls -> wreq with device emulation, gzip via wreq, hyper isolation (ff47763..d2d3242)
- Proxy support: added proxy support + media proxying fixes, OS certificates (5bfa7d6, c3acdfa, 35352a8, af002ab)
- CI/build: updated build deps, dockerfiles (20c6965, c881205)
- Tests: moved into mod blocks (e138fbb)

## Commits (oldest first)

### 1377859 2025-11-25 slydetector

feat(rss): add image enclosure to rss feed items


---

### 726c343 2025-11-26 slydetector

handle index out of bounds


---

### ba98178 2025-12-15 Matthew Esposito

Merge pull request #510 from Silvenga/match-ff-ciphers

Avoid 403 by Changing Ciphers List
---

### ff47763 2026-04-02 Mark Lopez

chore: added wreq, updated README.md


---

### 0f66cb1 2026-04-04 Mark Lopez

chore: proof of concept of wreq


---

### e138fbb 2026-04-04 Mark Lopez

refactor: moved all tests into test mod blocks


---

### c005e6f 2026-04-04 Mark Lopez

chore: switch gzip to default wreq handling, enabled device emulation, removed unused crates


---

### d2d3242 2026-04-04 Mark Lopez

chore: further isolated hyper


---

### c881205 2026-04-04 Mark Lopez

chore: updated dockerfiles


---

### 5bfa7d6 2026-04-04 Mark Lopez

feat: added proxy support


---

### 883728f 2026-04-04 Mark Lopez

chore: reduced random emulation list for privacy


---

### 20c6965 2026-04-04 Mark Lopez

ci: updated build dependencies


---

### 35352a8 2026-04-04 Mark Lopez

chore: use os certificates


---

### c3acdfa 2026-04-04 Mark Lopez

chore: fixed proxying of media


---

### af002ab 2026-04-04 Mark Lopez

chore: speculative fix for weird failures


---

### 48306ee 2026-04-24 Matthew Esposito

Merge pull request #544 from Silvenga/switch-to-wreq

Emulate Common TLS Fingerprints + HTTP/SOCK5 Proxy Support
---

### ba591a4 2026-04-24 Matthew Esposito

Merge branch 'main' into rss-feed-images


---

### a4d36e9 2026-04-24 Matthew Esposito

Merge pull request #508 from slydetector/rss-feed-images

feat(rss): add image enclosure to rss feed items
---


## File stat (`9e1c09610cdcb073d16559713dee6409d1a08b20..upstream/main`)

```
 .devcontainer/devcontainer.json        |   5 +-
 .dockerignore                          |   3 +
 .github/workflows/build-artifacts.yaml |   6 +-
 .github/workflows/main-rust.yml        |   2 +-
 Cargo.lock                             | 933 +++++++++++++++++++++++----------
 Cargo.toml                             |  14 +-
 Dockerfile.alpine                      |  17 +-
 Dockerfile.ubuntu                      |   7 +-
 README.md                              |  87 ++-
 src/client.rs                          | 397 +++++++-------
 src/config.rs                          | 128 ++---
 src/main.rs                            |  14 +-
 src/oauth.rs                           | 161 +++---
 src/post.rs                            |   5 +-
 src/search.rs                          |   2 -
 src/subreddit.rs                       | 129 ++++-
 src/user.rs                            |  19 +-
 src/utils.rs                           | 238 ++++-----
 18 files changed, 1324 insertions(+), 843 deletions(-)
```

## Full diff

Regenerate anytime with:
```sh
git diff 9e1c09610cdcb073d16559713dee6409d1a08b20..upstream/main
```

```diff
diff --git a/.devcontainer/devcontainer.json b/.devcontainer/devcontainer.json
index 3ec1ead..512bcf1 100644
--- a/.devcontainer/devcontainer.json
+++ b/.devcontainer/devcontainer.json
@@ -1,6 +1,6 @@
 {
     "name": "Rust",
-    "image": "mcr.microsoft.com/devcontainers/rust:1.0.9-bookworm",
+    "image": "mcr.microsoft.com/devcontainers/rust:dev-trixie",
     "features": {
         "ghcr.io/devcontainers/features/docker-in-docker:2": {}
     },
@@ -10,5 +10,6 @@
             "onAutoForward": "notify"
         }
     },
-    "postCreateCommand": "cargo build"
+    "postCreateCommand": "sudo apt-get update && sudo apt-get install -y git build-essential cmake libclang-dev",
+    "postStartCommand": "cargo build"
 }
diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..fc07e5f
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,3 @@
+target
+.github
+Dockerfile*
\ No newline at end of file
diff --git a/.github/workflows/build-artifacts.yaml b/.github/workflows/build-artifacts.yaml
index 695d1bf..2689b6b 100644
--- a/.github/workflows/build-artifacts.yaml
+++ b/.github/workflows/build-artifacts.yaml
@@ -38,17 +38,17 @@ jobs:
       - if: matrix.target == 'x86_64-unknown-linux-musl'
         run: |
           sudo apt-get update
-          sudo apt-get install -y --no-install-recommends musl-tools
+          sudo apt-get install -y --no-install-recommends musl-tools git cmake perl pkg-config libclang-dev
  
       - if: matrix.target == 'armv7-unknown-linux-musleabihf'
         run: |
           sudo apt update
-          sudo apt install -y gcc-arm-linux-gnueabihf musl-tools
+          sudo apt install -y gcc-arm-linux-gnueabihf musl-tools git cmake perl pkg-config libclang-dev
 
       - if: matrix.target == 'aarch64-unknown-linux-musl'
         run: |
           sudo apt update
-          sudo apt install -y gcc-aarch64-linux-gnu musl-tools
+          sudo apt install -y gcc-aarch64-linux-gnu musl-tools git cmake perl pkg-config libclang-dev
 
       - name: Versions
         id: version
diff --git a/.github/workflows/main-rust.yml b/.github/workflows/main-rust.yml
index f38c01d..823fec0 100644
--- a/.github/workflows/main-rust.yml
+++ b/.github/workflows/main-rust.yml
@@ -31,7 +31,7 @@ jobs:
           toolchain: stable
 
       - name: Install musl-gcc
-        run: sudo apt-get install musl-tools
+        run: sudo apt-get update && sudo apt-get install -y --no-install-recommends musl-tools git cmake perl pkg-config libclang-dev
 
       - name: Install cargo musl target
         run: rustup target add x86_64-unknown-linux-musl
diff --git a/Cargo.lock b/Cargo.lock
index e09e17b..fed126a 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -2,15 +2,6 @@
 # It is not intended for manual editing.
 version = 4
 
-[[package]]
-name = "addr2line"
-version = "0.24.2"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "dfbe277e56a376000877090da837660b4427aad530e3028d44e0bffe4f89a1c1"
-dependencies = [
- "gimli",
-]
-
 [[package]]
 name = "adler2"
 version = "2.0.0"
@@ -25,14 +16,14 @@ checksum = "aae1277d39aeec15cb388266ecc24b11c80469deae6067e17a1a7aa9e5c1f234"
 
 [[package]]
 name = "ahash"
-version = "0.8.11"
+version = "0.8.12"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "e89da841a80418a9b391ebaea17f5c112ffaaa96f621d2c285b5174da76b9011"
+checksum = "5a15f179cd60c4584b8a8c596927aadc462e27f2ca70c04e0071964a73ba7a75"
 dependencies = [
  "cfg-if",
  "once_cell",
  "version_check",
- "zerocopy",
+ "zerocopy 0.8.27",
 ]
 
 [[package]]
@@ -114,6 +105,18 @@ dependencies = [
  "winnow",
 ]
 
+[[package]]
+name = "async-compression"
+version = "0.4.41"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "d0f9ee0f6e02ffd7ad5816e9464499fba7b3effd01123b515c41d1697c43dad1"
+dependencies = [
+ "compression-codecs",
+ "compression-core",
+ "pin-project-lite",
+ "tokio",
+]
+
 [[package]]
 name = "async-recursion"
 version = "1.1.1"
@@ -150,25 +153,16 @@ dependencies = [
 ]
 
 [[package]]
-name = "autocfg"
-version = "1.4.0"
+name = "atomic-waker"
+version = "1.1.2"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "ace50bade8e6234aa140d9a2f552bbee1db4d353f69b8217bc503490fc1a9f26"
+checksum = "1505bd5d3d116872e7271a6d4e16d81d0c8570876c8de68093a09ac269d8aac0"
 
 [[package]]
-name = "backtrace"
-version = "0.3.74"
+name = "autocfg"
+version = "1.4.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "8d82cb332cdfaed17ae235a638438ac4d4839913cc2af585c3c6746e8f8bee1a"
-dependencies = [
- "addr2line",
- "cfg-if",
- "libc",
- "miniz_oxide",
- "object",
- "rustc-demangle",
- "windows-targets",
-]
+checksum = "ace50bade8e6234aa140d9a2f552bbee1db4d353f69b8217bc503490fc1a9f26"
 
 [[package]]
 name = "base2048"
@@ -176,12 +170,6 @@ version = "2.0.2"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "71f4fe417e8cc3bb9b437dfa9290ce92bd2730ba5374719bdfd9147fbc8f17cd"
 
-[[package]]
-name = "base64"
-version = "0.21.7"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "9d297deb1925b89f2ccc13d7635fa0714f12c87adce1c75356b39ca9b7178567"
-
 [[package]]
 name = "base64"
 version = "0.22.1"
@@ -197,6 +185,24 @@ dependencies = [
  "serde",
 ]
 
+[[package]]
+name = "bindgen"
+version = "0.72.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "993776b509cfb49c750f11b8f07a46fa23e0a1386ffc01fb1e7d343efc387895"
+dependencies = [
+ "bitflags",
+ "cexpr",
+ "clang-sys",
+ "itertools",
+ "proc-macro2",
+ "quote",
+ "regex",
+ "rustc-hash",
+ "shlex",
+ "syn",
+]
+
 [[package]]
 name = "bitflags"
 version = "2.9.0"
@@ -212,6 +218,31 @@ dependencies = [
  "generic-array",
 ]
 
+[[package]]
+name = "boring-sys2"
+version = "5.0.0-alpha.13"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "455d79965f5155dcc88a7abce112c3590883889131b799beda10bf9a813ed669"
+dependencies = [
+ "bindgen",
+ "cmake",
+ "fs_extra",
+ "fslock",
+]
+
+[[package]]
+name = "boring2"
+version = "5.0.0-alpha.13"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "183ccc3854411c035410dcdbffafca62084f3a6c33f013c77e83c025d2a08a28"
+dependencies = [
+ "bitflags",
+ "boring-sys2",
+ "foreign-types",
+ "libc",
+ "openssl-macros",
+]
+
 [[package]]
 name = "brotli"
 version = "7.0.0"
@@ -220,14 +251,35 @@ checksum = "cc97b8f16f944bba54f0433f07e30be199b6dc2bd25937444bbad560bcea29bd"
 dependencies = [
  "alloc-no-stdlib",
  "alloc-stdlib",
- "brotli-decompressor",
+ "brotli-decompressor 4.0.3",
+]
+
+[[package]]
+name = "brotli"
+version = "8.0.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "4bd8b9603c7aa97359dbd97ecf258968c95f3adddd6db2f7e7a5bef101c84560"
+dependencies = [
+ "alloc-no-stdlib",
+ "alloc-stdlib",
+ "brotli-decompressor 5.0.0",
 ]
 
 [[package]]
 name = "brotli-decompressor"
-version = "4.0.2"
+version = "4.0.3"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "74fa05ad7d803d413eb8380983b092cbbaf9a85f151b871360e7b00cd7060b37"
+checksum = "a334ef7c9e23abf0ce748e8cd309037da93e606ad52eb372e4ce327a0dcfbdfd"
+dependencies = [
+ "alloc-no-stdlib",
+ "alloc-stdlib",
+]
+
+[[package]]
+name = "brotli-decompressor"
+version = "5.0.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "874bb8112abecc98cbd6d81ea4fa7e94fb9449648c93cc89aa40c81c24d7de03"
 dependencies = [
  "alloc-no-stdlib",
  "alloc-stdlib",
@@ -269,27 +321,28 @@ checksum = "d71b6127be86fdcfddb610f7182ac57211d4b18a3e9c82eb2d17662f2227ad6a"
 
 [[package]]
 name = "cached"
-version = "0.54.0"
+version = "0.59.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "9718806c4a2fe9e8a56fd736f97b340dd10ed1be8ed733ed50449f351dc33cae"
+checksum = "53b6f5d101f0f6322c8646a45b7c581a673e476329040d97565815c2461dd0c4"
 dependencies = [
  "ahash",
  "async-trait",
  "cached_proc_macro",
  "cached_proc_macro_types",
  "futures",
- "hashbrown 0.14.5",
+ "hashbrown 0.16.1",
  "once_cell",
- "thiserror 1.0.69",
+ "parking_lot",
+ "thiserror 2.0.12",
  "tokio",
  "web-time",
 ]
 
 [[package]]
 name = "cached_proc_macro"
-version = "0.23.0"
+version = "0.27.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "2f42a145ed2d10dce2191e1dcf30cfccfea9026660e143662ba5eec4017d5daa"
+checksum = "8ebcf9c75f17a17d55d11afc98e46167d4790a263f428891b8705ab2f793eca3"
 dependencies = [
  "darling",
  "proc-macro2",
@@ -305,13 +358,25 @@ checksum = "ade8366b8bd5ba243f0a58f036cc0ca8a2f069cff1a2351ef1cac6b083e16fc0"
 
 [[package]]
 name = "cc"
-version = "1.2.16"
+version = "1.2.59"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "be714c154be609ec7f5dad223a33bf1482fff90472de28f7362806e6d4832b8c"
+checksum = "b7a4d3ec6524d28a329fc53654bbadc9bdd7b0431f5d65f1a56ffb28a1ee5283"
 dependencies = [
+ "find-msvc-tools",
+ "jobserver",
+ "libc",
  "shlex",
 ]
 
+[[package]]
+name = "cexpr"
+version = "0.6.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "6fac387a98bb7c37292057cffc56d62ecb629900026402633ae9160df93a8766"
+dependencies = [
+ "nom",
+]
+
 [[package]]
 name = "cfg-if"
 version = "1.0.0"
@@ -327,6 +392,17 @@ dependencies = [
  "num-traits",
 ]
 
+[[package]]
+name = "clang-sys"
+version = "1.8.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "0b023947811758c97c59bf9d1c188fd619ad4718dcaa767947df1cadb14f39f4"
+dependencies = [
+ "glob",
+ "libc",
+ "libloading",
+]
+
 [[package]]
 name = "clap"
 version = "4.5.31"
@@ -366,30 +442,43 @@ source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "f46ad14479a25103f283c0f10005961cf086d8dc42205bb44c46ac563475dca6"
 
 [[package]]
-name = "cookie"
-version = "0.18.1"
+name = "cmake"
+version = "0.1.58"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "4ddef33a339a91ea89fb53151bd0a4689cfce27055c291dfa69945475d22c747"
+checksum = "c0f78a02292a74a88ac736019ab962ece0bc380e3f977bf72e376c5d78ff0678"
 dependencies = [
- "time",
- "version_check",
+ "cc",
 ]
 
 [[package]]
-name = "core-foundation"
-version = "0.9.4"
+name = "compression-codecs"
+version = "0.4.37"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "91e195e091a93c46f7102ec7818a2aa394e1e1771c3ab4825963fa03e45afb8f"
+checksum = "eb7b51a7d9c967fc26773061ba86150f19c50c0d65c887cb1fbe295fd16619b7"
 dependencies = [
- "core-foundation-sys",
- "libc",
+ "brotli 8.0.2",
+ "compression-core",
+ "flate2",
+ "memchr",
+ "zstd",
+ "zstd-safe",
 ]
 
 [[package]]
-name = "core-foundation-sys"
-version = "0.8.7"
+name = "compression-core"
+version = "0.4.31"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "773648b94d0e5d620f64f280777445740e61fe701025087ec8b57f45c791888b"
+checksum = "75984efb6ed102a0d42db99afb6c1948f0380d1d91808d5529916e6c08b49d8d"
+
+[[package]]
+name = "cookie"
+version = "0.18.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "4ddef33a339a91ea89fb53151bd0a4689cfce27055c291dfa69945475d22c747"
+dependencies = [
+ "time",
+ "version_check",
+]
 
 [[package]]
 name = "core2"
@@ -545,6 +634,12 @@ version = "0.15.7"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "1aaf95b3e5c8f23aa320147307562d361db0ae0d51242340f558153b4eb2439b"
 
+[[package]]
+name = "either"
+version = "1.15.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "48c757948c5ede0e46177b7add2e67155f70e33c07fea8284df6576da70b3719"
+
 [[package]]
 name = "encoding_rs"
 version = "0.8.35"
@@ -598,17 +693,66 @@ version = "2.3.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "37909eebbb50d72f9059c3b6d82c0463f2ff062c9e95845c43a6c9c0355411be"
 
+[[package]]
+name = "find-msvc-tools"
+version = "0.1.9"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "5baebc0774151f905a1a2cc41989300b1e6fbb29aff0ceffa1064fdd3088d582"
+
+[[package]]
+name = "flate2"
+version = "1.1.9"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "843fba2746e448b37e26a819579957415c8cef339bf08564fe8b7ddbd959573c"
+dependencies = [
+ "crc32fast",
+ "miniz_oxide",
+]
+
 [[package]]
 name = "fnv"
 version = "1.0.7"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "3f9eec918d3f24069decb9af1554cad7c880e2da24a9afd88aca000531ab82c1"
 
+[[package]]
+name = "foldhash"
+version = "0.2.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "77ce24cb58228fbb8aa041425bb1050850ac19177686ea6e0f41a70416f56fdb"
+
+[[package]]
+name = "foreign-types"
+version = "0.5.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "d737d9aa519fb7b749cbc3b962edcf310a8dd1f4b67c91c4f83975dbdd17d965"
+dependencies = [
+ "foreign-types-macros",
+ "foreign-types-shared",
+]
+
+[[package]]
+name = "foreign-types-macros"
+version = "0.2.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "1a5c6c585bc94aaf2c7b51dd4c2ba22680844aba4c687be581871a6f518c5742"
+dependencies = [
+ "proc-macro2",
+ "quote",
+ "syn",
+]
+
+[[package]]
+name = "foreign-types-shared"
+version = "0.3.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "aa9a19cbb55df58761df49b23516a86d432839add4af60fc256da840f66ed35b"
+
 [[package]]
 name = "form_urlencoded"
-version = "1.2.1"
+version = "1.2.2"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "e13624c2627564efccf4934284bdd98cbaa14e79b0b5a141218e507b3a823456"
+checksum = "cb4cb245038516f5f85277875cdaa4f7d2c9a0fa0468de06ed190163b1581fcf"
 dependencies = [
  "percent-encoding",
 ]
@@ -619,6 +763,16 @@ version = "1.3.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "42703706b716c37f96a77aea830392ad231f44c9e9a67872fa5548707e11b11c"
 
+[[package]]
+name = "fslock"
+version = "0.2.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "04412b8935272e3a9bae6f48c7bfff74c2911f60525404edfdd28e49884c3bfb"
+dependencies = [
+ "libc",
+ "winapi",
+]
+
 [[package]]
 name = "futures"
 version = "0.3.31"
@@ -727,10 +881,10 @@ dependencies = [
 ]
 
 [[package]]
-name = "gimli"
-version = "0.31.1"
+name = "glob"
+version = "0.3.3"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "07e28edb80900c19c28f1072f2e8aeca7fa06b23cd4169cefe1af5aa3260783f"
+checksum = "0cc23270f6e1808e30a928bdc84dea0b9b4136a8bc82338574f23baf47bbd280"
 
 [[package]]
 name = "globset"
@@ -747,16 +901,16 @@ dependencies = [
 
 [[package]]
 name = "h2"
-version = "0.3.26"
+version = "0.3.27"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "81fe527a889e1532da5c525686d96d4c2e74cdd345badf8dfef9f6b39dd5f5e8"
+checksum = "0beca50380b1fc32983fc1cb4587bfa4bb9e78fc259aad4a0032d2080309222d"
 dependencies = [
  "bytes",
  "fnv",
  "futures-core",
  "futures-sink",
  "futures-util",
- "http",
+ "http 0.2.12",
  "indexmap",
  "slab",
  "tokio",
@@ -764,6 +918,12 @@ dependencies = [
  "tracing",
 ]
 
+[[package]]
+name = "hashbrown"
+version = "0.13.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "43a3c133739dddd0d2990f9a4bdf8eb4b21ef50e4851ca85ab661199821d510e"
+
 [[package]]
 name = "hashbrown"
 version = "0.14.5"
@@ -776,9 +936,14 @@ dependencies = [
 
 [[package]]
 name = "hashbrown"
-version = "0.15.2"
+version = "0.16.1"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "bf151400ff0baff5465007dd2f3e717f3fe502074ca563069ce3a6629d07b289"
+checksum = "841d1cc9bed7f9236f321df977030373f4a4163ae1a7dbfe1a51a2c1a51d9100"
+dependencies = [
+ "allocator-api2",
+ "equivalent",
+ "foldhash",
+]
 
 [[package]]
 name = "heck"
@@ -809,6 +974,16 @@ dependencies = [
  "itoa",
 ]
 
+[[package]]
+name = "http"
+version = "1.4.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "e3ba2a386d7f85a81f119ad7498ebe444d2e22c2af0b86b069416ace48b3311a"
+dependencies = [
+ "bytes",
+ "itoa",
+]
+
 [[package]]
 name = "http-body"
 version = "0.4.6"
@@ -816,10 +991,53 @@ source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "7ceab25649e9960c0311ea418d17bee82c0dcec1bd053b5f9a66e265a693bed2"
 dependencies = [
  "bytes",
- "http",
+ "http 0.2.12",
  "pin-project-lite",
 ]
 
+[[package]]
+name = "http-body"
+version = "1.0.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "1efedce1fb8e6913f23e0c92de8e62cd5b772a67e7b3946df930a62566c93184"
+dependencies = [
+ "bytes",
+ "http 1.4.0",
+]
+
+[[package]]
+name = "http-body-util"
+version = "0.1.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "b021d93e26becf5dc7e1b75b1bed1fd93124b374ceb73f43d4d4eafec896a64a"
+dependencies = [
+ "bytes",
+ "futures-core",
+ "http 1.4.0",
+ "http-body 1.0.1",
+ "pin-project-lite",
+]
+
+[[package]]
+name = "http2"
+version = "0.5.15"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "c45c6490693ee8a8d0d95fdbdf76fead9fb87548f7894137259a7c6d22821948"
+dependencies = [
+ "atomic-waker",
+ "bytes",
+ "fnv",
+ "futures-core",
+ "futures-sink",
+ "http 1.4.0",
+ "indexmap",
+ "parking_lot",
+ "slab",
+ "smallvec",
+ "tokio",
+ "tokio-util",
+]
+
 [[package]]
 name = "httparse"
 version = "1.10.1"
@@ -849,35 +1067,19 @@ dependencies = [
  "futures-core",
  "futures-util",
  "h2",
- "http",
- "http-body",
+ "http 0.2.12",
+ "http-body 0.4.6",
  "httparse",
  "httpdate",
  "itoa",
  "pin-project-lite",
- "socket2",
+ "socket2 0.5.10",
  "tokio",
  "tower-service",
  "tracing",
  "want",
 ]
 
-[[package]]
-name = "hyper-rustls"
-version = "0.24.2"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "ec3efd23720e2049821a693cbc7e65ea87c72f1c58ff2f9522ff332b1491e590"
-dependencies = [
- "futures-util",
- "http",
- "hyper",
- "log",
- "rustls",
- "rustls-native-certs",
- "tokio",
- "tokio-rustls",
-]
-
 [[package]]
 name = "icu_collections"
 version = "1.5.0"
@@ -1004,9 +1206,9 @@ checksum = "b9e0384b61958566e926dc50660321d12159025e767c18e043daf26b70104c39"
 
 [[package]]
 name = "idna"
-version = "1.0.3"
+version = "1.1.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "686f825264d630750a544639377bae737628043f20d38bbc029e8f29ea968a7e"
+checksum = "3b0875f23caa03898994f6ddc501886a45c7d3d62d04d2d90788d47be1b1e4de"
 dependencies = [
  "idna_adapter",
  "smallvec",
@@ -1025,12 +1227,12 @@ dependencies = [
 
 [[package]]
 name = "indexmap"
-version = "2.7.1"
+version = "2.13.1"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "8c9c992b02b5b4c94ea26e32fe5bccb7aa7d9f390ab5c1221ff895bc7ea8b652"
+checksum = "45a8a2b9cb3e0b0c1803dbb0758ffac5de2f425b23c28f518faabd9d805342ff"
 dependencies = [
  "equivalent",
- "hashbrown 0.15.2",
+ "hashbrown 0.16.1",
 ]
 
 [[package]]
@@ -1042,6 +1244,12 @@ dependencies = [
  "rustversion",
 ]
 
+[[package]]
+name = "ipnet"
+version = "2.12.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "d98f6fed1fde3f8c21bc40a1abb88dd75e67924f9cffc3ef95607bad8017f8e2"
+
 [[package]]
 name = "is-terminal"
 version = "0.4.16"
@@ -1053,12 +1261,30 @@ dependencies = [
  "windows-sys 0.59.0",
 ]
 
+[[package]]
+name = "itertools"
+version = "0.13.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "413ee7dfc52ee1a4949ceeb7dbc8a33f2d6c088194d9f922fb8318faf1f01186"
+dependencies = [
+ "either",
+]
+
 [[package]]
 name = "itoa"
 version = "1.0.15"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "4a5f13b858c8d314ee3e8f639011f7ccefe71f97f96e50151fb991f267928e2c"
 
+[[package]]
+name = "jobserver"
+version = "0.1.32"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "48d1dbcbbeb6a7fec7e059840aa538bd62aaccf972c7346c4d9d2059312853d0"
+dependencies = [
+ "libc",
+]
+
 [[package]]
 name = "js-sys"
 version = "0.3.77"
@@ -1071,9 +1297,9 @@ dependencies = [
 
 [[package]]
 name = "libc"
-version = "0.2.170"
+version = "0.2.184"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "875b3680cb2f8f71bdcf9a30f38d48282f5d3c95cbf9b3fa57269bb5d5c06828"
+checksum = "48f5d2a454e16a5ea0f4ced81bd44e4cfc7bd3a507b61887c99fd3538b28e4af"
 
 [[package]]
 name = "libflate"
@@ -1099,6 +1325,16 @@ dependencies = [
  "rle-decode-fast",
 ]
 
+[[package]]
+name = "libloading"
+version = "0.8.9"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "d7c4b02199fee7c5d21a5ae7d8cfa79a6ef5bb2fc834d6e9058e89c825efdc55"
+dependencies = [
+ "cfg-if",
+ "windows-link",
+]
+
 [[package]]
 name = "linux-raw-sys"
 version = "0.9.2"
@@ -1123,19 +1359,18 @@ checksum = "23fb14cb19457329c82206317a5663005a4d404783dc74f4252769b0d5f42856"
 
 [[package]]
 name = "lock_api"
-version = "0.4.12"
+version = "0.4.14"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "07af8b9cdd281b7915f413fa73f29ebd5d55d0d3f0155584dade1ff18cea1b17"
+checksum = "224399e74b87b5f3557511d98dff8b14089b3dadafcab6bb93eab67d3aace965"
 dependencies = [
- "autocfg",
  "scopeguard",
 ]
 
 [[package]]
 name = "log"
-version = "0.4.26"
+version = "0.4.29"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "30bde2b3dc3671ae49d8e2e9f044c7c005836e7a023ee57cffa25ab82764bb9e"
+checksum = "5e5032e24019045c762d3c0f28f5b6b8bbf38563a65908389bf7978758920897"
 
 [[package]]
 name = "memchr"
@@ -1156,17 +1391,18 @@ source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "8e3e04debbb59698c15bacbb6d93584a8c0ca9cc3213cb423d31f760d8843ce5"
 dependencies = [
  "adler2",
+ "simd-adler32",
 ]
 
 [[package]]
 name = "mio"
-version = "1.0.3"
+version = "1.2.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "2886843bf800fba2e3377cff24abf6379b4c4d5c6681eaf9ea5b0d15090450bd"
+checksum = "50b7e5b27aa02a74bac8c3f23f448f8d87ff11f92d3aac1a6ed369ee08cc56c1"
 dependencies = [
  "libc",
  "wasi 0.11.0+wasi-snapshot-preview1",
- "windows-sys 0.52.0",
+ "windows-sys 0.61.2",
 ]
 
 [[package]]
@@ -1209,15 +1445,6 @@ dependencies = [
  "libc",
 ]
 
-[[package]]
-name = "object"
-version = "0.36.7"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "62948e14d923ea95ea2c7c86c71013138b66525b86bdc08d2dcc262bdb497b87"
-dependencies = [
- "memchr",
-]
-
 [[package]]
 name = "once_cell"
 version = "1.20.3"
@@ -1225,10 +1452,15 @@ source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "945462a4b81e43c4e3ba96bd7b49d834c6f61198356aa858733bc4acf3cbe62e"
 
 [[package]]
-name = "openssl-probe"
-version = "0.1.6"
+name = "openssl-macros"
+version = "0.1.1"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "d05e27ee213611ffe7d6348b942e8f942b37114c00cc03cec254295a4a17852e"
+checksum = "a948666b637a0f465e8564c73e89d4dde00d72d4d473cc972f390fc3dcee7d9c"
+dependencies = [
+ "proc-macro2",
+ "quote",
+ "syn",
+]
 
 [[package]]
 name = "parking"
@@ -1238,9 +1470,9 @@ checksum = "f38d5652c16fde515bb1ecef450ab0f6a219d619a7274976324d5e377f7dceba"
 
 [[package]]
 name = "parking_lot"
-version = "0.12.3"
+version = "0.12.5"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "f1bf18183cf54e8d6059647fc3063646a1801cf30896933ec2311622cc4b9a27"
+checksum = "93857453250e3077bd71ff98b6a65ea6621a19bb0f559a85248955ac12c45a1a"
 dependencies = [
  "lock_api",
  "parking_lot_core",
@@ -1248,22 +1480,22 @@ dependencies = [
 
 [[package]]
 name = "parking_lot_core"
-version = "0.9.10"
+version = "0.9.12"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "1e401f977ab385c9e4e3ab30627d6f26d00e2c73eef317493c4ec6d468726cf8"
+checksum = "2621685985a2ebf1c516881c026032ac7deafcda1a2c9b7850dc81e3dfcb64c1"
 dependencies = [
  "cfg-if",
  "libc",
  "redox_syscall",
  "smallvec",
- "windows-targets",
+ "windows-link",
 ]
 
 [[package]]
 name = "percent-encoding"
-version = "2.3.1"
+version = "2.3.2"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "e3148f5046208a5d56bcfc03053e3ca6334e51da8dfb19b6cdc8b306fae3283e"
+checksum = "9b4f627cb1b25917193a259e49bdad08f671f8d9708acfd5fe0a8c1455d87220"
 
 [[package]]
 name = "pin-project-lite"
@@ -1277,6 +1509,12 @@ version = "0.1.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "8b870d8c151b6f2fb93e84a13146138f05d02ed11c7e7c54f8826aaaf7c9f184"
 
+[[package]]
+name = "pkg-config"
+version = "0.3.32"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "7edddbd0b52d732b21ad9a5fab5c704c14cd949e5e9a1ec5929a24fded1b904c"
+
 [[package]]
 name = "powerfmt"
 version = "0.2.0"
@@ -1289,7 +1527,7 @@ version = "0.2.20"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "77957b295656769bb8ad2b6a6b09d897d94f05c41b069aede1fcdaa675eaea04"
 dependencies = [
- "zerocopy",
+ "zerocopy 0.7.35",
 ]
 
 [[package]]
@@ -1313,9 +1551,9 @@ dependencies = [
 
 [[package]]
 name = "pulldown-cmark"
-version = "0.12.2"
+version = "0.13.3"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "f86ba2052aebccc42cbbb3ed234b8b13ce76f75c3551a303cb2bcffcff12bb14"
+checksum = "7c3a14896dfa883796f1cb410461aef38810ea05f2b2c33c5aded3649095fdad"
 dependencies = [
  "bitflags",
  "memchr",
@@ -1392,9 +1630,9 @@ dependencies = [
  "askama",
  "async-recursion",
  "base2048",
- "base64 0.22.1",
+ "base64",
  "bincode",
- "brotli",
+ "brotli 7.0.0",
  "build_html",
  "cached",
  "chrono",
@@ -1406,7 +1644,6 @@ dependencies = [
  "futures-lite",
  "htmlescape",
  "hyper",
- "hyper-rustls",
  "libflate",
  "lipsum",
  "log",
@@ -1418,7 +1655,6 @@ dependencies = [
  "route-recognizer",
  "rss",
  "rust-embed",
- "rustls",
  "sealed_test",
  "serde",
  "serde_json",
@@ -1431,6 +1667,8 @@ dependencies = [
  "toml",
  "url",
  "uuid",
+ "wreq",
+ "wreq-util",
 ]
 
 [[package]]
@@ -1473,38 +1711,24 @@ checksum = "2b15c43186be67a4fd63bee50d0303afffcef381492ebe2c5d87f324e1b8815c"
 
 [[package]]
 name = "revision"
-version = "0.10.0"
+version = "0.17.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "22f53179a035f881adad8c4d58a2c599c6b4a8325b989c68d178d7a34d1b1e4c"
+checksum = "11c3c8ec8b2be254beb5f8acdd80cdd57b7b5d40988c2ec3d0b7cdb6f7c2829b"
 dependencies = [
  "revision-derive",
 ]
 
 [[package]]
 name = "revision-derive"
-version = "0.10.0"
+version = "0.17.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "5f0ec466e5d8dca9965eb6871879677bef5590cf7525ad96cae14376efb75073"
+checksum = "76be63634a8b1809e663bc0b975d78f6883c0fadbcce9c52e19b9e421f423357"
 dependencies = [
  "proc-macro2",
  "quote",
  "syn",
 ]
 
-[[package]]
-name = "ring"
-version = "0.17.13"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "70ac5d832aa16abd7d1def883a8545280c20a60f523a370aa3a9617c2b8550ee"
-dependencies = [
- "cc",
- "cfg-if",
- "getrandom 0.2.15",
- "libc",
- "untrusted",
- "windows-sys 0.52.0",
-]
-
 [[package]]
 name = "rle-decode-fast"
 version = "1.0.3"
@@ -1564,12 +1788,6 @@ dependencies = [
  "walkdir",
 ]
 
-[[package]]
-name = "rustc-demangle"
-version = "0.1.24"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "719b953e2095829ee67db738b3bfa9fa368c94900df327b3f07fe6e794d2fe1f"
-
 [[package]]
 name = "rustc-hash"
 version = "2.1.1"
@@ -1590,47 +1808,10 @@ dependencies = [
 ]
 
 [[package]]
-name = "rustls"
-version = "0.21.12"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "3f56a14d1f48b391359b22f731fd4bd7e43c97f3c50eee276f3aa09c94784d3e"
-dependencies = [
- "log",
- "ring",
- "rustls-webpki",
- "sct",
-]
-
-[[package]]
-name = "rustls-native-certs"
-version = "0.6.3"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "a9aace74cb666635c918e9c12bc0d348266037aa8eb599b5cba565709a8dff00"
-dependencies = [
- "openssl-probe",
- "rustls-pemfile",
- "schannel",
- "security-framework",
-]
-
-[[package]]
-name = "rustls-pemfile"
-version = "1.0.4"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "1c74cae0a4cf6ccbbf5f359f08efdf8ee7e1dc532573bf0db71968cb56b1448c"
-dependencies = [
- "base64 0.21.7",
-]
-
-[[package]]
-name = "rustls-webpki"
-version = "0.101.7"
+name = "rustls-pki-types"
+version = "1.14.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "8b6275d1ee7a1cd780b64aca7726599a1dbc893b1e64144529e55c3c2f745765"
-dependencies = [
- "ring",
- "untrusted",
-]
+checksum = "be040f8b0a225e40375822a563fa9524378b9d63112f53e19ffff34df5d33fdd"
 
 [[package]]
 name = "rustversion"
@@ -1666,12 +1847,14 @@ dependencies = [
 ]
 
 [[package]]
-name = "schannel"
-version = "0.1.27"
+name = "schnellru"
+version = "0.2.4"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "1f29ebaa345f945cec9fbbc532eb307f0fdad8161f281b6369539c8d84876b3d"
+checksum = "356285bbf17bea63d9e52e96bd18f039672ac92b55b8cb997d6162a2a37d1649"
 dependencies = [
- "windows-sys 0.59.0",
+ "ahash",
+ "cfg-if",
+ "hashbrown 0.13.2",
 ]
 
 [[package]]
@@ -1680,16 +1863,6 @@ version = "1.2.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "94143f37725109f92c262ed2cf5e59bce7498c01bcc1502d7b9afe439a4e9f49"
 
-[[package]]
-name = "sct"
-version = "0.7.1"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "da046153aa2352493d6cb7da4b6e5c0c057d8a1d0a9aa8560baffdd945acd414"
-dependencies = [
- "ring",
- "untrusted",
-]
-
 [[package]]
 name = "sealed_test"
 version = "1.1.0"
@@ -1713,42 +1886,29 @@ dependencies = [
 ]
 
 [[package]]
-name = "security-framework"
-version = "2.11.1"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "897b2245f0b511c87893af39b033e5ca9cce68824c4d7e7630b5a1d339658d02"
-dependencies = [
- "bitflags",
- "core-foundation",
- "core-foundation-sys",
- "libc",
- "security-framework-sys",
-]
-
-[[package]]
-name = "security-framework-sys"
-version = "2.14.0"
+name = "serde"
+version = "1.0.228"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "49db231d56a190491cb4aeda9527f1ad45345af50b0851622a7adb8c03b01c32"
+checksum = "9a8e94ea7f378bd32cbbd37198a4a91436180c5bb472411e48b5ec2e2124ae9e"
 dependencies = [
- "core-foundation-sys",
- "libc",
+ "serde_core",
+ "serde_derive",
 ]
 
 [[package]]
-name = "serde"
-version = "1.0.218"
+name = "serde_core"
+version = "1.0.228"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "e8dfc9d19bdbf6d17e22319da49161d5d0108e4188e8b680aef6299eed22df60"
+checksum = "41d385c7d4ca58e59fc732af25c3983b67ac852c1a25000afe1175de458b67ad"
 dependencies = [
  "serde_derive",
 ]
 
 [[package]]
 name = "serde_derive"
-version = "1.0.218"
+version = "1.0.228"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "f09503e191f4e797cb8aac08e9a4a4695c5edf6a2e70e376d961ddd5c969f82b"
+checksum = "d540f220d3187173da220f885ab66608367b6574e925011a9353e4badda91d79"
 dependencies = [
  "proc-macro2",
  "quote",
@@ -1819,9 +1979,9 @@ dependencies = [
 
 [[package]]
 name = "serde_spanned"
-version = "0.6.8"
+version = "0.6.9"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "87607cb1398ed59d48732e575a4c28a7a8ebf2454b964fe3f224f2afc07909e1"
+checksum = "bf41e0cfaf7226dca15e8197172c295a782857fcb97fad1808a166870dee75a3"
 dependencies = [
  "serde",
 ]
@@ -1877,6 +2037,12 @@ dependencies = [
  "libc",
 ]
 
+[[package]]
+name = "simd-adler32"
+version = "0.3.9"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "703d5c7ef118737c72f1af64ad2f6f8c5e1921f818cdcb97b8fe6fc69bf66214"
+
 [[package]]
 name = "slab"
 version = "0.4.9"
@@ -1888,20 +2054,30 @@ dependencies = [
 
 [[package]]
 name = "smallvec"
-version = "1.14.0"
+version = "1.15.1"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "7fcf8323ef1faaee30a44a340193b1ac6814fd9b7b4e88e9d4519a3e4abe1cfd"
+checksum = "67b1b7a3b5fe4f1376887184045fcf45c69e92af734b7aaddc05fb777b6fbd03"
 
 [[package]]
 name = "socket2"
-version = "0.5.8"
+version = "0.5.10"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "c970269d99b64e60ec3bd6ad27270092a5394c4e309314b18ae3fe575695fbe8"
+checksum = "e22376abed350d73dd1cd119b57ffccad95b4e585a7cda43e286245ce23c0678"
 dependencies = [
  "libc",
  "windows-sys 0.52.0",
 ]
 
+[[package]]
+name = "socket2"
+version = "0.6.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "3a766e1110788c36f4fa1c2b71b387a7815aa65f88ce0229841826633d93723e"
+dependencies = [
+ "libc",
+ "windows-sys 0.61.2",
+]
+
 [[package]]
 name = "stable_deref_trait"
 version = "1.2.0"
@@ -1925,6 +2101,15 @@ dependencies = [
  "unicode-ident",
 ]
 
+[[package]]
+name = "sync_wrapper"
+version = "1.0.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "0bf256ce5efdfa370213c1dabab5935a12e49f2c58d15e9eac2870d3b4f27263"
+dependencies = [
+ "futures-core",
+]
+
 [[package]]
 name = "synstructure"
 version = "0.13.1"
@@ -2053,27 +2238,36 @@ dependencies = [
 
 [[package]]
 name = "tokio"
-version = "1.44.2"
+version = "1.51.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "e6b88822cbe49de4185e3a4cbf8321dd487cf5fe0c5c65695fef6346371e9c48"
+checksum = "2bd1c4c0fc4a7ab90fc15ef6daaa3ec3b893f004f915f2392557ed23237820cd"
 dependencies = [
- "backtrace",
  "bytes",
  "libc",
  "mio",
  "parking_lot",
  "pin-project-lite",
  "signal-hook-registry",
- "socket2",
+ "socket2 0.6.3",
  "tokio-macros",
- "windows-sys 0.52.0",
+ "windows-sys 0.61.2",
+]
+
+[[package]]
+name = "tokio-boring2"
+version = "5.0.0-alpha.13"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "0f81df1210d791f31d72d840de8fbd80b9c3cb324956523048b1413e2bd55756"
+dependencies = [
+ "boring2",
+ "tokio",
 ]
 
 [[package]]
 name = "tokio-macros"
-version = "2.5.0"
+version = "2.7.0"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "6e06d43f1345a3bcd39f6a56dbb7dcab2ba47e68e8ac134855e7e2bdbaf8cab8"
+checksum = "385a6cb71ab9ab790c5fe8d67f1645e6c450a7ce006a33de03daa956cf70a496"
 dependencies = [
  "proc-macro2",
  "quote",
@@ -2081,20 +2275,22 @@ dependencies = [
 ]
 
 [[package]]
-name = "tokio-rustls"
-version = "0.24.1"
+name = "tokio-socks"
+version = "0.5.2"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "c28327cf380ac148141087fbfb9de9d7bd4e84ab5d2c28fbc911d753de8a7081"
+checksum = "0d4770b8024672c1101b3f6733eab95b18007dbe0847a8afe341fcf79e06043f"
 dependencies = [
- "rustls",
+ "either",
+ "futures-util",
+ "thiserror 1.0.69",
  "tokio",
 ]
 
 [[package]]
 name = "tokio-util"
-version = "0.7.13"
+version = "0.7.18"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "d7fcaa8d55a2bdd6b83ace262b016eca0d79ee02818c5c1bcdf0305114081078"
+checksum = "9ae9cec805b01e8fc3fd2fe289f89149a9b66dd16786abd8b19cfa7b48cb0098"
 dependencies = [
  "bytes",
  "futures-core",
@@ -2105,9 +2301,9 @@ dependencies = [
 
 [[package]]
 name = "toml"
-version = "0.8.20"
+version = "0.8.23"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "cd87a5cdd6ffab733b2f74bc4fd7ee5fff6634124999ac278c35fc78c6120148"
+checksum = "dc1beb996b9d83529a9e75c17a1686767d148d70663143c7854d8b4a09ced362"
 dependencies = [
  "serde",
  "serde_spanned",
@@ -2117,26 +2313,74 @@ dependencies = [
 
 [[package]]
 name = "toml_datetime"
-version = "0.6.8"
+version = "0.6.11"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "0dd7358ecb8fc2f8d014bf86f6f638ce72ba252a2c3a2572f2a795f1d23efb41"
+checksum = "22cddaf88f4fbc13c51aebbf5f8eceb5c7c5a9da2ac40a13519eb5b0a0e8f11c"
 dependencies = [
  "serde",
 ]
 
 [[package]]
 name = "toml_edit"
-version = "0.22.24"
+version = "0.22.27"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "17b4795ff5edd201c7cd6dca065ae59972ce77d1b80fa0a84d94950ece7d1474"
+checksum = "41fe8c660ae4257887cf66394862d21dbca4a6ddd26f04a3560410406a2f819a"
 dependencies = [
  "indexmap",
  "serde",
  "serde_spanned",
  "toml_datetime",
+ "toml_write",
  "winnow",
 ]
 
+[[package]]
+name = "toml_write"
+version = "0.1.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "5d99f8c9a7727884afe522e9bd5edbfc91a3312b36a77b5fb8926e4c31a41801"
+
+[[package]]
+name = "tower"
+version = "0.5.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "ebe5ef63511595f1344e2d5cfa636d973292adc0eec1f0ad45fae9f0851ab1d4"
+dependencies = [
+ "futures-core",
+ "futures-util",
+ "pin-project-lite",
+ "sync_wrapper",
+ "tokio",
+ "tower-layer",
+ "tower-service",
+]
+
+[[package]]
+name = "tower-http"
+version = "0.6.8"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "d4e6559d53cc268e5031cd8429d05415bc4cb4aefc4aa5d6cc35fbf5b924a1f8"
+dependencies = [
+ "async-compression",
+ "bitflags",
+ "bytes",
+ "futures-core",
+ "http 1.4.0",
+ "http-body 1.0.1",
+ "http-body-util",
+ "pin-project-lite",
+ "tokio",
+ "tokio-util",
+ "tower-layer",
+ "tower-service",
+]
+
+[[package]]
+name = "tower-layer"
+version = "0.3.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "121c2a6cda46980bb0fcd1647ffaf6cd3fc79a013de288782836f6df9c48780e"
+
 [[package]]
 name = "tower-service"
 version = "0.3.3"
@@ -2168,6 +2412,26 @@ version = "0.2.5"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "e421abadd41a4225275504ea4d6566923418b7f05506fbc9c0fe86ba7396114b"
 
+[[package]]
+name = "typed-builder"
+version = "0.23.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "31aa81521b70f94402501d848ccc0ecaa8f93c8eb6999eb9747e72287757ffda"
+dependencies = [
+ "typed-builder-macro",
+]
+
+[[package]]
+name = "typed-builder-macro"
+version = "0.23.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "076a02dc54dd46795c2e9c8282ed40bcfb1e22747e955de9389a1de28190fb26"
+dependencies = [
+ "proc-macro2",
+ "quote",
+ "syn",
+]
+
 [[package]]
 name = "typenum"
 version = "1.18.0"
@@ -2192,21 +2456,16 @@ version = "0.2.11"
 source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "673aac59facbab8a9007c7f6108d11f63b603f7cabff99fabf650fea5c32b861"
 
-[[package]]
-name = "untrusted"
-version = "0.9.0"
-source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "8ecb6da28b8a351d773b68d5825ac39017e680750f980f3a1a85cd8dd28a47c1"
-
 [[package]]
 name = "url"
-version = "2.5.4"
+version = "2.5.8"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "32f8b686cadd1473f4bd0117a5d28d36b1ade384ea9b5069a1c40aefed7fda60"
+checksum = "ff67a8a4397373c3ef660812acab3268222035010ab8680ec4215f38ba3d0eed"
 dependencies = [
  "form_urlencoded",
  "idna",
  "percent-encoding",
+ "serde",
 ]
 
 [[package]]
@@ -2346,6 +2605,31 @@ dependencies = [
  "wasm-bindgen",
 ]
 
+[[package]]
+name = "webpki-root-certs"
+version = "1.0.6"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "804f18a4ac2676ffb4e8b5b5fa9ae38af06df08162314f96a68d2a363e21a8ca"
+dependencies = [
+ "rustls-pki-types",
+]
+
+[[package]]
+name = "winapi"
+version = "0.3.9"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "5c839a674fcd7a98952e593242ea400abe93992746761e38641405d28b00f419"
+dependencies = [
+ "winapi-i686-pc-windows-gnu",
+ "winapi-x86_64-pc-windows-gnu",
+]
+
+[[package]]
+name = "winapi-i686-pc-windows-gnu"
+version = "0.4.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "ac3b87c63620426dd9b991e5ce0329eff545bccbbb34f3be09ff6fb6ab51b7b6"
+
 [[package]]
 name = "winapi-util"
 version = "0.1.9"
@@ -2355,6 +2639,18 @@ dependencies = [
  "windows-sys 0.59.0",
 ]
 
+[[package]]
+name = "winapi-x86_64-pc-windows-gnu"
+version = "0.4.0"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "712e227841d057c1ee1cd2fb22fa7e5a5461ae8e48fa2ca79ec42cfc1931183f"
+
+[[package]]
+name = "windows-link"
+version = "0.2.1"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "f0805222e57f7521d6a62e36fa9163bc891acd422f971defe97d64e70d0a4fe5"
+
 [[package]]
 name = "windows-sys"
 version = "0.52.0"
@@ -2373,6 +2669,15 @@ dependencies = [
  "windows-targets",
 ]
 
+[[package]]
+name = "windows-sys"
+version = "0.61.2"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "ae137229bcbd6cdf0f7b80a31df61766145077ddf49416a728b02cb3921ff3fc"
+dependencies = [
+ "windows-link",
+]
+
 [[package]]
 name = "windows-targets"
 version = "0.52.6"
@@ -2439,9 +2744,9 @@ checksum = "589f6da84c646204747d1270a2a5661ea66ed1cced2631d546fdfb155959f9ec"
 
 [[package]]
 name = "winnow"
-version = "0.7.3"
+version = "0.7.15"
 source = "registry+https://github.com/rust-lang/crates.io-index"
-checksum = "0e7f4ea97f6f78012141bcdb6a216b2609f0979ada50b20ca5b52dde2eac2bb1"
+checksum = "df79d97927682d2fd8adb29682d1140b343be4ac0f08fd68b7765d9c059d3945"
 dependencies = [
  "memchr",
 ]
@@ -2455,6 +2760,56 @@ dependencies = [
  "bitflags",
 ]
 
+[[package]]
+name = "wreq"
+version = "6.0.0-rc.28"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "f79937f6c4df65b3f6f78715b9de2977afe9ee3b3436483c7949a24511e25935"
+dependencies = [
+ "ahash",
+ "boring2",
+ "brotli 8.0.2",
+ "bytes",
+ "flate2",
+ "futures-channel",
+ "futures-util",
+ "http 1.4.0",
+ "http-body 1.0.1",
+ "http-body-util",
+ "http2",
+ "httparse",
+ "ipnet",
+ "libc",
+ "percent-encoding",
+ "pin-project-lite",
+ "schnellru",
+ "serde",
+ "serde_json",
+ "smallvec",
+ "socket2 0.6.3",
+ "sync_wrapper",
+ "tokio",
+ "tokio-boring2",
+ "tokio-socks",
+ "tokio-util",
+ "tower",
+ "tower-http",
+ "url",
+ "want",
+ "webpki-root-certs",
+ "zstd",
+]
+
+[[package]]
+name = "wreq-util"
+version = "3.0.0-rc.10"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "6c6bbe24d28beb9ceb58b514bd6a613c759d3b706f768b9d2950d5d35b543c04"
+dependencies = [
+ "typed-builder",
+ "wreq",
+]
+
 [[package]]
 name = "write16"
 version = "1.0.0"
@@ -2498,7 +2853,16 @@ source = "registry+https://github.com/rust-lang/crates.io-index"
 checksum = "1b9b4fd18abc82b8136838da5d50bae7bdea537c574d8dc1a34ed098d6c166f0"
 dependencies = [
  "byteorder",
- "zerocopy-derive",
+ "zerocopy-derive 0.7.35",
+]
+
+[[package]]
+name = "zerocopy"
+version = "0.8.27"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "0894878a5fa3edfd6da3f88c4805f4c8558e2b996227a3d864f47fe11e38282c"
+dependencies = [
+ "zerocopy-derive 0.8.27",
 ]
 
 [[package]]
@@ -2512,6 +2876,17 @@ dependencies = [
  "syn",
 ]
 
+[[package]]
+name = "zerocopy-derive"
+version = "0.8.27"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "88d2b8d9c68ad2b9e4340d7832716a4d21a22a1154777ad56ea55c51a9cf3831"
+dependencies = [
+ "proc-macro2",
+ "quote",
+ "syn",
+]
+
 [[package]]
 name = "zerofrom"
 version = "0.1.6"
@@ -2554,3 +2929,31 @@ dependencies = [
  "quote",
  "syn",
 ]
+
+[[package]]
+name = "zstd"
+version = "0.13.3"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "e91ee311a569c327171651566e07972200e76fcfe2242a4fa446149a3881c08a"
+dependencies = [
+ "zstd-safe",
+]
+
+[[package]]
+name = "zstd-safe"
+version = "7.2.4"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "8f49c4d5f0abb602a93fb8736af2a4f4dd9512e36f7f570d66e65ff867ed3b9d"
+dependencies = [
+ "zstd-sys",
+]
+
+[[package]]
+name = "zstd-sys"
+version = "2.0.16+zstd.1.5.7"
+source = "registry+https://github.com/rust-lang/crates.io-index"
+checksum = "91e19ebc2adc8f83e43039e79776e3fda8ca919132d68a1fed6a5faca2683748"
+dependencies = [
+ "cc",
+ "pkg-config",
+]
diff --git a/Cargo.toml b/Cargo.toml
index a133b3a..b47bd05 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -17,7 +17,7 @@ askama = { version = "0.14.0", default-features = false, features = [
     "std",
     "derive",
 ] }
-cached = { version = "0.54.0", features = ["async"] }
+cached = { version = "0.59.0", features = ["async"] }
 clap = { version = "4.4.11", default-features = false, features = [
     "std",
     "env",
@@ -46,21 +46,21 @@ fastrand = "2.0.1"
 log = "0.4.20"
 pretty_env_logger = "0.5.0"
 dotenvy = "0.15.7"
-rss = "2.0.7"
+rss = "2.0.12"
 arc-swap = "1.7.1"
 serde_json_path = "0.7.1"
 async-recursion = "1.1.1"
-pulldown-cmark = { version = "0.12.0", features = ["simd", "html"], default-features = false }
-hyper-rustls = { version = "0.24.2", features = [ "http2" ] }
+pulldown-cmark = { version = "0.13.3", features = ["simd", "html"], default-features = false }
 tegen = "0.1.4"
 serde_urlencoded = "0.7.1"
-chrono = { version = "0.4.39", default-features = false, features = [ "std" ] }
+chrono = { version = "0.4.39", default-features = false, features = ["std"] }
 htmlescape = "0.3.1"
 bincode = "1.3.3"
 base2048 = "2.0.2"
-revision = "0.10.0"
+revision = "0.17.0"
 fake_user_agent = "0.2.2"
-rustls = "0.21.12"
+wreq = { version = "6.0.0-rc.28", features = ["brotli", "gzip", "deflate", "zstd", "json", "stream", "socks"] }
+wreq-util = { version = "3.0.0-rc.10" }
 
 [dev-dependencies]
 lipsum = "0.9.0"
diff --git a/Dockerfile.alpine b/Dockerfile.alpine
index 468cbc3..94b7087 100644
--- a/Dockerfile.alpine
+++ b/Dockerfile.alpine
@@ -1,24 +1,29 @@
 # supported versions here: https://hub.docker.com/_/rust
-ARG ALPINE_VERSION=3.20
+ARG RUST_BUILDER_VERSION=slim-bookworm
+ARG ALPINE_VERSION=3.23
 
 ########################
 ## builder image
 ########################
-FROM rust:alpine${ALPINE_VERSION} AS builder
+FROM ghcr.io/rust-cross/rust-musl-cross:x86_64-musl AS builder
 
-RUN apk add --no-cache musl-dev
+RUN apt-get update \
+    && apt-get -y install git cmake perl pkg-config libclang-dev \
+    && rm -rf /var/lib/apt/lists/*
+
+RUN rustup target add x86_64-unknown-linux-musl
 
 WORKDIR /redlib
 
 # download (most) dependencies in their own layer
 COPY Cargo.lock Cargo.toml ./
 RUN mkdir src && echo "fn main() { panic!(\"why am i running?\") }" > src/main.rs
-RUN cargo build --release --locked --bin redlib
+RUN cargo build --release --locked --bin redlib --target x86_64-unknown-linux-musl
 RUN rm ./src/main.rs && rmdir ./src
 
 # copy the source and build the redlib binary
 COPY . ./
-RUN cargo build --release --locked --bin redlib
+RUN cargo build --release --locked --bin redlib --target x86_64-unknown-linux-musl
 RUN echo "finished building redlib!"
 
 ########################
@@ -27,7 +32,7 @@ RUN echo "finished building redlib!"
 FROM alpine:${ALPINE_VERSION} AS release
 
 # Import redlib binary from builder
-COPY --from=builder /redlib/target/release/redlib /usr/local/bin/redlib
+COPY --from=builder /redlib/target/x86_64-unknown-linux-musl/release/redlib /usr/local/bin/redlib
 
 # Add non-root user for running redlib
 RUN adduser --home /nonexistent --no-create-home --disabled-password redlib
diff --git a/Dockerfile.ubuntu b/Dockerfile.ubuntu
index b6edfd5..c929de7 100644
--- a/Dockerfile.ubuntu
+++ b/Dockerfile.ubuntu
@@ -7,6 +7,10 @@ ARG UBUNTU_RELEASE_VERSION=noble
 ########################
 FROM rust:${RUST_BUILDER_VERSION} AS builder
 
+RUN apt-get update \
+    && apt-get -y install git build-essential cmake libclang-dev \
+    && rm -rf /var/lib/apt/lists/*
+
 WORKDIR /redlib
 
 # download (most) dependencies in their own layer
@@ -25,9 +29,6 @@ RUN echo "finished building redlib!"
 ########################
 FROM ubuntu:${UBUNTU_RELEASE_VERSION} AS release
 
-# Install ca-certificates
-RUN apt-get update && apt-get install -y ca-certificates
-
 # Import redlib binary from builder
 COPY --from=builder /redlib/target/release/redlib /usr/local/bin/redlib
 
diff --git a/README.md b/README.md
index 666ff6d..7ea145c 100644
--- a/README.md
+++ b/README.md
@@ -111,7 +111,7 @@ Last tested on January 12, 2024.
 Results from Google PageSpeed Insights ([Redlib Report](https://pagespeed.web.dev/report?url=https%3A%2F%2Fredlib.matthew.science%2F), [Reddit Report](https://pagespeed.web.dev/report?url=https://www.reddit.com)).
 
 | Performance metric  | Redlib   | Reddit    |
-| ------------------- | -------- | --------- |
+|---------------------|----------|-----------|
 | Speed Index         | 0.6s     | 1.9s      |
 | Performance Score   | 100%     | 64%       |
 | Time to Interactive | **2.8s** | **12.4s** |
@@ -409,36 +409,77 @@ Redlib supports the following command line flags:
 Assign a default value for each instance-specific setting by passing environment variables to Redlib in the format `REDLIB_{X}`. Replace `{X}` with the setting name (see list below) in capital letters.
 
 | Name                      | Possible values | Default value          | Description                                                                                               |
-| ------------------------- | --------------- | ----------------       | --------------------------------------------------------------------------------------------------------- |
+|---------------------------|-----------------|------------------------|-----------------------------------------------------------------------------------------------------------|
 | `SFW_ONLY`                | `["on", "off"]` | `off`                  | Enables SFW-only mode for the instance, i.e. all NSFW content is filtered.                                |
 | `BANNER`                  | String          | (empty)                | Allows the server to set a banner to be displayed. Currently this is displayed on the instance info page. |
 | `ROBOTS_DISABLE_INDEXING` | `["on", "off"]` | `off`                  | Disables indexing of the instance by search engines.                                                      |
 | `PUSHSHIFT_FRONTEND`      | String          | `undelete.pullpush.io` | Allows the server to set the Pushshift frontend to be used with "removed" links.                          |
 | `PORT`                    | Integer 0-65535 | `8080`                 | The **internal** port Redlib listens on.                                                                  |
 | `ENABLE_RSS`              | `["on", "off"]` | `off`                  | Enables RSS feed generation.                                                                              |
-| `FULL_URL`                | String          | (empty)                | Allows for proper URLs (for now, only needed by RSS)
+| `FULL_URL`                | String          | (empty)                | Allows for proper URLs (for now, only needed by RSS)                                                      |
+
 ## Default user settings
 
 Assign a default value for each user-modifiable setting by passing environment variables to Redlib in the format `REDLIB_DEFAULT_{Y}`. Replace `{Y}` with the setting name (see list below) in capital letters.
 
-| Name                                | Possible values                                                                                                                    | Default value |
-| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------- |
+| Name                                | Possible values                                                                                                                                                                                                                 | Default value |
+|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------|
 | `THEME`                             | `["system", "light", "dark", "black", "dracula", "nord", "laserwave", "violet", "gold", "rosebox", "gruvboxdark", "gruvboxlight", "tokyoNight", "icebergDark", "doomone", "libredditBlack", "libredditDark", "libredditLight"]` | `system`      |
-| `FRONT_PAGE`                        | `["default", "popular", "all"]`                                                                                                    | `default`     |
-| `LAYOUT`                            | `["card", "clean", "compact"]`                                                                                                     | `card`        |
-| `WIDE`                              | `["on", "off"]`                                                                                                                    | `off`         |
-| `POST_SORT`                         | `["hot", "new", "top", "rising", "controversial"]`                                                                                 | `hot`         |
-| `COMMENT_SORT`                      | `["confidence", "top", "new", "controversial", "old"]`                                                                             | `confidence`  |
-| `BLUR_SPOILER`                      | `["on", "off"]`                                                                                                                    | `off`         |
-| `SHOW_NSFW`                         | `["on", "off"]`                                                                                                                    | `off`         |
-| `BLUR_NSFW`                         | `["on", "off"]`                                                                                                                    | `off`         |
-| `USE_HLS`                           | `["on", "off"]`                                                                                                                    | `off`         |
-| `HIDE_HLS_NOTIFICATION`             | `["on", "off"]`                                                                                                                    | `off`         |
-| `AUTOPLAY_VIDEOS`                   | `["on", "off"]`                                                                                                                    | `off`         |
-| `SUBSCRIPTIONS`                     | `+`-delimited list of subreddits (`sub1+sub2+sub3+...`)                                                                            | _(none)_      |
-| `HIDE_AWARDS`                       | `["on", "off"]`                                                                                                                    | `off`         |
-| `DISABLE_VISIT_REDDIT_CONFIRMATION` | `["on", "off"]`                                                                                                                    | `off`         |
-| `HIDE_SCORE`                        | `["on", "off"]`                                                                                                                    | `off`         |
-| `HIDE_SIDEBAR_AND_SUMMARY`          | `["on", "off"]`                                                                                                                    | `off`         |
-| `FIXED_NAVBAR`                      | `["on", "off"]`                                                                                                                    | `on`          |
-| `REMOVE_DEFAULT_FEEDS`              | `["on", "off"]`                                                                                                                    | `off`         |
\ No newline at end of file
+| `FRONT_PAGE`                        | `["default", "popular", "all"]`                                                                                                                                                                                                 | `default`     |
+| `LAYOUT`                            | `["card", "clean", "compact"]`                                                                                                                                                                                                  | `card`        |
+| `WIDE`                              | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `POST_SORT`                         | `["hot", "new", "top", "rising", "controversial"]`                                                                                                                                                                              | `hot`         |
+| `COMMENT_SORT`                      | `["confidence", "top", "new", "controversial", "old"]`                                                                                                                                                                          | `confidence`  |
+| `BLUR_SPOILER`                      | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `SHOW_NSFW`                         | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `BLUR_NSFW`                         | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `USE_HLS`                           | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `HIDE_HLS_NOTIFICATION`             | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `AUTOPLAY_VIDEOS`                   | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `SUBSCRIPTIONS`                     | `+`-delimited list of subreddits (`sub1+sub2+sub3+...`)                                                                                                                                                                         | _(none)_      |
+| `HIDE_AWARDS`                       | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `DISABLE_VISIT_REDDIT_CONFIRMATION` | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `HIDE_SCORE`                        | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `HIDE_SIDEBAR_AND_SUMMARY`          | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+| `FIXED_NAVBAR`                      | `["on", "off"]`                                                                                                                                                                                                                 | `on`          |
+| `REMOVE_DEFAULT_FEEDS`              | `["on", "off"]`                                                                                                                                                                                                                 | `off`         |
+
+## Forward Proxies
+
+Redlib [supports](https://docs.rs/wreq/latest/wreq/#proxies) proxy usage using the standard `HTTP_PROXY` and
+`HTTPS_PROXY` environment variables. Use `ALL_PROXY` to set both at the same time (which you want to do).
+
+- `http://` is the scheme for http proxy
+- `https://` is the scheme for https proxy
+- `socks4://` is the scheme for socks4 proxy
+- `socks4a://` is the scheme for socks4a proxy
+- `socks5://` is the scheme for socks5 proxy
+- `socks5h://` is the scheme for socks5h proxy
+
+## Security
+
+This project uses [BoringSSL](https://boringssl.googlesource.com/boringssl/), built from source with patches from
+the [wreq](https://github.com/0x676e67/wreq) project. Certificates are validated against the embedded trust store from Mozilla.
+
+## Building
+
+Since Redlib uses [`boring-sys2`](https://crates.io/crates/boring-sys2), to build Redlib you will need to build
+BoringSSL from source.
+
+### Linux/MacOS
+
+Refer to the [boringssl](https://github.com/google/boringssl/blob/main/BUILDING.md) documentation for dependencies.
+
+### Windows
+
+Install MSVC, which you likely already have for Rust.
+
+```pwsh
+# Make sure to update your PATH, some of the installers don't do that by default (hense -i, interactive mode).
+winget install -i Kitware.CMake
+winget install -i NASM.NASM
+winget install -i LLVM.LLVM
+
+# For tests.
+winget install GoLang.Go
+```
diff --git a/src/client.rs b/src/client.rs
index 126dcc1..5fafa7d 100644
--- a/src/client.rs
+++ b/src/client.rs
@@ -1,25 +1,22 @@
+use crate::dbg_msg;
+use crate::oauth::{force_refresh_token, token_daemon, Oauth, OauthBackendImpl};
+use crate::server::RequestExt;
+use crate::utils::{format_url, Post};
 use arc_swap::ArcSwap;
 use cached::proc_macro::cached;
 use futures_lite::future::block_on;
 use futures_lite::{future::Boxed, FutureExt};
-use hyper::client::HttpConnector;
-use hyper::header::HeaderValue;
-use hyper::{body, body::Buf, header, Body, Client, Method, Request, Response, Uri};
-use hyper_rustls::{ConfigBuilderExt, HttpsConnector};
-use libflate::gzip;
-use log::{error, trace, warn};
+use hyper::{body::Buf, header, Body, Request as HyperRequest, Response as HyperResponse};
+use log::{error, info, trace, warn};
 use percent_encoding::{percent_encode, CONTROLS};
 use serde_json::Value;
-
+use std::result::Result;
 use std::sync::atomic::Ordering;
 use std::sync::atomic::{AtomicBool, AtomicU16};
 use std::sync::LazyLock;
-use std::{io, result::Result};
-
-use crate::dbg_msg;
-use crate::oauth::{force_refresh_token, token_daemon, Oauth, OauthBackendImpl};
-use crate::server::RequestExt;
-use crate::utils::{format_url, Post};
+use wreq::redirect::Policy;
+use wreq::{header as wreq_header, Client as WreqClient, EmulationFactory, Method, Response as WreqResponse};
+use wreq_util::{Emulation, EmulationOS, EmulationOption};
 
 const REDDIT_URL_BASE: &str = "https://oauth.reddit.com";
 const REDDIT_URL_BASE_HOST: &str = "oauth.reddit.com";
@@ -30,38 +27,7 @@ const REDDIT_SHORT_URL_BASE_HOST: &str = "redd.it";
 const ALTERNATIVE_REDDIT_URL_BASE: &str = "https://www.reddit.com";
 const ALTERNATIVE_REDDIT_URL_BASE_HOST: &str = "www.reddit.com";
 
-pub static HTTPS_CONNECTOR: LazyLock<HttpsConnector<HttpConnector>> = LazyLock::new(|| {
-	hyper_rustls::HttpsConnectorBuilder::new()
-		.with_tls_config(
-			rustls::ClientConfig::builder()
-				// These are the Firefox 145.0 cipher suite,
-				// minus the suites missing forward-secrecy support,
-				// in the same order.
-				// https://github.com/redlib-org/redlib/issues/446#issuecomment-3609306592
-				.with_cipher_suites(&[
-					rustls::cipher_suite::TLS13_AES_256_GCM_SHA384,
-					rustls::cipher_suite::TLS13_AES_128_GCM_SHA256,
-					rustls::cipher_suite::TLS13_CHACHA20_POLY1305_SHA256,
-					rustls::cipher_suite::TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
-					rustls::cipher_suite::TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
-					rustls::cipher_suite::TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
-					rustls::cipher_suite::TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
-					rustls::cipher_suite::TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
-					rustls::cipher_suite::TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
-				])
-				// .with_safe_default_cipher_suites()
-				.with_safe_default_kx_groups()
-				.with_safe_default_protocol_versions()
-				.unwrap()
-				.with_native_roots()
-				.with_no_client_auth(),
-		)
-		.https_only()
-		.enable_http2()
-		.build()
-});
-
-pub static CLIENT: LazyLock<Client<HttpsConnector<HttpConnector>>> = LazyLock::new(|| Client::builder().build::<_, Body>(HTTPS_CONNECTOR.clone()));
+pub static CLIENT: LazyLock<WreqClient> = LazyLock::new(build_client);
 
 pub static OAUTH_CLIENT: LazyLock<ArcSwap<Oauth>> = LazyLock::new(|| {
 	let client = block_on(Oauth::new());
@@ -78,6 +44,28 @@ const URL_PAIRS: [(&str, &str); 2] = [
 	(REDDIT_SHORT_URL_BASE, REDDIT_SHORT_URL_BASE_HOST),
 ];
 
+pub fn build_client() -> WreqClient {
+	// Keeping this list short to aid in privacy.
+	// The more emulations, the more unique a fingerprint each instance has.
+	// But some emulations should increase evasiveness.
+	let emulation = [Emulation::Chrome145, Emulation::Firefox147];
+	let emulation_os = [EmulationOS::Android, EmulationOS::Windows];
+
+	let rand = fastrand::usize(..);
+	let emulation = EmulationOption::builder()
+		.emulation(emulation[rand % emulation.len()])
+		.emulation_os(emulation_os[rand % emulation_os.len()])
+		.build()
+		.emulation();
+
+	info!("Building Wreq client with random emulation {:?}", emulation);
+	WreqClient::builder()
+		.emulation(emulation)
+		.redirect(Policy::none())
+		.build()
+		.expect("Should always be able to build a client")
+}
+
 /// Gets the canonical path for a resource on Reddit. This is accomplished by
 /// making a `HEAD` request to Reddit at the path given in `path`.
 ///
@@ -114,14 +102,14 @@ pub async fn canonical_path(path: String, tries: i8) -> Result<Option<String>, S
 
 	let res = res.ok_or_else(|| "Unable to make HEAD request to Reddit.".to_string())?;
 	let status = res.status().as_u16();
-	let policy_error = res.headers().get(header::RETRY_AFTER).is_some();
+	let policy_error = res.headers().get(wreq_header::RETRY_AFTER).is_some();
 
 	match status {
 		// If Reddit responds with a 2xx, then the path is already canonical.
 		200..=299 => Ok(Some(path)),
 
 		// If Reddit responds with a 301, then the path is redirected.
-		301 => match res.headers().get(header::LOCATION) {
+		301 => match res.headers().get(wreq_header::LOCATION) {
 			Some(val) => {
 				let Ok(original) = val.to_str() else {
 					return Err("Unable to decode Location header.".to_string());
@@ -159,13 +147,13 @@ pub async fn canonical_path(path: String, tries: i8) -> Result<Option<String>, S
 		_ => Ok(
 			res
 				.headers()
-				.get(header::LOCATION)
+				.get(wreq_header::LOCATION)
 				.map(|val| percent_encode(val.as_bytes(), CONTROLS).to_string().trim_start_matches(REDDIT_URL_BASE).to_string()),
 		),
 	}
 }
 
-pub async fn proxy(req: Request<Body>, format: &str) -> Result<Response<Body>, String> {
+pub async fn proxy(req: HyperRequest<Body>, format: &str) -> Result<HyperResponse<Body>, String> {
 	let mut url = format!("{format}?{}", req.uri().query().unwrap_or_default());
 
 	// For each parameter in request
@@ -174,22 +162,15 @@ pub async fn proxy(req: Request<Body>, format: &str) -> Result<Response<Body>, S
 		url = url.replace(&format!("{{{name}}}"), value);
 	}
 
-	stream(&url, &req).await
-}
-
-async fn stream(url: &str, req: &Request<Body>) -> Result<Response<Body>, String> {
 	// First parameter is target URL (mandatory).
-	let parsed_uri = url.parse::<Uri>().map_err(|_| "Couldn't parse URL".to_string())?;
+	let wreq_uri = wreq::Uri::try_from(url).map_err(|_| "Couldn't parse URL".to_string())?;
 
-	// Build the hyper client from the HTTPS connector.
-	let client: &LazyLock<Client<_, Body>> = &CLIENT;
-
-	let mut builder = Request::get(parsed_uri);
+	let mut builder = CLIENT.get(wreq_uri);
 
 	// Copy useful headers from original request
 	for &key in &["Range", "If-Modified-Since", "Cache-Control"] {
 		if let Some(value) = req.headers().get(key) {
-			builder = builder.header(key, value);
+			builder = builder.header(key, value.as_bytes());
 		}
 	}
 
@@ -199,13 +180,16 @@ async fn stream(url: &str, req: &Request<Body>) -> Result<Response<Body>, String
 		builder = builder.header("User-Agent", client.user_agent());
 	}
 
-	let stream_request = builder.body(Body::empty()).map_err(|_| "Couldn't build empty body in stream".to_string())?;
+	// This is needed or Reddit will redirect us to a /media landing page that just renders the image.
+	builder = builder.header(wreq_header::ACCEPT, "*/*");
 
-	client
-		.request(stream_request)
+	builder
+		.send()
 		.await
 		.map(|mut res| {
-			let mut rm = |key: &str| res.headers_mut().remove(key);
+			let headers = res.headers_mut();
+
+			let mut rm = |key: &str| headers.remove(key);
 
 			rm("access-control-expose-headers");
 			rm("server");
@@ -220,19 +204,19 @@ async fn stream(url: &str, req: &Request<Body>) -> Result<Response<Body>, String
 			rm("Nel");
 			rm("Report-To");
 
-			res
+			res.into_hyper_response()
 		})
 		.map_err(|e| e.to_string())
 }
 
 /// Makes a GET request to Reddit at `path`. By default, this will honor HTTP
 /// 3xx codes Reddit returns and will automatically redirect.
-fn reddit_get(path: String, quarantine: bool) -> Boxed<Result<Response<Body>, String>> {
+fn reddit_get(path: String, quarantine: bool) -> Boxed<Result<WreqResponse, String>> {
 	request(&Method::GET, path, true, quarantine, REDDIT_URL_BASE, REDDIT_URL_BASE_HOST)
 }
 
 /// Makes a HEAD request to Reddit at `path, using the short URL base. This will not follow redirects.
-fn reddit_short_head(path: String, quarantine: bool, base_path: &'static str, host: &'static str) -> Boxed<Result<Response<Body>, String>> {
+fn reddit_short_head(path: String, quarantine: bool, base_path: &'static str, host: &'static str) -> Boxed<Result<WreqResponse, String>> {
 	request(&Method::HEAD, path, false, quarantine, base_path, host)
 }
 
@@ -245,18 +229,12 @@ fn reddit_short_head(path: String, quarantine: bool, base_path: &'static str, ho
 /// Makes a request to Reddit. If `redirect` is `true`, `request_with_redirect`
 /// will recurse on the URL that Reddit provides in the Location HTTP header
 /// in its response.
-fn request(method: &'static Method, path: String, redirect: bool, quarantine: bool, base_path: &'static str, host: &'static str) -> Boxed<Result<Response<Body>, String>> {
+fn request(method: &'static Method, path: String, redirect: bool, quarantine: bool, base_path: &'static str, host: &'static str) -> Boxed<Result<WreqResponse, String>> {
 	// Build Reddit URL from path.
 	let url = format!("{base_path}{path}");
 
-	// Construct the hyper client from the HTTPS connector.
-	let client: &LazyLock<Client<_, Body>> = &CLIENT;
-
-	// Build request to Reddit. When making a GET, request gzip compression.
-	// (Reddit doesn't do brotli yet.)
 	let mut headers: Vec<(String, String)> = vec![
 		("Host".into(), host.into()),
-		("Accept-Encoding".into(), if method == Method::GET { "gzip".into() } else { "identity".into() }),
 		(
 			"Cookie".into(),
 			if quarantine {
@@ -277,114 +255,64 @@ fn request(method: &'static Method, path: String, redirect: bool, quarantine: bo
 	// shuffle headers: https://github.com/redlib-org/redlib/issues/324
 	fastrand::shuffle(&mut headers);
 
-	let mut builder = Request::builder().method(method).uri(&url);
+	let mut builder = CLIENT.request(method.clone(), &url);
 
 	for (key, value) in headers {
 		builder = builder.header(key, value);
 	}
 
-	let builder = builder.body(Body::empty());
-
 	async move {
-		match builder {
-			Ok(req) => match client.request(req).await {
-				Ok(mut response) => {
-					// Reddit may respond with a 3xx. Decide whether or not to
-					// redirect based on caller params.
-					if response.status().is_redirection() {
-						if !redirect {
-							return Ok(response);
-						};
-						let location_header = response.headers().get(header::LOCATION);
-						if location_header == Some(&HeaderValue::from_static(ALTERNATIVE_REDDIT_URL_BASE)) {
-							return Err("Reddit response was invalid".to_string());
-						}
-						return request(
-							method,
-							location_header
-								.map(|val| {
-									// We need to make adjustments to the URI
-									// we get back from Reddit. Namely, we
-									// must:
-									//
-									//     1. Remove the authority (e.g.
-									//     https://www.reddit.com) that may be
-									//     present, so that we recurse on the
-									//     path (and query parameters) as
-									//     required.
-									//
-									//     2. Percent-encode the path.
-									let new_path = percent_encode(val.as_bytes(), CONTROLS)
-										.to_string()
-										.trim_start_matches(REDDIT_URL_BASE)
-										.trim_start_matches(ALTERNATIVE_REDDIT_URL_BASE)
-										.to_string();
-									format!("{new_path}{}raw_json=1", if new_path.contains('?') { "&" } else { "?" })
-								})
-								.unwrap_or_default()
-								.to_string(),
-							true,
-							quarantine,
-							base_path,
-							host,
-						)
-						.await;
+		match builder.send().await {
+			Ok(response) => {
+				// Reddit may respond with a 3xx. Decide whether or not to
+				// redirect based on caller params.
+				if response.status().is_redirection() {
+					if !redirect {
+						return Ok(response);
 					};
-
-					match response.headers().get(header::CONTENT_ENCODING) {
-						// Content not compressed.
-						None => Ok(response),
-
-						// Content encoded (hopefully with gzip).
-						Some(hdr) => {
-							match hdr.to_str() {
-								Ok(val) => match val {
-									"gzip" => {}
-									"identity" => return Ok(response),
-									_ => return Err("Reddit response was encoded with an unsupported compressor".to_string()),
-								},
-								Err(_) => return Err("Reddit response was invalid".to_string()),
-							}
-
-							// We get here if the body is gzip-compressed.
-
-							// The body must be something that implements
-							// std::io::Read, hence the conversion to
-							// bytes::buf::Buf and then transformation into a
-							// Reader.
-							let mut decompressed: Vec<u8>;
-							{
-								let mut aggregated_body = match body::aggregate(response.body_mut()).await {
-									Ok(b) => b.reader(),
-									Err(e) => return Err(e.to_string()),
-								};
-
-								let mut decoder = match gzip::Decoder::new(&mut aggregated_body) {
-									Ok(decoder) => decoder,
-									Err(e) => return Err(e.to_string()),
-								};
-
-								decompressed = Vec::<u8>::new();
-								if let Err(e) = io::copy(&mut decoder, &mut decompressed) {
-									return Err(e.to_string());
-								};
-							}
-
-							response.headers_mut().remove(header::CONTENT_ENCODING);
-							response.headers_mut().insert(header::CONTENT_LENGTH, decompressed.len().into());
-							*(response.body_mut()) = Body::from(decompressed);
-
-							Ok(response)
-						}
+					let location_header = response.headers().get(wreq::header::LOCATION);
+					if location_header.and_then(|h| h.to_str().ok()) == Some(ALTERNATIVE_REDDIT_URL_BASE) {
+						return Err("Reddit response was invalid".to_string());
 					}
-				}
-				Err(e) => {
-					dbg_msg!("{method} {REDDIT_URL_BASE}{path}: {}", e);
+					return request(
+						method,
+						location_header
+							.map(|val| {
+								// We need to make adjustments to the URI
+								// we get back from Reddit. Namely, we
+								// must:
+								//
+								//     1. Remove the authority (e.g.
+								//     https://www.reddit.com) that may be
+								//     present, so that we recurse on the
+								//     path (and query parameters) as
+								//     required.
+								//
+								//     2. Percent-encode the path.
+								let new_path = percent_encode(val.as_bytes(), CONTROLS)
+									.to_string()
+									.trim_start_matches(REDDIT_URL_BASE)
+									.trim_start_matches(ALTERNATIVE_REDDIT_URL_BASE)
+									.to_string();
+								format!("{new_path}{}raw_json=1", if new_path.contains('?') { "&" } else { "?" })
+							})
+							.unwrap_or_default()
+							.to_string(),
+						true,
+						quarantine,
+						base_path,
+						host,
+					)
+					.await;
+				};
 
-					Err(e.to_string())
-				}
-			},
-			Err(_) => Err("Post url contains non-ASCII characters".to_string()),
+				Ok(response)
+			}
+			Err(e) => {
+				dbg_msg!("{method} {REDDIT_URL_BASE}{path}: {}", e);
+
+				Err(e.to_string())
+			}
 		}
 	}
 	.boxed()
@@ -434,7 +362,7 @@ pub async fn json(path: String, quarantine: bool) -> Result<Value, String> {
 			};
 
 			// asynchronously aggregate the chunks of the body
-			match hyper::body::aggregate(response).await {
+			match hyper::body::aggregate(response.into_hyper_response()).await {
 				Ok(body) => {
 					let has_remaining = body.has_remaining();
 
@@ -547,61 +475,92 @@ pub async fn rate_limit_check() -> Result<(), String> {
 	Ok(())
 }
 
-#[cfg(test)]
-use {crate::config::get_setting, sealed_test::prelude::*};
-
-#[tokio::test(flavor = "multi_thread")]
-async fn test_rate_limit_check() {
-	rate_limit_check().await.unwrap();
+trait IntoHyperResponse {
+	fn into_hyper_response(self) -> HyperResponse<Body>;
 }
 
-#[test]
-#[sealed_test(env = [("REDLIB_DEFAULT_SUBSCRIPTIONS", "rust")])]
-fn test_default_subscriptions() {
-	tokio::runtime::Builder::new_multi_thread().enable_all().build().unwrap().block_on(async {
-		let subscriptions = get_setting("REDLIB_DEFAULT_SUBSCRIPTIONS");
-		assert!(subscriptions.is_some());
+impl IntoHyperResponse for WreqResponse {
+	fn into_hyper_response(self) -> HyperResponse<Body> {
+		let status = self.status();
+		let version = self.version();
+
+		let mut builder = HyperResponse::builder().status(status.as_u16()).version(match version {
+			wreq::Version::HTTP_09 => hyper::Version::HTTP_09,
+			wreq::Version::HTTP_10 => hyper::Version::HTTP_10,
+			wreq::Version::HTTP_11 => hyper::Version::HTTP_11,
+			wreq::Version::HTTP_2 => hyper::Version::HTTP_2,
+			wreq::Version::HTTP_3 => hyper::Version::HTTP_3,
+			_ => hyper::Version::HTTP_11,
+		});
+
+		for (name, value) in self.headers() {
+			builder = builder.header(
+				header::HeaderName::from_bytes(name.as_str().as_bytes()).unwrap(),
+				header::HeaderValue::from_bytes(value.as_bytes()).unwrap(),
+			);
+		}
 
-		// check rate limit
-		rate_limit_check().await.unwrap();
-	});
+		builder.body(Body::wrap_stream(self.bytes_stream())).unwrap()
+	}
 }
 
 #[cfg(test)]
-const POPULAR_URL: &str = "/r/popular/hot.json?&raw_json=1&geo_filter=GLOBAL";
+mod tests {
+	use super::*;
+	use {crate::config::get_setting, sealed_test::prelude::*};
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_localization_popular() {
-	let val = json(POPULAR_URL.to_string(), false).await.unwrap();
-	assert_eq!("GLOBAL", val["data"]["geo_filter"].as_str().unwrap());
-}
+	const POPULAR_URL: &str = "/r/popular/hot.json?&raw_json=1&geo_filter=GLOBAL";
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_obfuscated_share_link() {
-	let share_link = "/r/rust/s/kPgq8WNHRK".into();
-	// Correct link without share parameters
-	let canonical_link = "/r/rust/comments/18t5968/why_use_tuple_struct_over_standard_struct/kfbqlbc/".into();
-	assert_eq!(canonical_path(share_link, 3).await, Ok(Some(canonical_link)));
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_rate_limit_check() {
+		rate_limit_check().await.unwrap();
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_private_sub() {
-	let link = json("/r/suicide/about.json?raw_json=1".into(), true).await;
-	assert!(link.is_err());
-	assert_eq!(link, Err("private".into()));
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_DEFAULT_SUBSCRIPTIONS", "rust")])]
+	fn test_default_subscriptions() {
+		tokio::runtime::Builder::new_multi_thread().enable_all().build().unwrap().block_on(async {
+			let subscriptions = get_setting("REDLIB_DEFAULT_SUBSCRIPTIONS");
+			assert!(subscriptions.is_some());
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_banned_sub() {
-	let link = json("/r/aaa/about.json?raw_json=1".into(), true).await;
-	assert!(link.is_err());
-	assert_eq!(link, Err("banned".into()));
-}
+			// check rate limit
+			rate_limit_check().await.unwrap();
+		});
+	}
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_localization_popular() {
+		let val = json(POPULAR_URL.to_string(), false).await.unwrap();
+		assert_eq!("GLOBAL", val["data"]["geo_filter"].as_str().unwrap());
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_gated_sub() {
-	// quarantine to false to specifically catch when we _don't_ catch it
-	let link = json("/r/drugs/about.json?raw_json=1".into(), false).await;
-	assert!(link.is_err());
-	assert_eq!(link, Err("gated".into()));
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_obfuscated_share_link() {
+		let share_link = "/r/rust/s/kPgq8WNHRK".into();
+		// Correct link without share parameters
+		let canonical_link = "/r/rust/comments/18t5968/why_use_tuple_struct_over_standard_struct/kfbqlbc/".into();
+		assert_eq!(canonical_path(share_link, 3).await, Ok(Some(canonical_link)));
+	}
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_private_sub() {
+		let link = json("/r/suicide/about.json?raw_json=1".into(), true).await;
+		assert!(link.is_err());
+		assert_eq!(link, Err("private".into()));
+	}
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_banned_sub() {
+		let link = json("/r/aaa/about.json?raw_json=1".into(), true).await;
+		assert!(link.is_err());
+		assert_eq!(link, Err("banned".into()));
+	}
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_gated_sub() {
+		// quarantine to false to specifically catch when we _don't_ catch it
+		let link = json("/r/drugs/about.json?raw_json=1".into(), false).await;
+		assert!(link.is_err());
+		assert_eq!(link, Err("gated".into()));
+	}
 }
diff --git a/src/config.rs b/src/config.rs
index b1a7cdb..8f92c9b 100644
--- a/src/config.rs
+++ b/src/config.rs
@@ -196,75 +196,79 @@ pub fn get_setting(name: &str) -> Option<String> {
 }
 
 #[cfg(test)]
-use {sealed_test::prelude::*, std::fs::write};
+mod tests {
+	use super::*;
+	use {sealed_test::prelude::*, std::fs::write};
+
+	#[test]
+	fn test_deserialize() {
+		// Must handle empty input
+		let result = toml::from_str::<Config>("");
+		assert!(result.is_ok(), "Error: {}", result.unwrap_err());
+	}
 
-#[test]
-fn test_deserialize() {
-	// Must handle empty input
-	let result = toml::from_str::<Config>("");
-	assert!(result.is_ok(), "Error: {}", result.unwrap_err());
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_SFW_ONLY", "on")])]
+	fn test_env_var() {
+		assert!(crate::utils::sfw_only())
+	}
 
-#[test]
-#[sealed_test(env = [("REDLIB_SFW_ONLY", "on")])]
-fn test_env_var() {
-	assert!(crate::utils::sfw_only())
-}
+	#[test]
+	#[sealed_test]
+	fn test_config() {
+		let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
+		write("redlib.toml", config_to_write).unwrap();
+		assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("best".into()));
+	}
 
-#[test]
-#[sealed_test]
-fn test_config() {
-	let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
-	write("redlib.toml", config_to_write).unwrap();
-	assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("best".into()));
-}
+	#[test]
+	#[sealed_test]
+	fn test_config_legacy() {
+		let config_to_write = r#"LIBREDDIT_DEFAULT_COMMENT_SORT = "best""#;
+		write("libreddit.toml", config_to_write).unwrap();
+		assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("best".into()));
+	}
 
-#[test]
-#[sealed_test]
-fn test_config_legacy() {
-	let config_to_write = r#"LIBREDDIT_DEFAULT_COMMENT_SORT = "best""#;
-	write("libreddit.toml", config_to_write).unwrap();
-	assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("best".into()));
-}
+	#[test]
+	#[sealed_test(env = [("LIBREDDIT_SFW_ONLY", "on")])]
+	fn test_env_var_legacy() {
+		assert!(crate::utils::sfw_only())
+	}
 
-#[test]
-#[sealed_test(env = [("LIBREDDIT_SFW_ONLY", "on")])]
-fn test_env_var_legacy() {
-	assert!(crate::utils::sfw_only())
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_DEFAULT_COMMENT_SORT", "top")])]
+	fn test_env_config_precedence() {
+		let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
+		write("redlib.toml", config_to_write).unwrap();
+		assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("top".into()))
+	}
 
-#[test]
-#[sealed_test(env = [("REDLIB_DEFAULT_COMMENT_SORT", "top")])]
-fn test_env_config_precedence() {
-	let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
-	write("redlib.toml", config_to_write).unwrap();
-	assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("top".into()))
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_DEFAULT_COMMENT_SORT", "top")])]
+	fn test_alt_env_config_precedence() {
+		let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
+		write("redlib.toml", config_to_write).unwrap();
+		assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("top".into()))
+	}
 
-#[test]
-#[sealed_test(env = [("REDLIB_DEFAULT_COMMENT_SORT", "top")])]
-fn test_alt_env_config_precedence() {
-	let config_to_write = r#"REDLIB_DEFAULT_COMMENT_SORT = "best""#;
-	write("redlib.toml", config_to_write).unwrap();
-	assert_eq!(get_setting("REDLIB_DEFAULT_COMMENT_SORT"), Some("top".into()))
-}
-#[test]
-#[sealed_test(env = [("REDLIB_DEFAULT_SUBSCRIPTIONS", "news+bestof")])]
-fn test_default_subscriptions() {
-	assert_eq!(get_setting("REDLIB_DEFAULT_SUBSCRIPTIONS"), Some("news+bestof".into()));
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_DEFAULT_SUBSCRIPTIONS", "news+bestof")])]
+	fn test_default_subscriptions() {
+		assert_eq!(get_setting("REDLIB_DEFAULT_SUBSCRIPTIONS"), Some("news+bestof".into()));
+	}
 
-#[test]
-#[sealed_test(env = [("REDLIB_DEFAULT_FILTERS", "news+bestof")])]
-fn test_default_filters() {
-	assert_eq!(get_setting("REDLIB_DEFAULT_FILTERS"), Some("news+bestof".into()));
-}
+	#[test]
+	#[sealed_test(env = [("REDLIB_DEFAULT_FILTERS", "news+bestof")])]
+	fn test_default_filters() {
+		assert_eq!(get_setting("REDLIB_DEFAULT_FILTERS"), Some("news+bestof".into()));
+	}
 
-#[test]
-#[sealed_test]
-fn test_pushshift() {
-	let config_to_write = r#"REDLIB_PUSHSHIFT_FRONTEND = "https://api.pushshift.io""#;
-	write("redlib.toml", config_to_write).unwrap();
-	assert!(get_setting("REDLIB_PUSHSHIFT_FRONTEND").is_some());
-	assert_eq!(get_setting("REDLIB_PUSHSHIFT_FRONTEND"), Some("https://api.pushshift.io".into()));
+	#[test]
+	#[sealed_test]
+	fn test_pushshift() {
+		let config_to_write = r#"REDLIB_PUSHSHIFT_FRONTEND = "https://api.pushshift.io""#;
+		write("redlib.toml", config_to_write).unwrap();
+		assert!(get_setting("REDLIB_PUSHSHIFT_FRONTEND").is_some());
+		assert_eq!(get_setting("REDLIB_PUSHSHIFT_FRONTEND"), Some("https://api.pushshift.io".into()));
+	}
 }
diff --git a/src/main.rs b/src/main.rs
index dc57818..399c0c9 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -4,11 +4,9 @@
 
 use cached::proc_macro::cached;
 use clap::{Arg, ArgAction, Command};
-use std::str::FromStr;
 use std::sync::LazyLock;
 
 use futures_lite::FutureExt;
-use hyper::Uri;
 use hyper::{header::HeaderValue, Body, Request, Response};
 use log::{info, warn};
 use redlib::client::{canonical_path, proxy, rate_limit_check, CLIENT};
@@ -433,11 +431,9 @@ pub async fn proxy_commit_info() -> Result<Response<Body>, String> {
 
 #[cached(time = 600)]
 async fn fetch_commit_info() -> String {
-	let uri = Uri::from_str("https://github.com/redlib-org/redlib/commits/main.atom").expect("Invalid URI");
+	let url = "https://github.com/redlib-org/redlib/commits/main.atom";
 
-	let resp: Body = CLIENT.get(uri).await.expect("Failed to request GitHub").into_body();
-
-	hyper::body::to_bytes(resp).await.expect("Failed to read body").iter().copied().map(|x| x as char).collect()
+	CLIENT.get(url).send().await.expect("Failed to request GitHub").text().await.expect("Failed to read body")
 }
 
 pub async fn proxy_instances() -> Result<Response<Body>, String> {
@@ -452,9 +448,7 @@ pub async fn proxy_instances() -> Result<Response<Body>, String> {
 
 #[cached(time = 600)]
 async fn fetch_instances() -> String {
-	let uri = Uri::from_str("https://raw.githubusercontent.com/redlib-org/redlib-instances/refs/heads/main/instances.json").expect("Invalid URI");
-
-	let resp: Body = CLIENT.get(uri).await.expect("Failed to request GitHub").into_body();
+	let url = "https://raw.githubusercontent.com/redlib-org/redlib-instances/refs/heads/main/instances.json";
 
-	hyper::body::to_bytes(resp).await.expect("Failed to read body").iter().copied().map(|x| x as char).collect()
+	CLIENT.get(url).send().await.expect("Failed to request GitHub").text().await.expect("Failed to read body")
 }
diff --git a/src/oauth.rs b/src/oauth.rs
index 0b4f7fe..166e29f 100644
--- a/src/oauth.rs
+++ b/src/oauth.rs
@@ -1,13 +1,11 @@
-use std::{collections::HashMap, sync::atomic::Ordering, time::Duration};
-
 use crate::{
 	client::{CLIENT, OAUTH_CLIENT, OAUTH_IS_ROLLING_OVER, OAUTH_RATELIMIT_REMAINING},
 	oauth_resources::ANDROID_APP_VERSION_LIST,
 };
 use base64::{engine::general_purpose, Engine as _};
-use hyper::{client, Body, Method, Request};
 use log::{error, info, trace, warn};
 use serde_json::json;
+use std::{collections::HashMap, sync::atomic::Ordering, time::Duration};
 use tegen::tegen::TextGenerator;
 use tokio::time::{error::Elapsed, timeout};
 
@@ -88,7 +86,7 @@ impl Oauth {
 					error!(
 						"[⛔] Failed to create OAuth client: {}. Retrying in 5 seconds...",
 						match e {
-							AuthError::Hyper(error) => error.to_string(),
+							AuthError::Wreq(error) => error.to_string(),
 							AuthError::SerdeDeserialize(error) => error.to_string(),
 							AuthError::Field((value, error)) => format!("{error}\n{value}"),
 						}
@@ -142,14 +140,14 @@ impl Oauth {
 
 #[derive(Debug)]
 enum AuthError {
-	Hyper(hyper::Error),
+	Wreq(wreq::Error),
 	SerdeDeserialize(serde_json::Error),
 	Field((serde_json::Value, &'static str)),
 }
 
-impl From<hyper::Error> for AuthError {
-	fn from(err: hyper::Error) -> Self {
-		AuthError::Hyper(err)
+impl From<wreq::Error> for AuthError {
+	fn from(err: wreq::Error) -> Self {
+		AuthError::Wreq(err)
 	}
 }
 
@@ -222,7 +220,7 @@ impl OauthBackend for MobileSpoofAuth {
 	async fn authenticate(&mut self) -> Result<OauthResponse, AuthError> {
 		// Construct URL for OAuth token
 		let url = format!("{AUTH_ENDPOINT}/auth/v2/oauth/access-token/loid");
-		let mut builder = Request::builder().method(Method::POST).uri(&url);
+		let mut builder = CLIENT.post(&url);
 
 		// Add headers from spoofed client
 		for (key, value) in &self.device.initial_headers {
@@ -239,16 +237,11 @@ impl OauthBackend for MobileSpoofAuth {
 		let json = json!({
 				"scopes": ["*","email", "pii"]
 		});
-		let body = Body::from(json.to_string());
 
-		// Build request
-		let request = builder.body(body).unwrap();
-
-		trace!("Sending token request...\n\n{request:?}");
+		trace!("Sending token request to {url}...");
 
 		// Send request
-		let client: &std::sync::LazyLock<client::Client<_, Body>> = &CLIENT;
-		let resp = client.request(request).await?;
+		let resp = builder.json(&json).send().await?;
 
 		trace!("Received response with status {} and length {:?}", resp.status(), resp.headers().get("content-length"));
 		trace!("OAuth headers: {:#?}", resp.headers());
@@ -259,19 +252,20 @@ impl OauthBackend for MobileSpoofAuth {
 		// Not worried about the privacy implications, since this is randomly changed
 		// and really only as privacy-concerning as the OAuth token itself.
 		if let Some(header) = resp.headers().get("x-reddit-loid") {
-			self.additional_headers.insert("x-reddit-loid".to_owned(), header.to_str().unwrap().to_string());
+			let header_val: &wreq::header::HeaderValue = header;
+			self.additional_headers.insert("x-reddit-loid".to_owned(), header_val.to_str().unwrap().to_string());
 		}
 
 		// Same with x-reddit-session
 		if let Some(header) = resp.headers().get("x-reddit-session") {
-			self.additional_headers.insert("x-reddit-session".to_owned(), header.to_str().unwrap().to_string());
+			let header_val: &wreq::header::HeaderValue = header;
+			self.additional_headers.insert("x-reddit-session".to_owned(), header_val.to_str().unwrap().to_string());
 		}
 
 		trace!("Serializing response...");
 
 		// Serialize response
-		let body_bytes = hyper::body::to_bytes(resp.into_body()).await?;
-		let json: serde_json::Value = serde_json::from_slice(&body_bytes).map_err(AuthError::SerdeDeserialize)?;
+		let json: serde_json::Value = resp.json().await?;
 
 		trace!("Accessing relevant fields...");
 
@@ -341,7 +335,7 @@ impl OauthBackend for GenericWebAuth {
 	async fn authenticate(&mut self) -> Result<OauthResponse, AuthError> {
 		// Construct URL for OAuth token
 		let url = "https://www.reddit.com/api/v1/access_token";
-		let mut builder = Request::builder().method(Method::POST).uri(url);
+		let mut builder = CLIENT.post(url);
 
 		// Add minimal headers
 		builder = builder.header("Host", "www.reddit.com");
@@ -356,16 +350,11 @@ impl OauthBackend for GenericWebAuth {
 
 		// Set up form body
 		let body_str = format!("grant_type=https%3A%2F%2Foauth.reddit.com%2Fgrants%2Finstalled_client&device_id={}", self.device_id);
-		let body = Body::from(body_str);
-
-		// Build request
-		let request = builder.body(body).unwrap();
 
-		trace!("Sending GenericWebAuth token request...\n\n{request:?}");
+		trace!("Sending GenericWebAuth token request to {url}...");
 
 		// Send request
-		let client: &std::sync::LazyLock<client::Client<_, Body>> = &CLIENT;
-		let resp = client.request(request).await?;
+		let resp: wreq::Response = builder.body(body_str).send().await?;
 
 		trace!("Received response with status {} and length {:?}", resp.status(), resp.headers().get("content-length"));
 		trace!("GenericWebAuth headers: {:#?}", resp.headers());
@@ -376,19 +365,20 @@ impl OauthBackend for GenericWebAuth {
 		// Not worried about the privacy implications, since this is randomly changed
 		// and really only as privacy-concerning as the OAuth token itself.
 		if let Some(header) = resp.headers().get("x-reddit-loid") {
-			self.additional_headers.insert("x-reddit-loid".to_owned(), header.to_str().unwrap().to_string());
+			let header_val: &wreq::header::HeaderValue = header;
+			self.additional_headers.insert("x-reddit-loid".to_owned(), header_val.to_str().unwrap().to_string());
 		}
 
 		// Same with x-reddit-session
 		if let Some(header) = resp.headers().get("x-reddit-session") {
-			self.additional_headers.insert("x-reddit-session".to_owned(), header.to_str().unwrap().to_string());
+			let header_val: &wreq::header::HeaderValue = header;
+			self.additional_headers.insert("x-reddit-session".to_owned(), header_val.to_str().unwrap().to_string());
 		}
 
 		trace!("Serializing GenericWebAuth response...");
 
 		// Serialize response
-		let body_bytes = hyper::body::to_bytes(resp.into_body()).await?;
-		let json: serde_json::Value = serde_json::from_slice(&body_bytes).map_err(AuthError::SerdeDeserialize)?;
+		let json: serde_json::Value = resp.json().await?;
 
 		trace!("Accessing relevant fields...");
 
@@ -479,62 +469,67 @@ fn choose<T: Copy>(list: &[T]) -> T {
 	*fastrand::choose_multiple(list.iter(), 1)[0]
 }
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_mobile_spoof_backend() {
-	// Test MobileSpoofAuth backend specifically
-	let mut backend = MobileSpoofAuth::new();
-	let response = backend.authenticate().await;
-	assert!(response.is_ok());
-	let response = response.unwrap();
-	assert!(!response.token.is_empty());
-	assert!(response.expires_in > 0);
-	assert!(!backend.user_agent().is_empty());
-	assert!(!backend.get_headers().is_empty());
-}
+#[cfg(test)]
+mod tests {
+	use super::*;
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_mobile_spoof_backend() {
+		// Test MobileSpoofAuth backend specifically
+		let mut backend = MobileSpoofAuth::new();
+		let response = backend.authenticate().await;
+		assert!(response.is_ok());
+		let response = response.unwrap();
+		assert!(!response.token.is_empty());
+		assert!(response.expires_in > 0);
+		assert!(!backend.user_agent().is_empty());
+		assert!(!backend.get_headers().is_empty());
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_generic_web_backend() {
-	// Test GenericWebAuth backend specifically
-	let mut backend = GenericWebAuth::new();
-	let response = backend.authenticate().await;
-	assert!(response.is_ok());
-	let response = response.unwrap();
-	assert!(!response.token.is_empty());
-	assert!(response.expires_in > 0);
-	assert!(!backend.user_agent().is_empty());
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_generic_web_backend() {
+		// Test GenericWebAuth backend specifically
+		let mut backend = GenericWebAuth::new();
+		let response = backend.authenticate().await;
+		assert!(response.is_ok());
+		let response = response.unwrap();
+		assert!(!response.token.is_empty());
+		assert!(response.expires_in > 0);
+		assert!(!backend.user_agent().is_empty());
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_oauth_client() {
-	// Integration test - tests the overall Oauth client
-	assert!(OAUTH_CLIENT.load_full().headers_map.contains_key("Authorization"));
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_oauth_client() {
+		// Integration test - tests the overall Oauth client
+		assert!(OAUTH_CLIENT.load_full().headers_map.contains_key("Authorization"));
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_oauth_client_refresh() {
-	force_refresh_token().await;
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_oauth_client_refresh() {
+		force_refresh_token().await;
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_oauth_token_exists() {
-	let client = OAUTH_CLIENT.load_full();
-	let auth_header = client.headers_map.get("Authorization").unwrap();
-	assert!(auth_header.starts_with("Bearer "));
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_oauth_token_exists() {
+		let client = OAUTH_CLIENT.load_full();
+		let auth_header = client.headers_map.get("Authorization").unwrap();
+		assert!(auth_header.starts_with("Bearer "));
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_oauth_headers_len() {
-	assert!(OAUTH_CLIENT.load_full().headers_map.len() >= 3);
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_oauth_headers_len() {
+		assert!(OAUTH_CLIENT.load_full().headers_map.len() >= 3);
+	}
 
-#[test]
-fn test_creating_device() {
-	Device::new();
-}
+	#[test]
+	fn test_creating_device() {
+		Device::new();
+	}
 
-#[test]
-fn test_creating_backends() {
-	// Test that both backends can be created
-	MobileSpoofAuth::new();
-	GenericWebAuth::new();
+	#[test]
+	fn test_creating_backends() {
+		// Test that both backends can be created
+		MobileSpoofAuth::new();
+		GenericWebAuth::new();
+	}
 }
diff --git a/src/post.rs b/src/post.rs
index 78bcaf3..8dac506 100644
--- a/src/post.rs
+++ b/src/post.rs
@@ -1,6 +1,4 @@
 #![allow(clippy::cmp_owned)]
-
-// CRATES
 use crate::client::json;
 use crate::config::get_setting;
 use crate::server::RequestExt;
@@ -8,9 +6,8 @@ use crate::subreddit::{can_access_quarantine, quarantine};
 use crate::utils::{
 	error, format_num, get_filters, nsfw_landing, param, parse_post, rewrite_emotes, setting, template, time, val, Author, Awards, Comment, Flair, FlairPart, Post, Preferences,
 };
-use hyper::{Body, Request, Response};
-
 use askama::Template;
+use hyper::{Body, Request, Response};
 use regex::Regex;
 use std::collections::{HashMap, HashSet};
 use std::sync::LazyLock;
diff --git a/src/search.rs b/src/search.rs
index c7d6b00..c04d1f8 100644
--- a/src/search.rs
+++ b/src/search.rs
@@ -1,6 +1,4 @@
 #![allow(clippy::cmp_owned)]
-
-// CRATES
 use crate::utils::{self, catch_random, error, filter_posts, format_num, format_url, get_filters, param, redirect, setting, template, val, Post, Preferences};
 use crate::{
 	client::json,
diff --git a/src/subreddit.rs b/src/subreddit.rs
index 4046be1..3487725 100644
--- a/src/subreddit.rs
+++ b/src/subreddit.rs
@@ -1,12 +1,10 @@
 #![allow(clippy::cmp_owned)]
 
-use crate::{config, utils};
-// CRATES
 use crate::utils::{
-	catch_random, error, filter_posts, format_num, format_url, get_filters, info, nsfw_landing, param, redirect, rewrite_urls, setting, template, val, Post, Preferences,
-	Subreddit,
+	Post, Preferences, Subreddit, catch_random, error, filter_posts, format_num, format_url, get_filters, info, nsfw_landing, param, redirect, rewrite_urls, setting, template, to_absolute_url, val
 };
 use crate::{client::json, server::RequestExt, server::ResponseExt};
+use crate::{config, utils};
 use askama::Template;
 use cookie::Cookie;
 use htmlescape::decode_html;
@@ -14,6 +12,7 @@ use hyper::{Body, Request, Response};
 
 use chrono::DateTime;
 use regex::Regex;
+use rss::{ChannelBuilder, Item, Enclosure};
 use std::sync::LazyLock;
 use time::{Duration, OffsetDateTime};
 
@@ -595,7 +594,6 @@ pub async fn rss(req: Request<Body>) -> Result<Response<Body>, String> {
 	}
 
 	use hyper::header::CONTENT_TYPE;
-	use rss::{ChannelBuilder, Item};
 
 	// Get subreddit
 	let sub = req.param("sub").unwrap_or_default();
@@ -605,6 +603,9 @@ pub async fn rss(req: Request<Body>) -> Result<Response<Body>, String> {
 	// Get path
 	let path = format!("/r/{sub}/{sort}.json?{}", req.uri().query().unwrap_or_default());
 
+	// Get subreddit link
+	let subreddit_link: String = format!("{}/r/{sub}", config::get_setting("REDLIB_FULL_URL").unwrap_or_default());
+
 	// Get subreddit data
 	let subreddit = subreddit(&sub, false).await?;
 
@@ -615,21 +616,23 @@ pub async fn rss(req: Request<Body>) -> Result<Response<Body>, String> {
 	let channel = ChannelBuilder::default()
 		.title(&subreddit.title)
 		.description(&subreddit.description)
+		.link(&subreddit_link)
 		.items(
 			posts
 				.into_iter()
-				.map(|post| Item {
-					title: Some(post.title.to_string()),
-					link: Some(format_url(&utils::get_post_url(&post))),
-					author: Some(post.author.name),
-					content: Some(rewrite_urls(&decode_html(&post.body).unwrap())),
-					pub_date: Some(DateTime::from_timestamp(post.created_ts as i64, 0).unwrap_or_default().to_rfc2822()),
-					description: Some(format!(
-						"<a href='{}{}'>Comments</a>",
-						config::get_setting("REDLIB_FULL_URL").unwrap_or_default(),
-						post.permalink
-					)),
-					..Default::default()
+				.map(|post| {
+					let mut item = Item {
+						title: Some(post.title.to_string()),
+						link: Some(format_url(&utils::get_post_url(&post))),
+						author: Some(post.author.name.to_string()),
+						content: Some(rewrite_urls(&decode_html(&post.body).unwrap())),
+						pub_date: Some(DateTime::from_timestamp(post.created_ts as i64, 0).unwrap_or_default().to_rfc2822()),
+						description: Some(format!("<a href='{}'>Comments</a>", to_absolute_url(&post.permalink))),
+						..Default::default()
+					};
+
+					apply_enclosure(&mut item, &post);
+					item
 				})
 				.collect::<Vec<_>>(),
 		)
@@ -645,16 +648,88 @@ pub async fn rss(req: Request<Body>) -> Result<Response<Body>, String> {
 	Ok(res)
 }
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_fetching_subreddit() {
-	let subreddit = subreddit("rust", false).await;
-	assert!(subreddit.is_ok());
+// Set enclosure image for RSS feed item
+fn apply_enclosure(item: &mut Item, post: &Post) {
+	item.set_enclosure(get_rss_image(&post));
+
+	// Embed the number of gallery images in description and content since
+	// only the first image in the gallery is used for the enclosure
+	if post.post_type == "gallery" && post.gallery.len() > 1 {
+		item.set_description(
+			format!("<a href='{}'>Gallery with {} images</a>",
+				to_absolute_url(&post.permalink),
+				post.gallery.len()
+			)
+		);
+
+		if let Some(content) = item.content() {
+			let new_content = format!(
+				"{}<br/>{}",
+				item.description().unwrap_or(""),
+				content,
+			);
+			item.set_content(new_content);
+		}
+	}
+
+}
+
+fn get_rss_image(post: &Post) -> Option<Enclosure> {
+	let image_url = match post.post_type.as_str() {
+		"image" => Some(post.media.url.clone()),
+		"gallery" => post.gallery.get(0).and_then(|media| decode_html(&media.url).ok()),
+		"gif" | "video" => decode_html(&post.media.poster).ok(),
+		_ => None,
+	};
+
+	image_url.map(|url| {
+		let mut enclosure = Enclosure::default();
+		enclosure.set_mime_type(get_mime_type(&url));
+		enclosure.set_url(to_absolute_url(&url));
+		enclosure.set_length("0");
+		enclosure
+	})
+}
+
+/// Determines the MIME type based on file extension in a URL.
+/// Handles both absolute and relative URLs with query parameters.
+fn get_mime_type(url: &str) -> &'static str {
+    // Extract the path component, removing query parameters
+    let path = url.split('?').next().unwrap_or(url);
+    
+    // Get the file extension (everything after the last dot)
+    let extension = path
+        .rsplit('.')
+        .next()
+        .unwrap_or("")
+        .to_lowercase();
+    
+    // Match common image extensions
+    match extension.as_str() {
+        "jpg" | "jpeg" => "image/jpeg",
+        "png" => "image/png",
+        "gif" => "image/gif",
+        "webp" => "image/webp",
+        "svg" => "image/svg+xml",
+        _ => "application/octet-stream",
+    }
 }
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_gated_and_quarantined() {
-	let quarantined = subreddit("edgy", true).await;
-	assert!(quarantined.is_ok());
-	let gated = subreddit("drugs", true).await;
-	assert!(gated.is_ok());
+#[cfg(test)]
+mod tests {
+	use super::*;
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_fetching_subreddit() {
+		let subreddit = subreddit("rust", false).await;
+		assert!(subreddit.is_ok());
+	}
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_gated_and_quarantined() {
+		let quarantined = subreddit("edgy", true).await;
+		assert!(quarantined.is_ok());
+		let gated = subreddit("drugs", true).await;
+		assert!(gated.is_ok());
+	}
 }
diff --git a/src/user.rs b/src/user.rs
index a352d25..36a06c4 100644
--- a/src/user.rs
+++ b/src/user.rs
@@ -1,6 +1,4 @@
 #![allow(clippy::cmp_owned)]
-
-// CRATES
 use crate::client::json;
 use crate::server::RequestExt;
 use crate::utils::{error, filter_posts, format_url, get_filters, nsfw_landing, param, setting, template, Post, Preferences, User};
@@ -58,7 +56,7 @@ pub async fn profile(req: Request<Body>) -> Result<Response<Body>, String> {
 	// Return landing page if this post if this Reddit deems this user NSFW,
 	// but we have also disabled the display of NSFW content or if the instance
 	// is SFW-only.
-	if user.nsfw && crate::utils::should_be_nsfw_gated(&req, &req_url) {
+	if user.nsfw && utils::should_be_nsfw_gated(&req, &req_url) {
 		return Ok(nsfw_landing(req, req_url).await.unwrap_or_default());
 	}
 
@@ -185,9 +183,14 @@ pub async fn rss(req: Request<Body>) -> Result<Response<Body>, String> {
 	Ok(res)
 }
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_fetching_user() {
-	let user = user("spez").await;
-	assert!(user.is_ok());
-	assert!(user.unwrap().karma > 100);
+#[cfg(test)]
+mod tests {
+	use super::*;
+
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_fetching_user() {
+		let user = user("spez").await;
+		assert!(user.is_ok());
+		assert!(user.unwrap().karma > 100);
+	}
 }
diff --git a/src/utils.rs b/src/utils.rs
index efe98b7..36b798f 100644
--- a/src/utils.rs
+++ b/src/utils.rs
@@ -2,9 +2,6 @@
 #![allow(clippy::cmp_owned)]
 
 use crate::config::{self, get_setting};
-//
-// CRATES
-//
 use crate::{client::json, server::RequestExt};
 use askama::Template;
 use cookie::Cookie;
@@ -1433,23 +1430,28 @@ pub fn url_path_basename(path: &str) -> String {
 	}
 }
 
-/// Returns the URL of a post, as needed by RSS feeds
+/// Returns the absolute URL of a post, as needed by RSS feeds
 pub fn get_post_url(post: &Post) -> String {
+	match post.post_type.as_str() {
+		"image" | "gallery" | "gif" | "video" => return to_absolute_url(&post.permalink),
+		_ => {}
+	}
+
 	if let Some(out_url) = &post.out_url {
-		// Handle cross post
-		if out_url.starts_with("/r/") {
-			format!("{}{}", config::get_setting("REDLIB_FULL_URL").unwrap_or_default(), out_url)
-		} else {
-			out_url.to_string()
-		}
-	} else {
-		format!("{}{}", config::get_setting("REDLIB_FULL_URL").unwrap_or_default(), post.permalink)
+		return if out_url.starts_with("/r/") { to_absolute_url(out_url) } else { out_url.clone() };
 	}
+
+	to_absolute_url(&post.permalink)
+}
+
+/// Returns an absolute URL given a relative URL, as needed by RSS feeds
+pub fn to_absolute_url(relative_path: &str) -> String {
+	format!("{}{}", config::get_setting("REDLIB_FULL_URL").unwrap_or_default(), relative_path)
 }
 
 #[cfg(test)]
 mod tests {
-	use super::{format_num, format_url, rewrite_urls, Preferences};
+	use super::{deflate_compress, deflate_decompress, format_num, format_url, render_bullet_lists, rewrite_emotes, rewrite_urls, url_path_basename, Post, Preferences};
 
 	#[test]
 	fn format_num_works() {
@@ -1546,76 +1548,75 @@ mod tests {
 
 		assert_eq!(urlencoded, "theme=laserwave&front_page=default&layout=compact&wide=on&blur_spoiler=on&show_nsfw=off&blur_nsfw=on&hide_hls_notification=off&video_quality=best&hide_sidebar_and_summary=off&use_hls=on&autoplay_videos=on&fixed_navbar=on&disable_visit_reddit_confirmation=on&comment_sort=confidence&post_sort=top&subscriptions=memes%2Bmildlyinteresting&filters=&hide_awards=off&hide_score=off&remove_default_feeds=off");
 	}
-}
 
-#[test]
-fn test_rewriting_emoji() {
-	let input = r#"<div class="md"><p>How can you have such hard feelings towards a license? <img src="https://www.redditstatic.com/marketplace-assets/v1/core/emotes/snoomoji_emotes/free_emotes_pack/shrug.gif" width="20" height="20" style="vertical-align:middle"> Let people use what license they want, and BSD is one of the least restrictive ones AFAIK.</p>"#;
-	let output = r#"<div class="md"><p>How can you have such hard feelings towards a license? <img src="/static/marketplace-assets/v1/core/emotes/snoomoji_emotes/free_emotes_pack/shrug.gif" width="20" height="20" style="vertical-align:middle"> Let people use what license they want, and BSD is one of the least restrictive ones AFAIK.</p>"#;
-	assert_eq!(rewrite_urls(input), output);
-}
+	#[test]
+	fn test_rewriting_emoji() {
+		let input = r#"<div class="md"><p>How can you have such hard feelings towards a license? <img src="https://www.redditstatic.com/marketplace-assets/v1/core/emotes/snoomoji_emotes/free_emotes_pack/shrug.gif" width="20" height="20" style="vertical-align:middle"> Let people use what license they want, and BSD is one of the least restrictive ones AFAIK.</p>"#;
+		let output = r#"<div class="md"><p>How can you have such hard feelings towards a license? <img src="/static/marketplace-assets/v1/core/emotes/snoomoji_emotes/free_emotes_pack/shrug.gif" width="20" height="20" style="vertical-align:middle"> Let people use what license they want, and BSD is one of the least restrictive ones AFAIK.</p>"#;
+		assert_eq!(rewrite_urls(input), output);
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_fetching_subreddit_quarantined() {
-	let subreddit = Post::fetch("/r/drugs", true).await;
-	assert!(subreddit.is_ok());
-	assert!(!subreddit.unwrap().0.is_empty());
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_fetching_subreddit_quarantined() {
+		let subreddit = Post::fetch("/r/drugs", true).await;
+		assert!(subreddit.is_ok());
+		assert!(!subreddit.unwrap().0.is_empty());
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_fetching_nsfw_subreddit() {
-	// Gonwild is a place for closed, Euclidean Geometric shapes to exchange their nth terms for karma; showing off their edges in a comfortable environment without pressure.
-	// Find a good sub that is tagged NSFW but that actually isn't in case my future employers are watching (they probably are)
-	// switched from randnsfw as it is no longer functional.
-	let subreddit = Post::fetch("/r/gonwild", false).await;
-	assert!(subreddit.is_ok());
-	assert!(!subreddit.unwrap().0.is_empty());
-}
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_fetching_nsfw_subreddit() {
+		// Gonwild is a place for closed, Euclidean Geometric shapes to exchange their nth terms for karma; showing off their edges in a comfortable environment without pressure.
+		// Find a good sub that is tagged NSFW but that actually isn't in case my future employers are watching (they probably are)
+		// switched from randnsfw as it is no longer functional.
+		let subreddit = Post::fetch("/r/gonwild", false).await;
+		assert!(subreddit.is_ok());
+		assert!(!subreddit.unwrap().0.is_empty());
+	}
 
-#[tokio::test(flavor = "multi_thread")]
-async fn test_fetching_ws() {
-	let subreddit = Post::fetch("/r/popular", false).await;
-	assert!(subreddit.is_ok());
-	for post in subreddit.unwrap().0 {
-		assert!(post.ws_url.starts_with("wss://k8s-lb.wss.redditmedia.com/link/"));
+	#[tokio::test(flavor = "multi_thread")]
+	async fn test_fetching_ws() {
+		let subreddit = Post::fetch("/r/popular", false).await;
+		assert!(subreddit.is_ok());
+		for post in subreddit.unwrap().0 {
+			assert!(post.ws_url.starts_with("wss://k8s-lb.wss.redditmedia.com/link/"));
+		}
 	}
-}
 
-#[test]
-fn test_rewriting_image_links() {
-	let input =
-		r#"<p><a href="https://preview.redd.it/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc">caption 1</a></p>"#;
-	let output = r#"<figure><a href="/preview/pre/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc"><img loading="lazy" src="/preview/pre/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc"></a><figcaption>caption 1</figcaption></figure>"#;
-	assert_eq!(rewrite_urls(input), output);
-}
+	#[test]
+	fn test_rewriting_image_links() {
+		let input =
+			r#"<p><a href="https://preview.redd.it/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc">caption 1</a></p>"#;
+		let output = r#"<figure><a href="/preview/pre/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc"><img loading="lazy" src="/preview/pre/6awags382xo31.png?width=2560&amp;format=png&amp;auto=webp&amp;s=9c563aed4f07a91bdd249b5a3cea43a79710dcfc"></a><figcaption>caption 1</figcaption></figure>"#;
+		assert_eq!(rewrite_urls(input), output);
+	}
 
-#[test]
-fn test_url_path_basename() {
-	// without trailing slash
-	assert_eq!(url_path_basename("/first/last"), "last");
-	// with trailing slash
-	assert_eq!(url_path_basename("/first/last/"), "last");
-	// with query parameters
-	assert_eq!(url_path_basename("/first/last/?some=query"), "last");
-	// file path
-	assert_eq!(url_path_basename("/cdn/image.jpg"), "image.jpg");
-	// when a full url is passed instead of just a path
-	assert_eq!(url_path_basename("https://doma.in/first/last"), "last");
-	// empty path
-	assert_eq!(url_path_basename("/"), "");
-}
+	#[test]
+	fn test_url_path_basename() {
+		// without trailing slash
+		assert_eq!(url_path_basename("/first/last"), "last");
+		// with trailing slash
+		assert_eq!(url_path_basename("/first/last/"), "last");
+		// with query parameters
+		assert_eq!(url_path_basename("/first/last/?some=query"), "last");
+		// file path
+		assert_eq!(url_path_basename("/cdn/image.jpg"), "image.jpg");
+		// when a full url is passed instead of just a path
+		assert_eq!(url_path_basename("https://doma.in/first/last"), "last");
+		// empty path
+		assert_eq!(url_path_basename("/"), "");
+	}
 
-#[test]
-fn test_rewriting_emotes() {
-	let json_input = serde_json::from_str(r#"{"emote|t5_31hpy|2028":{"e":"Image","id":"emote|t5_31hpy|2028","m":"image/png","s":{"u":"https://reddit-econ-prod-assets-permanent.s3.amazonaws.com/asset-manager/t5_31hpy/PW6WsOaLcd.png","x":60,"y":60},"status":"valid","t":"sticker"}}"#).expect("Valid JSON");
-	let comment_input = r#"<div class="comment_body "><div class="md"><p>:2028:</p></div></div>"#;
-	let output = r#"<div class="comment_body "><div class="md"><p><img loading="lazy" src="/emote/t5_31hpy/PW6WsOaLcd.png" width="60" height="60" style="vertical-align:text-bottom"></p></div></div>"#;
-	assert_eq!(rewrite_emotes(&json_input, comment_input.to_string()), output);
-}
+	#[test]
+	fn test_rewriting_emotes() {
+		let json_input = serde_json::from_str(r#"{"emote|t5_31hpy|2028":{"e":"Image","id":"emote|t5_31hpy|2028","m":"image/png","s":{"u":"https://reddit-econ-prod-assets-permanent.s3.amazonaws.com/asset-manager/t5_31hpy/PW6WsOaLcd.png","x":60,"y":60},"status":"valid","t":"sticker"}}"#).expect("Valid JSON");
+		let comment_input = r#"<div class="comment_body "><div class="md"><p>:2028:</p></div></div>"#;
+		let output = r#"<div class="comment_body "><div class="md"><p><img loading="lazy" src="/emote/t5_31hpy/PW6WsOaLcd.png" width="60" height="60" style="vertical-align:text-bottom"></p></div></div>"#;
+		assert_eq!(rewrite_emotes(&json_input, comment_input.to_string()), output);
+	}
 
-#[test]
-fn test_rewriting_bullet_list() {
-	let input = r#"<div class="md"><p>Hi, I&#39;ve bought this very same monitor and found no calibration whatsoever. I have an ICC profile that has been set up since I&#39;ve installed its driver from the LG website and it works ok. I also used <a href="http://www.lagom.nl/lcd-test/">http://www.lagom.nl/lcd-test/</a> to calibrate it. After some good tinkering I&#39;ve found the following settings + the color profile from the driver gets me past all the tests perfectly:
+	#[test]
+	fn test_rewriting_bullet_list() {
+		let input = r#"<div class="md"><p>Hi, I&#39;ve bought this very same monitor and found no calibration whatsoever. I have an ICC profile that has been set up since I&#39;ve installed its driver from the LG website and it works ok. I also used <a href="http://www.lagom.nl/lcd-test/">http://www.lagom.nl/lcd-test/</a> to calibrate it. After some good tinkering I&#39;ve found the following settings + the color profile from the driver gets me past all the tests perfectly:
 - Brightness 50 (still have to settle on this one, it&#39;s personal preference, it controls the backlight, not the colors)
 - Contrast 70 (which for me was the default one)
 - Picture mode Custom
@@ -1630,59 +1631,60 @@ fn test_rewriting_bullet_list() {
 - Color Temp Medium
 How`s your monitor by the way? Any IPS bleed whatsoever? I either got lucky or the panel is pretty good, 0 bleed for me, just the usual IPS glow. How about the pixels? I see the pixels even at one meter away, especially on Microsoft Edge&#39;s icon for example, the blue background is just blocky, don&#39;t know why.</p>
 </div>"#;
-	let output = r#"<div class="md"><p>Hi, I&#39;ve bought this very same monitor and found no calibration whatsoever. I have an ICC profile that has been set up since I&#39;ve installed its driver from the LG website and it works ok. I also used <a href="http://www.lagom.nl/lcd-test/">http://www.lagom.nl/lcd-test/</a> to calibrate it. After some good tinkering I&#39;ve found the following settings + the color profile from the driver gets me past all the tests perfectly:
+		let output = r#"<div class="md"><p>Hi, I&#39;ve bought this very same monitor and found no calibration whatsoever. I have an ICC profile that has been set up since I&#39;ve installed its driver from the LG website and it works ok. I also used <a href="http://www.lagom.nl/lcd-test/">http://www.lagom.nl/lcd-test/</a> to calibrate it. After some good tinkering I&#39;ve found the following settings + the color profile from the driver gets me past all the tests perfectly:
 <ul><li>Brightness 50 (still have to settle on this one, it&#39;s personal preference, it controls the backlight, not the colors)</li><li>Contrast 70 (which for me was the default one)</li><li>Picture mode Custom</li><li>Super resolution + Off (it looks horrible anyway)</li><li>Sharpness 50 (default one I think)</li><li>Black level High (low messes up gray colors)</li><li>DFC Off</li><li>Response Time Middle (personal preference, <a href="https://www.blurbusters.com/">https://www.blurbusters.com/</a> show horrible overdrive with it on high)</li><li>Freesync doesn&#39;t matter</li><li>Black stabilizer 50</li><li>Gamma setting on 0</li><li>Color Temp Medium</li></ul>
 How`s your monitor by the way? Any IPS bleed whatsoever? I either got lucky or the panel is pretty good, 0 bleed for me, just the usual IPS glow. How about the pixels? I see the pixels even at one meter away, especially on Microsoft Edge&#39;s icon for example, the blue background is just blocky, don&#39;t know why.</p>
 </div>"#;
 
-	assert_eq!(render_bullet_lists(input), output);
-}
-
-#[test]
-fn test_default_prefs_serialization_loop_json() {
-	let prefs = Preferences::default();
-	let serialized = serde_json::to_string(&prefs).unwrap();
-	let deserialized: Preferences = serde_json::from_str(&serialized).unwrap();
-	assert_eq!(prefs, deserialized);
-}
-
-#[test]
-fn test_default_prefs_serialization_loop_bincode() {
-	let prefs = Preferences::default();
-	test_round_trip(&prefs, false);
-	test_round_trip(&prefs, true);
-}
+		assert_eq!(render_bullet_lists(input), output);
+	}
 
-static KNOWN_GOOD_CONFIGS: &[&str] = &[
-	"ఴӅβØØҞÉဏႢձĬ༧ȒʯऌԔӵ୮༏",
-	"ਧՊΥÀÃǎƱГ۸ඣമĖฤ႙ʟาúໜϾௐɥঀĜໃહཞઠѫҲɂఙ࿔ǲઉƲӟӻĻฅΜδ໖ԜǗဖငƦơ৶Ą௩ԹʛใЛʃශаΏ",
-	"ਧԩΥÀÃÎŠ౭൩ඔႠϼҭöҪƸռઇԾॐნɔາǒՍҰच௨ಖມŃЉŐདƦ๙ϩএఠȝഽйʮჯඒϰळՋ௮ສ৵ऎΦѧਹಧଟƙŃ३î༦ŌပղयƟแҜ།",
-];
-
-#[test]
-fn test_known_good_configs_deserialization() {
-	for config in KNOWN_GOOD_CONFIGS {
-		let bytes = base2048::decode(config).unwrap();
-		let decompressed = deflate_decompress(bytes).unwrap();
-		assert!(bincode::deserialize::<Preferences>(&decompressed).is_ok());
+	#[test]
+	fn test_default_prefs_serialization_loop_json() {
+		let prefs = Preferences::default();
+		let serialized = serde_json::to_string(&prefs).unwrap();
+		let deserialized: Preferences = serde_json::from_str(&serialized).unwrap();
+		assert_eq!(prefs, deserialized);
 	}
-}
 
-#[test]
-fn test_known_good_configs_full_round_trip() {
-	for config in KNOWN_GOOD_CONFIGS {
-		let bytes = base2048::decode(config).unwrap();
-		let decompressed = deflate_decompress(bytes).unwrap();
-		let prefs: Preferences = bincode::deserialize(&decompressed).unwrap();
+	#[test]
+	fn test_default_prefs_serialization_loop_bincode() {
+		let prefs = Preferences::default();
 		test_round_trip(&prefs, false);
 		test_round_trip(&prefs, true);
 	}
-}
 
-fn test_round_trip(input: &Preferences, compression: bool) {
-	let serialized = bincode::serialize(input).unwrap();
-	let compressed = if compression { deflate_compress(serialized).unwrap() } else { serialized };
-	let decompressed = if compression { deflate_decompress(compressed).unwrap() } else { compressed };
-	let deserialized: Preferences = bincode::deserialize(&decompressed).unwrap();
-	assert_eq!(*input, deserialized);
+	static KNOWN_GOOD_CONFIGS: &[&str] = &[
+		"ఴӅβØØҞÉဏႢձĬ༧ȒʯऌԔӵ୮༏",
+		"ਧՊΥÀÃǎƱГ۸ඣമĖฤ႙ʟาúໜϾௐɥঀĜໃહཞઠѫҲɂఙ࿔ǲઉƲӟӻĻฅΜδ໖ԜǗဖငƦơ৶Ą௩ԹʛใЛʃශаΏ",
+		"ਧԩΥÀÃÎŠ౭൩ඔႠϼҭöҪƸռઇԾॐნɔາǒՍҰच௨ಖມŃЉŐདƦ๙ϩএఠȝഽйʮჯඒϰळՋ௮ສ৵ऎΦѧਹಧଟƙŃ३î༦ŌပղयƟแҜ།",
+	];
+
+	#[test]
+	fn test_known_good_configs_deserialization() {
+		for config in KNOWN_GOOD_CONFIGS {
+			let bytes = base2048::decode(config).unwrap();
+			let decompressed = deflate_decompress(bytes).unwrap();
+			assert!(bincode::deserialize::<Preferences>(&decompressed).is_ok());
+		}
+	}
+
+	#[test]
+	fn test_known_good_configs_full_round_trip() {
+		for config in KNOWN_GOOD_CONFIGS {
+			let bytes = base2048::decode(config).unwrap();
+			let decompressed = deflate_decompress(bytes).unwrap();
+			let prefs: Preferences = bincode::deserialize(&decompressed).unwrap();
+			test_round_trip(&prefs, false);
+			test_round_trip(&prefs, true);
+		}
+	}
+
+	fn test_round_trip(input: &Preferences, compression: bool) {
+		let serialized = bincode::serialize(input).unwrap();
+		let compressed = if compression { deflate_compress(serialized).unwrap() } else { serialized };
+		let decompressed = if compression { deflate_decompress(compressed).unwrap() } else { compressed };
+		let deserialized: Preferences = bincode::deserialize(&decompressed).unwrap();
+		assert_eq!(*input, deserialized);
+	}
 }
```
