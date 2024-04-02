1. **Format Drives**
 This command will iterate over the specified NVMe drives, create the partitions, and format them!

```bash
for drive in /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1; do
  sudo fdisk $drive <<EOF
g
n
p
1

+550M
t
1
n
p
2


w
EOF
  sudo mkfs.fat -F 32 "${drive}p1"
  sudo mkfs.ext4 "${drive}p2"
done
```

2. **Assemble RAID 10 Array**
Make sure you're connected to the internet first. Then from the terminal, run the following commands to update the package list and install mdadm, and assemble the Raid 10 array.

```bash
sudo apt update && sudo apt install mdadm && sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/nvme0n1p2 /dev/nvme1n1p2 /dev/nvme2n1p2 /dev/nvme3n1p2
```

Wait for the array to be assembled and monitor its progress with the cat script below, this is simple command that displays the output to the command prompt. This process could take up to 5+ hours depending on drive size and is honestly the worst part of this whole thing as you'll have to start all over if something goes wrong and you end up needing to reboot the install. Organize your day around this.

```bash
cat /proc/mdstat
```

3. **Install Ubuntu**:
Double-click the "Install Ubuntu" icon on the desktop to start the installation process, then when prompted for partitioning, select *Something Else*. Select */dev/md0* and click *New Partition Table*, creating */dev/md0p1*, then perform the following:

- Format **/dev/md0p1** as ext4 and set the Mount point: /
- Select **/dev/nvme0n1p1** and set it as the *EFI System Partition*, then manually format it from the termainal with `sudo mkfs.fat -F 32 /dev/nvme0n1p1` as Ubuntu cannot automatically format it during the installation process.
- *Set Device for boot loader installation:* to */dev/nvme0n1* from the dropdown menu.
- Proceed with setting up the rest of the Ubuntu installation.

4. **Post-Installation Setup**:
Reguardles if the installation completes or fails with an expected grub-install error, choose *Continue Testing* to enter a live environment, open the terminal, run: `lsblk`, and make sure /target is mounted (example below), then the fun stuff begins.

```bash
nvme3n1     259:0    0 372.6G  0 disk  
├─nvme3n1p1 259:10   0   550M  0 part  
└─nvme3n1p2 259:11   0 372.1G  0 part  
  └─md0       9:0    0   1.5T  0 raid10
    └─md0p1 259:13   0   1.5T  0 part  /target
nvme2n1     259:1    0 372.6G  0 disk  
├─nvme2n1p1 259:8    0   500M  0 part  
└─nvme2n1p2 259:9    0 372.1G  0 part  
  └─md0       9:0    0   1.5T  0 raid10 
    └─md0p1 259:13   0   1.5T  0 part  /target
nvme1n1     259:2    0 372.6G  0 disk  
├─nvme1n1p1 259:6    0   500M  0 part  /target/boot
└─nvme1n1p2 259:7    0 372.1G  0 part  
  └─md0       9:0    0   1.5T  0 raid10 
    └─md0p1 259:13   0   1.5T  0 part  /target
nvme0n1     259:3    0 372.6G  0 disk  
├─nvme0n1p1 259:4    0   500M  0 part  /target/boot/efi
└─nvme0n1p2 259:5    0 372.1G  0 part  
  └─md0       9:0    0   1.5T  0 raid10 
    └─md0p1 259:13   0   1.5T  0 part  /target
```

5. **chroot**
The purpose of these commands is to set up a chroot environment. By binding the /dev, /proc, and /sys directories and then using chroot, you create an environment where you can work on the freshly installed system as if it were the root filesystem. This allows you to perform system-level tasks, such as installing packages, and updating configurations, on the new system without actually booting into it.

Chrooting is commonly used in situations like system recovery, installation scripts, or when you need to perform maintenance tasks on a different Linux installation than the one you're currently running.

```bash
cd /target
mount --bind /dev dev
mount --bind /proc proc
mount --bind /sys sys
chroot .
```

6. **Install mdadm**
```bash
apt install mdadm
```
If you encounter a DNS error, add a nameserver to `/etc/resolv.conf`:

```bash
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

Then, re-run the `apt install mdadm` command.

7. **Configure mdadm**
Check the contents of **sudo nano /etc/mdadm/mdadm.conf**. It should contain information about your RAID array at the bottom:
   
```bash
ARRAY /dev/md/0  metadata=1.2 UUID=000000:000000:000000:000000 name=ubuntu:0
```

If the file is empty or not found, manually create the configuration:

```bash
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

8. **Configure RAID module loading at boot**
Add your RAID10 level to `/etc/modules`.

```bash
echo raid10 >> /etc/modules
```

9. **Verify initramfs**
Check if the initial ramdisk contains the necessary mdadm files:

```bash
lsinitramfs /boot/initrd.img | grep mdadm
```

You should see output similar to:

```bash
etc/mdadm
etc/mdadm/mdadm.conf
etc/modprobe.d/mdadm.conf
scripts/local-block/mdadm
scripts/local-bottom/mdadm
usr/sbin/mdadm
```

10. **Verify /boot partition:**
Use `lsblk` to check if `/boot` is a separate partition. It should be located on one of the disks, not on the RAID array.

7. **Update GRUB:**
Run `update-grub2` to update the GRUB configuration:

```bash
update-grub2
```

8. **Reboot**
If everything is configured correctly, your system should boot normally. If you encounter a GRUB2 prompt, it means GRUB2 is installed but cannot find its configuration. In this case, boot from the live installer USB again, open terminal, and run `update-grub2` from the chroot environment as described in step 5. You should have a working Ubuntu 22.04 system with mdadm RAID10 configured.
