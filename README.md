# Fixing the ELAN Fingerprint Reader (04f3:0c4b) on Omarchy / Arch Linux

If your laptop has an ELAN `04f3:0c4b` fingerprint reader (common on
several Lenovo ThinkPad E14/E15 Gen 3 and Gen 4 models) and fingerprint
**enrollment works but verification never matches** — `fprintd-verify`
always fails, Bozorth3 match score is exactly `0` every single time, no
matter what you try — this repo documents a working fix.

This was reproduced and fixed on a **ThinkPad E15 Gen 3 (AMD), running
[Omarchy](https://omarchy.org)**, but the same USB ID appears on other
ThinkPad E14/E15 Gen 3/Gen 4 models, so the fix likely applies there too.

## The problem

Omarchy's built-in fingerprint setup (`omarchy-setup-security-fingerprint`)
installs the standard, open-source `libfprint`, whose `elan.c` driver
treats this sensor as a **swipe** sensor: it captures raw frames,
assembles them into an image, and matches with the NBIS/Bozorth3
algorithm entirely in software.

For this specific hardware, that path is broken: enrollment completes,
but verification never succeeds. Across 25+ verify attempts under every
combination of finger orientation, swipe speed, and technique, the match
score was always **exactly `0`** — never even a partial match. This
looks like the sensor's physical/firmware behavior doesn't line up with
what the open driver expects (see "Why this happens" below).

## The fix

Lenovo ships a proprietary driver for this sensor (used on Windows via a
match-on-chip-style "press" flow, not swipe) as a `libfprint-tod`
("Touch OEM Driver") plugin. Switching to it — instead of patching the
open driver — is what actually works:

```
Broken:  04f3:0c4b → elan.c (open) → swipe capture → NBIS/Bozorth3 → always 0
Working: 04f3:0c4b → libfprint-tod → Lenovo's proprietary driver → treated as "press"
```

After switching, `fprintd` reports the device with `scan-type: press`
instead of `swipe`, and verification works reliably (10/10 in testing,
zero failures, across `sudo`, polkit, and the screen lock).

### Why this happens

`libfprint`'s driver interface (`probe`, `open`, `enroll`, `verify`,
`identify`, `capture`, ...) has no way to express "this sensor is
actually a press/area sensor with proprietary matching, not a swipe
sensor with software matching" other than picking the right driver
entirely. The generic `elan.c` driver is written for the *swipe* family
of ELAN sensors. This particular PID appears to be closer to a
press/area sensor in practice — Lenovo's own Windows driver and the
proprietary Linux TOD plugin both treat it that way — but nothing in the
open driver's device table reflects that, so it's silently driven the
wrong way and matching just never works.

## Steps

**1. Install the required AUR packages, in this order:**

```bash
yay -S openssl-1.1              # runtime dependency of the proprietary driver
yay -S libfprint-tod-git        # replaces libfprint, adds TOD plugin support
yay -S libfprint-2-tod1-elan    # the ELAN TOD plugin — pulls Lenovo's official
                                 # binary from download.lenovo.com at build time
```

`libfprint-tod-git` conflicts with `libfprint` — `pacman`/`yay` will ask
to remove it, confirm.

> **On security:** `libfprint-2-tod1-elan` bundles the proprietary
> binary Lenovo ships for the E14/E15 Gen 4 (`libfprint-2-tod1-elan.so`,
> from `download.lenovo.com/pccbbs/mobiles/r1sle01w.zip`). Before
> trusting it, we downloaded that ZIP directly from Lenovo's own domain
> and verified the `.so` inside is **byte-for-byte identical** (SHA256
> match) to the one shipped in the AUR package's source repo, and
> checked its imported/exported symbols for anything unexpected (none
> found — only standard USB/crypto/glib libraries). We did not find any
> tampering.

**2. Restart `fprintd` and confirm the driver switched:**

```bash
sudo systemctl restart fprintd
gdbus call --system --dest net.reactivated.Fprint \
  --object-path /net/reactivated/Fprint/Device/0 \
  --method org.freedesktop.DBus.Properties.GetAll \
  net.reactivated.Fprint.Device
```

You should see `scan-type: press` and the device name change from
`ElanTech Fingerprint Sensor` to `ELAN Fingerprint Sensor`. No udev rule
forcing `LIBFPRINT_DRIVER` was needed in testing — the TOD driver took
priority automatically. (An older guide for this hardware suggests such
a rule; it may apply to a different `libfprint-tod` version, but check
your own `GetAll` output above before assuming you need it.)

**3. Enroll and verify:**

```bash
fprintd-enroll
fprintd-verify   # run it a few times to build confidence
```

Press firmly, covering the full sensor pad, and hold for about a
second — a partial/rushed touch can get rejected for insufficient
contact area (`fprintd` will log `finger area NN% < 75%` if you enable
debug logging) even though the sensor is working correctly.

**4. PAM (sudo / polkit / lock screen)**

If you already ran `omarchy-setup-security-fingerprint` before this fix,
your PAM stacks (`/etc/pam.d/sudo`, `/etc/pam.d/polkit-1`,
`/etc/pam.d/omarchy-lock-fingerprint`) are already wired to
`pam_fprintd.so` — nothing to change there. It talks to `fprintd` over
D-Bus regardless of which driver is loaded underneath, so it picks up
the fix automatically once step 2-3 above work.

If you haven't run it yet, run `omarchy-setup-security-fingerprint`
*after* confirming `fprintd-verify` works reliably (steps 1-3), not
before — Omarchy's script enrolls and verifies with whatever driver is
active, so getting the working driver in place first avoids a false
failure.

**5. Protect it from `omarchy update` (recommended)**

`omarchy update` refreshes AUR packages too. `libfprint-tod-git` in
particular is a VCS package with no pinned version, so a routine update
could silently rebuild it against a newer upstream commit — and
`fprintd` itself could get bumped to a version with an ABI mismatch
(this has been reported for some `fprintd`/`libfprint-tod` version
combinations, though not reproduced here). Freeze the packages you don't
want touched automatically:

```bash
# /etc/pacman.conf, under [options]
IgnorePkg = libfprint-tod-git libfprint-2-tod1-elan fprintd libfprint
```

Both `pacman -Syu` and `yay -Sua` (which is what `omarchy update` calls
under the hood for AUR packages) respect `IgnorePkg` — verified
empirically before relying on it.

Deliberately **not frozen**: `openssl-1.1` — its AUR maintainer actively
backports current CVE patches, and the `1.1.1` branch has a stable ABI
for its whole lifecycle, so there's no compatibility risk in letting it
update, only security benefit.

If you ever want to update the frozen packages on purpose, remove the
`IgnorePkg` line, update, then re-run the enroll/verify smoke test
before re-adding it.

## Known limitation

With the open `elan.c` driver, the sensor's LED blinks green while
waiting for a touch. With the proprietary TOD driver, it doesn't.
`libfprint`'s driver interface has no concept of LED/indicator control
at all (checked the full `FpDeviceClass` vtable — nothing there), so
this isn't something the open-source `libfprint-tod` wrapper could
expose even if it wanted to. The proprietary binary does export a
function called `efd_state_indicator`, suggesting Lenovo's own SDK
handles this internally, but it's tied into an undocumented internal
state machine (confirmed via disassembly) — not something callable in
isolation without significant reverse engineering. Purely cosmetic, not
pursued.

## Tested on

- ThinkPad E15 Gen 3 (AMD), Omarchy, `04f3:0c4b`, kernel 7.1.8-arch1
- `fprintd 1.94.5-2`, `libfprint-tod-git 1.95.1+tod1-1`,
  `libfprint-2-tod1-elan 0.0.1-2`, `openssl-1.1 1.1.1.w-11`

If it works (or doesn't) on a different E14/E15 Gen 3/Gen 4 model,
please open an issue — that data point is useful for anyone else with
this hardware.

## Related evidence (same PID, other distros)

- ThinkPad E15 Gen 3, same `04f3:0c4b`, resolved on Fedora 43 by
  installing `libfprint-2-tod1-elan-0c4b` (enrollment demonstrated;
  no published verify-match output).
- A June 2026 Arch + Hyprland guide for this exact PID, using
  `libfprint-tod-git 1.95.1+tod1`, reporting success with
  `fprintd-verify`, `sudo`, login, and `hyprlock`.
- Canonical/Launchpad documented that `04f3:0c4b` needs ELAN's
  proprietary MOH driver, verified on ThinkPad E14 Gen 4 and ThinkBook
  14 Gen 4.
- Lenovo's official driver page for ThinkPad E14/E15 Gen 4 declares
  support for this PID (`libfprint-2-tod1-elan 0.0.8`,
  `download.lenovo.com/pccbbs/mobiles/r1sle01w.zip`).

## License

This documentation is released under CC0 — copy, adapt, and reuse
freely. The proprietary driver binary referenced here remains Lenovo's
property under its own license; this repo does not redistribute it, it
only documents how the existing AUR packaging fetches and uses it.
