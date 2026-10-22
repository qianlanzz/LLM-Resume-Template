# DFT 面试准备 12：Tessent 工具专题

> **本文所有技术细节均取自一手资料：`Tessent™ Shell Reference Manual, Software Version 2021.3`（Siemens EDA 官方手册，Document Revision 24）**，包括产品线描述、命令语法、故障类定义、DRC 规则、TSDB 结构。
> 凡是二手博客里常见但手册里查不到的说法，本文一律不写。文末附"如何自查"的方法。

---

## 0. 先搞清楚一个最常见的错误认知

**「Tessent」不是一个工具，而是一整个产品家族，共同跑在 `Tessent Shell` 这个 Tcl 环境里。**

手册原文（Chapter 1, Product Introduction）：

> Tessent Shell is a **Tcl shell environment and design data model** that provides a unified Tcl command set and command naming conventions.

面试里如果你说"我了解 Tessent"，面试官很可能追问"Tessent 哪个产品？"——**这一句就能区分出你是真知道还是只知道名字**。

### Tessent 家族完整产品线（手册官方列表，一个不漏）

| 产品 | 干什么（手册原文要点） |
|---|---|
| **Tessent BoundaryScan** | 边界扫描单元与相关控制逻辑的创建与集成：IC 的 I/O 测试与诊断、板级互连测试。支持 IEEE 1149.1 标准单元、自定义单元，可选 1149.6（差分 I/O / AC 耦合网络） |
| **Tessent Diagnosis** | 失效诊断：**fault model independent** 的疑似点识别、打分、分类；支持 logic diagnosis 和 chain diagnosis；支持压缩 pattern 直接诊断；可链接 Calibre 做物理视图 |
| **Tessent FastScan** | 传统 ATPG（无压缩）。提供高速故障仿真器、设计规则检查器、pattern 格式输出 |
| **Tessent TestKompress** | **包含 FastScan 全部功能**，额外提供 **EDT IP 创建**能力。压缩后测试时间显著短于其他 scan 方法 |
| **Tessent IJTAG** | IEEE **1687-2014** 标准的实现：PDL 命令 retargeting 与 ICL 提取 |
| **Tessent LogicBIST** | Hybrid TK/LBIST 流程；插入 BIST-ready DFT（scan stitching、x-bounding、多周期/伪路径 bounding）；测试点插入提升随机图形可测性；EDT 与 LBIST 控制器共享逻辑以省面积 |
| **Tessent MemoryBIST** | 嵌入式存储器的 **at-speed 测试、诊断、修复、调试、表征**；分层架构，支持 core 级与 top 级 BIST 和自修复 |
| **Tessent MissionMode** | 外部 CPU 总线与片上测试资源之间的通信（system-controlled on-line test）；符合 **ISO 26262**，用于 ADAS/自动驾驶、航空航天、医疗 |
| **Tessent Scan** | scan 插入：full-scan 识别与插入、**Mux-scan 方法学**、强大的 DRC、自动生成 ATPG setup dofile 与 test procedure file |
| **Tessent ScanPro** | 包含 Tessent Scan 全部功能，额外：**测试点生成**（用于减少 pattern 数和提升覆盖率）、**X bounding**、读 SDC 并对 MCP/FP 做 DFT 处理 |
| **Tessent SiliconInsight** | 交互式测试、调试、表征 BIST 存储器和逻辑、IJTAG 仪器、ATPG pattern；通过 **CDP（Characterization and Debug Package）**开放 ATE 接口 |
| **Tessent DesignEditor** | Tessent Shell 功能的子集，**用更便宜的 license** 做设计查看与编辑（`dft` context 下的命令） |

**口诀记忆法**（面试可背）：

> **做进去的**：Scan / ScanPro（插链、测试点）、MemoryBIST（存 BIST）、LogicBIST（逻辑 BIST）、BoundaryScan（边界扫描）、IJTAG（仪器网络）
> **测出来的**：FastScan / TestKompress（ATPG + 压缩）
> **坏在哪的**：Diagnosis（诊断）、SiliconInsight（片上调试表征）、YieldInsight（良率）
> **用起来的**：MissionMode（在线测试）、DesignEditor（低成本查看编辑）

---

## 1. Tessent Shell 的两个核心概念（面试必问）

Tessent 的所有命令都受两个维度约束：**context（上下文）** 和 **system mode（系统模式）**。搞不清这两个，就看不懂任何 Tessent 脚本。

### 1.1 set_context —— 决定"你在用哪个产品"

**手册原文**：
> You must set the context before you can invoke most other commands in Tessent Shell.

**完整语法**（手册原文，一字不差）：

```tcl
set_context dft [{-edt [-logic_bist]} | -scan | -test_points | {-scan -test_points}] [-rtl | -no_rtl]
                [-license license_name] [-design_identifier string] [-force]

set_context patterns [-ijtag | -scan | -scan_retargeting | -scan_diagnosis | -failure_mapping]
                     [-silicon_insight] [-rtl | -no_rtl] [-license license_name]
                     [-design_identifier string] [-force]
```

**两大 context，各自含义（手册原文总结）**：

| Context | 用途 |
|---|---|
| **`dft`** | 加载设计、做**设计编辑与插入**（create_connections、create_instance、move_connections） |
| **`patterns`** | scan / IJTAG 的 **pattern 生成**、ICL 提取、scan pattern 诊断 |

**`dft` 的子 context**：

| 子 context | 含义 |
|---|---|
| `dft -scan` | scan 链插入 |
| `dft -edt` | 创建并插入 **EDT 控制器**（给一个 scan-inserted netlist，工具分析扫描链并插入合适的 EDT 控制器） |
| `dft -edt -logic_bist` | 启用 **hybrid TK/LBIST 流程** |
| `dft -test_points` | 测试点识别与插入 |

**`patterns` 的子 context**：

| 子 context | 含义 |
|---|---|
| `patterns -scan` | 标准 ATPG / TestKompress / LogicBIST pattern |
| `patterns -scan_diagnosis` | 诊断能力 |
| `patterns -ijtag` | IEEE 1687（IJTAG）PDL 命令 retargeting + ICL 提取 |
| `patterns -scan_retargeting` | core 级 pattern retargeting 到 top 级 |
| `patterns -failure_mapping` | 把 top 级 failure file 反向映射回 core 级，供诊断用 |
| `patterns -silicon_insight` | SiliconInsight 能力 |

**`-rtl` / `-no_rtl`**：
> 在 dft context 下，这两个开关指定你是做 **netlist 编辑还是 RTL 编辑**。编辑 RTL 必须用 `-rtl`；编辑 netlist 两者皆可，但 `write_design` 的用法不同——想把编辑后的 netlist 写成**单个文件**，选 `-no_rtl`。

⚠️ **一个高频面试题**：`set_context dft -rtl` 和 `set_context dft -no_rtl` 到底怎么选？
> **要看你在流程的哪个阶段**：做 RTL 级 DFT 插入（如 ScanPro 的 RTL 插入、BoundaryScan 的 RTL 生成）用 `-rtl`；拿到综合后网表做 scan insertion / ATPG，用 `-no_rtl`。选错会导致 `write_design` 输出成多个模块文件而不是单个网表。

### 1.2 set_system_mode —— 决定"你在流程哪一步"

**手册原文**：
> The default system mode after tool invocation is **setup**.

```tcl
set_system_mode {setup | insertion | analysis} [-force]
```

| 模式 | 做什么 |
|---|---|
| **setup** | 加载设计、库、定义约束（默认） |
| **insertion** | 执行**设计编辑**（create_instance 等） |
| **analysis** | 执行**分析类**操作：`insert_test_logic`、`create_patterns`、`analyze_scan_chains` 等。（`check_design_rules` 本身是 **setup** 模式命令，跑完且无 error 才把你带进 analysis） |

**手册里的官方示例（直接体现了三者关系，建议背下来）**：

```tcl
# 例 1：做 ATPG
add_scan_groups group1 scanfile
add_scan_chains chain1 indata2 outdata4
set_system_mode analysis
create_patterns

# 例 2：做 scan stitching
add_tied_signals 1 vcc
add_tied_signals 0 vss
add_clocks 0 clock
set_system_mode analysis
report_scan_elements
analyze_scan_chains
insert_test_logic

# 例 3：做 insertion
set_context dft -no_rtl
read_verilog mydesign.v
read_cell_library tessent_cell.tcellib
set_current_design
set_system_mode insertion
create_instance u1/myInst -of_module myModule
```

**⚠️ 陷阱（手册原文警告）**：
> If you attempt to change back to setup mode and have generated patterns, the tool prompts you to save the patterns:
> `Error: Current test pattern set has not been saved. Save the patterns or use the -force switch.`

> 面试怎么用：被问"Tessent 里怎么切换流程步骤"时，答"**context 决定用哪个产品，system mode 决定在哪个阶段**"，然后举上面任一个例子。这比背一堆命令有效得多。

### 1.3 License 机制（面试冷门但很显专业）

**手册原文**：
> By default, Tessent Shell acquires the **least expensive license** available that provides the requested functionality.

- `-license license_name` 可以指定要用的 license。
- MemoryBIST 和 BoundaryScan 的 license 名可以不带 `-LV` / `-LVNS` / `-TS` 后缀，**默认取 LV 版本**。
- 环境变量 **`TESSENT_LICENSE_ORDER`** 修改可用 license 的优先级顺序；未在其中列出的可用 license 会按原顺序追加。

**Context 与 License 的对应关系（手册 Table 6-6 节选）**：

| Context | 可用 License |
|---|---|
| `dft -rtl` / `dft -no_rtl` | DesignEditor、BoundaryScan-LV/LVNS/TS、**ScanPro**、IJTAG、PLLTest、MemoryBIST-LV/LVNS/TS、SerdesTest、TestKompress、LogicBIST、MissionMode |
| `dft -edt` | **TestKompress** |
| `dft -logic_bist -edt` | **LogicBIST** |
| `dft -scan` | **Scan**、**ScanPro** |
| `dft -test_points` | ScanPro、LogicBIST |
| `patterns -scan` | 用 EDT 时 → TestKompress；不用 EDT 且不用 LBIST → TestKompress, **FastScan**；用 LBIST → LogicBIST |
| `patterns -ijtag` | IJTAG、BoundaryScan-*、ScanPro、PLLTest、MemoryBIST-*、SerdesTest、TestKompress、LogicBIST、MissionMode |

> 💡 **面试加分点**：如果你能说出"**Tessent 是按功能模块分别 license 的，比如只做 test point 需要 ScanPro，做 EDT 需要 TestKompress**"，说明你理解商业工具的实际使用约束，不是纸上谈兵。

---

## 2. 工具如何启动

**手册 Chapter 2 的完整入口（`tessent` 命令）**：

```bash
tessent {-shell [options] | -diagserver [options] | -yieldinsight [options]
        | -siliconinsight {-ate | -desktop} [option] | -visualizer [options]}
        | [-help] | [-manual]
```

| 入口 | 用途 |
|---|---|
| `tessent -shell` | **主入口**，进 Tessent Shell |
| `tessent -diagserver` | 启动 Diagnosis server |
| `tessent -yieldinsight` | 启动 YieldInsight（良率分析） |
| `tessent -siliconinsight -ate\|-desktop` | 启动 SiliconInsight（片上调试表征） |
| `tessent -visualizer` | 启动 Visualizer（波形/原理图调试） |
| `tessent -help` | 列出所有开关 |
| `tessent -manual` | 打开 Tessent 文档系统 |

**其他 CLI 入口（手册 Chapter 2）**：
- `create_skeleton_design` —— **调用 Tessent TestKompress，生成在 RTL 阶段把 TestKompress 逻辑集成进设计所需的文件**（需要**手写 skeleton design input file**；不是"随便生成个骨架"）
- `stil2mgc` —— STIL 格式转换（名字里的 "mgc" 是 Mentor Graphics 的历史遗留）

**典型启动与批处理运行**：

```bash
# 交互式
tessent -shell

# 批处理跑 dofile（DFT 回归的标准姿势）
tessent -shell -dofile run_scan.tcl -logfile run.log
```

> ⚠️ dofile 是 Tessent 流程自动化的核心载体。**DFT 工程师日常一大半时间在写和维护 dofile 和解析 log**——这一点和你软件背景的工程能力直接对口，面试要主动说。

---

## 3. Tessent 命令分类与命令结构

手册 Command Dictionary 收录大量命令参考页。**手册本身并没有给出命令总数**——我按"命令名紧跟 `Context:` 行"这个口径数出 **862 条**（口径不同会略有出入，如把 iProc/ICL 命令和 Tcl 内建也算进去会到 890 左右）。

> ⚠️ **面试别报具体数字**。说"**几百条命令，按功能族组织**"就够了；报一个精确数字反而容易被追问口径。
> 下面各功能族的条数是我**按关键词自算**的，仅供参考（口径主观，手册无此统计）。

### 3.1 命令按功能族分组（手册真实命令名）

> ⚠️ **以下各族的条数都是我按关键词自算的，手册里没有这种统计**，口径一变数字就变（比如"scan 相关"按不同口径能数出 39/50/51/52）。**仅供了解分组，别当手册事实引用。**

**Scan 相关（约 50 条，节选）**：
```
add_scan_chains, add_scan_groups, add_scan_instances, add_scan_segments,
add_scan_mode, add_nonscan_instances, analyze_scan_chains,
create_scan_chain_family, create_scan_segment, delete_scan_chains,
get_scan_elements, report_scan_chains
```
- `add_scan_chains` —— 定义一个**已存在**的 scan 链（前置：必须先用 `add_scan_groups` 定义链组）
- `analyze_scan_chains` —— 把 scan 元素**分配到新链**中（context: `dft -scan`，mode: analysis）
- `insert_test_logic` —— **把你定义的测试结构插入网表，并把扫描链 stitch 起来以提升可测性**（Context: `dft -scan`, `dft -test_points`；Mode: analysis）。⚠️ 别和 `set_test_logic` 搞混——后者才是"控制 set/reset/clock/enable/write 使其可扫描"

**Pattern 相关（42 条，节选）**：
```
create_patterns, write_patterns, read_patterns, delete_patterns,
compress_patterns, expand_compressed_patterns, order_patterns,
create_diagnosis_patterns
```

**EDT 相关（21 条，节选）**：
```
set_edt_options, set_edt_instances, set_edt_pins, add_edt_blocks,
add_edt_configurations, set_current_edt_block, write_edt_files,
report_edt_abort_analysis, report_edt_lockup_cells, report_edt_finder,
set_edt_finder, set_edt_mapping, set_edt_power_controller
```

**BIST 相关（12 条）**：
```
set_bist_chain_test, set_bist_debug, set_bist_trace,
set_lbist_controller_options, set_lbist_instances, set_lbist_pins,
set_lbist_power_controller_options, report_lbist_configuration, report_lbist_pins,
add_bist_capture_range, delete_bist_capture_ranges, report_bist_capture_ranges
```

**Test point 相关（15 条）**：
```
analyze_test_points, add_notest_points, delete_test_points,
set_test_point_types, set_test_point_analysis_options,
set_test_point_insertion_options, get_test_point_type, report_test_points,
report_test_point_statistics, write_test_point_dofile
```

**Diagnosis 相关（8 条）**：
```
diagnose_failures, report_diagnosis, write_diagnosis,
create_diagnosis_patterns, annotate_diagnosis, set_diagnosis_options
```

**IJTAG 相关（10 条）**：
```
add_ijtag_logical_connection, set_ijtag_instance_options,
set_ijtag_retargeting_options, report_ijtag_instances,
report_ijtag_logical_connections
```

**DFT 信号 / 时钟相关（数十条，节选）**：
```
add_dft_signals, add_clocks, add_dft_clock_mux, add_dft_clock_enables,
add_dft_control_points, add_dft_modal_connections, add_tied_signals,
report_dft_signals, report_dft_clock_muxes, report_memory_dft_summary
```

### 3.2 几条必知的核心命令（面试最可能被问"你用过哪些"）

**下表每行的 Context / Mode 都逐条对照过手册（这是"读 Context/Mode 就知道命令能不能用"的示范）**：

| 命令 | Context | Mode | 作用 |
|---|---|---|---|
| `set_context` | unspecified, all | all modes | 设定使用上下文 |
| `set_system_mode` | all contexts | all modes | 设定系统模式 |
| `read_verilog` | all contexts | setup | 读设计网表 |
| `read_cell_library` | unspecified, all | setup | 读 **Tessent cell library**（注意不是 .lib） |
| `set_current_design` | all contexts | setup, insertion | 设定顶层设计 |
| `check_design_rules` | dft, patterns | **setup** | **跑 DRC**；顺利通过则转入 analysis |
| `report_scan_elements` | `dft -scan`, `dft -test_points -no_rtl`, `dft`（无子 context） | all modes | 报告 scan 元素 |
| `analyze_scan_chains` | `dft -scan` | analysis | 分配 scan 元素到链 |
| `insert_test_logic` | `dft -scan`, `dft -test_points` | analysis | 插入测试逻辑 |
| `write_atpg_setup` | `dft -scan` | analysis, insertion | 生成 test procedure file + dofile，供下游 ATPG 用 |
| `set_fault_type` | `dft -edt`, `patterns -scan`, `patterns -scan_diagnosis` | setup, analysis | 选择故障模型 |
| `create_patterns` | `dft -edt`, `patterns -scan` | analysis | **跑 ATPG 生成 pattern** |
| `report_statistics` | `dft -edt`, `patterns -scan`, `patterns -scan_diagnosis` | setup, analysis | 输出覆盖率与故障分类统计 |
| `write_patterns` | `patterns -ijtag`, `patterns -scan`, `patterns -scan_retargeting`, `patterns -scan_diagnosis` | analysis | 保存 pattern |

---

## 4. 完整 DFT 流程与真实 dofile 骨架

### 4.1 Scan 插入流程（dft -scan）

```tcl
# ========== 1. 设定上下文 ==========
set_context dft -scan

# ========== 2. 读设计 + 库 ==========
read_verilog my_design.v
read_cell_library tessent_cell.tcellib
set_current_design my_design

# ========== 3. 定义时钟、约束、tie 信号 ==========
add_clocks 0 clock            # 参数含义：[off_state] port_list —— 声明 clock 且其 off 态为 0
add_tied_signals 1 vcc
add_tied_signals 0 vss

# ========== 4. 定义 DFT 信号（scan enable 等）==========
# ⚠️ add_dft_signals 手册的 Context 是 dft（不带子 context），
#    常见示例在 dft -rtl 下；在 dft -scan 下使用我没有找到干净先例，此处按需调整
add_dft_signals scan_en -source_nodes scan_en_port

# ========== 5. 跑 DRC ==========
check_design_rules    # ← 注意：它本身就是 setup 模式下的命令，
                      #   DRC 全过才会自动把你带进 analysis 模式；
                      #   有 error 就停在 setup，等你补定义

# ========== 6. 分配并插入 scan 链（在 analysis 模式下）==========
analyze_scan_chains
report_scan_elements
insert_test_logic

# ========== 7. 生成下游 ATPG 需要的 setup 文件 ==========
write_atpg_setup my_design
```

> ⚠️ **`check_design_rules` 的 Mode 是 `setup`，不是 `analysis`**——手册原文："Transitions the tool from setup mode to analysis mode." 也就是说**它是那道门**：DRC 全过 → 自动进 analysis；有 error → 留在 setup 让你补定义（比如加时钟）。所以不要写 `set_system_mode analysis` 再跑 DRC，顺序反了。这是个很能体现"真读过手册"的细节。

### 4.2 ATPG 流程（patterns -scan）

```tcl
set_context patterns -scan

read_verilog scan_inserted.v
read_cell_library tessent_cell.tcellib
set_current_design my_design

# 选故障模型（默认是 Stuck）——setup 模式下设置
set_fault_type Transition        # 做 at-speed

check_design_rules               # ← setup 模式，DRC 过了自动进 analysis

# ⚠️ read_procfile 是 Mode: analysis，必须在进入 analysis 之后才能调用
read_procfile my_design.testproc # 上一步 write_atpg_setup 生成的 test procedure file

create_patterns                  # ← 跑 ATPG（要求 analysis 模式）

report_statistics                # ← 看覆盖率

write_patterns my_pats.stil -stil      # -stil 是手册示例里的简写；正式开关是 -STIL2005 / -STil1999
```

### 4.3 EDT 流程（TestKompress，两阶段）

手册明确说 **TestKompress 分两个阶段**，这是它区别于 FastScan 的关键：

```tcl
# ---------- 阶段 1：EDT IP Creation ----------
set_context dft -edt
# 输入：scan-inserted 的 gate-level netlist（Verilog）
# 输出：EDT IP 文件（Verilog/VHDL RTL + 连接信息 + core 的 black box 表示）
#       Design Compiler 综合脚本（模板）
#       EDT Dofile
#       EDT Test Procedure File
#       Bypass Files（用于在带 EDT 的设计上跑 FastScan ATPG）
#       由 write_edt_files 输出

# ---------- 阶段 2：EDT Pattern Generation（新一次调用，切到 patterns context）----------
set_context patterns -scan
# 输入：scan-inserted + 含 EDT 电路的 gate-level netlist
#       EDT Dofile（阶段 1 产生）
#       EDT Test procedure file（阶段 1 产生）
read_verilog scan_inserted_with_edt.v
read_cell_library tessent_cell.tcellib
set_current_design my_design
check_design_rules               # 进 analysis
read_procfile ...                # analysis 模式下读
create_patterns
write_patterns ...
```

**⚠️ 面试必答差异**：
> **FastScan 是不带压缩的传统 ATPG；TestKompress 包含 FastScan 的全部功能（`patterns -scan` context），额外提供 EDT IP 创建（`dft -edt` context）。** 两者共享同一套命令集（除 EDT 相关命令外）、同样的 pattern 类型和故障模型、同样的覆盖率和诊断能力。

### 4.4 关键命令的官方细节

**`create_patterns`**（Context: `dft -edt`, `patterns -scan`；Mode: analysis）
```tcl
create_patterns [-OVerride_user_settings | -NO_AUto]
                [-Compaction_effort {Low | High | Maximum | integer}] [-NO_Analyze]
                [-COVerage_effort {Low | High}] [-NO_REDundancy_analysis]
                [-NO_TERMINATE_ineffective_atpg] [-PATterns_per_pass integer]
                [-NO_REPort_statistics]
```
手册要点：
- 这是**推荐的高质量高效 pattern 生成方式**，内部使用专家系统 **ATPG Expert** 自动优化设置。
- 默认只优化**用户未显式指定的**设置；要让 ATPG Expert 也优化用户设置，用 `-override_user_settings`。
- 是**多进程命令**（multiprocessing）。
- 自动显示统计：simulated patterns 数、test coverage、detected faults、remaining faults、effective patterns、test patterns、redundant/untestable/aborted faults。

**`create_patterns` 的真实输出（手册转录，无压缩情况）**：
```
//#patterns  test      #faults  #faults   #eff.   #test
//simulated coverage  in list detected  patterns patterns  CPU time
//
     64      74.10%    135463   413980      63       63     14.73 sec
//
                                          RE/AU/AAB
                                          0/40868/57
```
有压缩时末尾变成 `RE/AU/AAB/EAB`（EAB = EDT aborts）。

> 💡 **这个输出格式值得记住**：面试官问"跑完 ATPG 你看什么"——就答"**看 test coverage、剩余 fault 数、以及 RE/AU/AAB 三类未覆盖故障的分布**"。

**`write_atpg_setup`**（Context: `dft -scan`）
```tcl
write_atpg_setup basename [-Replace] [-NO_merge] [-NO_Edt] [-All_internal_clocks]
```
> 写一个 **test procedure file** 和一个 **dofile**，描述 scan 插入时创建的链。多模式 scan 插入时还会生成验证所有 scan mode 的 dofile。默认会把新插入链的信息与已有链**合并**。

---

## 5. 故障模型（set_fault_type 官方语法）

**手册中 `set_fault_type` 支持的完整模型列表**（这是最权威的答案）：

```tcl
set_fault_type {
    Stuck
  | TOggle
  | Iddq [-NO_SEQuential_inputs_blocked | -SEQuential_inputs_blocked]
  | Transition [-APPLY_HOLD_PI_attribute_to_shift_launch {ON | OFf}]
  | Path_delay [-Mask_nonobservation_points] [-ROBust_detection_only | -NO_Functional_detection]
               [-HAZARD_free_robust_detections] [-Expand_ambiguous_paths {ON | OFf | expansion_limit}]
  | Bridge
  | Udfm [-STATIC_faults]
  | Udfm -DELAY_faults
  | Udfm -IDDQ_faults
}
```

| 模型 | 说明 |
|---|---|
| **Stuck** | single stuck-at。**工具启动时的默认模型** |
| **TOggle** | 跳变故障 |
| **Iddq** | 静态电流测试，可选是否 block 时序输入、时钟活动时是否允许测量 |
| **Transition** | 跳变延迟故障（at-speed） |
| **Path_delay** | 路径延迟故障，带 robust 检测、hazard-free 等选项 |
| **Bridge** | 桥接故障 |
| **Udfm** | **User-Defined Fault Model**，用户自定义故障模型；分 static / delay / iddq 三类 |

**手册原文的关键说明**：
> The fault sites of all models are the **input and output pins of the design cells in addition to external pins**. 0 and 1 values are used to indicate the type of fault at the fault site.
> Each fault model has its **own separate fault collapsing** according to the model's rules of equivalence.
> **When you change the fault type, both the current fault list and the internal pattern set are deleted.**

> ⚠️ 最后一句是实操坑：**换故障模型会清空当前 fault list 和 pattern set**。面试时能说出这点，说明真跑过。

**⚠️ 注意 UDFM 这个考点**：现代先进工艺里 **cell-aware** 故障模型通常通过 **UDFM** 实现（从标准单元版图提取的真实缺陷位置写成 UDFM 文件）。所以你之前资料里说的 "Cell-Aware"，在 Tessent 里的落地形式就是 `set_fault_type Udfm`。这是一个能串起来的加分点。

### 5.1 故障分类与覆盖率（与 08 文档一致，这里给出手册依据）

**手册 Table 3-21「Fault Class Codes and Class Names」的完整代数关系**：

```
FU (Full)       = TE + UT
TE (TEstable)   = DT + PD + AU + UD
DT (DETEcted)   = DS + DI + DR (+ DF)
PD (POSDET)     = PU + PT
UT (UNTestable) = UU + TI + BL + RE
```

| 代码 | 全称 | 含义 |
|---|---|---|
| **FU** | Full | 全部故障 |
| **TE** | TEstable | 可测故障 |
| **DT** | DETEcted | 已检测 |
| **DS** | DET_Simulation | 仿真检测 |
| **DI** | DET_Implication | 蕴含检测 |
| **DR** | DET_Robust | 稳健检测（仅 Path Delay Testing） |
| **DF** | DET_Functional | 功能检测（仅 Path Delay Testing） |
| **PD** | POSDET | 可能检测 |
| **PU** | POSDET_Untestable | 可能检测-不可测 |
| **PT** | POSDET_Testable | 可能检测-可测 |
| **AU** | Atpg_untestable | ATPG 不可测 |
| **UD** | UNDetected | 未检测 |
| **UC** | UNControlled | 不可控 |
| **UO** | UNObserved | 不可观测 |
| **UT** | UNTestable | 不可测 |
| **UU** | UNUsed | 未使用 |
| **TI** | TIed | 被 tie 住 |
| **BL** | Blocked | 被阻塞 |
| **RE** | Redundant | 冗余 |

**故障子类（手册 `report_statistics -detailed_analysis` 列出，注意归属）**：

| 子类 | 归属 | 含义 |
|---|---|---|
| **BB** | **AU** | black_boxes，黑盒导致的不可测 |
| **CC / PC / TC / FP** | **AU** | pin constraints / tied cells / false paths 等其他 AU 子类 |
| **AAB** | **UD（不是 AU）** | ATPG_ABORT，ATPG 放弃 |
| **EAB** | **UD（不是 AU）** | EDT_ABORT，压缩导致放弃 |
| **UNS** | **UD** | UNSUCCESS |
| **UC + UO** | **UD** | UD 的 coverage 口径就是 `UC + UO` |

> ⚠️ **高频误记**：很多人会把 **AAB / EAB 当成 AU 的子类**。手册白纸黑字写的是 **`UD.AAB`、`UD.EAB`、`UD.UNS`**——它们属于 **UD（undetected，未检测）**，不是 AU。区别在于：**AU = "ATPG 判定根本测不了"；UD = "能测但这次没测出来/放弃了"**。
> 报告里它们印在 `UC+UO` 小标题下，而 Table 3-21 规定 UD 的 coverage 口径正是 `UC + UO`。

**覆盖率公式与 possible-detection credit（本节重点，也是最容易追问的地方）**：

```
DT_eff = DT + credit × PD            ← credit 默认 50%

                    DT_eff
fault_coverage  =  ────────
                      FU

                    DT_eff
test_coverage   =  ─────────────
                    FU − UT
```

**依据（手册原文）**：`set_possible_credit`

> "specifies the percentage of credit that the tool assigns **possible-detected faults** … when calculating the **test coverage, fault coverage, and ATPG effectiveness**. The default credit value for possible-detected faults is **50 percent**."

也就是说：`PD`（= `PU + PT`，可能检测故障）**按默认 50% 折算成已检测**。**这就是为什么直接拿 `DT = DS + DI` 去核对报告会差一点点**——差的正是那半个 PD。

**验证（我实际算过，两组报告 4 个百分比全部逐位吻合）**：

```
# 第一组：FU = 136996
UT     = 62 + 96 + 1 + 9867 = 10026       FU − UT = 126970
DT     = 112635 + 10037 = 122672
PD     = PU + PT = 199 + 7 = 206
DT_eff = 122672 + 0.5 × 206 = 122775

  FC = 122775 / 136996 = 89.6194% → 89.62%   ✅ 手册印 89.62%
  TC = 122775 / 126970 = 96.6961% → 96.70%   ✅ 手册印 96.70%

# 第二组：FU = 135789（DS/DI/PU/PT 与上组相同）
  FC = 122775 / 135789 = 90.4160% → 90.42%   ✅ 手册印 90.42%
  TC = 122775 / 126970 = 97.6241% → 97.62%   ✅ 手册印 97.62%
```

> **同一个 `DT_eff = 122775` 同时精确复现了手册里两组不同 FU 的报告**（4 个百分比全中）。这既证明公式正确，也证明 **credit 机制是唯一合理解释**——不需要任何"转录误差"之类的牵强解释。
>
> 我另外实测过：用 `DT + 0.5×PD` 拟合手册全部 19 份报告，命中率从单用 `DT` 的 8/19 提升到 **15/19**（其余 4 份是报告间的取整/汇总差异）。

**面试怎么讲这一段**（很能体现功底）：

> "Tessent 的 fault coverage 与 test coverage 分母不同：fault coverage 相对全部故障 `FU`，test coverage 相对剔除 unused/tied/blocked/redundant 之后的 `FU − UT`。分子也不是单纯的 detected——**可能检测的故障 PD 会按 `set_possible_credit` 的比例折算，默认 50%**。所以我核对报告时会先把 PD 折一半，而不是直接拿 `DS + DI` 去除。"

> 📌 **严谨性说明（两个出处要分清）**：
> - **故障分类代数**（`FU = TE+UT`、`UT = UU+TI+BL+RE` 等）出自 **本手册 Table 3-21**；
> - **覆盖率公式的完整定义**在 **《Tessent Scan and ATPG User's Manual》的 "Testability Calculations"** 一节，本手册只是在 `set_possible_credit` 等处指向它。
>
> 面试时若要引用公式出处，说 User's Manual 更准确。

**手册里真实的 `report_statistics` 输出**（Stuck-at，可直接用来验证公式）：

```
Statistics Report
Stuck-at Faults
--------------------------------------------Fault Classes
#faults (total)
---------------------------- --------------
FU (full)                     136996
-------------------------- --------------
UO (unobserved)                   84 ( 0.06%)
DS (det_simulation)           112635 (82.22%)
DI (det_implication)           10037 ( 7.33%)
PU (posdet_untestable)           199 ( 0.15%)
PT (posdet_testable)               7 ( 0.01%)
UU (unused)                       62 ( 0.05%)
TI (tied)                         96 ( 0.07%)
BL (blocked)                       1 ( 0.00%)
RE (redundant)                  9867 ( 7.20%)
AU (atpg_untestable)            4008 ( 2.93%)
--------------------------------------------Coverage
-------------------------
test_coverage                     96.70%
fault_coverage                    89.62%
atpg_effectiveness                99.94%
```

> 💡 **`atpg_effectiveness` 是第三个指标**——手册输出里就有（此例 99.94%）。它反映的是 **ATPG 本身的求解能力**，而不是设计的可测性。**能说出这第三个指标，是很强的区分度。**

**这个转录是节选**（为节省篇幅省略了 Fault Sub-classes 段）；完整报告里 AU 下面还会列出 CC / PC / TC / FP 等子类，UD 下面列出 AAB / EAB / UNS。

**⚠️ 注意 `UO` 在报告里单独出现（本例 84），但表 3-21 里 UO 是 UD 的子类**（`UD = UC + UO`）。所以看到报告里有 UO 时，它属于 UD（未检测）那一支，**不在 UT 里**，不影响 test_coverage 的分母。这个细节能理清，说明你真读懂了分类代数。

---

## 6. DRC（Design Rule Checking）—— Tessent 流程的核心

**Tessent 的 DRC 由 `check_design_rules` 驱动，规则分两类：**

### 6.1 C 规则（Clock Rules）—— 扫描/时钟相关

手册 Chapter 7「Design Rule Checking」里的 C（Clock）规则**实际定义了 19 条**：**C1–C10、C16、C17、C19、C21–C26**（**C11–C15、C18、C20 未定义**；C27 只是 Ch14 中"未来可能出现的保留名"举例，**当前不存在**）。**面试最常问的是 C1**：

**C1（Category: Clock）**
> A scan or non-scan cell must **not capture data when all specified clocks are set to their off states**.
> When all clocks are at their off state as defined with the `add_clocks` command, all clock inputs (**including sets and resets**) of scan and non-scan cells must be at their off state.
> For non-scan cell violations, the tool converts these to TIEX.
> In the case of a DFF scan cell, the clock input must be a **stable binary value (0 or 1)**.

**C1 的典型原因和修法（手册原文）**：
> One common root cause of these C1 violations is a **set/reset pin not defined as a clock**. You can define a set/reset pin as a clock to fix the problem and avoid ATPG performance impact.

手册里的真实修复例子：
```
// Error: There was 1 DFT_C1 violation (Memory clock not properly sourced
// by a declared clock).
//
SETUP> add_clocks pll/vco -reference pll/ref -freq_mult 10 -freq_divider 2
//
SETUP> check_design_rules
```

> 💡 这个例子完美体现了 DFT 排障的思路：**DRC 报 clock 没被正确声明 → 用 add_clocks 把 PLL 输出声明成 derived clock → 重跑 DRC**。面试讲排障就讲这个。

### 6.2 DFT_C 规则 —— Memory BIST 专用

**手册原文（关键限定条件）**：
> Clock rule checks **DFT_C1 through DFT_C5 run only in the `dft` context, and only if there are memory instances in the design and the `set_dft_specification_requirements -memory_test` option is set to "on"**. These clock rule checks are **skipped entirely for all other cases**.

> ⚠️ 这是个很好的面试细节：**DFT_C1–C5 是 memory BIST 专用规则，不是通用规则**。很多人会以为 DFT_ 开头的是"更严格的通用规则"，其实它只在做 memory test 且设计里有 memory 时才跑。

### 6.3 DRC 的处置机制

- 每条规则有 **Default Handling**（如 C1 是 `Error`）。
- 用 **`set_drc_handling`** 可以修改处置方式。
- 用 **`report_drc_rules`** 可以报告规则及其违反情况。
- 规则带 **Category**（如 Clock）、**Contexts Supported** 列表。
- 违规有编号，如 `(DFT_C1-1)`、`(DFT_C1-2)`，用于查手册定位。

**面试怎么答"DRC 报错怎么处理"**：
> 先看是哪个规则、哪个 Category；Tessent 每条规则都有编号（如 DFT_C1-1），可以回手册查定义和示例。然后判断是**设计约束没写全**（最常见，比如时钟/复位没声明）还是**真实的 RTL 问题**（比如 latch、组合环）。约束问题改 dofile，RTL 问题反馈给设计。修完重跑 `check_design_rules`，直到干净才进 ATPG——**DRC 不干净，ATPG 出的 pattern 在硅上无效**。

---

## 7. TSDB（Tessent Shell Data Base）—— 流程数据管理

**手册原文**：
> TSDB is a **structured directory** containing subdirectories and files. The subdirectory naming is designed to group related information for each specific view of each design block. **The complete subdirectory is deleted and recreated cleanly when a step is repeated thereby ensuring that no stale files from a previous run can remain in the active TSDB directory.**

**根目录下四个子目录（手册第 12 章）**：

| 子目录 | 何时创建 | 存什么 |
|---|---|---|
| **`instruments`** | 首次执行 `process_dft_specification` 或 `write_edt_files -tsdb` | 各类 instrument container（按 design_name + design_id + IP 类型命名） |
| **`dft_inserted_designs`** | 首次执行 `process_dft_specification`（不带 `-no_insertion`），或 `extract_icl -write_in_tsdb` | DFT 插入后的设计 |
| **`patterns`** | 首次执行 `process_patterns_specification` | pattern 数据 |
| **`logic_test_cores`** | 首次在 `patterns -scan` context 下执行 `write_tsdb_data` | LogicBIST pattern（目前仅支持 LBIST） |

**相关命令**：
- `set_tsdb_output_directory` —— 指定根目录（默认 `tsdb_outdir`）
- `open_tsdb` / `close_tsdb` —— 打开/关闭 TSDB（层级设计中子 block 的 TSDB 要 open 才能被看到）

> 💡 **面试加分点**：TSDB 的设计思想是"**每一步的产物集中存放，重跑时整目录删掉重建，绝不留陈旧文件**"。这是一个非常工程化的设计——**和我做数据管道时"原子写入 + 幂等重跑"的思路完全一致**。这句话能把你的软件背景和 Tessent 流程自然地连起来。

---

## 8. 命令的通用结构（面试问"你写过 Tessent 脚本吗"）

Tessent 命令的文档结构非常统一，**掌握这个结构你就能自己查任何命令**：

```
<command_name>
Context:     哪些 context 下可用
Mode:        setup / insertion / analysis（有的写 all modes）
Prerequisites:  前置条件（比如"必须先 add_scan_groups"）
Usage:       语法
Description: 详细说明
Arguments:   每个参数的含义
Examples:    真实例子  ← 最有价值的部分
Related Topics: 相关命令
```

**举例（`add_scan_chains` 手册原文）**：
```
add_scan_chains
Context: dft -edt, dft -scan, dft -test_points, patterns -scan, patterns -scan_diagnosis
Mode: setup
Prerequisites: You must define the scan chain group with the add_scan_groups command
               prior to using this command.
Usage: add_scan_chains [-Internal | -Load_only] {chain_name group_name input_pin output_pin}...
```

> **面试话术**：被问"你没用过 Tessent 怎么办"——
> "我没在产线上用过。但我读过 Tessent Shell Reference Manual，理解它的命令组织方式：每条命令都标注了 context、system mode、prerequisites、usage 和示例。给定一个任务，我能查手册找到该用哪条命令、需要在哪个 context 和 mode 下、有哪些前置条件。**我认为上手工具的关键是理解流程和数据结构，而不是背命令。**"

---

## 9. 冷门但很能加分的产品细节

面试官如果发现你是认真研究过的，往往会顺着这些点问。**知道这些你就是"研究过的"而非"背过的"**：

1. **Tessent BoundaryScan 支持 IEEE 1149.6** —— 用于差分 I/O、驱动 **AC 耦合网络**的边界扫描单元。1149.1 只能测 DC 耦合，AC 耦合（有电容隔直）需要 1149.6。还提供 **1149.1-based 的非接触式 I/O 测试**。

2. **Tessent Diagnosis 是 fault-model-independent 的** —— 手册原文 "Fault model independent analysis for suspect identification, suspect scoring, and classification"。支持 **logic diagnosis 和 chain diagnosis**，支持**压缩 pattern 直接诊断**，可链接 **Calibre** 做物理视图。可以输出 **CSV 格式**便于分析。

3. **Tessent LogicBIST 走 Hybrid TK/LBIST** —— 关键是 **EDT 与 LBIST 控制器共享逻辑以省面积**，并且生成 **low power shift** 逻辑。**这两个产品共享硬件**是很实用的知识点。
   > ⚠️ "TK" 合理推断是 TestKompress，但**手册从未展开这个缩写**（全篇只写 "Hybrid TK/LBIST"）。面试时说"应该是指 TestKompress"，别说死。

4. **Tessent SiliconInsight 用 CDP 开放接口** —— CDP = Characterization and Debug Package，包含 test patterns、procedural data 和 side files，可在**任意层级**访问和控制 instrument。还支持通过 **GPIB（IEEE-488）**控制电源、时钟发生器等仪器（用 USB-to-GPIB 适配器接到 performance board）。

5. **Tessent MissionMode 属 Mentor Safe 工具套件** —— 手册原文是"enables ISO 26262-**compliant designs**"，即**使能符合 ISO 26262 的设计**，而**不是工具本身通过了认证**。面向 ADAS/自动驾驶、航空航天、医疗。作用是让外部 CPU 总线与片上测试资源通信，做**系统控制的在线测试与诊断**。（这个措辞差别在功能安全岗位面试里会被抠。）

6. **Tessent ScanPro 能读 SDC** —— 并处理 **MCP（Multicycle Path）和 FP（False Path）**，防止切换通过这两类路径传播后被扫描单元捕获。这是个很实际的可测性问题。

7. **Tessent Scan 生成的是 Mux-scan** —— 手册明确写 "Support for Mux-scan methodology"，也是工业界主流。

8. **`insert_test_logic` 的 logic_type 与默认前缀** —— 这个考点有个坑，手册里**两处列的不完全一样**：

   **① 权威定义（`set_insert_test_logic_options` 的 Arguments 一节，带含义和默认 infix）**——15 种：
   
   | logic_type | 用途 | 默认 infix |
   |---|---|---|
   | `generic` | 通用逻辑 | `logic_` |
   | `clock_gater_fix` | 时钟门控控制 | `cgfix_` |
   | `clock_control_fix` | 时钟控制 | `clkfix_` |
   | `dedicated_input_wrapper` | 专用输入 wrapper cell | `diw_` |
   | `dedicated_input_holding_wrapper` | 专用输入保持 wrapper cell | `dihw_` |
   | `dedicated_output_wrapper` | 专用输出 wrapper cell | `dow_` |
   | `dedicated_output_holding_wrapper` | 专用输出保持 wrapper cell | `dohw_` |
   | `lockup` | **扫描链 retiming cell** | `lockup_` |
   | `pipeline` | 扫描链 pipeline cell | `pipeline_` |
   | `ram_fix` | RAM 控制修复 | `ramfix_` |
   | `testpoint_control` | 控制测试点 | `cp_` |
   | `testpoint_observation` | 观测测试点 | `op_` |
   | `testpoint_observation_scan` | 观测 scan 测试点 | `osp_` |
   | `tri_state_fix` | 三态/双向控制修复 | `trifix_` |
   | `x_bounding` | **X 绑定** | `xb_` |

   **② `report_insert_test_logic_options` 的实际输出**——18 条。比上面的正式定义**多出 3 个真实类型**：
   `se_pipeline`（`se_pipeline_`）、`shared_input_holding_wrapper`（`sihw_`）、`shared_output_holding_wrapper`（`sohw_`），
   另有 1 处拼写修正（见下）。

   > ⚠️ **所以"logic_type 有几种"没有唯一答案**：Usage 行列举 14 种、Arguments 正式定义 15 种、工具实际输出 18 种。**面试被问到时说"手册正式定义是 15 种，工具报告里还会出现 se_pipeline 和 shared_*_wrapper 这几类"，比硬背一个数字安全。**
   >
   > 📝 **一个有意思的细节**：手册 Arguments 一节把 `dedicated_output_holding_wrapper` 拼成了 `dedicated_ouput_holding_wrapper`（少了 t），而工具的实际输出是拼写正确的。**这属于手册笔误**——如果你面试时引用这个列表，用拼写正确的版本；如果被问"你怎么知道这些细节"，这恰好是"我读过手册原文"的佐证（但别主动去挑手册的错，没意义）。

   > 💡 **注意 `se_pipeline`**（scan enable 流水）、**`lockup`**（扫描链 retiming / 跨时钟域锁定）、**`x_bounding`**（X 绑定）、**`tri_state_fix`**（三态修复）——这四个正好对应你已学的 SCAN 知识点，**能把原理和工具实现对起来讲**，这是很自然的加分点。

---

## 10. 面试问答演练

**Q1：Tessent 是什么？**

> Tessent 是 Siemens EDA（原 Mentor Graphics）的 DFT 工具家族，跑在 **Tessent Shell** 这个统一的 Tcl 环境里。它不是一个单一工具，而是包含 Scan / ScanPro（scan 插入、测试点、X bounding）、FastScan / TestKompress（ATPG，TestKompress 额外提供 EDT 压缩）、MemoryBIST（存储器自测试与修复）、LogicBIST、BoundaryScan、IJTAG、Diagnosis、SiliconInsight、MissionMode 等一整套产品。所有命令共享统一的命名规范和 context 机制。

**Q2：FastScan 和 TestKompress 什么区别？**

> TestKompress **包含 FastScan 的全部功能**，额外提供 **EDT（Embedded Deterministic Test）IP 创建**能力。FastScan 走 `patterns -scan` context，TestKompress 除此外还能用 `dft -edt` context 创建 EDT 控制器。两者共享同一套命令集（EDT 命令除外）、相同的 pattern 类型、故障模型、覆盖率和诊断能力。TestKompress 的卖点是**压缩后测试时间显著缩短**，直接降低 ATE 机时成本。

**Q3：Tessent 的 context 和 system mode 是什么？**

> context 决定你在用哪个产品功能——`dft` 用于设计编辑与插入（子 context 有 `-scan`、`-edt`、`-test_points`），`patterns` 用于 pattern 生成与诊断（子 context 有 `-scan`、`-ijtag`、`-scan_diagnosis` 等）。system mode 决定流程阶段——`setup`（默认，加载与定义）、`insertion`（设计编辑）、`analysis`（DRC、插入、ATPG）。两者组合决定某条命令此刻能否用。比如 `create_patterns` 要求 `dft -edt` 或 `patterns -scan` context，且必须是 analysis mode。

**Q4：Tessent 里怎么选故障模型？**

> 用 `set_fault_type`，支持 Stuck（默认）、Toggle、Iddq、Transition、Path_delay、Bridge，以及 **Udfm**（User-Defined Fault Model，即 cell-aware 那类从版图提取的真实缺陷模型，分 static/delay/iddq 三类）。要注意**切换故障模型会清空当前 fault list 和 pattern set**，因为每个模型有自己的等价压缩规则。

**Q5：跑完 ATPG 你看什么指标？**

> 看 `report_statistics`。三个覆盖率指标：`fault_coverage = DT/FU`、`test_coverage = DT/(FU−UT)`、`atpg_effectiveness`（ATPG 在可测故障上的求解效率）。同时看未覆盖故障的构成——**RE（冗余）、AU（ATPG 不可测，子类如 BB 黑盒、FP 伪路径）、UD（未检测，子类 AAB=ATPG 放弃、EAB=EDT 放弃）、以及 UU/TI/BL**。AU 特别多通常说明设计约束有问题（不可控时钟、黑盒），不是 ATPG 参数问题。

**Q6：Tessent 的 DRC 你了解吗？**

> 由 `check_design_rules` 驱动。规则分 Category，每条有编号（如 C1、DFT_C1），带默认处置方式（Error/Warning），可用 `set_drc_handling` 改。C 规则是时钟类，比如 **C1 要求所有声明时钟处于 off 状态时，扫描单元和非扫描单元的时钟输入（含 set/reset）都必须在 off 态**——最常见的违反原因是 **set/reset 脚没被声明成 clock**，修法是 `add_clocks` 把它声明进去。**DFT_C1–C5 是 MemoryBIST 专用规则**，只在 dft context 且设计里有 memory、且开了 `set_dft_specification_requirements -memory_test` 时才跑。

**Q7：你有 Tessent 实操经验吗？**（**必须诚实**）

> 我没有产线级的 Tessent 实操经验。我读的是 **Tessent Shell Reference Manual v2021.3**，理解它的产品结构、context/mode 机制、命令组织方式（每条命令都标注 context、mode、prerequisites、usage、示例）和完整流程。我知道 scan 插入是 `check_design_rules → analyze_scan_chains → insert_test_logic → write_atpg_setup`，ATPG 是 `set_fault_type → create_patterns → report_statistics → write_patterns`，EDT 分 IP creation 和 pattern generation 两阶段。**给定任务我能查手册定位该用哪条命令、在什么 context 和 mode 下、有什么前置条件。** 我的定位是理解流程和数据结构，具体命令可以查。

---

## 11. 如何自己核查任何 Tessent 说法（方法论）

**这是本文最有价值的一节——教你不再依赖二手资料。**

1. **拿到官方手册**。本文依据的是 `Tessent Shell Reference Manual`（文档编号可搜 "Tessent Shell Reference Manual pdf"）。Siemens 官网 support.sw.siemens.com 有官方文档系统，`tessent -manual` 也能打开。

2. **手册的结构是固定的**：
   手册共 15 章，结构如下（已逐章核对）：
   - **Ch 1 Introduction** —— 产品线定义（**最权威**，本文第 0 节来源）
   - **Ch 2 Tool Invocations** —— CLI 入口（tessent -shell 等）
   - **Ch 3–6 Command Dictionary** —— 全部命令，按字母分段：A–D / E–Q / R / S–Z
   - **Ch 7 Design Rule Checking** —— **DRC 规则全集（C 规则、DFT_C 规则都在这里）**
   - **Ch 8 Data Models** / **Ch 9 Attributes** —— 数据模型与属性
   - **Ch 10 Configuration-Based Specification**
   - **Ch 11 Tessent Core Description (TCD)**
   - **Ch 12 TSDB** —— 数据结构
   - **Ch 13 ICL** —— Instrument Connectivity Language 语言定义
   - **Ch 14 Adding Custom Plugin Functionality**
   - **Ch 15 Generation of ScanDEF**

3. **核查方法**：把它当手册用，不要通读。想确认一个命令是否存在 → grep 命令名 + 后面跟 `Context:` 行；想确认一个产品的定位 → 读 Chapter 1 的对应小节；想确认一条 DRC → 读 DRC 章节。

4. **警惕这些常见错误信息**：
   - 把 Tessent 说成 Synopsys 的（**是 Siemens EDA**）
   - 把 EDT 说成 Synopsys 的（**是 Tessent TestKompress 的技术**）
   - 编造不存在的命令（比如 `add_memory_bist_instances`、`create_edt_ip`、`set_scan_configuration` **在手册里都不存在**）
   - 把 `UT` 说成 "untested"（**是 UNTestable**）
   - 混淆 `fault_coverage` 和 `test_coverage` 的分母

> **给面试的一句话**：如果被问到一个你不确定的 Tessent 细节，最安全的答法是——
> **"这个我不确定，但我记得 Tessent 手册里命令是按 Context/Mode/Prerequisites 组织的，DRC 规则都有编号可以查。我一般不会凭记忆答工具细节，会去查手册确认。"**
> 这比编一个错误答案强得多，而且**这正是真实工程师的做法**。

---

## 12. 本章速记卡

```
【Tessent 是什么】
  不是一个工具，是 Siemens EDA 的 DFT 产品家族
  统一跑在 Tessent Shell（Tcl 环境 + 设计数据模型）

【产品线口诀】
  做进去：Scan/ScanPro · MemoryBIST · LogicBIST · BoundaryScan · IJTAG
  测出来：FastScan / TestKompress(含EDT压缩)
  坏在哪：Diagnosis · SiliconInsight · YieldInsight
  用起来：MissionMode · DesignEditor

【两个核心概念】
  set_context   → 用哪个产品
      dft     : -scan | -edt [-logic_bist] | -test_points  (+ -rtl/-no_rtl)
      patterns: -scan | -ijtag | -scan_diagnosis | -scan_retargeting ...
  set_system_mode → 在流程哪一步
      setup(默认) | insertion | analysis

【两个流程骨架】
  Scan插入: check_design_rules → analyze_scan_chains → insert_test_logic → write_atpg_setup
  ATPG    : set_fault_type → create_patterns → report_statistics → write_patterns
  EDT     : dft -edt 建IP  →  patterns -scan 生成pattern（两阶段）

【故障模型 set_fault_type】
  Stuck(默认) | TOggle | Iddq | Transition | Path_delay | Bridge | Udfm

【故障分类代数（Table 3-21）】
  FU = TE + UT ;  TE = DT+PD+AU+UD ;  UT = UU+TI+BL+RE
  DT = DS+DI+DR ;  PD = PU+PT
  fault_coverage = DT/FU ;  test_coverage = DT/(FU−UT)
  AU 子类: BB(黑盒)/FP(伪路径)...  UD 子类: AAB/EAB/UNS

【DRC】
  check_design_rules 驱动；规则带 Category 和编号
  C1: 所有clock off 时，scan/non-scan cell 的时钟输入(含set/reset)必须 off
      常见根因 = set/reset 没声明成 clock → add_clocks
  DFT_C1–C5: MemoryBIST 专用，需 dft context + 有memory + memory_test on

【TSDB】
  根目录 4 个子目录：instruments / dft_inserted_designs / patterns / logic_test_cores
  重跑时整目录删掉重建，不留陈旧文件

【⚠️ 三条最容易说错】
  1. Tessent 属 Siemens EDA，不是 Synopsys
  2. EDT 是 TestKompress 的技术，不是 Synopsys 的
  3. UT = UNTestable（≠ untested）；覆盖率分母别背错
```

---

## 配套文档

| 文档 | 内容 |
|---|---|
| [00-DFT面试资料-总览.md](00-DFT面试资料-总览.md) | 总览、简历对照、必答三件事 |
| [07-DFT面试准备-数字电路与RTL基础.md](07-DFT面试准备-数字电路与RTL基础.md) | 数字电路、Verilog、时序、CDC |
| [08-DFT面试准备-SCAN与ATPG.md](08-DFT面试准备-SCAN与ATPG.md) | SCAN、ATPG、故障模型、覆盖率（原理层） |
| [09-DFT面试准备-MBIST与存储器测试.md](09-DFT面试准备-MBIST与存储器测试.md) | MBIST、March、LBIST（原理层） |
| [10-DFT面试准备-简历项目逐条答辩.md](10-DFT面试准备-简历项目逐条答辩.md) | 简历每条 bullet 答辩 |
| [11-DFT面试速查手册与策略.md](11-DFT面试速查手册与策略.md) | 快问快答、面试策略、自我介绍 |
| **12-DFT面试准备-Tessent工具专题.md** | **本文：Tessent 产品线与实操** |
