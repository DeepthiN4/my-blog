---
title: "Why Your SD Card Is Ghosting WSL2 — And the Fix"
date: 2026-06-28
draft: false
tags: ["WSL2", "Linux", "Embedded", "Windows"]
description: "wsl --mount not working with your USB card reader? Here's the USB/IP approach that actually works."
showToc: true
---

*A practical guide for embedded Linux developers who need reliable SD card access from WSL2*

Microsoft's official recommendation is to use `wsl --mount \\.\PHYSICALDRIVEx`. It's elegant, it's built-in, and for internal drives it works beautifully. But the moment your SD card is plugged in through a USB card reader? Ugh! The disk may not appear, `wsl --mount` may throw a cryptic error, or the device shows up but can't be written to correctly.

I hit exactly this wall while setting up SD cards for a project. After spending more time than I'd like to admit, I switched to **USB/IP via `usbipd-win`** — and it worked.

## Why `wsl --mount` Fails for USB Card Readers

Before diving into the solution, it's worth understanding why the common approach breaks down.

`wsl --mount` works by exposing a Windows disk (identified by `\\.\PHYSICALDRIVEx`) to WSL2. This works well when Windows itself cleanly recognises the storage as a block device. But USB card readers introduce an extra layer of abstraction - the operating system sees a USB mass storage device first, and the disk inside it second. This indirection causes `wsl --mount` to either fail entirely or produce unreliable results.

**USB/IP takes a fundamentally different approach.** Instead of letting Windows handle the USB device and then sharing the disk, it passes the USB device itself - at the USB protocol level - directly into the WSL2 kernel. From that point on, WSL2 owns the device completely and handles all the storage logic natively. No Windows driver intermediary, no abstraction mismatch.

## Prerequisites

Before you begin, make sure you have:

- Windows 11 - I had it.
- WSL2 installed and running (I am using Ubuntu)
- Administrator access on your Windows machine
- A USB card reader with your SD card inserted
- A few minutes and a terminal

## Step 1: Install `usbipd-win`

`usbipd-win` is a small open-source Windows service that implements the USB/IP protocol, allowing USB devices to be shared with WSL2 (and even remote machines).

The quickest way to install it is via winget. Open PowerShell as administrator:

```powershell
winget install dorssel.usbipd-win
```

After installation, restart your PowerShell session (or open a new window) so the `usbipd` command is available on your PATH.

## Step 2: List Connected USB Devices

Plug in your USB SD card reader with the SD card already inserted.

Open **PowerShell as Administrator** (right-click → Run as administrator — this is required for the next steps).

Now list all USB devices visible to Windows:

```powershell
usbipd list
```

You'll see output similar to this:
```powershell
BUSID  VID:PID    DEVICE                                                        STATE

1-2    0bda:0129  Realtek USB3.0 Card Reader                                    Not shared

2-4    0951:1666  DataTraveler 3.0                                               Not shared

3-1    046d:c52b  USB Receiver                                                   Not shared
```

Scan the **DEVICE** column for your SD card reader. Common names include terms like "Card Reader", "Mass Storage Device", or your reader's brand name (Realtek, Transcend, etc.).

Note down the **BUSID** for your card reader. In this example it's `1-2`, but yours will likely differ.

> **Tip:** If you're unsure which entry is your card reader, unplug it, run `usbipd list`, plug it back in, and run the command again. The new entry that appears is your device.

## Step 3: Bind the Device

Binding makes your USB device available for sharing with WSL. You only need to do this **once** per device — the binding persists across reboots.

```powershell
usbipd bind --busid 1-2
```

Replace `1-2` with the BUSID from your output. If the command succeeds, you'll see no output — that's normal. Run `usbipd list` again and you should now see the device's state change to **Shared**:
```powershell
BUSID  VID:PID    DEVICE                          STATE

1-2    0bda:0129  Realtek USB3.0 Card Reader      Shared
```

> **Why is this step needed?** By default, Windows retains exclusive control over all USB devices. Binding explicitly marks a device as available for USB/IP sharing. Without it, the attach step in the next section will fail.

## Step 4: Attach the Device to WSL2

Now attach the shared device to your running WSL2 instance:

```powershell
usbipd attach --wsl --busid 1-2
```

You'll see a brief pause while `usbipd` establishes the connection, followed by a confirmation message. At this point, the USB device is detached from Windows and handed over to WSL2. Windows File Explorer will no longer show the SD card — that's expected and correct.

> **Note:** You may need to run this command each time you reconnect the card reader, since physical reconnection resets the USB device state. The `bind` step, however, only needs to be done once.

## Step 5: Verify the Device Inside WSL2

Switch to your WSL2 terminal and check that the device is visible:

```bash
lsblk
```

You should see a new block device — typically `/dev/sdb` or `/dev/sdc` depending on how many disks are already attached:
```powershell
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT

sda      8:0    0   1T  0  disk

└─sda1   8:1    0   1T  0  part /

sdb      8:16   1  29.7G  0  disk

├─sdb1   8:17   1   256M  0  part

└─sdb2   8:18   1  29.4G  0  part
```

For more detail, including partition types and filesystem information:

```bash
sudo fdisk -l /dev/sdb
```

Your SD card is now fully accessible from WSL2. You can use standard Linux tools — `fdisk`, `parted`, `mkfs`, `dd`, `mount` — exactly as you would on a native Linux machine.

## Step 6: Detach the Device When Done

When you're finished working with the SD card, detach it from WSL2 so Windows can access it again, through PowerShell:

```powershell
usbipd detach --busid 1-2
```

After detaching, Windows will re-detect the USB device and it will reappear in File Explorer as normal. You can now safely remove the card reader.

## Conclusion

If you've been fighting `wsl --mount` with a USB card reader and losing, USB/IP is the tool you've been missing. It bypasses the entire Windows disk abstraction layer and gives WSL2 direct, native access to the USB device.

The setup is a one-time effort: install `usbipd-win`, bind your device once, and from then on it's a single `usbipd attach` command before each session.


