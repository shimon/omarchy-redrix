# omarchy-redrix

**Omarchy Linux (Arch + Hyprland) on the HP Elite Dragonfly Chromebook (board `redrix`, Intel Alder Lake, Google "brya"
family) running MrChromebox UEFI firmware: what was broken, why, and pacman packages that fix it.**

**New to this laptop?** [How to get the best value Omarchy machine ever](docs/best-value-omarchy-machine.md) covers
buying a used one, unlocking and replacing the firmware, installing Omarchy, and then using this repo.

Tested on one machine: i5-1245U, 16 GB, OV5675 camera, MrChromebox 2603.2, Omarchy with kernel 7.2, November 2026.

For components outside Omarchy/Arch packaging (Google's speaker DSM files, S3 suspend, haptics, pen,
a chart-calibrated tuning for the Hi-556 camera variant), see
**[maximeallanic/redrix-linux](https://github.com/maximeallanic/redrix-linux)** (Debian/GNOME-focused). This repo
covers the Omarchy side and a few findings that apply to any distro.

## Status at a glance

| Area | Out of the box on Omarchy | With this repo |
|---|---|---|
| Sustained performance | thermald applies ChromeOS DPTF tables: PL1 held at 13 W, ~1.53 GHz all-core | `redrix-thermald`: PL1 15 W, ~1.75-1.8 GHz all-core, package ~71 C |
| Speakers, headphones, mics | "Dummy Output" only (and the usual audio script refuses to run on Omarchy) | `chromebook-audio-adl`: all outputs and the 4-mic array, amp current limits applied |
| Webcam (OV5675, IPU6) | Not visible to apps | `redrix-camera`: works in PipeWire and browsers, ~5 % of one core (GPU ISP) |
| Webcam colour | Flat, hazy, green-cyan | `redrix-camera` ships a hand-tuned OV5675 profile |
| Webcam in Chrome/Chromium | No picture | `user-config/apply`: `WebRtcPipeWireCamera` flag (+ portal permission fix) |
| Video playback | CPU decoding in Chrome/Chromium | `user-config/apply`: VA-API flags (verified on the GPU video engine) |
| Fullscreen / Overview keys | Do nothing | `user-config/apply`: fullscreen; app grid; Shift+Overview = Omarchy menu |
| Battery longevity | Charges to 100 % | `redrix-charge-limit` (opt-in): stop at 80 %, resume below 75 % |
| Brightness, volume, mute, mic mute, kbd backlight, play/pause, screenshot keys | Work (Omarchy bindings) | - |
| Touchpad, touchscreen, Wi-Fi, Bluetooth, GPU (i915 + GuC/HuC) | Work | - |
| SD card reader (Genesys GL9755) | Works (`sdhci-pci`) | - |
| Suspend / resume (s2idle) | Works: lid close/open, 99.6 % of a 6.6-minute suspend in hardware sleep (S0ix), all devices restored. The power-button LED goes **off** while suspended | - |
| Power profiles | Omarchy already switches performance (AC) / balanced (battery) | - |
| Lock key | **Suspends immediately** (sends `KEY_SLEEP`) | Documented; [redrix-linux](https://github.com/maximeallanic/redrix-linux) has a fix |
| Fingerprint reader | Not supported by Arch's libfprint; Omarchy's fingerprint setup can't see it | `libfprint-crfpmoc` (experimental) + `scripts/enable-fingerprint-auth`: sudo, polkit and lock screen, survives reboots |

## Quick start

```sh
git clone https://github.com/shimon/omarchy-redrix
cd omarchy-redrix
scripts/diagnose                 # read-only: what is wrong on this machine, with hints
```

Then build and install the packages you want (each is independent):

```sh
cd packages/chromebook-audio-adl && makepkg -si && cd -   # audio; reboot afterwards for the mics
cd packages/redrix-camera        && makepkg -si && cd -   # webcam
cd packages/redrix-thermald      && makepkg -si && cd -   # full 15 W sustained power limit
cd packages/redrix-charge-limit  && makepkg -si && cd -   # optional: 75-80 % charge window
cd packages/libfprint-crfpmoc    && makepkg -si && cd -   # optional, experimental: fingerprint (see docs/fingerprint.md)
user-config/apply --dry-run      # browser flags + Hyprland bindings; drop --dry-run to apply
```

Remove any of them with `sudo pacman -R <package>`; each puts things back (the charge limit returns to 100 %,
thermald to its default mode). `user-config/apply` backs up every file it edits as `<file>.bak.<timestamp>`.

## Findings

### 1. thermald throttles to ChromeOS reference limits

Omarchy installs thermald on Intel laptops (`--adaptive`). On REDRIX it exports coreboot's DPTF passive trips into
`/run/thermald/thermal-conf.xml.auto`: CPU power is cut when the **DDR sensor passes 45 C**, values coreboot
marks as "initial reference values only". Under sustained load PL1 sits at 13 W while the package is ~64 C, far
from its 97 C limit. **Dropping `--adaptive` is not enough**: thermald still exports the ACPI tables unless it is
given its own config file. `redrix-thermald` gives it one that only guards the CPU package at 90 C. Measured with
12 busy threads on PD power:

| | sustained all-core | package | fans |
|---|---|---|---|
| thermald DPTF (13 W) | ~1.53 GHz | ~64 C | ~3.9k / 4.3k rpm |
| redrix-thermald (15 W) | ~1.75-1.8 GHz | ~71 C | EC curve, starts later |

Bursts (PL2 55 W) are the same either way. If you prefer the cooler, quieter ChromeOS behaviour, don't install it.

### 2. Audio: Omarchy is refused by the usual script

The card (`sof-rt5682`: four MAX98390 amps, RT5682S headset codec) loads, but without ChromeOS UCM profiles PipeWire
shows only "Dummy Output". [chromebook-linux-audio](https://github.com/WeirdTreeThing/chromebook-linux-audio) is the
standard fix, but `setup-audio` exits on Omarchy ("Use a real distro"). `chromebook-audio-adl` packages what it does
on Alder Lake, pinned to reviewed commits: the UCM profiles from
[alsa-ucm-conf-cros](https://github.com/WeirdTreeThing/alsa-ucm-conf-cros), `dmic_num=4` for the internal mics (the
firmware has no NHLT table), and two WirePlumber settings.

The REDRIX profile caps each amp's digital volume and sets a **2.2 A current limit** (the default is 4.1 A), but
only from its UCM *BootSequence*, which runs on `alsactl init`. The package runs `alsactl init` and
`alsactl store` once on install. The amps still run without Google's DSM speaker model
(`dsm_param_*_Google_Redrix.bin`); redrix-linux's `speakers` component extracts those from the recovery image.

### 3. Webcam: libcamera, a dma-heap permission, and two browser gotchas

The OV5675 sits behind the IPU6 input system; the kernel binds it, but apps need libcamera's software ISP
(`pipewire-libcamera`, `libcamera-ipa`), and the ISP needs `/dev/dma_heap/system`, which is root-only by default.
`redrix-camera` adds both. The ISP runs on the GPU (EGL): 720p30 costs ~5 % of one core.

Chrome and Chromium then still show no picture unless:

- the **`WebRtcPipeWireCamera`** feature is on. Otherwise they open the ~30 raw IPU6 `/dev/video*` nodes, which only
  carry Bayer data. Chromium honours only the **last** `--enable-features=` line in its flags file, so features
  must be merged into one line. Google Chrome reads `~/.config/chrome-flags.conf`, separate from Chromium's.
- the **camera portal** has not stored a denial for the browser. Ours had `com.google.Chrome -> no`, probably from a
  dismissed permission dialog. `scripts/diagnose` reports it and `user-config/apply --fix-portal` resets it.

### 4. Webcam colour

Without `ov5675.yaml`, libcamera uses `uncalibrated.yaml`: grey-world white balance with no colour matrix. The
result is flat and, on this sensor, consistently 3-7 % too green on white surfaces. The shipped profile adds a
luma-preserving 1.5x saturation, per-channel gains that cancel the green/cyan bias, a 0.78 scale (exposure is
metered before the matrix, so this darkens the output) and a higher black level for contrast. It was tuned by eye
over five A/B rounds, not against a colour chart: see [docs/camera-tuning.md](docs/camera-tuning.md) to retune it
for your room or taste.

### 5. Top row

Full scancode/keycode map in [docs/top-row.md](docs/top-row.md). Most keys already hit Omarchy bindings. Fullscreen
(`XF86FullScreen`) and Overview (`XF86LaunchA`) do nothing until bound, and the lock key sends `KEY_SLEEP`, which
systemd-logind turns into an immediate suspend.

### 6. Fingerprint: an unmerged driver and three fixes

The fingerprint MCU (`/dev/cros_fp`, FPC1025 sensor) matches on-chip, ChromeOS-style. Arch's libfprint has no driver
for it; libfprint [MR !512](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/512) (`crfpmoc`) does,
but on REDRIX it failed to enroll, then crashed fprintd, then lost every enrolled finger on reboot (it set a new
random encryption seed on each start). `libfprint-crfpmoc` is Arch's libfprint plus MR !512 plus fixes for all three.
Omarchy's `omarchy-setup-security-fingerprint` must not be used here: it only detects USB readers and would replace
the driver with its own `libfprint-git`. `scripts/enable-fingerprint-auth` makes the same PAM changes instead. Setup,
details and security notes: [docs/fingerprint.md](docs/fingerprint.md).

## Troubleshooting

**Battery draining, or very slow, while plugged in? Try the other USB-C port first.** On our unit the **right** port
(`CROS_USBPD_CHARGER1`) never negotiates USB Power Delivery with any orientation of the plug: it gets only 5 V /
0.5 A, so the battery drains and the EC holds the CPU at its minimum clock (~270-400 MHz). The same charger and cable
in the **left** port (`CROS_USBPD_CHARGER0`) get 19.4 V / 3 A within two seconds. We have not yet established whether
this is a hardware fault on our unit or the custom EC firmware; reports from other REDRIX owners are welcome.
`scripts/diagnose` flags a non-PD charger and names the port. A 5 V USB-A charger or adapter causes the same symptom
on either port; use a 45 W+ USB-C PD charger.

## Repository layout

| Path | What |
|---|---|
| `scripts/diagnose` | Read-only checker for everything above, with a hint for each failure |
| `user-config/apply` | Per-user settings: browser flags, Hyprland key bindings, optional portal fix |
| `packages/chromebook-audio-adl` | UCM profiles, DMIC count, WirePlumber config (BSD-3 upstream files, fetched at pinned commits) |
| `packages/redrix-camera` | libcamera/PipeWire dependencies, dma-heap udev rule, OV5675 tuning |
| `packages/redrix-thermald` | thermald drop-in and package-only config |
| `packages/redrix-charge-limit` | Opt-in charge window via `cros-charge-control` |
| `packages/libfprint-crfpmoc` | libfprint with the ChromeOS fingerprint driver (MR !512 + REDRIX fixes, LGPL patch) |
| `scripts/enable-fingerprint-auth` | Fingerprint for sudo, polkit and the lock screen (run as root after enrolling) |
| `docs/` | Buying-to-working guide, top-row map, camera tuning method, fingerprint |

## Contributing

Reports from other REDRIX units (especially the Hi-556 camera variant and i7 models) are welcome: open an issue with
the output of `scripts/diagnose --no-color`. Hardware fixes that are not Omarchy-specific are better contributed to
[redrix-linux](https://github.com/maximeallanic/redrix-linux), so every distro benefits.

## Credits and licence

MIT, see [LICENSE](LICENSE). Third-party files are fetched at build time, not stored here; see
[NOTICE.md](NOTICE.md). Thanks to WeirdTreeThing (audio UCM and configs), MrChromebox (firmware), the chrultrabook
community, the libcamera developers, and the libfprint developers and MR !512's authors (Abhinav Baid,
Marco Trevisan and others).
