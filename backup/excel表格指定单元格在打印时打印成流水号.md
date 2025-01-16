###编辑excel表格内容，设置页面和打印范围。记下需要打印流水号的单元格如G25
##ALT+F11调出vba窗口，左侧双击需要打印的页面，在右侧输入以下内容
Sub dayin()
    n = InputBox("打印次数") * 1
    For i = 1 To n
        ActiveSheet.PrintOut Copies:=1
        [G25] = [G25] + 1 '假设流水号在B2单元格，每次打印后该单元格数字递增1
    Next i
End Sub
##视图，宏，Sheet1.dayin，点击执行并输入打印次数，打印机开始打印。
###以下为输入值纠错优化后的代码，
Sub dayin()
    On Error Resume Next
    n = InputBox("打印次数") * 1
    On Error GoTo 0
    
    ' 检查输入是否有效
    If n <= 0 Then
        MsgBox "请输入有效的打印次数！"
        Exit Sub
    End If
    
    ' 检查 G25 是否为数值
    If Not IsNumeric([G25]) Then [G25] = 0
    
    ' 禁用屏幕更新以提高效率
    Application.ScreenUpdating = False
    
    ' 循环打印
    For i = 1 To n
        ActiveSheet.PrintOut Copies:=1
        [G25] = [G25] + 1
    Next i
    
    ' 恢复屏幕更新
    Application.ScreenUpdating = True
End Sub
