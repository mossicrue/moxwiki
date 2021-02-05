# Extend a disk from fdisk
The following step shows how to resize a filesytem by extend the disk in which is placed.
The filesystem is `/` which is mapped with `/dev/mapper/ubuntu--vg-ubuntu--lv` on the partition `/dev/sda3`
The guides assume that the disk is already resized on the vcenter or any vm software.

1. Reboot the VM to recalculate the disk geometry.
   This is necessary because in case like vmware you can extend a disk without rebooting the server

2. Rescan all the SCSI connector to find the disk change, this can be done in 2 ways:
- using the rescan-scsi-bus.sh script
```
# rescan-scsi-bus.sh -a -w 0--15
```
- or via echo "- - -" to the SCSI controller
```
# ls /sys/class/scsi_host/
# echo "- - -" > /sys/class/scsi_host/hostX/scan
```

3. Check if the disk is now extended, for example /dev/sda
```
# fdisk -l /dev/sda
```

4. Then enter in the fdisk console to make all the change to the disk
```
# fdisk /dev/sda
```

5. Now we are in the fdisk prompt interface, all the next command must be issued in the fdisk session
    1. Print all the partition and find the one to resize, in this example the 3rd, and note the Partition Type
    ```
    Command (m for help): p

    Device       Start      End  Sectors Size Type
    /dev/sda1     2048     4095     2048   1M BIOS boot
    /dev/sda2     4096  2101247  2097152   1G Linux filesystem
    /dev/sda3  2101248 16775167 14673920   7G Linux filesystem
    ```
    2. Start the partition delete by running the command `d`
    ```
    Command (m for help): d
    ```
    3. Now choose the partition to delete, this action will not delete the data
    ```
    Partition number (1-3, default 3): 3
    Partition 3 has been deleted.
    ```
    4. Now create a new 3rd partition but with a bigger number of sector running the `n` command
    ```
    Command (m for help): n
    ```
    5. Insert its partition number: 3
    ```
    Partition number (3-128, default 3): 3
    ```
    6. Now select the first and last sector by using the default value, just press enter
    ```
    First sector (2101248-25165790, default 2101248):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2101248-25165790, default 25165790):
    Created a new partition 3 of type 'Linux filesystem' and of size 11 GiB.
    ```
    7. If a text containing this snippet appear, chose no issuing `N`
    ```
    Partition #3 contains a LVM2_member signature.
    Do you want to remove the signature? [Y]es/[N]o: N
    ```
    8. Now run the `p` command to print all the partitions and check the created one
    ```
    Command (m for help): p

    Device       Start      End  Sectors Size Type
    /dev/sda1     2048     4095     2048   1M BIOS boot
    /dev/sda2     4096  2101247  2097152   1G Linux filesystem
    /dev/sda3  2101248 25165790 23064543  11G Linux filesystem
    ```
    9. If the Partition Type is incorrect for any reasons, run the `t` command and follow the Type Change prompts
    ```
    Command (m for help): t
    ```
    10. Insert the partition number to change, 3
    ```
    Partition number (1-3, default 3): 3
    ```
    11. At the partition type selection prompt, first type `L` to see which ID the partition use, and then insert the right partition
    ```
    Partition type (type L to list all types): L
    20 Linux filesystem

    Partition type (type L to list all types): 20
    Changed type of partition 'Linux LVM' to 'Linux filesystem'.
    ```
    12. Finally, if everything it's correct, save all the work typing the `w` command
    ```
    Command (m for help): w
    The partition table has been altered.
    Syncing disks.
    ```

6. Now check the size of the physical volume (pv) associated to that partition
```
# pvs
PV         VG        Fmt  Attr PSize  PFree
/dev/sda3  ubuntu-vg lvm2 a--  <7.00g    0
```

7. Resize the pv
```
# pvresize /dev/sda3
Physical volume "/dev/sda3" changed
1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

8. Check if also the volume group (vg) has been extended, pvresize will also resize the vg
```
# vgs ubuntu-vg
VG        #PV #LV #SN Attr   VSize   VFree
ubuntu-vg   1   1   0 wz--n- <11.00g 4.00
```

9. Now extend the logical volume (lv)
```
# lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
Size of logical volume ubuntu-vg/ubuntu-lv changed from <7.00 GiB (1791 extents) to <11.00 GiB (2815 extents).
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

10. Finally resize the filesystem to the new size. This can be done in 2 ways depending on filesystem type:
  - xfs
```
# xfs_growfs /
```
  - ext4, ext3, extX ...
```
# resize2fs /dev/ubuntu-vg/ubuntu-lv
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/ubuntu-vg/ubuntu-lv is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/ubuntu-vg/ubuntu-lv is now 2882560 (4k) blocks long.
```
