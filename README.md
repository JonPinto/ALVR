<p align="center"> <img width="500" src="resources/ALVR-Grey.svg"/> </p>

# ALVR - Air Light VR (PICO Neo 2 fork)

[![badge-discord][]][link-discord] [![badge-matrix][]][link-matrix] [![badge-opencollective][]][link-opencollective]

Stream VR games from your PC to your headset over Wi-Fi.

This repository is a fork of [ALVR](https://github.com/alvr-org/ALVR), based on the
**v20.14.1** release. Its sole purpose is to stream to the **PICO Neo 2** using the legacy
Pico client, with a PICO Neo 2 emulation mode — see [About this fork](#about-this-fork).

## About this fork

Disclaimer: This fork, although simple, relied heavily on llm agentic coding. I'm stating it here for clarity. Regardless, the changes that this fork implement, are quite minor.

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

First alpha. Needs further testing and making sure basic functionality remains unharmed.

### Known limitations

Remains to be seen, looking into edge cases, to see that the models behave properly at all times. afaik the models should also carry into games that accept /rendermodels via steamvr, but let me know if something appears otherwise.

## Install

Either check the releases tab, or build from source yourself - works same as the mainline alvr.

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

If you want to support the mainline project you can make a donation to [Open Source Collective account](https://opencollective.com/alvr).

You can also donate to the original author of ALVR using Paypal (polygraphene@gmail.com) or with bitcoin (1FCbmFVSjsmpnAj6oLx2EhnzQzzhyxTLEv).

[badge-discord]: https://img.shields.io/discord/720612397580025886?style=for-the-badge&logo=discord&color=5865F2 "Join us on Discord"
[link-discord]: https://discord.gg/ALVR
[badge-matrix]: https://img.shields.io/static/v1?label=chat&message=%23alvr&style=for-the-badge&logo=matrix&color=blueviolet "Join our Matrix"
[link-matrix]: https://matrix.to/#/#alvr:ckie.dev?via=ckie.dev
[badge-opencollective]: https://img.shields.io/badge/donate-open%20collective-blue.svg?style=for-the-badge&logo=open-collective "Donate"
[link-opencollective]: https://opencollective.com/alvr
