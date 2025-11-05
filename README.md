# RAID1-NAS-Server
How to create a RAID1 NAS Server
List of prompts

//Show filesystem types and usage
//Show block devices and any partitions
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT,MODEL

//show filesystem types and usage
lsblk -f /dev/sda /dev/sdb
blkid /dev/sda* /dev/sdb* || true

//print udev/device ID info (helps match physical drive/serial)
udevadm info --query=all --name=/dev/sda | egrep 'ID_SERIAL|ID_MODEL|ID_VENDOR'
udevadm info --query=all --name=/dev/sdb | egrep 'ID_SERIAL|ID_MODEL|ID_VENDOR'


//If lsblk shows /dev/sda1 or /dev/sdb1 already, note that — those partitions will be affected by formatting/RAID. If there are existing partitions with data, mount read-only for inspection:

sudo mkdir -p /mnt/tmp_sda /mnt/tmp_sdb
sudo mount -o ro /dev/sda1 /mnt/tmp_sda || echo "no partition/sda1"
sudo mount -o ro /dev/sdb1 /mnt/tmp_sdb || echo "no partition/sdb1"
ls -la /mnt/tmp_sda /mnt/tmp_sdb

//Run SMART checks (non‑destructive)
Install smart tools and run info + health checks

sudo apt update
sudo apt install smartmontools -y

//Drive info
sudo smartctl -i /dev/sda
sudo smartctl -i /dev/sdb

// Overall health (PASS/FAIL)
sudo smartctl -H /dev/sda
sudo smartctl -H /dev/sdb

//Full SMART attributes (scan for reallocated/pending)
sudo smartctl -A /dev/sda | egrep -i 'Realloc|Pending|Raw_Read|Error'
sudo smartctl -A /dev/sdb | egrep -i 'Realloc|Pending|Raw_Read|Error'

Destructive part

Mount and Setup
Use GPT (GUID Partition Table) which is a modern partition scheme and replaces MBR
Benefits: supports >2 TB (12 TB HDDs need GPT) more partitions and better integrity (works with UEFI and modern BIOS just fine).
Create RAID1 XFS or EXT4 (which is better?): For a home SMB NAS used mainly for back-ups from a laptop or phone, both XFS  or ext4 are fine. The choice is usually on comfort with recovery tools however if: 
You have large file sharing, multi TB (millions of files) and mostly big files with lots of directory activity (read and write): pick XFS (it is a high performance 64bit filesystem and it is the default in RedHat Enterprise Linux, but it doesn’t support journaling.
If you prefer simplest recovery tooling and conservative default that handles tiny files slightly better then pick ext4. In practice either work great, your network speed will matter more than the filesystem

//Create GPT + one partition per disk (DESTRUCTIVE)
sudo parted /dev/sda --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on
sudo parted /dev/sdb --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on

//Create RAID1 (metadata 1.2 is default; optional --name helps identify)
sudo apt install -y mdadm lvm2 xfsprogs
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 \
  --metadata=1.2 --name=nas-md0 \
  /dev/sda1 /dev/sdb1

//Watch sync progress
watch -n 2 cat /proc/mdstat  # Ctrl+C to exit

//Persist mdadm config (append) and initramfs
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

//LVM on top of RAID
sudo pvcreate /dev/md0
sudo vgcreate nas-vg /dev/md0
//Example split; adjust sizes to your needs
sudo lvcreate -L 9T -n media nas-vg
sudo lvcreate -l 100%FREE -n db nas-vg

//Filesystems (XFS for media; XFS for DB with CoW/reflink disabled OR ext4)
sudo mkfs.xfs -f /dev/nas-vg/media
sudo mkfs.xfs -f -m reflink=0 /dev/nas-vg/db
//(Alternative for DB)
// sudo mkfs.ext4 -F /dev/nas-vg/db

//Mount points
sudo mkdir -p /mnt/media /mnt/db
sudo mount /dev/nas-vg/media /mnt/media
sudo mount /dev/nas-vg/db /mnt/db

//Get UUIDS: 
sudo blkid /dev/nas-vg/media /dev/nas-vg/db

//Quick Check:

lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL,MOUNTPOINT
sudo parted -s /dev/sda unit MiB print
sudo parted -s /dev/sdb unit MiB print

Good signs: sda and sdb1 exist and span almost full disk, Type shows “part”.FSTYPE may be blank

//Qick Status: cat /proc/mdstat

Good signs: Line like: md0 : active raid1 sda1[0] sdb1[1]

Great! Check if mounted:
df -hT | grep -E 'nas-vg|mnt' && echo "---" && ls -la /mnt/
ls -la /mnt/
findmnt /mnt/media /mnt/db || echo "Not mounted"
	
If not Mounted then:

sudo mount /dev/nas-vg/media /mnt/media && sudo mount /dev/nas-vg/db /mnt/db && df -hT /mnt/media /mnt/db

Great! Now let’s get the UUIDs and add them to fstab so they automount on boot:

sudo blkid /dev/nas-vg/media /dev/nas-vg/db

echo "UUID=29bf93cf-39e8-4151-8d8a-aa999687d2a3  /mnt/media  xfs  defaults,noatime  0  2
UUID=4891daf8-d79a-4eb3-a647-25203ea2dd0d  /mnt/db     xfs  defaults,noatime  0  2" | sudo tee -a /etc/fstab

Test fstab

sudo mount -a && echo "fstab test successful"

How to replace a failed HDD

//1. Check array status (shows which disk failed)
cat /proc/mdstat
sudo mdadm --detail /dev/md0

//2. Mark failed disk (if not auto-detected)
sudo mdadm --manage /dev/md0 --fail /dev/sda1

//3. Remove failed disk from array
sudo mdadm --manage /dev/md0 --remove /dev/sda1

//4. Physically replace the drive, boot

//5. Partition new drive (copy layout from good drive)
sudo sfdisk -d /dev/sdb | sudo sfdisk /dev/sda

//6. Add new partition to array
sudo mdadm --manage /dev/md0 --add /dev/sda1

//7. Watch resync progress
watch cat /proc/mdstat
//or
sudo mdadm --detail /dev/md0


Secure remote access (outside your home network)
Don’t expose SMB directly to the internet. Use one of:

WireGuard VPN (recommended): install WireGuard on NAS and clients (laptop/phone), forward one UDP port on your router, then access SMB as if you were at home.
Nextcloud: app + web UI for uploads/sync; put data directory on /mnt/media or /mnt/db; access via HTTPS (with Let’s Encrypt) or via a secure tunnel.
I can give you a minimal, copy-paste WireGuard setup or a Docker Compose for Nextcloud if you want that next.
Nord VPN which also has Meshnet configuration;
//Download and install from NordVPN website
sh <(curl -sSf https://downloads.nordcdn.com/apps/linux/install.sh)
sudo usermod -aG nordvpn $user
//log out and back in
nordvpn login
nordvpn set meshnet on

//1.Turn into a NAS(LAN access with SAMBA)
sudo apt install -y samba
# use an existing Linux user or create one
# sudo adduser nasuser
sudo smbpasswd -a whatever
sudo smbpasswd -e whatever

//2. Append shares (replace 'nasuser' if needed)
sudo tee -a /etc/samba/smb.conf > /dev/null << 'EOF'

[Media]
   path = /mnt/media
   browsable = yes
   read only = no
   guest ok = no
   valid users = nasuser
   create mask = 0660
   directory mask = 0770

[DB]
   path = /mnt/db
   browsable = yes
   read only = no
   guest ok = no
   valid users = nasuser
   create mask = 0660
   directory mask = 0770
EOF

//3. Create Samba user (option: use existing user 'xyz' or create 'nasuser')
# For existing user:
sudo smbpasswd -a nauser
sudo smbpasswd -e nauser

//4. Set permissions
sudo chown -R nauser:nauser /mnt/media /mnt/db
sudo chmod -R 770 /mnt/media /mnt/db

//5. Validate config
testparm -s

//6. Restart Samba
sudo systemctl restart smbd
sudo systemctl status smbd --no-pager

//7. Open firewall
sudo ufw allow Samba

//8. Test
smbclient -L localhost -U nauser

//List shares
smbclient -L localhost -U nauser
//Enter your Samba password when prompted
//You should see [Media] and [DB] in the output

ip -br addr show | grep UP
//or
hostname -I

Add more storage to your NAS (let’s say we have bought and installed a new 2tb ssd and it’s a xfs filesystem:

ls -ld /storage && df -hT /storage

sudo tee -a /etc/samba/smb.conf > /dev/null << 'EOF'

[Storage]
   path = /storage
   browsable = yes
   read only = no
   guest ok = no
   valid users = nauser
   create mask = 0660
   directory mask = 0770
   comment = Fast NVMe SSD Storage
EOF
testparm -s 2>/dev/null | grep -A8 "\[Storage\]"

testparm -s 2>/dev/null | grep -A8 "\[Storage\]" && sudo systemctl restart smbd


Connect from devices:
Windows: \<nas-ip>\Media and \<nas-ip>\DB
macOS/iOS: smb://<nas-ip>/Media (Files app > Connect to Server)
Linux: smb://<nas-ip>/Media in file manager, or mount with cifs
Android: Solid Explorer / CX File Explorer with SMB3
Secure remote access (outside your home network)
Don’t expose SMB directly to the internet. Use one of:

WireGuard VPN (recommended): install WireGuard on NAS and clients (laptop/phone), forward one UDP port on your router, then access SMB as if you were at home.
Nextcloud: app + web UI for uploads/sync; put data directory on /mnt/media or /mnt/db; access via HTTPS (with Let’s Encrypt) or via a secure tunnel.
I can give you a minimal, copy-paste WireGuard setup or a Docker Compose for Nextcloud if you want that next.

//Browse Media share locally
smbclient //localhost/Media -U nauser
//Type: ls
//Type: quit

//Check listening ports (should see 445, 139)
sudo ss -tlnp | grep smbd

Access problems to send media?
sudo chown -R nauser:nauser /mnt/media /mnt/db && sudo chmod -R 770 /mnt/media /mnt/db && ls -la /mnt/ | grep -E 'media|db'

How do create folders in your Media folder:
mkdir -p /mnt/media/Youtube && ls -la /mnt/media/

Summary of my NAS shares now:


How to install Docker and how to use it:

You use Docker volumes and bind mounts to map specific container paths to specific NAS locations:

docker run -v /storage/ai-models:/models my-ai-container
//Container sees /models, which is actually /storage/ai-models on the host
docker run -v /mnt/db/postgres-data:/var/lib/postgresql/data postgres
//Container's DB files live on redundant RAID1 storage
docker run -v /mnt/media:/media plex
//Container accesses your RAID1 media files

//docker-compose.yml example
version: '3'
services:
  postgres:
    image: postgres:16
    volumes:
      - /mnt/db/postgres:/var/lib/postgresql/data  //RAID1 for safety
  
  ai-service:
    image: my-ai-model
    volumes:
      - /storage/ai-models:/models // SSD for speed
  
  nextcloud:
    image: nextcloud
    volumes:
      - /mnt/media/nextcloud-data:/var/www/html/data  // RAID1 for user files

NEXTCLOUD:

Create Docker Compose for Nextcloud + PostgreSQL database + Nginx Proxy Manager
Store Nextcloud data on media (RAID1)
Store database on db (RAID1)
Set up reverse proxy for HTTPS access
Show how to access from phone/laptop

mkdir -p /mnt/db/docker-configs/nextcloud && cd /mnt/db/docker-configs/nextcloud && pwd

Create docker-compose.yml
sudo systemctl start docker && sudo docker-compose up -d

Docker compose plugin
sudo apt update && sudo apt install -y docker-compose-v2

Start Nextcloud
sudo docker compose up -d

You should receive in the terminal the Nextcloud details like username and password which should be changed asap.

Check Nextcloud logs:

sudo docker logs nextcloud-app | tail -50

Check if everything is mounted in docker and nextcloud

ls -lah /mnt/media/nextcloud-data/

sudo docker exec -it nextcloud-app cat /var/www/html/config/config.php

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Media Storage" local null::null -c datadir=/media

Add media folder as external storage and create mount points for existing samba shares:

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Media Storage" local null::null -c datadir=/media

Add media storage to Nexcloud:

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Media Storage" local null::null -c datadir=/media

Check what is mounted in the container: 

sudo docker exec nextcloud-app ls -lah /media

sudo docker exec -u www-data nextcloud-app php occ files_external:list

Restart NextCloud container to apply changes:

sudo docker compose up -d

Check Mounts:

sudo docker exec nextcloud-app ls -lah /media

sudo docker exec nextcloud-app ls -lah /database

sudo docker exec nextcloud-app ls -lah /storage

Scan external storage so Nexcloud regocnizes your files:

sudo docker exec -u www-data nextcloud-app php occ files:scan --all

If there is a permission issue, just run:

sudo docker exec -u www-data nextcloud-app php occ files_external:delete -y 1 && sudo docker exec -u www-data nextcloud-app php occ files_external:delete -y 2

Add storage locations properly:

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Media (9TB)" local null::null -c datadir=/media

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Database (2TB)" local null::null -c datadir=/database]

sudo docker exec -u www-data nextcloud-app php occ files_external:create "Storage (2TB NVMe)" local null::null -c datadir=/storage

sudo chmod -R o+rX /mnt/media /mnt/db /storage

sudo docker exec nextcloud-app ls -la /media

Next Steps: Meshnet via NordVPN or Cloudflare Domain:

CloudFlare installation for HTTPS and extra security:

Create an account on cloudflare and register a domain where you can add the host to the domain then you will receive a DNS and you will complete the proxy setting after a tunnel. I personally don’t want to go with this step i will remain locally and use nordVPN to access my cloud.

Setup meshnet to nextcloud:

sudo docker exec -u www-data nextcloud-app php occ config:system:get trusted_domains



If Ubuntu OS drive fails:

❌ Docker images/containers (in /storage/Docker Storage) would need to be rebuilt
✅ Your actual data on media, db, storage survives (they're on separate physical drives)
✅ RAID1 protects media and db even if one HDD fails
⚠️ storage (SSD) has no redundancy — if the NVMe fails, that data is lost
Recovery steps:

Reinstall Ubuntu
Remount RAID1 (media, db) — data intact
Remount storage — data intact (unless the SSD itself failed)
Reinstall Docker
Recreate containers using docker-compose (images download again, but mount existing data volumes)
Your databases, AI models, media files are unchanged because they lived on the mounted volumes

Quick safety checklist:
✅ RAID1 protects against single HDD failure — you're safe for media and db
⚠️ SSD has no redundancy — backup important SSD data to RAID1
✅ Store docker-compose configs on RAID1 — easy recovery after OS reinstall
✅ fstab, smb.conf, wireguard configs — keep copies on RAID1
✅ Test recovery — simulate OS reinstall in a VM or test how to remount RAID and restart containers


Sefety Net Strategies
// Create a docker-compose.yml for each service
// Store it in /mnt/db/docker-configs/ (RAID1) or Git repo
mkdir -p /mnt/db/docker-configs
// After reinstall, just run:
docker-compose -f /mnt/db/docker-configs/nextcloud.yml up -d

// Regularly backup /storage important data to RAID1
rsync -av /storage/ai-models/ /mnt/media/backups/ai-models/
// Or use a cron job

// Keep a copy of key configs on RAID1
sudo cp /etc/fstab /mnt/db/backups/system-configs/
sudo cp /etc/samba/smb.conf /mnt/db/backups/system-configs/
sudo cp /etc/wireguard/wg0.conf /mnt/db/backups/system-configs/  # if you set up VPN

Use a backup OS drive (optional but paranoid-safe); Clone your Ubuntu system drive periodically to a spare drive

Data volumes (the important stuff):
Databases: db (RAID1 HDDs) — redundant, safe
AI models (in use): /storage/ai-models (SSD) — fast; backup to RAID1 periodically
Media/backups: media (RAID1 HDDs) — redundant, safe
Nextcloud user data: /mnt/media/nextcloud (RAID1) — redundant, safe

Example recovery test:
Note your current UUIDs: sudo blkid /dev/md0 /dev/nas-vg/*
Simulate reinstall: boot from live USB, manually mount RAID1, verify data
Reinstall Ubuntu, reinstall Docker, restore configs from /mnt/db/backups/, run docker-compose up

