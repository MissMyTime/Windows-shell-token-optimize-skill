# Windows Shell 命令对照与省 token 细节

## 〇、Shell 路由决策表

先选 shell，再写命令。

| 场景 | 推荐 shell | 原因 |
|---|---|---|
| 平台有专用文件工具（读/搜/找） | 不用 shell | 输出已结构化，省 token 幅度大 |
| 项目在 WSL 文件系统、依赖 Linux 工具链（apt/gcc/docker/make） | WSL2：`wsl -e bash -lc '...'` | 真 Linux 环境，按 Linux 思维工作，出错概率低 |
| Windows 本地项目 + 通用文件/文本命令 | Git Bash：`bash -c '...'` | bash 语法不用翻译，出错概率低 |
| Windows 系统管理（服务/注册表/进程/ACL/事件日志） | PowerShell 7：`pwsh -NoProfile -Command '...'` | 对象管道 + cmdlet 是 Windows 管理的最强工具 |
| 只有自带 Windows PowerShell 5.1 | `powershell.exe` | 禁用 `&&`/`||`/三元，用 `;` 与 `if ($?)` |
| execution policy 拦截、PS 反复失败 | cmd 临时兜底 | 无对象管道、单引号字面量、多行被截断 |

环境探测，会话开头跑一次，记住结果：

```powershell
Get-Command pwsh,bash,wsl,git -ErrorAction SilentlyContinue | % Source
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
export NO_COLOR=1 TERM=dumb PAGER=cat GIT_PAGER=cat
```

- UTF-8：避免中文/特殊符号乱码产生大量无效 token。
- `$ProgressPreference='SilentlyContinue'`：屏蔽下载进度条，非交互终端里进度条会刷成垃圾字符。
- `NO_COLOR=1`：关 ANSI 颜色码，省不少 token。
- `PAGER=cat`：防 git 等命令卡 pager 等按键，agent 会卡住不返回。
- `TERM=dumb`：让 TUI/颜色程序退化为纯文本；极少数安装器/TUI 在 dumb 终端下报"终端能力"错误，遇到就去掉 `TERM=dumb`，保留其余三项。
- 个别老 Win32 程序只看控制台代码页，上述设置后中文仍乱码，先执行 `chcp 65001` 再重跑。

## 二、PowerShell 别名与简写

短则省 token。

- `gci` = `Get-ChildItem`，`gc` = `Get-Content`，`gps` = `Get-Process`
- `sls` = `Select-String`，`?` = `Where-Object`，`%` = `ForEach-Object`
- `rm` = `Remove-Item`，`mv` = `Move-Item`，`cp` = `Copy-Item`

`?`、`%` 只在单行命令里用，别写进脚本文件。

## 三、常用任务 → 最短命令

### 列目录，只取名字
```powershell
gci -Name
```

### 按模式找文件，只要完整路径
```powershell
gci -Recurse -Filter '*.log' | % { $_.FullName }
```

### 搜索文件内容，只要含匹配的行
```powershell
sls '错误码' app.log | % { $_.Line }
```

### 只要匹配数量
```powershell
(sls '错误码' app.log).Count
```

### 只看是否匹配（布尔探测）
```powershell
sls -Quiet '错误码' app.log
```

### 读取文件前 N 行 / 末 N 行
```powershell
gc app.log -TotalCount 10
gc app.log -Tail 10
```

### 数行数

小文件：
```powershell
(Get-Content app.log).Count
```

大文件（数百 MB+）别用上面的写法，`(gc f).Count` 会把整个文件读进内存，卡很久且最终刷屏，改用流式：
```powershell
[Linq.Enumerable]::Count([IO.File]::ReadLines((Resolve-Path 'app.log').Path))
```
bash 侧 `wc -l` 本身是流式，无需替代。

### 看进程，只要名字 / 探测某进程是否存在
```powershell
gps | % ProcessName
[bool](gps -Name chrome -ErrorAction SilentlyContinue)
```

### 文件夹占用排行（压缩输出）
```powershell
gci -Directory | % { [pscustomobject]@{Name=$_.Name; MB=[math]::Round((gci $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum/1MB,1)} } | Sort MB -Descending
```

### 环境变量 / 当前目录
```powershell
$env:PATH
(Get-Location).Path
```

## 四、外部 CLI 消噪全表

| 工具 | 默认输出 | 省 token 写法 |
|---|---|---|
| git log | `git log`（约 624 token） | `git --no-pager log --oneline -5`（约 55） |
| git diff | `git diff`（约 2400+ token） | `git --no-pager diff --stat`（约 320） |
| git 任意子命令 | 可能卡 pager | `git --no-pager ...` 或 `PAGER=cat` |
| gh | 可能卡 pager | `gh --no-pager ...` |
| npm install | 进度+审计+募捐信息 | `npm install --silent --no-audit --no-fund` |
| mocha | 全部用例输出（约 3100+ token） | `mocha --reporter=dot`（约 150） |
| jest | 全部用例输出 | `jest --silent`（`--reporter=dot` 是 mocha/vitest 参数，jest 不认） |
| vitest | 全部用例输出 | `vitest run --reporter=dot` |
| pip | 下载/依赖树 | `pip install -q ...` |
| pytest | 全量 traceback | `pytest -q --tb=short` |
| dotnet | logo+详细 MSBuild 输出 | `dotnet build --nologo -v q` |
| docker build | tty 进度动画刷屏 | `docker build --progress plain` |
| make / gradle / maven | 全量构建日志 | `make -s` / `gradle -q` / `mvn -q` |
| 只要退出码 | 全部输出 | bash：`cmd >/dev/null 2>&1 && echo OK \|\| echo FAIL` |
| 测试只看失败 | 全量 | bash：`pytest -q 2>&1 \| grep -E 'FAIL\|ERROR' \| head -30` |

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

紧凑（约 150 token，按底层框架选参数）：
```bash
# mocha 用 --reporter=dot；jest 用 jest --silent；vitest 用 vitest run --reporter=dot
npm test --silent -- --reporter=dot 2>&1 | tail -20
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
| 三元 `?:`、null 合并 `??` | 不支持 | 可用 |
| `ConvertTo-Json` 默认深度 | `-Depth 2`，深层静默截断，显式给 `-Depth 5` | 同左 |
| `ForEach-Object -Parallel` | 无 | 有 |
| 查版本 | `$PSVersionTable.PSVersion.Major` | 同左 |

## 八、常见坑

- 编码乱码：中文输出乱码会变成大量无效 token，先跑会话初始化。
- pager 卡死：命令无输出不返回，多半是进了 pager/交互提示，加 `--no-pager`/`PAGER=cat` 重试，不要原样重发。
- PS 5.1 合并 stderr 刷红字：5.1 下 `exe 2>&1` 会把 stderr 每行包成 ErrorRecord，显示红字甚至抛 NativeCommandError，agent 容易误判命令失败。合并后立即字符串化：`exe 2>&1 | % { "$_" } | Select-Object -First 50`。PS 7 已大幅改善，5.1 问题比较多。
- 管道对象 vs 字符串：`Select-String` 返回对象，纯文本取 `.Line`。
- `Select-String -Quiet` 只给布尔，适合探测，写 `-Quiet` 全称，不用 `-Q` 缩写，避免误解。
- 递归扫描慢且刷屏：用 `-Filter` 而非 `Where-Object` 全扫，输出必挂 `-First`/`head`。
- 命令存在性探测：`Get-Command cmd -ErrorAction SilentlyContinue` / bash `command -v cmd`。
- WSL 跨文件系统：`/mnt/c` 下操作慢，项目在 WSL 内就放 `~/`。
- 写操作别静默：删/移/安装用 `Stop` 快速失败 + `-WhatIf` 预演，吞错会导致误判成功、连环返工。

## 九、何时一定保留完整输出

- 用户明确要原始输出/完整日志。
- 排障需要完整上下文时，但仍先 `-First`/`-Tail`/`head`/`tail` 控制篇幅。

## 附：工具层互补（可选）

这份文档是行为层，教 agent 怎么选 shell、怎么写命令。如需工具层兜底，可叠加输出压缩类工具，在输出回流 agent 前过滤进度条/重复行，二者互补不冲突。优化的是整个任务的成本，不是单条命令最短，过度精简导致缺信息再补命令，反而更亏。
