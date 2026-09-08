# Windows Shell 命令对照与省 token 细节

## 〇、Shell 路由决策表

先选 shell，再写命令。

| 场景 | 推荐 shell | 原因 |
|---|---|---|
| 平台有专用文件工具（读/搜/找） | 不用 shell | 输出已结构化，省 token 幅度大 |
| 项目在 WSL 文件系统、依赖 Linux 工具链（apt/gcc/docker/make） | WSL：`wsl -e bash -c '...'` | 真 Linux 环境，按 Linux 思维工作，出错概率低。默认用非登录 `-c`，启动快、环境干净；只有需要读 `~/.profile` 里的 PATH 等登录环境时才用 `-lc` |
| Windows 本地项目 + 通用文件/文本命令 | Git Bash：`bash -c '...'` | bash 语法不用翻译，出错概率低 |
| Windows 系统管理（服务/注册表/进程/ACL/事件日志） | PowerShell 7：`pwsh -NoProfile -Command '...'` | 对象管道 + cmdlet 是 Windows 管理的最强工具 |
| 只有自带 Windows PowerShell 5.1 | `powershell.exe` | 禁用 `&&`/`||`/三元，用 `;` 串接；判断成败用 `if ($LASTEXITCODE -eq 0)`，不用 `$?`（5.1 下受 stderr 影响会误判） |
| execution policy 拦截、PS 反复失败 | **先进程级绕过**：`pwsh -ExecutionPolicy Bypass -File x.ps1` 或 `Set-ExecutionPolicy -Scope Process Bypass`（零成本、不改系统策略）；只有绕过仍失败才用 **cmd 临时兜底**（最后手段） | cmd 无对象管道、单引号字面量、多行被截断，语法速查见第十节 |

环境探测 + 初始化合并，会话开头**一次调用**做完（省往返），记住结果后复用：

```powershell
Get-Command pwsh,bash,wsl,git -EA SilentlyContinue | % Source
$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8
$ProgressPreference = 'SilentlyContinue'
$env:NO_COLOR = '1'
```

注意事项：
- WSL：项目放在 WSL 文件系统内（`~/`），避免 `/mnt/c/`，跨文件系统 I/O 很慢。
- Git Bash：Windows 路径写 `/c/Users/...` 形式，注意引号转义。
- 混用时记住：WSL/Git Bash 里调的 `node/npm/python` 是 Linux/MinGW 视角，与 Windows 安装的工具链可能不是同一份。

## 一、会话初始化

会话开头跑一次。

pwsh / Windows PowerShell：

```powershell
$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8
$ProgressPreference = 'SilentlyContinue'
$env:NO_COLOR = '1'
```

bash / Git Bash / WSL：

```bash
export NO_COLOR=1 TERM=dumb PAGER=cat GIT_PAGER=cat GH_PAGER=cat
```

- UTF-8：避免中文/特殊符号乱码产生大量无效 token。
- `$ProgressPreference='SilentlyContinue'`：屏蔽下载进度条，非交互终端里进度条会刷成垃圾字符。
- `NO_COLOR=1`：关 ANSI 颜色码，省不少 token。
- `PAGER=cat`/`GIT_PAGER=cat`/`GH_PAGER=cat`：防 git、gh 等命令卡 pager 等按键，agent 会卡住不返回。
- `TERM=dumb`：让 TUI/颜色程序退化为纯文本；极少数安装器/TUI 在 dumb 终端下报"终端能力"错误，遇到就去掉 `TERM=dumb`，保留其余项。
- 个别老 Win32 程序只看控制台代码页，上述设置后中文仍乱码，先执行 `chcp 65001 > $null` 再重跑（`> $null` 抑制自身回显行）。
- 消色兜底：`NO_COLOR=1` 不是所有工具都认，老版本 GNU 命令显式加 `--color=never`（`ls --color=never`、`grep --color=never`），git 用 `git -c color.ui=never`。

## 二、PowerShell 别名与简写

短则省 token。

- `gci` = `Get-ChildItem`，`gc` = `Get-Content`，`gps` = `Get-Process`
- `sls` = `Select-String`，`?` = `Where-Object`，`%` = `ForEach-Object`
- `rm` = `Remove-Item`，`mv` = `Move-Item`，`cp` = `Copy-Item`
- `curl` = `Invoke-WebRequest`（**坑，见第八节**，仅 PS 5.1 有此别名），原生 curl 要写 `curl.exe`
- `where` = `Where-Object`（**坑，见第八节**），查命令位置要写 `where.exe`

`?`、`%` 只在单行命令里用，别写进脚本文件。

## 三、常用任务 → 最短命令

### 列目录，只取名字
```powershell
gci -Name
```

### 按模式找文件，只要完整路径
```powershell
gci -Recurse -File -Filter '*.log' | % { $_.FullName }
```
`-File` 排除同名目录；结果不可预估时挂 `| Select-Object -First <n>`。

### 搜索文件内容，只要含匹配的行
```powershell
sls '错误码' app.log | % { $_.Line }
```

### 只要匹配数量（流式，大文件也安全）
```powershell
sls '错误码' app.log | Measure-Object | % Count
```
> 注：直接取 `.Count` 会把全部匹配对象读进内存；**大文件务必用上面的流式写法**（`sls ... | Measure-Object | % Count`）。bash 侧 `grep -c` 本身流式，无需替代。

### 只看是否匹配（布尔探测）
```powershell
sls -Quiet '错误码' app.log
```

### 读取文件前 N 行 / 末 N 行
```powershell
gc app.log -TotalCount 10
gc app.log -Tail 10
```
PS 5.1 读无 BOM 的 UTF-8 中文可能乱码，加 `-Encoding UTF8`。

### 数行数

小文件（一次性，简单）：
```powershell
(Get-Content app.log).Count
```

大文件（数百 MB+）别用上面的写法，`(gc f).Count` 会把整个文件读进内存，卡很久且最终刷屏。管道写法逐行读、不占内存，效果等同于 LINQ，还更短：
```powershell
gc app.log | Measure-Object | % Count
```
只有极端场景才需要 LINQ：
```powershell
[System.Linq.Enumerable]::Count[string]([IO.File]::ReadLines((Resolve-Path 'app.log').Path))
```
显式 `[string]` 后 5.1/7 通用；省略类型参数只有 PS 7.3+ 能自动推断。
bash 侧 `wc -l` 本身是流式，无需替代。

### 看进程，只要名字 / 探测某进程是否存在
```powershell
gps | % ProcessName
[bool](gps -Name chrome -ErrorAction SilentlyContinue)
```

### 文件夹占用排行（压缩输出）
```powershell
gci -Directory | % { [pscustomobject]@{Name=$_.Name; MB=[math]::Round((gci $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum/1MB,1)} } | Sort MB -Descending
```
仅适合小目录，大目录递归扫描很慢。

### 环境变量 / 当前目录
```powershell
$env:PATH -split ';' | sls python
(Get-Location).Path
```
PATH 整行可能几千字符，按需过滤或 `| Select-Object -First <n>` 截断，别直接打全量。

### Windows 管理最短命令模板（路由到 PS 后的常用片段）

```powershell
# 端口占用 → PID
Get-NetTCPConnection -LocalPort 8080 -EA SilentlyContinue | % OwningProcess
# 是否管理员（返回布尔）
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole('Administrator')
# 错误事件只取 10 条核心字段
Get-WinEvent -FilterHashtable @{LogName='Application';Level=2} -MaxEvents 10 | % { "$($_.TimeCreated) $($_.ProviderName): $($_.Message)" }
Get-FileHash f -Algorithm MD5 | % Hash        # 校验和
Expand-Archive x.zip -DestinationPath d       # 解压（Win10+ 也可用 tar -xf，更短）
gsv x | % Status;  kill -Name x -Force        # gsv/kill 都是内置别名
```

PS 5.1 在 Server Core 等无 IE 环境用 `Invoke-WebRequest` 会报 IE 引擎不可用，加 `-UseBasicParsing`（PS 7 不需要）；纯下载直接 `curl.exe -sL -o f url` 更短。

### PS 输出压缩补充

- 丢弃输出三选一：`$null = ...` / `... | Out-Null` / `... > $null`。
- **`$FormatEnumerationLimit` 默认 4**：对象的集合属性只显示 4 项就 `+ N more`，agent 极易误判"只有 4 个"，需要完整时先设 `-1`：`$FormatEnumerationLimit = -1`。
- 集合压成一行：`(gci).Name -join ','`。
- 别名速查：`select`/`sort`/`measure`/`gsv`/`kill`/`ii`。

## 四、外部 CLI 消噪全表

| 工具 | 默认输出 | 省 token 写法 |
|---|---|---|
| git log | `git log`（约 624 token） | `git --no-pager log --oneline -5`（约 55） |
| git status | 含大段提示文案 | `git status --short`（或 `-sb` 带分支信息） |
| git diff | `git diff`（约 2400+ token） | `git --no-pager diff --stat`（约 320） |
| git 含中文路径 | 文件名变 `\346\226\207…` 八进制转义 | 加 `-c core.quotePath=false` |
| git 任意子命令 | 可能卡 pager | `git --no-pager ...` 或 `PAGER=cat` |
| gh | 可能卡 pager | `gh --no-pager ...` 或 `GH_PAGER=cat` |
| npm install | 进度+审计+募捐信息 | 排查期：`npm install --no-audit --no-fund --no-progress`（保留错误/警告）；确认无错再叠 `--silent`（会吞错误，Agent 易误判成功） |
| mocha | 全部用例输出（约 3100+ token） | `mocha --reporter min`（仅摘要，失败保留错误；**`min` 依赖 TTY 清屏，非交互管道下可能吐光标转义序列，此时改用 `--reporter dot`；`dot` 每用例打一个点，量大仍费 token**） |
| jest | 测试代码里的 console 输出 + 冗长报告 | `jest --silent`（**只压测试内的 `console.*`，不压结果报告**，配 `2>&1 \| tail -20` 取摘要；`--reporter=dot` 是 mocha/vitest 参数，jest 不认） |
| vitest | 全部用例输出 | `vitest run --silent --reporter=dot`（vitest 没有 `min` reporter，那是 mocha 的） |
| pip | 下载/依赖树/版本提示 | `pip install -q --disable-pip-version-check ...` |
| pytest | 全量 traceback | `pytest -q --tb=short`（还要更短：`--tb=line -rN`，每失败只留一行定位） |
| tsc（TypeScript） | pretty 模式重复打印代码帧、带颜色 | `tsc --pretty false` |
| eslint | stylish 多行报告 | `eslint -f unix`（或 `--format compact`） |
| cargo | 编译过程逐行输出 | `cargo build -q` |
| pnpm / yarn | 安装进度与募捐信息 | `pnpm install --reporter=silent` / `yarn -s` |
| git log 装饰 | 默认带 branch/tag/HEAD 装饰 | 加 `--no-decorate`；只取当前分支名用 `git branch --show-current`（短于 `rev-parse --abbrev-ref HEAD`） |
| gh 字段裁剪 | 默认返回完整 JSON | `gh --jq '<expr>'` 直接裁字段（bash 侧处理 JSON 用 `jq -rc`） |
| dotnet | logo+详细 MSBuild 输出 | `dotnet build --nologo -v q` |
| docker build | tty 进度动画刷屏 | `docker build --progress quiet`（成功仅输出镜像 ID；`plain` 是详尽原始日志，输出更多，不用于省 token） |
| make / gradle / maven | 全量构建日志 | `make -s` / `gradle --console=plain` / `mvn -ntp`（**别用 `-q`**：会把测试摘要和 BUILD SUCCESS 一起吞掉，agent 反而失去成败判据；`-q` 只留给「只要错误」的场景） |
| 安装/脚手架命令 | 交互提示等输入卡死 | 带 `-y`/`--yes`/`--non-interactive`/`--accept-license` |
| curl（HTTP 请求） | PS 5.1 里 `curl` 是 Invoke-WebRequest 别名，输出响应对象大堆属性 | 写 `curl.exe` 调用原生 curl，或用 `irm`（Invoke-RestMethod）拿纯数据 |
| 命令退出码 / 只要退出码 | 全部输出 | PS：`cmd; if ($LASTEXITCODE -ne 0) { throw }` 或 `cmd; exit $LASTEXITCODE`；bash：`cmd >/dev/null 2>&1 && echo OK \|\| echo FAIL` |
| 测试只看失败 | 全量 | bash：`pytest -q 2>&1 \| grep -E 'FAIL\|ERROR' \| head -30` |
| 消色兜底 | 老 GNU 工具不认 NO_COLOR | `ls/grep --color=never`，`git -c color.ui=never` |

表中 token 数值是示例量级，实际随提交历史、用例数量变化，重点看数量级差异。

## 五、前后对比示例

任务：确认最近 3 次提交改了哪些文件。

啰嗦（约 624 token）：
```bash
git log
```

紧凑（约 55 token）：
```bash
git --no-pager log --oneline -3 && git --no-pager diff --stat HEAD~3..HEAD | tail -5
```

任务：跑测试看有没有失败。

啰嗦（约 3100 token）：
```bash
npm test
```

紧凑（约 150 token，按项目框架选其一）：

```bash
# mocha 项目
npm test -- --reporter min 2>&1 | tail -20
```

```bash
# jest 项目
npm test -- --silent 2>&1 | tail -20
```

```bash
# vitest 项目
npm test -- --silent --reporter=dot 2>&1 | tail -20
```

任务：在日志里找报错。

啰嗦（整个文件刷屏）：
```powershell
Get-Content app.log
```

紧凑：
```powershell
sls 'ERROR' app.log | Select-Object -First 20 | % { $_.Line }
```

先想清楚 agent 需要什么（pass/fail + 失败摘要 / 最近 N 条 / 匹配行），再让命令只产出那部分。

## 六、减少 token 的通用手法

1. 只选字段：`| % $_.<属性>` 比 `Format-List *` 省约 80%。
2. `Select-Object -First <n>` / `head -n` 截断长列表（硬规则）。
3. `-ExpandProperty` / `% { $_.X }` 把对象压成纯文本行。
4. 需要 JSON：`ConvertTo-Json -Compress -Depth 5`，先 `Select-Object -Property <字段>` 裁剪。
5. 避免 `Format-*` 展开、`Get-Member`、`-Verbose`。
6. 多条相关命令用 `&&`（bash/pwsh7）或 `;`（PS5.1）合并成一次调用，省来回往返的固定开销。
7. 等待构建/文件变化时，用哈希或大小比较代替反复全量重读。

## 七、PowerShell 5.1 / 7 差异速查

| 特性 | PS 5.1 | PS 7+ |
|---|---|---|
| `&&` / `||` 命令链 | 语法报错 | 可用 |
| 三元 `? :`（含 `condition ? a : b` 条件赋值，5.1 用 `if/else`）、null 合并 `??` | 不支持 | 可用 |
| `ConvertTo-Json` 默认深度 | `-Depth 2`，深层静默截断，显式给 `-Depth 5` | 同左 |
| `ForEach-Object -Parallel` | 无 | 有 |
| 原生 stderr 受 `-ErrorAction` 约束 | 无 | `$PSNativeCommandUseErrorActionPreference=$true`（7.3+） |
| 泛型方法类型推断（如 `[Linq.Enumerable]::Count(...)`） | 无，需显式写 `Count[string]` | 7.3+ 可推断 |
| `>` 重定向默认编码 | UTF-16LE | utf8NoBOM |
| 查**当前宿主**版本 | `$PSVersionTable.PSVersion.Major`（**只反映当前进程，不能据此判断系统是否装了 PS7**；探测 PS7 语法走 `pwsh` 子进程，见第八节） | 同左 |

## 八、常见坑

- **`curl` 别名陷阱**：PowerShell 5.1 里 `curl` 是 `Invoke-WebRequest` 别名（PS 7 已移除，统一写 `curl.exe` 两版通用），返回响应对象，默认输出 `StatusCode`/`Headers`/`RawContent` 等大量冗余属性，轻松刷爆 token。需要 HTTP 请求一律用 `curl.exe`（系统原生 curl）或 `irm`（Invoke-RestMethod）。
- **`where` 别名陷阱**：PS 里 `where` 是 `Where-Object` 的别名，`where git` 不会告诉你 git 在哪，只会按过滤语义处理；查命令位置用 `where.exe git` 或 `Get-Command git | % Source`。
- **PS 5.1 落盘编码**：`>`/`Out-File` 默认写 UTF-16LE，随后交给 bash 工具（`wc`/`grep` 等）读取会乱码或被当二进制；跨工具传递的文件用 `Out-File -Encoding utf8`。`gc` 读无 BOM 的 UTF-8 中文可能乱码，加 `-Encoding UTF8`。
- **参数被 PS 解析改写**：原生 exe 的参数含引号、`@`、`%` 等被 PowerShell 重写导致行为异常时，用停止解析符号：`findstr --% /c:"a b" f.txt`。
- **git 中文文件名变八进制**：默认 `core.quotePath=true` 会把非 ASCII 路径转义成 `\346\226\207…`，费 token 还难读，加 `-c core.quotePath=false`。
- 编码乱码：中文输出乱码会变成大量无效 token，先跑会话初始化。
- pager/交互卡死：命令无输出不返回，多半是进了 pager 或等用户输入，加 `--no-pager`/`PAGER=cat` 重试或补非交互参数（`-y`/`--yes`/`--non-interactive`），不要原样重发。需要喂空 stdin 时：bash 用 `< /dev/null`，PowerShell **不支持 `<` 重定向**，用 `'' | exe`（或 `$null | exe`），也可走 cmd：`cmd /c 'exe < NUL'`。
- PS 5.1 合并 stderr 刷红字：5.1 下 `exe 2>&1` 会把 stderr 每行包成 ErrorRecord，显示红字甚至抛 NativeCommandError，agent 容易误判命令失败。合并后立即字符串化：`exe 2>&1 | % { "$_" } | Select-Object -First 50`。PS 7 已大幅改善，5.1 问题比较多。
- **外部命令退出码**：`$ErrorActionPreference='Stop'` 管不到外部 exe 的 stderr，出错退出也不抛异常。写操作后务必检查退出码：一律以 `$LASTEXITCODE` 为准、紧贴外部命令后立刻读——它**只会被原生外部命令覆盖，PowerShell cmdlet 不会重置它**，中间再跑别的 exe 就会被冲掉；**不要用 `$?` 判断外部命令**（5.1 下外部命令往 stderr 写内容就会让 `$?` 误报 `$false`，详见 SKILL 硬规则 6）。透传退出码用 `cmd; exit $LASTEXITCODE`。
- 管道对象 vs 字符串：`Select-String` 返回对象，纯文本取 `.Line`。
- `Select-String -Quiet` 只给布尔，适合探测，写 `-Quiet` 全称，不用 `-Q` 缩写，避免误解。
- 递归扫描慢且刷屏：用 `-Filter` 而非 `Where-Object` 全扫，输出必挂 `-First`/`head`。
- 命令存在性探测：`Get-Command cmd -ErrorAction SilentlyContinue` / bash `command -v cmd`。
- WSL 跨文件系统：`/mnt/c` 下操作慢，项目在 WSL 内就放 `~/`。
- 写操作别静默：删/移/安装用 `$ErrorActionPreference='Stop'` 快速失败 + `-WhatIf` 预演，吞错会导致误判成功、连环返工。
- **`.ps1` 脚本编码坑**：`.ps1` 文件若以 GBK/ANSI 保存，中文与 `✅`/`❌` 等特殊符号被当成代码字节解析，报 `UnexpectedToken` / `缺少右“）”`。**目标宿主含 5.1 时必须带 BOM**；`Set-Content -Encoding UTF8` 在 5.1 下带 BOM、PS 7+ 下为无 BOM，跨版本行为不一致，**稳妥写法一律用 .NET**：
  ```powershell
  [System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($true))  # 5.1/7 下均带 BOM
  ```
  状态标记/日志用纯 ASCII（`[PASS]`/`[FAIL]`），执行前用 `Get-Content -Encoding UTF8 -TotalCount 3` 核对未被转码。
- **`.ps1` 默认宿主是 5.1**：双击/右键运行、或 `powershell -File` 都走 5.1，即使已装 PS 7 也不会自动切换。探测「PS7 语法是否可用（`&&`/`??`/`ForEach-Parallel`）」不要读 `$PSVersionTable`，应通过 `pwsh -NoProfile -Command '<单行代码>'` 子进程回收结果；传代码用**单行 `$oneLiner`**——若要让 `$var` 这类变量**原样传到子进程、不被外层 PS 插值**，需对 `$` 用反引号转义（转义后字面量写作 `` `$var ``），子进程最终拿到 `$var` 这样的源代码；**避免嵌套 `@'`/`@"`**（内层会被当作外层 here-string 结束符，导致后续代码漏出即时执行）。
- **`&&`/`??` 探测务必在 pwsh 子进程内执行**：5.1 下 `$PSVersionTable.PSVersion.Major -ge 7` 恒为 `$false`，据此判断会误判「PS7 未安装」；正确做法是把 `??`、`&&` 的真语法作为字符串放进 `pwsh -NoProfile -Command` 里实际跑一次，看是否抛语法异常。

### 防卡死：非交互参数速查（命中率高，务必带齐）

安装/包管理类命令**不喂 stdin、不加非交互参数必卡**，按工具记熟：

| 场景 | 防卡死写法 |
|---|---|
| ssh / scp | `ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes ...`（否则卡 `yes/no` 或密码输入） |
| terraform | `terraform apply -no-color -input=false`（**`-input=false` 是防卡死关键**） |
| WSL / apt 装包 | `sudo DEBIAN_FRONTEND=noninteractive apt-get install -y ...` |
| winget 装软件 | `winget install x --accept-package-agreements --accept-source-agreements --disable-interactivity` |
| MSI 静默 | `msiexec /i x.msi /qn /norestart` |
| 命令限时 | Git Bash：`timeout 30 cmd`；**PS / cmd 里的 `timeout.exe` 是「等待按键」不是限时**，PS 限时要用 `Start-Job { ... } \| Wait-Job -Timeout 30`（这个反差务必记牢） |

### 引号与跨 shell 互操作坑（Windows 真机高频）

- **Git Bash 路径自动转换**：调 `cmd /c` 必须写 `cmd //c`（`/c` 会被改写成 `C:/`）；排除转换用 `MSYS_NO_PATHCONV=1` 或 `MSYS2_ARG_CONV_EXCL='*'`。
- **Git Bash 调 Windows 原生命令，参数是 Windows 风格**：`ping -n 2`（不是 `-c`）、`netstat -ano | findstr`。
- **`wsl -e bash -c '...'` 三层引号**：外层 PowerShell 用单引号（`$` 不被插值），bash 内层用双引号，bash 单引号用 `'\''` 转义。
- **`wsl -l -v` 输出是 UTF-16LE**：管道捕获全是 NUL 字符，先 `[Console]::OutputEncoding=[Text.Encoding]::Unicode` 再调。
- **PS 命令名/路径含空格必须用 `&` 调用**；单引号字符串里的单引号写两个：`'it''s'`。

## 九、何时一定保留完整输出

- 用户明确要原始输出/完整日志。
- 排障需要完整上下文时，但仍先 `-First`/`-Tail`/`head`/`tail` 控制篇幅。

## 十、cmd 兜底速查（仅兜底时用）

- 列目录只要名字：`dir /b`
- 搜内容：`findstr /n /c:"关键词" app.log`
- 读文件：`type app.log`（无内置截断，长文件先落盘再分段读）
- 退出码：`cmd && next` / `cmd || fallback` 可用；读数值用 `echo %ERRORLEVEL%`
- 转义与引号：参数用双引号，单引号是字面量；`& | < > ^` 作字面量时用 `^` 转义，`%` 在批处理脚本里写 `%%`
- 保持单行：多行命令经 agent 传入容易被截断

## 附：工具层互补（可选）

这份文档是行为层，教 agent 怎么选 shell、怎么写命令。如需工具层兜底，可叠加输出压缩类工具，在输出回流 agent 前过滤进度条/重复行，二者互补不冲突。优化的是整个任务的成本，不是单条命令最短，过度精简导致缺信息再补命令，反而更亏。
