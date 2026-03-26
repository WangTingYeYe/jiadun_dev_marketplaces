---
name: merge-conflict-resolver
description: 分支冲突合并与 PR 发起。当迭代分支与其他分支（如 master）有冲突需要合并时，自动创建 merge 专用分支、合并目标分支、解决冲突并发起 PR 合回迭代分支。当用户提到「合并冲突」「解决冲突」「merge 冲突」「分支冲突」「合入 master」「合入主分支」「XX分支和XX分支有冲突」「需要 merge」或类似表述时触发。即使用户只是说「跟 master 合一下」或「把 master 合进来」也应触发此 skill。
---

# 分支冲突合并与 PR 发起

## 背景

团队分支管理规范中，迭代分支不允许直接 push 代码。当迭代分支与其他分支（通常是 master）产生冲突时，需要：

1. 从迭代分支创建一个 `_merge` 后缀的临时合并分支
2. 将目标分支 merge 到这个临时分支中
3. 在临时分支上解决冲突
4. 通过 PR 将临时合并分支合回迭代分支

这样做的好处是：迭代分支保持干净，合并过程可追溯、可 review，出问题可以直接关闭 PR 回滚。

## 执行流程

按以下步骤**顺序执行**：

### Step 1: 收集信息

向用户询问以下信息（如果用户已在对话中提供，则跳过对应问题）：

1. **迭代分支名**（当前要合并的分支） - 例如 `EIM65086922_20260304_1`
2. **目标分支名**（要合入的分支） - 例如 `master`

通过当前 git 状态辅助判断：如果用户当前已在某个迭代分支上，可以提示确认。

### Step 2: 拉取最新代码

确保本地两个分支都是最新的：

```bash
git fetch origin <迭代分支名>
git fetch origin <目标分支名>
```

如果本地没有迭代分支，先创建本地跟踪分支：
```bash
git branch <迭代分支名> origin/<迭代分支名>
```

更新本地迭代分支到最新：
```bash
git checkout <迭代分支名>
git pull origin <迭代分支名>
```

### Step 3: 创建 merge 专用分支

从迭代分支创建新的合并分支：

```bash
git checkout -b <迭代分支名>_merge
```

**命名规则：** `<迭代分支名>_merge`
- 示例：`EIM65086922_20260304_1` → `EIM65086922_20260304_1_merge`

如果分支已存在，询问用户是否删除重建。

### Step 4: 合并目标分支

将目标分支 merge 到当前 merge 分支：

```bash
git merge origin/<目标分支名>
```

#### 冲突处理策略

**无冲突：** 直接进入 Step 5。

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

如果用户需要手动处理某些文件，暂停流程，提示用户处理完后告知继续。

### Step 5: 提交并推送

所有冲突解决后，提交 merge 结果：

```bash
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

### Step 6: 发起 PR

使用 `gh` CLI 创建 PR：

```bash
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

### Step 7: 汇报结果

向用户展示：
- PR 链接（可点击）
- merge 分支名
- 解决的冲突文件列表（如有）
- 提示用户 review 后合入 PR

## 异常处理

- **merge 分支已存在**：询问用户是删除重建还是在现有分支上继续
- **push 被拒绝**：检查是否有权限问题，或远程已有更新需要先 pull
- **gh CLI 未安装**：提示用户安装 `gh` 并登录（`gh auth login`）
- **PR 创建失败**：检查错误信息，常见原因是 base 分支不存在或没有权限
- **无冲突**：如果 merge 没有产生任何冲突，告知用户并询问是否仍要通过 PR 方式合入（因为可能只是想走 review 流程）
