    现状：家中有一条移动宽带，是个大内网，光猫做桥接，ikuai路由器拨号，网络类型设置为NAT1。没有办法获取公网ipv4
地址，虽然可以获取公网ipv6地址，但是在绝大多数的环境下并没有ipv6的上网环境，所以无法直连家中设备。如果需要连家
中设备的时候需要使用frp做中转，frp服务端也是必须有公网ipv4地址才行，所以适应性受限。
   解决方案：1.移动宽带是NAT1（full cone）类型网络，可以使用lucky在公网ip+端口号1的方式来访问映射到内网IP+端口号2的服务，而且网速不受限。缺点是内网ip和端口固定，但是外网的ip和端口经常发生变化，需要及时知道公网ip和端口号。
2.鉴于以上原因采用cloudflare的跳转规则来达到访问固定域名，然后跳转到stun穿透后的公网ip（或者DDNS后域名）和端口号的方式，这样就不需要知道具体的公网ip和端口号。而公网ip和端口号可以通过webhook和cloudflare的api接口实时更新到已经建立的跳转规则。
   具体步骤：
        1.老旧极路由1s刷入openwrt的rom，3.8M英文版本，有足够空间刷入lucky。
        2.stun穿透设置如下：
          ①启用stun穿透模块
          ②开启全局stun穿透模块webhook
          ③接口地址：http://iyuu.cn/IYUU40841Tcbbb500627e9227241c151ee0e4b23b2c483ce10.send?text=映射线路：#{ruleName}&desp=IP端口：#{ipAddr}        .sen前面部分为爱语飞飞的token
          ④webhook手动触发测试，此时微信爱语飞飞账号会推送虚拟的IP和端口号码
        3.添加穿透规则：
          ①规则名称：自定义即可
          ②操作模式：简易
          ③穿透类型：ipv4-tcp
          ④穿透通道本地端口：0默认即可
          ⑤防火墙自动放行：开启
          ⑥目标地址：192.168.0.1   穿透的内部ip地址
          ⑦目标端口：8443   穿透的内部端口号
          ⑧启用全局webhook
          ⑨保存即可，待穿透完成后会发送微信到爱语飞飞公众号
    进阶版：使用stun穿透内部的webhook单个更新cloudflare的重定向api接口下的重定向后的ip地址和端口
      具体步骤：
            1.获取cloudflare的区域id：进入需要跳转的域名，下拉复制获取区域id，记录到文本1
            2.点击左侧“规则”-》“重定向规则”-》创建规则-》部署，最多可以建立10个重定向规则。
            3.创建重定向api口令：我的个人资料-》api令牌-》创建自定义令牌-》令牌名称随意，权限：账户，账户规则集，编辑。区域，单一重定向，编辑。账户资源，包括yi******u@qq.com整个账户。区域资源，全部，继续，创建令牌。复制令牌记录到文本2
            4.获取重定向规则集：lucky计划任务-》备注：随意-》执行周期：仅一次-》执行时间：任意选-》添加子任务-》子任务类型：callweb-》接口地址：https://api.cloudflare.com/client/v4/zones/你的区域ID/rulesets ，注意把文本1的区域id替换到地址中-》请求方法：get-》请求头：Authorization: Bearer 你的令牌
Content-Type: application/json ，注意把文本2记录的令牌替换掉连接中“你的令牌”-》打开字符串检测和输出到日志-》添加保存-》手动触发刚建立的计划任务并打开计划任务的日志-》关键字"http_request_dynamic_redirect"前的id几位规则集id，记录到文本3
            5.获取重定向规则id：修改之前建立的任务，接口地址：https://api.cloudflare.com/client/v4/zones/你的区域ID/rulesets/重定向规则集ID，注意把文本1的内容替换你的区域，文本3的内容替换“重定向规则集ID”-》修改保存-》手动触发刚修改的计划任务并打开计划任务的日志-》关键字"ikuai"（前面手动建立的重定向规则名称）前的id即为规则id，记录到文本4
            6.回到stun穿透规则：修改并启用webhook，注意不是“全局webhook”，是下面的那个
            7.启用地址不同触发webhook
            8.接口地址：https://api.cloudflare.com/client/v4/zones/你的区域ID/rulesets/重定向规则集ID/rules/重定向规则ID 用文本1，文本3，文本4的内容分别替换“你的区域”，“重定向规则集ID”，“重定向规则ID”
            9.请求方法：patch    请求头：跟计划任务的一样  
            10.请求体如下：
{
  "description": "stun-redirect-nas",                   **_“”内的为跳转规则名称_**
 "expression": "(http.host eq \"nas.yusy.us.kg\")",           **_nas.yusy.us.kg为跳转时用的网址_**
  "action": "redirect",
  "action_parameters": {
    "from_value": {
      "status_code": 302,
      "target_url": {
        "expression": "wildcard_replace(http.request.full_uri, \"*://*.yusy.us.kg/*\", \"http://#{ip}:#{port}/${3}\")" 
      },                                                               **_如果用https，改此处，#{ip}表示穿透后的公网ip  #{port}表示穿透后的公网端口_**           
      "preserve_query_string": true
    }
  }
} 
                   注意：复制的时候去除粗斜体注释部分
              11.接口调用成功包含的字符串："success": true，然后保存
              12.如此，当移动宽带重启后或重拨或移动更改ip地址端口后可以自动更新cloudflare的重定向规则，当我们访问在cloudflare托管的访问网址的时候，可以重定向到穿透后的ip+端口的网址。
