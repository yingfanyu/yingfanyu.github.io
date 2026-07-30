# 飞牛 ARM OnlyOffice 在线编辑 — 完整部署指南

## 最终效果
浏览器打开 `https://oo.929295.xyz/` → 输账号密码 → 直接展示 `/vol1/1000/onlyoffice` 目录（科幻风深色 UI）→ 新建/删除文件夹、上传/新建/下载/删除文件 → 点击 xlsx/docx/pptx 在线编辑 → 关闭标签页自动保存回写磁盘。

## 前提条件

| 条件 | 说明 |
|------|------|
| 设备 | ARM64（rk3399/aarch64），飞牛 fnOS |
| Docker | 已安装 `docker` 和 `docker compose` |
| Tailscale | 已安装并获取 IPv4 地址 |
| 外部 OnlyOffice DS | https 可访问，JWT 空（本例：`https://of.929295.xyz`） |
| Cloudflare 域名 | DNS 已代理（橙色云朵） |

## 文件清单

| 文件 | 用途 | 需修改 |
|------|------|:---:|
| `docker-compose.yaml` | 主栈 3 容器 | 是 |
| `nginx.conf` | 反代 + basic-auth | 否 |
| `.htpasswd` | 账号 yingfanyu，apr1 哈希 | **是** |
| `files_app.py` | 文件浏览/编辑器入口/API（端口 8080） | 否 |
| `cloudflared-standalone.yaml` | Cloudflare 隧道（独立 compose） | **是** |

---

## 第一步：获取 Tailscale IP

```bash
tailscale ip -4
# 记下输出（如 100.65.232.26）
```

## 第二步：修改配置

### 2.1 生成密码
```bash
openssl passwd -apr1 "你的飞牛登录密码"
# 输出写入 .htpasswd：yingfanyu:哈希
```

### 2.2 docker-compose.yaml 替换项

| 搜索 | 替换为 |
|------|--------|
| `https://of.929295.xyz` | 你的外部 DS 地址 |
| `https://oo.929295.xyz` | 你的公网域名 |
| `/vol1,/vol2,/vol3` | 实际存储卷 |
| `/vol1/1000/onlyoffice` | 默认打开目录 |

### 2.3 隧道令牌
1. Cloudflare Zero Trust → Networks → Tunnels → Create
2. 选 Docker → 复制 `eyJ...` 令牌 → 填入 `cloudflared-standalone.yaml`
3. 添加 Public Hostname：`oo.929295.xyz` → `http://<Tailscale IP>:10099`

## 第三步：部署

```bash
mkdir -p /opt/onlyoffice-connector
# 把 6 个文件拷贝到 /opt/onlyoffice-connector/

cd /opt/onlyoffice-connector
docker compose up -d                    # 主栈
docker compose -f cloudflared-standalone.yaml up -d   # 隧道

# 验证
docker ps                               # 4 个容器均应 Up
docker logs onlyoffice-cloudflared --tail 10   # 应有 Registered tunnel connection
```

## 第四步：Cloudflare 关键设置

Cloudflare **Auto Minify（JS）**和 **Rocket Loader** 会破坏页面的 JavaScript，**必须关闭**：

1. Cloudflare 仪表盘 → 选域名
2. **速度 → 优化 → Auto Minify** → JavaScript **关**
3. **速度 → 优化 → Rocket Loader** → **关**
4. **缓存 → 配置 → 清除所有内容**

## 第五步：验证

```bash
# 本机测试（绕开隧道）
curl -u yingfanyu:密码 -s "http://localhost:10099/" | grep "page.js"
# 应有输出

curl -u yingfanyu:密码 -s "http://localhost:10099/api/list?path=/vol1/1000/onlyoffice"
# 应返回 JSON 文件列表
```

浏览器无痕窗口 → `https://oo.929295.xyz/` → 输密码 → 直接进入 `/vol1/1000/onlyoffice` 目录。

## 日常更新

```bash
cd /opt/onlyoffice-connector
# 覆盖文件后必须 restart，docker compose up -d 不会重新加载 bind mount
docker restart onlyoffice-files       # 更新 files_app.py
docker restart onlyoffice-nginx       # 更新 nginx.conf 或 .htpasswd
```

## 架构

```
浏览器 ──→ CF Tunnel ──→ fnnas:10099 (nginx) ─┬─→ onlyoffice-files:8080   (文件/API)
                                               ├─→ onlyoffice-connector     (DS 连接器)
                                               └─→ of.929295.xyz           (外部 DS)
```

| 容器 | 镜像 | 说明 |
|------|------|------|
| onlyoffice-nginx | nginx:alpine | 10099→80 反代 + basic-auth |
| onlyoffice-connector | xingheliufang/onlyoffice-fnos:main | Go DS 连接器 |
| onlyoffice-files | python:3-slim | 文件浏览/API/编辑器入口 |
| onlyoffice-cloudflared | cloudflare/cloudflared:latest | 隧道（独立 compose） |

## 安全

- 人类入口（/、/edit、/api/*）→ **basic-auth 鉴权**
- DS 回写路径（/download、/callback、/api/save）→ **不设密码**（DS 无浏览器登录态）
- 前端 apiFetch 带 credentials，遇 401 自动重新登录

## 排障

| 症状 | 检查 |
|------|------|
| 页面「初始化中」/ JS 报错 | F12 → Console → 报错信息。常见：Python 三引号中 `\n` 被转义为换行 → 用 `\\n` |
| 页面打不开 / 522 | `docker ps \| grep cloudflared` + `docker logs onlyoffice-cloudflared --tail 10` |
| 401 反复弹 | `.htpasswd` 哈希是否正确 + `docker restart onlyoffice-nginx` |
| 编辑器 Loading | DS 地址是否正确 + `DOC_SERVER_PATH` 是否填完整 https |
| 更新代码不生效 | bind mount 必须 `docker restart`，不是 `docker compose up -d` |

[飞牛 ARM OnlyOffice 在线编辑 — 完整部署指南.zip](https://github.com/user-attachments/files/30529403/ARM.OnlyOffice.zip)