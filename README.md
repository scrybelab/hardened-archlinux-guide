# hardened-archlinux-guide

## Target Configuration
- **Hardware:** AMD CPU/GPU, NVMe Drive (`/dev/nvme0n1`), Intel AX200/AX210 Wi-Fi
- **Encryption:** LUKS2 + LVM (Root, Home, Swap)
- **Filesystem:** ext4
- **Bootloader:** Limine (UEFI)
- **Network:** `iwd` + `dhcpcd` + `dnscrypt-proxy` (DNS + DNSSEC)
- **Security:** `opendoas`, `ufw`, hardened `sysctl` rules

### 01. Verify UEFI boot mode:
```bash
ls /sys/firmware/efi/efivars
```

### 02. Connect to the Wi-Fi:
```bash
iwctl
[iwd]# device list
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect YOUR_SSID
[iwd]# station wlan0 show
[iwd]# exit
```
`wlan0` may not be the standard interface name; use `device list` to verify.

### 03. Update the system clock:
```bash
timedatectl set-ntp true
```

> [!WARNING]
> **Destructive Operation:** The following `fdisk` and `cryptsetup` commands will completely erase all data on `/dev/nvme0n1`. Verify your target drive carefully before proceeding.

### 04. Partition the disk using fdisk:
```bash
fdisk /dev/nvme0n1
```
Inside fdisk:
- Type `g` (Create a new empty GPT partition table)
- Type `n`, `1`, `Enter`, `+512M` (Create partition 1, 512M)
- Type `t`, `1`, `1` (Change partition 1 type to '1' = EFI System)
- Type `n`, `2`, `Enter`, `Enter` (Create partition 2, remaining space)
- Type `w` (Write changes and exit)

### 05. Format and open the LUKS2 encrypted partition:
```bash
cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 cryptlvm
```

### 06. Initialize LVM on the decrypted container:
```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg0 /dev/mapper/cryptlvm
lvcreate -L 8G vg0 -n lv_swap
lvcreate -L 32G vg0 -n lv_root
lvcreate -l 100%FREE vg0 -n lv_home
```

### 07. Format the filesystems:
```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/vg0/lv_root
mkfs.ext4 /dev/vg0/lv_home
mkswap /dev/vg0/lv_swap
swapon /dev/vg0/lv_swap
```

### 08. Mount the filesystems (using explicit, classic directory creation):
```bash
mount /dev/vg0/lv_root /mnt
mkdir -p /mnt/home /mnt/boot
mount /dev/vg0/lv_home /mnt/home
mount /dev/nvme0n1p1 /mnt/boot
```

### 09. Install the base system:
```bash
pacstrap -K /mnt base linux linux-firmware-amdgpu linux-firmware-intel lvm2 iwd dhcpcd chrony fwupd reflector limine nano opendoas ufw amd-ucode mesa vulkan-radeon efibootmgr bash-completion dnscrypt-proxy
```

### 10. Generate the fstab file and configure volatile /tmp:
```bash
genfstab -U /mnt >> /mnt/etc/fstab

# Volatile /tmp: Limited to 2G to prevent RAM exhaustion.
echo 'tmpfs /tmp tmpfs rw,nosuid,nodev,relatime,size=2G,mode=1777 0 0' >> /mnt/etc/fstab

# volatile /var/crash: Core dumps write to the ram only and clears on reboot.
mkdir -p /mnt/var/crash
echo 'tmpfs /var/crash tmpfs rw,nosuid,nodev,noexec,size=512M,mode=0700 0 0' >> /mnt/etc/fstab
```

### 11. Enter the new system environment:
```bash
arch-chroot /mnt
```

### 12. Optimized Mirrors (Optional):
```bash
reflector --country CH,IS,NO,SE,DE --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
pacman -Syyu
```

### 13. Set Timezone and Hardware Clock:
```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
hwclock --systohc
```

### 14. Configure Locale and Keymap:
```bash
sed -i '/en_GB.UTF-8/s/^# //g' /etc/locale.gen
locale-gen
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
echo "KEYMAP=uk" > /etc/vconsole.conf
```

### 15. Set the hostname and hosts file
```bash
echo "desktop" > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   desktop.localdomain desktop
EOF
```

### 16. Configure mkinitcpio with the required hooks:
```bash
sed -i 's/^MODULES=.*/MODULES=(amdgpu)/' /etc/mkinitcpio.conf

# Note: 'consolefont' is intentionally omitted. It is only required if a custom FONT= is defined in /etc/vconsole.conf. 
# Omitting it keeps the initramfs minimal.
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)/' /etc/mkinitcpio.conf
mkinitcpio -P
```

### 17. Set the root password:
```bash
passwd
```

### 18. Configure Network and Encrypted DNS:
```bash
# 1. Disable iwd's internal DHCP client to prevent conflicts with dhcpcd
mkdir -p /etc/iwd
cat <<EOF > /etc/iwd/main.conf
[General]
EnableNetworkConfiguration=false
EOF

# 2. Disable conflicting systemd networking services
systemctl disable systemd-timesyncd
systemctl disable systemd-resolved

# 3. Prevent dhcpcd from overwriting /etc/resolv.conf with plaintext
echo "nohook resolv.conf" >> /etc/dhcpcd.conf

# 4. Configure dnscrypt-proxy for maximum privacy (Anonymized DNS + DNSSEC)
# Routes queries through a relay so Quad9 sees the relay's IP, not yours.
cat <<EOF > /etc/dnscrypt-proxy/dnscrypt-proxy.toml
server_names = ['quad9-dnscrypt-ip4-filter-pri', 'quad9-dnscrypt-ip6-filter-pri']
listen_addresses = ['127.0.0.1:53', '[::1]:53']
require_dnssec = true
fallback_resolvers = ['9.9.9.9:53', '149.112.112.112:53']

[sources]
  [sources.'public-resolvers']
  urls = ['https://raw.githubusercontent.com/DNSCrypt/dnscrypt-resolvers/master/v3/public-resolvers.md', 'https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md']
  minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
  cache_file = '/var/cache/dnscrypt-proxy/public-resolvers.md'
  refresh_delay = 72
  prefix = ''
EOF
```

### 19. Enable required services to start on boot:
```bash
systemctl enable iwd dhcpcd chronyd fwupd ufw dnscrypt-proxy.socket
```

### 20. Install and configure the Limine bootloader:
```bash
mkdir -p /boot/EFI/limine
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/

cat <<EOF > /boot/EFI/limine/limine.conf
timeout: 5

/Arch Linux
    protocol: linux
    kernel_path: boot():/vmlinuz-linux
    cmdline: cryptdevice=UUID=<UUID>:cryptlvm root=/dev/mapper/vg0-lv_root rw quiet
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux.img
EOF

# UUID injection into limine.conf
sed -i "s/<UUID>/$(blkid -s UUID -o value /dev/nvme0n1p2)/g" /boot/EFI/limine/limine.conf
```

### 21. Register the bootloader in UEFI NVRAM:
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "archlinux" --loader '\EFI\limine\BOOTX64.EFI'
```
(Note: UEFI paths in efibootmgr use backslashes \ per the UEFI specification).

### 22. Configure opendoas
```bash
echo "permit persist :wheel" > /etc/doas.conf
```

### 23. Configure ufw (Uncomplicated Firewall) defaults:
```bash
ufw default deny incoming
ufw default allow outgoing
ufw enable
```

### 24. Create a user
```bash
useradd -m -G video,audio,wheel -s /bin/bash anon
passwd anon
```

### 25. Exit the chroot, unmount, and reboot:
```bash
exit
umount -R /mnt
reboot
```
(Remove the installation media upon reboot).

## Post-Installation
### Connect to Wi-Fi
```bash
iwctl
[iwd]# station wlan0 connect YOUR_SSID
[iwd]# exit

# Verify dnscrypt-proxy is actively resolving
doas systemctl status dnscrypt-proxy.socket
ping archlinux.org
```

### Post-Installation Hardening (Developer-Friendly)
#### 1. Configure systemd-coredump for volatile, size-limited storage
```bash
# 1. Symlink the default coredump directory to the volatile /var/crash tmpfs
doas rm -rf /var/lib/systemd/coredump
doas ln -s /var/crash /var/lib/systemd/coredump

# 2. Configure systemd-coredump limits
doas mkdir -p /etc/systemd/coredump.conf.d
cat <<EOF | doas tee /etc/systemd/coredump.conf.d/10-secure.conf
[Coredump]
Storage=external
ProcessSizeMax=512M
MaxUse=512M
EOF

# 3. Reload systemd to apply the new coredump config
doas systemctl daemon-reload
```

#### 2. Unified sysctl rules (IPv6 Privacy, Network Hardening)
```bash
cat <<EOF | doas tee /etc/sysctl.d/99-security.conf
# Core Dump Handling (Prevents setuid binaries from leaking sensitive memory)
fs.suid_dumpable = 0

# IPv6 Privacy Extensions
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2

# Kernel Info & Debugging Leaks
kernel.kptr_restrict = 1
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1

# Network Spoofing & Redirect Protections
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
EOF

doas sysctl --system
```
