# Top-row keys on REDRIX

The internal keyboard is the firmware-emulated PS/2 device "AT Translated Set 2 keyboard" (`0001:0001`). Its
`function_row_physmap` (`/sys/devices/platform/i8042/serio0/function_row_physmap`) lists 13 ChromeOS "Vivaldi"
top-row keys. systemd's hwdb has no entry for REDRIX, so the kernel defaults apply. Captured with raw evdev reads,
left to right:

| # | Key | Scancode | Kernel keycode | XKB keysym | On Omarchy |
|---|---|---|---|---|---|
| 1 | Esc | 0x01 | `KEY_ESC` | Escape | Escape |
| 2 | Back | 0xea | `KEY_BACK` (158) | XF86Back | No binding; browsers and file managers go back |
| 3 | Refresh | 0xe7 | `KEY_REFRESH` (173) | XF86Reload | No binding; browsers reload |
| 4 | Fullscreen | 0x91 | `KEY_FULL_SCREEN` (372, = `KEY_ZOOM`) | XF86FullScreen | Nothing (bound by `user-config/apply`) |
| 5 | Overview | 0x92 | `KEY_SCALE` (120) | XF86LaunchA | Nothing (bound by `user-config/apply`) |
| 6 | Screenshot | 0x93 | `KEY_SYSRQ` (99) | Print | Omarchy screenshot |
| 7 | Brightness down | 0x94 | `KEY_BRIGHTNESSDOWN` | XF86MonBrightnessDown | Screen brightness |
| 8 | Brightness up | 0x95 | `KEY_BRIGHTNESSUP` | XF86MonBrightnessUp | Screen brightness |
| 9 | Keyboard backlight | 0x9e | `KEY_KBDILLUMTOGGLE` | XF86KbdLightOnOff | Cycles keyboard backlight |
| 10 | Play/pause | 0x9a | `KEY_PLAYPAUSE` | XF86AudioPlay | Play/pause |
| 11 | Mic mute | 0x9b | `KEY_MICMUTE` | XF86AudioMicMute | Mic mute |
| 12 | Mute | 0xa0 | `KEY_MUTE` | XF86AudioMute | Mute |
| 13 | Volume down | 0xae | `KEY_VOLUMEDOWN` | XF86AudioLowerVolume | Volume |
| 14 | Volume up | 0xb0 | `KEY_VOLUMEUP` | XF86AudioRaiseVolume | Volume |
| 15 | Lock | 0x5d | `KEY_SLEEP` (142) | XF86Sleep | **systemd-logind suspends immediately** (`HandleSuspendKey=suspend`) |

The power button arrives separately, on the ACPI "Power Button" device. Omarchy sets `HandlePowerKey=ignore`
and binds `XF86PowerOff` to its system menu.

## Bindings added by `user-config/apply`

```lua
o.bind("XF86FullScreen", "Full screen", hl.dsp.window.fullscreen({ mode = "fullscreen" }))
o.bind("XF86LaunchA", "Apps", "omarchy-shell omarchy.dock apps")
o.bind("SHIFT + XF86LaunchA", "Omarchy menu", { menu = "root" })
```

## No F1-F12

There are no function keys. The usual ChromeOS-style answer is a [keyd](https://github.com/rvaiya/keyd) layer on
Search (which arrives as Super/Meta): keys mapped inside keyd's `[meta]` layer are sent *without* Meta, so
Search+Back can send a plain F1 while every other Super shortcut keeps working. keyd names: `back refresh zoom scale
sysrq brightnessdown brightnessup kbdillumtoggle playpause micmute mute volumedown volumeup` map to F1-F13.

Two caveats if you do this on Omarchy:

- ChromeOS's Search+Backspace (Delete) and Search+arrows (Home/End/PgUp/PgDn) collide with Omarchy's
  Super+Backspace (transparency) and Super+arrows (window focus). Right Alt is a conflict-free alternative layer.
- keyd re-emits keys from a virtual keyboard that libinput does not treat as built-in, so the touchpad's
  disable-while-typing stops working until you add the libinput quirk from keyd's documentation.

This repo does not ship a keyd config (yet).
