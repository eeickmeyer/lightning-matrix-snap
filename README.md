# lightning-matrix snap

Snap packaging for [Lightning](https://github.com/Mizerd/lightning), a native
Qt 6 / QML Matrix client built on the official Matrix Rust SDK.

The snap builds from the upstream git tag named by `source-tag` in
[snap/snapcraft.yaml](snap/snapcraft.yaml); its version is derived from that
tag.

## Build

```sh
sudo snap install snapcraft --classic
sudo snap install lxd && sudo lxd init --auto
snapcraft pack
```

The snap uses `base: core26` (Ubuntu 26.04), whose Qt 6.10, GStreamer 1.28 and
Rust 1.93 meet Lightning's requirements. Qt and GStreamer are staged into the
snap rather than taken from a KDE content snap; graphics come from the
`mesa-2604` content snap.

## Install and connect

```sh
sudo snap install --dangerous ./lightning-matrix_*.snap
sudo snap install mesa-2604
sudo snap connect lightning-matrix:gpu-2604 mesa-2604:gpu-2604
sudo snap connect lightning-matrix:audio-record
sudo snap connect lightning-matrix:camera
sudo snap connect lightning-matrix:password-manager-service
```

Without `gpu-2604` the app still starts, on a software renderer that cannot
draw call video. Run it from a terminal to see the hint.

## Updating to a new upstream release

Change `source-tag` in `snap/snapcraft.yaml`. If a release adds build
dependencies, mirror `Build-Depends` and `Depends` from upstream's
`debian/control`, and check that the update patch still applies.

## Choices worth knowing

- **GIF provider keys are embedded**, as in upstream's Flatpak and Debian
  builds. The GIPHY and KLIPY application keys are set through
  `build-environment` on the `lightning` part and the build passes
  `-DLIGHTNING_REQUIRE_GIF_KEYS=ON`, so a missing key fails the configure step.
  A key compiled into a distributed binary is extractable; these are provider
  application keys, not Matrix credentials. `LIGHTNING_GIPHY_API_KEY` /
  `LIGHTNING_KLIPY_API_KEY` in the environment still override them.
- **DeepFilterNet and RNNoise noise suppression are off**, as in the Debian
  package: their model weights carry no explicit licence.
- **Qt's SVG image plugin is excluded.** Lightning must never decode untrusted
  SVG.
- **The MPRIS slot is named `lightning`**, because that is the bus name the
  application owns and it differs from the snap name.
- **Updates come from snapd only.** `snap/local/patches/0001-update-disable-network-checks.patch`
  (adapted from the Debian package's `0003-update-disable-network-checks.patch`)
  makes `UpdateManager::startCheck()` refuse unconditionally, so the app never
  contacts an upstream update server or offers to replace the snap's files, and
  the Updates settings page shows a notice pointing at `snap refresh`. It is
  applied in `override-pull`, so a tag it no longer fits fails the build; rebase
  it when bumping `source-tag`. It is written against 0.10.2 and does not apply
  to v0.10.1. No update signing key is compiled in.
- `snap/local/check-payload` fails the build when a runtime-only dependency
  (Qt plugins, GStreamer call plugins, NSS modules, PipeWire client files,
  dictionaries) is missing from the payload, since a strictly confined snap
  cannot see the build host's copies.
