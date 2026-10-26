# This is a fork of [alvr-org/ALVR](https://github.com/alvr-org/ALVR)

Base: upstream `master` at `adaa16ba`. Everything on top of that base is this
fork's work; nothing else diverges from upstream, so syncing stays easy
(see [Syncing with upstream](#syncing-with-upstream)).

## What this fork adds

A **"Pico Neo 2" controller emulation mode** (Headset → Controllers →
Emulation mode) that presents the original Pico Neo 2 render models to
SteamVR, using assets extracted from Pico's own PCVR SteamVR driver.

| Commit | Contents |
|---|---|
| `feat: package Pico Neo 2 SteamVR assets with the driver` | New `alvr/resources/` tree (render models, 32×32 icon set, self-contained SteamVR input profile), the `xtask` copy step that ships it into the driver's `resources/` folder, workspace `exclude`, `.gitattributes` rules keeping vendor bytes exact |
| `feat: add Pico Neo 2 controller emulation mode` | The mode itself: enum variant, button set (reuses the Pico Neo 3 component set), SteamVR restart id 70, full device identity + render model + icons + input profile wired in `props.rs` |
| `docs(wiki): add "Creating a Controller Emulation Mode"` | End-to-end guide to emulation modes using this mode as the worked example, with confidence tags on every non-obvious claim |

The detailed rationale for every change lives in the commit messages; the
conceptual walkthrough lives in
[`wiki/Creating-a-Controller-Emulation-Mode.md`](wiki/Creating-a-Controller-Emulation-Mode.md).

## Building

Clone **with submodules** (the `openvr` headers are required):

```bash
git clone --recurse-submodules <this repo>
cd ALVR
```

### Windows 11 (recommended target)

Prerequisites: [Rust](https://rustup.rs), Visual Studio 2022 with the
"Desktop development with C++" workload, [Chocolatey](https://chocolatey.org).

```powershell
cargo xtask prepare-deps --platform windows   # downloads FFmpeg/x264/vulkan headers; auto-installs its own choco tools
cargo xtask build-streamer --platform windows # add --release for optimized
```

Result: `build/alvr_streamer_windows/` — run `alvr_dashboard.exe` from there
to register the driver with SteamVR.

Alternatively, without any local toolchain: the repo's CI
(`.github/workflows/build.yml`) builds a Windows package — open a pull
request **from any branch into `master` inside this fork** (not against
upstream) and download the `Windows packages` artifact from the Actions run.

### Linux

```bash
sudo apt install -y pkg-config libclang-dev libssl-dev libasound2-dev libjack-dev \
  libgtk-3-dev libvulkan-dev libunwind-dev yasm nasm unzip libx264-dev libx265-dev \
  libxcb-render0-dev libxcb-shape0-dev libxcb-xfixes0-dev libspeechd-dev \
  libxkbcommon-dev libdrm-dev libva-dev libspa-0.2-dev libpipewire-0.3-dev
cargo xtask prepare-deps --platform linux --no-nvidia   # drop --no-nvidia on NVIDIA hosts
cargo xtask build-streamer --keep-config
```

Result: `build/alvr_streamer_linux/`.

## Test status

| Check | Status |
|---|---|
| `cargo xtask build-streamer` (Linux) | ✅ clean, resources land byte-identical in `lib64/alvr/resources/` |
| `cargo test -p alvr_session`, `cargo fmt --check`, `cargo clippy` | ✅ clean (only pre-existing C++ warnings) |
| Binary contains the new props/paths (string check) | ✅ |
| Windows build | ⬜ not yet produced — use the CI artifact or a Windows build |
| Runtime test in SteamVR (model renders, icons, profile loads, buttons respond) | ⬜ pending — watch `vrserver.txt` for property/path errors |

## About the shipped assets

`alvr/resources/` (~106 MB) contains render models, textures and icons
extracted from Pico's proprietary PCVR SteamVR driver, committed verbatim on
purpose (including the vendor's own stale/unused files — see the guide's
component-manifest section for why they are left untouched). ALVR is GPLv3;
redistributing third-party proprietary assets in a fork is a decision specific
to this repository.

## Syncing with upstream

```bash
git remote add upstream https://github.com/alvr-org/ALVR.git   # once
git fetch upstream
git rebase upstream/master        # or: git merge upstream/master
```

The fork's changes are self-contained (4 small commits: 6 files of
code/manifest edits + the new `alvr/resources/` tree + docs), so conflicts
with upstream should be rare. If you rebase, push with `--force-with-lease`
to this fork only.
