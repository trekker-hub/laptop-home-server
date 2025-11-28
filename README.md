# Convert Old Laptop into Self-Hosted Cloud Server

A complete guide to transform an old laptop into a powerful self-hosted Nextcloud server for private cloud storage accessible from anywhere.

## Table of Contents
- [What You'll Build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Hardware Requirements](#hardware-requirements)
- [Part 1: Ubuntu Server Installation](#part-1-ubuntu-server-installation)
- [Part 2: Nextcloud AIO Setup](#part-2-nextcloud-aio-setup)
- [Part 3: Storage Optimization](#part-3-storage-optimization)
- [Part 4: External Access Configuration](#part-4-external-access-configuration)
- [Part 5: Laptop Configuration](#part-5-laptop-configuration)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

## What You'll Build

✅ **Private Cloud Storage** - Your own Dropbox/Google Drive that you fully control  
✅ **External Access** - Access files from anywhere in the world  
✅ **Collaboration Tools** - Nextcloud Office for documents, spreadsheets, presentations  
✅ **Video Calls** - Built-in video conferencing with Nextcloud Talk  
✅ **Calendar & Contacts** - Sync across all your devices  
✅ **Mobile Apps** - Android and iOS apps available  
✅ **File Sync** - Desktop clients for Windows/Mac/Linux  

**Cost:** ~$3-5/month in electricity vs $120+/year for cloud subscriptions

## Prerequisites

- Old laptop with at least 4GB RAM and 100GB storage
- Basic command line knowledge (I'll guide you through everything)
- Home network with router access
- USB drive (8GB+) for installation

## Hardware Requirements

**Minimum:**
- CPU: Dual-core processor
- RAM: 4GB (8GB+ recommended)
- Storage: 100GB (SSD preferred for OS, HDD for data)
- Network: WiFi or Ethernet

**Example Setup:**
- SSD: 238GB (for OS and applications)
- HDD: 932GB (for user data)
- RAM: 16GB
- WiFi: Connected to home network

## Part 1: Ubuntu Server Installation

### 1.1 Download Ubuntu Server

1. Download Ubuntu Server 24.04 LTS: https://ubuntu.com/download/server
2. Download Rufus: https://rufus.ie/

### 1.2 Create Bootable USB

1. Insert USB drive (**all data will be erased!**)
2. Open Rufus
3. Select your USB drive
4. Click "SELECT" and choose the Ubuntu ISO
5. Partition scheme: **GPT**
6. Click **START**

### 1.3 Install Ubuntu Server

1. Insert USB into laptop and restart
2. Press **F12** (or F2/Del) during startup to access boot menu
3. Select USB drive
4. Choose **"Install Ubuntu Server"**
5. Follow the installation wizard:
   - Language: Choose your language
   - Keyboard: Choose your layout
   - Network: Configure WiFi or Ethernet
   - Storage: **Use entire disk** (select SSD if you have both SSD and HDD)
   - Profile Setup:
     - Your name
     - Server name (e.g., `homeserver`)
     - Username
     - Password (**write this down!**)
   - SSH: **Check "Install OpenSSH server"**
   - Featured Server Snaps: **Skip Docker** (we'll install manually)

6. Wait for installation to complete
7. Remove USB when prompted and reboot

### 1.4 Initial Setup

From another computer, connect via SSH:
```bash
ssh yourusername@<server-ip>
```

(The server IP is shown on the laptop screen after boot)

Update the system:
```bash
sudo apt update && sudo apt upgrade -y
```

Install Docker:
```bash
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER
```

Log out and back in:
```bash
exit
```

Then SSH back in for Docker permissions to take effect.

## Part 2: Nextcloud AIO Setup

### 2.1 Install Nextcloud AIO Master Container
```bash
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 80:80 \
  --publish 8080:8080 \
  --publish 8443:8443 \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  nextcloud/all-in-one:latest
```

Wait for it to download and start (1-2 minutes).

### 2.2 Access Nextcloud AIO Interface

Open browser: `https://<server-ip>:8080`

You'll see a security warning - click **Advanced** → **Proceed** (this is normal)

**IMPORTANT: Copy and save the admin password shown on screen!**

### 2.3 Get a Free Domain (for External Access)

1. Go to https://www.duckdns.org
2. Sign in with GitHub/Google
3. Create a subdomain (e.g., `yourname.duckdns.org`)
4. Save your DuckDNS token

### 2.4 Configure Router Port Forwarding

Log into your router (usually `192.168.1.1` or `192.168.0.1`)

Add these port forwarding rules (all pointing to your server IP):

| External Port | Internal IP | Internal Port | Protocol |
|---------------|-------------|---------------|----------|
| 80 | `<server-ip>` | 80 | TCP |
| 443 | `<server-ip>` | 443 | TCP |
| 8443 | `<server-ip>` | 8443 | TCP |

### 2.5 Complete Nextcloud Setup

1. In Nextcloud AIO interface, enter your DuckDNS domain
2. Click **"Submit domain"**
3. Select optional containers:
   - ✅ Collabora (Nextcloud Office)
   - ✅ Imaginary (image previews)
   - ✅ Nextcloud Talk (video calls)
   - ✅ Whiteboard
4. Click **"Download and start containers"**
5. Wait 10-15 minutes for everything to download and start

You'll see a new admin password - **save it!**

Click **"Open your Nextcloud"** to access your cloud!

## Part 3: Storage Optimization

If you have both SSD and HDD, optimize for best performance:

### 3.1 Check Your Drives
```bash
lsblk
```

Identify which is SSD (usually `nvme0n1`) and HDD (usually `sda`).

### 3.2 Format HDD (if needed)

**⚠️ WARNING: This erases all data on the HDD!**
```bash
sudo umount /dev/sda1
sudo mkfs.ext4 /dev/sda1
```

### 3.3 Mount HDD
```bash
sudo mkdir -p /mnt/hdd
sudo mount /dev/sda1 /mnt/hdd
```

### 3.4 Auto-Mount on Boot
```bash
sudo nano /etc/fstab
```

Add this line at the bottom:
```
/dev/sda1 /mnt/hdd ext4 defaults 0 2
```

Save: `Ctrl+X`, `Y`, `Enter`

Reload:
```bash
sudo systemctl daemon-reload
```

### 3.5 Reconfigure Nextcloud for HDD Storage

**Stop Nextcloud:**

Go to `https://<server-ip>:8080` and click **"Stop containers"**

**Remove existing setup:**
```bash
docker stop nextcloud-aio-mastercontainer
docker rm nextcloud-aio-mastercontainer
```

Remove volumes (from AIO interface first, then):
```bash
docker volume rm nextcloud_aio_apache nextcloud_aio_database nextcloud_aio_database_dump nextcloud_aio_nextcloud nextcloud_aio_nextcloud_data nextcloud_aio_redis
```

**Create data directory:**
```bash
sudo mkdir -p /mnt/hdd/nextcloud-data
```

**Recreate with HDD storage:**
```bash
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 80:80 \
  --publish 8080:8080 \
  --publish 8443:8443 \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  --env NEXTCLOUD_DATADIR=/mnt/hdd/nextcloud-data \
  nextcloud/all-in-one:latest
```

**Verify data location:**
```bash
sudo ls -lah /mnt/hdd/nextcloud-data
```

After Nextcloud starts, you should see folders like `admin/`, `appdata_*/`.

## Part 4: External Access Configuration

### 4.1 DuckDNS Auto-Update

Create update script:
```bash
mkdir -p ~/duckdns
nano ~/duckdns/duck.sh
```

Add (replace `YOUR_DOMAIN` and `YOUR_TOKEN`):
```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=YOUR_DOMAIN&token=YOUR_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Make executable:
```bash
chmod +x ~/duckdns/duck.sh
```

Test:
```bash
~/duckdns/duck.sh
cat ~/duckdns/duck.log
```

Should show "OK".

### 4.2 Auto-Update Every 5 Minutes
```bash
crontab -e
```

Add this line:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

Save and exit.

### 4.3 Access Your Cloud

- **Locally:** `https://<server-ip>`
- **From anywhere:** `https://yourname.duckdns.org`

## Part 5: Laptop Configuration

### 5.1 Keep Running with Lid Closed
```bash
sudo nano /etc/systemd/logind.conf
```

Find these lines and change them:

**From:**
```
#HandleLidSwitch=suspend
#HandleLidSwitchExternalPower=suspend
#HandleLidSwitchDocked=ignore
```

**To:**
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Save: `Ctrl+X`, `Y`, `Enter`

Restart service:
```bash
sudo systemctl restart systemd-logind
```

Test by closing the lid - SSH should stay connected!

### 5.2 Ensure Good Ventilation

- Don't stack anything on the laptop
- Consider a laptop cooling pad
- Leave laptop slightly open if it gets hot
- Monitor temperature:
```bash
sensors
```

(Install if needed: `sudo apt install lm-sensors`)

## Troubleshooting

### Nextcloud Not Accessible Externally

- Check port forwarding (80, 443, 8443 → server IP)
- Verify DuckDNS: `cat ~/duckdns/duck.log` (should show "OK")
- Test ports: https://www.yougetsignal.com/tools/open-ports/
- Verify domain points to your public IP

### WiFi Disconnects
```bash
ip a  # Check connection
sudo netplan apply  # Reconnect
```

### SSH Not Working
```bash
sudo systemctl status ssh  # Check status
sudo systemctl start ssh  # Start if needed
sudo systemctl enable ssh  # Enable on boot
```

### Laptop Sleeps with Lid Closed

Verify configuration:
```bash
cat /etc/systemd/logind.conf | grep HandleLid
```

Should show `ignore` (no `#`). If not, edit and restart:
```bash
sudo systemctl restart systemd-logind
```

### HDD Not Mounting

Check fstab:
```bash
cat /etc/fstab
```

Should have HDD line. Test mount:
```bash
sudo mount -a
df -h  # Verify it's mounted
```

## Maintenance

### Update Ubuntu
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot  # If kernel updated
```

### Update Nextcloud

1. Go to `https://<server-ip>:8080`
2. Click **"Stop containers"**
3. Click **"Start and update containers"**

### Backup Nextcloud

**Setup:**
1. AIO interface → "Backup and restore"
2. Enter backup location (e.g., `/mnt/backup`)
3. Click **"Submit backup location"**

**Create backup:**
- Click **"Create backup"**
- Wait for completion

**Restore:**
- Enter backup location and password
- Click **"Restore"**

### Monitor Disk Space
```bash
df -h
sudo du -sh /mnt/hdd/nextcloud-data
```

### Monitor Resources
```bash
sudo apt install htop
htop  # Press Q to exit
docker stats  # Check container usage
```

## Security Best Practices

1. **Keep system updated:**
```bash
   sudo apt update && sudo apt upgrade -y
```

2. **Strong passwords** - Use unique, complex passwords

3. **Enable 2FA** in Nextcloud:
   - Settings → Security → Two-Factor Authentication

4. **Regular backups** - Configure automatic backups

5. **Monitor logs:**
```bash
   docker logs nextcloud-aio-nextcloud
```

6. **Firewall (optional):**
```bash
   sudo ufw enable
   sudo ufw allow 22,80,443,8080,8443/tcp
```

## What You Get

✅ **1TB+ Storage** (depending on your HDD)  
✅ **Access Anywhere** via web, mobile, desktop  
✅ **File Sync** across all devices  
✅ **Collaboration** - Shared folders, Office suite  
✅ **Privacy** - You own your data  
✅ **Cost Savings** - $36-60/year vs $120+/year for cloud subscriptions  

## Technologies Used

- **Ubuntu Server 24.04 LTS** - Operating system
- **Docker** - Container platform
- **Nextcloud AIO** - All-in-one cloud platform
- **DuckDNS** - Free dynamic DNS
- **Let's Encrypt** - Free SSL certificates

## Credits

Tutorial created based on personal implementation in November 2025.

## License

MIT License - Free to use and modify

## Contributing

Found an issue? Have improvements? Feel free to open an issue or pull request!

## Support

- **Nextcloud Documentation:** https://docs.nextcloud.com/
- **Nextcloud Community:** https://help.nextcloud.com/
- **Ubuntu Server Guide:** https://ubuntu.com/server/docs
