用以下命令即可以控制树莓派的GPIO端口，$14表示GPIO4号端口:

echo $14 > /sys/class/gpio/export                     #暴露GPIO4

echo out > /sys/class/gpio/gpio$14/direction   #输出到GPIO4

echo 1 > /sys/class/gpio/gpio$14/value            #使GPIO4高电平

sleep 5 #延时5秒                                                  #延时5秒

echo 0 > /sys/class/gpio/gpio$14/value             #使GPIO4低电平

echo $14 > /sys/class/gpio/unexport                  #取消暴露GPIO4







以下备注：

$12->GPIO2

$13->GPIO3

$14->GPIO4

$15->GPIO5

$16->GPIO6

$17->GPIO7

$18->GPIO8

$19->GPIO9