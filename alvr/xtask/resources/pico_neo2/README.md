# PICO Neo 2 emulation resources

SteamVR resources used by ALVR's `Pico Neo 2` headset and controller emulation
modes. They are copied into `<driver root>/resources/` by the `build-streamer`
xtask, next to `driver.vrdrivermanifest`.

## Provenance

Extracted from PICO's SteamVR driver (`driver_pico.dll` and its `resources/`
folder, shipped with PICO Link / PICO Streaming Assistant). See
`NOTICE-PICO.txt` for the licensing situation.

The layout mirrors the original driver so that SteamVR's `{alvr_server}` wildcard
resolves the same paths PICO's driver uses with `{pico}`:

```
rendermodels/pico_neo2_{left,right}controller/   pico_neo2_*controller.json + obj/mtl/tga
input/pico_neo2_controller_profile.json          controller profile (controller_type: pico_controller)
input/pico_neo2_hmd_profile.json                 HMD profile (controller_type: pico_hmd)
input/*_bindings_pico.json                       default bindings referenced by the profile
icons_neo2/                                      status icons (PNG only)
icons/pico_neo2_*.svg                            binding UI images
```

## How the files were derived

Only the two input profiles were modified; the render models, textures and icons
are byte-identical to PICO's files.

```sh
# controller profile: originally pico_neo3_controller_profile.json
sed -e 's|{pico}|{alvr_server}|g' \
    -e 's|"resource_root": "pico"|"resource_root": "alvr_server"|' \
    -e 's|"driver_name": "pico"|"driver_name": "alvr_server"|' \
    -e 's|/icons/neo3_left.svg|/icons/pico_neo2_controller_left.svg|' \
    -e 's|/icons/neo3_right.svg|/icons/pico_neo2_controller_right.svg|' \
    <pico_neo3_controller_profile.json > pico_neo2_controller_profile.json

# HMD profile: originally pico_hmd_profile.json
sed -e 's|{pico}|{alvr_server}|g' \
    -e 's|/icons/headset.svg|/icons/pico_neo2_headset.svg|' \
    <pico_hmd_profile.json > pico_neo2_hmd_profile.json
```

The Neo 2 and Neo 3 share the same `pico_controller` input profile, which is why
no Neo 2 specific profile exists upstream: it reuses the Neo 3 one, and its
`oculus_touch` compatibility fallback makes existing SteamVR bindings work.

`*_occ.tga` and `*_spec.tga` were dropped from both render model folders: PICO
ships them but no material in the model references them, and SteamVR's render
model loader only reads the textures named in the `.mtl` files. This removes
about 67 MB of dead weight. If you ever notice missing reflections/occlusion,
those are the files to restore.

The `_oculus.json` and `_vive.json` model variants are kept as shipped. SteamVR
picks them when the controller is simulated as an Oculus Touch or Vive wand
controller.

## Adding or updating assets

The assets are committed directly as regular files (not Git LFS: Git LFS objects
cannot be uploaded to a public fork of a repository you do not own). The whole
`pico_neo2/` tree is marked `-text -eol` in `.gitattributes` so the files keep
their original CRLF bytes instead of being normalized on checkout. `build-streamer`
copies the tree verbatim into the driver's `resources/` folder.
