# Debian Server

*Setting up a Debian server from scratch.*

Features:

- OS: Debian stable
- SSH: OpenSSH
- Firewall: UFW
- Auto update OS
- Encrypted drive with USB unlock
- Remote lookup of dynamic IP

## Bootable USB

First step is creating a bootable USB to do a fresh install of Debian.

- Visit [https://www.debian.org/download](https://www.debian.org/download)
- Download the latest "netinst" of the "stable" release.
- Verify download using the SHA checksum.
- Determine the name of the USB drive, something of the kind `/dev/diskX`
- Copy the image, replacing `X`:

```shell
sudo dd if=debian-X-X-netinst.iso of=/dev/diskX bs=1024k status=progress
```

## Installation

Boot of the USB and follow instructions. Do:

- Not create a root user.
- Use LVM with encrypted drive (skip wipe step if disk was encrypted).
- Not install a desktop environment.
- Install SSH server.

## Inspect Server

On server, get the fingerprint of the public key and the IP address:

```shell
ssh-keygen -lf /etc/ssh/ssh_host_rsa_key.pub
ssh-keygen -lf /etc/ssh/ssh_host_ecdsa_key.pub
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
hostname -I
```

## Client SSH

On client, update your ssh config, replacing `X` with hostname and IP address:

```shell
nano ~/.ssh/config
Host X
    HostName 192.168.X.X
```

Copy SSH key, replacing `X` with hostname, confirm above finger print:

```shell
ssh-copy-id X
```

SSH into server using private key, replacing `X` with hostname:

```shell
ssh X
```

*From here on, all commands are to be run on the server over SSH.*

## Server SSH

Update the SSH config, replacing `X`:

```shell
sudo nano /etc/ssh/sshd_config
AllowUsers X
PermitRootLogin no
MaxAuthTries 6
PasswordAuthentication no
AllowAgentForwarding no
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 3
```

Restart SSH service:

```shell
sudo systemctl restart sshd
systemctl status sshd
```

## Firewall

Set up UFW firewall, with ssh server ports:

```shell
sudo apt update && sudo apt upgrade
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

## USB Passphrase

Have you server boot without requiring you to enter disk encryption passphrase.

Insert USB stick. Find USB and encrypted drive:

```shell
lsblk
```

If necessary, create a partition:

```shell
sudo fdisk /dev/sdX
d
d
n
p
1
2048
+1M
w
sudo mkfs.ext4 /dev/sdXX
```

Label the partition:

```shell
sudo e2label /dev/sdXX keydrive
sudo udevadm trigger
```

Mount usb drive:

```shell
sudo mkdir /mnt/usb
sudo mount /dev/sdXX /mnt/usb
```

Create a random passphrase, write it to USB, and add it to LUKS, replacing `X`
according to the above listing:

```shell
head -c 256 /dev/random > /mnt/usb/key.bin
sudo chmod 400 /mnt/usb/key.bin
sudo cryptsetup luksAddKey /dev/sdaX /mnt/usb/key.bin
```

Update `crypttab`, keeping the `UUID`:

```shell
sudo nano /etc/crypttab
sda3_crypt UUID=X /dev/disk/by-label/keydrive:/key.bin:10 luks,discard,keyscript=/lib/cryptsetup/scripts/passdev
```

Update initramfs:

```shell
sudo update-initramfs -u
```

Restart the system to confirm change:

```shell
sudo shutdown -r now
```

## Auto Update OS

Keep your software up to date, automatically.

Set up Unattended Upgrades:

```shell
sudo apt install unattended-upgrades -y
```

Update the config:

```shell
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:10";
```

Create auto-upgrade:

```shell
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

## Optional: Dynamic IP

If you are running your server from home you might have a dynamic IP address and
get locked out.

Create track ip script according to [`track-ip`](bin/track-ip), replacing `X`
with a unique url-safe base64 key between 11 and 22 characters:

```shell
sudo nano /usr/local/bin/track-ip
#!/bin/bash
set -e
exec &>> /var/log/track-ip.log
echo "$(date -Is) $(basename "$0"): send ip"
wget -q -O- --post-data "" "https://ip.leovandriel.com/X"
echo "$(date -Is) $(basename "$0"): ip sent"
```

Make script executable:

```shell
sudo chmod +x /usr/local/bin/track-ip
```

Setup log file:

```shell
sudo touch /var/log/track-ip.log
sudo chown X:X /var/log/track-ip.log
```

Update `crontab` according to [`crontab`](etc/crontab):

```shell
sudo -u X crontab -e
*/5 * * * * /usr/local/bin/track-ip
```

Test script:

```shell
sudo -u X /usr/local/bin/track-ip
tail /var/log/track-ip.log
```

Now, you can [https://ip.leovandriel.com/X](https://ip.leovandriel.com/X),
replacing `X`.

## License

MIT
