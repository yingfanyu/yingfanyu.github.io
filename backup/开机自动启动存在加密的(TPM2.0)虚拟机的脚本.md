运行加密虚拟机的脚本
@echo off
chcp 65001 >nul
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"  -vp "yu19761976" start "D:\Virtual Machines\Windows 11 x64\Windows 11 x64.vmx"  nogui
关闭加密虚拟机的脚本
@echo off
chcp 65001 >nul
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"  -vp "yu19761976" stop "D:\Virtual Machines\Windows 11 x64\Windows 11 x64.vmx"  nogui

上述脚本分别存储到一个bat文件，使用计划任务启动事件运行脚本即可完成，定时关机前使用关闭虚拟机脚本关闭后再执行关机脚本
