# Third-party components

No third-party files are stored in this repository. `packages/chromebook-audio-adl` fetches them at build time,
pinned to the commits below, and installs their licences under `/usr/share/licenses/chromebook-audio-adl/`.

| Project | Commit | Licence | Used for |
|---|---|---|---|
| [WeirdTreeThing/alsa-ucm-conf-cros](https://github.com/WeirdTreeThing/alsa-ucm-conf-cros) | `109a49349d5a9cd1a3a3e42c88da67453bc35ce9` | BSD-3-Clause | ChromeOS-derived ALSA UCM profiles (`ucm2/`) |
| [WeirdTreeThing/chromebook-linux-audio](https://github.com/WeirdTreeThing/chromebook-linux-audio) | `84b700e271145df82ae71d0497af11705a2bb166` | BSD-3-Clause | `conf/sof/adl-sof.conf`, `conf/common/51-*.conf` |

The camera udev rule mirrors one shipped by [Omarchy](https://github.com/omacom/omarchy) (MIT). The OV5675 tuning
file's structure follows libcamera's `uncalibrated.yaml` (CC0-1.0).
