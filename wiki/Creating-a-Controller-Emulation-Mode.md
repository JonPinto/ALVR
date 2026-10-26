Creating a Controller Emulation Mode
===

This guide explains how ALVR's controller emulation modes work and walks you through adding your own: making SteamVR show *your* choice of controller, with your choice of 3D model, while the player's real inputs keep flowing from whatever headset and controllers they are actually using.

Written against ALVR at commit `adaa16ba`. Line numbers drift over time — treat them as pointers to search from, not gospel.

Throughout, we use one running example: a **"Pico Neo 2"** mode that presents custom-extracted Neo 2 render models to SteamVR while aliasing the existing Pico input identity. This guide is not hypothetical — the example was implemented exactly as described below and passes `cargo xtask build-streamer` (2026-10); where the guide says a step was executed, it was. Your own mode follows the same shape.

> **Certainty markers.** Passages are tagged **[high]** (verified directly in the source), **[medium]** (reasonable inference from docs/history, not yet executed), or **[low/verify]** (plausible but must be tested empirically). Pay special attention to the **[low/verify]** items — there are only a few and they are all flagged.

---

## 1. How it works — the abstract view

ALVR moves input through three layers:

```
 real controller (client)          mapping layer                SteamVR (game side)
┌──────────────────────┐      ┌─────────────────────┐      ┌──────────────────────────┐
│ headset's OpenXR     │      │ ButtonMappingManager │      │ device components        │
│ actions polled,     │─────▶│ adapts the real      │────▶│ /input/trigger/value ... │
│ sent as abstract    │      │ buttons onto the     │      │ + device properties:     │
│ paths e.g.          │      │ emulated profile's   │      │ render model, serial,    │
│ /user/hand/left/    │      │ button set           │      │ controller type, icons   │
│ input/trigger/value │      └─────────────────────┘      └──────────────────────────┘
└──────────────────────┘
```

- The **client** picks its own OpenXR interaction profile based on its hardware and streams button presses as abstract paths (`/user/hand/left/input/a/click`, …). It knows nothing about emulation modes.
- The **server** translates those into the *emulated* profile's buttons automatically — threshold-based mapping from whatever the client sends to whatever the mode declares (`alvr/server_core/src/input_mapping.rs`).
- The **driver** presents a SteamVR device whose identity comes from OpenVR properties (render model, controller type, serial, icons…).

An **emulation mode** is therefore just a named bundle of three server-side definitions **[high]**:

1. **Button set** — which input components SteamVR registers for the controller. Picked in `registered_button_set()` (`alvr/server_core/src/input_mapping.rs:9`). This also drives the automatic source→destination mapping.
2. **Device properties** — the OpenVR property block in `set_device_openvr_props()` (`alvr/server_openvr/src/props.rs`, match at ~422-685): manufacturer, model number, **render model**, **controller type**, **input profile path**, icons, registered device type.
3. **Restart id** — an integer in `compute_restart_settings_hash()` (`alvr/server_core/src/connection.rs:107`) so that switching modes forces a SteamVR restart (device identities are registered at startup).

Plus one optional fourth piece:

4. **Resource files on disk** — render models (`*.obj/*.mtl/*.png`), icons, and a custom input-profile JSON, shipped inside the driver's `resources/` folder. Stock ALVR ships none of this — the copy step, the files, and the workspace exclusion are all yours to add (§6–§8; this guide's example does exactly that).

**What a mode deliberately does *not* do** **[high]**: touch the client, change tracking, or define pose offsets (offsets are global settings, `alvr/session/src/settings.rs:1239-1259`). The dashboard dropdown is generated automatically from the Rust enum by the `SettingsSchema` derive — there is no UI code or JSON schema file to edit. Existing `session.json` files migrate automatically when a new variant appears.

---

## 2. Requirements

**Tooling**
- Rust toolchain + platform build dependencies — see [Building From Source](https://github.com/alvr-org/ALVR/wiki/Building-From-Source). On Linux the streamer additionally requires bundled FFmpeg/x264 headers: run `cargo xtask prepare-deps --platform linux --no-nvidia` once before the first build (it needs `unzip`, `nasm`, `yasm` on the PATH).
- The `openvr` git submodule checked out (`git submodule update --init`); the settings build script parses its headers.
- A SteamVR installation to test against.

**Knowledge / inputs**
- The physical button layout of the controller you are emulating (which buttons exist, analogue vs digital, touch or not).
- Which SteamVR identity to alias (controller type + input profile) — usually you reuse one that already ships with SteamVR rather than writing your own JSON.
- Optionally: model files for a custom render model (§6), icons (§8), your own input-profile JSON (§7).

**A test path**
- A headset/client running ALVR, so you can actually press buttons and see them arrive. Emulation modes are only fully verifiable end-to-end.

---

## 3. Three decisions before you touch code

**D1 — Button set.** Which abstract inputs will the emulated controller expose?
- *Reuse an existing profile* (recommended to start): e.g. `PICO_NEO3_CONTROLLER_PROFILE_ID`. Extra registered components that never fire are harmless. **[high]**
- *Custom set of existing inputs*: hash your own list of path strings (this is what the built-in `Custom` mode does).
- *Genuinely new inputs* (a button ALVR has never modelled): needs changes to the abstract vocabulary in `alvr/common/src/inputs.rs` and possibly the C++ path tables — out of scope for a first profile (§9).

**D2 — Identity.** What does SteamVR call the controller (`Prop_ControllerType_String`) and which input profile does it read (`Prop_InputProfilePath_String`)? These two must match: the input profile JSON declares a `controller_type` that SteamVR checks against the property. Aliasing an existing pair — e.g. `pico_controller` + `{vrlink}/input/pico_controller_profile.json` — means zero new JSON. Quirk worth knowing: the modes themselves already mix and match freely — Vive Wand mode uses controller type `vive_controller` with Oculus's input profile (`props.rs:549`) **[high]**.

**D3 — Render model.** Either an existing SteamVR model (bare name like `oculus_quest2_controller_left`, or token-prefixed like `{vrlink}/rendermodels/pico_4_controller_left`) or your own files under `{alvr_server}/rendermodels/...` (§6).

---

## 4. Step-by-step: the core mode (4 files)

### Step 1 — Add the enum variant

`alvr/session/src/settings.rs`, enum `ControllersEmulationMode` (~line 1008):

```rust
#[schema(strings(display_name = "Pico Neo 2"))]
PicoNeo2,
```

The `display_name` string is what appears in the dashboard dropdown. The enum is matched **exhaustively** in exactly four other places in the codebase — when you build, the compiler will point at every site you still need to handle, which is your safety net for missing a step **[high]**.

> **Note [high]:** `emulation_mode` carries `#[schema(flag = "steamvr-restart")]` (settings.rs:1222), so the dashboard knows switching it requires a SteamVR restart. You don't need to do anything for this.

### Step 2 — Map the mode to a button set

`alvr/server_core/src/input_mapping.rs`, function `registered_button_set()` (~line 9):

```rust
ControllersEmulationMode::PicoNeo2 => CONTROLLER_PROFILE_INFO
    .get(&PICO_NEO3_CONTROLLER_PROFILE_ID)
    .unwrap()
    .button_set
    .clone(),
```

What this single arm controls **[high]**:
- which SteamVR components get registered at device activation (`RegisterButtons` → `server_openvr/src/lib.rs:533`),
- which inputs the automatic mapping will bind *to*.

The registry it points into, `CONTROLLER_PROFILE_INFO`, lives in `alvr/common/src/inputs.rs:160-550`, and is keyed by hashed OpenXR interaction-profile paths declared at `inputs.rs:18-27`.

### Step 3 — Reserve a restart id

`alvr/server_core/src/connection.rs`, in `compute_restart_settings_hash()` (~line 107):

```rust
ControllersEmulationMode::PicoNeo2 => 70,
```

Ids already taken: 0–4 (Rift S, Quest 1/2/3, Pro), 10 (Pico 4), 20 (Index), 40/41 (Wand/Tracker), 60 (PSVR2), 500 (Custom). Pick any unused number **[high]**. If the value collides with another mode's, switching between those two modes won't restart SteamVR — subtle bug, easy to avoid.

### Step 4 — Device properties

Two edits in `alvr/server_openvr/src/props.rs`.

**(a) Serial number** — function `serial_number()` (~line 133). Give the mode its own serial so SteamVR and games see real device identities:

```rust
ControllersEmulationMode::PicoNeo2 => "WMHD316Q512X12_Controller",
//  → "WMHD316Q512X12_Controller_Left" / "..._Right"
```

The function appends `_Left`/`_Right` per hand (props.rs:147-151) **[high]**. Use the vendor's own serial format if you extracted one from their driver — it is just a string, but authenticity helps games that fingerprint hardware. (Folding your mode into the `"ALVR Remote Controller"` group instead also works; that's what Pico 4 does.)

**(b) The property block** — add an arm to the `match config.emulation_mode` (~line 382). The arm as shipped by this example:

```rust
ControllersEmulationMode::PicoNeo2 => {
    set_prop(TrackingSystemNameString, "pico");
    set_prop(ManufacturerNameString, "Pico");
    if left_hand {
        set_prop(ModelNumberString, "Pico Neo2 Controller left");
        set_prop(
            RenderModelNameString,
            "{alvr_server}/rendermodels/pico_neo2_leftcontroller",
        );
        set_neo2_icons("{alvr_server}/icons/neo2/controller");
    } else if right_hand {
        set_prop(ModelNumberString, "Pico Neo2 Controller right");
        set_prop(
            RenderModelNameString,
            "{alvr_server}/rendermodels/pico_neo2_rightcontroller",
        );
        set_neo2_icons("{alvr_server}/icons/neo2/controller");
    }
    set_prop(ControllerTypeString, "pico_controller");
    set_prop(
        InputProfilePathString,
        "{alvr_server}/input/pico_controller_profile.json",
    );
    set_prop(
        RegisteredDeviceTypeString,
        format!("pico/{device_serial}").as_str(),
    );
}
```

`set_neo2_icons` is a local closure next to `set_icons` (see §8) — it exists because this icon set is all-PNG and `set_icons` hardcodes `.gif` for three states.

Property meanings (all **[high]**, all verified against existing arms):

| Property | Purpose |
|---|---|
| `RenderModelNameString` | which 3D model SteamVR displays (§6 for your own) |
| `ControllerTypeString` | identity games/bindings key off; must match the input profile's `controller_type` |
| `InputProfilePathString` | the input profile JSON SteamVR loads for this device (§7 for shipping your own) |
| `ModelNumberString` / `ManufacturerNameString` / `TrackingSystemNameString` | mostly cosmetic, shown in SteamVR UI |
| `RegisteredDeviceTypeString` | `manufacturer/serial`-style identity string, used by some games (e.g. VRChat) |
| `set_icons(base)` | sets 8 icon properties from one base path: `_off.png`, `_searching.gif`, `_searching_alert.gif`, `_ready.png`, `_ready_alert.png`, `_ready_low.png`, `_standby.png`, `_standby_alert.gif` (props.rs:213-246) |

Gotchas in this file **[high]**:
- The serial is written **after** your match arm (props.rs:620), so setting `SerialNumberString` inside the arm is pointless for controllers — use `serial_number()`. (`device_serial` is still in scope inside the arm if you need it, e.g. for `RegisteredDeviceTypeString`.)
- `Custom` deliberately sets nothing (props.rs:617) — it's the "bring your own props via settings" mode.
- All these props are only applied when `Headset → Controllers` is enabled (props.rs:359).
- String property values are copied into a fixed **256-byte** buffer without a bounds check (props.rs:77-85) — keep values short; a very long string is memory corruption, not truncation. **[high]**

### Step 5 — Build and smoke-test

```bash
cargo test -p alvr_session        # settings schema round-trip tests
cargo xtask build-streamer --keep-config
cargo xtask run-streamer          # or launch the dashboard yourself
```

Checklist for the smoke test **[high]**:
1. Dashboard → **Headset → Controllers → Emulation mode** shows your new display name. (It appears because of the `SettingsSchema` derive — if it doesn't, the enum edit didn't land.)
2. Select it, accept the SteamVR restart.
3. SteamVR shows two controllers with your serial/model in the device list; `vrserver.txt` in the SteamVR logs has no property errors.
4. Press buttons in a game or the SteamVR binding UI and confirm the emulated layout responds. The automatic mapping handles translation from whatever your client sends.

---

## 5. Using the built-in `Custom` mode as a laboratory

Before writing any code you can learn a lot with the stock `Custom` mode (`emulation_mode` → Custom) **[high]**:
- `serial_number`: any string you like.
- `button_set`: list of abstract paths, e.g. `"/user/hand/left/input/a/click"` — these become the registered components.
- `extra_openvr_props` (settings.rs:1227): a list of *(OpenVR property key, value)* pairs you can set from the dashboard. Point `RenderModelNameString` at a model and SteamVR will try to load it.

This is also the cheapest **feasibility spike** for custom render models: drop your model folder into the installed driver's `resources/rendermodels/`, set the prop, restart SteamVR, and see whether the model appears — no recompilation. The path spelling itself is settled (§6), so what you're testing here is that *your* files actually load and look right.

---

## 6. Shipping your own render model (optional)

SteamVR's requirements, quoted from `openvr/docs/Driver_API_Documentation.md` **[high]** (direct quote):

- minimum **three files**: a Wavefront **OBJ** with a material (**MTL**) and a texture (**TGA or PNG**);
- the model **must** be aligned with existing controller render models;
- an `.obj` **must not** contain more than **65,000 vertices**;
- an `.obj` **must** contain only **one object**.

**Where files go.** The layout (from SteamVR's runtime rules and ALVR's existing prop values) is:

```
<driver root>/resources/rendermodels/<model_name>/
    <model_name>.obj
    <model_name>.mtl
    <texture>.png            # whatever map_Kd references
    <model_name>.json        # component manifest, see below
```

`<driver root>` is the folder containing `driver.vrdrivermanifest` — the install root on Windows, `<install>/lib64/alvr` on Linux. The `{alvr_server}` token in a prop path expands to that `resources/` directory, hence `{alvr_server}/rendermodels/<model_name>`.

**The path-spelling question — resolved [high].** Two spellings exist in ALVR's history:
- `{alvr_server}/rendermodels/<name>` — current style, used by the PSVR2 mode (`props.rs:479`);
- `{alvr_server}<name>` (no slash) — what ALVR shipped when it *did* ship Quest 2 meshes (commit `73b0b814`, 2021).

We settled it against the vendor driver this example was extracted from: its DLL contains the literal strings `{pico}/rendermodels/pico_neo2_leftcontroller` and `{pico}/rendermodels/pico_neo2_rightcontroller`, and its `driver.vrdrivermanifest` declares `"name": "pico"` — i.e. the driver name token *plus a slash plus `rendermodels/`*, with the files living in `<driver root>/resources/rendermodels/`. `alvr/xtask/resources/driver.vrdrivermanifest` declares `"name": "alvr_server"`, so `{alvr_server}/rendermodels/<name>` is the correct form. (The no-slash historical form used SteamVR's bare-model-name lookup, not a folder.) The same token rule is why `InputProfilePathString` and icon paths read `{alvr_server}/input/...` and `{alvr_server}/icons/...`.

**The build now ships a `resources/` folder.** Historically `cargo xtask build-streamer` copied exactly one static file — `driver.vrdrivermanifest` (`alvr/xtask/src/build.rs`) — while `Layout::resources_dir()` (`alvr/filesystem/src/lib.rs:183`) sat unused. This example wired it up.

So add the files to the repo and a copy step:

```
alvr/resources/                     ← new directory
├── rendermodels/
│   ├── pico_neo2_leftcontroller/
│   │   ├── pico_neo2_leftcontroller.json   # component manifest
│   │   ├── pico_neo2_leftcontroller.obj    # fallback mesh (see below)
│   │   ├── pico_neo2_leftcontroller.mtl
│   │   ├── pico_neo2_leftcontroller.png    # texture
│   │   ├── pico_neo2_left_body.obj/.mtl    # the 8 animated parts …
│   │   └── ...
│   └── pico_neo2_rightcontroller/
│       └── ...
├── icons/neo2/                     ← §8
└── input/                          ← §7
```

```rust
// alvr/xtask/src/build.rs — inside the "copy static resources" block.
// walkdir is already a dependency of xtask; xshell's Shell does the copying.
let resources_src = afs::workspace_dir().join("alvr/resources");
let resources_dst = build_layout.resources_dir();
sh.create_dir(&resources_dst).unwrap();
for entry in walkdir::WalkDir::new(&resources_src) {
    let entry = entry.unwrap();
    let relative_path = entry.path().strip_prefix(&resources_src).unwrap();
    if relative_path.as_os_str().is_empty() {
        continue;
    }
    let destination_path = resources_dst.join(relative_path);
    if entry.file_type().is_dir() {
        sh.create_dir(&destination_path).unwrap();
    } else {
        sh.copy_file(entry.path(), &destination_path).unwrap();
    }
}
```

Two gotchas **[high]**:
1. **Workspace membership** *(hit during this example)*: `Cargo.toml` has `members = ["alvr/*"]`, so a new `alvr/resources/` directory makes *every* cargo invocation fail with "failed to read alvr/resources/Cargo.toml". Fix: add `exclude = ["alvr/resources"]` to the `[workspace]` table.
2. **Parent directories for copies** *(precaution, [medium])*: xshell's `sh.copy_file` is a thin `fs::copy` wrapper — don't rely on it creating missing parents. Walkdir yields directories before their contents, so create every directory entry and only then copy files (as above).

This mirrors the approach of ALVR's historical model support (commit `73b0b814` added a near-identical block to `xtask`), which was later removed together with the model files (`c1d2a943`) **[high]**.

**Component manifest (button animation).** Models extracted from a vendor's SteamVR driver often include a `<model>.json` describing movable parts — trigger rotation, button press translation, thumbstick motion. ALVR's removed Quest 2 models used this schema (recovered from `git show 73b0b814:...`) **[high]**:

```json
{
    "thumbnail": "pico_neo2_leftcontroller_thumbnail.png",
    "components": {
        "trigger": {
            "filename": "pico_neo2_leftcontroller_trigger.obj",
            "motion": {
                "type": "rotate",
                "controller_axis": 1,
                "trigger_path": "/input/trigger",
                "value_mapping": [0.0, -15.0],
                "pivot": [-0.0031, -0.0287, 0.040629],
                "axis": [1.0, 0.0, 0.0]
            },
            "component_local": {
                "origin": [0.012068, -0.037348, 0.049628],
                "rotate_xyz": [315.0, 0.0, 0.0]
            }
        }
    }
}
```

Conventions: `origin`/`pivot` in **metres**, `rotate_xyz` in **degrees**, `motion.type` ∈ `rotate | translate | joystick`, and the `*_path` fields are SteamVR input paths that should exist in your chosen input profile **[medium]** — Valve's driver docs reference this manifest format but the linked section is a dead link in the submodule, so the schema above is reconstructed from working ALVR history rather than official docs.

**What the manifest is actually for — main obj vs components [medium].** Vendor packages strongly suggest the *manifest's part files* are what SteamVR renders, and the `<name>.obj` at folder root is fallback data:
- the vendor's Neo 3 **right**-controller folder ships with **no main `.obj` at all** — parts + manifest only — yet is a complete, working model;
- in the Neo 2 left folder, the main `.obj` is a byte-copy of the *right*-hand mesh, its `.mtl` is missing entirely from the vendor's own driver, and its material name doesn't match any mtl that ever shipped — yet the left controller displays correctly for everyone, because the 8 manifest-listed `left_*` parts are correct.

Practical takeaway: ship vendor packages verbatim (don't "repair" mains), and if you build a model from scratch, either provide all parts the manifest lists or a correct single main obj. The other direction — a manifest-less package — is the untested case: expect the main obj to render but with no button animation. Also note the manifest's `*_path` inputs must exist in your input profile, and SteamVR draws those paths from `InputProfilePathString` (§7) — a mismatch silently means static buttons.

**Licensing caution.** ALVR is GPLv3. Model files extracted from a vendor's proprietary driver are not GPL material; redistributing them in a fork is a legal question only you can answer. Keeping them out of the repo (users drop their own into `resources/`, gitignored) sidesteps it — that also means not committing `alvr/resources/` contents, only the copy step.

---

## 7. Shipping your own input profile (optional)

Skip this if you're aliasing a SteamVR-shipped profile (§3 D2 — that's the easy path). Ship your own when you need your own `controller_type`, your own button declarations, or your own default bindings.

**The precedent.** The PSVR2 mode points `InputProfilePathString` at `{alvr_server}/input/playstation_vr2_sense_controller_profile.json` — **no such file has ever existed in the repo**, so that path has never been exercised. This example (Pico Neo 2) is the first mode that actually ships one.

**Where the files come from.** Vendor drivers ship them under `resources/input/`. Ours needed four (all copied into `alvr/resources/input/`, packaged by the §6 copy step — no extra code needed):

| File | Role |
|---|---|
| `pico_controller_profile.json` | the profile itself: declares `controller_type`, every `input_source` (with `binding_image_point`s), `legacy_binding`, `remapping`, `default_bindings` |
| `legacy_bindings_pico_controller.json` | legacy-input fallback binding (`legacy_binding` field) |
| `pico_controller_remapping.json` | oculus_touch → pico translation rules (`remapping` field) |
| `vrcompositor_bindings_pico.json` | default binding for the SteamVR compositor; referenced **by relative URL** from `default_bindings`, so it must sit next to the profile |

**The rewrites.** Vendor profiles reference their own driver by token (`{pico}/...`). Ours needed exactly three edits to `pico_controller_profile.json` (everything else ships byte-identical) **[high, executed]**:

1. `"legacy_binding": "{pico}/input/..."` → `"{alvr_server}/input/..."` — functional; makes the profile self-contained.
2. `"remapping": "{pico}/input/..."` → `"{alvr_server}/input/..."` — same.
3. `"input/application_menu"` → `"/input/application_menu"` — vendor typo (missing leading slash); without it SteamVR never matches ALVR's registered `/input/application_menu/click` component.

Left untouched on purpose: the `input_bindingui_*` image entries still say `{pico}/icons/controller_{left,right}.svg` — those files don't exist even in the vendor's own driver, which proves SteamVR tolerates missing binding-UI art (it's a cosmetic placeholder in the binding screen). The fields `resource_root: "pico"` / `driver_name: "pico"` were also left as vendor wrote them — if binding images or legacy bindings misbehave in your SteamVR, these two are the first suspects.

**Matching rules** **[high]**:
- the profile's `controller_type` must equal `ControllerTypeString` (props.rs arm, §4 Step 4b);
- every `input_source` you want animated/registered should line up with the button set you chose in D1 (`registered_button_set()` → `Paths.cpp` component tables) — a source declared only in the profile but never registered does nothing, and a registered component missing from the profile simply won't show in the binding UI;
- `default_bindings[].binding_url` is relative to the profile's folder.

**Verification status:** file packaging verified end-to-end (byte-identical copies land in `build/alvr_streamer_linux/lib64/alvr/resources/input/`, `cargo xtask build-streamer` clean) **[high]**; that SteamVR accepts the rewritten `{alvr_server}` refs at runtime still needs one live SteamVR session **[medium]** — this is the item to watch in the §4 Step 5 smoke test.

---

## 8. Custom icons (optional)

Two options:

1. **Borrow** an existing icon set — e.g. what Pico 4 mode does with `set_icons("{vrlink}/icons/left_pico4")`. Zero files, correct-enough visuals. **[high]** that the paths resolve: ALVR already uses them for Pico 4 mode.
2. **Ship your own** under `alvr/resources/icons/<set>/`, packaged by the §6 copy step, referenced as `{alvr_server}/icons/<set>/<base>`.

If you ship your own, SteamVR's rules **[high]** (direct from `Driver_API_Documentation.md` ~line 486):
- controllers: **32×32 or 64×64** png/gif (add `@2x` for 64×64); wrong sizes get cropped from the top-left;
- one file per state, named exactly as the property values you set imply;
- icons must be a **green-blue gradient** — SteamVR generates the gradient itself and caches it next to your file (vendor packages often contain content-hash duplicates like `controller_off.b4bfb144.png` — those are SteamVR's cache artifacts, byte-identical to the plain file; drop them when vendoring);
- a missing file falls back to `{system}/icons/invalid_status.png`.

**Worked example — the Neo 2 set (executed) [high].** `set_icons()` cannot be used here: it appends `.gif` for `_searching`, `_searching_alert`, `_standby_alert`, while the vendor set is 100% `.png` and has no `standby_alert` at all. The shipped helper, placed right after `set_icons` in props.rs:

```rust
let set_neo2_icons = |base_path: &str| {
    set_prop(NamedIconPathDeviceOffString,           format!("{base_path}_off.png").as_str());
    set_prop(NamedIconPathDeviceSearchingString,     format!("{base_path}_searching.png").as_str());
    set_prop(NamedIconPathDeviceSearchingAlertString,format!("{base_path}_searching_alert.png").as_str());
    set_prop(NamedIconPathDeviceReadyString,         format!("{base_path}_ready.png").as_str());
    set_prop(NamedIconPathDeviceReadyAlertString,    format!("{base_path}_ready_alert.png").as_str());
    set_prop(NamedIconPathDeviceAlertLowString,      format!("{base_path}_ready_low.png").as_str());
    set_prop(NamedIconPathDeviceStandbyString,       format!("{base_path}_standby.png").as_str());
    // no standby_alert in the vendor set: reuse the plain standby icon
    set_prop(NamedIconPathDeviceStandbyAlertString,  format!("{base_path}_standby.png").as_str());
    // gap in set_icons(): the NotReady state, valid since openvr_driver.h Prop_NamedIconPathDeviceNotReady_String = 5006
    set_prop(NamedIconPathDeviceNotReadyString,      format!("{base_path}_error.png").as_str());
};
```

Files: `alvr/resources/icons/neo2/controller_{off,searching,searching_alert,ready,ready_alert,ready_low,standby,error}.png` (32×32 RGBA — verified against the openvr header's "32x32 for others" rule). Both hands share the same generic icons, which is what the vendor driver itself does (its DLL references only the handedness-neutral paths).

---

## 9. Going further (when you actually need it)

| You want… | Then touch… | Cost |
|---|---|---|
| Buttons the client doesn't have (touch, dual-stage trigger…) | abstract input list `controller_inputs!` in `alvr/common/src/inputs.rs:120-153`, a `CONTROLLER_PROFILE_INFO` entry, and possibly SteamVR component tables `alvr/server_openvr/cpp/alvr_server/Paths.cpp` | medium — all server-side |
| The *client* to read a new physical device | `Platform` enum + detection in `alvr/system_info/src/lib.rs:17`, profile selection in `alvr/client_openxr/src/interaction.rs:209` | large — Android/OpenXR work; also your device must run OpenXR |
| A unique SteamVR identity (own input profile JSON, own binding UI) | ship `resources/input/<name>_profile.json` + its binding files, point `InputProfilePathString` at `{alvr_server}/input/<name>_profile.json`, set a unique `ControllerTypeString` — full walkthrough in §7 (the PSVR2 mode's reference, `props.rs:496`, is a dangling example of the same idea) | medium — **[high]** packaging proven by this example; runtime load needs one SteamVR session |
| Different grip pose per mode | no per-mode support: position/rotation offsets are global settings (settings.rs:1239-1259) | — |

---

## 10. Reference

### Where everything lives (verified **[high]**)

| Concern | File |
|---|---|
| Mode enum + display name | `alvr/session/src/settings.rs` (~1009) |
| Default variant (leave unless you want it default) | `alvr/session/src/settings.rs` (~2070, currently `Quest2Touch`) |
| Mode → button set | `alvr/server_core/src/input_mapping.rs:9-48` |
| Button registry / abstract inputs | `alvr/common/src/inputs.rs` (macro ~80-118, registry 160-550) |
| Restart id | `alvr/server_core/src/connection.rs:107-119` |
| Serial numbers | `alvr/server_openvr/src/props.rs:118-178` |
| Device properties (render model, type, profile, icons) | `alvr/server_openvr/src/props.rs` (`set_icons`/helpers 214-275, match block 422-685) |
| Button registration → C++ | `alvr/server_openvr/src/lib.rs:533`, `cpp/alvr_server/Controller.cpp` |
| ALVR path → SteamVR component paths (C++) | `alvr/server_openvr/cpp/alvr_server/Paths.cpp:42-314` |
| Resource files (models, icons, input profiles) | `alvr/resources/{rendermodels,icons,input}/` — committed in this example |
| Driver packaging (resource copy step) | `alvr/xtask/src/build.rs` — "copy static resources" block |
| Workspace exclusion for the resources dir | `/Cargo.toml` (`exclude = ["alvr/resources"]`) |
| `resources_dir()` (destination of the copy) | `alvr/filesystem/src/lib.rs:183` |
| Client's own profile selection (usually untouched) | `alvr/client_openxr/src/interaction.rs:209-222` |

### Existing modes at a glance **[high]**

| Mode | Restart id | Controller type | Input profile | Render model |
|---|---|---|---|---|
| Rift S / Quest 1 / 2 / 3 / Pro | 0–4 | `oculus_touch` | `{oculus}/input/touch_profile.json` | bare `oculus_*_controller_{left,right}` |
| Pico 4 | 10 | `pico_controller` | `{vrlink}/input/pico_controller_profile.json` | `{vrlink}/rendermodels/pico_4_controller_{left,right}` |
| Valve Index | 20 | `knuckles` | `{indexcontroller}/input/index_controller_profile.json` | `{indexcontroller}valve_controller_knu_1_0_{left,right}` |
| Vive Wand | 40 | `vive_controller` | `{oculus}/input/touch_profile.json` ⚠ | `vr_controller_vive_1_5` |
| Vive Tracker | 41 | `vive_tracker_*` | `{htc}/input/vive_tracker_profile.json` | `{htc}vr_tracker_vive_1_0` |
| PSVR2 Sense | 60 | `playstation_vr2_sense` | `{alvr_server}/input/...` ⚠ no file | `{alvr_server}/rendermodels/...` ⚠ no file |
| **Pico Neo 2 (this guide)** | 70 | `pico_controller` | `{alvr_server}/input/pico_controller_profile.json` ✔ ships | `{alvr_server}/rendermodels/pico_neo2_{left,right}controller` ✔ ships |
| Custom | 500 | (none set) | (none set) | (none set) |

⚠ = points at resources that don't exist in the repo; ✔ = ships in `alvr/resources/` (this guide's example). The Vive Wand/Oculus-profile mix is intentional-looking but odd. Learn from these, don't blindly copy them.

### Gotchas checklist
- [ ] New variant added to all four exhaustive `match` sites — enum in `settings.rs`, then `input_mapping.rs`, `connection.rs`, `props.rs` (twice: serial + property block). The compiler forces this; build once and fix what it reports.
- [ ] Restart id is unique.
- [ ] `ControllerTypeString` matches the `controller_type` in the input profile JSON you reference.
- [ ] String props kept well under 256 bytes.
- [ ] Model files: ≤65k verts, single object, textures referenced by the `.mtl` actually present.
- [ ] Resource path spelling: `{alvr_server}/rendermodels/x` (slash form — resolved, see §6).
- [ ] New directory under `alvr/` added to `exclude` in the workspace manifest, or every cargo command fails (§6).
- [ ] If icons aren't all-PNG-with-`_standby_alert`, don't use `set_icons` — write a helper (§8).
- [ ] `cargo test -p alvr_session` passes; mode selectable in dashboard; SteamVR restarts when switching.
- [ ] After selecting the mode: `vrserver.txt` shows no property/path errors, the model renders, and the binding UI lists your profile's inputs.

---

### Confidence summary

- **High** (source-verified, several executed end-to-end in this example): all file/line locations, the four-match safety net, property meanings and gotchas, resource packaging (copy step + workspace exclude + `cargo xtask build-streamer` clean, byte-identical files in the build output), `{driver}/rendermodels/` path spelling (vendor DLL strings + manifest name), icon rules and the explicit-icons workaround (openvr_driver.h:667), SteamVR file rules (quoted), historical commits `73b0b814`/`c1d2a943`, existing-mode table.
- **Medium**: `{driver}` token expands to `<driver>/resources` (confirmed indirectly by the vendor's consistent usage, not by Valve docs), component-manifest schema details (reconstructed from ALVR history, official link is dead), main-obj-is-fallback (strong vendor evidence, never runtime-tested), SteamVR accepting the rewritten `{alvr_server}` input-profile refs at runtime (files verified in the build; one SteamVR session pending), Neo 2 being a strict subset of the Pico Neo 3 button set (based on Pico SDK docs and reviews).
- **Low / verify empirically**: nothing outstanding in the code path — the remaining unknown is purely runtime: launch SteamVR with the mode selected and watch `vrserver.txt`.
