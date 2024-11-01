Win10 LSTC安装OPENSSH服务器后无法用用户名和密码登录的解决办法：
0.搜索服务打开，查找openssh的两个服务，改为自动启动和运行
1.用文本打开‪C:\ProgramData\ssh\sshd_config 
2.找到#PasswordAuthentication,去除前面的#号，如果后面是yes，保持即可，如果是no，改为yes
3.保存
4.powershell管理员运行net stop sshd
   net start sshd 重启openssh服务即可
5.用es文件浏览器访问时，可以在ip地址后直接加盘符，如：10.1.2.43/G:/   这样可以直接进入G盘，否则进入C盘
6.用winscp客户端登陆的时候，直接列出电脑所有盘，分别进入即可