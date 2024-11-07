写外置armbian系统倒tf卡
插入tf卡，开关拨到service模式，重新启动。初始化设置tf卡新系统，进入后。

使用lsblk命令查看emmc分区情况：

mmcblk2   179:0  0 14.6G 0 disk 

mmcblk2boot0 179:32  0  4M 1 disk 

mmcblk2boot1 179:64  0  4M 1 disk 



一般备份主分区mmcblk2就可以了，boot0和boot1是厂商特定的OEM信息，一定要备份到外置存储上，外置存储的空间一定要比eMMC大，下面of路径需要修改为自己的外置存储设备路径。

备份mmcblk2主分区：

dd if=/dev/mmcblk2 of=/mnt/sda4/backup/emmc_backup.img bs=4M status=progress

备份mmcblk2boot0分区：

dd if=/dev/mmcblk2boot0 of=/mnt/sda4/backup/emmc_boot0.img bs=4M status=progress

备份mmcblk2boot1分区：

dd if=/dev/mmcblk2boot1 of=/mnt/sda4/backup/emmc_boot1.img bs=4M status=progress



恢复的话，dd命令的路径反过来就可以了，if=就是外置设备，存放备份文件的路径：

恢复mmcblk2主分区：

dd if=/mnt/sda4/backup/emmc_backup.img of=/dev/mmcblk2 bs=4M status=progress

恢复mmcblk2boot0分区：

dd if=/mnt/sda4/backup/emmc_boot0.img of=/dev/mmcblk2boot0 bs=4M status=progress

恢复mmcblk2boot1分区：

dd if=/mnt/sda4/backup/emmc_boot1.img of=/dev/mmcblk2boot1 bs=4M status=progress 