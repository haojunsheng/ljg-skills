---
name: fix-ljg-out-doc
description: 同步 hjs-md 分支：合并 md 分支、修正 skills 输出路径、提交代码、同步到全局。Use when user says '/fix-ljg-out-doc', 'fix output doc', '修正输出路径', '合并 md 分支', '更新 skills'.
user_invocable: true
version: "1.0.0"
---

# fix-ljg-out-doc: 同步 skills 工作流

维护 `hjs-md` 分支的完整同步流程，四步一气呵成。

## 硬编码路径

```
SKILLS_REPO="$HOME/learning/code/github/ljg-skills"
SKILLS_DIR="$SKILLS_REPO/skills"
GLOBAL_SKILLS="$HOME/.claude/skills"
CURRENT_BRANCH="hjs-md"
UPSTREAM_BRANCH="md"
```

## 四步工作流

### Step 1 — 合并 md 分支

```bash
cd $SKILLS_REPO
git checkout hjs-md
git fetch origin
git merge origin/md
```

**冲突处理原则**：
- `SKILL.md` 冲突：保留 `hjs-md` 侧对输出路径、输出格式的修改；接受 `md` 侧对逻辑、内容的更新
- 其他文件冲突：优先接受 `md` 侧（upstream 变更），除非 `hjs-md` 侧有明确的本地定制
- 逐个冲突文件用 Read 读取，判断后用 Edit 修复，不要盲目 `git checkout --theirs/--ours`

合并完成后验证：

```bash
git status   # 确认无残留冲突
```

### Step 2 — 修正 skills 输出路径

扫描 `$SKILLS_DIR` 下所有文件，把错误路径替换为正确路径：

| 错误路径 | 正确路径 |
|---------|---------|
| `~/Documents/notes/` | `~/doc/notes/` |

执行方式：

```bash
grep -rl "~/Documents/notes/" $SKILLS_DIR
```

对每一个命中文件，用 Edit 工具做替换（`replace_all: true`）：

```
old_string: ~/Documents/notes/
new_string:  ~/doc/notes/
```

替换完后再跑一次 grep 确认清零：

```bash
grep -r "~/Documents/notes/" $SKILLS_DIR
# 预期：无输出
```

### Step 3 — 提交代码

```bash
cd $SKILLS_REPO
git add -A
git status   # 展示给用户看，确认变更范围
git commit -m "sync: merge md branch + fix output path to ~/doc/notes/"
```

如果 Step 2 无任何文件被修改，commit message 改为：

```
sync: merge md branch (no path fixes needed)
```

### Step 4 — 同步到全局 skills

把修改过的 skills 从 repo 复制到 `~/.claude/skills/`：

```bash
# 只同步本次涉及的 skills（Step 1/2 改动的目录）
# 若无法确定范围，全量同步
rsync -av --delete $SKILLS_DIR/ $GLOBAL_SKILLS/
```

同步后验证（抽查一个改过的 skill）：

```bash
grep "~/doc/notes/" $GLOBAL_SKILLS/ljg-book/SKILL.md   # 视实际改动 skill 而定
```

## 完成报告

输出一份简洁的执行摘要：

```
fix-ljg-out-doc 完成
- 合并：origin/md → hjs-md（X 个冲突已解决 / 无冲突）
- 路径修正：X 个文件（或"无需修正"）
- Commit：<sha> <message>
- 全局同步：X 个 skills 已更新
```

## Gotchas

- **合并前必须 fetch**——本地 `origin/md` 可能落后远端，不 fetch 直接 merge 会漏掉新提交
- **冲突解决后必须 `git add`**——Edit 修复冲突标记后，文件状态不会自动变为 resolved，需要手动 stage
- **路径替换用 replace_all**——同一文件里可能有多处 `~/Documents/notes/`，单次替换会漏
- **rsync --delete 会删除 global 里 repo 没有的 skill**——如果 global 有本地实验性 skill，提前告知用户
