# Windows‑shell‑token‑optimize‑skill

一套面向AI Agent的Windows Shell执行规则集。
定义Shell路由选择逻辑、环境初始化约束、错误处理、版本差异、命令映射表、执行前检查清单与反模式。
目标：减少Token消耗，规避PowerShell 5.1 / 7、cmd、Git‑Bash、WSL环境下各类执行异常。

## 文件说明
- `SKILL.md`：主规则文件，完整Skill定义。包含frontmatter标识，可直接交给Agent加载使用。
- `references/shell‑mappings.md`：命令映射附表，规则中会通过相对路径引用此文件。

## 环境覆盖
- PowerShell 5.1
- PowerShell 7
- Windows cmd
- Git‑Bash
- WSL

## 使用方式

### 安装

把整个仓库放进 skills 目录，**目录名必须等于 frontmatter 里的 `name`**，且 `references/` 要跟着一起放：


