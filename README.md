# Home server journey (from scratch using Debian GNU/Linux)

## Goals
1. Local NAS
2. Home assistant
3. Pi-hole
4. Frigate NVR
5. Nextcloud
6. Immich
7. Jellyfin?

## OS choice
There are many options here. I decided to go for Debian stable as a base and just use docker to manage container deployments of most of the apps/services. My hardware is a bit different, but this is a pretty detailed guide I found online: https://tongkl.com/building-a-nas-part-1/.

## Hardware

### Processor and motherboard
I initially wanted a system with Intel N100 processor, but couldn't really find the ASUS N100 motherboard, except through shady third parties.
In the end I was able to snatch an Intel i5-14500 processor for half the price (at the time) and decided to get an [ASUS PRIME H610I-PLUS D4-CSM](https://www.asus.com/us/motherboards-components/motherboards/prime/prime-h610i-plus-d4-csm/) to go along with it.

### Memory
TODO

### OS drive
TODO

### Power supply
TODO

### Case
I chose the Jonsbo N3 for its compactness.

### Hard drives
This was a bit of an overkill. I probably should have gotten something smaller. I got 4 x Seagate EXOS 14Tb HDD.

## Power management
I have been unable to get the CPU to get past C2 states with USB ZigBee and ZWave dongles plugged in and C3 state if unplugged.
I tried forcing better power management by the OS (forcing L1 on Realtek LAN PCI device), but that yielded no difference vs useing  the BIOS settings (which force enable L1 on the Realtek LAN device by default).
All the powertop tunings are in place using TLP.

## Updating home assistant container
```bash
sudo docker ps
sudo docker stop homeassistant
sudo docker rm homeassistant
docker pull ghcr.io/home-assistant/home-assistant:stable
sudo docker pull ghcr.io/home-assistant/home-assistant:stable
sudo docker-compose up -d
```

## ZFS pool
Decided to use `raidz2` for the 4 drives. This offers tolerance for 2 drive failures, while still providing about 50% of the space, which is plenty for me. I used the `ashift=12` option since these drives are modern and suppprot 4KiB sector size.
```bash
sudo zpool create -o ashift=12 pool raidz2 /dev/disk/by-id/ata-ST14000NM000J-2TX103_XXXXXXX1 /dev/disk/by-id/ata-ST14000NM000J-2TX103_XXXXXXX2 /dev/disk/by-id/ata-ST14000NM000J-2TX103_XXXXXXX3 /dev/disk/by-id/ata-ST14000NM000J-2TX103_XXXXXXX4
```
Used the following global settings on my ZFS pool for optimizing it to be used as SMB (Samba) shares as well as providing fast compression.
```bash
sudo zfs set compression=lz4 pool
sudo zfs set xattr=sa pool
sudo zfs set acltype=posixacl pool
sudo zfs set atime=off pool
sudo zfs set mountpoint=/mnt/pool pool
```
### ZFS datasets
For my use cases I decided to break down the ZFS pool into the following datasets:
1. userdata
   Starting with two users (one of them being on a Mac)
2. media (shared)
   1. Photos
   2. Music
   3. Videos
   4. Camera
3. backups (shared)
   1. timemachine

#### User data
Create general `userdata` dataset and individual user-level sub-datasets.
```bash
sudo zfs create pool/userdata
sudo zfs set recordsize=128K pool/userdata
sudo zfs create pool/userdata/user1
sudo zfs create -o normalization=formD pool/userdata/user2
```
The `normalization=formD` is meant to help with the Mac User 2.
The `recordsize=128K` can hopefully help with performance for small and medium large files.

#### Shared Media
Create general `media` dataset.
```bash
sudo zfs create pool/media
```
Media dataset hierarchy:
1. Photos (optimizing for smaller files):
   ```bash
   sudo zfs create pool/media/photos
   sudo zfs set recordsize=128K pool/media/photos
   ```
2. Videos (optimizing for larger files):
   ```bash
   sudo zfs create pool/media/videos
   sudo zfs set recordsize=1M pool/media/videos
   ```
3. Music (optimizing for smaller files):
   ```bash
   sudo zfs create pool/media/music
   sudo zfs set recordsize=128K pool/media/music
   ```
4. Camera (optimizing for larger files and live recording):
   ```bash
   sudo zfs create pool/media/camera
   sudo zfs set compression=off recordsize=1M logbias=throughput sync=always pool/media/camera
   ```
#### Backups
Create general `backups` dataset.
```bash
sudo zfs create pool/backups
```
Backups dataset hierarchy:
1. timemachine (For Apple users):
   ```bash
   sudo zfs create pool/backups/timemachine
   sudo zfs set quota=4T pool/backups/timemachine
   ```
2. others (for server and PC backups)
   ```bash
   sudo zfs create pool/backups/others
   sudo zfs set quota=4T pool/backups/others
   ```

## HDD maintenance
### SMARTD
Customize smartd.conf (TODO)

### ZFS Trimming
Disable ZFS trimming in cron.d (since using HDD only pool)

### ZFS Scrubs
Monthly (TODO)

### ZFS Snapshots
Frequency policy managed by `zfs-auto-snapshot`

#### User Data
Daily, keep for 1 month.

```
sudo zfs set com.sun:auto-snapshot:daily=30 pool/userdata
```

#### Media
Weekly, keep for 6 months.

```
sudo zfs set com.sun:auto-snapshot:weekly=26 pool/media
```

Disable snapshoting of `camera` sub-dataset (live camera recordings snapshots are not recommended due to rapid write churn)

```
sudo zfs set com.sun:auto-snapshot=false pool/media/camera
```

#### Backups
Weekly, keep for 3 months (including `timemachine`)

```
sudo zfs set com.sun:auto-snapshot:weekly=13 pool/backups
```



## Permissions and ownership
I decided to create a `nasusers` group:
```bash
sudo groupadd nasusers
```
Then add existing allowed users to this group:
```bash
sudo usermod -aG nasusers user1
```
For new users, then create the account (and password) first:
```bash
sudo useradd -m user2
sudo passwd user2
```
Now, for ownership, I have following scheme:
```bash
sudo chown -R :nasusers /mnt/pool/userdata
sudo chown -R user1:nasusers /mnt/pool/userdata/user1
sudo chown -R user2:nasusers /mnt/pool/userdata/user2
sudo chmod -R 770 /mnt/pool/userdata
sudo chown -R :nasusers /mnt/pool/media
sudo chmod -R 770 /mnt/pool/media
sudo chown -R :nasusers /mnt/pool/backups
sudo chmod -R 770 /mnt/pool/backups
```

## Samba configuration
It is important to add the unix user as a Samba users, like so:
```bash
sudo smbpasswd -a user1
sudo smbpasswd -a user2
```
In general I will try to keep the same password

Here's a few snippet configurations for the various datasets:

### Globals
Samba global settings for taking advantage of ZFS options: `xattr=sa` and `acltype=posixacl`
```
[global]
vfs objects = acl_xattr
map acl inherit = yes
store dos attributes = yes
```

### User Data
For each user like so:
```
[user1]
path = /mnt/pool/userdata/user1
valid users = user1
browseable = no
writable = yes
create mask = 0660
directory mask = 0770
```

### Media
```
[media]
path = /mnt/pool/media
valid users = @nasusers
browseable = yes
writable = yes
read only = no
create mask = 0660
directory mask = 0770
vfs objects = catia fruit streams_xattr
```

### Backups (time machine)
```
[timemachine]
path = /mnt/pool/backups/timemachine
valid users = @nasusers
browseable = no
writable = yes
vfs objects = catia fruit streams_xattr
fruit:aapl = yes
fruit:time machine = yes
create mask = 0600
directory mask = 0700
```

### Backups (others)
```
[others]
path = /mnt/pool/backups/others
valid users = @nasusers
browseable = no
writable = yes
create mask = 0600
directory mask = 0700
```

## Other debian configurations
- Set up unattended-upgrades
