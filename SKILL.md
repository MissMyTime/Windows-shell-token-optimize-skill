---
name: windows-shell-token-saver
title: Windows Shell 省 Token 妙招
description: 在 Windows 终端（PowerShell/cmd/Git Bash/WSL）执行命令时，通过 Shell 路由、紧凑输出、单条合并命令、UTF-8 编码、CLI 消噪与分场景错误处理，把 token 消耗压到接近 macOS/zsh 的水平。
when_to_use: 当在 Windows 上让 agent 运行终端命令（PowerShell/cmd/Git Bash/WSL）时使用，尤其是命令输出冗长、需要多层管道、存在编码乱码、需要反复探测、shell 选择不定时。不用于网页浏览、文档编辑、表格编辑等非 shell 任务。
---

# Windows Shell 省 Token 妙招

## 目标

Windows 终端默认输出冗长，shell 种类多且语法和 macOS/zsh 差异大，编码容易乱码，错误信息啰嗦，agent 每条命令都要消耗大量 token。这份文档整理了路由和执行两层默认行为，让 Windows 终端命令选得对、输出短、一次完成、无乱码。

## 第 0 层：先路由，再执行

按顺序决策，命中即停：

1. 平台提供专用文件工具（读文件/搜内容/找文件）时，用专用工具，不要启动 shell 用 `sls/gci/gc/grep/find` 代替。输出已经结构化，省 token 幅度很大。
2. 必须用 shell 时，按场景选：
   - 项目在 WSL 文件系统，或依赖 Linux 工具链（apt/gcc/docker/make）→ WSL：`wsl -e bash -lc '...'`
   - Windows 本地项目、通用文件/文本命令 → Git Bash：`bash -c '...'`，bash 语法不用翻译，出错概率低
   - Windows 系统管理（服务/注册表/进程/ACL/事件日志）→ PowerShell 7：`pwsh -NoProfile -Command '...'`
   - 只有自带 5.1 → `powershell.exe`，但禁用 `&&`（见版本差异节）
   - cmd 仅兜底，execution policy 拦截或 PS 反复失败时临时用
3. 会话开头探测一次环境并记住结果，不要每条命令都探测：

```powershell
Get-Command pwsh,bash,wsl,git -ErrorAction SilentlyContinue | % Source
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
$OutputEncoding=[Console]::OutputEncoding=[Text.Encoding]::UTF8; $ProgressPreference='SilentlyContinue'; $env:NO_COLOR='1'
```

bash / Git Bash / WSL：

```bash
export NO_COLOR=1 TERM=dumb PAGER=cat GIT_PAGER=cat
```

- `$ProgressPreference='SilentlyContinue'`：屏蔽 Invoke-WebRequest 等下载进度条，进度条在非交互环境会刷出大量垃圾字符。
- `NO_COLOR=1`：去掉 ANSI 颜色转义码，颜色码会让输出 token 膨胀不少。
- `PAGER=cat`：防止 git 等命令进入 pager 等待人工按键，agent 撞上 pager 会卡住不返回。
- `TERM=dumb` 让 TUI/颜色程序退化为纯文本；极少数安装器在 dumb 终端下报"终端能力"错误，遇到就去掉 `TERM=dumb` 保留其余。
- 个别老 Win32 程序只看控制台代码页，上述设置后中文仍乱码，先执行 `chcp 65001` 再重跑。

## 硬规则

1. 输出行数不可预估的命令，必须挂 `| Select-Object -First 50`（bash：`| head -50`）兜底，确认不够再加大。不封顶的命令很容易把上下文撑爆。
2. 输出不可控先落盘再选读：`cmd > out.txt 2>&1`，然后 `wc -l out.txt`（或 `gc out.txt -TotalCount/-Tail`）分段读需要的部分。
3. 外部 CLI 必须消噪：git/gh 加 `--no-pager`；npm install 加 `--silent --no-audit --no-fund`；dotnet 加 `--nologo -v q`；docker build 加 `--progress plain`；测试只回传失败摘要，`pytest -q --tb=short`、mocha `--reporter=dot`、jest `--silent`、vitest `vitest run --reporter=dot`。`--reporter=dot` 是 mocha/vitest 参数，jest 不认，按框架选择。全表见 references。
4. 只要概览不要全量：diff 用 `git --no-pager diff --stat`，日志用 `git --no-pager log --oneline -5`。

## 错误处理分场景

- 探测性命令（存在性检查）：bash 用 `command -v x >/dev/null && echo ok`；PS 用 `Test-Path`、`Get-Command -ErrorAction SilentlyContinue`、`Select-String -Quiet`，返回布尔即可。
- 读操作：`SilentlyContinue` 合理。
- 写操作（删/移/安装/改配置）：`$ErrorActionPreference='Stop'` 快速失败；危险操作先 `-WhatIf` 预览再执行。不要静默吞错，agent 会误判成功，后续连环返工更费 token。
- 只报结论，不贴报错原文，除非用户要求排查。

## PowerShell 5.1 / 7 版本差异

- `&&`、`||` 命令链仅 PS 7+，5.1 直接语法报错，改用 `;` 或 `if ($?) {...}`。
- 5.1 下原生命令合并 stderr（`exe 2>&1`）会把每行错误包成 ErrorRecord 刷红字，甚至触发 NativeCommandError 让 agent 误判失败，合并后立即字符串化：`exe 2>&1 | % { "$_" }`。PS 7 已大幅改善。
- `ConvertTo-Json` 默认 `-Depth 2`，深层对象被静默截断，agent 拿到残缺 JSON 会重试，显式给 `-Depth 5` 起步。
- 三元 `?:`、null 合并 `??` 仅 PS 7+。
- 不确定版本先查：`$PSVersionTable.PSVersion.Major`。

## 命令等价映射

bash 首选，pwsh 备选。

| 任务 | bash（Git Bash/WSL） | pwsh / PS |
|---|---|---|
| 列目录 | `ls` | `gci -Name` |
| 搜内容 | `grep -n 'x' f` | `sls 'x' f \| % Line` |
| 匹配计数 | `grep -c 'x' f` | `(sls 'x' f).Count` |
| 前 N 行 | `head -10 f` | `gc f -TotalCount 10` |
| 后 N 行 | `tail -10 f` | `gc f -Tail 10` |
| 递归找文件 | `find . -name '*.log'` | `gci -Recurse -Filter '*.log' \| % FullName` |
| 进程过滤 | `ps aux \| grep x` | `gps \| ? {$_.ProcessName -match 'x'} \| % ProcessName` |
| 统计行数 | `wc -l < f` | `(gc f).Count` |
| 删除目录 | `rm -rf d` | `rm -Recurse -Force d` |
| 当前路径 | `pwd` | `(Get-Location).Path` |

大文件（数百 MB+）不要用 `(gc f).Count` 统计行数，会全量读入内存，用流式写法，见 references 第三节。

详细对照、CLI 消噪全表、前后 token 对比、Shell 决策表、常见坑见 `references/shell-mappings.md`。

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
