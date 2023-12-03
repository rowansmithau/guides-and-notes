### Covers the process of resizing the partition on a CentOS/RHEL/Rocky/? based distro using LVM and XFS or EXT4 after you have increased the physical disk size.

1. Confirm that Linux sees increased disk size
```
$ lsblk
NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda              8:0    0   50G  0 disk
├─sda1           8:1    0    1G  0 part /boot
└─sda2           8:2    0   19G  0 part
  ├─rl_k1-root 253:0    0   17G  0 lvm  /
```  

2. Determine the disk, use LVM tools to analyse
```
$ df -h | grep mapper
/dev/mapper/rl_k1-root   17G   13G  4.5G  74% /

$ pvdisplay
...
  PV Size               <19.00 GiB / not usable 3.00 MiB
  
$ lvdisplay
  --- Logical volume ---
  LV Path                /dev/rl_k1/root
  LV Name                root
  ...
  LV Size                <17.00 GiB
```

3. Extend the partition in fdisk (delete + recreate (no data is lost))

```
$ fdisk /dev/sda
```

p (print disk info - record the output)
```
Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x7f689156

Device     Boot   Start      End  Sectors Size Id Type
/dev/sda1  *       2048  2099199  2097152   1G 83 Linux
/dev/sda2       2099200 41943039 39843840  19G 8e Linux LVM
```

d (delete, select LVM partition (2 for me))
n (new partition)
enter (primary partition)
enter (partition number 2)
enter (after confirming first sector it suggests matches the `Start` value from the fdisk output for your partition)
enter (end of disk should automatically be selected)
N (No, you do not want to remove the existing LVM2_member signature)
w (write and exit)

4. Run partprobe to have the OS pick up the changes

```
$ partprobe
```

5. Extend the physical volume group

```
$ pvresize /dev/sda2
  Physical volume "/dev/sda2" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

6. Confirm the new size is displayed

```
$ pvdisplay
...
  PV Size               <49.00 GiB / not usable 2.00 MiB
```

7. Resize the logical volume group to use 100% of available free space

```
$  lvresize -l +100%FREE /dev/mapper/rl_k1-root
  Size of logical volume rl_k1/root changed from <17.00 GiB (4351 extents) to <47.00 GiB (12031 extents).
  Logical volume rl_k1/root successfully resized.
```

8.. Resize the XFS partition to make use of the new space

```
$ xfs_growfs /dev/mapper/rl_k1-root
...
data blocks changed from 4455424 to 12319744
```

9. Confirm that OS sees the new size / increase free space

```
$ df -h | grep mapper
/dev/mapper/rl_k1-root   47G   13G   35G  28% /
```
