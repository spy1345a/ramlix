# Ramlix

A minimal custom Linux live distro that boots entirely into RAM, built from scratch with a custom kernel, Alpine Linux userland, and Openbox GUI.

![Boot](https://img.shields.io/badge/boot-RAM-blue) ![Kernel](https://img.shields.io/badge/kernel-7.2.3-green) ![Base](https://img.shields.io/badge/base-Alpine%203.20-brightgreen) ![WM](https://img.shields.io/badge/wm-Openbox-orange)

---

## What is Ramlix

Ramlix is a live Linux distribution that:

- Boots from an ISO into RAM — no installation needed
- Runs entirely in memory after boot, the disc is never touched again
- Has a working package manager (`apk`) so you can install software at runtime
- Uses a custom compiled Linux kernel
- Provides a minimal but functional GUI via Openbox + X11

---

## Stack

| Component | Choice | Why |
|---|---|---|
| Kernel | Linux 7.2.3 (custom compiled) | Full control over drivers and config |
| Base | Alpine Linux 3.20 minirootfs | ~3MB base, musl libc, very lightweight |
| Package manager | apk | Fast, simple, 24000+ packages available |
| Init system | OpenRC | Lightweight, no systemd |
| Window manager | Openbox | ~1MB, minimal, right-click menu |
| Display server | X11 (xorg) | Stable, works with vesa/fbdev drivers |
| Terminal | xterm | Minimal, always works |
| Bootloader | GRUB | Hybrid BIOS + UEFI support |

---

## Requirements

### Build host

- Arch Linux or Arch-based distro (CachyOS, Manjaro, EndeavourOS etc.)
- At least 8GB free disk space
- Internet connection

### Build dependencies

```bash
sudo pacman -S --needed \
  base-devel bc flex bison openssl libelf ncurses \
  pahole dwarves cpio xz zstd lz4 gzip perl python \
  tar xorriso grub mtools dosfstools qemu-system-x86
```

---

## Build Instructions

### 1. Get the kernel source

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.2.3.tar.xz
tar -xf linux-7.2.3.tar.xz
cd linux-7.2.3
```

### 2. Configure the kernel

```bash
make x86_64_defconfig

# Enable required options for live RAM boot
scripts/config --enable BLK_DEV_INITRD
scripts/config --enable TMPFS
scripts/config --enable OVERLAY_FS
scripts/config --enable SQUASHFS
scripts/config --enable SQUASHFS_XZ
scripts/config --enable SQUASHFS_ZSTD
scripts/config --enable ISO9660_FS
scripts/config --enable BLK_DEV_LOOP
scripts/config --enable EFI_STUB
scripts/config --enable DEVTMPFS
scripts/config --enable DEVTMPFS_MOUNT
scripts/config --set-val BLK_DEV_RAM_SIZE 65536

make olddefconfig
```

### 3. Compile the kernel

```bash
make -j$(nproc)
# Output: arch/x86/boot/bzImage
# Takes ~10-15 minutes on a 4-core machine
```

### 4. Set up the rootfs

```bash
cd ..
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
mkdir -p rootfs
cd rootfs
sudo tar -xzf ../alpine-minirootfs-3.20.3-x86_64.tar.gz
cd ..
```

### 5. Mount virtual filesystems and chroot

```bash
sudo mount --bind /proc rootfs/proc
sudo mount --bind /sys rootfs/sys
sudo mount --bind /dev rootfs/dev
sudo mount --bind /dev/pts rootfs/dev/pts

sudo chroot rootfs /bin/bash --login
```

### 6. Inside chroot — configure the system

```bash
# Fix PATH and apk
export PATH=/sbin:/bin:/usr/sbin:/usr/bin
ln -sf /sbin/apk /usr/bin/apk

# Set DNS
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# Set apk repos
cat > /etc/apk/repositories << 'EOF'
https://dl-cdn.alpinelinux.org/alpine/v3.20/main
https://dl-cdn.alpinelinux.org/alpine/v3.20/community
EOF

# Update and install packages
apk update
apk add \
  alpine-base openrc util-linux e2fsprogs dosfstools \
  bash vim htop networkmanager dhclient iproute2 iputils \
  xorg-server xf86-video-vesa xf86-video-fbdev \
  xf86-input-libinput openbox xinit xterm font-dejavu setxkbmap

# Set root password
passwd root

# Set hostname
echo "ramlix" > /etc/hostname
```

### 7. Inside chroot — write the init script

```bash
cat > /init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
mount -t tmpfs none /tmp
mount -t tmpfs none /run
mkdir -p /dev/pts
mount -t devpts none /dev/pts
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
echo "ramlix" > /proc/sys/kernel/hostname
ip link set lo up
/sbin/openrc sysinit
/sbin/openrc boot
/sbin/openrc default
exec /sbin/init
EOF

chmod +x /init
```

### 8. Inside chroot — configure Openbox

```bash
cat > /root/.xinitrc << 'EOF'
#!/bin/sh
setxkbmap us
xset r rate 300 50
xset s off
xset -dpms
exec openbox-session
EOF
chmod +x /root/.xinitrc

mkdir -p /root/.config/openbox

cat > /root/.config/openbox/autostart << 'EOF'
xterm &
EOF

cat > /root/.config/openbox/menu.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<openbox_menu>
  <menu id="root-menu" label="Menu">
    <item label="Terminal">
      <action name="Execute"><command>xterm</command></action>
    </item>
    <separator/>
    <item label="Reboot">
      <action name="Execute"><command>reboot</command></action>
    </item>
    <item label="Shutdown">
      <action name="Execute"><command>poweroff</command></action>
    </item>
  </menu>
</openbox_menu>
EOF

exit
```

### 9. Unmount and pack the initramfs

```bash
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs/proc
sudo umount rootfs/sys

sudo bash -c 'cd rootfs && find . | cpio -oH newc | gzip -9 > ../initramfs.img'
```

### 10. Build the ISO

```bash
mkdir -p iso/boot/grub

cp linux-7.2.3/arch/x86/boot/bzImage iso/boot/vmlinuz
cp initramfs.img iso/boot/initramfs.img

cat > iso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

menuentry "Ramlix - Boot to RAM" {
    linux  /boot/vmlinuz quiet
    initrd /boot/initramfs.img
}

menuentry "Ramlix - Verbose Boot" {
    linux  /boot/vmlinuz
    initrd /boot/initramfs.img
}

menuentry "Ramlix - Safe Mode" {
    linux  /boot/vmlinuz nomodeset
    initrd /boot/initramfs.img
}
EOF

sudo grub-mkrescue -o ramlix.iso iso/
```

### 11. Test in QEMU

```bash
qemu-system-x86_64 \
  -cdrom ramlix.iso \
  -m 2G \
  -enable-kvm \
  -vga std \
  -boot d \
  -display gtk
```

### 12. Write to USB (for real hardware)

```bash
# Find your USB device
lsblk

# Write the ISO (replace /dev/sdX with your USB device)
sudo dd if=ramlix.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

---

## Directory Structure

```
ramlix/
├── linux-7.2.3/          ← kernel source (not committed)
├── rootfs/               ← Alpine rootfs (not committed)
├── initramfs.img         ← packed initramfs (not committed)
├── ramlix.iso            ← final ISO (not committed)
├── iso/                  ← ISO tree
│   └── boot/
│       ├── vmlinuz       ← kernel binary (not committed)
│       ├── initramfs.img ← initramfs copy (not committed)
│       └── grub/
│           └── grub.cfg  ← GRUB config (committed)
├── .gitignore
└── README.md
```

---

## Build Output Sizes

| Artifact | Size |
|---|---|
| Kernel bzImage | ~12MB |
| initramfs | ~55MB |
| Final ISO | ~98MB |
| RAM required to boot | 2GB recommended |

---

## Installing packages at runtime

Once booted, `apk` works normally with internet access:

```bash
apk update
apk add firefox
apk add neofetch
apk search <package>
```

Note: packages installed at runtime are lost on reboot since everything runs in RAM. To make packages permanent, install them inside the chroot and rebuild the ISO.

---

## Built by

AJ — [github.com/spy1345](https://github.com/spy1345)

Built in one session on an i5-4440 with 16GB RAM. Kernel compiled in 9 minutes 35 seconds.
