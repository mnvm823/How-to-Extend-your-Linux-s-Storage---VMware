# How-to-Extend-your-Linux-s-Storage---VMware
> Recently I've encountered this problem on VMWare while extending the storage of my Kali Linux Machine
---
 
## Overview
 
| | |
|---|---|
| **Environment** | Kali Linux (Rolling) VM on VMware Workstation |
| **Symptom** | VM boots to a black screen with a blinking cursor, no login screen |
| **Root cause** | Root partition (`/dev/sda1`) was 100% full, so the display manager couldn't start |
| **Fix** | Free a little space, remove the swap partitions blocking growth, grow the partition and the filesystem |
| **Result** | `/dev/sda1` went from 24G (100% used) to 50G (47% used) |
 
---
 
## Key concept: two places need resizing
 
Extending the disk in VMware **only makes the virtual disk bigger**. Linux does not use that space automatically. Three layers must be grown, in order:
 
1. **Virtual disk** (VMware settings): done on the host
2. **Partition** (`sda1`): done inside the VM with `parted`
3. **Filesystem** (ext4): done inside the VM with `resize2fs`
A common mistake is stopping after step 1 and wondering why `df -h` still shows the old size.
 
### What does `sda` mean?
 
- `sd` = SCSI/SATA-type disk (how VMware presents the virtual disk)
- `a` = the first disk (a second disk would be `sdb`)
- Numbers (`sda1`, `sda2`, `sda5`) are **partitions** on that disk
Think of `sda` as a whole pizza and the numbered partitions as slices. VMware made the pizza bigger, but the slices stayed the same size.
 
---
 
## Part 1: Symptoms
 
After extending the disk, the VM powers on but only shows a **black screen with a blinking cursor** in the top-left corner. There is no login screen and no desktop.
 
This looks like a boot failure, but often it isn't: the system booted fine and only the **graphical session failed to start**. A full disk is a common reason, because the desktop can't write the temporary files it needs.
 
---
 
## Part 2: Get a terminal (Ctrl+Alt+F2)
 
Click inside the VM window, then press:
 
```
Ctrl + Alt + F2
```
 
(Try `F3` if `F2` doesn't work.) If you get a login prompt like this, the system booted correctly:
 
```
Kali GNU/Linux Rolling kali tty2
kali login:
```
 
Log in with your normal user (e.g. `kali`) and password.
 
> **Tip:** If `Ctrl + Alt + F2` is captured by the host, click inside the VM first, and make sure the VMware window has focus.
 
---
 
## Part 3: Diagnose from the terminal
 
### 3.1 Check disk usage
 
```bash
df -h /
```
 
Example output showing the problem:
 
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        24G   22G     0 100% /
```
 
`Avail` of `0` and `Use%` of `100%` confirm the disk is full.
 
### 3.2 Check the partition layout
 
```bash
lsblk
```
 
Example output:
 
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   50G  0 disk
├─sda1   8:1    0 23.7G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  1.3G  0 part [SWAP]
sr0     11:0    1 1024M  0 rom
```
 
How to read this:
 
- `sda` is **50G**, so VMware's resize worked
- `sda1` is still **23.7G**, so the partition has not grown yet
- `sda2` (1K) is an **extended partition container**, and `sda5` (1.3G) is the **swap** inside it
- The new space sits **after** sda5, so `sda1` can't reach it until `sda2` and `sda5` are removed
This is the most important finding: **the swap partition physically blocks root from growing.**
 
### 3.3 Check for snapshots (host side)
 
VMware can't safely resize disks that have snapshots. In VMware, power off the VM and check **VM > Snapshot > Snapshot Manager**. Delete any snapshots before resizing.
 
---
 
## Part 4: Get the desktop back with `startx` (workaround)
 
While the disk is still full, the normal login screen (lightdm) may fail to start. From the terminal you can still launch a desktop directly:
 
```bash
startx
```
 
> **Important:** if you run `startx` from a root shell, the desktop opens as **root**. That desktop shows `/root` as Home, **not** your normal files.
>
> Your files are **not deleted**. They live in `/home/kali`. Check with:
>
> ```bash
> whoami
> ls -la /home/kali
> ```
>
> Once the disk is fixed and you log in normally as `kali`, everything is back.
 
`startx` is a workaround only. Running a desktop as root isn't recommended for daily use.
 
---
 
## Part 5: The fix
 
> If you are already **root**, leave out `sudo` in every command. If you are a normal user, add `sudo`.
 
### Step 1: Free a little space
 
You need a few hundred MB free for the tools to work.
 
```bash
apt clean
journalctl --vacuum-size=50M
df -h /
```
 
`Avail` should now be above 0 (in the real case it went to 351M).
 
If it is still 0, find what's using space:
 
```bash
du -xh / --max-depth=2 | sort -h | tail -15
```
 
### Step 2: Turn off swap and remove it from fstab
 
```bash
swapoff -a
sed -i '/swap/ s/^/#/' /etc/fstab
grep swap /etc/fstab
```
 
The `grep` output should show the swap line **starting with `#`**. Without this, boot may wait around 90 seconds for a swap partition that no longer exists.
 
### Step 3: Delete the swap partitions and grow root with `parted`
 
Start parted on the **whole disk** (`/dev/sda`, not `/dev/sda1`):
 
```bash
parted /dev/sda
```
 
At the `(parted)` prompt, type each command **one at a time** and press Enter after each:
 
```
print
rm 5
rm 2
print
resizepart 1 100%
print
quit
```
 
Checklist while doing this:
 
- After the first `print`, confirm you see partitions 1, 2 and 5
- After `rm 5` and `rm 2`, `print` again and confirm **only partition 1 remains**
- If `resizepart` warns the partition is in use, answer `Yes` (normal, because root is mounted)
- The last `print` should show `sda1` extending to about 53.7 GB (the end of the 50 GB disk)
- **Never run `rm 1`.** That is your root partition with all your data
> **Common mistake: do not paste the commands as one block.**
> If you paste them all at once, parted reads them as one long invalid command and prints its help list repeatedly, and nothing changes. In the real case this is exactly what happened the first time. Type them one at a time.
 
When you quit you may see: `Information: You may need to update /etc/fstab.` This is expected, because the swap entry was already commented out in Step 2.
 
### Step 4: Grow the filesystem
 
```bash
partprobe /dev/sda
resize2fs /dev/sda1
df -h /
```
 
Successful output:
 
```
Filesystem at /dev/sda1 is mounted on /; on-line resizing required
The filesystem on /dev/sda1 is now 13106944 (4k) blocks long.
 
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   22G   26G  47% /
```
 
If `resize2fs` says `Nothing to do`, either the partition was not grown yet (go back to Step 3), or the kernel hasn't picked up the new size. In that case run `reboot`, log in, and run `resize2fs /dev/sda1` again.
 
### Step 5: Reboot and test the normal login
 
```bash
reboot
```
 
At the login screen, log in as your normal user (`kali`, not root). The desktop and files should be back.
 
---
 
## Part 6: Optional: add swap back as a swapfile
 
The swap partition is gone, so the system has no swap. With 8 GB of RAM it will run fine, but a swapfile is a good safety net:
 
```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
free -h
```
 
`free -h` should show about 2G of swap.
 
---
 
## Part 7: Keeping the disk from filling up again
 
- Check usage regularly with `df -h /`
- Find big folders: `du -xh / --max-depth=2 | sort -h | tail -15`
- Clean the apt cache: `apt clean`
- Limit journal logs: `journalctl --vacuum-size=100M`
- Watch for large wordlists, old tool downloads, and VM-created files in your home folder
---
 
## FAQ
 
### Can I shrink the disk back to 25 GB?
 
VMware **cannot shrink** a virtual disk, and you usually don't need to:
 
- If the disk is not preallocated, the 50 GB is only a **maximum**. The host only uses the space Linux actually fills (check VM Settings > Hard Disk > Capacity).
- Options if you really want it smaller: restore a backup or snapshot from before the resize, or clone to a new smaller disk with Clonezilla (slow and fiddly).
- The **Compact** button only reclaims unused space inside the file. It does not lower the maximum size.
### Why are my files missing after `startx`?
 
You are probably logged in as root, which has its own home folder (`/root`). Your real files are in `/home/kali` and are untouched.
 
### Why did `resize2fs` say "Nothing to do"?
 
The partition was still the old size. Grow the partition first (`parted ... resizepart`), then run `resize2fs`.
 
---
 
## Quick reference (cheat sheet)
 
```bash
# 1. Diagnose
df -h /
lsblk
 
# 2. Free space
apt clean
journalctl --vacuum-size=50M
 
# 3. Disable swap
swapoff -a
sed -i '/swap/ s/^/#/' /etc/fstab
 
# 4. Remove swap partitions and grow root (type inside parted, one at a time)
parted /dev/sda
#   print
#   rm 5
#   rm 2
#   resizepart 1 100%
#   print
#   quit
 
# 5. Grow the filesystem
partprobe /dev/sda
resize2fs /dev/sda1
df -h /
 
# 6. Reboot
reboot
```
 
> **Note:** This layout (root `sda1`, extended `sda2`, swap `sda5`) is the Kali/Debian default MBR install. Your partition numbers may differ, so always run `lsblk` first and adjust. If root is already the **last** partition on the disk, skip the swap removal and just use `growpart /dev/sda 1` (from `cloud-guest-utils`) followed by `resize2fs`.
 
