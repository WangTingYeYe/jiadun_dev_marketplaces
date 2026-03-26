---
name: merge-conflict-resolver
description: 分支冲突合并与 PR 发起。当迭代分支与其他分支（如 master）有冲突需要合并时，自动通过 git worktree 创建隔离的合并工作空间、合并目标分支、解决冲突并发起 PR 合回迭代分支。当用户提到「合并冲突」「解决冲突」「merge 冲突」「分支冲突」「合入 master」「合入主分支」「XX分支和XX分支有冲突」「需要 merge」或类似表述时触发。即使用户只是说「跟 master 合一下」或「把 master 合进来」也应触发此 skill。
---

# 分支冲突合并与 PR 发起

## 背景

团队分支管理规范中，迭代分支不允许直接 push 代码。当迭代分支与其他分支（通常是 master）产生冲突时，需要：

1. 从迭代分支创建一个 `_merge` 后缀的临时合并分支
2. 将目标分支 merge 到这个临时分支中
3. 在临时分支上解决冲突
4. 通过 PR 将临时合并分支合回迭代分支

整个过程通过 **git worktree** 在独立工作目录中完成，不会影响当前工作区的代码状态。这样即使正在开发其他需求，也可以安全地进行冲突合并。

## 目录结构

与 `dev-branch-setup` 保持一致，所有 worktree 统一放在主仓库的 `.worktrees/` 目录下：

```
主仓库/
├── .git/
├── .worktrees/
│   ├── 记忆冲突检测优化/                  ← 开发需求（dev-branch-setup 创建）
│   ├── merge-EIM65086922_20260304_1/      ← 合并工作空间（本技能创建）
│   └── ...
└── (主仓库代码文件)
```

## 执行流程

按以下步骤**顺序执行**：

### Step 1: 收集信息

向用户询问以下信息（如果用户已在对话中提供，则跳过对应问题）：

1. **迭代分支名**（当前要合并的分支） - 例如 `EIM65086922_20260304_1`
2. **目标分支名**（要合入的分支） - 例如 `master`

通过当前 git 状态辅助判断：如果用户当前已在某个迭代分支上（包括在 worktree 中），可以提示确认。

### Step 2: 确定主仓库路径

合并操作需要在**主仓库**（而非 worktree）中创建新的 worktree。检测当前是否在 worktree 中：

```bash
git rev-parse --git-common-dir
```

如果输出不是 `.git`，说明当前在 worktree 中，需要找到主仓库路径：

```bash
# 获取主仓库的绝对路径
MAIN_REPO=$(git rev-parse --path-format=absolute --git-common-dir | sed 's/\/.git$//')
```

如果当前就在主仓库，则 `MAIN_REPO` 就是当前目录。

后续所有 git worktree 操作都在 `$MAIN_REPO` 下执行。

### Step 3: 拉取最新代码

在主仓库目录下确保两个分支都是最新的：

```bash
git -C "$MAIN_REPO" fetch origin <迭代分支名>
git -C "$MAIN_REPO" fetch origin <目标分支名>
```

如果本地没有迭代分支，创建本地跟踪分支：
```bash
git -C "$MAIN_REPO" branch <迭代分支名> origin/<迭代分支名>
```

更新本地迭代分支引用到最新（不 checkout，避免影响主仓库工作区）：
```bash
git -C "$MAIN_REPO" fetch origin <迭代分支名>:<迭代分支名>
```

如果上述命令因分支已 checkout 而失败，说明主仓库当前就在该分支上，改用：
```bash
git -C "$MAIN_REPO" pull origin <迭代分支名>
```

### Step 4: 通过 worktree 创建 merge 隔离工作空间

1. 确保 `.worktrees/` 目录存在并在 `.gitignore` 中：
```bash
mkdir -p "$MAIN_REPO/.worktrees"
grep -q '^\\.worktrees/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.worktrees/' >> "$MAIN_REPO/.gitignore"
```

2. 基于迭代分支创建 worktree + merge 分支：
```bash
git -C "$MAIN_REPO" worktree add "$MAIN_REPO/.worktrees/merge-<迭代分支名>" -b <迭代分支名>_merge <迭代分支名>
```

**分支命名规则：** `<迭代分支名>_merge`
- 示例：`EIM65086922_20260304_1` → `EIM65086922_20260304_1_merge`

**worktree 目录命名规则：** `merge-<迭代分支名>`
- 示例：`.worktrees/merge-EIM65086922_20260304_1/`

**如果 merge 分支已存在：** 询问用户是删除重建还是在现有分支上继续。删除重建时：
```bash
git -C "$MAIN_REPO" worktree remove "$MAIN_REPO/.worktrees/merge-<迭代分支名>" --force 2>/dev/null
git -C "$MAIN_REPO" branch -D <迭代分支名>_merge
git -C "$MAIN_REPO" worktree add "$MAIN_REPO/.worktrees/merge-<迭代分支名>" -b <迭代分支名>_merge <迭代分支名>
```

设置 worktree 路径变量，后续所有操作在此目录中进行：
```bash
WORKTREE="$MAIN_REPO/.worktrees/merge-<迭代分支名>"
```

### Step 5: 在 worktree 中合并目标分支

进入 worktree 目录，将目标分支 merge 进来：

```bash
cd "$WORKTREE"
git merge origin/<目标分支名>
```

#### 冲突处理策略

**无冲突：** 直接进入 Step 6。

**有冲突：** 按以下顺序处理：

1. 列出所有冲突文件：
```bash
git diff --name-only --diff-filter=U
```

2. 逐个分析冲突文件，尝试自动解决：
   - 对于明确可以自动解决的冲突（如 import 顺序、格式化差异、非功能性变更），直接解决
   - 对于涉及业务逻辑的冲突，展示冲突内容给用户，说明两边的改动意图，给出建议方案，等待用户确认

3. 每解决一个文件，执行：
```bash
git add <已解决的文件>
```

4. 所有冲突解决后，检查是否还有未解决的冲突：
```bash
git diff --name-only --diff-filter=U
```

如果用户需要手动处理某些文件，告知用户 worktree 路径，提示在该目录下手动修改后告知继续。

### Step 6: 提交并推送

在 worktree 中提交 merge 结果：

```bash
cd "$WORKTREE"
git commit -m "$(cat <<'EOF'
merge: 合入<目标分支名>到<迭代分支名>

将 <目标分支名> 分支合并到 <迭代分支名>，解决合并冲突。

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

推送到远程：
```bash
git push -u origin <迭代分支名>_merge
```

### Step 7: 发起 PR

使用 `gh` CLI 创建 PR（可在 worktree 目录中执行）：

```bash
cd "$WORKTREE"
gh pr create \
  --base <迭代分支名> \
  --head <迭代分支名>_merge \
  --title "merge: 合入<目标分支名>到<迭代分支名>" \
  --body "$(cat <<'EOF'
## Summary
- 将 <目标分支名> 分支合并到 <迭代分支名>
- 解决合并冲突

## 冲突文件
<列出解决的冲突文件列表>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### Step 8: 清理 worktree 并汇报结果

PR 创建成功后，清理 worktree 工作空间：

```bash
cd "$MAIN_REPO"
git worktree remove "$WORKTREE"
```

如果清理失败（如有未提交的修改），不强制删除，告知用户手动处理。

向用户展示：
- PR 链接（可点击）
- merge 分支名
- 解决的冲突文件列表（如有）
- 提示用户 review 后合入 PR
- worktree 清理状态

## 异常处理

- **merge 分支已存在**：询问用户是删除重建还是在现有分支上继续
- **worktree 已存在**：提示路径，询问是否复用或删除重建
- **push 被拒绝**：检查是否有权限问题，或远程已有更新需要先 pull
- **gh CLI 未安装**：提示用户安装 `gh` 并登录（`gh auth login`）
- **PR 创建失败**：检查错误信息，常见原因是 base 分支不存在或没有权限
- **无冲突**：如果 merge 没有产生任何冲突，告知用户并询问是否仍要通过 PR 方式合入（因为可能只是想走 review 流程）
- **当前在 worktree 中**：自动检测主仓库路径，所有 worktree 操作在主仓库下执行
