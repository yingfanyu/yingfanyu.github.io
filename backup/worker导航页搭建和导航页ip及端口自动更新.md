  1.cloudflare的存储和数据库新建一个worker kv实例，命名空间名称NAV_STORE
  2.workers和page模块创建应用程序，从hello world开始，部署，编辑代码
  3.清除原有代码，把附件中worker.202512042028.txt的代码粘贴到代码框中，部署
  4.返回到该worker设置页面，顶部有绑定添加KV命名空间，添加绑定输入变量名称NAV_STORE，选择命名空间NAV_STORE
  5.点击设置，添加自定义域，输入test.929295.xyz
  6.添加变量和机密，选择文本，变量名称PASSWORD，值（密码值）
  7.添加触发事件*/30 * * * *
  8.在lucky的stun穿透新建一个穿透规则，名称，ip，端口任意，主要是用它的webhook功能，开启自定义webhook
  9.接口地址：https://test.929295.xyz/api/update
      请求方法：post
      请求头：X-Webhook-Token: yu19761976
                     Content-Type: application/json
      请求体：复制粘贴附件webhook.ok202512041834.txt中的脚本
      重试3次，重试间隔500ms
  10.在lucky计划任务新建一个计划：每5分钟执行一个穿透规则，规则就是webhook所在规则。操作设置为开关切换。这样每5分钟切换一次webhook所在穿透规则就能同步一次，当ip和端口变化最多不超过10分钟就能同步到worker所在的导航页。

[worker主动检查测试宝塔侦测错误解决版202512060834.txt](https://github.com/user-attachments/files/23972901/worker.202512060834.txt)

[webhook测试ok202512041834.txt](https://github.com/user-attachments/files/23949085/webhook.ok202512041834.txt)

[Lucky_backup_1.929295.xyz_2.15.7_20251206083254.zip](https://github.com/user-attachments/files/23972922/Lucky_backup_1.929295.xyz_2.15.7_20251206083254.zip)