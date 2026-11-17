# CHANGELOG

本插件的版本号只跟随**插件仓库**（`package.json`），与 TIA Portal、Openness、以及三个后端各自的版本都没有对应关系——那些版本见 README 的「兼容性」一节。

## v0.2.0 — 2026-11-17

### 文档修正（v0.1.0 写错的事实）
- bulaofen 后端的 `command` 路径改为真实布局 `…\runtime\v21\TiaMcpServer.exe` / `…\runtime\v20\TiaMcpServer.exe`（原文写的 `…\bin\Release\net48\…` 在交付包里不存在），并说明安装第 1 步是**整包解压**目录形态的交付包，而不是搬走单个 exe。
- 工具数更新为 238（v2.7.11）/ 默认精简档 65；原文的「224（默认 62）」已过时。
- 删掉「下载到 PLC/HMI、进入在线模式不属于 Openness 自动化范围、不属于本插件范围」的错误表述：默认精简档里就带着 `DownloadToPlc`、`GoOnline`/`GoOffline`、`SetWatchTableModifyValue`，后者会写真实 CPU。
- 关键配置项表的默认值一列改为标注**宿主默认**，`reconnect` 更正为**默认开启**（500ms 起、指数退避到 30s、最多 10 次）。
- 安装后验证的 `Test-Path` 去掉写死的本机路径。

### 新增
- **只读 / 写入参数（不要混用）** 小节：三个后端的写开关语义相反（bulaofen `--read-only`、heilingbrunner `--allow-write`、Czarnak `--access-mode`），并注明 `lite`/`full` 档只决定列出哪些工具、不是写保护。
- 本机实测记录（2.7.11 + `--read-only`）：握手 `2.7.11.0`、`tools/list` 65 个、`Bootstrap` ready、三个写工具被 `READ-ONLY MODE` 拒绝的原文。
- 「安装后验证：五层定位」——路径 → 进程 → 握手 → 枚举 → 最小调用。
- 「兼容性」三栏表（上游声明 / 本机已实测 / 未验证），含 TIA 主版本、后端版本、.NET 4.8.1、DSH 版本与验证日期。
- 「本地检查」：`npm run validate` / `npm test` 的用法。

### 变更
- 后端选择以 **bulaofen 2.7.x 为主路径**（配置、验证、排障、LAD 教程都按它写），heilingbrunner / Czarnak 降为参考，并明确「不要照抄本文命令行」。
- LAD/SCL 教程顶部标注「适用后端：bulaofen 2.7.x」，说明本节工具名换后端后不通用。
- `validate.mjs` 从「在文本里找子串」改为**解析 YAML 再校验结构**（顶层 id/name/config、`serverName` 字符集、`transport`、`command`、`args`、`toolCallTimeoutMs`、`failOnStartupError`、`insert` 递归、重复 id），新增 `--patch <file>` 与 `--root <dir>` 两个参数，并删掉原文里不成立的 `exit 2` 说明。
- 新增 `test/`（`node --test`，5 个用例 + fixtures）与 `.github/workflows/validate.yml`（push/PR 跑 validate + test）。
- 模板 `cordis.patch.yml` 增加 `--read-only`（bulaofen ≥2.7.11）与对应的使用说明；`command` 占位符改为 bulaofen 的真实目录布局。
- `package.json`：`version` 0.1.0 → 0.2.0，新增 `scripts.validate`、`scripts.test` 与 `devDependencies.yaml`。

### 后端升级（本机验证，2026-11-15）
- bulaofen 后端 v2.7.5 → **v2.7.11**：`tools/list` 65 个（精简档）、`Bootstrap` 返回 `ready: true` / `opennessGroupOk: true` / `tiaVersionInUse: 21`，`SaveProject` / `CompileSoftware` / `DeleteDevice` 在 `--read-only` 下全部被拒。

## v0.1.0 — 2026-10-16

首个版本：把第三方 TIA Portal Openness MCP Server 接进 DSH 的纯配置桥接（`cordis.patch.yml` + 文档），不含 TIA Portal、Openness DLL 与任何后端二进制。