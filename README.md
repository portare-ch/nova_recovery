# Retroid Pocket Nova recovery kit

Tested recovery through Qualcomm Emergency Download mode (EDL/9008) for a
bricked **Retroid Pocket Nova**, including a broken bootloader (ABL) or broken
Android installation. EDL can be entered without a working ABL or Android.
Verified on 2026-10-23.

**These instructions and recovery files are for the Retroid Pocket Nova only.**

If Android itself is broken, EDL provides a recovery entry point, but repairing
Android requires firmware and flashing instructions specifically for the
Retroid Pocket Nova. This kit does not include an Android factory image.

## Getting the code

`edl/` is a Git submodule pinned to the version used for this recovery kit.
Clone the kit and initialise the submodule:

```bash
git clone https://github.com/portare-ch/nova_recovery.git
cd nova_recovery
git submodule update --init edl
```

For an existing clone, run `git submodule update --init edl` from its root.

The commands below use a Unix shell. Install Python 3.12 and libusb first
(on macOS: `brew install python@3.12 libusb`). See
[edl's installation instructions](https://github.com/bkerler/edl/blob/1cda1a616748d608667b2d13b73a28d3197ceb45/README.md#installation) for other hosts'
USB drivers and permissions. From the recovery-kit directory, create the
Python environment used below:

```bash
python3.12 -m venv .venv
.venv/bin/python -m pip install -r edl/requirements.txt
```

## EDL programmer

**Provenance of the programmer.** It's byte-identical to `xbl_s_devprg_ns.melf` from
<https://github.com/strongtz/linux-next/releases/tag/odin2-release>, the
AYN Odin 2 community's SM8550 programmer, and to `loaders/rp6_loader.elf` in
<https://github.com/nibiru5/Retroid-Pocket-6_12gb_EDL-Factory-Image> (commit
`ea1a87d`), used for an RP6. Its signing chain is a Xiaomi one, while the
Nova's own XBL is signed with Qualcomm's SecTools test root. It loads because
these devices don't enforce secure boot: in Sahara the Nova reports
`OEM PKHASH : 00000000`, MSM HWID `0x2520e1`, and serial `0x4372c1217d1`.

**Proven on this Nova on 2026-10-23:**

- Holding VOL UP + VOL DOWN while inserting a USB cable reached EDL.
- The programmer loaded, with no authorisation asked for.
- `printgpt` on LUN 4 worked.
- `abl_a` could be read.
- `abl_b` was written and read back with a matching hash.
- `reset` left EDL.

## Entering EDL (9008)

Power the Retroid Pocket Nova off. **Hold VOL UP + VOL DOWN together, then
insert a USB cable connected to your computer while holding both buttons.**
The screen stays black in EDL. This works even when ABL or Android is broken.

On macOS, check that the computer sees it:

```bash
system_profiler SPUSBDataType | grep -i -A3 "QUSB\|9008\|05c6"
```

If Portare BIOS still works, you can also hold Volume Down at power-on and
select **Emergency download mode**, then confirm.

To leave EDL, run the following from the recovery-kit directory, or hold Power
for about 15 seconds:

```bash
.venv/bin/python edl/edl.py reset --loader=loader/rp6_loader.elf
```

## Read-only check (do this first)

Run these commands from the recovery-kit directory:

```bash
cd /path/to/nova_recovery
.venv/bin/python edl/edl.py printgpt --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
.venv/bin/python edl/edl.py r abl_a /tmp/abl_a-edl.img --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
shasum -a 256 /tmp/abl_a-edl.img backup/abl_a.img
```

`abl_a`/`abl_b` are on UFS LUN 4 (`/dev/sde` in Linux).
Confirm that `printgpt` lists both partitions and that each is 1 MiB before
writing anything. If either is missing or has a different size, stop.
The hashes may differ if the installed ABL is damaged or is a different version;
matching the supplied backup is not a prerequisite for recovery. After writing
the recovery image, its read-back hash must match.

Continue below to restore ABL. To finish without writing anything, run
`.venv/bin/python edl/edl.py reset --loader=loader/rp6_loader.elf`.

## Restoring the bootloader

The included recovery images contain
[Portare BIOS v1.4.2](https://github.com/portare-ch/BIOS/releases/tag/v1.4.2),
padded with zeros to the full 1 MiB partition size. Verify the included files
before flashing:

```bash
(cd backup && shasum -a 256 -c SHA256SUMS)
```

Write and verify both bootloader slots.
Run each command in order; stop on an error or a hash mismatch, and do not reset:

```bash
.venv/bin/python edl/edl.py w abl_a backup/abl_a.img --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
.venv/bin/python edl/edl.py r abl_a /tmp/abl_a-check.img --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
shasum -a 256 /tmp/abl_a-check.img backup/abl_a.img   # must match before continuing
.venv/bin/python edl/edl.py w abl_b backup/abl_b.img --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
.venv/bin/python edl/edl.py r abl_b /tmp/abl_b-check.img --memory=ufs --lun=4 --loader=loader/rp6_loader.elf
shasum -a 256 /tmp/abl_b-check.img backup/abl_b.img   # must match before reset
```

Once both read-back hashes match their recovery images, leave EDL:

```bash
.venv/bin/python edl/edl.py reset --loader=loader/rp6_loader.elf
```

With Portare BIOS installed, hold Volume Down at power-on to open its menu.
**Boot** lists systems on internal storage and the SD card, plus Android when
available. USB is for fastboot and EDL, not for booting a system.

This bootloader repair writes only `abl_a`/`abl_b`. It does not repair Android's
partitions; a broken Android installation needs matching Retroid Pocket Nova
firmware and its flashing instructions.
