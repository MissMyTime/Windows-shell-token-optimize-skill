---
name: windows-shell-token-saver
description: 在 Windows 终端（PowerShell 5.1/7、cmd、Git Bash、WSL）执行命令时降低 token 消耗。提供 shell 路由决策、命令等价映射、CLI 消噪参数、UTF-8 与编码处理、PS 版本差异与分场景错误处理。当 agent 需要在 Windows 上跑终端命令、输出冗长、出现中文乱码、需要反复探测或 shell 选择不定时使用。
license: MIT
version: 0.2.0
metadata:
  title: Windows Shell 省 Token 妙招
---



# Windows Shell 省 Token 妙招

## 目标

Windows 终端默认输出冗长，shell 种类多且语法和 macOS/zsh 差异大，编码容易乱码，错误信息啰嗦，agent 每条命令都要消耗大量 token。这份文档整理了路由和执行两层默认行为，让 Windows 终端命令选得对、输出短、一次完成、无乱码。

## 第 0 层：先路由，再执行

按顺序决策，命中即停：

1. 平台提供专用文件工具（读文件/搜内容/找文件）时，用专用工具，不要启动 shell 跑 `gci/sls/gc/grep/find` 之类的命令。输出已经结构化，省 token 幅度很大。
2. 必须用 shell 时，按场景选：
   - 项目在 WSL 文件系统，或依赖 Linux 工具链（apt/gcc/docker/make）→ WSL：`wsl -e bash -c '...'`。`-c` 是非登录 shell，不加载 profile、启动更快、环境更干净；只有确实需要登录环境（读 `~/.profile` 里的 PATH 等）时才用 `-lc`。
   - Windows 本地项目、通用文件/文本命令 → Git Bash：`bash -c '...'`，bash 语法不用翻译，出错概率低
   - Windows 系统管理（服务/注册表/进程/ACL/事件日志）→ PowerShell 7：`pwsh -NoProfile -Command '...'`
   - 只有自带 5.1 → `powershell.exe`，但禁用 `&&`/`||` 与三元 `? :`（详见版本差异节）
   - execution policy 拦截时**先做进程级绕过**（`pwsh -ExecutionPolicy Bypass -File x.ps1`，零成本、不改系统策略，详见「停止条件」）；绕过仍失败、或 PS 反复失败时才用 cmd 临时兜底（语法速查见 references 第十节）
3. 会话开头**一次调用**完成环境探测 + 初始化（合并命令省往返），结果记住后复用，不要每条命令都探测：

```powershell
# 探测与初始化合并：一次调用做完
Get-Command pwsh,bash,wsl,git -EA SilentlyContinue | % Source
$OutputEncoding=[Console]::OutputEncoding=[Text.Encoding]::UTF8
$ProgressPreference='SilentlyContinue'
$env:NO_COLOR='1'
```

## 核心原则

1. 专用工具优先，能不用 shell 就不用。
2. 一次命令完成一件事，别拆成多次探测。
3. 输出越短越省 token，只输出需要的字段，拒绝默认宽表和对象展开。
4. 单条合并命令优先于长管道，能用内置参数就用内置参数。
5. 先设编码 UTF-8，再跑命令。
6. 错误处理分场景（见下），不一刀切静默。
7. 复用会话内已知结果，不重复执行同一探测。

## 会话初始化

按 shell 选一种，会话开头跑一次。

pwsh / Windows PowerShell：

```powershell
$OutputEncoding=[Console]::OutputEncoding=[Text.Encoding]::UTF8
$ProgressPreference='SilentlyContinue'
$env:NO_COLOR='1'
```

bash / Git Bash / WSL：

```bash
export NO_COLOR=1 TERM=dumb PAGER=cat GIT_PAGER=cat GH_PAGER=cat
```

- `$ProgressPreference='SilentlyContinue'`：屏蔽 Invoke-WebRequest 等下载进度条，进度条在非交互环境会刷出大量垃圾字符。
- `NO_COLOR=1`：去掉 ANSI 颜色转义码，颜色码会让输出 token 膨胀不少。
- `PAGER=cat`/`GIT_PAGER=cat`/`GH_PAGER=cat`：防止 git、gh 等命令进入 pager 等待人工按键，agent 撞上 pager 会卡住不返回。
- `TERM=dumb` 让 TUI/颜色程序退化为纯文本；极少数安装器在 dumb 终端下报"终端能力"错误，遇到就去掉 `TERM=dumb` 保留其余。
- 个别老 Win32 程序只看控制台代码页，上述设置后中文仍乱码，先执行 `chcp 65001 > $null` 再重跑（`> $null` 抑制 `chcp` 自身回显的 `Active code page: 65001` 一行）。
- **`.ps1` 脚本文件编码（Agent 生成脚本必读）**：`.ps1` 若以 **GBK/ANSI** 保存，中文与 `✅`/`❌` 等特殊符号会被当成代码字节解析，直接报 `UnexpectedToken` / `缺少右“)”`。写入要求、5.1/7 的 BOM 差异与跨版本 .NET 写法统一见「硬规则」第 10 条，此处不重复。**状态标记、日志尽量用纯 ASCII**（`[PASS]`/`[FAIL]`），执行前可用 `Get-Content -Encoding UTF8 -TotalCount 3 xxx.ps1` 快速核对首行未被转码。

## 硬规则

1. 输出行数不可预估的命令，必须挂 `| Select-Object -First 50`（bash：`| head -50`）兜底，确认不够再加大。不封顶的命令很容易把上下文撑爆。命令自带内置行数限制（如 `git log -5`）不受此约束。
2. 输出不可控先落盘再选读：`cmd > out.txt 2>&1`，然后 `wc -l out.txt`（PS：`(gc out.txt | Measure-Object).Count`）看规模，再用 `gc out.txt -TotalCount/-Tail` 分段读需要的部分。**呼应第 0 层「专用工具优先」：落盘后若平台有专用读文件工具，优先用工具按偏移分段读，比手写管道更省 token。**
3. 外部 CLI 必须消噪：git/gh 加 `--no-pager`；npm install 排查期用 `--no-audit --no-fund --no-progress`（保留错误），确认无错再叠 `--silent`（会吞错误，Agent 易误判成功）；dotnet 加 `--nologo -v q`；docker build 加 `--progress quiet`（`plain` 是详尽日志，反而更费 token）；测试只回传失败摘要，`pytest -q --tb=short`、mocha `--reporter min`（非 TTY 管道下若吐光标转义序列就换 `dot`）、jest `--silent`（只压测试内 `console.*`，不压结果报告，配 `2>&1 | tail` 取摘要）、vitest `vitest run --silent --reporter=dot`（vitest 没有 `min` reporter，那是 mocha 的）。全表见 references。
4. 只要概览不要全量：状态用 `git status --short`，diff 用 `git --no-pager diff --stat`，日志用 `git --no-pager log --oneline -5`；git 输出含中文路径时加 `-c core.quotePath=false`，否则文件名显示为 `\346\226\207…` 八进制转义。
5. PowerShell 5.1 里 `curl` 是 `Invoke-WebRequest` 的别名（PS 7 已移除，改而直接调用系统原生 `curl.exe`），返回的是响应对象，默认输出一堆冗余属性，会刷爆 token。需要 HTTP 请求时一律写 `curl.exe` 调用系统原生 curl（**`curl.exe` 在 5.1 与 7 下行为一致，推荐作为通用写法**），或用 `irm`（Invoke-RestMethod）拿纯数据。同理 `where` 是 `Where-Object` 的别名，查命令位置写 `where.exe cmd` 或 `Get-Command cmd`。
6. 外部 exe（npm/dotnet/git 等）的 stderr 不受 `$ErrorActionPreference='Stop'` 控制，出错退出也不会抛异常。写操作调用后检查退出码：`cmd; if ($LASTEXITCODE -ne 0) { throw "failed" }`，或用 `exit $LASTEXITCODE` 直接透传退出码。**一律以 `$LASTEXITCODE` 为准，紧贴外部命令后立刻读**（该变量只会被原生外部命令覆盖，中间再跑别的 exe 就会被冲掉）。**不要用 `$?` 判断外部命令成败**：`$?` 表示"上一条命令是否成功"，在 PS 5.1 下一旦外部命令往 stderr 写过内容（git 进度、node warning，哪怕退出码是 0），`$?` 就会变 `$false`，导致**误报失败**；只在 cmdlet / PS 7 的纯 PS 流程里 `$?` 才可靠。硬规则第 5 条 `curl.exe` 类调用尤其要注意。
7. 消色兜底：`NO_COLOR=1` 不是所有工具都认，老版本 GNU 命令显式加 `--color=never`（`ls --color=never`、`grep --color=never`），或 `git -c color.ui=never`。
8. PS 5.1 落盘编码：`>`/`Out-File` 默认写 UTF-16LE，随后要交给 bash 工具读取的文件用 `Out-File -Encoding utf8`；`gc` 读无 BOM 的 UTF-8 中文可能乱码，加 `-Encoding UTF8`。
9. 防交互卡死：安装/脚手架/审批类命令带非交互参数（`-y`/`--yes`/`--non-interactive`/`--accept-license`）；原生 exe 的参数被 PowerShell 解析搞乱（引号、`@`、`%` 等）时，用停止解析符号：`exe --% 原始参数`。
10. **`.ps1` 脚本编码**：目标宿主含 5.1 时，`.ps1` **必须带 BOM**（PS 7 无 BOM 也能自读，但带 BOM 更稳）。`Set-Content -Encoding UTF8` **在 5.1 下带 BOM、PS 7+ 下为无 BOM**，行为不一致，**跨版本稳妥写法一律用 .NET**：
    ```powershell
    [System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($true))  # $true = 带 BOM
    ```
    源码内状态标记、日志用纯 ASCII（如 `[PASS]`/`[FAIL]`），避免 `✅`/`❌`/中文在 GBK 环境下解析报错；执行前可用 `Get-Content -Encoding UTF8 -TotalCount 3 xxx.ps1` 核对首行未被转码。

## 错误处理分场景

- 探测性命令（存在性检查）：bash 用 `command -v x >/dev/null && echo ok`；PS 用 `Test-Path`、`Get-Command -ErrorAction SilentlyContinue`、`Select-String -Quiet`，返回布尔即可。
- 读操作：`SilentlyContinue` 合理。
- 写操作（删/移/安装/改配置）：`$ErrorActionPreference='Stop'` 快速失败；危险操作先 `-WhatIf` 预览再执行，外部命令再叠加硬规则第 6 条的退出码检查。不要静默吞错，agent 会误判成功，后续连环返工更费 token。
- 只报结论，不贴报错原文，除非用户要求排查。

## PowerShell 5.1 / 7 版本差异

- `&&`、`||` 命令链仅 PS 7+，5.1 直接语法报错，改用 `;` 串接；需短路判断成败时用 `if ($LASTEXITCODE -eq 0) {...}`（不要用 `$?`，5.1 下受 stderr 影响会误判，一律以 `$LASTEXITCODE` 为准）。
- 5.1 下原生命令合并 stderr（`exe 2>&1`）会把每行错误包成 ErrorRecord 刷红字，甚至触发 NativeCommandError 让 agent 误判失败，合并后立即字符串化：`exe 2>&1 | % { "$_" }`。PS 7 已大幅改善。
- `ConvertTo-Json` 默认 `-Depth 2`，深层对象被静默截断，agent 拿到残缺 JSON 会重试，显式给 `-Depth 5` 起步。
- 三元 `? :`（含 `condition ? a : b` 条件赋值，5.1 下用 `if/else`）、null 合并 `??` 仅 PS 7+。
- PS 7.3+ 可设 `$PSNativeCommandUseErrorActionPreference=$true`，让原生命令的 stderr 受 `-ErrorAction` 约束；5.1 无此能力。
- **查「当前宿主」版本才用 `$PSVersionTable.PSVersion.Major`**——它只反映此刻跑这段代码的 PS 进程是 5.1 还是 7。
- **`.ps1` 脚本默认宿主是 Windows PowerShell 5.1**：双击/右键运行 `.ps1`、或 `powershell -File xxx.ps1` 都用 5.1 解析，即便系统已装 PS 7 也不会自动切换。因此**「PS 7 是否可用」与「当前脚本跑在哪个版本」是两回事**：判断系统有没有装 PS 7、其语法（`&&`/`??`/`ForEach-Parallel`）能不能用时，**不要读 `$PSVersionTable`，应走 `pwsh` 子进程探测**——把待测代码作为字符串传给 `pwsh -NoProfile -Command '<单行代码>'`，从 stdout 回收结果。例：
  ```powershell
  $pwsh = "C:\Program Files\PowerShell\7\pwsh.exe"
  if (Test-Path $pwsh) {
      $oneLiner = @"
  `$major=`$PSVersionTable.PSVersion.Major; `$chain=(`$major -ge 7); try{`$x=`$null;`$y=`$x ?? 'ok';`$nc=`$true}catch{`$nc=`$false}; 'CHAIN='+`$chain+';NULLCOALESCE='+`$nc
  "@
      $res = & $pwsh -NoProfile -Command $oneLiner 2>&1 | Out-String
      # $res 含 "CHAIN=True;NULLCOALESCE=True" 即说明 PS7 语法可用
  }
  ```
  **注意 here-string 传参**：把代码作为**单行字符串** `$oneLiner` 传给子进程，变量要原样传到子进程侧、不被外层 PS 插值，需对 `$` 用反引号转义（转义后字面量 = `` `$var ``）；最终效果是子进程拿到形如 `$var` 的代码。同时**避免嵌套 `@'`/`@"`**，内层会被当作外层 here-string 的结束符，导致后续代码漏出即时执行。

## 命令等价映射

bash 首选，pwsh 备选。

| 任务 | bash（Git Bash/WSL） | pwsh / PS |
|---|---|---|
| 列目录 | `ls` | `gci -Name` |
| 搜内容 | `grep -n 'x' f` | `sls 'x' f \| % Line` |
| 匹配计数 | `grep -c 'x' f` | `(sls 'x' f \| Measure-Object \| % Count)` |
| 前 N 行 | `head -10 f` | `gc f -TotalCount 10` |
| 后 N 行 | `tail -10 f` | `gc f -Tail 10` |
| 递归找文件 | `find . -name '*.log'` | `gci -Recurse -File -Filter '*.log' \| % FullName` |
| 进程过滤 | `ps aux \| grep x` | `gps \| ? {$_.ProcessName -match 'x'} \| % ProcessName` |
| 统计行数（小文件） | `wc -l < f` | `(gc f).Count` |
| 统计行数（大文件） | `wc -l < f`（本身流式） | `gc f \| Measure-Object \| % Count` |
| 删除目录 | `rm -rf d` | `rm -Recurse -Force d` |
| 当前路径 | `pwd` | `(Get-Location).Path` |

大文件（数百 MB+）不要用 `(gc f).Count` 统计行数，会全量读入内存。`gc f | Measure-Object | % Count` 走管道逐行读，不占内存，效果和 LINQ 写法一样，也更短；极端场景才用 `[System.Linq.Enumerable]::Count[string]([IO.File]::ReadLines((Resolve-Path 'f').Path))`（显式 `[string]` 后 5.1/7 通用，省略类型参数只有 PS 7.3+ 能推断）。

详细对照、CLI 消噪全表、前后 token 对比、Shell 决策表、cmd 兜底速查、常见坑见 `references/shell-mappings.md`。

## 输出契约

最终交付给用户时，只写紧凑结论：
- 目标是否达成（是/否 + 关键数值）。
- 需要的字段列表，一行一个。
- 不粘贴冗长原始输出、不贴管道链路、不贴完整报错。

## 停止条件

- 用户明确要求完整/原始输出时才展开。
- 涉及系统级删除、权限修改、安装卸载等副作用，先向用户确认再执行。
- 命令疑似卡在 pager/交互提示（无输出、不返回）时，不要原样重试：下次调用加 `--no-pager`/`PAGER=cat`；需要喂空 stdin 时，bash 用 `< /dev/null`，PowerShell 不支持 `<` 重定向，用 `'' | exe`（或 `$null | exe`），也可走 cmd：`cmd /c 'exe < NUL'`。
- 遇到权限/策略阻止（ExecutionPolicy、受保护目录），报告原因并给替代路径，不要反复尝试同一被禁路径。
  - **ExecutionPolicy 拦截首选进程级绕过（零成本，不改系统策略）**：`pwsh -ExecutionPolicy Bypass -File x.ps1`，或当前会话内 `Set-ExecutionPolicy -Scope Process Bypass`；只有绕过仍失败才考虑 `cmd` 重写，**`cmd` 兜底放最后**。
