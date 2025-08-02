# Debian Boot Repair Cheat Sheet (Dell Latitude E7440)
Linux boot repair cheat sheet

This is a step‑by‑step guide for fixing **Debian boot issues** on the Dell Latitude E7440 caused by:
- Hybrid MBR/GPT corruption
- GRUB/bootloader problems
- Black screen lockups

This cheat sheet condenses ~2 weeks of troubleshooting into a **replicable process**.  
Tested with Debian 12 Live USB.

---

## 🛠 Prerequisites

- **Boot** from a Debian Live USB (create with Rufus or `dd`, boot via `F12 > USB`)
- Enter BIOS (`F2`):
  - Set **Boot Mode** = `UEFI`
  - Turn **Secure Boot OFF** (can be re‑enabled later)
- Install tools on the Live USB:
  ```bash
  sudo apt update && sudo apt install -y gdisk gptfdisk parted testdisk
  ```

---

## 1️⃣ Diagnose Partition Table (Hybrid MBR/GPT Mess)

1. List partitions:
   ```bash
   lsblk -f
   ```
   Expect: `/dev/sda1` ext4 (root), `/dev/sda3` FAT32 (EFI), maybe ghosts like `/dev/sda2` or `/dev/sda5`.

2. Inspect with **gdisk**:
   ```bash
   sudo gdisk /dev/sda
   ```
   - `p` (print) – review the table
   - `v` (verify) – check for errors
   - `q` – quit without changes

3. If hybrid, clean with **fixparts**:
   ```bash
   sudo fixparts /dev/sda
   ```
   Inside fixparts:
   - `p` (print table)
   - `d 2` (delete ghost sda2), `d 5` (delete sda5 swap)
   - `w` → `y` (write)

4. Convert to GPT with **gdisk**:
   ```bash
   sudo gdisk /dev/sda
   ```
   - `r` → `g` (convert MBR to GPT)
   - `v` (verify no errors)
   - `w` → `y` (write)

5. Verify:
   ```bash
   sudo parted /dev/sda print
   ```
   Should show “gpt” and EFI partition with **boot**/**esp** flags.

---

## 2️⃣ Reinstall GRUB for UEFI

1. **Mount partitions:**
   ```bash
   sudo mkdir -p /mnt/efi /mnt/root
   sudo mount /dev/sda3 /mnt/efi         # EFI FAT32
   sudo mount /dev/sda1 /mnt/root        # Root ext4
   sudo mount --bind /dev /mnt/root/dev
   sudo mount --bind /dev/pts /mnt/root/dev/pts
   sudo mount --bind /proc /mnt/root/proc
   sudo mount --bind /sys /mnt/root/sys
   sudo mount --bind /run /mnt/root/run
   ```

2. **Chroot and reinstall GRUB:**
   ```bash
   sudo chroot /mnt/root /bin/bash
   grub-install --target=x86_64-efi --efi-directory=/mnt/efi --bootloader-id=GRUB
   update-grub
   exit
   ```

3. **Unmount:**
   ```bash
   sudo umount -l /mnt/efi /mnt/root/dev/pts /mnt/root/{dev,proc,sys,run} /mnt/root
   ```

---

## 3️⃣ Configure BIOS for UEFI Boot

1. Reboot → Press **F2** for BIOS.
2. Set:
   - **Boot Mode** = UEFI
   - **Secure Boot** = Disabled (enable later if shim is installed)
3. Boot Sequence → **Add Boot Option**:
   - **Name:** `Debian GRUB`
   - **File System:** Select EFI partition (sda3)
   - **Navigate:** `EFI > GRUB > grubx64.efi`
   - Move Debian GRUB to the **top** of the list.
4. Save & Exit (`F10`) → remove USB → reboot.

---

## 4️⃣ Fix Black Screen / GUI Issues

If Debian boots to a console but not GNOME:

- **Force X11 instead of Wayland:**
  ```bash
  sudo nano /etc/gdm3/custom.conf
  ```
  Add:
  ```
  [daemon]
  WaylandEnable=false
  ```

- **Restart GDM**:
  ```bash
  SYSTEMD_IGNORE_CHROOT=1 systemctl restart gdm3
  ```
  *(or just reboot)*

- **Reinstall Intel drivers**:
  ```bash
  sudo apt install --reinstall xserver-xorg-video-intel gdm3
  ```

- **Optional graphics fix:**  
  Add `nomodeset` to `/etc/default/grub`:
  ```bash
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
  sudo update-grub
  sudo reboot
  ```

---

## 🧰 Troubleshooting Tips

- **No bootable device?**  
  Mount EFI partition and verify:
  ```bash
  ls /mnt/efi/EFI/GRUB
  ```
- **Secure Boot blocking boot?**  
  Disable Secure Boot temporarily, or use `shim-signed` package for signed GRUB.
- **Lost partitions?**  
  Use `testdisk` or `photorec` from Live USB.

---

## 📜 License

MIT License – use, share, and improve freely.

---

✅ This process resolved a **two-week boot failure** and restored a clean UEFI Debian setup on a Dell Latitude E7440.  

Copy/paste this guide to save someone else the pain.
