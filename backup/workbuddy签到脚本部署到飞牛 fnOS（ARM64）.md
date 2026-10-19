# 部署到飞牛 fnOS（ARM64）

> 结论：**能跑，而且比 Windows 更适合。** NAS 24 小时开机、不需要桌面会话、开机自启，
> 这三点正好补上 Windows 版的短板（关机漏签、要等人登录桌面）。
>
> 唯一必须改的地方是浏览器：Windows 用 Edge，飞牛上没有 Edge，改用系统 Chromium。
> 代码已经改好，`tools/` 一套代码两个平台通用。

---

## 一、可行性说明（为什么能）

| 依赖 | Windows 版 | 飞牛 ARM 版 |
|---|---|---|
| 运行时 | WorkBuddy 内置 Node | 系统 Node 22+ 或 `/opt/workbuddy-node` |
| 浏览器 | Microsoft Edge | Debian 仓库的 `chromium`（ARM64 有官方包） |
| 显示 | 屏幕外窗口 | **Xvfb 虚拟屏**（无显示器也能「有界面」运行） |
| 调度 | 计划任务 | systemd timer |
| 登录态 | `.edge-profile` | 同样是 `.edge-profile`，**但和 Windows 的不通用，要重新扫码** |

代码里已经处理了 ARM/Linux 的坑：

- `--no-sandbox`、`--disable-dev-shm-usage`（NAS 上 `/dev/shm` 很小，不加会崩）
- `--password-store=basic --use-mock-keychain`（无 keyring 环境，不加会导致**每次重启后登录态丢失**）
- 浏览器启动失败自动换 headless 重试
- 失败时清理残留 Chromium 进程，避免越积越多

---

## 二、前提条件

- 飞牛 fnOS（ARM64 / aarch64），系统基于 Debian 12
- Web UI 里已开启 SSH
- 你的飞牛账号（OOBE 创建的那个，如 `admin`）**有 sudo 权限**
- NAS 能访问 `www.workbuddy.cn` 和外网（Server酱、smtp.qq.com）

> fnOS 默认禁止 root 远程登录，这是正常的。全程用普通账号 + `sudo`，不要改成 root 直连。

---

## 三、部署（5 步）

### 1. 把项目传到 NAS

在 Windows 上执行（IP 换成你的飞牛）：

```bash
scp -r "C:/Users/Administrator/WorkBuddy/2026-10-19-22-25-00/workbuddy-checkin" admin@192.168.1.100:~/
```

也可以用飞牛的 SMB 共享，把整个目录拷到 `/vol1/1000/` 下的某个位置。

> 只拷代码就够了。**不要拷 `.edge-profile`** —— Windows 的 cookie 用 DPAPI 加密，
> Linux 解不开，拷过去只会让脚本认为「有登录态」然后一直 401。

### 2. 装依赖 + 注册定时任务

```bash
cd ~/workbuddy-checkin
sudo bash linux/install.sh
```

脚本会：装 `chromium xvfb fonts-wqy-zenhei`，准备 Node 22（没有就下 ARM64 官方包到
`/opt/workbuddy-node`），写入 systemd timer 并启用。

### 3. 填通知配置

```bash
nano config.json
```

必须填的：`sctKey`（Server酱 SendKey）、`email.user` / `email.pass`（QQ 邮箱 SMTP 授权码）。
**用你自己账号的凭据**，不要用原来 Windows 上那份（那份已经泄露过，建议重置）。

### 4. 扫码登录（关键一步）

```bash
bash linux/login.sh
```

脚本会打印：

```
== 需要扫码登录 ==
   截图已保存: /home/admin/workbuddy-checkin/login-qrcode.png
   打开这张图片，用微信扫一扫（图片每 20 秒自动更新）
```

在飞牛「文件管理」里找到这个 PNG 打开，微信扫码。看到 `✔ 登录成功，接口可用` 就好。

> 之后一般一年才需要一次。**不要在这台机器上再手动开浏览器登录**，脚本只认这个 profile。

#### 备选：手机号 + 短信验证码（不用传截图，推荐）

```bash
bash linux/phone-login.sh 17316929295
```

脚本会自动切到「手机号」页签、填号码、勾选协议、点「获取验证码」，然后停下来等你：

```
== 验证码已请求（25 分钟内有效，直接输入验证码后回车即可）==
   也可发送 resend 重新获取，quit 退出
```

手机收到短信后，**直接在终端输入 6 位验证码回车**，脚本自动填入、提交、校验接口，成功即打印 `✔ 登录成功，接口可用`。

- 验证码过期就输入 `resend` 重新获取（按钮有 60 秒倒计时，倒计时内重发无效）
- 后台运行（`setsid nohup`）时改用命令文件投递：`printf 'code:123456' > /tmp/wb-sms-cmd`
- 这条路走的是腾讯 OneID 短信登录，**不限手机号是否绑定过微信**，但登录后仍是同一个账号

### 5. 验证

```bash
bash linux/run.sh --force
systemctl list-timers 'workbuddy*'
```

正常输出应该是 `签到成功 · 连续 N 天` + 积分明细。

**建议再做一次重启验证**（一次性，确认整条链路）：`sudo reboot`，等设备回来后再等 4 分钟，执行：

```bash
journalctl -u workbuddy-checkin.service --since "-10min" | tail -20
node tools/daily-checkin.js --dry-run
```

预期：开机后 3 分钟出现一次 `Finished workbuddy-checkin.service`（当天已签则输出「已成功签到，跳过本次执行」），dry-run 能正常读出连续天数与积分 —— 说明登录态扛住了重启。

> 已在实机（飞牛 ARM64 / Debian 12）验证通过：重启后 07:10:36 自动触发、登录态保留、连续 21 天与积分读取正常、Server酱 + 邮件推送成功。

---

## 四、时间表（装完就有）

| 时间 | 动作 | 说明 |
|---|---|---|
| 开机后 3 分钟 | 跑一次签到 | 当天已签则静默跳过，不打扰 |
| **每天 05:01** | **正常签到** | **主任务** |
| 每天 09:05 | 重试 | 05:01 若因网络/重启失败，这次补上；已成功则静默跳过、不推送 |
| 每天 21:50 | 再跑一次 | 当天没签成才真正执行（等于自动补签） |
| 每天 22:30 | 兜底巡检 | 仍未成功 → 推送告警 |

`Persistent=true`：如果 05:01 时 NAS 正好在重启或断网，恢复后会自动补跑。

---

## 五、「签到失败也推送」是怎么保证的

你要求的是失败也要通知，现在有三层：

| 场景 | 谁负责 | 会不会推送 |
|---|---|---|
| 登录态失效（401） | `daily-checkin.js` | ✅ 立即推送「登录已失效」 |
| 浏览器起不来 / 页面执行异常 | `daily-checkin.js` | ✅ 立即推送失败原因 |
| 接口返回异常 code | `daily-checkin.js` | ✅ 推送 `签到返回 code=xxx` |
| 05:01 失败 | 09:05 重试 / 21:50 补签 | 补签成功不重复推送，失败会推 |
| **任务根本没启动**（Node 丢了、服务没起来） | 22:30 巡检 | ✅ 推送「今日签到未成功」 |
| NAS 整机断电/断网 | 无 | ❌ 发不出去；恢复后 `Persistent` 会补跑 |

最后一行是物理限制：机器不在线，任何本地通知都发不出去。所以「22:30 巡检」是最后一道兜底。

> Server酱免费版每天限 5 条，正常情况下一天最多 1 条成功 + 1 条失败告警，不会超。

---

## 六、排障

| 现象 | 排查 |
|---|---|
| `找不到 Chromium` | `which chromium`；没有就 `sudo apt install -y chromium` |
| `浏览器没有打开页面` | `sudo apt install -y xvfb`；或 `chromium --version` 手动试 |
| 登录页截图是空白 | 可能页面还没渲染，等 20 秒再看（脚本每 20 秒刷新一次图） |
| 一直 401 | 登录态失效，重跑 `bash linux/login.sh` |
| 服务没跑 | `systemctl status workbuddy-checkin.service --no-pager`<br>`journalctl -u workbuddy-checkin.service -n 100 --no-pager` |
| 想看下次触发时间 | `systemctl list-timers 'workbuddy*'` |
| 改签到时间 | `sudo systemctl edit workbuddy-checkin.timer` 覆盖 `OnCalendar` |

ARM 特有：

- **Node 版本必须 ≥ 22**：脚本用了内置 `WebSocket`（Node 21 起稳定），用 Debian 源里的
  Node 18 会直接报 `WebSocket is not defined`。install.sh 会自动处理。
- **Chromium 内存**：单次运行峰值约 400–600 MB，跑完自动退出。NAS 内存 2 GB 以上没问题。
- **不要在 Docker 里再套一层 `--no-sandbox` 之外的限制**，否则 CDP 端口连不上。

---

## 七、和 Windows 版的关系

同一套 `tools/`，靠 `process.platform` 自动选浏览器路径和启动参数。区别只有：

| | Windows | 飞牛 |
|---|---|---|
| 启动方式 | `每日签到.bat` / 计划任务 | `linux/run.sh` / systemd timer |
| 浏览器 | Edge | chromium |
| 登录 | 双击 `登录.bat`，扫码 | `bash linux/login.sh`，看截图扫码 |
| 登录态 | 各自的 `.edge-profile` | 各自的 `.edge-profile`（不通用） |

两边**不要同时跑同一个 profile**（本来也不可能，在不同机器上）。同一个账号在两边各自签到
是幂等的，谁先签都行，另一次会返回「今日已签到」。

---

## 八、卸载

```bash
sudo bash linux/uninstall.sh
```

只删定时任务，项目目录、登录态、配置都保留。
