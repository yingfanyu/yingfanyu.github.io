#为什么选择极路由刷padavan？
##因为极路由原厂rom解决不了安装软件到sd卡后，重启消失的问题。刷openwrt解决不了不识别sd卡的问题。
1.从网址下载cloudflared：https://github.com/lmq8267/cloudflared/releases/download/2024.9.1/cloudflared
2.登录web管理页面，系统，挂载点，添加，启用挂载项，使用uuid挂载注意容量，挂载点root_system，保存并应用
3.使用winscp软件连接到padavan路由器，用户名root，密码***
4.传输下载的cloudflared软件到路由器/overlay目录下
5.putty连接路由器，进入目录/overlay目录下，chmod +x cloudflared授权可执行
6.vi cloudflare.sh ，输入内容/overlay/cloudflared --no-autoupdate tunnel run --token eyJhIjoiN2RhYTFjODZl*************************************************1YzItNGU5NTBmNDMwNGI1IiwicyI6Ill6QmtNRGt3WmpVdFpHSXpZeTAwWTJNMExXRTJPRE10TURObFkyVmlaR0kzTnpZeiJ9    (token后面的时cloudflare对应隧道的token，页面查看)
7.保存退出后chmod +x cloudflare.sh授权可执行
8.crontab -e运行启动项，输入*/1 * * * * /overlay/frpc -c /overlay/frpc.ini          #frpc客户端每分钟运行一次
*/1 * * * * /overlay/cloudflare.sh    #cloudflare客户端每分钟运行一次
9.在cloudflare页面查看是否上线，上线后配置public hostname即可访问对应页面


