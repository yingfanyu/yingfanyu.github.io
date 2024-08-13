在制作车铣复合非链接一体后处理时碰到问题：在初始化运动中判断完毕操作属性之后输出不同的操作模式，当基于车削后处理在制作车铣复合后处理时，在输出钻孔循环时（非中心线钻孔）快速移动段落能正常输出，但是把循环分解成线性移动之后不能完整输出的问题，刀路clsf文件中有线性移动语句，但是就算用ug后处理调试工具在跟踪时也会把循环分解的线性运动忽略。

    经研究测试后发现是默认的机床类型导致的，在车削机床类型中它并不能输出钻孔的线性运动。于是需要根据操作的不同设置不同默认机床类型

global mom_kin_machine_type operation_mode  #宣告全局变量，机床类型和操作模式（在初始化运动中判断并宣告赋值）

uplevel #0 {                                                            #进入tcl绝对第0层

if {$operation_mode == "M202"} {                        #如果操作模式时M202,设置机床类型为lathe

set mom_kin_machine_type "lathe"

} else {

set mom_kin_machine_type "3_axis_mill"                #否则设置机床类型为三轴铣削

MOM_reload_kinematics                                         #重新加载所有运动学事件生成器

#MOM_output_literal "[info level]"                          #调试用输出所在层号0                         

}

#MOM_output_literal "$mom_kin_machine_type"     #调试用输出机床类型

}

#MOM_output_literal "[info level]"                          #调试用输出所在层号2    

#MOM_output_literal "$mom_kin_machine_type"    #调试用输出机床类型