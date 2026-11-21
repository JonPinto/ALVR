<p align="center"> <img width="500" src="resources/ALVR-Grey.svg"/> </p>

# ALVR - Air Light VR (PICO Neo 2 fork)

[![badge-discord][]][link-discord] [![badge-matrix][]][link-matrix] [![badge-opencollective][]][link-opencollective]

Stream VR games from your PC to your headset over Wi-Fi.

This repository is a fork of [ALVR](https://github.com/alvr-org/ALVR), based on the
**v20.14.1** release. Its sole purpose is to stream to the **PICO Neo 2** using the legacy
Pico client, with a PICO Neo 2 emulation mode — see [About this fork](#about-this-fork).

## About this fork

This fork adds a **PICO Neo 2 headset and controller emulation mode** to ALVR.

When the mode is selected, SteamVR identifies the connected devices as a PICO Neo 2 and loads the
Neo 2 controller render models, status icons and `pico_controller` input profile **directly from
ALVR's own driver resources** — the same way PICO's proprietary streaming driver serves them from
its `{pico}` driver folder. Games and overlays that special-case the Neo 2 (and its controller
bindings) then behave as they would with PICO's streaming app, without PICO's driver installed.

### Why v20.14.1?

The only client that still supports the PICO Neo 2 is
[**alvr-pico-legacy**](https://github.com/Juspertinry/alvr-pico-legacy), which wraps the ALVR
**20.14.1** client core. ALVR derives a *protocol id* from the version
(`protocol_id()` → `major-pre`, so `20.14.1` → `"20"`), and the server refuses any client whose
protocol id differs. This fork therefore stays on the **v20.14.1** codebase so its protocol id is
`"20"` and it can talk to that client. It is **not** compatible with v21 servers/clients.

### What it adds

- A new **"Pico Neo 2"** entry in both the headset and the controller *emulation mode* settings.
- Controller render models `pico_neo2_leftcontroller` / `pico_neo2_rightcontroller`, including the
  `_oculus` and `_vive` simulation variants, served from the ALVR driver under
  `{alvr_server}/rendermodels/…`.
- The Neo 2 status icons (PNG set) and binding-UI images.
- The Neo 2 input profile, using the shared `pico_controller` controller type so the existing
  SteamVR bindings and the `oculus_touch` compatibility fallback keep working, plus all of its
  default bindings.
- Distinct serial numbers (`PICONEONEO2`, `PICONEONEO2_Controller`) so SteamVR bindings attach to a
  stable device identity.

The headset itself uses SteamVR's built-in `generic_hmd` model, exactly like PICO's driver.

### Enabling the emulation

1. Build and register this fork (see [Build from Source](#build-from-source)).
2. In the ALVR Dashboard, set **Headset → Emulation mode → Pico Neo 2** and
   **Controllers → Emulation mode → Pico Neo 2**.
3. Install/launch [alvr-pico-legacy](https://github.com/Juspertinry/alvr-pico-legacy) on the Neo 2
   and connect.
4. Restart SteamVR.

The controllers should then show up as *"Pico Neo2 Controller left"* / *"Pico Neo2 Controller right"*.

### Status

First alpha. The code builds, type-checks and is format-clean, and the vendored resource tree is
validated. The one thing that has **not** been verified against a live SteamVR is the
`{alvr_server}` resource wildcard resolving to the driver's own `resources/` folder (it is
documented OpenVR behaviour and mirrors exactly how PICO uses `{pico}`). If the controller models do
not appear, see [Known limitations](#known-limitations).

### Known limitations

- The `{alvr_server}` → `<driver>/resources` wildcard mapping is unverified on a live SteamVR. If the
  Neo 2 models do not load, the fallback is a one-line change in `alvr/server_openvr/src/props.rs`:
  replace the two `RenderModelNameString` values
  (`{alvr_server}/rendermodels/pico_neo2_{left,right}controller`) with the bare model names
  `pico_neo2_leftcontroller` / `pico_neo2_rightcontroller`. Nothing else depends on the token form.
- Only the PICO Neo 2 controller models are shipped. The HMD uses SteamVR's `generic_hmd`.
- Based on v20.14.1: it will **not** interoperate with v21 servers or clients.
- The render model assets are proprietary PICO files. They are **not** covered by ALVR's MIT license;
  see [`alvr/xtask/resources/pico_neo2/NOTICE-PICO.txt`](alvr/xtask/resources/pico_neo2/NOTICE-PICO.txt)
  before redistributing this fork.

### Direct download

There are no prebuilt installers for this fork yet. Build it from source (see
[Build from Source](#build-from-source)); any releases will be on the [Releases page](../../releases).

## Compatibility

|          VR Headset          |                                        Support                                         |
| :--------------------------: | :------------------------------------------------------------------------------------: |
|       Apple Vision Pro       |    :heavy_check_mark: ([store link](https://apps.apple.com/app/alvr/id6479728026))     |
|      Quest 1/2/3/3S/Pro      | :heavy_check_mark: ([store link](https://www.meta.com/experiences/7674846229245715) *) |
|     Pico Neo 3/4/4 Ultra     |                                   :heavy_check_mark:                                   |
|    Play For Dream YVR 1/2/MR |                                   :heavy_check_mark:                                   |
| Vive Focus 3/Vision/XR Elite |                                   :heavy_check_mark:                                   |
|           Lynx R1            |                                   :heavy_check_mark:                                   |
|     PhoneVR (smartphone)     |     :heavy_check_mark: ** ([repo](https://github.com/PhoneVR-Developers/PhoneVR))      |
|        Android/Monado        |                                      :warning: **                                      |
|          Oculus Go           |                 :x: ([old repo](https://github.com/polygraphene/ALVR))                 |

\* : ALVR for Quest 1 not available through the Meta store.  
\** : Only works on some smartphones, not enough testing.  

> **PICO Neo 2:** stream to it with the legacy
> [alvr-pico-legacy](https://github.com/Juspertinry/alvr-pico-legacy) client, and enable the
> [Pico Neo 2 emulation mode](#about-this-fork) on this server.

|     PC OS      |                                    Support                                    |
| :------------: | :---------------------------------------------------------------------------: |
| Windows 10/11  | :heavy_check_mark: ([store link](https://store.steampowered.com/app/3312710)) |
| Windows XP/7/8 |                                      :x:                                      |
|     Linux      |                             :heavy_check_mark:***                             |
|     macOS      |                                      :x:                                      |

\*** : Linux support is still in beta. To be able to make audio work or run ALVR at all you may need advanced knowledge of your distro for debugging or building from source.

### Requirements

-   A supported standalone VR headset (see compatibility table above)

-   SteamVR

-   High-end gaming PC
    -   See OS compatibility table above.
    -   NVIDIA GPU that supports NVENC (1000 GTX Series or higher) (or with an AMD GPU that supports AMF VCE) with the latest driver.
    -   Laptops with an onboard (Intel HD, AMD iGPU) and an additional dedicated GPU (NVidia GTX/RTX, AMD HD/R5/R7): you should assign the dedicated GPU or "high performance graphics adapter" to the applications ALVR, SteamVR for best performance and compatibility. (NVidia: Nvidia control panel->3d settings->application settings; AMD: similiar way)

-   802.11ac 5Ghz wireless or ethernet wired connection
    -   It is recommended to use 802.11ac 5Ghz for the headset and ethernet for PC
    -   You need to connect both the PC and the headset to same router (or use a routed connection as described [here](https://github.com/alvr-org/ALVR/wiki/ALVR-v14-and-Above))

## Install

This fork does not ship an installer. Build and register it as described in
[Build from Source](#build-from-source), then configure the emulation mode as described in
[About this fork](#about-this-fork).

## Troubleshooting

-   Please check the [Troubleshooting](https://github.com/alvr-org/ALVR/wiki/Troubleshooting) page, and also [Linux Troubleshooting](https://github.com/alvr-org/ALVR/wiki/Linux-Troubleshooting) if applicable.
-   Configuration recommendations and information may be found [here](https://github.com/alvr-org/ALVR/wiki/Information-and-Recommendations)

## Uninstall

Open `ALVR Dashboard.exe`, go to `Installation` tab then press `Remove firewall rules`. Close ALVR window and delete the ALVR folder.

## Build from Source

ALVR can be built on Windows and Linux. The build produces a self-contained OpenVR driver folder;
there is no separate installer.

### Prerequisites

Common:

-   [rustup](https://rustup.rs) — the pinned toolchain from `rust-toolchain.toml` is installed
    automatically on the first build.
-   [Git](https://git-scm.com/).

Windows only:

-   **Visual Studio Build Tools** with the *Desktop development with C++* workload (MSVC toolchain),
    required for the C++ code and the CMake-built dependencies.
-   [Chocolatey](https://chocolatey.org/install). `prepare-deps` uses it to install `zip`, `unzip`,
    `llvm`, `pkgconfiglite` and `cmake`. That step needs an **elevated** shell.

Linux only:

-   Debian/Ubuntu: `sudo apt install build-essential pkg-config nasm libva-dev libdrm-dev libvulkan-dev libx264-dev libasound2-dev libxrandr-dev libunwind-dev libgtk-3-dev libpipewire-0.3-dev libspa-0.2-dev`

### Clone

```bash
git clone --recurse-submodules https://github.com/JonPinto/ALVR.git
cd ALVR
```

If you already cloned without the submodules, run
`git submodule update --init --recursive` once.

### Build the streamer

Run from the repository root. On Windows, use an **Administrator** terminal:

```powershell
cargo xtask prepare-deps --platform windows
cargo xtask build-streamer --release
```

On Linux:

```bash
cargo xtask prepare-deps --platform linux   # add --no-nvidia on AMD GPUs
cargo xtask build-streamer --release
```

Notes:

-   Add `--gpl` to **both** commands only if you need software video encoding (it downloads, builds
    and bundles FFmpeg/x264, and the resulting package may only be redistributed under GPLv2.0).
    Without it you get hardware encoders only (NVENC / AMF / Quick Sync), which is enough to test.
-   Rebuilding after code edits: skip `prepare-deps` and re-run `build-streamer`.
-   `build-streamer` deletes and recreates the output folder. On Windows the dashboard configuration
    (`session.json`) lives inside it, so pass `--keep-config` if you want to keep your settings.

The result is:

-   Windows: `build\alvr_streamer_windows\`
-   Linux: `build\alvr_streamer_linux\`

That folder **is** the OpenVR driver: it contains `driver.vrdrivermanifest`,
`bin\<platform>\driver_alvr_server.{dll,so}`, the `ALVR Dashboard` executable, and the `resources\`
folder holding the PICO Neo 2 emulation assets.

### Run and register

Windows:

```powershell
.\build\alvr_streamer_windows\"ALVR Dashboard.exe"
```

In the dashboard, open the **Installation** tab and press **Register ALVR driver** (pressing
**Launch SteamVR** also does this automatically). Then set the emulation mode as described in
[About this fork](#about-this-fork).

> Only one ALVR driver should be registered at a time — running two causes a socket conflict. The
> dashboard automatically unregisters other registered ALVR drivers when it launches SteamVR, but if
> you also have a released ALVR installed, prefer to unregister (Installation tab) or uninstall it
> first, and avoid opening the official ALVR Launcher while testing this fork.

For detailed, per-distribution prerequisites (including the optional Android client), see the
upstream wiki's
[Building-From-Source](https://github.com/alvr-org/ALVR/wiki/Building-From-Source) page.

## License

ALVR is licensed under the [MIT License](LICENSE).

The vendored PICO Neo 2 render models, textures and icons are proprietary PICO assets and are **not**
covered by the MIT license. See
[`alvr/xtask/resources/pico_neo2/NOTICE-PICO.txt`](alvr/xtask/resources/pico_neo2/NOTICE-PICO.txt).

## Privacy policy

ALVR apps do not directly collect any kind of data.

## Donate

If you want to support this project you can make a donation to our [Open Source Collective account](https://opencollective.com/alvr).

You can also donate to the original author of ALVR using Paypal (polygraphene@gmail.com) or with bitcoin (1FCbmFVSjsmpnAj6oLx2EhnzQzzhyxTLEv).

[badge-discord]: https://img.shields.io/discord/720612397580025886?style=for-the-badge&logo=discord&color=5865F2 "Join us on Discord"
[link-discord]: https://discord.gg/ALVR
[badge-matrix]: https://img.shields.io/static/v1?label=chat&message=%23alvr&style=for-the-badge&logo=matrix&color=blueviolet "Join our Matrix"
[link-matrix]: https://matrix.to/#/#alvr:ckie.dev?via=ckie.dev
[badge-opencollective]: https://img.shields.io/badge/donate-open%20collective-blue.svg?style=for-the-badge&logo=open-collective "Donate"
[link-opencollective]: https://opencollective.com/alvr
