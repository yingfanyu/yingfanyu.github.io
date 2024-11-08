1.安装docker，用aliyun的源
2.更新docker更新源sudo nano /etc/docker/daemon.json，可用源：https://docker.1ms.run
https://dockerpull.com
https://hub2.nat.tf
https://docker.m.daocloud.io
https://hub.nat.tf
https://docker.1panelproxy.com
https://hub.rat.dev
https://docker.1panel.live
3.拉取docker pull nextcloud
4.运行nextcloud的docker容器 docker run -i -t -d --restart=always -p 8085:80 -v /mnt/disk:/mnt/disk nextcloud   注意：-v /mnt/disk:/mnt/disk的意思是我外接硬盘挂接到了物理机的/mnt/disk，整个命令就是把物理机的u盘连接链接到docker里面的/mnt/disk目录。
完成