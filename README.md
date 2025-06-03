# How I Recovered My Linux System After Breaking `/home`

> **Date:** June 2025  
> **Author:** Your Name  

---

## Problem

While experimenting with Flux 1 Dev under an enroot/NVIDIA GPU container, my Ubuntu `/home` partition ran out of space (it was only 100 GB, but the container images and models required ~300 GB). In an attempt to expand `/home` on the fly, I ended up corrupting my partition layout and breaking my boot process. When I next rebooted:

- The GRUB menu failed to load properly.
- Ubuntu dropped into emergency mode with a broken GNOME shell.
- My `/home` directory was unmountable, and I could no longer log in normally.

In short: **I had lost access to my user files and my desktop environment**.

---

## Diagnosis

1. **Cannot Boot Normally**  
   - My system displayed a GRUB rescue prompt (no GUI).  
   - When I forced a boot into a Live USB, I ran:
     ```bash
     lsblk
     ```
     and confirmed that the root partition (`/dev/nvme1n1p3`) was still intact but that `/home` had become misaligned/mis-mounted.

2. **Inspecting Partitions**  
   - From the Live USB, I launched **GParted** and saw:  
     - My original `/home` (100 GB) was now labeled as “ext4” but unmounted.  
     - There was an unallocated block of ~396 GB (previously freed from my data partition).  
     - `/etc/fstab` on the root partition pointed to a non-existent UUID for `/home`.

   ![GParted showing unallocated space where /home should be](docs/screenshots/gparted-unallocated-before.png)

3. **Broken GNOME & Corrupted Packages**  
   - When I tried to boot anyway, Ubuntu dropped into emergency mode.  
   - In a TTY, I saw errors complaining about missing `libgstreamer1.0-0`, a broken GNOME session, and a read-only root filesystem.  
   - I ran:
     ```bash
     df -h
     # Confirmed that /home was not mounted and root was nearly full.
     ps aux | grep gparted
     ```
   - My `/etc/fstab` entry for `/home` was pointing to a UUID that no longer existed (because I had deleted a partition previously and recreated it).  

At this point I knew two things needed fixing:

1. **Partition/Layout**: Re-create and mount a large `/home` partition.  
2. **System Packages**: Reinstall any broken desktop components (GNOME, GStreamer, Nautilus).

---

## Solution

### 1. Boot from a Live USB

- I created an Ubuntu 24.04 live USB (from another machine) using `dd`.  
  ```bash
  sudo dd if=~/Downloads/ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
  ```
- Rebooted my computer, chose “Try Ubuntu”, and opened a terminal.

### 2. Rebuild & Resize the `/home` Partition

1. **Open GParted**  
   - Located my NVMe disk (`/dev/nvme1n1`).  
   - **Deleted** the old 100 GB “home” partition (`nvme1n1p3`), leaving ~396 GB unallocated.  
   - **Created** a new 396 GB `ext4` partition and labeled it `home_new`.  

   ![GParted: New ext4 partition ~396 GB for /home](docs/screenshots/gparted-unallocated-before.png)

2. **Mount Old Root & New `/home`**  
   ```bash
   sudo mkdir -p /mnt/ubuntu
   sudo mount /dev/nvme1n1p3 /mnt/ubuntu

   sudo mkdir -p /mnt/home_new
   sudo mount /dev/nvme1n1p2 /mnt/home_new   # new partition
   ```

3. **Copy Existing `/home` Data**  
   - I still had my user directories under `/mnt/ubuntu/home/`, so I ran:
     ```bash
     sudo rsync -aHv /mnt/ubuntu/home/ /mnt/home_new/
     ```
     This preserved permissions, ownership, and symlinks.

4. **Prepare a Clean `/home` on Root**  
   ```bash
   # Rename the old broken /home
   sudo mv /mnt/ubuntu/home /mnt/ubuntu/home.backup

   # Create an empty /home directory
   sudo mkdir -p /mnt/ubuntu/home
   sudo chown root:root /mnt/ubuntu/home
   sudo chmod 755 /mnt/ubuntu/home
   ```

5. **Update `/etc/fstab`**  
   - I fetched the new partition’s UUID:
     ```bash
     sudo blkid -s UUID -o value /dev/nvme1n1p2
     # Example output: c2ab34f0-1234-4a7b-9a0f-abcdef123456
     ```
   - Edited `/mnt/ubuntu/etc/fstab` with my editor:
     ```ini
     # <file system>                         <mount point> <type>  <options>       <dump> <pass>
     UUID=c2ab34f0-1234-4a7b-9a0f-abcdef123456  /home         ext4    defaults        0      2
     ```
   - Confirmed there were no existing `/home` entries pointing to a bad UUID.

6. **Unmount & Reboot**  
   ```bash
   sudo umount /mnt/home_new
   sudo umount /mnt/ubuntu
   sudo reboot
   ```
   - On normal boot, I ran:
     ```bash
     df -h | grep "/home"
     # Output: /dev/nvme1n1p2   396G   48G  324G  13% /home
     ```
   - My user directories and files were all intact at `/home/username`.

   ![`df -h` showing /home mounted on ~396 GB partition](docs/screenshots/final-df-h.png)

### 3. Recover Broken Desktop Packages

Even after restoring `/home`, my GNOME desktop was still broken (Ubuntu booted to a CLI or dropped back to emergency mode).

1. **Log In to a TTY (Ctrl+Alt+F2)**  
2. **Remount Root Read/Write & Update APT**  
   ```bash
   sudo mount -o remount,rw /
   sudo apt-get update
   ```
3. **Fix Broken Packages & Reinstall Desktop**  
   ```bash
   sudo apt-get install --fix-broken -y
   sudo dpkg --configure -a
   sudo apt-get install --reinstall        ubuntu-desktop        gnome-shell        nautilus        libgstreamer1.0-0        gdm3 -y
   ```
4. **Remove Any Stale GNOME Cache & Extensions**  
   ```bash
   mv ~/.local/share/gnome-shell/extensions ~/.local/share/gnome-shell/extensions.bak 2>/dev/null || true
   rm -rf ~/.cache/gnome-shell
   ```
5. **Reboot Again**  
   ```bash
   sudo reboot
   ```
   - After reboot, GNOME loaded normally.  
   - My original fonts, settings, and desktop icons were all present.

---

## Lesson Learned

1. **Always Plan for Space Growth**  
   - If you anticipate needing hundreds of gigabytes for models, datasets, or containers, create a separate data partition (or use LVM) at install time. That way, `/home` stays under steady growth.

2. **Never Resize Critical Partitions While Mounted**  
   - Attempting to resize or move `/home` (or `/`) from within a running installation invites filesystem corruption. Always use a Live USB environment so that the target is not in use.

3. **Back Up `fstab` Before Changes**  
   - A single typo or wrong UUID in `/etc/fstab` can leave your system unbootable. Maintaining a simple backup like `/etc/fstab.bak` saved me hours of trial and error.

4. **Build Quick “Recovery Scripts”**  
   - Rather than repeating manual `rsync`, `blkid`, and `vim /etc/fstab` steps, I now keep one short Bash script that automates:
     1. Mounting root & data partitions  
     2. Copying `/home` via `rsync`  
     3. Injecting the correct line into `/etc/fstab`  
   - Automation minimizes human error when time is short and the stakes are high.

5. **Test Major System Changes in a VM First**  
   - I now spin up a disposable Ubuntu VM before modifying any production box. If something breaks in the VM, I learn without risking real data.

---

## Skills Demonstrated

- **Live USB Environment & GParted**  
  - Booted from a Live USB, used GParted to inspect, delete, and create partitions.  
  - Labeled partitions (`home_new`) and verified with `blkid`.

- **Filesystem Copy (`rsync`) & Permissions**  
  - Ran `sudo rsync -aHv /old/home/ /new/home/` to preserve ownership, symlinks, ACLs, and extended attributes.

- **`/etc/fstab` Editing & UUID Management**  
  - Retrieved the new partition’s UUID with `blkid`.  
  - Updated `/etc/fstab` safely to mount `/home` on the correct device.

- **Chroot/Recovery for Broken Packages**  
  - Reinstalled `libgstreamer1.0-0`, `ubuntu-desktop`, `gnome-shell`, and `nautilus` to recover a broken GNOME environment.  
  - Ran `apt-get install --fix-broken -y` and `dpkg --configure -a` to fix incomplete package states.

- **Troubleshooting & Diagnostics**  
  - Used `lsblk`, `df -h`, `blkid`, and `ps aux | grep gparted` to confirm partition states.  
  - Inspected boot logs and emergency-mode output to zero in on missing dependencies.

- **Shell Scripting & Automation**  
  - (Optionally) Created a small `safe_resize_home.sh` that mounts partitions, copies `/home`, and updates `/etc/fstab` automatically—showing that I can turn manual steps into reproducible automation.

---

## TL;DR

**Yes, you should share this story as a technical write-up**, because:

- It proves you can troubleshoot a complete system fail—something rare in most student portfolios.  
- Recruiters and DevOps engineers will immediately see that you know GRUB, partitions, `fstab`, Live USB recovery, and package repair.  
- It ties together your AI/Flux 1 Dev work with **real** Linux administration skills—demonstrating that you can manage everything from user‐level container workflows down to OS-level recovery.

---

**Thank you for reading!** If you have questions, feedback, or suggestions, feel free to reach out.