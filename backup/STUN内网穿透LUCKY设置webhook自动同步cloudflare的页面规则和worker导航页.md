原始方案的问题：
   1.直接在穿透规则设置webhook的时候，不能设置延时触发。在ip变换或重启后当webhook需要调用其它端口的时候，有可能其它端口还没穿透完毕会导致cloudflare页面规则和worker网页导航同步出错。
   2.在定时任务设置每五分钟执行脚本同步cloudflare页面规则和worker网页时，一开始的时候也有可能会存在同步出错的问题，但是下一个五分钟就能纠正回来。缺陷是一次成功率低，后台频繁运行计划任务。计划任务自定义脚本代码如下：
#!/bin/sh
# 在 Lucky 脚本任务中执行的多规则更新脚本

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/8ff9628ce3e14fd7868d78e4472129eb" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d '{"description":"stun-redirect-dl","expression":"(http.host eq \"dl.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://aria2.1.929295.xyz:{STUN_kodbox_PORT}#?port&pwd=#{STUN_6800port_PORT}&P3TERX&pi_aria2&KH95&{STUN_kh6800_PORT}"},"status_code":302,"preserve_query_string":false}}}'

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/a582c23c822e434e9d859510fcd75400" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d '{"description":"stun-redirect-ai","expression":"(http.host eq \"ai.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://ai.1.929295.xyz:{STUN_kodbox_PORT}"},"status_code":302,"preserve_query_string":false}}}'

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/b10d43aae26c4d2a91225564ae06dbb1" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d '{"description":"stun-redirect-ikuai","expression":"(http.host eq \"ikuai.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://ikuai.1.929295.xyz:{STUN_kodbox_PORT}"},"status_code":302,"preserve_query_string":false}}}'

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/94a37c2ce9344891adda30470b5005ae" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d '{"description":"stun-redirect-nc","expression":"(http.host eq \"nc.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://nc.1.929295.xyz:{STUN_kodbox_PORT}"},"status_code":302,"preserve_query_string":false}}}'

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/06063ea264f7482d9fc6c6634235c547" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d '{"description":"stun-redirect-lucky","expression":"(http.host eq \"lucky.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://lucky.1.929295.xyz:{STUN_kodbox_PORT}"},"status_code":302,"preserve_query_string":false}}}'

curl -X POST "https://dh.929295.xyz/api/update" \
  -H "X-Webhook-Token: yu19761976" \
  -H "Content-Type: application/json" \
  -d '{
  "services": [
    {
      "id": "service_001",
      "url": "https://nc.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_002",
      "url": "https://ikuai.1.929295.xyz:{STUN_kodbox_PORT}/?kh95_3389_:{STUN_3389_PORT}&win11_3389_:{STUN_win113389_PORT}&kh95_22:{STUN_22_PORT}&armbian_22:{STUN_armbian22_PORT}"
    },
    {
      "id": "service_003",
      "url": "https://lucky.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_004",
      "url": "https://aria2.1.929295.xyz:{STUN_kodbox_PORT}#?ARMBIAN_ARIA2_port&pwd={STUN_6800port_PORT}&P3TERX"
    },
    {
      "id": "service_005",
      "url": "https://of.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_006",
      "url": "https://ai.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_007",
      "url": "https://bt.1.929295.xyz:{STUN_kodbox_PORT}/yingfanyu"
    },
    {
      "id": "service_008",
      "url": "https://jxc.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_009",
      "url": "https://{STUN_gaoketls_IP}:{STUN_gaoketls_PORT}"
    },
    {
      "id": "service_010",
      "url": "https://ikuai1.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_012",
      "url": "https://h3c.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_011",
      "url": "http://openwrt.1.929295.xyz:{STUN_kodbox_PORT}"
    },
    {
      "id": "service_013",
      "url": "https://aria2.1.929295.xyz:{STUN_kodbox_PORT}#?KH095_ARIA2_port&pwd={STUN_kh6800_PORT}&P3TERX"
    }
  ]
}

'
新方案：
   1.每个穿透规则写入自定义脚本触发：echo ${port} > /home/kodbox
      把当前穿透规则的外部端口写入到服务器home目录下，名叫kodbox的文件，所有规则以此类推(只支持当前端口，不能跨规则调用，只能出此下策)
   2.任取一个穿透规则写入自定义脚本触发,代码如下：
echo ${port} > /home/armbian22  #把自己的端口写入文件

sleep 30  #暂停30秒，足够所有的端口穿透成功
#!/bin/sh

# 读取端口和IP信息（修正变量名）
win113389_PORT=$(head -n 1 /home/win113389 2>/dev/null | tr -d '\r\n\t ')  #读取所有的端口记录文件到相应变量
armbian22_PORT=$(head -n 1 /home/armbian22 2>/dev/null | tr -d '\r\n\t ')
aria2_PORT=$(head -n 1 /home/aria2 2>/dev/null | tr -d '\r\n\t ')
PORT22=$(head -n 1 /home/22 2>/dev/null | tr -d '\r\n\t ')
kh6800_PORT=$(head -n 1 /home/kh6800 2>/dev/null | tr -d '\r\n\t ')
PORT6800port=$(head -n 1 /home/6800port 2>/dev/null | tr -d '\r\n\t ')
gaoketls_PORT=$(head -n 1 /home/gaoketls 2>/dev/null | tr -d '\r\n\t ')
kodbox_PORT=$(head -n 1 /home/kodbox 2>/dev/null | tr -d '\r\n\t ')
PORT3389=$(head -n 1 /home/3389 2>/dev/null | tr -d '\r\n\t ')
ikuai_PORT=$(head -n 1 /home/ikuai 2>/dev/null | tr -d '\r\n\t ')
gaoketls_IP=$(head -n 1 /home/gaoketls_ip 2>/dev/null | tr -d '\r\n\t ')


# 第一个curl命令，更新dl.929295.xyz的规则
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/8ff9628ce3e14fd7868d78e4472129eb" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"description":"stun-redirect-dl","expression":"(http.host eq \"dl.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://aria2.1.929295.xyz:%s#?port&pwd=%s&P3TERX&pi_aria2&KH95&%s"},"status_code":302,"preserve_query_string":false}}}' "$kodbox_PORT" "$PORT6800port" "$kh6800_PORT")"

# 第二个curl命令，更新ai.929295.xyz的规则
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/a582c23c822e434e9d859510fcd75400" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"description":"stun-redirect-ai","expression":"(http.host eq \"ai.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://ai.1.929295.xyz:%s"},"status_code":302,"preserve_query_string":false}}}' "$kodbox_PORT")"

# 第三个curl命令，更新ikuai.929295.xyz的规则
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/b10d43aae26c4d2a91225564ae06dbb1" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"description":"stun-redirect-ikuai","expression":"(http.host eq \"ikuai.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://ikuai.1.929295.xyz:%s"},"status_code":302,"preserve_query_string":false}}}' "$kodbox_PORT")"

# 第四个curl命令，更新nc.929295.xyz的规则
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/94a37c2ce9344891adda30470b5005ae" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"description":"stun-redirect-nc","expression":"(http.host eq \"nc.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://nc.1.929295.xyz:%s"},"status_code":302,"preserve_query_string":false}}}' "$kodbox_PORT")"

# 第五个curl命令，更新lucky.929295.xyz的规则
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/184bcfddb3651d531e0f04480a1425e3/rulesets/986d4b63ba5f42c38d52ba14cfe55de9/rules/06063ea264f7482d9fc6c6634235c547" \
  -H "Authorization: Bearer qiS5uHl1t-NYLsU1-eYNMJTxrAKzjbftwqOlUUEI" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"description":"stun-redirect-lucky","expression":"(http.host eq \"lucky.929295.xyz\")","action":"redirect","action_parameters":{"from_value":{"target_url":{"value":"https://lucky.1.929295.xyz:%s"},"status_code":302,"preserve_query_string":false}}}' "$kodbox_PORT")"

# 最后一个curl命令，发送更新到dh.929295.xyz
curl -X POST "https://dh.929295.xyz/api/update" \
  -H "X-Webhook-Token: yu19761976" \
  -H "Content-Type: application/json" \
  -d "$(cat <<EOF
{
  "services": [
    {
      "id": "service_001",
      "url": "https://nc.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_002",
      "url": "https://ikuai.1.929295.xyz:${kodbox_PORT}/?kh95_3389_:${PORT3389}&win11_3389_:${win113389_PORT}&kh95_22:${PORT22}&armbian_22:${armbian22_PORT}"
    },
    {
      "id": "service_003",
      "url": "https://lucky.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_004",
      "url": "https://aria2.1.929295.xyz:${kodbox_PORT}#?ARMBIAN_ARIA2_port&pwd=${PORT6800port}&P3TERX"
    },
    {
      "id": "service_005",
      "url": "https://of.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_006",
      "url": "https://ai.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_007",
      "url": "https://bt.1.929295.xyz:${kodbox_PORT}/yingfanyu"
    },
    {
      "id": "service_008",
      "url": "https://jxc.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_009",
      "url": "https://${gaoketls_IP}:${gaoketls_PORT}"
    },
    {
      "id": "service_010",
      "url": "https://ikuai1.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_012",
      "url": "https://h3c.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_011",
      "url": "http://openwrt.1.929295.xyz:${kodbox_PORT}"
    },
    {
      "id": "service_013",
      "url": "https://aria2.1.929295.xyz:${kodbox_PORT}#?KH095_ARIA2_port&pwd=${kh6800_PORT}&P3TERX"
    },
    {
      "id": "service_014",
      "url": "https://web.929295.xyz"
    },
    {
      "id": "service_015",
      "url": "https://tv.929295.xyz"
    }
  ]
}
EOF
)"

   3.当本条穿透成功后，写入自己的端口到文件，然后等待30秒，把所有的端口号都读取到变量，然后执行webhook脚本同步cloudflare的页面规则和worker的导航页。
   4.成功！