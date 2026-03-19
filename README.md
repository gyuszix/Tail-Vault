# PiNAS

A lightweight, secure NAS solution using a Raspberry Pi and Tailscale mesh networking. Access your files from anywhere over an encrypted WireGuard tunnel.
Simple and very useful little project for anyone. Some of these steps are optional and are just my preferences. For example, you might be able to get away with using less memory or an earlier model.
Equally, originally I wanted to use a NVMe hat and M.2 SSD but with the current SSD prices that seemed unreasonable. Since I ended up using an Portable SSD as my NAS storage I created two partitions, one in ext4 format while the other in exFAT.
With this setup I can still carry around my SSD and use it on any OS should the need arise. 

<img width="2553" height="1440" alt="Screenshot 2026-03-19 at 19 54 42" src="https://github.com/user-attachments/assets/1b8e704e-1ca1-448e-a16d-2160b4388ab8" />


## Hardware Requirements

| Component | Minimum Spec | Notes |
|-----------|--------------|-------|
| Raspberry Pi | 4+ GB RAM | Pi 4 or Pi 5 recommended |
| Storage | 500+ GB SSD/NVMe | NVMe HAT for Pi 5, USB 3.0 SSD for Pi 4 |
| Cooling | Active cooler | Required for sustained workloads |
| Power Supply | 5V 5A USB-C | Official Pi power supply recommended |
| SD Card | 16+ GB | For booting the OS |
| Ethernet Cable | Cat5e or better | For reliable network speeds |

## Software Overview

- **Raspberry Pi OS Lite** — Headless Debian-based OS
- **Tailscale** — Secure WireGuard mesh VPN
- **NFS** — Network file system for sharing storage
- **btop** — Resource monitoring

---

## Pi Setup

### 1. Flash the OS

Download [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/) and flash it to your SD card using [balenaEtcher](https://etcher.balena.io/).
Alternatively use any other imager or the built in option on Linux.

Before ejecting, enable SSH by creating an empty file named `ssh` in the boot partition:
Alternatively, set up ssh options with the built in tools if you are using the official raspberry imager.

```bash
touch /Volumes/boot/ssh
```

### 2. Initial Configuration

Insert the SD card, connect ethernet and power, then SSH into the Pi:

```bash
ssh pi@raspberrypi.local
```

Update the system and install required packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nfs-kernel-server btop
```

Change the default password and hostname:

```bash
passwd
sudo hostnamectl set-hostname piNAS
```

### 3. Partition and Format the SSD

Identify your drive:

```bash
lsblk
```

Partition the drive (replace `/dev/sda` with your device):

```bash
sudo fdisk /dev/sda
```

In fdisk:
- `g` — Create new GPT partition table
- `n` — New partition, accept defaults for first partition, set size (e.g., `+800G`)
- `n` — New partition for remaining space
- `w` — Write changes and exit

Format the partitions:

```bash
sudo mkfs.ext4 /dev/sda1
sudo mkfs.exfat /dev/sda2
```

### 4. Mount the Storage

Create mount points:

```bash
sudo mkdir -p /mnt/nas
sudo mkdir -p /mnt/portable
```

Get the UUIDs of your partitions:

```bash
sudo blkid
```

Add to `/etc/fstab` for persistent mounting (replace UUIDs with your own):

```bash
sudo nano /etc/fstab
```

```
UUID=<your-ext4-uuid> /mnt/nas ext4 defaults,nofail 0 2
UUID=<your-exfat-uuid> /mnt/portable exfat defaults,nofail 0 0
```

Mount everything:

```bash
sudo mount -a
```

Set ownership of the NAS directory:

```bash
sudo chown -R $USER:$USER /mnt/nas
```

### 5. Install and Configure Tailscale

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start and authenticate:

```bash
sudo tailscale up
```

Follow the link to authenticate. Note your Pi's Tailscale IP:

```bash
tailscale ip -4
```

Enable Tailscale on boot:

```bash
sudo systemctl enable tailscaled
```

### 6. Configure NFS Server

Edit the exports file:

```bash
sudo nano /etc/exports
```

Add your NFS share (replace `<your-tailnet-ip-range>` with your Tailscale subnet, e.g., `100.64.0.0`):

```
/mnt/nas <your-tailnet-ip-range>/10(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
```

Export the share and start NFS:

```bash
sudo exportfs -a
sudo systemctl enable --now nfs-kernel-server
```

---

## Client Setup

### macOS

Install Tailscale from the [App Store](https://tailscale.com/download) or via Homebrew:

```bash
brew install tailscale
```

Authenticate:

```bash
tailscale up
```

Create the mount point:

```bash
mkdir -p ~/nas
```

Test the mount (replace `<NAS-tailscale-ip>` with your Pi's Tailscale IP):

```bash
sudo mount_nfs -o nfsvers=4,resvport <NAS-tailscale-ip>:/mnt/nas ~/nas
```

For persistent mounting, add to `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

```
<NAS-tailscale-ip>:/mnt/nas /Users/<your-username>/nas nfs resvport,nfsvers=4,soft,bg 0 0
```

Mount it:

```bash
sudo mount -a
```

### Linux

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Install NFS client:

```bash
sudo apt install -y nfs-common
```

Create mount point:

```bash
sudo mkdir -p /mnt/nas
```

Test the mount:

```bash
sudo mount -t nfs -o nfsvers=4 <NAS-tailscale-ip>:/mnt/nas /mnt/nas
```

For persistent mounting, add to `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

```
<NAS-tailscale-ip>:/mnt/nas /mnt/nas nfs nfsvers=4,soft,bg 0 0
```

Mount and set permissions:

```bash
sudo mount -a
sudo chown -R $USER:$USER /mnt/nas
```

---

## Optional: SSH Key Authentication

Setting up SSH keys lets you log into the Pi without a password. Since everything is behind Tailscale, this is optional but convenient.

### Generate a Key (on your client machine)

```bash
ssh-keygen -t ed25519
```

Accept the default location and optionally add a passphrase.

### Copy the Key to the Pi

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <your-pi-username>@<NAS-tailscale-ip>
```

### Test It

```bash
ssh <your-pi-username>@<NAS-tailscale-ip>
```

Should log you in without asking for a password.

### Using the Same Key on Multiple Machines

Copy the private key to your other machines:

```bash
scp ~/.ssh/id_ed25519 <user>@<other-machine-ip>:~/.ssh/id_ed25519
```

Set the correct permissions on the destination machine:

```bash
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### Optional: Disable Password Authentication

Once your keys are working, you can disable password auth on the Pi for extra security:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and change:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

---

## Testing Performance

Test write speed:

```bash
dd if=/dev/zero of=/path/to/nas/testfile bs=64M count=16
```

Test read speed:

```bash
dd if=/path/to/nas/testfile of=/dev/null bs=64M
```

Test network baseline with iperf3:

```bash
# On the Pi
iperf3 -s

# On the client
iperf3 -c <NAS-tailscale-ip>
```

---

## Monitoring

SSH into the Pi and run:

```bash
btop
```

---

## Troubleshooting

**Mount fails on macOS**

Make sure you include the `resvport` option — macOS requires it for NFS.

**Permission denied when writing**

Check that the `anonuid` and `anongid` in `/etc/exports` match the owner of `/mnt/nas`:

```bash
id $USER
ls -la /mnt/nas
```

**NFS server not starting**

```bash
sudo systemctl status nfs-kernel-server
sudo exportfs -a
```

---

## Limitations

**Slow speeds**

Your bottleneck is likely your ISP's upload speed. Test with `iperf3` to confirm the network ceiling. In my particular case my upload speed was capped at around 200Mbps. QuickMaths says that would mean downloading a 1Gb file would take about 40 seconds.

## Future Work

Some cool additions to this current setup:

### ntfy — Push Notifications from the CLI

[ntfy](https://ntfy.sh) is a simple pub/sub notification service that lets you send push notifications to your phone or desktop from scripts or the command line.

Example use cases for a NAS:
- Get notified when a large file transfer completes
- Alert when the Pi reboots or goes offline
- Monitor disk space and warn when running low

Basic usage:
```bash
# Send a notification
curl -d "Backup complete" ntfy.sh/your-topic

# Or use the CLI
ntfy publish your-topic "NAS is online"
```

You can self-host ntfy on the Pi itself or use the free public server at ntfy.sh. Install the app on your phone to receive notifications.

### Other ideas

- **Prometheus + Grafana** — Full monitoring stack for disk I/O, network throughput, and system metrics
- **Healthchecks.io** — Dead man's switch to alert if the Pi stops checking in
- **Restic/Borg** — Automated encrypted backups to cloud storage

## License

MIT
