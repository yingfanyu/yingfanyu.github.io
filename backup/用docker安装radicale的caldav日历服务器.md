docker pull tomsquest/docker-radicale

docker run -d \
    --name radicale \
    --restart always \
    -p 5232:5232 \
    --init \
    --read-only \
    --security-opt="no-new-privileges:true" \
    --cap-drop ALL \
    --cap-add CHOWN \
    --cap-add SETUID \
    --cap-add SETGID \
    --cap-add KILL \
    --pids-limit 50 \
    --memory 500M \
    --health-cmd="curl --fail http://localhost:5232 || exit 1" \
    --health-interval=30s \
    --health-retries=3 \
    -v /mnt/disk/radicale/data:/data \
    tomsquest/docker-radicale

如果服务器奔溃，重新运行docker命令，路径用/mnt/disk/radicale/data恢复即可。