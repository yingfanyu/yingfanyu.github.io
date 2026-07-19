1.浏览器下载fnos刷机包 https://github.com/ophub/fnnas/releases下的fnnas_rockchip_smart-am40_k6.12.41_2026.06.25.img.gz文件
2.解压为fnnas_rockchip_smart-am40_k6.12.41_2026.06.25.img
3.使用rufus把镜像写入tf卡
4.插入TF卡，拨码到非normal service，开机
5.浏览器打开192.168.0.15:5666，设置用户名，密码等
6.进入系统设置，SSH，打开SSH功能
7.putty输入192.168.0.15，输入刚设置的用户名，密码，sudo -s 再次输入密码
8.在浏览器里fnos设置存储空间管理，安装应用（lucky，飞牛影音等）
9.在putty按如下提示操作：
停止 eMMC 上的卷：
umount /vol2
vgchange -an
mdadm --stop /dev/md1
复制 TF：
dd if=/dev/mmcblk1 of=/dev/mmcblk0 bs=4M status=progress conv=fsync
写 RK3399 bootloader：
dd if=/usr/lib/u-boot/idbloader.bin of=/dev/mmcblk0 seek=64 conv=fsync

dd if=/usr/lib/u-boot/uboot.img of=/dev/mmcblk0 seek=16384 conv=fsync

dd if=/usr/lib/u-boot/trust.bin of=/dev/mmcblk0 seek=24576 conv=fsync
4.：
sync
poweroff
拔 TF 启动。