一.用gemini写小说：
1.进入宝塔面板的终端页面
2.进入/home/yingfanyu目录后输入source venv/bin/activate开启虚拟环境
3.运行python gw.py（附件）
[gw.py](https://github.com/user-attachments/files/28945175/gw.py)
二.用ollama写小说：
1.进入宝塔面板的终端页面
2.进入/home/yingfanyu目录后输入source venv/bin/activate开启虚拟环境
3.运行python ow.py（附件）
[ow.py](https://github.com/user-attachments/files/28946033/ow.py)
三.tmux后台运行：
1.进入宝塔面板的终端页面
2.进入/home/yingfanyu目录后输入source venv/bin/activate开启虚拟环境
3.tmux new -s xiaoshuo （新建一个对话后台）
   tmux kill-session -t xiaoshuo（强制删除一个后台）
4.运行python ow.py（附件）
5.需要离开ctrl+b然后按D离开，后台继续运行
6.需要重新进入后台：tmux a -t xiaoshuo