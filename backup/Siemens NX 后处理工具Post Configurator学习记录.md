1.如何去除fanuc程序开头和结尾的%?
答：打开对应的后处理文件，点击右下角扳手图案，编辑后处理配置器的配置文件，在文件的最后添加
LIB_GE_command_buffer_edit_remove OOTB_program_header OOTB_OEM_program_header @rewind_output
LIB_GE_command_buffer_edit_remove OOTB_program_footer OOTB_OEM_program_footer @rewind_output

2.如何修改输出的程序后缀名？
答：打开后处理，依次常规设置->文件输出处理->输出后缀：改成EIA

3.如何关闭输出序列号？
答：打开后处理，依次控制器功能->输出设置->序列号->选择关闭

4.如何只在换刀的时候输出序列号？
答：打开后处理，依次常规设置->换刀->自动换刀->切换换刀模板为定制过程->输入以下代码:
global mom_tool_number mom_next_tool_number
MOM_output_literal "G0 G28 G91 Z0"
MOM_set_seq_on
MOM_output_literal "T$::mom_tool_number T$::mom_next_tool_number M06"   #增加了马扎克格式的备刀格式
MOM_set_seq_off