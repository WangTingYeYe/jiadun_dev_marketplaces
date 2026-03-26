---
name: dev-branch-setup
description: 需求开发前期准备 - 使用 git worktree 创建隔离工作空间。当用户提到「新需求」「开始开发」「迭代分支」「创建开发分支」「准备开发环境」「切分支」或给出类似 EIM/EI 开头的迭代分支名时触发。自动完成：询问需求内容、修改窗口标题、拉取迭代分支、通过 worktree 创建隔离的开发工作空间。支持多需求并行开发，每个需求一个独立 worktree。即使用户只是说「开个新需求」或「准备下分支」也应触发此 skill。
---

# 需求开发前期准备

## 背景

团队的分支管理规范：每个迭代有一个主分支（如 `EIM65086922_20260304`），该分支不能直接 push 代码。开发者需要从迭代分支创建一个 `_common` 后缀的分支进行实际开发，开发完成后通过 PR 合入迭代分支。

使用 `git worktree` 为每个需求创建独立的工作目录，这样可以：
- 同时并行开发多个需求，互不干扰
- 每个需求有独立的工作空间，不需要 stash/切换分支
- 每个需求可以在独立的 Claude Code 窗口中开发

## 目录结构

所有 worktree 统一放在主仓库的 `.worktrees/` 目录下，以分支名命名：

```
主仓库/
├── .git/
├── .worktrees/                              ← worktree 根目录
│   ├── 记忆冲突检测优化/                      ← 需求A
│   └── 角色抽取 v2/                           ← 需求B
└── (主仓库代码文件)
```

## 执行流程

按以下步骤**顺序执行**，每步完成后再进入下一步：

### Step 1: 收集信息

向用户询问以下信息（如果用户已在对话中提供，则跳过对应问题）：

1. **迭代分支名** - 例如 `EIM65086922_20260304`
2. **需求简述** - 简短描述当前要做的需求（用于窗口标题）

### Step 2: 修改窗口标题

使用 Bash 工具执行以下命令，将 Claude Code 窗口标题修改为需求简述，方便用户在多窗口间区分：

```bash
printf '\033]0;%s\007' "<需求简述>"
```

### Step 3: 拉取迭代分支

确保本地有最新的迭代分支代码：

```bash
git fetch origin <迭代分支名>
```

如果本地已有该分支：
```bash
git fetch origin <迭代分支名>
```

如果本地没有该分支，先创建本地跟踪分支：
```bash
git fetch origin <迭代分支名>
git branch <迭代分支名> origin/<迭代分支名>
```

注意：这里只 fetch 和创建本地分支引用，不需要 checkout 迭代分支（避免影响当前工作目录）。

### Step 4: 创建 worktree 开发工作空间

1. 确保 `.worktrees/` 目录存在：
```bash
mkdir -p .worktrees
```

2. 确保 `.worktrees` 在 `.gitignore` 中（避免被提交）：
```bash
grep -q '^\.worktrees/' .gitignore 2>/dev/null || echo '.worktrees/' >> .gitignore
```

3. 基于迭代分支创建 worktree + 新分支：
```bash
git worktree add .worktrees/<worktree目录名> -b <迭代分支名>_common <迭代分支名>
```

**分支命名规则：** `<迭代分支名>_common`，例如：
- 迭代分支 `EIM65086922_20260304` → 开发分支 `EIM65086922_20260304_common`

**worktree 目录命名规则：** 直接使用用户提供的需求简述作为目录名。如果需求简述包含不适合作为目录名的字符（如 `/`、`\`、`:`、`*`、`?`、`"`、`<`、`>`、`|`），需要移除或替换为 `-`。中文和空格可以直接使用。
- 示例：
  - 需求「记忆冲突检测优化」→ 目录名 `记忆冲突检测优化`
  - 需求「角色抽取 v2」→ 目录名 `角色抽取 v2`
  - 需求「fix OCR timeout」→ 目录名 `fix OCR timeout`
- worktree 路径示例：`.worktrees/记忆冲突检测优化/`

### Step 5: 在当前 tab 右侧分屏打开 worktree

在 iTerm2 当前 tab 页右侧创建垂直分屏，自动 cd 到 worktree 目录并启动 Claude Code：

```bash
osascript -e '
tell application "iTerm"
    tell current session of current tab of current window
        set newSession to (split vertically with default profile)
        tell newSession
            write text "cd \"<worktree绝对路径>\" && c"
        end tell
    end tell
end tell
'
```

### Step 6: 确认完成

向用户汇报：
- worktree 工作目录的**完整绝对路径**
- 当前分支名
- 分支来源（基于哪个迭代分支）
- 已在新终端窗口中打开并启动 Claude Code

## 异常处理

- **分支已存在**：如果 `_common` 分支已存在，询问用户是要切换到已有 worktree 还是删除重建
- **worktree 已存在**：提示用户已有该 worktree，告知路径，询问是否直接使用
- **远程分支不存在**：提示用户确认分支名是否正确
- **有未提交的更改**：worktree 模式下通常不受影响，但如果当前分支与目标分支有冲突需提示

## 清理 worktree

当需求开发完成并合入后，可以清理 worktree：

```bash
# 删除 worktree
git worktree remove .worktrees/<迭代分支名>_common

# 删除本地开发分支（如果已合入）
git branch -d <迭代分支名>_common
```

列出所有 worktree：
```bash
git worktree list
```
