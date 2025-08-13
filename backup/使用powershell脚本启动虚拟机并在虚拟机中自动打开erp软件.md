1.安装了erp软件的虚拟机内系统中开启远程桌面功能，并允许远程连接，如果3389被阻拦需要放行
2.在虚拟机内系统添加计划任务，在登录时开启erp软件，输入erp执行文件的目录
3.按 Win+R，输入 gpedit.msc  计算机配置 → 管理模板 → Windows 组件 → Windows PowerShell 启用"允许脚本执行"策略
4.解压文末erp.zip文件，右键选择使用powershell运行即可
5.出现powershell界面，依次静默开启虚拟机，自动远程连接到虚拟机，进入虚拟机并运行erp软件
6.需要关机的时候在虚拟机内部桌面上有关机.bat脚本，双击3秒后关闭虚拟机
7.在远程桌面连接上10秒后自动退出posershell命令行

[启动ERP.zip](https://github.com/user-attachments/files/21752533/ERP.zip)