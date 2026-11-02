# NX 车间文档表头变量：输入与使用方法

> 适用环境：`192.168.95.102` / NX 2606（`D:\Program Files\Siemens\Designcenter2606`）
> 定制目录：`C:\NX_shopdoc`
> 文档类型：Tool List（刀具清单）HTML 车间文档

---

## 一、原理

表头不是手动打字填进去的。模板里写死了变量占位符 `${...}`，NX 在生成车间文档时会用实际值替换：

```
NX 属性  →  mom_attr_<作用域>_<标题>  →  模板 ${...}  →  HTML 单元格
```

所以正确做法是**在 NX 对应位置放好值**，而不是在生成的表格里改文字。

`--` 是引擎的占位符，表示该变量**没有取到值**。

---

## 二、表头变量对照

| 表格标签 | 模板变量 | 值从哪来 | 是否需要输入 |
|---|---|---|---|
| Part name: | `${mom_part_name}` | 部件名（prt 文件名） | 否，改文件名即变 |
| **Drawing name:** | `${mom_attr_PART_DRWNAME}` | **部件属性** `DRWNAME` | **是** |
| Unit: | `${mom_Output Unit}` | 车间文档输出单位设置 | 否 |
| **Part number:** | `${mom_attr_PART_PARTNUM}` | **部件属性** `PARTNUM` | **是** |
| **Description ：** | `${mom_attr_PROGRAMVIEW_DESC}` | **程序组属性** `DESC` | **是** |

### 变量命名规则

```
mom_attr_<作用域>_<属性标题>
```

| 作用域 | 属性挂在哪 |
|---|---|
| `PART` | 部件（`文件 → 属性`） |
| `PROGRAMVIEW` | 工序导航器的程序组 / 程序视图节点 |
| `OPER` | 单条工序 |
| `TOOL` | 刀具 |

表头只用到 `PART` 和 `PROGRAMVIEW`。

> 表格正文里的 `mom_tool_name`、`mom_tool_diameter`、`mom_operation_name` 等由引擎自动采集，**不需要也不能手填**。

---

## 三、输入步骤

### 3.1 Drawing name / Part number（部件属性）

1. 菜单 `文件 → 属性`，切到 **属性** 选项卡
2. 新建一条属性，类型选 **字符串**
3. 标题填 `DRWNAME`，值填图纸名称
4. 再建一条：标题 `PARTNUM`，值填零件号

### 3.2 Description（程序组属性）

1. 打开 **工序导航器**，切到 **程序顺序视图**
2. 选中程序组节点（`PROGRAM` / `NC_PROGRAM`）
3. 右键 `属性` → **属性** 选项卡
4. 新建字符串属性 `DESC`，值填描述

> 同一程序组下的所有工序共用这条描述；换程序组就要重新建。

### 3.3 生效条件

- 属性标题必须是**纯英文大写、不带空格**，建议标题和名称填写一致
- 属性值支持中文
- 改完**保存部件**，并**重新生成**车间文档（生成时实时读取，不重新生成不会更新）

---

## 四、多行文字怎么写

NX 的属性输入框是单行的，换行会被吞掉，因此约定：

**一个值 + 竖线分隔**

```
粗加工|留量0.3|精加工
```

脚本会把 `|`（含全角 `｜`）转换成 `<br>`，在单元格内换行。
分隔符定义在 `shopdoc_user.tcl` 的 `::SD_ATTR_br_sep`。

### 两条限制

| 现象 | 原因 | 对策 |
|---|---|---|
| 回车/粘贴的换行丢失 | NX 在属性字段层就把换行去掉了 | 用 `\|` 分隔 |
| 多值属性只显示最后一条 | 引擎只转发最后一个值 | 不要用"允许多值"做多行 |
| 文字太多被截断 | 表格行高在模板里写死（`height=`） | 改模板行高，或精简文字 |

单元格已设为 `white-space:normal`，Excel 里会自动换行。

---

## 五、常见问题排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 显示 `--` | 属性没建 / 标题拼错 / 作用域错（如 DESC 建到部件上）/ 没重新生成 | 对照第二节核对标题与作用域，重新生成 |
| 只显示一部分文字 | 粘贴的换行被吞 / 用了多值属性 | 改用一个值 + `\|` |
| 中文变乱码 | 属性值编码问题 | 启用 probe 查看实际字符（见第六节） |
| 表头有值但表格是空的 | 生成时未选中工序，或模板选错（audit / shop floor） | 检查车间文档对话框的模板与选择范围 |

---

## 六、日志与诊断

| 文件 | 内容 |
|---|---|
| `C:\NX_shopdoc\sd_attr.log` | 属性发现与合并记录（常规） |
| `C:\NX_shopdoc\sd_format.log` | 版式选择器（audit / shop floor）记录 |
| `C:\NX_shopdoc\sd_cutting.log` | 切削参数采集 |
| `C:\NX_shopdoc\sd_stop.log` | 停机/辅助时间 |
| `C:\NX_shopdoc\sd_tool_vars.log` | 刀具变量 |
| `C:\NX_shopdoc\sd_attr_probe.log` | **需手动开启**，见下 |

### 开启深度排查

编辑 `C:\NX_shopdoc\shopdoc_user.tcl`，把：

```tcl
if { ![info exists ::SD_ATTR_probe_on] } { set ::SD_ATTR_probe_on 0 }
```

改为 `1`，然后重新生成一次车间文档。NX 实际发布的所有 `mom_attr_*` 变量（含作用域前缀、中文 `\uXXXX` 转义）会全部写入 `sd_attr_probe.log`，可据此确认真实变量名。

排查完记得改回 `0`。

---

## 七、改哪里，影响谁

| 改动位置 | 影响范围 |
|---|---|
| NX 里加/改属性 | 只影响当前零件 |
| 改 HTML 模板里的 `${...}` | 影响**所有**零件；写死文字则所有零件一样 |
| 改 `shopdoc_user.tcl` | 影响全局行为（换行、列宽、时间统计等） |

---

## 八、相关文件清单

### 定制目录 `C:\NX_shopdoc`

| 文件 | 说明 |
|---|---|
| `shopdoc_user.tcl` | 主定制脚本（属性换行、列宽、切削参数、时间统计等） |
| `shopdoc_user_post.tcl` / `.def` | 后处理挂钩 |
| `shopdoc_template_English_tool_list_shopfloor.htm` | shop floor 版式模板（15 列） |
| `NXShopDocAsk.exe` | 生成时弹出的版式选择器（0=audit，1=shop floor，2=取消），**不是**填表头的地方 |
| `shopdoc2xlsx.bat` / `.ps1` | 一键把生成的 `.htm` 转成 `.xlsx` |

### NX 安装目录

| 路径 | 说明 |
|---|---|
| `...\MACH\resource\shop_doc\shopdoc_template_English_tool_list_select.htm` | audit 版式模板（19 列） |
| `...\MACH\resource\shop_doc\excel_templates\shopdoc_excel.tcl` | 车间文档引擎 |

---

## 九、附：导出 Excel

NX 的"Excel 输出"实际写出的仍是 `.htm`（Excel 网页格式，含 mso 样式与 VML 图片）。因此：

1. 用 Excel 直接打开生成的 `.htm` → 另存为 `.xlsx`
2. 或使用 `C:\NX_shopdoc\shopdoc2xlsx.bat`：
   - 双击 → 转换同目录下所有 `*.htm`（自动跳过 `shopdoc_template*`）
   - 或把生成的 `.htm` 拖到 bat 上，只转那一个

实测保真：合并单元格、边框、列宽、图片、表头位置均保留。

**注意**

- `.xlsx` 是静态快照，改 NX 后需重新生成 + 重新转换
- 不要改用 NX 自带的 `shopdoc_template_English.xlsx` 等模板，会丢失全部定制（列数、切削参数列、`|` 换行、属性表头）
- 若转换失败并提示"受保护的视图"，把 `C:\NX_shopdoc` 加入 Excel 的**受信任位置**
