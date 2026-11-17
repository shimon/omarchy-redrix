# How to get the best value Omarchy machine ever

A used **HP Elite Dragonfly Chromebook** costs a fraction of what it sold for new. Once you replace its firmware, it's
an ordinary x86 laptop with:

- a 12th-gen Intel Core (Alder Lake) processor and 16 GB of RAM
- a 13.5" 3:2 touchscreen at 2256x1504
- a pen digitizer, a fingerprint reader and a webcam
- an SD card reader and USB-C charging

This guide gets you from an eBay listing to a fully working [Omarchy](https://omarchy.org/) install. It mostly points
to the authoritative guides for each step, since those stay current. The last step uses this repository to get the
audio, camera, fingerprint reader and thermals working.

> **Risk:** replacing the firmware is safe if you follow the guides, and it's reversible if you keep a backup. But
> you can make the laptop unbootable, and ChromeOS will be gone. Read each linked page fully before running anything.

## What it costs

| Item | Approximate cost (November 2026) |
|---|---|
| Used HP Elite Dragonfly Chromebook (i5-1245U, 16 GB, 256 GB) | ~$120–300 used, ~$270–400 refurbished |
| SuzyQ debug board ([Fyra Labs](https://shop.fyralabs.com/products/suzyq-board)) | ~$12.50 plus a USB-C data cable |
| USB stick, 8 GB or larger | probably in a drawer |

Prices are from asking prices on [eBay](https://www.ebay.com/sch/i.html?_nkw=hp+elite+dragonfly+chromebook),
[Back Market](https://www.backmarket.com/en-us/p/hp-dragonfly-chromebook-core-i5-1245u-33-ghz-256-gb-ssd-16-gb/18e1a1e9-12de-4674-bb8e-0d7bc8a630a7)
and refurbishers. Check eBay's *sold* listings for what people actually pay.

## 1. Buy the right one

Search for **"HP Elite Dragonfly Chromebook"**. The internal board name is `REDRIX`. Don't confuse it with the HP
Elite Dragonfly (non-Chromebook) Windows laptops, which are a different machine.

Check these before buying:

- **It must not be enterprise-enrolled ("managed").** Many come from corporate fleets. A device still enrolled by an
  organisation has its debug interface locked, so **none of the firmware steps below will work**. Ask the seller to
  confirm it was deprovisioned, or for a photo of the ChromeOS sign-in screen with no "This device is managed by…"
  notice. Avoid listings that say "managed", "enrolled" or "locked".
- **RAM:** 16 GB is common. The RAM can't be upgraded, so avoid 8 GB units unless they're very cheap.
- **Charger:** any 45 W or more USB-C Power Delivery charger works. If none is included, budget for one.
- **Battery health and screen condition:** ask, as with any used laptop.
- **Camera:** units ship with either an OV5675 or a Hi-556 camera sensor. Both work with this repository, but its
  colour tuning is for the OV5675. [redrix-linux](https://github.com/maximeallanic/redrix-linux) has a tuning for the Hi-556.

## 2. Get the tools

- **A SuzyQ cable or board.** This is Google's closed-case debugging (CCD) cable, which lets you turn off the
  firmware write protection without opening the laptop.
  - Google's original [SparkFun SuzyQable](https://www.sparkfun.com/products/14746) is retired.
  - [Fyra Labs' SuzyQ board](https://shop.fyralabs.com/products/suzyq-board) is the current equivalent
    ([docs](https://developer.fyralabs.com/suzyqboard)). Check stock.
  - The chrultrabook forum has a thread on
    [alternatives](https://forum.chrultrabook.com/t/alternative-to-suzyqable/1132).
- **A USB stick of 8 GB or more** for the Omarchy installer.
- **A USB keyboard is optional.** Omarchy's disk-encryption password prompt needs a wired or 2.4 GHz keyboard, but the
  built-in keyboard works.

The [MrChromebox supported-devices list](https://docs.mrchromebox.tech/docs/supported-devices.html) lists REDRIX under
Intel Alderlake, with full UEFI firmware supported. Its write-protect method is listed as **CR50 (SuzyQ)**, which is why
you need the cable.

## 3. Enter Developer Mode

Follow MrChromebox's [Developer Mode guide](https://docs.mrchromebox.tech/docs/boot-modes/developer.html). In short:
hold **Esc + Refresh**, press **Power**, then press **Ctrl+D** at the recovery screen. This wipes the device. Let it
finish, and then **don't sign in**: the steps below run from the login screen's VT2 console (**Ctrl+Alt+F2**, user
`chronos`, no password).

## 4. Disable firmware write protection with the SuzyQ

Follow MrChromebox's
[Disabling Firmware Write Protection: Using CCD with a SuzyQable](https://docs.mrchromebox.tech/docs/firmware/wp/disabling.html#using-closed-case-debugging-ccd-using-a-suzyqable).
The outline:

1. **Open CCD:** run `sudo gsctool -a -o` in VT2, and press the power button each time it asks. This takes up to
   about **five minutes**, so be patient. If you press at the wrong moment, start again. The laptop then reboots out
   of Developer Mode. Re-enable it and return to VT2.
2. **Plug in the SuzyQ** with the USB-C end in the laptop's CCD port, which is usually the left one. Check that
   `ls /dev/ttyUSB*` shows `ttyUSB0`–`ttyUSB2`. If it doesn't, flip the USB-C plug or try the other port. The
   debug connection only works one way up.
3. **Disable write protection and reset CCD** with `wp disable`, `wp disable atboot` and `ccd reset factory`, sent to
   `/dev/ttyUSB0`. The factory reset keeps the CCD unlocked, so you can recover over the SuzyQ later without
   opening the case.
4. **Verify** with `sudo gsctool -a -w`, which should report `forced disabled`, and `crossystem wpsw_cur`, which
   should print `0`.

## 5. Flash MrChromebox's UEFI firmware

Still in VT2, run MrChromebox's
[Firmware Utility Script](https://docs.mrchromebox.tech/docs/fwscript.html):

```sh
cd; curl -LOf https://mrchromebox.tech/firmware-util.sh && sudo bash firmware-util.sh
```

1. **Back up the stock firmware to a USB stick first**, using the script's backup option, and copy that file
   somewhere safe. It's your way back to ChromeOS.
2. Choose **Install/Update UEFI (Full ROM) Firmware**.
3. Reboot. You now have a normal UEFI PC. ChromeOS is gone.

If something goes wrong, see MrChromebox's [unbricking guides](https://docs.mrchromebox.tech/docs/support/unbricking/).
The [SuzyQ unbrick guide](https://docs.mrchromebox.tech/docs/support/unbricking/unbrick-suzyq.html) works because of the
`ccd reset factory` in step 4. MrChromebox supports the firmware but not any particular operating system: Linux help
lives on the [chrultrabook forum](https://forum.chrultrabook.com/) and in the [chrultrabook docs](https://docs.chrultrabook.com/).

## 6. Install Omarchy

Follow the Omarchy manual's [Getting Started](https://learn.omacom.io/2/the-omarchy-manual/50/getting-started) page:

1. Download the ISO from [omarchy.org](https://omarchy.org/).
2. Write it to the USB stick with [balenaEtcher](https://etcher.balena.io/) on Mac or Windows, or
   [caligula](https://github.com/ifd3f/caligula) on Linux.
3. Boot from the stick. Press **Esc** at the MrChromebox splash screen for the boot menu.
4. Run the installer. It erases the internal drive. Keep the default full-disk encryption, which also protects the
   fingerprint data.

Omarchy's help channel is on its [Discord](https://omarchy.org/discord).

## 7. Make all the hardware work (this repository)

Out of the box, Omarchy on this laptop has:

- no sound
- a webcam that apps can't see
- a CPU held back by ChromeOS thermal limits
- a fingerprint reader nothing can use

This repository fixes all of these:

```sh
git clone https://github.com/shimon/omarchy-redrix && cd omarchy-redrix
scripts/diagnose                                         # read-only: what's wrong, with a fix for each item
cd packages/chromebook-audio-adl && makepkg -si && cd -  # speakers, headphones, mics (reboot afterwards)
cd packages/redrix-camera        && makepkg -si && cd -  # webcam, with colour tuning
cd packages/redrix-thermald      && makepkg -si && cd -  # full 15 W sustained CPU power
cd packages/redrix-charge-limit  && makepkg -si && cd -  # optional: stop charging at 80 % to save the battery
user-config/apply                                        # webcam + hardware video in Chrome/Chromium, top-row keys
```

The fingerprint reader is experimental and takes a few more steps. They're in
[docs/fingerprint.md](fingerprint.md) and come down to building `packages/libfprint-crfpmoc`, enrolling a finger
and running `sudo scripts/enable-fingerprint-auth`. **Don't** use Omarchy's own fingerprint setup on this machine:
it can't detect this reader and would replace the driver.

Run `scripts/diagnose` again at the end. The [README](../README.md) explains what each package fixes and why.

## What to expect afterwards

- **Works:**
  - performance close to a new laptop's for everyday work
  - suspend/resume (the power LED goes off while asleep)
  - touchscreen, touchpad, Wi-Fi, Bluetooth, the SD card reader
  - hardware video decoding in the browser
  - fingerprint unlock for sudo and the lock screen
- **Quirks:**
  - The top-row **lock key suspends immediately**: it sends `KEY_SLEEP`, which systemd treats as "suspend now".
  - The speakers run without Google's tuned speaker model, which
    [redrix-linux](https://github.com/maximeallanic/redrix-linux) can extract.
  - On our unit the **right USB-C port only trickle-charges**. If the battery drains while plugged in, use the
    left port.
- **The top row** is mapped in [docs/top-row.md](top-row.md).

Found something different on your unit? Open an issue with the output of `scripts/diagnose --no-color`.
