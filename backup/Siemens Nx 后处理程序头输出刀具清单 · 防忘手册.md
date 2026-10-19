# 程序头输出刀具清单 · 防忘手册

> 适用后处理：`JGJJ_2021.pui / .tcl / .def`（NX Post Builder，FANUC 宏程序风格）
> 记录日期：2026-10-19
> 目标：**一次后处理**，在程序名下一行输出每把刀一行的注释清单（刀号 / 切削时间 / 参考寿命 / 可加工次数 / H、D 补偿宏变量号）

---

## 0. 一句话讲清做法

照常把整份 NC 输出完 → 在程序结束的最后一步把文件**读回来** → 在"程序名"下一行**插入几行注释** → 写临时文件 → 改名覆盖原文件。

不用临时变量攒正文，也不用再遍历一遍程序。

---

## 1. 这东西有什么用

| 用途 | 说明 |
|---|---|
| 程序头一眼看全刀具 | 不用翻到程序尾才知道这台程序要用几把刀、哪几把 |
| 备刀 / 对刀 | 清单里直接给出 H、D 补偿用的宏变量号（如 `#603` / `#604`），现场对刀不用翻后处理 |
| 工时与寿命预警 | `Cut_Time` 是实际切削时间，`QTY` 是"这把刀还能做几件" |
| 机床端可见 | 清单写进 NC 的注释里，操作工在机床上就能看到，不是外部报表 |
| 与程序尾呼应 | 程序尾那段寿命计数用的是同一批数据，头部是"预告"，尾部是"结算" |

实际输出（用一份 5 工序 / 3 把刀的程序跑出来的真实数据）：

```
%
testnewtool
(T1 Cut_Time:4.01min,Life_Time:60.00min,QTY=14,Length_No.=#603,Radius_No.=#604)
(T15 Cut_Time:0.57min,Life_Time:60.00min,QTY=105,Length_No.=#673,Radius_No.=#674)
(T10 Cut_Time:0.23min,Life_Time:60.00min,QTY=260,Length_No.=#648,Radius_No.=#649)
G40 G17 G90 G49 G21
(XCB_COPY)
...
```

其中 `#603 = 600 + 1*5 - 2`、`#604 = 600 + 1*5 - 1`，与程序里 `clear_compensation_variable_H/D` 用的是同一个公式。

---

## 2. 为什么不能"直接在程序开头循环输出"

这是位置上做不到，不是写法问题。

| 做法 | 为什么不行 |
|---|---|
| 在 `Start of Program` 里循环 `toollist` | 那一刻 MOM 刚拿到第一道工序，后面的刀还没派发，`toollist` 里只有第一把刀 |
| 用 `mom_tool_count` / `mom_tool_use(n,0..1)` | 这两个要到程序结束才累加完整；而且它们是喂给 `.lpt` 清单文件的，本 post 的 `mom_sys_list_output = "OFF"`，整条链是死代码 |
| Shop Doc 预扫描（`MOM_do_template_file pb_post_tool_list.tpl`） | 可行（NX 自带现成命令 `pb_cmd_tool_list.tcl`），但会让引擎**把整个程序再走一遍**，大程序后处理时间翻倍 |
| 边输出边把正文攒进变量 | 内核是**流式**写文件的，正文早已落盘；再 `append` 一份，最后写出去就是两份正文 |

所以选"结束后重排"：正文不动，只在头部插几行，代价是**多一次磁盘读写**（几百 KB，毫秒级）。

---

## 3. 实现逻辑

### 3.1 数据从哪来（全是现成的，不用新攒）

| 变量 | 谁写的 | tcl 位置 | 说明 |
|---|---|---|---|
| `toollist` | `PB_CMD_tool_change_tool_data` 换刀时 | 10637~10641 | `lsearch` 去重 + `lappend`，**顺序 = 首次使用顺序**；`PB_CMD_init_flag` 7286 起每次后处理清空 |
| `cuttime($t)` | `PB_CMD_cancel_variable` 每道工序结束 | 5547~5551 | 累加 `mom_toolpath_cutting_time`（实际进给切削时间，不含快移） |
| `toollife($t)` | 同换刀段 | 10643~10648 | 从 `mom_tool_text`（刀具 → 更多 → 机床控制 → 文本）抄来，单位 min，带 3~300 分钟校验 |
| `base_variable_number` | `PB_CMD_init_flag` | 7283 | `600`，宏变量基址 |
| `mom_sys_control_out/in` | `ugpost_base` 初始化 | 178~179 | `(` 和 `)`，注释定界符 |

### 3.2 时序

```
     换刀 T..M06            ← 记 toollist（去重）、记 toollife(t)
        ...
     每道工序结束            ← cuttime(t) += 本工序切削时间
        ...
     程序结束
       ├ end_of_program_5
       ├ PB_CMD_output1_tool_life   ← 程序尾原有那段寿命结算（不改）
       ├ rewind_code                ← M99
       ├ rewind_stop_code           ← 末尾 %
       ├ MOM_set_seq_off
       ├ ★ PB_CMD_prepend_tool_list ← 我们要加的：此时文件已完整
       ├ LIST_FILE_TRAILER          ← 本 post 里是空转
       └ CLOSE_files                ← 只关 list_file / warn_file，不碰 NC
```

### 3.3 插入动作的五步

1. **让内核关句柄** `MOM_close_output_file`（否则读到缺尾内容，Windows 上改名还会报"拒绝访问"）
2. **二进制读全文** 到一个字符串 `body`
3. **按 `toollist` 生成要插入的文本** `ins`（每把刀一行）
4. **数 2 个换行定位**，插在"程序名"那一行的下面
5. **写 `.tmp` → `file rename -force`** 覆盖原文件

---

## 4. 完整代码

### 4.1 在 Post Builder 里新建定制命令

- 名字：`PB_CMD_prepend_tool_list`
- 界面路径：**Post Builder → 定制命令（Custom Command）→ 新建 → 改名 → 粘贴代码**
- ⚠️ 只粘**脚本体**，不要粘 `proc PB_CMD_prepend_tool_list { } {` 和末尾的 `}`（否则 proc 套 proc，第一次调用不生效）

### 4.2 代码（注释已写在内）

```tcl
#=============================================================
# PB_CMD_prepend_tool_list
# 作用：NC 全部写完后，把"每把刀一行"的清单插到程序名下一行
# 挂载：程序和刀轨 → 程序 → 程序结束序列 → {End of Program} 成员表【最后一位】
# 前提：toollist / cuttime() / toollife() / base_variable_number 由换刀段维护
#=============================================================

# 借用外部变量。proc 里不写 global，读到的是不存在的局部变量，会直接报错
global ptp_file_name toollist cuttime toollife base_variable_number
global mom_sys_control_out mom_sys_control_in mom_sys_ptp_output

# ---------- 四道保险：任一不满足就原样退出，一个字节都不动 ----------
if { ![info exists toollist] || [llength $toollist] == 0 } { return }   ;# 没有换刀记录（空程序）
if { ![info exists ptp_file_name] } { return }                          ;# 没有输出文件名
if { ![info exists mom_sys_ptp_output] || $mom_sys_ptp_output != "ON" } { return }  ;# 没生成 NC
if { ![file exists $ptp_file_name] } { return }                         ;# 文件不存在

# ---------- ① 先让内核关句柄 ----------
# 不关：可能读到还没刷盘的最后几行；Windows 上第 ⑤ 步改名会被"文件被占用"拒绝
# 套 catch：万一内核认为文件本来就没打开，这条会抛错，不能让后处理因此中断
catch { MOM_close_output_file $ptp_file_name }

# ---------- ② 二进制读全文 ----------
# 必须 binary：文本模式读时把 \r\n 折成 \n，写回时再补成 \r\n，
# 而我们自己写的 ins 里已经是 \r\n，会被再转一次变成 \r\r\n（多空行）；
# 另外文本模式会按系统编码转码，程序里的 GBK 中文注释会乱码
set fh [open $ptp_file_name r]
fconfigure $fh -translation binary
set body [read $fh]
close $fh

# ---------- ③ 生成插入文本：一行一把刀 ----------
set ins ""
foreach t $toollist {

   # 切削时间：没有记录就是 0
   set ct 0.0
   if { [info exists cuttime($t)] } { set ct $cuttime($t) }

   # 参考寿命：没有记录或不是数字就当 0（防止 expr 报错把后处理打断）
   set lf 0.0
   if { [info exists toollife($t)] && [string is double -strict [string trim $toollife($t)]] } {
      set lf [string trim $toollife($t)]
   }

   # 可加工次数 = 参考寿命 / 单件切削时间，与程序尾 IF[#605 GT n] 的门槛完全同源
   set qty 0
   if { $ct > 0 && $lf > 0 } { set qty [expr {int($lf / $ct)}] }

   # H / D 补偿宏变量号，与 clear_compensation_variable_H/D 同一公式：
   #   H = 600 + t*5 - 2  →  T1 = #603
   #   D = 600 + t*5 - 1  →  T1 = #604
   set h [expr $base_variable_number + $t * 5 - 2]
   set d [expr $base_variable_number + $t * 5 - 1]

   # 拼一整行；控制符取自 mom_sys_control_out/in，以后换注释符这里会自动跟
   # ⚠️ 注释内容里不能再出现 ( ) —— FANUC 会把它当定界符，提前截断或报警
   append ins "${mom_sys_control_out}T$t Cut_Time:[format %.2f $ct]min,Life_Time:[format %.2f $lf]min,QTY=$qty,Length_No.=#$h,Radius_No.=#$d${mom_sys_control_in}\r\n"
}

# ---------- ④ 定位插入点：数 2 个换行 ----------
# 第 1 个换行后是 %
# 第 2 个换行后是 程序名   ← pos 停在这里，新行插在程序名下面
# 想插到 % 之前改成 {1 1}；想插到第一条 G 代码之后改成 {1 3}
set pos 0
foreach _ {1 2} {
   set nl [string first "\n" $body $pos]
   if { $nl < 0 } { set pos [string length $body] ; break }   ;# 行数不足就追加到末尾
   set pos [expr {$nl + 1}]
}

# ---------- ⑤ 写临时文件再整体改名 ----------
# 不能直接 open $ptp_file_name w：那会立刻清空原文件，中途一出错 NC 就没了
set tmp "${ptp_file_name}.tmp"
set fw [open $tmp w]
fconfigure $fw -translation binary                          ;# 与读保持同一模式
puts -nonewline $fw [string range $body 0 [expr {$pos - 1}]] ;# 第一段：文件头 → 程序名行尾
puts -nonewline $fw $ins                                     ;# 第二段：新插入的清单
puts -nonewline $fw [string range $body $pos end]            ;# 第三段：原文件剩下的全部
close $fw
file rename -force $tmp $ptp_file_name                       ;# 原子替换；失败时原 NC 仍完好
```

### 4.3 代码逐段说明

| 段 | 在干什么 | 关键点 |
|---|---|---|
| `global` 那两行 | 把外部变量"借"进来 | `t` / `ins` / `ct` / `lf` / `qty` / `h` / `d` 是本地变量，**不要**写进 global |
| 四道 `return` | 条件不齐就退出 | 空程序、只出清单不出 NC、文件被删，都会走到这里 |
| `MOM_close_output_file` | 关闭内核的写通道 | 本 post 在 `MOM_start_of_group`（tcl 680）已经用过同一句，可用 |
| `open` + `fconfigure binary` + `read` + `close` | 一次性读全文 | `fh` 是**通道号**不是内容；`body` 才是字节串 |
| `foreach t $toollist` | 逐把刀生成一行 | `t` 由 `foreach` 自己创建，不是"没声明" |
| `format %.2f` | 两位小数 | 与程序尾 `%3.2f` 输出结果一致 |
| `foreach _ {1 2}` | 数两个换行 | `_` 表示"不关心这个值"，纯粹当计数器用 |
| `string range` 三段 | 切文件 | `end` 是 Tcl 的特殊下标，表示最后一位 |
| `file rename -force` | 覆盖 | 同一磁盘内是原子操作，不会出现"写了一半的 NC" |

---

## 5. 挂载位置（最容易错的一步）

### 正确位置

**Post Builder → 程序和刀轨 → 程序 → 程序结束序列 → 选中 `{End of Program}` 那一行 → 添加 `PB_CMD_prepend_tool_list` → 拖到最后一位。**

本 post 的成员表（`.pui` 第 1051 行）现在是：

```
{End of Program} { ... end_of_program_5 PB_CMD_output1_tool_life rewind_code rewind_stop_code MOM_set_seq_off }
```

加完应变成：

```
{End of Program} { ... end_of_program_5 PB_CMD_output1_tool_life rewind_code rewind_stop_code MOM_set_seq_off PB_CMD_prepend_tool_list }
```

### 为什么必须是最后一位

PB 生成 `MOM_end_of_program` 时，成员按序展开，**之后**才固定补上 `LIST_FILE_TRAILER` 和 `CLOSE_files`：

```tcl
MOM_do_template rewind_code          ← M99
MOM_do_template rewind_stop_code     ← 末尾 %
MOM_set_seq_off
PB_CMD_prepend_tool_list             ← 我们加在这里（已确认安全）
LIST_FILE_TRAILER
CLOSE_files
```

- 此时 `M99` 和末尾 `%` 已经写完，读到的文件是完整的；
- `LIST_FILE_TRAILER` 在本 post 里空转（`mom_sys_list_output = OFF`，`list_file` 不存在）；
- `CLOSE_files` 只关 `list_file` / `warn_file`，**不碰 NC 通道**；
- 末尾那句 `if [CMD_EXIST PB_CMD_kin_end_of_program]` 因为该 proc 不存在而跳过 ⇒ **关句柄之后没人再写 NC**。

如果挂在中间（比如 `rewind_code` 之前），一关句柄，后面的 `M99`、`%` 就写不进去了——**尾部会丢代码**，而且不报错，最难发现。

---

## 6. 使用注意事项

### 6.1 五条硬性要求（缺一就坏）

| # | 要求 | 不做的后果 |
|---|---|---|
| 1 | 挂在 `{End of Program}` **最后一位** | 尾部 `M99` / `%` 丢失 |
| 2 | 先 `MOM_close_output_file` | 读到缺尾内容；Windows 改名报"拒绝访问" |
| 3 | 读写都 `-translation binary` | 每行后面多一个空行（`\r\r\n`）；中文注释乱码 |
| 4 | `.tmp` + `file rename -force` | 中途出错时 NC 被清空，直接报废 |
| 5 | 注释内容里不能有 `(` `)` | FANUC 提前截断注释，甚至报警 |

### 6.2 状态与缓存

- **`toollist` / `cuttime()` / `toollife()` 是全局变量，NX 同一会话里是"活的"。** 三者都已在本 post 的 `PB_CMD_init_flag` / 换刀段维护（`toollist` 在 7286 起清空）。**如果你自己再加别的数组（比如 `tooldia`），务必在 `PB_CMD_init_flag` 里 `unset -nocomplain`**，否则上一次后处理的刀会残留到这一次。
- **改了 `.tcl` 必须重启 NX 再 post。** 这个后处理有 `ugpost_base is only loaded once` 的保护，同一会话里会继续用内存中的旧脚本，表现为"改了不起作用"。
- **PB 保存后会重新生成 `.tcl`。** 保存后请搜一下 `proc PB_CMD_prepend_tool_list` 是否真的生成了：没挂事件的自定义命令，个别版本 PB 不会生成，此时先临时挂到一个事件上让 PB 生成，再挪走。

### 6.3 数据口径

- `Cut_Time` 是 **`mom_toolpath_cutting_time` 累加值**：只统计进给切削，不含快移、换刀、暂停。和机床实际切削时间接近，但不等于机床计时器。
- `QTY = int(Life_Time / Cut_Time)`，与程序尾 `IF [#605 GT 14] GOTO1` 里的 14 是同一个数。**改了寿命输入规则或切削时间算法，头尾要一起看**，否则头说"能做 14 件"、尾按 15 件报警。
- 程序尾那三行 `#605=#605+1 / IF ... GOTO / #3006=1` 是**运行时**在机床上累加的，**不要搬到程序头**——搬上去会变成"程序一开始就检查并报警"，逻辑反了。

### 6.4 其他

- 清单里**尽量别写中文**。虽然 binary 模式能保证不转码，但部分系统按单字节读注释会错位。
- 大程序（上千工序）只是多读一次几百 KB 的文件，几乎无感；真正慢的是 Shop Doc 预扫描方案。
- 空程序 / 没有换刀的程序：`toollist` 为空，直接 `return`，NC 一个字节不变。
- 若最后一步的 `file rename` 失败（文件被编辑器或杀毒软件占用），后处理会在最后报一个错——**此时 NC 正文已经完整，只是没插上清单**，关掉占用程序重新 post 即可。

---

## 7. 验收清单

post 一个 2~3 道工序、2 把刀的小程序，然后逐条确认：

1. 清单出现在 **程序名下一行**，不是文件最前面、也不是程序尾；
2. 每把刀**一行**，刀号顺序 = 首次使用顺序；
3. 用 Notepad++ 看行尾：全篇 **CRLF**，没有 `\r\r`、没有多余空行；
4. 文件尾部 `M99` 和最后的 `%` **完整**；
5. 清单里的 `Cut_Time` 与程序尾 `(T1 Total_Cutting_Time:...)` **数值一致**；
6. `QTY` 与程序尾 `IF [#60x GT n]` 里的 **n 一致**；
7. 同级目录**没有残留 `.tmp`**；
8. 再 post 一个只有 1 道工序、无换刀的程序，确认不报错、不乱插。

---

## 8. 想改的时候改哪里

| 想改的 | 改哪 |
|---|---|
| 插入位置（`%` 前 / 程序名后 / 第一条 G 代码后） | `foreach _ {1 2}` 里的 `2` → `1` 或 `3` |
| 行内字段 | `append ins` 那一行 |
| 一行总览（`TOOLS: T1 T15 T10`） | 先 `lappend` 到列表，最后 `[join $items { }]` 拼一次 |
| 加刀具直径 / 名称 | 必须在**换刀时**抄进 `tooldia($t)` / `toolname($t)`（程序尾读 `mom_tool_diameter` 只能拿到最后一把刀），并在 `PB_CMD_init_flag` 里 `unset -nocomplain` |
| 按刀号排序 | `foreach t [lsort -integer $toollist]` |
| 加表头 | 在 `set ins ""` 后先 `append ins` 一行 `(...)` |

---

## 9. 附：两条走不通/不推荐的路（避免以后重复踩）

**① `mom_tool_count` / `mom_tool_use(n,0..1)` + `.lpt` 清单**

- 它们是喂给 `.lpt` 报表的，`LIST_FILE_TRAILER`（`ugpost_base.tcl` 2331）在程序结束时写四段：加工总时间、纸带长度、警告数、刀具使用时间；
- 本 post `mom_sys_list_output = "OFF"`（tcl 109），`.lpt` 从来没生成过（桌面全盘搜不到 `.lpt`）；
- 即使打开，`mom_tool_use(n,1)` 在整个 MACH 目录里**没有任何一处赋值**，时间列会是空的。

**② Shop Doc 预扫描**

- `MOM_do_template_file pb_post_tool_list.tpl` + NX 自带的 `pb_cmd_tool_list.tcl`（`PB_CMD_init_tool_list` / `PB_CMD_create_tool_list`），能在程序头拿到全部刀具；
- 代价是引擎**再遍历一遍整个程序**，大程序时间翻倍；本手册的"结束后重排"只需多读一次文件，更快。

---

## 10. 与"新刀标记"那套的关系

这两件事是同一个位置的产物，别搞混：

- **换刀段**（`PB_CMD_tool_change_tool_data`）负责：记 `toollist`、记 `toollife($t)`、决定这一段输出 New 段还是 Old 段；
- **工序结束**（`PB_CMD_cancel_variable`）负责：累加 `cuttime($t)`；
- **程序结束最后一位**（`PB_CMD_prepend_tool_list`）负责：把上面三样汇总成几行注释，插到文件头。

所以清单里出现的每一把刀，都必须在换刀段被登记过——如果某道工序同刀连做、没有触发换刀，它不会重复登记（本来就已登记过），这正是我们要的去重效果。
