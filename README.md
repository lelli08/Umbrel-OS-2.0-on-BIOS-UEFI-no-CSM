# Running umbrelOS 2.0 on a legacy-BIOS PC (no UEFI) with GRUB

How I got **umbrelOS 2.0** to boot natively on an **Acer Extensa 5620**, a 2007 laptop that only has a legacy BIOS (no UEFI, and no usable UEFI/CSM path), even though Umbrel officially supports UEFI only.

The trick in one sentence: write the official **disk image** straight to the internal drive from a Debian Live USB, then install a **legacy (i386-pc) GRUB** into the drive's MBR, pointing at the boot partition that the image already contains.

> **Unofficial and unsupported.** Umbrel's support told me umbrelOS is UEFI-only. This works for me, but it is a hack. Read [Known limitations](#known-limitations) before relying on it.

---

## Tested hardware and result

| | |
|---|---|
| Laptop | Acer Extensa 5620 (MS2205) |
| CPU | Intel Core 2 Duo T5450 (no VT-x) |
| RAM | 4 GB |
| Disk | 2.5" SATA SSD, 500 GB |
| Firmware | Legacy BIOS only (`/sys/firmware/efi` does not exist, SMBIOS 2.4) |
| Network | Ethernet cable |
| umbrelOS | 2.0 (Debian 13 "trixie" base, kernel 6.12, Rugix A/B updater) |

Result: at power-on you see the GRUB screen, then umbrelOS boots and the dashboard is reachable at `http://umbrel.local`.

## Why the normal install fails

- The official images (the `.img.xz` and the installer `.iso`) are built for **UEFI + GPT**. On a legacy BIOS you get a blinking cursor in the top-left corner and nothing else.
- Ventoy (with or without Super GRUB2 Disk), Rufus MBR/BIOS modes and the Clover bootloader did not help.
- The `curl https://umbrel.sh | bash` script no longer installs on top of Debian/Ubuntu; it just tells you to use the OS image.
- Running the image inside QEMU on Debian works, but without VT-x it is unusably slow.

What does work: the image's partitions and kernel are perfectly fine for a BIOS machine. Only the bootloader is missing, so we add one.

## What you need

- The target PC (legacy BIOS) with the destination drive installed. **Everything on that drive will be erased.**
- A USB stick with **Debian Live**, bootable in legacy mode (I used a Debian 13 USB written with Rufus in *DD Image* mode, partition scheme *MBR*, target *BIOS or UEFI-CSM*).
- A second USB stick or disk that holds the umbrelOS image (formatted **exFAT** or ext4/NTFS: the decompressed image is bigger than 4 GB, so FAT32 will not work).
- Another computer to download and decompress the image.
- Internet access on the Live system (to `apt install` GRUB) and an Ethernet cable for umbrelOS.

## Files and Tools Used

The key components required to replicate this procedure are as follows:

- **umbrelOS 2.0 Image**: 
  * *Description*: The official image intended for USB booting, used here to be written directly to the internal drive.
  * *Link*: [Official umbrelOS Downloads](https://umbrel.com/downloads) (select `umbrelos-amd64.img.xz`).
- **Debian Live USB**:
  * *Description*: A live environment bootable in legacy BIOS mode to execute the `dd` commands and install GRUB.
  * *Recommended Flashing Tool*: [Rufus](https://rufus.ie/) (*DD Image* mode, *MBR* partition scheme).
- **Backup GRUB Configuration** (optional):
  * If you wish to save a backup copy of the configuration file (`grub.cfg`), you can find it inside the boot partition of the image and place it in the `assets/grub.cfg` directory of this repository.
  

## Step by step

### 0. Get the image (on another computer)

1. Download **`umbrelos-amd64.img.xz`** from <https://umbrel.com/downloads> (the "Run from a USB drive" image, about 1.5 GB). Do **not** use the installer ISO.
2. Decompress it to `umbrelos-amd64.img` (about 8.5 GB; use `xz -d`, 7-Zip, Keka, The Unarchiver...).
3. Copy the `.img` to the exFAT stick.

### 1. Boot Debian Live on the target PC

Press the boot-menu key (F12 on this Acer), pick the Debian Live USB, and open a terminal. Plug in the second stick with the image.

### 2. Identify the disks (be careful)

```bash
lsblk
```

In my case the internal SSD was `/dev/sda` and the stick with the image was `/dev/sdb1`. Names can differ on your machine. Writing to the wrong disk destroys its data.

### 3. Write the image to the internal drive

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb
sudo dd if=/mnt/usb/umbrelos-amd64.img of=/dev/sda bs=4M status=progress conv=fsync
sudo umount /mnt/usb
sudo partprobe /dev/sda
```

`partprobe` may print a warning about the GPT table. That is expected.

### 4. Install legacy GRUB into the MBR

The image's boot partition is partition 2 (ext4, 512 MB). Mount it and install GRUB there:

```bash
sudo apt update && sudo apt install -y grub-pc
sudo mkdir -p /mnt/target
sudo mount /dev/sda2 /mnt/target
sudo grub-install --target=i386-pc --boot-directory=/mnt/target/boot --force /dev/sda
```

Notes:

- `grub-install: command not found` means the package is missing: install `grub-pc` (installing only `grub-pc-bin` was not enough for me).
- The image is GPT and has no "BIOS boot" partition, so `grub-install` complains. **`--force` is required**. GRUB then falls back to block lists, which is fine here.
- It should end with `Installation finished. No error reported.`
- Running `grub-mkconfig` from the Live system fails (`failed to get canonical path of overlay`). **Skip it**: the image already ships a working `grub.cfg`.

Unmount and reboot, removing all USB sticks:

```bash
sudo umount /mnt/target
sudo reboot
```

### 5. First boot lands on a `grub>` prompt (expected)

GRUB starts but looks for its config in `/boot/grub/`, which does not exist yet. Boot manually. First find the partition that contains `vmlinuz`, `initrd.img` and `grub.cfg`:

```
grub> ls
grub> ls (hd0,gpt2)/
```

In my case it was `(hd0,gpt2)`, and the files sit at the **root** of that partition, not in `/boot`. Then:

```
grub> set root=(hd0,gpt2)
grub> configfile /grub.cfg
```

umbrelOS starts booting.

### 6. Make the boot automatic

Boot the Debian Live USB again and copy the config to where GRUB looks for it:

```bash
sudo mount /dev/sda2 /mnt
sudo mkdir -p /mnt/boot/grub
sudo cp /mnt/grub.cfg /mnt/boot/grub/grub.cfg
sudo umount /mnt
sudo reboot
```

Remove the USB stick. From now on: GRUB screen, then umbrelOS, with no keyboard needed.

### 7. Open the dashboard

Connect Ethernet, wait a few minutes for the first boot to finish (the official docs say about 5), then open `http://umbrel.local` or the IP that your router assigned.

## How it boots (verified on the installed system)

```
BIOS -> MBR (GRUB boot.img)
     -> core.img, stored as a file in sda2 at /boot/grub/i386-pc/ (found via block list)
     -> /boot/grub/grub.cfg on sda2
          load_env -f /boot.grubenv
          linux /vmlinuz ... ${rugpi_bootargs}
          initrd /initrd.img
     -> kernel, init=/usr/bin/rugix-ctrl, root=PARTUUID of sda4 (system slot A)
```

Resulting layout (GPT, 500 GB disk):

| Partition | Size | Type | Role |
|---|---|---|---|
| sda1 | 256 MB | vfat | Rugix config (`EFI/BOOT/BOOTX64.efi`, `rugpi/grub.cfg`, `*.grubenv`). Not used by the legacy boot path |
| sda2 | 512 MB | ext4 | Boot slot A: `vmlinuz`, `initrd.img`, `grub.cfg`, `boot.grubenv`, plus the `/boot/grub` we added |
| sda3 | 512 MB | (empty) | Boot slot B |
| sda4 | 10 GB | ext4 | System slot A (the active one) |
| sda5 | 10 GB | (empty) | System slot B |
| sda6 | rest | ext4 | Data (Docker, `/home`, ...) |

### Check your own install

```bash
ls /sys/firmware/efi                                  # should NOT exist (legacy boot)
sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINTS,PARTUUID   # layout as above
sudo dd if=/dev/sda bs=512 count=1 2>/dev/null | strings | head   # shows "GRUB"
sudo rugix-ctrl system info                           # activeGroup "a", bootFlow "grub"
```

## Known limitations

- **Updates are untested and probably unsafe.** umbrelOS updates through Rugix, which writes the new system into the spare slot (sda3/sda5) and switches slots through the `*.grubenv` files on sda1. Our legacy GRUB always boots slot A and never reads those files. An update may therefore never boot, or may need manual intervention. Do not assume the in-app updater works.
- **`core.img` lives at a fixed disk location** inside sda2 (no BIOS boot partition). If sda2 is reformatted, resized or the file is moved, the machine stops booting. Fix: boot Debian Live and repeat step 4.
- No UEFI means no Secure Boot and none of the UEFI conveniences. That does not matter on this hardware.
- Only tested on one machine and one image version.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Blinking cursor from a USB stick | Official image/installer is UEFI-only. Use this method instead |
| Ventoy: "No ISO or supported file found" | It does not list raw `.img` files. Not needed for this method |
| `grub-install: command not found` | `sudo apt install grub-pc` |
| `grub-install` fails about a missing BIOS boot partition | Add `--force` |
| `grub-mkconfig`: `failed to get canonical path of overlay` | Ignore it, the image has its own `grub.cfg` |
| Stuck at `grub>` | Do step 5, then step 6 |
| Debian installer does not see the disk after `dd`-ing the image | The GPT/UEFI layout confused it. Do not use the installer; follow this guide with the Live USB |

## Credits

I have created this repo with Claude. 
This project is created by me with the help of AI (Gemini, Claude).
I'm italian, sorry for my bad english.
For me this configuration works well, i've also home assistant, tailscale, adguard ecc. and they works as fine as my previous conf. with CasaOs on ubuntu (Hp pavilion i5 8thgen, 16 ram, 1tbssd.
I know the hardware limitation but umbrel os works on his native os (Rugix, Debian) so is very light. I'm waiting for the integration with the ios app (now on TestFLight).
This is an opensource project so i'm open for help and implementation.


thanksss ;)                                                                                                                                                                      by lelli
