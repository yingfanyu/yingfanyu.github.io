远程用户可以通过远程桌面服务 （RDP） 连接到其 Windows 10/11 计算机。在设备设置中启用RDP并使用任何远程桌面客户端连接到计算机就足够了。
但是，同时进行的 RDP 会话的数量存在限制, 只有一个远程用户可以同时工作。如果尝试打开第二个 RDP 会话，将显示一条警告，要求您断开第一个用户的会话。报警如下：
其他用户已登录。如果继续，它们将断开连接。是否仍要登录？
您可以从[GitHub 存储库](https://github.com/binarymaster/rdpwrap/releases)下载 RDP Wrapper（RDP Wrapper的最新可用版本是 v1.6.2）。RDPWrap-v1.6.2.zip 存档包含一些文件：

RDPWinst.exe — RDP Wrapper库安装/卸载程序;
RDPConf.exe — RDP Wrapper配置工具;
RDPCheck.exe — 一个 RDP 检查实用程序（本地 RDP 检查器）;
install.bat,uninstall.bat, update.bat — 用于安装、卸载和更新RDP Wrapper的批处理文件。
若要安装 RDPWrap，请以管理员身份运行 **install.bat** 文件。该程序将安装到 C:\Program Files\RDP Wrapper目录中。

安装完成后，运行RDPConfig.exe。很有可能在安装后，该工具将立即显示RDP Wrapper正在运行 (Installed, Running, Listening)，但不起作用。
请注意红色警告[not supported]。它报告此版本的 Windows 10（版本 10.0.19041.2913）不受 RDPWrapper 支持。

使用 PowerShell cmdlet Invoke-WebRequest 下载文件（必须先停止远程桌面服务）：代码如下
Stop-Service termservice -Force
Invoke-WebRequest https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini -outfile "C:\Program Files\RDP Wrapper\rdpwrap.ini"

重新启动计算机，运行 RDPConfig.exe 工具。检查Diagnostics部分的所有项目是否为绿色，并显示[Fully supported]。

进入gpedit.msc，本地计算机策略，计算机配置，管理模板，windows组件，远程桌面服务，远程桌面主机，连接，限制连接到数量为5，
powershell运行：net user 用户名 密码 /add   添加用户名和密码
                            net localgroup "Remote Desktop Users" 用户名 /add        添加用户组
此后就可以从多个rdp终端连接到同一台win11电脑上，而且互相独立操作。