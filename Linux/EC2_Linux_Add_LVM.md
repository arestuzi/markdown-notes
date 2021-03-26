# How to migrate RHEL 8.1 to LVM

1. 使用RHEL-8.1.0_HVM-20191029-x86_64-0-Hourly2-GP2 (ami-0f3aafab00d4e2fe6)创建一个第五代类型(t3, c5, m5,etc)的实例，并且挂载20GiB的EBS卷

2. 使用fdisk /dev/nvme1n1对新挂载的20G设备进行分区

    ```bash
    [root@ip-172-31-29-83 ec2-user]# fdisk /dev/nvme1n1 

    Welcome to fdisk (util-linux 2.32.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    Device does not contain a recognized partition table.
    Created a new DOS disklabel with disk identifier 0x8858d623.

    Command (m for help): n
    Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
    Select (default p): 

    Using default response p.
    Partition number (1-4, default 1): 
    First sector (2048-41943039, default 2048): 
    Last sector, +sectors or +size{K,M,G,T,P} (2048-41943039, default 41943039): 4095

    Created a new partition 1 of type 'Linux' and of size 1 MiB.

    Command (m for help): n
    Partition type
    p   primary (1 primary, 0 extended, 3 free)
    e   extended (container for logical partitions)
    Select (default p): 

    Using default response p.
    Partition number (2-4, default 2): 
    First sector (4096-41943039, default 4096): 
    Last sector, +sectors or +size{K,M,G,T,P} (4096-41943039, default 41943039): +1G

    Created a new partition 2 of type 'Linux' and of size 1 GiB.

    Command (m for help): n
    Partition type
    p   primary (2 primary, 0 extended, 2 free)
    e   extended (container for logical partitions)
    Select (default p): p
    Partition number (3,4, default 3): Enter
    First sector (2101248-41943039, default 2101248): Enter
    Last sector, +sectors or +size{K,M,G,T,P} (2101248-41943039, default 41943039): 

    Created a new partition 3 of type 'Linux' and of size 19 GiB.

    Command (m for help): t
    Partition number (1-3, default 3): Enter
    Hex code (type L to list all codes): 8e

    Command (m for help): p
    Disk /dev/nvme1n1: 20 GiB, 21474836480 bytes, 41943040 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x8858d623

    Device         Boot   Start      End  Sectors Size Id Type
    /dev/nvme1n1p1         2048     4095     2048   1M 83 Linux
    /dev/nvme1n1p2         4096  2101247  2097152   1G 83 Linux
    /dev/nvme1n1p3      2101248 41943039 39841792  19G 8e Linux LVM

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

3. 制作boot分区和LVM逻辑卷

    ```bash
    mkfs.xfs /dev/nvme1n1p2 
    yum install lvm2 -y
    vgcreate vgEBS /dev/nvme1n1p3
    lvcreate -L 5G -n lvRoot vgEBS
    lvcreate -L 512M -n lvHome vgEBS
    lvcreate -L 512M -n lvTmp vgEBS
    lvcreate -L 512M -n lvAudit vgEBS
    lvcreate -L 2G -n lvVar vgEBS
    lvcreate -L 2G -n lvLog vgEBS
    lvcreate -L 2G -n lvOpt vgEBS
    lvcreate -L 1G -n lvVtmp vgEBS
    lvcreate -L 2G -n lvVopt vgEBS
    lvcreate -L 3G -n lvSwap vgEBS
    [root@ip-172-31-29-83 ec2-user]# lsblk 
    NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    nvme1n1           259:0    0   20G  0 disk 
    ├─nvme1n1p1       259:4    0    1M  0 part 
    ├─nvme1n1p2       259:5    0    1G  0 part 
    └─nvme1n1p3       259:6    0   19G  0 part 
    ├─vgEBS-lvRoot  253:0    0    5G  0 lvm  
    ├─vgEBS-lvHome  253:1    0  512M  0 lvm  
    ├─vgEBS-lvTmp   253:2    0  512M  0 lvm  
    ├─vgEBS-lvAudit 253:3    0  512M  0 lvm  
    ├─vgEBS-lvVar   253:4    0    2G  0 lvm  
    ├─vgEBS-lvLog   253:5    0    2G  0 lvm  
    ├─vgEBS-lvOpt   253:6    0    2G  0 lvm  
    ├─vgEBS-lvVtmp  253:7    0    1G  0 lvm  
    ├─vgEBS-lvVopt  253:8    0    2G  0 lvm  
    └─vgEBS-lvSwap  253:9    0    3G  0 lvm  
    ```

4. 分别挂载以下分区至/mnt目录对应挂载点

    ```bash
    # Format the filesystem
    mkfs.xfs /dev/mapper/vgEBS-lvAudit 
    mkfs.xfs /dev/mapper/vgEBS-lvLog
    mkfs.xfs /dev/mapper/vgEBS-lvRoot
    mkfs.xfs /dev/mapper/vgEBS-lvTmp
    mkfs.xfs /dev/mapper/vgEBS-lvVopt
    mkfs.xfs /dev/mapper/vgEBS-lvHome
    mkfs.xfs /dev/mapper/vgEBS-lvOpt
    mkfs.xfs /dev/mapper/vgEBS-lvVar
    mkfs.xfs /dev/mapper/vgEBS-lvVtmp
    mkswap /dev/mapper/vgEBS-lvSwap

    # mount the root filesystem
    mount /dev/mapper/vgEBS-lvRoot /mnt/

    # Create directory
    mkdir /mnt/home
    mkdir /mnt/tmp
    mkdir /mnt/var    
    mkdir /mnt/opt
    mkdir /mnt/boot
    mkdir /mnt/mnt

    mkdir /mnt/proc
    mkdir /mnt/sys
    mkdir /mnt/dev

    # mount other filesystem
    mount /dev/nvme1n1p2 /mnt/boot/
    mount /dev/mapper/vgEBS-lvHome /mnt/home/
    mount /dev/mapper/vgEBS-lvTmp /mnt/tmp/
    mount /dev/mapper/vgEBS-lvVar /mnt/var/
    mount /dev/mapper/vgEBS-lvOpt /mnt/opt/
    mount -o bind /proc/ /mnt/proc/
    mount -o bind /sys /mnt/sys/
    mount -o bind /dev/ /mnt/dev/

    # Create directory under /mnt/var
    mkdir /mnt/var/log
    mkdir /mnt/var/tmp
    mkdir /mnt/var/opt

    # mount filesystem under /mnt/var
    mount /dev/mapper/vgEBS-lvLog /mnt/var/log/
    mount /dev/mapper/vgEBS-lvVtmp /mnt/var/tmp/
    mount /dev/mapper/vgEBS-lvVopt /mnt/var/opt/

    # Create directory under /mnt/var/log
    mkdir /mnt/var/log/audit

    # mount the last filelsystem
    mount /dev/mapper/vgEBS-lvAudit /mnt/var/log/audit/
    ```

5. 复制文件到对应的目录

    ```bash
    cp -arf /boot/ /mnt/
    cp -arf /data /mnt/
    cp -arf /etc /mnt/
    cp -arf /home /mnt/
    cp -arf /media /opt /root /srv /tmp /usr /var /mnt/
    cd /mnt
    ln -s usr/bin bin
    ln -s usr/lib lib
    ln -s usr/lib64 lib64
    ln -s usr/sbin sbin
    ```

6. 使用chroot /mnt跳转到新的操作系统

7. 执行grub2-install /dev/nvme1n1 创建GRUB

8. 旧的GRUB菜单内核路径有问题，执行以下命令删除，并添加新的启动项

    ```bash
    grub2-editenv - set "kernelopts=root=/dev/mapper/vgEBS-lvRoot ro console=ttyS0,115200n8 console=tty0 net.ifnames=0 rd.blacklist=nouveau nvme_core.io_timeout=4294967295 crashkernel=auto"
    # verify the grub2 env
    grub2-editenv - list | grep kernelopts

    grubby --remove-kernel=0
    grubby --add-kernel=/boot/vmlinuz-4.18.0-147.el8.x86_64 --title="Red Hat Enterprise Linux (4.18.0-147.el8.x86_64) 8.1 (Ootpa)" --set-default-index=0 --make-default

    # verify grub2 menu
    grubby --info DEFAULT
    ```

    Ref:  
    [How to change the root device](https://www.golinuxcloud.com/update-grub2-grubby-grub2-editenv-rhel-8/)  
    [How to setup the default kernel](https://access.redhat.com/solutions/4326431)

9. 重新生成/boot/initramfs文件，添加LVM相关软件包和驱动程序, 并更新/boot/grub2/grub.cfg

    ```bash
    dracut -f -v /boot/initramfs-4.18.0-147.el8.x86_64.img 4.18.0-147.el8.x86_64
    # verify LVM packages within initramfs
    lsinitrd /boot/initramfs-4.18.0-147.el8.x86_64.img | grep -i lvm
    # update the grub.cfg and ignore the Command failed output.
    grub2-mkconfig -o /boot/grub2/grub.cfg
    ```

10. 修改当前系统中的/etc/fstab启动项为, 并添加swap分区

    ```bash
    [root@ip-172-31-29-83 /]# cat /etc/fstab 
    UUID=231596cf-8237-431a-9382-cf29e54c58d2       /boot   xfs     defaults 0 0
    /dev/mapper/vgEBS-lvRoot        /       xfs     defaults        0 0
    /dev/mapper/vgEBS-lvHome        /home   xfs     defaults        0 0
    /dev/mapper/vgEBS-lvTmp         /tmp    xfs     defaults        0 0
    /dev/mapper/vgEBS-lvVar         /var    xfs     defaults        0 0
    /dev/mapper/vgEBS-lvOpt         /opt    xfs     defaults        0 0
    /dev/mapper/vgEBS-lvLog         /var/log        xfs     defaults        0 0
    /dev/mapper/vgEBS-lvVtmp        /var/tmp        xfs     defaults        0 0
    /dev/mapper/vgEBS-lvVopt        /var/opt        xfs     defaults        0 0
    /dev/mapper/vgEBS-lvAudit       /var/log/audit  xfs     defaults        0 0
    /dev/mapper/vgEBS-lvSwap        swap    swap    defaults 0 0
    ```

11. 重置SELINUX上下文，总共需要重启两次。
    1. 创建.autolabel文件后，即可把当前EBS卷做成AMI,启动实例

        ```bash
        # change Enforcing to permissive in /etc/sysconfig/selinux
        grep "^SELINUX" /etc/sysconfig/selinux 
        SELINUX=permissive

        # create a file under / to reset the selinux contexts.
        touch /.autorelabel
        ```

    2. 系统正常启动后，修改permissive为Enforcing，再创建一次/.autolabel,再重启系统。

        ```bash
        grep "^SELINUX" /etc/sysconfig/selinux 
        SELINUX=enforcing

        touch /.autorelabel
        ```

        Ref:  
        [How to reset SELINUX context](https://access.redhat.com/solutions/24845)  
