# 后处理改造 · 修订 15 —— 刀具寿命输出（程序尾自动判断 + 到期报警）防忘文章

| 项 | 内容 |
|---|---|
| 适用后处理 | `mill_intergrex_300_2022`（Mazak Integrex 300，FANUC 30i 底子，NX 2606 / Tcl 8.6.15） |
| 文件 | `mill_intergrex_300_2022.tcl`（明文母本 = `mill_intergrex_300_2022_bf.tcl`，559,160 字节） |
| 本文代码依据 | 从**真实源文件逐行抽取**，行号即明文母本行号 |
| 实测依据 | 用 NX 自带的真 Tcl 8.6.15（`NXBIN\python\tcl86t.dll`）把四个 proc **原文抽出来跑**，见第 6 章 |
| 上游文档 | 修订 2（New/Old 按工序）、修订 3（缺数据即中断）、修订 14（坐标加变量号） |
| 整理日期 | 2026-10-17 |

> **一句话**：后处理在每道工序结束时把该道工序的切削时间记到"刀号账本"上，程序结尾按 `上限 = int(参考寿命 ÷ 单件切削时间)` 给**每一把刀**输出一组 FANUC 宏程序计数段；件数超限就 `#3006=1` 报警停机，叫你换刀片。
> **前提**：每把刀必须在 **NX 刀具 → 更多 → 机床控制 → 文本** 里填一个数字（该刀的参考寿命，单位分钟）。

---

## 目录

- 第 0 章 · 结论速查
- 第 1 章 · 为什么要在后处理里做刀具寿命
- 第 2 章 · 实现方法是什么
- 第 3 章 · 具体代码（5 段，原文）
- 第 4 章 · 主要代码逐行解释
- 第 5 章 · 这个实现的逻辑是什么
- 第 6 章 · 真 Tcl 8.6 实测数据
- 第 7 章 · 关键问题（坑）清单
- 第 8 章 · 维护、排错、验收
- 第 9 章 · 附录：变量 / 常量 / 文件速查

---

# 第 0 章 · 结论速查

**输出的 NC 长这样**（T10 单件切削 20.09 min，参考寿命 60 min）：

```text
(T10 Total_Cutting_Time:20.09min,Reference Life Time:60.00min)
#650=#650+1
IF [#650 GT 2] GOTO100
GOTO101
N100
#3006=1(T10 @0400CAD9C3FCD6D5C1CBA3A1@)
#650=0
N101
```

**这 8 行在干什么**（刀具寿命只在**程序结尾**输出一次，挂在 End of Program 序列里 `G10.9 X1` 之后、`M99` 之前）：

| # | 行 | 作用 |
|---|---|---|
| 1 | `(T10 Total_Cutting_Time:…)` | 注释，给人看：这把刀这一件干了多久、参考寿命多少 |
| 2 | `#650=#650+1` | **加工件数 +1**（`#650` = `600 + 刀号×5`） |
| 3 | `IF [#650 GT 2] GOTO100` | 件数超过上限 → 跳去报警段（上限 = `int(60 ÷ 20.09)` = 2） |
| 4 | `GOTO101` | 没超 → 跳过报警段，正常结束 |
| 5 | `N100` | 报警段标号 |
| 6 | `#3006=1(T10 …)` | **机床停机 + 弹窗**（消息内容是 GB2312 编码的「寿命终了！」） |
| 7 | `#650=0` | **件数清零**（换完刀片从 0 重新数） |
| 8 | `N101` | 正常段标号 |

**五个关键数字**：

| 项 | 值 | 出处 |
|---|---|---|
| 寿命计数器变量号 | `600 + 刀号 × 5`（T10 → `#650`、T11 → `#655`） | 源码 9271 行 |
| 上限 | `int(参考寿命 ÷ 该刀单件有效切削时间)`，**截断取整** | 源码 9267 行 |
| GOTO 标号 | 从 `N100` 起，**每把刀 +2**（`N100/N101`、`N102/N103`…） | 源码 9260、9269–9270 行 |
| 参考寿命来源 | NX 里 **刀具 → 更多 → 机床控制 → 文本** 那一格里的数字 | 源码 6370–6375 行 |
| 切削时间来源 | NX 变量 `mom_cutting_time`（单位**分钟**，NX 自己算好的） | 源码 1358–1362 行 |

**总开关**：程序里**任意一道工序**加了 `set_tool_life` 这个 UDE 且状态 `Active` → 整段逻辑生效；
**一道都没加** → 零输出、不检查、不报错（保持原样）。

---

# 第 1 章 · 为什么要在后处理里做刀具寿命

## 1.1 需求起点：刀片什么时候该换？

现场问题是：**一批零件要连着做几十上百件，刀片不是每一件都换，但也撑不了一整批。** 靠人记"这把刀做过几件了"必然出错，做少了浪费刀片、做多了报废工件甚至撞刀。

要的是：**程序自己知道"这件加工完，这把刀还能不能再干一件"**，到点了就让机床停下来提示换刀。

## 1.2 为什么放在后处理里，而不是机床侧

- **机床侧没有"件数"这个概念**。机床只知道自己走了多少刀路、主轴转了多少圈，不知道"这一件是不是做完了"。
- **NX 已经算好了"每道工序的切削时间"**（`mom_cutting_time`，单位分钟，NX 根据刀路长度和进给算的），后处理直接拿来用，不用自己估。
- **参考寿命是工艺数据**，只可能写在刀具库里。NX 的"刀具 → 文本"格是唯一能跟着刀具一起走、又能被后处理读到的字段。
- 所以正确分工是：**NX 提供两个输入（切削时间、参考寿命）→ 后处理翻译成机床能懂的宏程序 → 机床执行计数与报警。**

## 1.3 为什么用 FANUC 宏变量 `#6xx`

机床端要"记住跨程序、跨开机次数的计数"，只有 **FANUC 的公共变量（`#100`~`#999`）** 有这个性质：断电不丢（`#500`~`#999` 是保持型）。

这台机床上 `#6xx` 区**已经有一套既定分区——每把刀固定占 5 格**：

| 槽位 | 变量（以 T10 为例） | 用途 | 谁在用 |
|---|---|---|---|
| `5N-4` | `#646` | **X 坐标偏移** | 修订 14（坐标加变量号） |
| `5N-3` | `#647` | **Y 坐标偏移** | 修订 14（坐标加变量号） |
| `5N-2` | `#648` | **H 调正量**（`(H TIAO ZHENG BIAN LIANG:#648)`） | 修订 2（New/Old） |
| `5N-1` | `#649` | **D 调正量** | 修订 2（New/Old） |
| **`5N`** | **`#650`** | **寿命计数（加工件数）** | **本篇** |

**为什么本篇用 `5N` 而不是别的号**：这套 5 格分区是既定约定，`5N` 就是留给寿命的格子。新功能只能塞进"已经约好的空位"，否则会和换刀段/调正量互相踩。

## 1.4 为什么"只在程序结尾输出一次"

- 计数是"这件做完了 +1"，**一件只加一次**，所以写在程序尾（`End of Program`），而不是每道工序尾。
- 放在 `M99` **之前**，是因为 `M99` 会跳回程序头（重复执行）。报警必须在 `M99` 前发生，否则程序会先循环回去再把机床停掉，中间多做一件。
- 报警后 `#650=0`，是因为**换刀片是人的动作**：人看到报警 → 换刀片 → 重新跑，程序从 0 重新数。

## 1.5 与"修订 2（New/Old）"的关系

两者是**同一批需求、同一套 `#6xx` 分区**，但触发条件和生命周期完全不同，别混：

| | 修订 2：`set_new_tool` | 本篇：`set_tool_life` |
|---|---|---|
| 干什么 | 新刀装上去时，把该刀的**调正量/各类计数器清零**、并提示"这是新刀" | 程序尾按件数判断**该不该报警换刀** |
| 开关语义 | **按工序一次性**：只有加了事件的**那一道工序**输出 New 段，用完立刻清零 | **全局粘性**：只要有一道工序加了，**整个程序**都输出寿命段 |
| 输出位置 | 换刀段（每道工序） | 程序结尾（一次） |
| 用到的槽位 | `5N-2`、`5N-1`（+ 一批 `#10000+N`~`#18000+N` 清零） | `5N` |

> 这个"生命周期不同"是本篇最容易做错的地方，详见 §7.2。

---

# 第 2 章 · 实现方法是什么

## 2.1 整体手法：三件事

```
① 记账   每道工序结束 -> 把 mom_cutting_time 累加到"刀号账本"，顺手记下刀具"文本"格
② 判据   程序结束    -> 上限 = int(参考寿命 ÷ 单件切削时间)，先预检有没有刀缺数据
③ 输出   程序结束    -> 逐把刀吐一组"计数 / 判断 / 报警 / 清零"的宏程序段
```

**贯穿始终的一个设计**：所有状态都用 **`::` 全限定名的全局变量**（`::nc_cut_time`、`::nc_tool_order`…）。
理由：这些变量要跨"每道工序"活到"程序结束"，而且在 `.tcl` 里跨 proc 共享，用真全局最省心。

## 2.2 数据流（一次程序的完整时序）

```
程序开始
  └─ MOM_start_of_program
       └─ PB_CMD_custom_command_11   ← 一次性初始化
            · unset ::nc_cut_time / ::nc_tool_order / ::nc_ref_text   （清账本）
            · unset ::nc_tool_life_on                                  （清总开关）
            · set ::nc_dbg 0                                           （调试总开关）

第 1 道工序
  ├─ 开始事件：set_tool_life = Active
  │    └─ MOM_set_tool_life -> PB_CMD_set_tool_life_flag
  │         └─ set ::nc_tool_life_on 1        ← 总开关置位（此后不再清）
  ├─ 换刀段：T… T… M06（顺带处理 New/Old，见修订 2）
  ├─ 刀轨 …
  └─ End of Path
       └─ PB_CMD_collect_cutting_time        ← 记账
            · ::nc_cut_time(T10) += mom_cutting_time
            · lappend ::nc_tool_order 10      （第一次出现才入队）
            · ::nc_ref_text(10) ← 刀具"文本"格（取第一次读到的非空值）
            · 顺带清 ::nc_new_tool_request 和 ::nc_addvar_on（修订 2 / 14 的保险）

第 2..N 道工序  同上（T10 累加、T11 入队 …）

程序结束
  └─ MOM_end_of_program
       ├─ MOM_do_template end_of_program_1     (G10.9 X1)
       ├─ PB_CMD_output_tool_life             ← 本篇主角
       │    ├─ 调试段（::nc_dbg=1 时把账本打进 NC）
       │    ├─ 总开关没置位 -> return（零输出）
       │    ├─ 第一遍预检：缺数据/非法 -> 打 ASCII 证据行 + MOM_abort
       │    └─ 第二遍输出：逐把刀吐 8 行寿命段
       └─ MOM_do_template rewind_code          (M99)
```

## 2.3 要动哪些文件（清单）

| # | 文件 | 位置 | 内容 |
|---|---|---|---|
| 1 | `.tcl` 1958–1962 | 事件入口 | `proc MOM_set_tool_life` → 转调开关函数 |
| 2 | `.tcl` 10314–10327 | 开关函数 | `PB_CMD_set_tool_life_flag`：读状态 → 置 `::nc_tool_life_on` |
| 3 | `.tcl` 6333–6385 | 记账 | `PB_CMD_collect_cutting_time` |
| 4 | `.tcl` 9177–9282 | 输出 | `PB_CMD_output_tool_life` |
| 5 | `.tcl` 6879–6890 | 初始化 | `PB_CMD_custom_command_11` 里两小段（清账本 + 调试开关） |
| 6 | `.pui` 790 | Control Functions | `{set_tool_life} {PB_CMD_set_tool_life_flag} {set_tool_life} {UDE}` |
| 7 | `.pui` 740 | Start of Program | 成员表里含 `PB_CMD_custom_command_11` |
| 8 | `.pui` 837 | End of Path | 成员表**第一个**就是 `PB_CMD_collect_cutting_time` |
| 9 | `.pui` 841 | End of Program | `… end_of_program_1 PB_CMD_output_tool_life rewind_code …` |
| 10 | `.cdl` | UDE 定义（后处理自带） | `EVENT set_tool_life { POST_EVENT "set_tool_life" … }` |
| 11 | NX 全局 `ude.cdl` | UDE 定义（**每台机器各一份**） | 同上 |

## 2.4 在 NX / PB 里怎么用

**日常使用（工艺员）**：
1. 刀具库/工序里逐把刀打开 **刀具 → 更多 → 机床控制 → 文本**，**只填数字**（例如 `60`）。
2. 在某一道工序上加 **开始事件 → `set_tool_life` → Status = Active**（一道就够，全程序生效）。
3. 后处理。程序尾自动出现各刀的寿命段；有刀没填数字 → **后处理当场中断**并告诉你缺哪几把。

**改代码（维护者）**：
- 全部改动**只在 Post Builder 界面里做**（定制命令的"步骤"框 = 脚本体），别手改 `.tcl` —— 详见 §8.1。

## 2.5 为什么不用别的做法

| 想法 | 为什么否掉 |
|---|---|
| 在**机床侧**做计数 | 机床不知道"一件做完了"，没有件数信号 |
| 在 `.def` 里加块输出寿命段 | `.def` 是"格式定义"，做不了"累加 / 比较 / 分支出 8 行"这种逻辑；而且块模板加行 PB Save 后会丢 |
| 用 `MOM_output_text` / 注释代替报警 | 只有 `#3006=1(...)` 能让机床**停下来**，注释只是给人看的 |
| 每道工序尾都输出寿命段 | 一件会加好几次 `#650`，计数完全错 |
| 缺数据时"写条警告、继续出程序" | **试过，被现场否掉**：产物是"看起来正常、实则少了一组寿命段"的 NC，到机床或事后才发现，比当场不出程序危险得多 —— 改成一刀切断（详见 §5.3、§7 说明） |
| 用 `#3000` 而不是 `#3006` | `#3000` 通常要求按 RESET 才能恢复；本机用 `#3006`（提示 + 停机）。**换机台前必须先确认该机床的宏报警约定** |

---

# 第 3 章 · 具体代码

> 以下 5 段全部从明文母本 `mill_intergrex_300_2022_bf.tcl` **逐字抽取**，行号即母本行号。
> 已用一个脚本与源文件做过**逐字比对（差异 0）**，见 §8.3。

## 3.1 事件入口（`.tcl` 第 1958–1962 行）

```tcl
#=============================================================
proc MOM_set_tool_life { } {
#=============================================================
   PB_CMD_set_tool_life_flag
}
```

## 3.2 开关函数（`.tcl` 第 10314–10327 行，函数名 `PB_CMD_set_tool_life_flag`）

```tcl
#=============================================================
proc PB_CMD_set_tool_life_flag { } {
#=============================================================
# 需求②：只要程序里【任意一道】工序把 UDE "set_tool_life" 设为 Active，
#         就在程序结尾为程序里用到的【每一把刀】输出刀具寿命判断段。
# 注意：只在 Active 时置 1，不置 0 —— 这是一个"一旦打开就保持"的全局开关。
#       NX 送进来的状态文本是全大写 "ACTIVE"，所以必须用 -nocase 比较。
   global mom_command_status

   if { [info exists mom_command_status] && \
        [string equal -nocase "Active" [string trim $mom_command_status]] } {
      set ::nc_tool_life_on 1
   }
}
```

## 3.3 记账（`.tcl` 第 6333–6385 行，函数名 `PB_CMD_collect_cutting_time`）

```tcl
#=============================================================
proc PB_CMD_collect_cutting_time { } {
#=============================================================
#=============================================================
# 【需求②】每道工序结尾：把该道工序的有效切削时间(mom_cutting_time，单位分钟)
#           累加到对应刀号账上；记录刀号出现顺序；并记下刀具"文本"格的内容。
# 挂在：程序与刀轨 -> 程序 -> 工序结束序列 -> {End of Path}（该行命令列表的第一个）
#=============================================================
   global mom_tool_number mom_cutting_time mom_tool_text

   # 需求①保险：本道工序结束，把"本工序的 set_new_tool 标记"清掉，
   #            防止没走到换刀段的工序把标记漏给下一道工序。
   if { [info exists ::nc_new_tool_request] } { set ::nc_new_tool_request 0 }

   if { ![info exists mom_tool_number] } { return }

   # 刀号归整：>100 视为 刀号*100+补偿号（与 PB_CMD_custom_command_10 的规则一致）
   if { $mom_tool_number > 100 } {
      set t [expr {int(double($mom_tool_number) / 100)}]
   } else {
      set t [expr {int(double($mom_tool_number))}]
   }

   # 切削时间：个别工序可能没有这个变量，用 0 兜底；
   #           但刀号【一定要登记】，否则这把刀会在程序结尾整刀消失。
   if { [info exists mom_cutting_time] } {
      set _cut $mom_cutting_time
   } else {
      set _cut 0.0
   }

   if { ![info exists ::nc_cut_time($t)] } {
      set ::nc_cut_time($t) 0.0
      lappend ::nc_tool_order $t
   }
   set ::nc_cut_time($t) [expr {$::nc_cut_time($t) + $_cut}]

   # 参考寿命：刀具对话框 -> 更多 -> 机床控制 -> 文本 格（同把刀取第一次读到的非空值）
   if { ![info exists ::nc_ref_text($t)] } {
      if { [info exists mom_tool_text] && [string trim $mom_tool_text] != "" } {
         set ::nc_ref_text($t) [string trim $mom_tool_text]
      }
   }

   # ===== 调试：每道工序打一行，确认"刀号 / 时间 / 文本"三个值都读到了 =====
   if { [info exists ::nc_dbg] && $::nc_dbg == 1 } {
      if { [info exists ::nc_ref_text($t)] } { set _r $::nc_ref_text($t) } else { set _r "<none>" }
      MOM_output_literal "(DBGLIFE add tool=$t cut=$_cut ref=$_r)"
   }
# 【需求③】本道工序结束，关闭坐标宏开关，
   #            防止污染后面没加 add_variable 事件的工序。
   if { [info exists ::nc_addvar_on] } { set ::nc_addvar_on 0 }
}
```

## 3.4 程序尾输出（`.tcl` 第 9177–9282 行，函数名 `PB_CMD_output_tool_life`）

```tcl
#=============================================================
proc PB_CMD_output_tool_life { } {
#=============================================================
#=============================================================
# 【需求②】程序结尾输出每一把刀的"刀具寿命判断段"。
# 位置：End of Program 序列里 end_of_program_1(G10.9 X1) 之后、rewind_code(M99) 之前。
#
# 输出样例（T10，有效切削 20.09min，参考寿命 60）：
#   (T10 Total_Cutting_Time:20.09min,Reference Life Time:60.00min)
#   #650=#650+1
#   IF [#650 GT 2] GOTO100
#   GOTO101
#   N100
#   #3006=1(T10 @0400CAD9C3FCD6D5C1CBA3A1@)
#   #650=0
#   N101
#
#   寿命变量编号 = 600 + 刀号*5 ；每把刀一组 GOTO 标号，从 N100 起每刀 +2。
#   上限 = int(参考寿命 / 该件有效切削时间)（截断取整）。
#
# ⚠ 只要程序里有一道工序加了 set_tool_life=Active（结尾就要输出寿命段），
#   则程序里用到的【每一把刀】都必须能在
#   【刀具参数 -> 更多 -> 机床控制 -> 文本】里读到数字。
#   任何一把缺数据/非法：
#      -> NC 里留一行 (TOOL LIFE DATA MISSING : T11 T12 )
#      -> 再中断后处理，并把所有有问题的刀号一次性列出来
#=============================================================

   # ===== 调试：把刀具账本打印进 NC（确认后把 custom_command_11 里的 ::nc_dbg 改成 0）=====
   if { [info exists ::nc_dbg] && $::nc_dbg == 1 } {
      if { [info exists ::nc_tool_life_on] } { set _on $::nc_tool_life_on } else { set _on "unset" }
      MOM_output_literal "(DBGLIFE life_on=$_on)"
      if { [info exists ::nc_tool_order] } {
         MOM_output_literal "(DBGLIFE order=$::nc_tool_order)"
         foreach _t $::nc_tool_order {
            if { [info exists ::nc_cut_time($_t)] } { set _c $::nc_cut_time($_t) } else { set _c "?" }
            if { [info exists ::nc_ref_text($_t)] } { set _r $::nc_ref_text($_t) } else { set _r "<none>" }
            MOM_output_literal "(DBGLIFE tool=$_t cut=$_c ref=$_r)"
         }
      } else {
         MOM_output_literal "(DBGLIFE order=<none>)"
      }
   }

   # 全局开关：程序里没有任何一道工序加过 set_tool_life=Active -> 什么都不输出（也不检查）
   if { ![info exists ::nc_tool_life_on] || $::nc_tool_life_on != 1 } { return }
   if { ![info exists ::nc_tool_order] } { return }

   # ---------- 第一遍：预检，把所有"缺数据/非法数据"的刀一次性收集起来 ----------
   set miss {}      ;# 文本格是空的
   set bad {}       ;# 文本格有内容但抠不出数字，或数字 <= 0
   set ok_list {}   ;# 通过检查的：{刀号 参考寿命 切削时间}

   foreach t $::nc_tool_order {
      if { ![info exists ::nc_cut_time($t)] } { continue }
      set cut [expr {double($::nc_cut_time($t))}]
      # 该刀在本程序里没有真正切削（时间 0）-> 不参与寿命计算（否则会除以 0），也不算"缺数据"
      if { $cut <= 0 } { continue }

      set ref_txt ""
      if { [info exists ::nc_ref_text($t)] } { set ref_txt [string trim $::nc_ref_text($t)] }

      if { $ref_txt == "" } { lappend miss $t ; continue }

      set ref ""
      if { ![regexp {[-+]?[0-9]*\.?[0-9]+} $ref_txt ref] } { lappend bad $t ; continue }
      set ref [expr {double($ref)}]
      if { $ref <= 0 } { lappend bad $t ; continue }

      lappend ok_list [list $t $ref $cut]
   }

   if { [llength $miss] > 0 || [llength $bad] > 0 } {
      set allbad [concat $miss $bad]
      set ascii ""
      foreach t $allbad { append ascii "T$t " }
      # 先在 NC 里留一行纯 ASCII 证据（就算弹窗里中文乱码，也能一眼看出是哪几把刀）
      MOM_output_literal "(TOOL LIFE DATA MISSING : $ascii)"
      MOM_abort "TOOL LIFE DATA MISSING :【刀具寿命数据缺失 = 后处理已中止】在刀具参数->更多->机床控制->文本输入数字（例如 60）"
   }

   # ---------- 第二遍：全部通过，逐把输出寿命判断段 ----------
   set flag_txt "@0400CAD9C3FCD6D5C1CBA3A1@"   ;# 含义：寿命终了！
   set n 100                                   ;# GOTO 标号起点

   foreach item $ok_list {
      set t   [lindex $item 0]
      set ref [lindex $item 1]
      set cut [lindex $item 2]

      set limit [expr {int($ref / $cut)}]
      set n1 $n
      set n2 [expr {$n + 1}]
      incr n 2
      set lifevar [expr {600 + $t * 5}]

      MOM_output_literal "(T$t Total_Cutting_Time:[format "%.2f" $cut]min,Reference Life Time:[format "%.2f" $ref]min)"
      MOM_output_literal "#$lifevar=#$lifevar+1"
      MOM_output_literal "IF \[#$lifevar GT $limit\] GOTO$n1"
      MOM_output_literal "GOTO$n2"
      MOM_output_literal "N$n1"
      MOM_output_literal "#3006=1(T$t $flag_txt)"
      MOM_output_literal "#$lifevar=0"
      MOM_output_literal "N$n2"
   }
}
```

## 3.5 初始化：账本清零 + 调试总开关（`.tcl` 第 6879–6890 行，在 `PB_CMD_custom_command_11` 里）

```tcl
# 调试总开关：1 = 在 NC 里打印 (DBGNEW …) / (DBGLIFE …) 诊断行；全部确认无误后改成 0
set ::nc_dbg 0
#-----------------------------------------------------------

#--------- 需求②：清空切削时间账本 ------------------------
if { [info exists ::nc_cut_time] }  { unset ::nc_cut_time }
if { [info exists ::nc_tool_order] } { unset ::nc_tool_order }
if { [info exists ::nc_ref_text] }  { unset ::nc_ref_text }
# 注意：::nc_tool_life_on 故意不在这里清零，
#       它在 Start of Program 之后由工序里的 set_tool_life 事件置位。
if { [info exists ::nc_tool_life_on] } { unset ::nc_tool_life_on }
#-----------------------------------------------------------
```

## 3.6 对照：另一个开关（修订 2 的 `set_new_tool`），两者的生命周期不同

**入口**（`.tcl` 第 1943–1947 行）：

```tcl
#=============================================================
proc MOM_set_new_tool { } {
#=============================================================
   PB_CMD_set_new_tool_flag
}
```

**开关函数**（`.tcl` 第 10171–10195 行）：

```tcl
#=============================================================
proc PB_CMD_set_new_tool_flag { } {
#=============================================================
#=============================================================
# 【需求①】工序"开始事件"里加了 UDE "set_new_tool" 且状态为 Active 时，
#           只对【本道工序】置一个标志；该标志被换刀段命令用完立刻清零。
#   注意：NX 实际送进来的值是【全大写 ACTIVE】，所以这里做大小写无关判断。
#=============================================================
   global mom_command_status

   if { [info exists mom_command_status] } {
      set st [string tolower [string trim $mom_command_status]]
   } else {
      set st ""
   }
   if { [info exists ::nc_dbg] && $::nc_dbg == 1 } {
      MOM_output_literal "(DBGNEW status=$st)"
   }

   if { [lsearch -exact {inactive off false disable disabled} $st] < 0 } {
      set ::nc_new_tool_request 1
   } else {
      set ::nc_new_tool_request 0
   }
}
```

**消费点（New/Old 判定）**（`.tcl` 第 6800–6812 行，在 `PB_CMD_custom_command_10` 里）：

```tcl
#--------- 【需求①】New / Old 判定 ---------------------------------
#  new_tool : 只代表【本道工序】有没有加 set_new_tool 事件(Active)。
#             用完立刻清零（本命令倒数第 4 行），所以没加事件的工序必然走 Old 段。
#  first_time: 该刀号在本程序里是不是【第一次带着标记出现】。
#             只有真正输出了 New 段才登记，以后再带标记也走 Old 段。
if { ![info exists ::nc_new_tool_request] } { set ::nc_new_tool_request 0 }
set new_tool $::nc_new_tool_request

if { $new_tool == 1 && ![info exists ::nc_new_tool_seen($tooln)] } {
set first_time 1
} else {
set first_time 0
}
```

---

# 第 4 章 · 主要代码逐行解释

## 4.1 `MOM_set_tool_life`（事件入口，3.1）

| 行 | 代码 | 起什么作用 | 有什么用 |
|---|---|---|---|
| 1–3 | `#====` / `proc MOM_set_tool_life { } {` / `#====` | 定义事件处理函数 | 名字必须是 `MOM_` + UDE 名（`set_tool_life`），NX 才会在遇到该事件时调它 |
| 4 | `PB_CMD_set_tool_life_flag` | 一行转调真正的开关函数 | **入口只做转发**：逻辑放在 `PB_CMD_` 里，这样在 PB 界面上能独立编辑、且 PB Save 后不会按 `.pui` 重写 |

> ⚠ `MOM_xxx` 这类事件 proc 是**按 `.pui` 的事件表生成的**，PB 一保存就会被重写；所以**永远不要把逻辑写在这 5 行里**。

## 4.2 `PB_CMD_set_tool_life_flag`（状态 → 总开关，3.2）

| 行 | 代码 | 起什么作用 | 有什么用 |
|---|---|---|---|
| `global mom_command_status` | 声明 | 把 NX 传进来的"事件状态"引到本函数可见 | 这个变量是 UDE 的 `Status` 下拉值，NX 会在事件触发时设好 |
| `if { [info exists …] && \` … | 双条件守卫 | 先确认变量存在，再比较 | 变量不存在时 `string` 会报错；先 `info exists` 兜住 |
| `[string equal -nocase "Active" [string trim $mom_command_status]]` | 大小写无关比较 + 去空白 | 判断"是不是 Active" | 🔴 **NX 送进来的是全大写 `ACTIVE`**，用 `equal`（区分大小写）永远为假 —— 这是本项目最坑的一条，详见 §7.1。`-nocase` 一次堵死 |
| `set ::nc_tool_life_on 1` | 置全局开关 | 打开"整程序要出寿命段" | **只置 1、不置 0**：这是"一旦打开就保持"的粘性开关（对比 3.6 那个每次都被置 0/1 的按工序开关） |

## 4.3 `PB_CMD_collect_cutting_time`（记账，3.3）

| 行 | 代码 | 起什么作用 | 有什么用 |
|---|---|---|---|
| `global mom_tool_number mom_cutting_time mom_tool_text` | 引入三个 NX 变量 | 刀号 / 本道工序切削时间(mim) / 刀具"文本"格 | `mom_tool_text` 就是"参考寿命"的唯一来源 |
| `if { [info exists ::nc_new_tool_request] } { set … 0 }` | 清修订 2 的标志 | 本道工序结束，把按工序标记清掉 | 防止"某道工序没走到换刀段"时标记漏给下一道工序（两条需求共用同一道工序尾，顺手保险） |
| `if { ![info exists mom_tool_number] } { return }` | 没刀号就退出 | 无刀号的工序不进账本 | ⚠ **这是本 proc 里唯一的 early return**；所以后面所有"复位"都必须排在它**之前**（本例清 `::nc_new_tool_request` 就在它前面，是对的） |
| `if { $mom_tool_number > 100 } { set t [expr {int(double(...)/100)}] } else { set t … }` | 刀号归整 | `>100` 视为"刀号×100 + 补偿号"，还原出真刀号 | 与 `PB_CMD_custom_command_10` 的编码规则**必须一致**，否则寿命段的变量号和换刀段的变量号会对不上（详见 §7.3） |
| `if { [info exists mom_cutting_time] } { set _cut … } else { set _cut 0.0 }` | 时间兜底 | 变量不存在时按 0 分钟算 | 🔴 **关键**：绝对不能写成 `if {![info exists …]} { return }` —— 那样这把刀**整刀不进账本**，程序尾就"少了一把刀"，而现象很难往"某个变量不存在"上想（见 §7.4） |
| `if { ![info exists ::nc_cut_time($t)] } { set … 0.0 ; lappend ::nc_tool_order $t }` | 首次登记 | 第一次见这把刀：初始化 + 入队 | `::nc_tool_order` 决定**输出顺序**（= 程序里刀具出现的先后），也决定 GOTO 标号的分配顺序 |
| `set ::nc_cut_time($t) [expr {$::nc_cut_time($t) + $_cut}]` | 累加 | 同一把刀多道工序 → 时间相加 | 这才是"这一件这把刀总共干了多久"，是算上限的分母 |
| `if { ![info exists ::nc_ref_text($t)] } { … }` | 取参考寿命 | 只在**还没记过**时读"文本"格 | **同把刀取第一次读到的非空值**：避免后面工序（刀具参数可能被改）覆盖掉 |
| `MOM_output_literal "(DBGLIFE add …)"` | 诊断行 | 每道工序打一行，显示读到的三个值 | 受 `::nc_dbg` 开关控制；**验"有没有读到数据"只需这一行** |

## 4.4 `PB_CMD_output_tool_life`（输出，3.4）—— 分四段看

### 4.4.1 调试段（9205–9219）

| 行 | 作用 | 用途 |
|---|---|---|
| `if { [info exists ::nc_dbg] && $::nc_dbg == 1 } { … }` | 调试总开关 | 一个变量管全部诊断行；**位置在"总开关 return"之前** ⇒ 即使没加 UDE，也能看到账本 |
| `(DBGLIFE life_on=$_on)` | 打总开关的值 | `unset` = 一道工序都没加 UDE |
| `(DBGLIFE order=$::nc_tool_order)` | 打账本刀序 | 一眼看出"程序里到底记录了哪几把刀" |
| `(DBGLIFE tool=$_t cut=$_c ref=$_r)` | 逐刀打三个值 | `ref=<none>` 就是缺数据的刀 |

### 4.4.2 守卫（9221–9223）

| 行 | 作用 | 用途 |
|---|---|---|
| `if { ![info exists ::nc_tool_life_on] \|\| $::nc_tool_life_on != 1 } { return }` | 总开关没置位 → 直接退出 | **保持原样**：没加 UDE 的程序，行为与本功能上线前完全一致（零输出、零检查、零报错） |
| `if { ![info exists ::nc_tool_order] } { return }` | 没有账本 → 退出 | 防止"程序里一道工序都没走 End of Path"的极端情况报错 |

### 4.4.3 第一遍：预检（9225–9256）

| 行 | 作用 | 用途 |
|---|---|---|
| `set miss {} / set bad {} / set ok_list {}` | 三个收集篮 | 空文本 / 非法数字 / 通过 —— 分开收集才能给出**不同的提示** |
| `foreach t $::nc_tool_order` | 按账本顺序遍历 | 顺序与输出一致，报错顺序也和刀具出现顺序一致 |
| `if { ![info exists ::nc_cut_time($t)] } { continue }` | 没有时间记录就跳过 | 防御性；正常记账流程不会缺 |
| `set cut [expr {double($::nc_cut_time($t))}]` | 取真值 | 后面要做除法，必须确保是数字 |
| `if { $cut <= 0 } { continue }` | **切削时间为 0 的刀直接跳过** | 不参与计算（否则除零），**也不算"缺数据"**（它确实没用上）。例：程序里"只换刀没切削"的刀 |
| `if { $ref_txt == "" } { lappend miss $t ; continue }` | 文本格空 → 记进 `miss` | 这是最常见的缺数据情形 |
| `if { ![regexp {[-+]?[0-9]*\.?[0-9]+} $ref_txt ref] } { lappend bad $t ; continue }` | 正则抠数字 | **宽容设计**：填 `60` 行、填 `60min` 也行（抠出 60）；填 `abc` 才算非法（见 §6 S5） |
| `if { $ref <= 0 } { lappend bad $t ; continue }` | `<=0` 也算非法 | 0 或负数做分母/上限毫无意义 |
| `lappend ok_list [list $t $ref $cut]` | 通过的收进 ok_list | 第二遍只处理通过项 |
| `if { [llength $miss] > 0 \|\| [llength $bad] > 0 } { … }` | **预检闸门** | 有任何一把不合格 → 整段不输出（见下两行） |
| `set allbad [concat $miss $bad]` / `foreach … append ascii "T$t "` | 拼刀号串 | **一次列全**，不让用户"改一把、再报一把"来回折腾 |
| `MOM_output_literal "(TOOL LIFE DATA MISSING : $ascii)"` | 在 NC 里留**纯 ASCII** 证据行 | 弹窗可能中文乱码，但 NC 里这行一定看得懂 |
| `MOM_abort "TOOL LIFE DATA MISSING :【…】在刀具参数->更多->机床控制->文本输入数字（例如 60）"` | **中断后处理** | 消息里**指名道姓告诉用户去哪一格填**；ASCII 打头 + 中文跟后，两头都能读 |

> **为什么 `MOM_abort` 放在 `MOM_output_literal` 之后**：这样"证据行"已经进了 NC 文件。
> 实测确认（§6 S3/S4/S6/S8）：中断后**一行寿命段都不会输出**，NC 里只留那一行 `(TOOL LIFE DATA MISSING : …)`。

### 4.4.4 第二遍：输出（9258–9281）

| 行 | 作用 | 用途 |
|---|---|---|
| `set flag_txt "@0400CAD9C3FCD6D5C1CBA3A1@"` | 报警里的中文密文 | `@` + GB2312 十六进制 + `@`；解出来是「**寿命终了！**」。这样机床弹窗显示中文而不是乱码（直接写中文会被机床按单字节拆坏） |
| `set n 100` | GOTO 标号起点 | `N100` 起，不和程序里其它标号冲突的概率最高 |
| `set limit [expr {int($ref / $cut)}]` | **上限** | `int()` = 截断取整。60 ÷ 20.09 = 2.98 → **2**（第 3 件时 `#650=3 > 2` → 报警） |
| `set n1 $n` / `set n2 [expr {$n + 1}]` / `incr n 2` | 本刀的两个标号 + 指针前移 2 | 每把刀占 2 个标号，互不重叠 |
| `set lifevar [expr {600 + $t * 5}]` | 寿命计数器变量号 | 与 §1.3 的 5 格分区一致（T10 → `#650`） |
| 第 1 行 `MOM_output_literal "(T$t Total_Cutting_Time:…)"` | 注释行 | `[format "%.2f"]` 固定两位小数，便于人眼比对；**只给人看，机床不执行** |
| 第 2 行 `"#$lifevar=#$lifevar+1"` | 件数 +1 | 累计加工件数（公共变量断电保持） |
| 第 3 行 `"IF \[#$lifevar GT $limit\] GOTO$n1"` | 比较 + 跳转 | `\[` `\]` 是为了让 `[` 原样输出（Tcl 的 `[` 是命令替换符，必须转义） |
| 第 4 行 `"GOTO$n2"` | 未超限 → 跳到正常段 | 结构上是 `if…else` 的经典 GOTO 写法 |
| 第 5 行 `"N$n1"` | 报警段标号 | 跳转目标 |
| 第 6 行 `"#3006=1(T$t $flag_txt)"` | **机床停机 + 弹窗** | 这是唯一能让机床"停下来"的语句；括号里是消息内容 |
| 第 7 行 `"#$lifevar=0"` | 件数清零 | 换完刀片重新计数 |
| 第 8 行 `"N$n2"` | 正常段标号 | 未超限时从这里继续（也就是直接往下走） |

## 4.5 初始化（3.5）

| 行 | 作用 | 用途 |
|---|---|---|
| `set ::nc_dbg 0` | 调试总开关默认关 | 平时 NC 干净；要查问题改成 1（**改一处管全部诊断行**） |
| `unset ::nc_cut_time / ::nc_tool_order / ::nc_ref_text` | 清三个账本 | 在 `Start of Program` 跑，保证一个程序一份干净账本 |
| `unset ::nc_tool_life_on` | 清总开关 | **必须在这里清**：否则跑第二个程序时不加 UDE 也会出寿命段。清完之后由工序里的事件重新置位 |

> 源码里紧跟着的一句注释"注意：`::nc_tool_life_on` 故意不在这里清零"**和代码相反 —— 代码是对的、注释是错的**（这个命令只在 Start of Program 跑一次，就该在这里清零）。别被注释带偏，建议直接删掉那句注释。

---

# 第 5 章 · 这个实现的逻辑是什么

## 5.1 状态机：2 个开关 + 3 本账

| 名称 | 类型 | 生命期 | 谁写 | 谁读 |
|---|---|---|---|---|
| `::nc_tool_life_on` | 开关 | **程序级粘性**（置 1 后不再清，直到下个程序开始） | `PB_CMD_set_tool_life_flag` | `PB_CMD_output_tool_life` |
| `::nc_dbg` | 开关 | 程序级（Start of Program 置值） | `PB_CMD_custom_command_11` | 所有诊断行 |
| `::nc_tool_order` | 账本 | 程序级 | `PB_CMD_collect_cutting_time`（首次出现时 `lappend`） | 输出顺序、标号分配 |
| `::nc_cut_time(t)` | 账本 | 程序级 | 记账（累加） | 算上限的分母 |
| `::nc_ref_text(t)` | 账本 | 程序级 | 记账（只记第一次非空） | 算上限的分子 |

**为什么"开关"和"账本"要分开**：开关回答"要不要出寿命段"，账本回答"给哪些刀、出什么数"。这样"没加 UDE 但程序里跑了 5 把刀"不会误出（开关守卫在前），"加了 UDE 但没切削"也不会报假错（`cut<=0` 跳过）。

## 5.2 为什么是"两遍式"

一遍扫描同时做"检查"和"输出"，就会出现**出一半才发现有问题**的结果：NC 里前 2 把刀有寿命段、第 3 把缺数据中断 —— 用户拿到一个半成品程序，还得手工删。

**两遍式**把"能不能做"和"怎么做"彻底分开：

```
第一遍：只检查，往 miss / bad / ok_list 三个篮子里装东西，一个字节都不输出
        有任何 miss/bad -> 打一行 ASCII 证据 + MOM_abort（此时 ok_list 被丢弃）
第二遍：只输出，遍历 ok_list
```

⇒ 结果只有两种：**要么一行不出（+ 明确的错误提示），要么全出（且一定完整）**。

## 5.3 为什么缺数据必须"中断"而不是"跳过"

这条是**用户明确要求的**，也是被现场事故推出来的：

- "跳过 + 写警告继续出程序"的产物，是一个**看起来完全正常、实则少了一组寿命段**的 NC。
- 这种程序到机床上不会有任何异常提示，只是**那把刀的寿命永远不报警** —— 等发现时刀片可能已经崩了、工件已经废了。
- 而"当场不出程序"的代价只是工艺员去填一个数字。
- ⇒ **默认选中断**。只有当"技术上无法参与计算"（如切削时间恰好为 0，参与就会除零）时才允许静默跳过，并且要有注释写明理由 —— 本例 `if { $cut <= 0 } { continue }` 正是这种。

## 5.4 上限算法：为什么是 `int(参考寿命 ÷ 单件切削时间)`

设参考寿命 `ref`（分钟），本件这把刀实际切削 `cut`（分钟）：

- 该刀还能加工多少件 ≈ `ref / cut`。
- 取**截断整数** `int()`：60/20.09 = 2.98 → 2。意思是"第 3 件做完时已经超过 60 分钟了，所以在第 3 件开始前就该报警"。
- 计数器在**程序尾 +1**，所以判断的是"**已经做完的件数**是否超过上限"：第 1 件尾 `#650=1`（1>2 假）、第 2 件尾 `#650=2`（2>2 假）、第 3 件尾 `#650=3`（3>2 真 → 报警）。
- 边界：`ref < cut` 时上限 = 0 ⇒ **每件都报警**（第一件尾 `#650=1 > 0`）。这是合理的：一件的切削量就超过了刀片寿命。
- ⚠ `cut` 是**本程序里这把刀的总切削时间**（多道工序累加），不是单道工序 —— 所以"单件"= 这一个 NC 程序对应的那一个工件。

## 5.5 变量号与标号算法

```
寿命计数器     lifevar = 600 + 刀号 × 5      （T10 → #650，T11 → #655）
报警段标号     n1/n2   = 100 + 2×序号        （第 1 把 → N100/N101，第 2 把 → N102/N103）
```

- 为什么 `×5`：`#6xx` 是"每把刀 5 格"的既定分区（§1.3）。
- 为什么标号 `+2`：每把刀需要"报警段入口 + 正常段入口"两个标号。
- 为什么从 `100` 起：NX 生成的程序里 `N` 号通常从 `1` 开始连续编号，从 `100` 起能最大限度避开。**但这不是绝对保证** —— 若你的程序里本来就有 `N100`，会冲突。改 `set n 100` 那一行即可（例如改成 `900`）。

## 5.6 `#3006=1(…)` 与那串 `@…@` 密文

- FANUC 的宏报警/宏消息入口是 `#3000`~`#3006` 系列。本机后处理选的是 **`#3006`**：在那一程序段结束时停机并显示消息。
- **具体表现（是否停主轴、是否要求按 RESET、是否只 feed hold）取决于机床参数**，不同机台不一样 —— **首次上机务必空跑一件确认**。
- 消息里的 `@0400CAD9C3FCD6D5C1CBA3A1@`：`0400` 是前缀，其后 `CAD9 C3FC D6D5 C1CB A3A1` 是 **GB2312 编码**的「寿命终了！」。用 `@` 包起来 + 十六进制，是为了**把中文安全地送进 FANUC 的消息字段**（直接写中文会被按单字节拆坏、显示成乱码）。
- 想改提示文字：用任意 GB2312 编码器把新文字转成十六进制，替换 `flag_txt` 那一行即可。**别改成非中文以外的东西**（比如加英文），消息字段长度有限。

## 5.7 单位与精度

| 量 | 单位 | 精度处理 |
|---|---|---|
| `mom_cutting_time` | **分钟**（NX 原始单位） | 不做转换 |
| 输出注释里的时间/寿命 | 分钟 | `[format "%.2f"]` → 固定两位小数 |
| 上限 `limit` | 件（整数） | `int()` 截断，**不四舍五入** |

> 若以后要做"秒"或"小时"，**只改显示那一行**（`Total_Cutting_Time:` 那行是纯注释），**不要动 `limit` 的算法** —— 它跟单位无关（分子分母同单位即可）。

---

# 第 6 章 · 真 Tcl 8.6 实测数据

- 解释器：`C:\Program Files\Siemens\Designcenter2606\NXBIN\python\tcl86t.dll`（**Tcl 8.6.15**，NX 自带）
- 方法：用 ctypes 起解释器 → 按大括号配对**从真实 `.tcl` 里抠出 4 个 proc 的原文** → 桩掉 `MOM_output_literal` / `MOM_abort` / `MOM_reload_variable` → 逐场景跑
- 脚本：`Test-ToolLife.py`（桌面）/ `tclenc_verify\toollife_test.py`（工作区）；完整报告 `toollife_test_report.txt`
- ⚠ 桩 `MOM_abort` **必须抛错**才能复刻真实中断行为（否则第二遍会继续跑，测出来的是假的）

## 6.1 事件状态 → 开关（8 种取值）

| NX 送进来的值 | `::nc_tool_life_on` | `::nc_new_tool_request` | 说明 |
|---|---|---|---|
| `ACTIVE` | 1 | 1 | **全大写 = NX 真实送进来的值** |
| `Active` | 1 | 1 | 大小写变体 |
| `active` | 1 | 1 | 小写 |
| `Inactive` | *unset* | 0 | 不生效 |
| `INACTIVE` | *unset* | 0 | 不生效 |
| `Off` | *unset* | 0 | 不生效 |
| （变量不存在） | *unset* | **1** | ⚠ 见 §7.8 |
| `USER DEFINED` | *unset* | **1** | ⚠ 见 §7.8 |

> 两个开关实现方式不同：`set_tool_life` 用**白名单 + `-nocase`**（只有 Active 才算）；
> `set_new_tool` 用**黑名单**（不在 `{inactive off false disable disabled}` 里就算）。
> 结果差异就在最后两行：黑名单写法把"空值 / 其它值"一律当成 Active（**fail-open**）。

## 6.2 记账：4 道工序喂进去

| 喂进去的工序 | 累计切削(min) | 参考寿命文本 | 账本刀序 |
|---|---|---|---|
| T10 第 1 道工序 | 1.20 | `60` | `10` |
| T10 第 2 道工序（应累加） | 0.38 | `60` | `10` |
| T11 文本格为空 | 2.50 | *（空）* | `10 11` |
| 刀号 1005 → 应归整成 T10 并继续累加 | 0.75 | `45` | `10 11` |

记完账后：**T10 → cut=2.33、ref=60**（1.20 + 0.38 + 0.75，第三条 1005 归到 T10）；**T11 → cut=2.5、ref=无** ✅ 完全符合预期。

## 6.3 刀号编码边界

| 喂进 `mom_tool_number` | 账本记成 | 说明 |
|---|---|---|
| 99 | T99 | 直接取整 |
| 100 | T100 | **`>100` 才算组合码，所以 100 原样保留** |
| 101 | T1 | 1×100 + 01 |
| 1005 | T10 | 10×100 + 05 |

⚠ **`mom_cutting_time` 不存在时**：账本记 `order = 12`、`cut_time(12) = 0.0` ✅（**刀号登记住了**，不会整刀消失）。

## 6.4 程序尾八种情形

| 场景 | 输出行数 | 是否中断 | 结果 |
|---|---|---|---|
| S1 没加过 `set_tool_life` | **0** | 否 | 零输出、不检查、不报错 ✅ |
| S2 正常：T10 / T11 两把刀 | 16 | 否 | 各出一组 8 行 ✅ |
| S3 缺数据：T11 文本格为空 | **1** | **是** | 只留 `(TOOL LIFE DATA MISSING : T11 )`，**一行寿命段都不出** ✅ |
| S4 非数字：文本 = `abc` | 1 | 是 | 同上 ✅ |
| S5 文本 = `60min` | 8 | 否 | 正则抠出 60 → **正常通过**（宽容，非报错）✅ |
| S6 参考寿命 = `0` | 1 | 是 | `<=0` 按非法 ✅ |
| S7 该刀切削时间为 0 | 8 | 否 | T10 静默跳过（不算缺数据），只输出 T11 ✅ |
| S8 T11 + T13 都缺 | 1 | 是 | `(TOOL LIFE DATA MISSING : T11 T13 )` — **一次报全** ✅ |

## 6.5 输出原文逐字对照（S2）

16 行与期望值**逐行相同，不一致项 = 0**：

```text
(T10 Total_Cutting_Time:20.09min,Reference Life Time:60.00min)
#650=#650+1
IF [#650 GT 2] GOTO100
GOTO101
N100
#3006=1(T10 @0400CAD9C3FCD6D5C1CBA3A1@)
#650=0
N101
(T11 Total_Cutting_Time:1.58min,Reference Life Time:60.00min)
#655=#655+1
IF [#655 GT 37] GOTO102
GOTO103
N102
#3006=1(T11 @0400CAD9C3FCD6D5C1CBA3A1@)
#655=0
N103
```

（T10：`600+10×5 = #650`、`int(60/20.09) = 2`；T11：`#655`、`int(60/1.58) = 37`；标号 `N100/N101` → `N102/N103` ✅）

## 6.6 上限算法

| 参考寿命(min) | 单件切削(min) | 上限 | 说明 |
|---|---|---|---|
| 60 | 20.09 | **2** | 60/20.09 = 2.98 → 2 |
| 60 | 1.58 | **37** | 60/1.58 = 37.9 → 37 |
| 60 | 0.5 | **120** | 整除也算 |
| 60 | 61.0 | **0** | 一件就超寿命 → **每件都报警** |
| 45 | 0.75 | **60** | |

---

# 第 7 章 · 关键问题（坑）清单

## 7.1 🔴 UDE 状态是**全大写 `ACTIVE`**

最坑的一条。`PARAM command_status` 的选项在 `.cdl` 里写的是 `"Active"`/`"Inactive"`，但**下发到后处理时被大写成 `ACTIVE`**。

```tcl
# ❌ 永远为假
if { ![string compare "Active" $mom_command_status] } { … }
# ✅
if { [string equal -nocase "Active" [string trim $mom_command_status]] } { … }
```

症状极具欺骗性：**不报错、不中断**，只是条件永远为假 —— 于是"事件触发了等于没触发"。排查手法：在该 UDE 的处理命令里临时插一行 `MOM_output_literal "(DBG status=$mom_command_status)"`，出一次程序看 NC 里到底是什么值。

## 7.2 🔴 两个开关的**生命周期完全不同**

| | `set_tool_life`（本篇） | `set_new_tool`（修订 2） |
|---|---|---|
| 语义 | **全局粘性**：任意一道工序 Active → 整程序生效 | **按工序一次性**：只有加了事件的那道工序 |
| 谁清零 | `Start of Program` 清一次 | 消费它的命令（`custom_command_10`）用完立刻清 + `End of Path` 兜底再清 |
| 实现 | 只 `set 1`，不 `set 0` | 每次都 `set 1` 或 `set 0` |

**动手前必须和用户把口径敲死**。本项目在这上面返工过一次：第一版把需求①（New/Old）也做成了"全局粘性"，结果**每把刀第一次出现都判成 New**，而用户要的是"只有加了标记的那道工序才 New"。

## 7.3 刀号 `>100` 的编码约定（三处必须一致）

| 地方 | 代码 |
|---|---|
| 记账（6333 行区） | `if { $mom_tool_number > 100 } { set t [expr {int(double($mom_tool_number)/100)}] }` |
| 换刀段 `custom_command_10` | `if {$mom_tool_number > 100 } { set tooln [expr $mom_tool_number/100] }` |
| 坐标宏（修订 14） | 刀号 >100 先 `int(tn/100.0+0.5)` 还原 |

⇒ **改任何一处都要同步改另外两处**，否则"寿命段的变量号"和"换刀段的变量号"会指向不同的刀 → 计数记到别人头上去。

⚠ 边界（实测 §6.3）：`>100` 才当组合码 ⇒ **刀号正好 100 会被当成"真 T100"**；而 101 会被解释成 T1。若你的刀库里有 100~999 的"纯刀号"，这个约定就歧义了，必须先统一。

## 7.4 🔴 可选变量缺失时**绝不能 `return`**

```tcl
# ❌ 一旦某道工序没有 mom_cutting_time，这把刀就整刀不进账本
if { ![info exists mom_cutting_time] } { return }
# ✅ 按 0 记账，但刀号一定要登记
if { [info exists mom_cutting_time] } { set _cut $mom_cutting_time } else { set _cut 0.0 }
```

漏登记的后果是**程序尾"少了一把刀"**，而现象很难往"某个变量不存在"上想。诊断行里会显示 `cut=0.0`，能抓出来。

## 7.5 诊断行必须在**所有 early return 之前**

`PB_CMD_output_tool_life` 的第一句实际代码是总开关守卫（9222 行）。若把调试段放在它**后面**，那么"没加 UDE"时就什么都看不到 —— 而"没加 UDE"恰恰是最需要确认的情形（"我明明加了，怎么没输出？"）。

⇒ 本项目把调试段放在 9205–9219，**位于守卫之前**。同理 `PB_CMD_collect_cutting_time` 的诊断行放在末尾（那里没有 early return 冲突，但 `return` 在 6347 行 —— **清 `::nc_new_tool_request` 那行必须排在 6347 之前**，源码里是对的）。

## 7.6 `::nc_tool_life_on` 的清零时机

- **清在哪**：`PB_CMD_custom_command_11`（挂 `Start of Program`，整程序只跑一次）。
- **不在哪清**：不在 `End of Program`、不在 `End of Path`。理由：它是"整程序"级的开关，End of Path 清掉就会导致后面的刀都不出寿命段。
- 源码里那句注释（"故意不在这里清零"）是**写错的**，代码是对的。

## 7.7 UDE 定义有**两份**，其中一份不随后处理文件走

| 位置 | 作用 | 会不会跟着拷贝 |
|---|---|---|
| `mill_intergrex_300_2022.cdl`（后处理自带） | 后处理侧的 EVENT 映射 | ✅ 会 |
| `<NX>\MACH\resource\user_def_event\ude.cdl`（**NX 全局**） | 事件定义库，NX 据此把 UDE 送进后处理 | ❌ **不会**，每台机器的 NX 各一份 |

⇒ **换电脑 / 重装 NX 后，必须在新机器上补上全局 `ude.cdl` 里的 `EVENT set_tool_life` 块，然后重启 NX。** 少一份的后果是**静默丢弃**（不报错、不输出），最容易被误判成"代码写错了"。

> 顺带一处**不一致**（不影响功能，但要知道）：后处理自带 `.cdl` 的 `set_tool_life` 有 **3 个选项**（`Active/Inactive/User Defined`），NX 全局 `ude.cdl` 里只有 **2 个**（`Active/Inactive`）。选 `User Defined` 在后处理侧**不会**打开寿命开关（实测 §6.1）。

## 7.8 已知边界（实测确认）

| 边界 | 现象 | 要不要改 |
|---|---|---|
| `set_new_tool` 的**黑名单**写法 | `mom_command_status` 缺失、或值不在黑名单里（如 `USER DEFINED`）→ **判成 Active**（fail-open） | 想更保守就换成白名单：`if { [lsearch -exact {active activate on yes true enable enabled 1} $st] >= 0 }` |
| 文本填 `60min`、`约60` | 正则抠出 `60`，**通过** | 宽容设计，不用改；但**建议还是只填数字**，避免"约"字后面跟别的数字时抠错（正则取的是**第一个**匹配）|
| 文本填 `60 70` | 抠出第一个 `60` | 同上，只填一个数字 |
| 刀号正好 100 | 当成真 T100 | 见 §7.3 |
| 该刀切削时间为 0 | 静默跳过（不算缺数据、不输出） | 合理（除零保护） |
| `N100` 与程序现有标号冲突 | 跳转错乱 | 改 `set n 100` 那一行 |
| 多把刀共用同一刀号 | 时间会累加到同一账上 | 是设计如此（同刀号=同物理刀） |

## 7.9 ⚠ 现场踩过的最大一个坑：**两台电脑的文件不同步**

历史事故：向用户交付两个新 proc 后，功能仍不生效。诊断定案手法（**值得复用**）：

- **同开关控制的两组诊断行，一组出现、另一组不出现** ⇒ 开关是开的，而"不出现那组"所在的 proc **没被替换**。
- 本例：NC 里有 7 处 `(DBGNEW …)`，却**一行 `(DBGLIFE …)` 都没有** ⇒ 出程序那台上的 `PB_CMD_output_tool_life` 还是"写警告、悄悄跳过"的旧版。**不是账本问题、不是新代码有 bug。**

⇒ 给每段新逻辑配一条**同开关控制的、独一无二的诊断行**，以后任何"改了但没效果"都能 30 秒内区分"**没换上**"和"**换上了但有别的 bug**"。

---

# 第 8 章 · 维护、排错、验收

## 8.1 三条纪律（老规矩）

1. **先在 PB 界面里改完一切 → Save 一次 → 之后永不再在 PB 里 Save。**
   理由：`MOM_xxx` 事件 proc 与 `.def` 块模板会被 PB 按 `.pui` 重写；`PB_CMD_*` 的正文虽实测能存活，但"两台越差越多"多半就是这么来的。
2. **手改只改 `PB_CMD_*` 的正文，绝不改 `MOM_xxx` 那 5 行入口。**
3. **明文母本必须留着**（`_bf.tcl`）；加密版**绝不能再让 PB 打开并保存**（PB 会按 `.pui` 重写 `.tcl`，加密当场作废）。

## 8.2 现场排错表（打开 NC 搜这些串）

| 看到什么 | 说明 | 下一步 |
|---|---|---|
| 一句 `DBGLIFE` 都没有 | 该命令**还是旧版**，或跑的不是这份后处理 | 在**出程序那台**上搜 `DBGLIFE`，确认文件；按 §7.9 定案 |
| `(DBGLIFE life_on=unset)` | 一道工序都没加 `set_tool_life`，或 UDE 没送达 | 检查工序开始事件；检查 NX 全局 `ude.cdl`（§7.7） |
| `(DBGLIFE life_on=1)` + `(DBGLIFE order=10 11 12 13)` | 开关生效，4 把刀都进账本了 | 继续往下看 |
| `(DBGLIFE tool=11 cut=… ref=<none>)` | **T11 的文本格是空的** | 去填数字 |
| `(DBGLIFE add tool=11 cut=0.0 …)` | 这道工序没读到切削时间 | 把这段 NC 发维护者 |
| `(TOOL LIFE DATA MISSING : T11 T12 )` + 后处理停下 | **新逻辑生效了，正是要的行为** | 去把那两把刀填上 |
| 每把刀都有 `ref=60`，结尾出现 N 组寿命段 | 全部通过 ✅ | 把 `::nc_dbg` 改回 `0` |
| 程序尾**没有**寿命段，也没有任何报错 | 总开关没置位（正常行为） | 若你确实加了 UDE → 按上一行排查 |
| 弹窗中文乱码 | 机床不支持 GB2312 或该字段是 ASCII-only | 看 NC 里那行纯 ASCII 的 `TOOL LIFE DATA MISSING`，或按 §5.6 换编码 |

## 8.3 验收判据（每次改完必查）

1. **代码逐字一致**：跑 `check_md_code15.py`，把本文档里 **8 段源码代码块**（§3.1–§3.6，§3.6 下含 3 段）与明文母本做逐字比对 —— **差异必须为 0**。（§7 里那些故意写错的 ❌ 示例不算母本原文，脚本已按设计跳过；当前实测：8/8 一致、合计不一致 0。）
2. **逻辑离线回归**：跑 `Test-ToolLife.py`，第 6 章 5 张表的结论必须复现（尤其 **S2 逐字对照不一致项 = 0**）。
3. **开关两态都要测**：不加 UDE（应零输出）+ 加 UDE（应出寿命段）。
4. **缺数据要真断**：故意留一把刀不填文本，确认**后处理当场停**、NC 里只有一行 `MISSING`。
5. **验收完关诊断**：`::nc_dbg` 改回 `0`，NC 里不应再有 `DBG…` 行。
6. **PB 保存后的存活检查**：`grep -c "proc PB_CMD_output_tool_life { } {"` 必须 = **1**（不是 0、也不是 2）。

## 8.4 机床侧前提（上机前必须确认）

- [ ] `#600`~`#999` 区在这台机床上**没有别的程序在用**（公共变量是全局的，会互相踩）。
- [ ] `#650`、`#655`… 这些号在本机床的**刀具寿命/调正量体系**里确实是按 `600+刀号×5` 分的。
- [ ] `#3006=1(…)` 在本机床上的**实际表现**（停机方式、是否需 RESET）—— **首次空跑一件确认**。
- [ ] 消息里的中文能正常显示（GB2312 段）。
- [ ] **本程序里没有 `N100` 及以上的标号**，或已把 `set n 100` 改到冲突区间之外。
- [ ] 第一件跑完后，机械侧确认 `#650` 变成了 `1`（而不是 `2` 或 `0`）。

---

# 第 9 章 · 附录：变量 / 常量 / 文件速查

## 9.1 本功能用到的全局变量（`::` 前缀）

| 变量 | 含义 | 谁写 |
|---|---|---|
| `::nc_tool_life_on` | 总开关（任意工序 Active 即 1） | `PB_CMD_set_tool_life_flag` |
| `::nc_dbg` | 调试总开关（1 = 打诊断行） | `PB_CMD_custom_command_11` |
| `::nc_tool_order` | 账本：刀具出现顺序 | `PB_CMD_collect_cutting_time` |
| `::nc_cut_time(t)` | 账本：该刀本程序累计切削分钟 | 同上 |
| `::nc_ref_text(t)` | 账本：该刀参考寿命文本（首次非空值） | 同上 |
| `::nc_new_tool_request` | （修订 2）按工序的 New 标记 | `PB_CMD_set_new_tool_flag` |
| `::nc_new_tool_seen(t)` | （修订 2）本程序该刀是否已出过 New 段 | `PB_CMD_custom_command_10` |
| `::nc_addvar_on` | （修订 14）坐标宏开关 | `PB_CMD_add_variable_flag` |

## 9.2 NX 送进来的变量

| 变量 | 含义 | 单位 / 值域 |
|---|---|---|
| `mom_tool_number` | 刀号（可能是"刀号×100+补偿号"的组合码） | 数 |
| `mom_cutting_time` | **本道工序**有效切削时间 | 分钟；个别工序可能不存在 |
| `mom_tool_text` | 刀具对话框「更多 → 机床控制 → 文本」格 | 字符串 |
| `mom_command_status` | UDE 的 Status 下拉值 | **全大写** `ACTIVE` / `INACTIVE` |

## 9.3 关键常量与公式

| 项 | 值 / 公式 |
|---|---|
| 寿命计数器变量号 | `600 + 刀号 × 5` |
| 报警段标号 | `100 + 2×序号`（`N100/N101`、`N102/N103`…） |
| 上限 | `int(参考寿命 ÷ 单件累计切削时间)` |
| 报警语句 | `#3006=1(T<刀号> <GB2312-hex>)` |
| 报警文案 | `@0400CAD9C3FCD6D5C1CBA3A1@` = 「寿命终了！」 |
| 缺数据证据行 | `(TOOL LIFE DATA MISSING : T11 T12 )` |
| 诊断行前缀 | `DBGLIFE` / `DBGNEW`（受 `::nc_dbg` 控制） |

## 9.4 相关文件

| 文件 | 位置 / 用途 |
|---|---|
| 明文母本 | `桌面\mill_intergrex_300_2022_bf.tcl`（改代码的依据；`.tcl` 是加密产物时以它为准） |
| `.pui` | `桌面\mill_intergrex_300_2022.pui`（UDE 登记 790 行 / 事件成员 740、837、841 行） |
| `.cdl` | `桌面\mill_intergrex_300_2022.cdl`（`EVENT set_tool_life` @13276） |
| NX 全局 UDE | `…\Designcenter2606\MACH\resource\user_def_event\ude.cdl`（`EVENT set_tool_life` @17） |
| 离线验收脚本 | `桌面\Test-ToolLife.py` / `tclenc_verify\toollife_test.py` |
| 实测报告 | `tclenc_verify\toollife_test_report.txt` |
| 文档代码校验 | `tclenc_verify\check_md_code15.py` |
| 真 Tcl 解释器 | `…\Designcenter2606\NXBIN\python\tcl86t.dll`（8.6.15） |
| 上游文档 | `后处理改造_修订2_NewOld按工序.md`、`修订3_刀具寿命中断版.md`、`修订14_坐标加变量号_防忘文章.md` |

---

*（本文所有代码块与源文件做过逐字比对；所有行为结论来自真 Tcl 8.6 实测，非推测。改代码前请先跑 §8.3 的两条。）*
