# Third-party components

Apart from the libfprint patch below, no third-party files are stored in this repository.
`packages/chromebook-audio-adl` fetches them at build time, pinned to the commits below, and installs their licences under `/usr/share/licenses/chromebook-audio-adl/`.

| Project | Commit | Licence | Used for |
|---|---|---|---|
| [WeirdTreeThing/alsa-ucm-conf-cros](https://github.com/WeirdTreeThing/alsa-ucm-conf-cros) | `109a49349d5a9cd1a3a3e42c88da67453bc35ce9` | BSD-3-Clause | ChromeOS-derived ALSA UCM profiles (`ucm2/`) |
| [WeirdTreeThing/chromebook-linux-audio](https://github.com/WeirdTreeThing/chromebook-linux-audio) | `84b700e271145df82ae71d0497af11705a2bb166` | BSD-3-Clause | `conf/sof/adl-sof.conf`, `conf/common/51-*.conf` |

The camera udev rule mirrors one shipped by [Omarchy](https://github.com/omacom/omarchy) (MIT). The OV5675 tuning
file's structure follows libcamera's `uncalibrated.yaml` (CC0-1.0).

`packages/libfprint-crfpmoc/crfpmoc-v1.94.100.patch` is a patch to
[libfprint](https://gitlab.freedesktop.org/libfprint/libfprint) (LGPL-2.1-or-later) and is distributed under that
licence, not MIT. It contains the commits of libfprint
[MR !512](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/512) (authors include Abhinav Baid and
Marco Trevisan) rebased onto v1.94.100 (`80a4b5ec612892c5c056c48dddaa561452cf37ec`), plus three fixes from this
repository, also LGPL-2.1-or-later.
