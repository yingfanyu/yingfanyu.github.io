[各摆线计算.xlsx](https://github.com/user-attachments/files/19543122/default.xlsx)

##内摆线的定义：当一个圆在另一个圆的内部沿着圆做无滑动的滚动时，内圆周上一个顶点所描绘的轨迹称为摆线。
###为方便理解设起始位置，大圆圆心设为X0Y0,半径rd为60.滚动圆的起始位置圆心坐标(0,-40)半径为rx(20)，当滚动圆转过一个α角度(90°)的时候，Y坐标=rx-COS(α/180pi())\*rx-rd=-40,X坐标=SIN(α/180pi())\*rx\*-1=-20,此时滚动圆转过的弧长arc=rx\*2\*pi()/360\*α=31.41593，滚动圆沿着大圆滚动，滚过的弧长是一样的，可以计算出，滚动圆绕着大圆圆心旋转一个相同弧长的角度β=arc/(2\*pi()*rd)\*360=30,跟踪滚动圆起始切点自转α角度后又旋转(公转)β角度后的新位置即为摆线位置。以此类推不同的角度就能形成摆线轨迹，滚动角度越小精度越高。
###在UG中使用参数方程绘制摆线轨迹，菜单，工具，实用工具，表达式，此处演示还是使用了角度制方式，输入t=1无单位，rd=60长度，rx=20长度，arc=pi()\*2\*rx/360\*t\*1080长度=376.9911，ang=arc/(2\*pi()\*rd)\*360角度=360 ，rxx=sin(t\*1080) \*rx\*-1长度=0 ,rxy=rx-cos(t\*1080) \*rx-rd长度=-60，    xt=rxx\*cos(ang)-rxy\*sin(ang)长度=0，yt=rxx\*sin(ang)+rxy\*cos(ang)长度=-60，zt=0，应用确定后退出。进入菜单，插入，曲线，规律曲线，XYZ规律使用规律曲线，参数输入t，坐标分别输入xt,yt,zt，距离公差根据情况设置，精度越高设置越小。点击确定后在绘图区的XY平面生成一条规律曲线，该曲线即为摆线。

![Image](https://github.com/user-attachments/assets/7fd01cff-c50f-47c9-bba7-a15b3e195116)