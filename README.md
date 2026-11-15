# dsh-mcp-tia 插件说明

> [!IMPORTANT]
> **装完不能直接用。** 本插件只是"接线"，它**不含 TIA Portal，也不含任何 TIA Portal MCP Server 可执行文件**——西门子**没有**官方 MCP Server。从插件市场/命令安装后，你还需要自己准备一个 Openness MCP Server（见 [后端怎么选](#后端怎么选)）并填好路径，`mcp__tia__*` 工具才会出现。
>
> - **自己配**：按下面的 [安装后还需要三步](#安装后还需要三步) 做。
> - **让 AI 帮你配**：把 [这段提示词](#让-ai-帮你完成配置) 原样发给你的 DSH 会话，剩下的交给它。
>
> 安装本身不会再让 DSH 崩溃（`failOnStartupError` 已默认 `false`）；配置不对时的表现是"**没有工具**"，而不是"起不来"。

## 这是什么

`dsh-mcp-tia` 是一个 **DSH（DeepSeek Harness）桥接插件**，用来把一个本地运行的 **TIA Portal MCP Server** 接入 DSH，让你能在 DSH 会话里用自然语言读写博途工程（创建项目/设备、生成块与变量、编译诊断、保存）。

**它不是 MCP Server 本身，也不包含 TIA Portal、Openness DLL 或任何西门子软件。** 它只负责"接线"：启动你指定的 Server 可执行文件，把它暴露的工具注册成 DSH 的 `mcp__tia__*` 工具。真正驱动博途的是那个 Server，它通过西门子官方的 **TIA Portal Openness** API 操作本地已安装的 TIA Portal。

底层使用 DSH 宿主自带的 `@deepseek-ai/dsh-mcp-client`，本插件只是提供一份配置（`cordis.patch.yml`）。

## 前置条件（必须先准备好）

1. **Windows**——Openness 只有 Windows 实现，别的系统没有替代。
2. **TIA Portal 已安装**（V15–V21 均被社区项目覆盖）。注意版本差异：
   - **V21** 的 Openness 是**模块化**的：`PublicAPI\V21\net48\Siemens.Engineering.Base.dll`、`…Step7.dll`、`…WinCCUnified.dll` 等**一组** DLL，**没有**单一的 `Siemens.Engineering.dll`。
   - **V20 及更早**是单体 `Siemens.Engineering.dll`。
   - 所以后端 exe 必须和你的博途主版本**匹配**（多数项目靠 `--tia-major-version` 选择）。
3. **.NET Framework 4.8**（Openness API 本身要求）。
4. **当前 Windows 用户加入本地组 `Siemens TIA Openness`，然后注销/重登**——组权限重登才生效。这是最常见的"一切看起来都对但没有工具/一调用就报权限错"的原因。管理员 PowerShell：

   ```powershell
   net localgroup "Siemens TIA Openness" "%USERNAME%" /add
   ```

5. **博途侧允许 Openness 访问**（V17 起有对应的安全设置项）。后端自带的 `doctor` 子命令会检查这一项，先跑它。
6. **一个 Openness MCP Server 可执行文件**（见下）。本仓库**不提供**。

## 后端怎么选

三个维护中的开源后端，都能被这一个插件接进来（只换 `command`/`args`）：

| 后端 | 支持版本 | 工具数 | `command` / `args` 要点 |
| --- | --- | --- | --- |
| [heilingbrunner/tiaportal-mcp](https://github.com/heilingbrunner/tiaportal-mcp) | V21 默认，旧版靠参数 | 65 只读 + **49 写**（写工具要 `--allow-write` 才注册） | `command`: `…\TiaMcpServer.exe`；`args`: `--tia-major-version 21`，可选 `--project-path`、`--software-path`、`--export-path`、`--preserve-path`、`--with-dependencies`、`--allow-write`。自检：`TiaMcpServer.exe --doctor` |
| [Czarnak/tia-portal-mcp](https://github.com/Czarnak/tia-portal-mcp)（NuGet `TiaMcpServer`） | **仅 V21** | 7 只读 / **17 读写** | `command`: `tia-mcp`（`dotnet tool install -g TiaMcpServer` 装出来的全局工具）；`args`: `--access-mode read-write`（默认只读）或 `full`。自检：`tia-mcp doctor` |
| [bulaofen0036-coder/TIA_Portal_Openness_MCP](https://github.com/bulaofen0036-coder/TIA_Portal_Openness_MCP) | **V20 + V21**（交付包内有 `runtime\v20` 与 `runtime\v21` 两个运行时） | 238（2.7.11；默认精简档 65，其余用 `FindTools` + `CallTool` 按需展开） | `command`: V21 用 `…\runtime\v21\TiaMcpServer.exe`，V20 用 `…\runtime\v20\TiaMcpServer.exe`（Release zip 解压后跑 `配置MCP.bat` 生成配置）；`args`: `--tia-major-version 21`，非默认安装加 `--tia-portal-location "<Portal 根目录>"`；只读加 `--read-only`（≥2.7.11）。自检命令以该后端 README 为准 |

> 工具数是各后端 README 的记录值，会随版本变化——**以连接后实际枚举为准**（本机实测：bulaofen 2.7.5 默认档 62 个、2.7.11 精简档 65 个）。

选择建议：

- **要写 SCL/LAD 代码**（本插件的主要用途）→ heilingbrunner（能读写真·文本格式，见 [能写梯形图吗](#能写梯形图吗lad梯形图)）或 bulaofen（自带 SCL/LAD 指令库与模板，中文文档）。
- **只要小而安全的工具面** → Czarnak（工具最少、写操作有 `dryRun` 与内容哈希守卫、带审计日志），但只支持 V21。
- **想少编译、拿现成 exe** → bulaofen 的 Release 包里是预编译运行时；它的 `配置MCP.bat` 只认 Claude/Cursor/VS Code，**不认 DSH**，所以在 DSH 里仍然要按本文手写这一行。

> 博途工程是二进制，Openness 的写入**只在内存生效**，必须再调保存类工具才落盘。
> ⚠️ **在线能力默认就在工具面里**：bulaofen 后端的默认精简档就包含 `DownloadToPlc`、`GoOnline`/`GoOffline`、`SetWatchTableModifyValue`（其中 `SetWatchTableModifyValue` 会写真实 CPU），能直接作用到真实设备。升级到后端 2.7.11 后可用 `--read-only`（或环境变量 `TIA_MCP_READ_ONLY=1`）强制拒绝改工程/编译/保存/下载/写在线值；**不加该参数时不存在任何强制拦截**。

## 安装

```sh
dsh plugin --profile desktop add github:yh01010/dsh-mcp-tia
```

（`desktop` 换成你的 profile 名。）安装后**重启 DSH** 生效——但注意：重启后**不会自动出现工具**，还要做完 [安装后还需要三步](#安装后还需要三步)。

也可以把 `dsh-mcp-tia` 加入 profile 的 `package.json` → `dsh.profile.bundles` 后 `pnpm install`，重启生效。

> 注意：本包**不发布到 npm**，只能从 GitHub 安装。

## 安装后还需要三步

1. **拿到一个后端可执行文件**（上表任选一个）。注意 bulaofen 是**目录形态**的交付包：exe 在 `runtime\v21\` 里，旁边有它的运行时文件，**整包**解压到固定位置即可（例如 `%USERPROFILE%\.local\TIA_Portal_Openness_MCP\`），别只把那一个 exe 搬走；只有 heilingbrunner 那种单 exe 后端才适合随手放到 `%USERPROFILE%\.local\bin\`。多数后端要求先编译一次，或用它的 Release 预编译包。
2. **确认环境**：用后端自带的 `doctor` 自检 .NET Framework、Openness DLL、`Siemens TIA Openness` 组成员身份与博途访问开关。**先跑 doctor，再谈接线**。
3. **填真实路径并重启**：按 [配置](#配置) 一节在 `$DSH_HOME/cordis.patch.yml` 里覆盖 `command`，重启 DSH，工具列表里应出现 `mcp__tia__*`。

## 让 AI 帮你完成配置

不想手改配置，就把下面这段原样发给你的 DSH 会话：

```text
请帮我把 DSH 的博途插件（dsh-mcp-tia）配置到可用状态，参考
https://github.com/yh01010/dsh-mcp-tia 的 README。

现状：插件包已装好，但 mcp__tia__* 工具没出现，因为
cordis.patch.yml 里还是占位路径。

请依次做：
1. 先确认本机 TIA Portal 的主版本和安装根目录（例如
   C:\Program Files\Siemens\Automation\Portal V21）。V21 的 Openness
   是模块化的 PublicAPI\V21\net48\Siemens.Engineering.Base.dll，
   没有单体 Siemens.Engineering.dll，选后端时必须匹配。
2. 确认当前用户是否已在本地组 "Siemens TIA Openness" 里；不在就先
   告诉我需要管理员执行 net localgroup 命令并注销重登。
3. 检查我选定的后端可执行文件是否存在；不存在就告诉我该从哪个
   项目/Release 取、放到哪里，并先把它的 doctor 子命令跑通。
4. 编辑 $DSH_HOME/cordis.patch.yml，用「id 覆盖」写法（不要用 insert，
   否则会出现两行同 id，serverName 冲突导致启动中止）填上真实路径：
   command: <exe 的完整路径>
   args: <按该后端的参数规范，见 README 的「后端怎么选」表>
5. 提醒我重启 DSH，重启后确认 mcp__tia__* 工具是否出现；先只跑只读
   工具（列设备/导出块），确认无误后再决定是否打开写模式。

不要改动插件包内部的 cordis.patch.yml。
不要在没有备份的工程上执行任何写操作。
```

> 前提：会话要有该文件的写入权限。`$DSH_HOME` 通常在会话工作区之外，可能触发一次权限确认。

## 配置

本插件**刻意不携带任何个人机器路径**，`cordis.patch.yml` 里的 `command`/`args` 是占位符。你必须改成自己机器上的真实路径。

两种接法 **二选一，不要同时用**：

| 你的情况 | `$DSH_HOME/cordis.patch.yml` 里怎么写 |
| --- | --- |
| **已装了插件包**（市场/`dsh plugin add`） | 只写 **id 覆盖**（下面那段 `- id: mcp-tia`，**不要** `insert`）。包里的 bundle 行已经插入了这一行，覆盖它的 `config` 即可 |
| **没装插件包**，手工接线 | 用 `insert` 把整行插进去（见下方折叠块） |

> [!WARNING]
> 插件包已装、又在 home patch 里 `insert` 同一 id 的行 → 会得到**两行同 id** → 两个 mcp-client 实例抢 `serverName: tia` → DSH 启动时报 `serverName "tia" is already in use by another mcp-client instance` 并中止。这是最容易踩的坑。

### 只读 / 写入参数（不要混用）

三个后端的开关语义相反，参数不能互相照抄：

| 后端 | 默认档 | 强制只读 | 启用写入 |
| --- | --- | --- | --- |
| bulaofen0036-coder **≥2.7.11** | 可写 | `--read-only`（等价环境变量 `TIA_MCP_READ_ONLY=1`） | 不加（默认就能写） |
| heilingbrunner | 只读 | 不加 `--allow-write`（写工具根本不注册） | `--allow-write` |
| Czarnak | 只读 | 省略 `--access-mode` | `--access-mode read-write` |

本包模板的 `--tia-major-version` 是 bulaofen / heilingbrunner 共用的；`--allow-write` **只属于 heilingbrunner**，加到 bulaofen 上不会生效。另外 `lite`/`full` 档只决定**列出哪些工具**，不是写保护——真正的写保护是 `--read-only`。

> 本机实测（2.7.11 + `--read-only`）：MCP 握手报 `2.7.11.0`，`tools/list` 列出 **65** 个工具；`Bootstrap` 正常返回 `ready: true`（`opennessGroupOk: true`、`tiaVersionInUse: 21`）；`SaveProject`、`CompileSoftware`、`DeletePlcBlock` 全部返回 `isError: true`，原文 `READ-ONLY MODE: '<工具名>' was refused because it is not on the read-only list (...). Nothing was executed.` 想核对你的实例是否真的只读，直接调一次 `SaveProject` 看这句即可。

id 覆盖写法（**装了插件包时用这个**，不会改到仓库文件）：

```yaml
# $DSH_HOME/cordis.patch.yml
- id: mcp-tia
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: tia
    transport: stdio
    command: 'C:\Users\<你的用户名>\.local\bin\TiaMcpServer.exe'
    args:
      - '--tia-major-version'
      - '21'
      # bulaofen ≥2.7.11：加这一段 = 强制只读（见上表）
      # - '--read-only'
    toolCallTimeoutMs: 180000
    failOnStartupError: false
```

<details>
<summary>不装插件包时的手工接法（<code>insert</code> 整行）</summary>

```yaml
# $DSH_HOME/cordis.patch.yml
- insert:
    - id: mcp-tia
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: tia
        transport: stdio
        command: 'C:\Users\<你的用户名>\.local\bin\TiaMcpServer.exe'
        args:
          - '--tia-major-version'
          - '21'
        toolCallTimeoutMs: 180000
        failOnStartupError: false
```

这种方式下**不要**再装插件包，否则与上面的坑同源。

</details>

关键配置项：

| 字段 | 含义 | 宿主默认 |
| --- | --- | --- |
| `command` | 后端可执行文件 | 占位符（必填） |
| `serverName` | 工具名前缀，profile 内须唯一 | `tia` |
| `args` | 后端自己的命令行参数（各后端不同） | `--tia-major-version 21` |
| `toolCallTimeoutMs` | 单次工具调用超时（ms）。**博途冷启动、编译、批量导入都可能超过 60 秒**，故调大 | 60000 |
| `failOnStartupError` | 启动失败时立即抛错，而不是静默缺少工具。**开启会让 DSH 起不来**（每次启动都中止），只在路径已验证后再开 | false |
| `reconnect` | 断线重连策略。**默认就是开启的**：500ms 起、指数退避到 30s、最多 10 次 | `{ enabled: true, initialDelayMs: 500, maxDelayMs: 30000, maxAttempts: 10 }` |
| `env` / `cwd` | 传给后端进程的环境变量 / 工作目录 | 空 |

> 第三列是**宿主**（`@deepseek-ai/dsh-mcp-client`）的默认值。本仓库 `cordis.patch.yml` 模板在此之上把 `toolCallTimeoutMs` 覆盖成 **180000**（给编译/批量导入留余量）；你自己 home patch 里写多少以你为准（作者本机实际用 300000）。

**路径含空格**：在 YAML 单引号字符串里直接写整串即可，例如 `'D:\Program Files\your\tia\TiaMcpServer.exe'`（注意本包内示例均用占位路径，真机路径只写在 home patch 里）。

## 能写梯形图吗（LAD/梯形图）

**能——而且是文本写法。** 但也**不要手写 SimaticML FlgNet XML**：这是后端写死的规则，`GetAuthoringGuide('lad')` 原文：

> NEVER hand-write FlgNet XML for ladder logic — UId bookkeeping and entity escaping make it fail constantly.

可靠路径是 **S7DCL 梯形图文本**（SIMATIC SD，在 S7-1500 / V21 上编译验证过）：

```text
AI 写 .s7dcl 梯形图文本（+ 可选 .s7res 承载中文文本）
   → ImportFromDocuments / ImportBlocksFromDocuments
   → CompileSoftware → SaveProject
```

### S7DCL 梯形图长什么样

```text
{ S7_Optimized := "TRUE"; S7_PreferredLanguage := "LAD"; S7_Version := "0.1" }
FUNCTION "Name" : Void
    VAR_INPUT  Start : Bool; END_VAR
    VAR_OUTPUT Run   : Bool; END_VAR
    { S7_Language := "LAD" }
    NETWORK
        RUNG wire#powerrail
            Contact( #Start )
            Coil( #Run )
        END_RUNG
    END_NETWORK
END_FUNCTION
```

| 元件 | 写法 |
| --- | --- |
| 常开 / 常闭 / 取反 | `Contact(op)` / `I_Contact(op)` / `Not()` |
| 线圈 | `Coil(op)` / `S_Coil(op)` 置位 / `R_Coil(op)` 复位 |
| 边沿 | `P_Trig(mem)` / `N_Trig(mem)` —— **一个**参数；`P_Contact(signal, mem)` / `N_Contact(...)` —— **两个**参数。两者不可互换，参数给少了导入直接失败 |
| 比较框 | 前置 `{ S7_Templates := "SrcType := Int" }`，再 `GT_Contact(in1 := , in2 := )`（也有 `LT_` / `EQ_` / `NE_` / `LE_`） |
| 数学 / 传送框 | `Move(in := , out1 => )`、`Sub(in1 := , in2 := , out => )`（Add / Mul / Div / Neg 同形） |
| 区间框（LAD 独有） | `InRange(min := , in := , max := )` / `OutRange(...)` |
| IEC 定时器 | 静态变量 `tmr : TON_TIME;` + `{ S7_Templates := "time_type := Time" }` + `#tmr.TON(pt := T#5s, et => )`，输出用 `Contact(#tmr.Q)` 读（块必须是 FUNCTION_BLOCK） |
| 调用 FC | rung 里写 `"FC_Name"()` |
| 操作数 | `#局部变量`、`"全局变量"`、`"DB".成员` |
| 并联 / 多输出 | rung 内联 `wire#w1` 分叉，后续 rung 用 `END_RUNG wire#w1` 汇回该节点 |

三个最容易踩的坑：

1. **每个 NETWORK 前面都必须有它自己的 `{ S7_Language := "LAD" }`**。漏了会报 `Syntax error 'NETWORK' expecting END_FUNCTION_BLOCK`（真实导出的 10147 个网络里，每一个都带这个 pragma）。
2. **通用指令需要 `{ S7_Templates := "..." }`，且模板名因指令而异**（`SrcType` 用于比较/数学、`DestType` 用于 Convert、`time_type` 用于 IEC 定时器、`date_type`、`str_type`）。写错时错误信息会列出 `Allowed Template Names:`，照抄那串，别猜。
3. **编码是头号杀手**：`.s7dcl` / `.s7res` 一律 **UTF-8 带 BOM**；`.scl` 只在纯 ASCII 时才可以不带 BOM，含中文要么加 BOM 要么注释全用 ASCII——否则会被当 GBK 读，得到乱码 + 假的语法错误。

### 一个块里混用 LAD + SCL

块头保持 `S7_PreferredLanguage := "LAD"`，每个 NETWORK 各带自己的 `{ S7_Language := "SCL" }`，SCL 网络里写普通 SCL 语句（没有 RUNG）。这是"梯形图块里做算术"的官方做法。

### 读懂已有的 LAD

用 `DescribeBlockLogic`，不要把块导出来手工追线。它把每个 rung 还原成可读表达式（串联 ` · `、并联 ` + `、常闭显示为 `/操作数`），列出线圈与 MOVE/比较/定时器框，并且会**标出接在字面常量上的触点**（`⟨恒断·禁用本行⟩`——常开触点接 FALSE 会静默废掉整行，肉眼几乎看不出来）。

### 实操建议

- 写 LAD/SCL 之前先 `GetAuthoringGuide('lad' | 'scl' | 'db' | 'hmi' | 'errors')`——后端反复强调 "Most quality problems come from skipping this"。
- 某个元件不确定写法：先对一个真实 LAD 块跑 `ExportAsDocuments`，抄它导出的 `.s7dcl` 结构（SD 比 XML 小约 9 倍且可读）。
- 只需要"调用一个 FC"的网络，可以直接用 `BuildFlgNetCallXml` / `ComposePlcLadFcBlockXml`，它们替你生成 XML。
- 写完必须 `CompileSoftware`（或 `CompileAndDiagnosePlc`），再 `SaveProject`——**不会自动保存**。
- 语言在建块时就定死：**Openness 不会把 SCL 自动翻译成 LAD**。
- **F-LAD / 安全程序**：`PublicAPI` 不暴露 F-CPU 编译，要在 TIA UI 里手工触发；安全块与安全变量也拒绝多数修改。

各后端的 LAD 路径：

| 后端 | LAD 怎么写 | 备注 |
| --- | --- | --- |
| bulaofen | 推荐 **S7DCL 文本** → `ImportFromDocuments`；也支持 LAD XML 导入（`BuildFlgNetCallXml` 等替你生成），自带 `docs/lad-instruction-library.md` 与 `GetAuthoringGuide('lad')` | LAD 资料与护栏最全 |
| heilingbrunner | `ImportXmlBlock` 导 LAD XML；或 V20+ 的 `.s7dcl/.s7res` | 单 exe；文档记录了 `.s7res` 必须对**所有条目**有 en-US 标签、否则可能失败的 Openness 已知问题 |
| Czarnak | 只能 XML（`format=source` 仅对 SCL 语言块、DB、UDT 可用） | 工具少、守卫严 |

## 安装后验证

1. 重启 DSH 后，看工具列表里是否出现 `mcp__tia__*`。
2. 若没有，先手动确认后端 exe 存在、且能独立启动并自检：

   ```powershell
   # 路径换成你选定后端的 exe；bulaofen 交付包是 <解压目录>\runtime\v21\TiaMcpServer.exe
   Test-Path '<你选定后端的 exe 完整路径>'
   & '<你选定后端的 exe 完整路径>' --doctor   # 该后端没有 doctor 的话，按它的 README 换成对应自检命令
   ```

   后端能独立启动但工具仍不出现，说明问题在 DSH 接线（patch/配置），不在后端。
3. 接上后**先只做只读**：列设备、列块、导出块，确认 AI 看到的工程结构和你的一致，再考虑开写模式。
4. 装了 [`dsh-mcp-panel`](https://github.com/PerryLink/dsh-mcp-panel) 的话，可以在设置页看这个 MCP 的连接状态、已注册工具、错误与重连计数。

## 排障

| 现象 | 可能原因 |
| --- | --- |
| 看不到 `mcp__tia__*` 工具 | 最常见：占位路径还没换成真实路径（刚装完就是这个状态）。其余：exe 路径错、后端启动即崩、patch 没生效 |
| 一调用就报权限/组错误 | 当前用户不在 `Siemens TIA Openness` 本地组，或**加了组但没注销重登**；也可能是博途侧未允许 Openness 访问。跑后端 `doctor` 确认 |
| DSH 因该插件无法启动 | ① home patch 里 `insert` 了与插件包重复的 `mcp-tia` 行 → `serverName` 冲突；删掉重复的那条 `insert`。② 你自己把 `failOnStartupError` 开成了 `true` 且路径错；设回 `false` 或改对路径 |
| 工具调用超时 | 博途冷启动/编译/批量导入超过 `toolCallTimeoutMs`（调大它；部分后端另有"预热"脚本可让后续调用快很多） |
| 导出的块报"不一致"错误 | 博途拒绝导出未编译通过的块。先编译，再导出 |
| 写入"成功"但工程没变 | Openness 的修改只在内存，**必须调用保存类工具**才落盘 |
| 工具太多、模型被淹没 | 选默认只暴露少量工具的后端（Czarnak 17 / bulaofen 精简档 62–65 / heilingbrunner 只读 65），或在后端参数里限制工具面 |

## 安全提醒

以下操作**必须人工确认**后再执行，且先备份工程。注意：这段是**操作要求**，不是已经生效的机制——本插件是纯配置桥接，DSH 侧不会按 MCP 注解自动拦写入，唯一的强制手段是后端的 `--read-only`（见上文「只读 / 写入参数」表）。

- 写入任何东西到生产工程（先只读跑通，再开写模式）
- 修改硬件地址、网络参数、通信设置
- 删除设备、块、变量表、库
- 修改安全相关对象或 Know-how 保护对象
- 下载到 PLC/HMI、进入在线模式——**这些默认就在工具面里**（`DownloadToPlc`、`GoOnline`/`GoOffline`、`SetWatchTableModifyValue`），插件层没有审批拦截：要么用后端 `--read-only` 强制只读，要么你本人全程盯场

推荐在 **PLCSIM**（S7-PLCSIM / PLCSIM Advanced）的离线工程上先验证整条链路。

## 移除

```sh
dsh plugin --profile desktop remove dsh-mcp-tia
```

（或从 `dsh.profile.bundles` 移除后 `pnpm install`），重启生效。想临时停用，可在 profile 的 `cordis.patch.yml` 里给 `mcp-tia` 行加 `disabled: true`。

## 许可证

MIT。本插件与西门子（Siemens AG）无隶属关系；TIA Portal、SIMATIC、STEP 7、WinCC 是西门子商标。Openness DLL 与博途本体不随本包分发，需自行安装并遵守其许可条款。

## 说明来源

- 西门子官方：[TIA Portal Openness API 文档](https://docs.tia.siemens.cloud/r/en-us/v21/tia-portal-openness-api-for-automation-of-engineering-workflows/)
- 三个后端各自的 README（工具清单、`doctor`、写模式开关、已知限制）
- 社区汇总：[TIA Openness / MCP 开源项目清单](https://cloud.tencent.cn/developer/article/2708400)
