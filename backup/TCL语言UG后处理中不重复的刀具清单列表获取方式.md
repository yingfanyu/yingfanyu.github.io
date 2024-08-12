.在程序头，非中间循环过程建立一个空列表

global toollist

set toollist [list ]

2.在初始化运动中添加以下命令

global toollist mom_tool_number search_variable                                                宣告全局变量

set search_variable [lsearch -exact $toollist $mom_tool_number]                        设search_variable=lsearch搜索列表 -exact精确匹配$toollist列表名 $mom_tool_number当前刀具号  

if {$search_variable == -1} {                                                                                   （用当前刀具号去列表精确查找是否存在当前刀具号，如果存在返回序号，注意第一位是0，不存在返回-1）

lappend toollist $mom_tool_number                                                                    如果search_variable=-1，附加当前刀具号到刀具列表中

}

3.在程序结束处，非中间循环过程添加以下命令

global toollength toollist tooln toolindex

set toollength [llength $toollist ]                                                                            设toollength=llength列表数据位数 $toollist刀具清单，即获取刀具清单中刀具总数量到toollength

set tooln 0                                                                                                               设tooln=0，从清单按顺序获取所有的刀具号的初始值

while {$tooln <= [expr $toollength-1]} {                                                                如果tooln≤总数-1，设toolindex=从列表中按顺序从0到最后一位提取对应的刀具号

set toolindex [lindex $toollist $tooln]                                                                     按tooln的序号从toollist提取需要的刀号，如果想用刀号输出相应的信息，就可以用toolindex代替刀号输出相应信息

set tooln [expr $tooln+1]                                                                                        tooln=tooln+1，循环增量

}