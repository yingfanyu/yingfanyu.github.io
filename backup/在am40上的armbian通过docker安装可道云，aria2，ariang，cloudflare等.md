1.安装docker，用aliyun的源
2.更新docker更新源sudo nano /etc/docker/daemon.json，可用源：https://docker.1ms.run
https://dockerpull.com
https://hub2.nat.tf
https://docker.m.daocloud.io
https://hub.nat.tf
https://docker.1panelproxy.com
https://hub.rat.dev
https://docker.1panel.live
3.拉取docker并安装
docker pull nextcloud

docker run -i -t -d --restart=always -p 8085:80 -v /mnt/disk:/mnt/disk  -v /home/nextcloud/config:/var/www/html/config -v /home/nextcloud/apps:/var/www/html/apps -v /home/nextcloud/data:/var/www/html/data nextcloud

docker run -i -t -d --restart=always -p 8088:80 -e JWT_ENABLED=true -e JWT_SECRET=yu19761976 onlyoffice/documentserver

[ssh]
type = tcp
local_ip = 192.168.95.102
local_port = 22
remote_port = 5222

docker run -d --restart=always \
  --name aria2-pro \
  -p 6800:6800 \
  -p 6881:6881 \
  -p 6881:6881/udp \
  -v /mnt/disk2:/downloads \
  p3terx/aria2-pro

docker run -d --restart=always \
  --name ariang \
  --log-opt max-size=1m \
  --restart unless-stopped \
  -p 6880:6880 \
  p3terx/ariang

docker run -d --restart=slways cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoiN2RhYTFjODZlZDRiZTkxZmVmZmZjYTlmNGYxYTIzZTQiLCJ0IjoiMzRhYjRlNzYtN2Y4OS00NDY1LWExOTMtODE0ZDhiMTZmY2Y1IiwicyI6IlkyVTVOVFkwTWpjdE9UZG1ZUzAwTjJZNUxXSmlZVEV0T1dZeFlXTXlZekppWVdVeCJ9
