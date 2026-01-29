1.从机床导出当前PMC程序：1️⃣拍急停后pmcconf操作停止当前PMC程序2️⃣pmcmainte下IO菜单write到存储卡
2.打开FANUC LADDER III，open，找到复制到电脑上的PMC程序
3.进入LEVE1，ctrl+f查找SUB4(单M代码译码)，往后翻到1011 workpiece blast off这一行
4.鼠标单击下一行代码处，鼠标右键insert，net(forward),此时新增一行。
5.鼠标从左拖到右全选中1011 workpiece blast off这一行，右键复制
6.鼠标单击新增行，右键paste粘贴刚才复制到内容
7.双击1011，修改为1111，前面两位即新增的M代码（M11）,双击粘贴行最后的R501.3为R501.4(修改的R参数必须是未使用过的)
8.重复以上4-6再新粘贴一行。
9.双击新粘贴的1011改为1211，前面两位即新增的M代码（M12）,双击粘贴行最后的R501.3为R501.5(修改的R参数必须是未使用过的)

<img width="731" height="421" alt="Image" src="https://github.com/user-attachments/assets/3b5a899c-cb0b-421e-9e8b-e8a5d77d288b" />

10.重复以上步骤四的动作两次，即新增两行
11.鼠标单击新增第一行最左，插入常开触点一个（双击设名字为R501.4），右侧新增一个常闭触点（双击设名字为R502.4）,右侧再新增一个常闭触点（双击设名字为F1.1）,右侧连续新增导线，最右侧新增一个线圈（双击设名字为Y10.1）即为目标控制点
12.鼠标单击新增第二行最左，插入常开触点一个（双击设名字为Y10.1）自锁，右侧新增一根向上的导线到R501.4的右侧。

<img width="734" height="168" alt="Image" src="https://github.com/user-attachments/assets/2389adfd-f01f-4229-8187-2a36cd2fe80f" />

13.ctrl+f查找G4.3（结束信号），翻页到该功能块最后一行A0.4处，鼠标单击最后一行，右键insert，net(forward),此时新增一行。重复相同动作指导新增两行
14.新增两行最左侧分别插入一个常开触点，（分别双击设名字为R501.4，R501.5）,右侧互联并入到G4.3完成信号的左侧。

<img width="889" height="438" alt="Image" src="https://github.com/user-attachments/assets/d5221138-894c-416f-ba60-ac50beee505d" />

15.在program list表单打开symbol comment，找到机床信号Y10.1，写上字符和注释，找到etc下R501.4和R501.5也分别写上字符和注释，这样子机床上的信号点就有对应的字符了，自定义即可。

<img width="715" height="61" alt="Image" src="https://github.com/user-attachments/assets/78198e75-f850-4b5e-8718-ce9129f025e7" />

<img width="768" height="93" alt="Image" src="https://github.com/user-attachments/assets/c8a80fb1-12d7-4073-8535-8bb9e10717b9" />

16.保存当前PMC程序，导出为记忆卡格式文件，输入名字编译保存到cf卡
17.机床插入CF卡，pmcmainte下IO菜单从存储卡READ到pmc，再写入到flash，pmcconf操作运行当前导入PMC程序，解除急停
18.MDI状态下输入M11时，Y10.1变为1，气动按钮正常熄灭，熄灭后Y10.1仍为1，按复位Y10.1变回0，运行M12也可变为0

PMC程序(删除后缀即可，是记忆卡格式直接导入即可)：![Image](https://github.com/user-attachments/assets/6b91441c-3cdf-4372-8ee8-d70357f6ac9b)