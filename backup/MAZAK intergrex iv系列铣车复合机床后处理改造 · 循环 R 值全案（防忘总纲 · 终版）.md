# 后处理改造 · 循环 R 值全案（防忘总纲 · 终版）

> **对象**：`mill_intergrex_300_2022`（`.tcl` / `.def` / `.cdl` / `.pui`）
> **机床**：Mazak INTEGREX 300（车铣复合，5 轴，MSYS 带旋转）
> **定稿**：2026-10-16 · **修订 12（终版）** —— 合并「修订10 的方案」+「修订11 的逐行注释」+「19:27 版复核记录」，并按 19:38 的最终文件校正
> **本文用途**：防忘。忘了就从头看第 0 章，够用；要改代码看第 4 章。

---

## 目录

| 章 | 内容 |
|---|---|
| 0 | 结论速查（忘了只看这章） |
| 1 | 前因：`R0.0` 是怎么冒出来的 |
| 2 | 三条公式的对错（含一条会撞刀的） |
| 3 | 正解的架构：值在 TCL 算，格式在 `.def` 算 |
| 4 | `PB_CMD_before_output` **逐行注释**（全文） |
| 5 | `.def` 的 21 处改动清单 |
| 6 | 初始化与兜底（`PB_CMD_custom_command_11`） |
| 7 | 真 TCL 8.6 实测数据 |
| 8 | 操作步骤（在 Post Builder 里怎么改） |
| 9 | 维护与跨机注意事项 |
| 10 | 附录：变量与术语速查 |

---

# 第 0 章 · 结论速查

## 0.1 机床对 R 的定义（一切的地基）

Mazak INTEGREX 这一台，程序里 `G8x ... R<数>` 的 `<数>` 含义是：

> **R = 执行这条 `G8x` 那一刻刀具实际所在的 Z − 该循环 R 点的 Z**

不是"R 点的绝对坐标"，也不是"NX 给的某个距离量"。**是"刀站在哪"与"R 面在哪"的差。**

## 0.2 目标输出（就是你验收时看到的）

| 转移方式 | 第 1 孔 | 第 2 孔 |
|---|---|---|
| 安全距离-最短距离 | `R15.5` | `R31.5152` |
| 直接 | `R15.5` | `R0.0` |

`R0.0` 不是错误，是**正确**的 —— 直接模式下第 2 孔开始前，刀已经站在 R 面本身（`Z30.4848`）上了，差就是 0。

## 0.3 三处改动位置（全部已在 19:38 版落地）

| # | 文件 | 位置 | 内容 |
|---|---|---|---|
| 1 | `.tcl` | `PB_CMD_before_output`（行 5931–6010），**A 段** | 跟踪刀位 + 算出 `::nc_cycR` |
| 2 | `.def` | 21 个循环块的 R 单元 | 一律改成 `R[$nc_cycR]` |
| 3 | `.tcl` | `PB_CMD_custom_command_11`（行 6873–6874） | 每程序开头把 R 账本归零 |

`.tcl` 里**原有的 B 段（坐标宏，需求③）一字未动**。

## 0.4 验收判据（每次改完必查）

```
① Z(G8x 前一个带 Z 的移动) − R(该 G8x) = 该循环 R 点的绝对 Z
   —— 若为负，说明刀位跟踪被污染，立刻停。
② R 字个数 = 21（改前改后相等）
③ 文件行数不变
④ 安全距离那版必须与你现在跑通的逐字一致（零回归）
```

---

# 第 1 章 · 前因：`R0.0` 是怎么冒出来的

## 1.1 你当时看到的现象

**安全距离-最短距离**（看着没问题）：

```
G68.5 X0.0 Y0.0 Z0.0 I0.0 J1. K0.0 R90.
X111.5 Y0.0 Z62. C0.0 S3537 M203
G000 X111.5 Y0.0
G083 X111.5 Y0.0 Z31.7357 F141.48 R15.5   Q1.9607
G080
G000 Z62.
      X84.9138 Y0.0
G083 X84.9138 Y0.0 Z15.7205 F141.48 R31.5152 Q1.9607
G080
G000 Z62.
G080
```

**直接**（看着不对）：

```
G68.5 X0.0 Y0.0 Z0.0 I0.0 J1. K0.0 R90.
X111.5 Y0.0 Z62. C0.0 S3537 M203
G000 X111.5 Y0.0
G083 X111.5 Y0.0 Z31.7357 F141.48 R0.0    Q1.9607   <-- 第 1 孔：应为 15.5
G080
G000 Z46.5                                            <-- 只抬到 R 面
G080
G000 X84.9138 Z30.4848                                <-- 中间定位（关键行）
G000 X84.9138 Y0.0
G083 X84.9138 Y0.0 Z15.7205 R31.5152 Q1.9607          <-- 第 2 孔
G080
G000 Z62.
```

两者差在**抬刀高度**和**有没有中间定位行**。

## 1.2 为什么"安全距离"看着对

安全距离模式下，NX 每个孔后都把刀抬回**退刀平面 62**，下一孔再下来。所以每个 `G8x` 执行时，刀都恰好从**同一个高度 62** 出发：

```
R(孔1) = 62 − 46.5    = 15.5   ✓
R(孔2) = 62 − 30.4848 = 31.5152 ✓
```

于是任何一种"看起来合理"的公式都能算对 —— **这是巧合**，它把 bug 藏住了。

## 1.3 为什么"直接"会 `R0.0`

直接模式 NX 不再抬满，它**只把刀抬到 R 面本身**（`G000 Z46.5`），然后走一段**中间定位**（`G000 X84.9138 Z30.4848`）直接横移到第 2 孔。

第 1 孔出 `R0.0` 的原因：**如果刀位跟踪把 `G8x` 那一行自己的 Z 当成了刀位**，就会拿孔底深度去减 R 点：

```
R = 31.7357（孔底深度，不是刀位！） − 46.5 = −14.7643
    → 负数被夹零 → R0.0   ✗
```

第 2 孔：刀实际站在 `Z30.4848`，R 点也在 `30.4848`：

```
R = 30.4848 − 30.4848 = 0.0   ✓（这才是对的）
```

**同一个 `R0.0`，第 1 孔是错的、第 2 孔是对的** —— 这就是为什么必须"跟踪真实刀位"，不能拿格式凑。

## 1.4 🔴 顺便拆掉的一颗雷（务必记住）

你当时那份程序里，**直接模式第 2 孔曾输出 `R31.5152`**（用"退刀平面 − R 点"算的）。但执行时刀在 `Z30.4848`：

```
机床算出的 R 面 = 30.4848 − 31.5152 = Z−1.03
孔底 Z15.7205  →  R 面比孔底还低 16.75 mm
```

**刀会先扎到 R 面以下的空气里再"钻"上去 —— 直接撞刀。**

> **结论**：任何"用固定参考面（退刀平面 / 初始平面）去减"的公式，在**直接模式**下都可能给出**偏大的 R**，R 面被推到孔底下。必须用真实刀位。

---

# 第 2 章 · 三条公式的对错

| # | 公式 | 安全距离 | 直接 | 判定 |
|---|---|---|---|---|
| A | `R[$mom_cycle_rapid_to_pos(轴)]`（出厂） | `R46.5` / `R30.4848`（绝对坐标） | 同 | ❌ 机床不认，等于把 R 点绝对 Z 当距离 |
| B | `$mom_cycle_retract_to_pos(轴) − $mom_cycle_rapid_to_pos(轴)` | 62−46.5=`15.5` ✓ / 62−30.4848=`31.5152` ✓ | 62−46.5=`15.5` ✓ / 62−30.4848=`31.5152` ✗ | ⚠️ 直接模式偏大 → **撞刀** |
| C | **`刀位Z − $mom_cycle_rapid_to_pos(轴)`，下限 0** | `15.5` / `31.5152` ✓ | `15.5` / `0.0` ✓ | ✅ **采用** |

公式 B 之所以在"直接"下错，是因为它假设"刀在退刀平面 62 上"。直接模式下 NX 只抬到 46.5，这个假设塌了。

公式 C 唯一依赖的是"**刀此刻在哪**" —— 而这是 NX 一定会告诉我们的（就是那条 `G000 X84.9138 Z30.4848`）。

---

# 第 3 章 · 正解的架构

## 3.1 分工：值在 TCL 算，格式在 `.def` 算

```
      ┌──────────────────────────── .tcl ────────────────────────────┐
      │  PB_CMD_before_output                                        │
      │   ① 从当前 NC 块里读出真实刀位 Z      -> ::nc_cycR_z          │
      │   ② ::nc_cycR = ::nc_cycR_z − R点Z   （负数夹 0）             │
      └───────────────────────────────┬──────────────────────────────┘
                                      │  全局变量
                                      ▼
      ┌──────────────────────────── .def ────────────────────────────┐
      │  21 个循环块：R[$nc_cycR]                                    │
      │  格式仍走原有的  ADDRESS R { FORMAT Coordinate ... }          │
      │    ⇒ 自动出 15.5 / 31.5152 / 0.0                             │
      └──────────────────────────────────────────────────────────────┘
```

**为什么坚持这个分工？**

| 方案 | 问题 |
|---|---|
| 全塞进 `.def` 表达式（`R[$nc_cycR_z - $mom_cycle_rapid_to_pos(...)]`） | 21 处各写一段加减+夹零，改一处漏一处；`.def` 表达式里做"小于 0 则取 0"很别扭 |
| 在 TCL 里**重写整行文本**（自己拼 `R15.5`） | 要手工复刻 `15.5` / `31.5152` / `0.0` 三种写法，还得自己处理坐标宏、字头 —— 格式一错就撞刀（**这是我们走了一圈后否掉的路**） |
| ✅ **值在 TCL、格式在 `.def`** | 21 处表达式只要**一行一改**；输出形态完全由现有的 `FORMAT Coordinate` 保证，与跑通的那版逐字一致 |

`.def` 的 `ADDRESS R` 定义（行 458，**未改动**）：

```
  ADDRESS R
  {
      FORMAT      Coordinate
      FORCE       off
      MAX         99999.999 Truncate
      MIN         -99999.999 Truncate
      LEADER      "R"
      ZERO_FORMAT Zero_real
  }
```

> ⚠️ `MIN -99999.999 Truncate` 只做**截断**，**不会把负数归零**。所以负 R 必须由 TCL 侧夹掉 —— 这也是第 4 章里那句夹零必须存在的原因。

## 3.2 ★ 运行次序（"隔一块"）—— 理解全案的关键

NX 对**每一个** NC 块的处理顺序是：

```
     块 N 要输出了
        │
        ├─(a) 用 .def 模板把块 N 的文本填好   ←── 此刻读 $nc_cycR
        │
        └─(b) 调 PB_CMD_before_output        ←── 此刻写 $nc_cycR
```

**所以：某个 `G8x` 行印出来的 R，用的是"它前面那个块处理时算出的" `$nc_cycR`。**

这够用 —— 因为定位移动（`G000 X84.9138 Z30.4848`）永远排在 `G8x` 前面，它先把刀位改掉，R 自然跟着走到正确值。

### 逐块追踪（安全距离，孔 2）

| 步 | 块 | (a) 印出的 R | (b) 之后的 `::nc_cycR_z` | (b) 之后的 `::nc_cycR` |
|---|---|---|---|---|
| 1 | `X111.5 Y0.0 Z62. C0.0 S3537 M203` | — | **62.** | （R 点还没进循环，不算） |
| 2 | `G000 X111.5 Y0.0` | — | 62. | 62−46.5 = **15.5** |
| 3 | `G083 … R[15.5]` | **15.5** ✓ | 62.（G8x 不跟踪） | 15.5 |
| 4 | `G080` | — | 62.（G8x 不跟踪） | 15.5 |
| 5 | `G000 Z62.` | — | 62. | 62−**30.4848** = **31.5152** |
| 6 | `G000 X84.9138 Y0.0` | — | 62.（无 Z） | 31.5152 |
| 7 | `G083 … R[31.5152]` | **31.5152** ✓ | 62. | 31.5152 |

### 逐块追踪（直接，孔 2）

| 步 | 块 | (a) 印出的 R | (b) 之后 `::nc_cycR_z` | (b) 之后 `::nc_cycR` |
|---|---|---|---|---|
| … | （孔 1 同上，出 `15.5`） | 15.5 ✓ | | |
| 5 | `G000 Z46.5` | — | 46.5 | 46.5−46.5 = 0.0 |
| 6 | `G080` | — | 46.5 | 0.0 |
| 7 | **`G000 X84.9138 Z30.4848`** | — | **30.4848** | **0.0** |
| 8 | `G000 X84.9138 Y0.0` | — | 30.4848（无 Z） | **0.0** |
| 9 | `G083 … R[0.0]` | **0.0** ✓ | 30.4848 | 0.0 |

**第 7 步那条中间定位行把刀位更新成 `30.4848`，是"直接模式第 2 孔得 `0.0`"的全部原因。** 这条线不跟踪，就会退回到错误的 `31.5152`。

---

# 第 4 章 · `PB_CMD_before_output` 逐行注释

文件 `mill_intergrex_300_2022.tcl`，**行 5931 – 6010**。下面按"先原文、后逐行解释"两块给出。

## 4.1 原文（你机器上的当前版本）

```tcl
#=============================================================
proc PB_CMD_before_output { } {
#=============================================================
   global mom_o_buffer
   global mom_sys_leader
   global mom_sys_control_out
   global mom_tool_number
   global mom_cycle_rapid_to_pos
   global mom_cycle_spindle_axis

   # ============ A. CYCLE R : value + tool-Z tracker ============
   #  (1) update "current tool Z" from this block
   #      -> ::nc_cycR_z
   #  (2) recompute  ::nc_cycR = tool Z - R point Z   (the value .def reads)
   #  Order matters: track FIRST, then compute.
   #  Runs for every NC block (MOM_before_output -> PB_CMD_kin_before_output).
   #  Only reads mom_o_buffer; never calls any MOM_ output command.
   if { ![info exists ::nc_cycR] } { set ::nc_cycR 0.0 }

   if { [info exists mom_o_buffer] && [string length $mom_o_buffer] > 0 } {

      # ---- (1) track tool Z ----
      set nc_tb $mom_o_buffer
      set nc_tk 1
      if { [string first $mom_sys_control_out $nc_tb] >= 0 } { set nc_tk 0 }
      if { [string first ")" $nc_tb] >= 0 } { set nc_tk 0 }
      if { [regexp {G0*8[0-9]\M} $nc_tb] } { set nc_tk 0 }
      if { [regexp {G0*(53|28|30|52|92|10|04|4|68|91)(\.[0-9]+)?\M} $nc_tb] } { set nc_tk 0 }
      set nc_lz $mom_sys_leader(Z)
      if { $nc_tk && [string length $nc_lz] > 0 } {
         set nc_hits [regexp -all -inline -- [format {%s(-?[0-9.]+)} $nc_lz] $nc_tb]
         if { [llength $nc_hits] > 0 } { set ::nc_cycR_z [lindex $nc_hits end] }
      }

      # ---- (2) recompute R for the next block to read ----
      set nc_ax 2
      if { [info exists mom_cycle_spindle_axis] } { set nc_ax $mom_cycle_spindle_axis }
      if { [info exists mom_cycle_rapid_to_pos($nc_ax)] && [info exists ::nc_cycR_z] } {
         set ::nc_cycR [expr {double($::nc_cycR_z) - double($mom_cycle_rapid_to_pos($nc_ax))}]
         if { $::nc_cycR < 0.0 } { set ::nc_cycR 0.0 }
      }
   }

   # ============ B. coord macro (UNCHANGED - rev6) ============
   if { ![info exists ::nc_addvar_on] || $::nc_addvar_on != 1 } { return }
   if { ![info exists mom_o_buffer] || [string length $mom_o_buffer] == 0 } { return }

   set buf $mom_o_buffer

   if { [string first $mom_sys_control_out $buf] >= 0 } { return }
   if { [string first ")" $buf] >= 0 } { return }
   if { [regexp {G(53|28|30|52|92|10|04|4|68)\M} $buf] } { return }

   set tn 0
   if { [info exists mom_tool_number] } {
      set tn $mom_tool_number
      if { $tn > 100 } { set tn [expr {int(double($tn) / 100.0 + 0.5)}] }
   }
   set xn [expr {600 + $tn * 5 - 4}]
   set yn [expr {600 + $tn * 5 - 3}]
   if { [info exists ::nc_x_var] && $::nc_x_var > 0 } { set xn $::nc_x_var }
   if { [info exists ::nc_y_var] && $::nc_y_var > 0 } { set yn $::nc_y_var }

   set lx $mom_sys_leader(X)
   set ly $mom_sys_leader(Y)
   set n 0

   if { [string length $lx] > 0 } {
      set pat [format {%s(-?[0-9.]+)} $lx]
      set rep [format {%s[%s+#%s]} $lx "\\1" $xn]
      incr n [regsub -all $pat $buf $rep buf]
   }
   if { [string length $ly] > 0 } {
      set pat [format {%s(-?[0-9.]+)} $ly]
      set rep [format {%s[%s+#%s]} $ly "\\1" $yn]
      incr n [regsub -all $pat $buf $rep buf]
   }

   if { $n > 0 } { set mom_o_buffer $buf }

}
```

## 4.2 逐行解释

### （0）`global` 声明 —— 为什么必须写

```tcl
   global mom_o_buffer
   global mom_sys_leader
   global mom_sys_control_out
   global mom_tool_number
   global mom_cycle_rapid_to_pos
   global mom_cycle_spindle_axis
```

| 变量 | 是什么 | 用在哪 |
|---|---|---|
| `mom_o_buffer` | **即将输出的这一整块 NC 文本**（`.def` 已经填好） | A 段读它的 Z；B 段改写它 |
| `mom_sys_leader` | 字头数组，`mom_sys_leader(Z)` = `"Z"`、`(X)` = `"X"`…（tcl:295 定义） | 用**字头**而不是硬编码 `Z` 去找 Z 字，换后处理配置也不会失效 |
| `mom_sys_control_out` | 注释起始符（本机 `"("`） | 用来跳过注释行，别把注释里的数字当刀位 |
| `mom_tool_number` | 当前刀号 | B 段算坐标宏的刀补变量号 |
| `mom_cycle_rapid_to_pos` | **R 点绝对坐标数组**（`(0)`=X, `(1)`=Y, `(2)`=Z） | A 段算 R 的被减数 |
| `mom_cycle_spindle_axis` | 当前循环沿哪个轴钻（本机 `2` = Z） | 决定取 `_pos` 的哪一分量 |

> **TCL 规则**：过程内不 `global`，读到的就是自己的局部（不存在 → 报 `no such variable`）。`.tcl` 顶层的 `set mom_xxx` 都是全局，所以这里必须逐一 `global` 才能看见。

### （1）兜底初始化 —— 一行保险，防止后处理被中断

```tcl
   if { ![info exists ::nc_cycR] } { set ::nc_cycR 0.0 }
```

`.def` 里已经是 `R[$nc_cycR]`。如果 `::nc_cycR` 从来没被赋过值，`.def` 求值时会抛 `can't read "::nc_cycR": no such variable`，**整个后处理中断**。

正常流程走不到（第 7 章实测的第一个块就会赋值），但这行是**白捡的保险**。写 `0.0` 而不是 `0` 是为了让格式输出 `R0.0`（`ZERO_FORMAT Zero_real`）。

> 用 `::` 前缀 = 显式指全局，在过程里不需要 `global` 声明（`::` 是全局命名空间的绝对路径）。

### （2）外层 `if` —— 空块直接跳过

```tcl
   if { [info exists mom_o_buffer] && [string length $mom_o_buffer] > 0 } {
```

两块保险：变量存在 **且** 非空串。空块（比如纯换行）里没有 Z 可读，跳过能省掉不必要的正则开销，也避免 `regexp -all -inline` 拿到空列表。

### （3）跟踪刀位：三个"不跟踪"的开关

```tcl
      set nc_tb $mom_o_buffer
      set nc_tk 1
      if { [string first $mom_sys_control_out $nc_tb] >= 0 } { set nc_tk 0 }
      if { [string first ")" $nc_tb] >= 0 } { set nc_tk 0 }
      if { [regexp {G0*8[0-9]\M} $nc_tb] } { set nc_tk 0 }
      if { [regexp {G0*(53|28|30|52|92|10|04|4|68|91)(\.[0-9]+)?\M} $nc_tb] } { set nc_tk 0 }
```

`nc_tk = 1` 表示"这一块的 Z 可以当刀位"，任何一条命中就置 0。

| 排除项 | 挡的是 | 不挡会怎样 |
|---|---|---|
| 含 `$mom_sys_control_out`（`(`） | 注释块，如 `(DBGNEW …)` | 注释里的数字被当刀位 |
| 含 `)` | 同上（另一侧） | 同上 |
| **`G0*8[0-9]`** | **本行就是循环行**（`G080`~`G089`，含 `G080` 取消行） | ⚠️ **致命**：`G083 … Z31.7357` 的 `Z31.7357` 是**孔底深度**不是刀位；跟踪它 → R 全错 |
| `G0*(53\|28\|30\|52\|92\|10\|04\|4\|68\|91)` | 机床坐标 / 回零 / 局部坐标 / 绝对模式 / 暂停 / 第四轴（`G68` 坐标旋转）/ 增量模式 | `G68.5 X0 Y0 Z0 …` 里的 `Z0.0` 是**旋转中心**，跟踪它 → R 从 0 起算，全错 |

> `\M` 是 TCL 正则的**词尾锚**，防止 `G04` 在 `G040` 里被误配。
> `G91`/`G90` 一并排除：`G91` 之后行内坐标可能变成增量值，不是刀位。

### （4）抓 Z 字（关键的一句话）

```tcl
      set nc_lz $mom_sys_leader(Z)
      if { $nc_tk && [string length $nc_lz] > 0 } {
         set nc_hits [regexp -all -inline -- [format {%s(-?[0-9.]+)} $nc_lz] $nc_tb]
         if { [llength $nc_hits] > 0 } { set ::nc_cycR_z [lindex $nc_hits end] }
      }
```

- 用 `$mom_sys_leader(Z)` 取值 = **不硬编码 `Z`**。
- `-all -inline` 把匹配项**连同捕获组**一起返回，所以 `nc_hits` 形如 `{Z62. 62.}`。
- 取 `[lindex … end]` = **这一个捕获组**（最后一项）。若一行里有两个 Z，取最后一个更接近"该行结束时刀的位置"。
- `nc_tk` 为假时**整段跳过** —— 即便行里有 `Z31.7357`，也绝不动 `::nc_cycR_z`。

> 变量名带 `::` 前缀（`::nc_cycR_z`）= **全局**，这样 `.def` 和 `custom_command_11` 都能看见/清理。

### （5）算 R（顺序：先跟踪，后计算）

```tcl
      set nc_ax 2
      if { [info exists mom_cycle_spindle_axis] } { set nc_ax $mom_cycle_spindle_axis }
      if { [info exists mom_cycle_rapid_to_pos($nc_ax)] && [info exists ::nc_cycR_z] } {
         set ::nc_cycR [expr {double($::nc_cycR_z) - double($mom_cycle_rapid_to_pos($nc_ax))}]
         if { $::nc_cycR < 0.0 } { set ::nc_cycR 0.0 }
      }
```

- `nc_ax` 默认 `2`（Z），有 `mom_cycle_spindle_axis` 就用它的值 —— 车铣复合换主轴方向时不会算错轴。
- 两个 `info exists` 同时成立才算：**R 点存在** 且 **刀位已跟踪到**。少任何一个就保持上一次的值（不动，比乱算安全）。
- `double(...)`：**必须显式转**。原因见下。
- `expr` 的结果赋给 `::nc_cycR`，TCL 会把结果**规范化成最短字符串** —— 实测 `15.5` / `31.5152` / `0.0`，**没有浮点尾巴**。
- `if { $::nc_cycR < 0.0 }` 夹零：因为 `ADDRESS R` 的 `MIN` 只截断不归零（见 3.1）。

> **为什么一定要 `double()`**：TCL 的 `expr` 会把前导 0 的数字当**八进制**。`expr {08 - 1}` 直接报 `invalid octal number`。NC 里的数虽然少见 `08.0`，但 `double()` 一劳永逸，值得养成习惯。

> **为什么"跟踪在前、计算在后"**：同一块里，要先让 `::nc_cycR_z` 更新成这一块的真实刀位，再拿新刀位算 R。反过来的话，算出来的 R 永远慢一块。**这一条在攻丝场景下尤其致命** —— 攻丝前会先 `G000 Z46.5` 快移到 R 面，那次快移必须被计入。

### （6）B 段（坐标宏）—— 与 R 完全解耦

```tcl
   # ============ B. coord macro (UNCHANGED - rev6) ============
   if { ![info exists ::nc_addvar_on] || $::nc_addvar_on != 1 } { return }
```

**B 段一字未动**，只强调两点：

1. **A 段必须在 B 段的 `return` 之前。** 因为 B 段一开头的 `return` 会在"本工序没开坐标宏"时直接退出。若把 A 段放到它后面，"没开坐标宏的工序"就**不跟踪刀位**，R 必错。
2. A 段**只读 `mom_o_buffer`，从不调用任何 `MOM_` 输出命令**。所以它与 B 段互不干扰，也不会有"重复输出"的风险。

---

# 第 5 章 · `.def` 的 21 处改动清单

改法统一：把 `R[<原来的东西>]` 换成 `R[$nc_cycR]`，`\opt` 后缀**原样保留**。

| # | 行号 | 块名 | 界面归属（`.pui` 事件名） | 尾缀 |
|---|---|---|---|---|
| 1 | 997 | `cycle_bore` | Bore | `\opt` |
| 2 | 1036 | `cycle_bore_back` | Bore Back | `\opt` |
| 3 | 1076 | `cycle_bore_drag` | Bore Drag | `\opt` |
| 4 | 1115 | `cycle_bore_dwell` | Bore Dwell | `\opt` |
| 5 | 1155 | `cycle_bore_manual` | Bore Manual | `\opt` |
| 6 | 1194 | `cycle_bore_manual_dwell` | Bore Manual Dwell | `\opt` |
| 7 | 1234 | `cycle_bore_no_drag` | Bore No Drag | `\opt` |
| 8 | 1274 | `cycle_drill` | Drill | `\opt` |
| 9 | 1313 | `cycle_drill_break_chip` | Drill Break Chip | `\opt` |
| 10 | **1353** | **`cycle_drill_deep`** | **Drill Deep（你实际在用的）** | `\opt` |
| 11 | 1393 | `cycle_drill_dwell` | Drill Dwell | `\opt` |
| 12 | 1535 | `cycle_tap` | Tap | `\opt` |
| 13 | 1558 | `cycle_tap_4` | —— **孤儿块**（`.pui` 无引用） | 无 |
| 14 | 1575 | `cycle_tap_5` | —— 孤儿块 | 无 |
| 15 | 1614 | `cycle_tap_break_chip` | Tap Break Chip | `\opt` |
| 16 | 1659 | `cycle_tap_deep` | Tap Deep | `\opt` |
| 17 | 1704 | `cycle_tap_float` | Tap Float | `\opt` |
| 18 | 2426 | `tap_3` | —— 孤儿块 | 无 |
| 19 | 2443 | `tap_deep` | —— 孤儿块 | 无 |
| 20 | 2461 | `tap_deep_break_chip_move` | —— 孤儿块 | 无 |
| 21 | 2559 | `post_cycle_set` | **Cycle Parameters（循环参数）** | `\opt` |

**几点说明**

- 第 13/14/18/19/20 这 5 个"孤儿块"在 `.pui` 里**没有被任何事件引用**，界面上根本找不到 —— 这是正常的，一并改掉是为了将来万一被启用时不带旧毛病。
- 第 21 处 `post_cycle_set` **不在任何具体循环底下**，它是 **`Cycle Parameters`（循环参数）事件**的第 4 个块（`.pui` 第 795 行的铁证）。在"钻孔 / 深孔钻 / 攻丝"底下翻是**永远找不到**的。
- 改动为什么安全：`.pui` 里**没有一处 `BLOCK_TEMPLATE`**，也**不存任何 TCL 代码正文**（第 17–18 行只是文件名引用）⇒ **直接改 `.def` 不会被 Post Builder 保存时覆盖**。

---

# 第 6 章 · 初始化与兜底

## 6.1 每程序开头把 R 账本归零

文件 `mill_intergrex_300_2022.tcl` 行 **6865–6874**，`PB_CMD_custom_command_11` 内：

```tcl
proc PB_CMD_custom_command_11 { } {
#=============================================================
#=============================================================
global block_number mom_tool_number

set block_number 1
set new_flag 0

   set ::nc_cycR 0.0
   unset -nocomplain ::nc_cycR_z
```

| 行 | 作用 |
|---|---|
| `set ::nc_cycR 0.0` | R 账本清零（保证 `.def` 读到的是个数，不是未定义） |
| `unset -nocomplain ::nc_cycR_z` | 刀位跟踪清零 —— **换程序必须清**，否则新程序第一孔会沿用上个程序最后一次的 Z |

挂钩点：`.pui` 第 **740** 行，**`Start of Program`** 事件 —— 每个程序开头跑一次。

> `unset -nocomplain` 而不是 `unset`：变量本来就不存在时也不会报错（第一个程序开头必然不存在）。

## 6.2 挂钩链条（三处，缺一不可）

```
MOM_before_output            (tcl:123，注释明写"每个 NC 块输出前执行")
      └─ PB_CMD_kin_before_output   (tcl:8285)
              └─ PB_CMD_before_output  (tcl:5931)   ← A 段 + B 段
```

`.pui` 侧则把 `PB_CMD_custom_command_15` 挂在 **16 个循环事件的首位**（Drill / Drill Dwell / Drill Deep / Drill Break Chip / Tap / Tap Float / Tap Deep / Tap Break Chip / Bore / Bore Drag / Bore No Drag / Bore Manual / Bore Dwell / Bore Back / Bore Manual Dwell），把 `PB_CMD_custom_command_11` 挂在 `Start of Program`。

> `PB_CMD_custom_command_15` 目前是**空 proc**（只剩注释）。留着它是因为它是挂钩位，将来要加"循环前干预"直接往里写就行，不用再动 `.pui`。

---

# 第 7 章 · 真 TCL 8.6 实测数据

用本机 **NX 自带解释器**跑你机器上这段 proc 的**真实原文**：

```
解释器 = C:\Program Files\Siemens\Designcenter2606\NXBIN\python\tcl86t.dll
patchlevel = 8.6.15
全文语法：info complete = 1   （括号 / 引号 / 方括号 全配平）
proc 体内是否还有 "proc" 关键字：无   ✓（"proc 头写两遍"已修掉）
```

## 7.1 你给的两种模式

| 场景 | 第 1 孔 | 第 2 孔 |
|---|---|---|
| 安全距离-最短距离 | **R15.5** ✓ | **R31.5152** ✓（与跑通那版一致，**零回归**） |
| 直接 | **R15.5** ✓（原来错成 `0.0`） | **R0.0** ✓ |

## 7.2 边界场景（8 项全过）

| 场景 | 构造 | 结果 | 期望 |
|---|---|---|---|
| A 安全距离 | `Z62.` → `G083 R46.5` → `Z62.` → `G083 R30.4848` | `15.5` / `31.5152` | ✓ |
| B 直接 | …`G000 Z46.5`…`G000 X84.9138 Z30.4848` → `G083 R30.4848` | `15.5` / `0.0` | ✓ |
| C 攻丝（先快移到 R 面） | `G000 Z46.5` → `G084 R46.5` | `0.0` | ✓ |
| D 干扰行 | 混入 `G053`/`G091`/`G030`/`G004`/`G68.5` | `15.5` | ✓ 不污染 |
| E 起始 Z 在 R 点之下 | `G000 Z20.0` → `G083 R46.5` | `0.0` | ✓ 夹零 |
| F 完全没有跟踪 | 直接 `G083 R46.5` | 不崩，`0.0` | ✓ |
| G 坐标宏 ON + R | 开 `::nc_addvar_on` | X/Y 包上 `[..+#..]`，R 仍 `15.5` | ✓ 共存 |
| H 连续三孔 | 每孔 R 点不同 | 逐孔刷新 `12.0` / `22.0` | ✓ |

## 7.3 首次调用（历史上真会崩的那个场景）

```
第一次调用后 ::nc_cycR_z = 62.
R = 15.5   （62 − 46.5）
```

最初 `.tcl` 曾被粘成 **"proc 套 proc"**（旧 proc 头没删，第二个 `proc` 成了第一个的正文）：
TCL 只在**外层 body 执行时**才真正重定义自己，于是**第一次调用整段正文一行不跑** → `::nc_cycR` 未定义 → `.def` 抛 `no such variable`。
**现在 proc 头只剩 1 处（tcl:5931），此隐患已消除。**

## 7.4 `expr` 的输出形态

```
expr {double(62.0) - double(46.5)}    =>  15.5
expr {double(62.0) - double(30.4848)} =>  31.5152
expr {double(30.4848) - double(30.4848)} => 0.0
```

**没有浮点尾巴**，NC 写法与你现在跑通的逐字一致。

---

# 第 8 章 · 操作步骤

## 8.1 改（若将来重建后处理）

1. **`.tcl`**：在 Post Builder 里打开 **`PB_CMD_before_output`** 定制命令，用 4.1 的原文整体替换。
   - **自检**：正文第一行应是 `global mom_o_buffer`；框里**不能再出现 `proc ` 字样**。
2. **`.tcl`**：`PB_CMD_custom_command_11` 里加第 6.1 的两行。
3. **`.def`**：21 处 R 单元改成 `R[$nc_cycR]`（保留 `\opt`）。
4. **最后**跑一次 `Apply-CycleR2.ps1`（纯文本替换、幂等、自动备份、带自检）。
5. **Save**。

> **顺序要点**：`.pui` 只是母本，PB 的 Save **不会**重生成 `.def`（`.pui` 里没有 `BLOCK_TEMPLATE`），所以先改 `.tcl` / `.def` 再 Save 都行；但**脚本始终放在最后跑**，避免 PB 的编辑动作把 `.def` 拉回出厂。

## 8.2 验收（必做）

```
① 出一个"安全距离"程序  → 应仍是 R15.5 / R31.5152（零回归）
② 出一个"直接"程序      → 应为 R15.5 / R0.0
③ 随手挑一个 G8x：
   Z(G8x 前最后一个带 Z 的移动) − R(该 G8x)  =  该循环 R 点的绝对 Z
   若算出负数 → 刀位跟踪被污染 → 停，回第 4 章 (3)(4) 排查
```

---

# 第 9 章 · 维护与跨机注意事项

## 9.1 换电脑最容易漏的一条 🔴

`.def` 第 4 行：

```
INCLUDE { $UGII_CAM_USER_DEF_EVENT_DIR/ude.cdl }
```

它读的是 **NX 全局**那份 `ude.cdl`，**不是**后处理目录里那份（你桌面上的 `mill_intergrex_300_2022.cdl` 是另外一回事）。

**需求③ 的 `add_variable` 事件必须在目标机器的全局 `ude.cdl` 里也存在**，否则 UDE 根本选不到、坐标宏静默失效。

**核查法**：在目标机执行 `后处理 → 输出 UDE`，看自定义事件是否出现在列表里。

## 9.2 四个文件 + `ude.cdl` 必须同步

| 文件 | 承载什么 | 不同步的后果 |
|---|---|---|
| `.tcl` | A 段 R 逻辑 + B 段坐标宏 + 兜底 | R 全错 / 坐标宏失效 |
| `.def` | 21 处 `R[$nc_cycR]` + `ADDRESS R` 格式 | R 出绝对值或负值 |
| `.cdl` | 事件定义（含需求③ 的 `add_variable`） | UDE 选不到 |
| `.pui` | 事件顺序 / 地址表 / MOM 变量 | 挂钩丢失 |
| `ude.cdl`（NX 全局） | 自定义事件注册 | 同上（且最难发现） |

## 9.3 三处可选加固（状态：第 1 条已落地）

| # | 内容 | 状态 |
|---|---|---|
| 1 | A 段开头兜底 `if { ![info exists ::nc_cycR] } { set ::nc_cycR 0.0 }` | ✅ **已在 tcl:5947** |
| 2 | 把"循环前没有任何带 Z 的块"从**静默输出 R0.0** 改成 `MOM_abort` 报错 | ⬜ 未做（与第 1 条二选一，**建议保持现状**：静默出 `0.0`，机床上表现为"刀已站在 R 面"，不会撞） |
| 3 | `PB_CMD_custom_command_15` 留作循环前干预的挂钩位 | ✅ 空 proc 保留 |

## 9.4 记住这几条"不要再踩"

1. **不要**在 TCL 里重写整行 R 文本 —— 格式得手工复刻，一错就撞刀。**值走 TCL、格式走 `.def`**。
2. **不要**用 `mom_cycle_retract_to_pos − mom_cycle_rapid_to_pos` —— 直接模式下偏大，R 面会被推到孔底以下（**撞刀**）。
3. **不要**让跟踪吞下 `G8x` 行的 Z（那是孔底深度）和 `G68.x` 的 Z（那是旋转中心）。
4. **不要**把 A 段放到 B 段的 `return` 之后 —— 没开坐标宏的工序就不跟踪刀位了。
5. **不要**忘记 `double()` —— TCL `expr` 会把 `08` 当八进制报错。
6. **不要**在 PB 里粘代码时留下第二行 `proc` 头 —— 第一次调用会整段失效。
7. **不要**忘记 `MIN` 不归零 —— 负 R 必须 TCL 夹。

---

# 第 10 章 · 附录

## 10.1 变量速查

| 变量 | 含义 | 本机实测值 |
|---|---|---|
| `mom_cycle_rapid_to_pos(2)` | **R 点绝对 Z** | 孔1 `46.5`；孔2 `30.4848` |
| `mom_cycle_retract_to_pos(2)` | 退刀平面绝对 Z | `62.` |
| `mom_cycle_rapid_to` | 孔顶到 R 点的**距离**（NX 语义，不是我们要的） | 变量本身 |
| `mom_cycle_spindle_axis` | 循环沿哪个轴 | `2`（Z） |
| `mom_pos` / `mom_prev_pos` | 当前 / 上一刀位 | — |
| `mom_o_buffer` | 即将输出的块文本 | — |
| `mom_sys_leader(Z/X/Y)` | 字头 | `"Z"` / `"X"` / `"Y"` |
| `mom_sys_control_out` | 注释起始符 | `"("` |
| `::nc_cycR` | **本方案新增**：给 `.def` 读的 R 值 | `15.5` / `31.5152` / `0.0` |
| `::nc_cycR_z` | **本方案新增**：跟踪到的刀位 Z | `62.` / `46.5` / `30.4848` |
| `::nc_addvar_on` | B 段坐标宏总开关 | 需求③ 控制 |
| `::nc_x_var` / `::nc_y_var` | 坐标宏目标变量号（覆盖刀号推算） | 默认 `600+n*5-4/-3` |

## 10.2 `.def` 表达式为什么能读 TCL 全局（本方案的唯一前提，已证实）

同一个 `.def` 里早就这么干了：

```
def:1849   rotate_r[$dpp_ge(coord_rot_angle,1)]
```

`dpp_ge` 是 **`.tcl` 里赋值的全局数组**（`set dpp_ge(coord_rot_angle,0/1)`），而它出现在 `.def` 的表达式里，并且**你 NC 里 `G68.5 … R90.` 的 `90` 就是它**。

⇒ `.def` 的表达式求值器**能读 `.tcl` 设的全局变量**。所以 `R[$nc_cycR]` 这条路是通的。

## 10.3 相关交付物索引（都在桌面）

| 文件 | 内容 |
|---|---|
| `后处理改造_修订12_循环R值_防忘总纲.md` | **本文**（终版总纲） |
| `后处理改造_修订11_循环R值_原理与逐行注释.md` | 上一版逐行注释（本文的前身） |
| `后处理改造_修订11_复核记录_19时27分版.md` | 19:27 版的复核记录（含 3 处加固的精确代码） |
| `后处理改造_修订10_循环R值_起始位置版.md` | 方案定案（含撞刀警告） |
| `后处理改造_修订8_钻孔R值_实测定值.md` | 早期公式（**已被修订 12 否掉，仅存档**） |
| `后处理改造_修订9_循环R值_PB原生改法.md` | 早期 PB 原生改法（同上，仅存档） |
| `Apply-CycleR2.ps1` | `.def` 21 处 R 单元的幂等替换脚本 |

---

## 一页纸总结（打印贴在工位上）

```
┌──────────────────────────────────────────────────────────────┐
│  机床规则： R = 执行 G8x 时刀所在的 Z − 该循环 R 点 Z          │
│                                                              │
│  实现：  值在 .tcl 算（PB_CMD_before_output 的 A 段）          │
│          格式在 .def 算（R[$nc_cycR] + FORMAT Coordinate）     │
│                                                              │
│  跟踪要点：                                                    │
│    跟踪 —— 所有带 Z 的移动块                                   │
│    不跟踪 —— G8x 行（Z 是孔底深度）、G68/G53/G28/G30/G91…      │
│    先跟踪、后算 R；负数夹 0                                    │
│                                                              │
│  验收： 安全距离 → 15.5 / 31.5152                             │
│        直接     → 15.5 / 0.0                                  │
│        Z(前一个带 Z 的移动) − R 必须 = R 点绝对 Z，不得为负     │
└──────────────────────────────────────────────────────────────┘
```
