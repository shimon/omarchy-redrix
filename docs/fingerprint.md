# Fingerprint reader (experimental)

REDRIX has Google's fingerprint MCU (FPMCU): a `bloonchipper` board (STM32F412) with an FPC1025 sensor on SPI,
exposed as `/dev/cros_fp`. Matching happens **on the MCU**: fingerprint images never reach Linux. The MCU encrypts
each template it creates and hands it to the host; `fprintd` stores it in `/var/lib/fprint/` and uploads it again
before each match.

Arch's `libfprint` has no driver for it. The `crfpmoc` driver in libfprint
[MR !512](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/512) does, but as of November 2026
the MR is unmerged and needed three fixes to work here. `packages/libfprint-crfpmoc` is Arch's libfprint 1.94.100
plus MR !512 plus those fixes.

Tested on one unit: enroll, verify (enrolled finger matches, other finger rejected), sudo, polkit and the Omarchy lock
screen, including after reboots.

## Setup

```sh
cd packages/libfprint-crfpmoc && makepkg -si && cd -   # replaces libfprint; builds and runs 130 tests (~1 min)
sudo pacman -S --needed fprintd
sudo systemctl stop fprintd                             # so it reloads the new library
fprintd-enroll                                          # touch and lift until enroll-completed
fprintd-verify                                          # enrolled finger: verify-match; another: verify-no-match
sudo scripts/enable-fingerprint-auth                    # sudo, polkit and lock screen
```

Reboot once afterwards and check the lock screen still accepts your finger.

**Do not run Omarchy's `omarchy-setup-security-fingerprint`.** It looks for USB readers only, so it reports
"No fingerprint sensor detected"; and it installs Omarchy's `libfprint-git`, which has no ChromeOS driver and would
replace this one. `scripts/enable-fingerprint-auth` makes the same PAM changes it would (including its rule that skips
the reader when the lid is closed), so Omarchy's `omarchy-remove-security-fingerprint` still undoes them. That
script also uninstalls `fprintd`.

If you installed an earlier build of this package (before pkgrel 4), your enrolled print is encrypted with a key
that no longer exists: install pkgrel 4, run `fprintd-delete "$USER"`, **reboot** (the MCU accepts a new key only
after a reset), then enroll again.

## The three fixes

1. **Enroll fails at once with `enroll-unknown-error`.** The driver sends `EC_CMD_FP_MODE` with no room for its
   4-byte reply in four places. `cros-ec-spi` rejects the oversized reply (`packet too long (4 bytes, expected 0)`,
   `EMSGSIZE`). The fix requests `sizeof (struct ec_response_fp_mode)` everywhere.
2. **fprintd crashes when the sensor reports its first event** (assertion `FP_IS_DEVICE (device)`, then SIGSEGV; the
   client just waits). `crfpmoc_read_bytes` was a `GSourceFunc (gpointer)` attached to a pollable source, which
   calls `GPollableSourceFunc (GObject *stream, gpointer user_data)`. It received the stream as its state machine.
3. **Enrolled fingers stop working after a reboot** (`command failed: UNAVAILABLE`, then "fingerprint reader not
   available" on the lock screen). The MCU derives the template key from a seed it accepts once per MCU boot, and
   the driver generated a new random seed every time fprintd started. Templates enrolled under one seed can't be
   decrypted under the next. ChromeOS keeps its seed in the TPM; the fix generates the seed and user context once
   and keeps them in `/var/lib/fprint/crfpmoc-keys` (mode 0600, root-only, next to the templates themselves).

## Security notes

- Templates are encrypted by the MCU, but the key material that unlocks them (`crfpmoc-keys`) sits on the same disk.
  Anyone with root, or with your disk unencrypted, can use your enrolled templates on this machine. Use full-disk
  encryption.
- `pam_fprintd` is `sufficient`: a fingerprint replaces the password for sudo, polkit and the lock screen. The
  password still works as a fallback everywhere.
- This is a pre-release driver with local fixes. Keep a password you can type.

## Debugging

`fprintd-enroll`/`fprintd-verify` are thin clients; the logs come from the daemon:

```sh
sudo mkdir -p /etc/systemd/system/fprintd.service.d
printf '[Service]\nEnvironment=G_MESSAGES_DEBUG=all\n' | sudo tee /etc/systemd/system/fprintd.service.d/debug.conf
sudo systemctl daemon-reload && sudo systemctl restart fprintd
journalctl -fu fprintd
```

Remove `debug.conf` afterwards. Don't use `systemctl revert fprintd`: it also deletes Omarchy's
`10-stop-timeout.conf` in the same directory. The driver logs commands and MCU events, not images.

## Updating

This package replaces `libfprint` and won't update with `pacman -Syu`. Rebuild when Arch updates libfprint (bump
the base commit and regenerate the patch), and switch back to the stock package once a libfprint release ships the
crfpmoc driver: `sudo pacman -S libfprint` (enrolled prints may then need re-enrolling).
