# Git 使用指南

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际使用场景

---

## 一、Git 是什么

Git 是一个**版本管理工具**。每次改完代码，打一个"快照"（commit），以后随时能回到任何一次快照的状态。

**四个关键区域：**

```
工作区              暂存区              本地仓库           远程仓库
(你的文件)   git add →    (准备提交)   git commit →    (版本历史)   git push →    (GitHub/GitLab)
                                         
                                         ← git pull
```

- **工作区**：你正在编辑的文件，改了什么 Git 都追踪着
- **暂存区**：`git add` 之后，"我打算提交这些改动"
- **本地仓库**：`git commit` 之后，改动正式记入版本历史
- **远程仓库**：GitHub/GitLab 上的，`git push` 推上去，别人才能看到

---

## 二、日常最常用命令

### `git status` — 查看当前状态

告诉你：哪些文件改了但没 add（红色），哪些 add 了但没 commit（绿色），当前哪个分支、有没有落后/超前远程。

```bash
git status                  # 显示完整状态信息
git status -s               # -s：简短模式，只显示文件名和状态标记（??=未跟踪, M=已修改, A=已暂存）
git status -b               # -b：同时显示当前分支信息（等效 git status --branch）
```

### `git add` — 把改动加入暂存区

```bash
git add .                             # 添加当前目录及子目录所有改动
git add -A                            # -A：添加整个仓库所有改动（不管你在哪个目录执行）
git add mock/user.js                  # 添加单个文件
git add src/views/testboard/          # 添加整个文件夹
git add -p                            # -p：交互式逐块确认（hunk by hunk），改了很多但只想提交其中一部分时用
```

### `git commit` — 提交到本地仓库

```bash
git commit -m "修复screen组件定位"     # -m：后面直接写提交信息
git commit -am "修复样式"              # -a：跳过 git add，直接把所有已跟踪文件的改动 add + commit（新文件不会自动加）
git commit --amend -m "新的提交信息"   # --amend -m：修改上一次 commit 的信息，不新增 commit
git commit --amend --no-edit          # --amend --no-edit：把当前暂存区的改动追加到上一次 commit，不改变提交信息
```

### `git remote` — 关联远程仓库

本地仓库要和远程仓库（GitHub/GitLab）通信用，先得"关联"。

```bash
# 本地已有仓库，关联远程
git remote add origin <远程仓库URL>     # add：添加远程仓库，起名叫 origin（业界约定俗成的默认名）

# 远程已有仓库，直接克隆（自动关联，不用手动 add）
git clone <远程仓库URL>                 # 克隆下来自动关联好，远程地址叫 origin
```

**查看和管理远程关联：**

```bash
git remote -v                        # -v：查看所有远程仓库及其 URL（fetch=拉取地址, push=推送地址）
git remote add upstream <URL>        # add：添加第二个远程仓库，比如 fork 场景下叫 upstream（上游仓库）
git remote set-url origin <新URL>    # set-url：修改已有的远程地址（比如仓库搬家了）
git remote remove origin             # remove：删除远程关联
git remote rename origin upstream    # rename：重命名远程仓库
```

> 常见场景：`origin` 指向你自己的远程仓库，`upstream` 指向原始开源仓库。从 upstream 拉最新代码，推到自己 origin。

### `git push` — 推送到远程

```bash
git push                              # 推到当前分支对应的远程分支
git push origin main                  # origin=远程仓库名，main=分支名
git push -u origin feature            # -u：设置上游分支，以后在这个分支直接 git push 就行，不用每次指定
git push --force                      # --force：强制推送覆盖远程（⚠️ 会覆盖别人的提交，慎用）
git push --force-with-lease           # --force-with-lease：更安全的强制推送，如果远程有别人新提交就拒绝推送
```

### `git pull` — 拉取远程更新

```bash
git pull                              # 拉取 + 自动 merge 到本地
git pull origin main                  # 指定拉 origin 仓库的 main 分支
git pull --rebase                     # --rebase：拉取后用 rebase 而不是 merge（推荐，历史更干净，避免无意义的合并节点）
git pull --rebase origin main         # 组合用法：指定仓库、分支，用 rebase 方式拉取
```

---

## 三、分支 — 日常开发的核心

**分支就是一条独立的开发线。** 多人协作时，每个人在自己的分支上改，互不影响，最后合并。

### 创建和切换分支

```bash
# 传统写法（checkout 身兼多职）
git branch feature-btn-style          # 创建名为 feature-btn-style 的新分支（不会自动切过去）
git branch -d feature-btn-style       # -d：删除已合并的分支（安全，没合并会拒绝）
git branch -D feature-btn-style       # -D：强制删除，不管有没有合并
git checkout feature-btn-style        # 切换到 feature-btn-style 分支
git checkout -b feature-btn-style     # -b：创建并切换到新分支（等于 branch + checkout 二合一）

# Git 2.23+ 推荐写法（switch 专管分支切换，语义更清晰）
git switch feature-btn-style          # 切换到已有分支（switch 只做切换，不做别的）
git switch -c feature-btn-style       # -c：创建并切换到新分支（等于 branch + switch 二合一，推荐替代 checkout -b）
git switch -                           # -：切回上一个分支（类似 cd -）
```

### 列出分支

```bash
git branch                            # 列出本地所有分支，当前分支前有 * 号
git branch -a                         # -a：同时列出本地和远程分支
git branch -r                         # -r：只列出远程分支
```

### switch vs checkout 的区别

| | `git checkout` | `git switch` |
|------|------|------|
| 切换分支 | ✅ | ✅（推荐） |
| 创建并切换 | `checkout -b` | `switch -c`（推荐） |
| 恢复文件 | ✅ `checkout -- 文件` | ❌ 不支持（用 `git restore`） |
| 语义 | 身兼多职，容易搞混 | 只做一件事：切换分支 |

> `checkout` 既能切分支又能恢复文件，新手容易搞混。Git 2.23 把它拆成了 `switch`（切换分支）和 `restore`（恢复文件），各司其职。

### 恢复文件 — git restore（替代 checkout --）

```bash
git restore 文件名                     # 撤销工作区改动，回到上次 commit 的状态
git restore --staged 文件名            # --staged：把已 add 的文件从暂存区撤回工作区（替代 git reset HEAD）
git restore --source=HEAD~1 文件名    # --source：从指定 commit 恢复某个文件
```

**典型工作流：**

```
main ────●────●────●────●──────  （稳定版本）
              \
feature ───────●────●────●       （你的开发分支，互不影响）
```

你在 feature 上开发完毕，测试没问题后，合并回 main：

```bash
git checkout main                     # 先切回 main
git merge feature-btn-style           # 把 feature 的改动合并进来
git branch -d feature-btn-style       # 删掉已合并的分支
```

---

## 四、变基（rebase）— 让历史干净如一条线

### merge vs rebase

```
merge 的方式（保留真实分叉）：
main  ──●──●──●──●──M    （多了一个合并节点 M）
           \      /
feature ────●──●─●

rebase 的方式（历史一条直线）：
main  ──●──●──●──●
                  \
feature ──────────●′──●′   （原来的提交被"搬"到 main 最新提交后面）
```

**merge** 保留真实的分支历史，但有很多分叉。**rebase** 让历史干净得像一条线，但改写了提交记录（提交 hash 变了）。

### 用法

```bash
# 在 feature 分支上，把自己的提交接到 main 的最新提交后面
git checkout feature
git rebase main

# 拉取远程 main 的最新代码，把自己的提交接在后面（日常最常用）
git pull --rebase origin main
# --rebase：不用 merge，把自己的 commit 续在拉下来的最新 commit 之后
# origin：远程仓库名（默认叫 origin）
# main：要拉取的分支名
```

### rebase 过程中遇到冲突

```bash
# 按提示手动解决冲突后：
git add 冲突文件
git rebase --continue                # --continue：继续 rebase 流程

# 放弃本次 rebase：
git rebase --abort                   # --abort：中止，回到 rebase 之前的干净状态
```

> ⚠️ **不要 rebase 已经 push 到远程的提交。** 因为 rebase 会改写提交 hash，别人如果基于旧提交继续开发，历史就乱了。只 rebase 本地还没 push 的提交。

---

## 五、撤销操作 — 救命的

| 场景 | 命令 | 说明 |
|------|------|------|
| 改错了文件，想回到上次 commit 的状态 | `git checkout -- 文件名` | 丢弃工作区改动（⚠️ 不可恢复） |
| add 错了，想从暂存区撤回 | `git reset HEAD 文件名` | 文件从暂存区退回工作区，改动不丢 |
| commit 信息写错了，想改 | `git commit --amend -m "新信息"` | 替换上一次 commit 的信息 |
| commit 本身搞错了，想撤销但保留代码 | `git reset --soft HEAD~1` | --soft：撤销 commit，改动留在暂存区 |
| commit 和代码都彻底不要了 | `git reset --hard HEAD~1` | --hard：彻底删除 commit 和代码（⚠️ 不可恢复） |
| 想回到某次特定 commit | `git reset --hard commit哈希` | 回到指定版本，后面的改动全丢 |

`HEAD~1` = 回退 1 次 commit，`HEAD~3` = 回退 3 次。

### `git checkout` 切换和恢复

```bash
git checkout main                     # 切换到 main 分支
git checkout -b new-branch            # -b：创建并切换
git checkout -- 文件名                 # --：撤销工作区改动，回到上次 commit 的状态（⚠️ 不可恢复）
```

### `git reset` 三种模式对比

| 模式 | commit | 暂存区 | 工作区 | 安全性 |
|------|------|------|------|------|
| `--soft` | 撤销 | 保留 | 保留 | ✅ 最安全 |
| `--mixed`（默认） | 撤销 | 清空 | 保留 | ✅ 安全 |
| `--hard` | 撤销 | 清空 | 清空 | ⚠️ 不可恢复 |

---

## 六、冲突 — 一定会遇到

两个人改了同一个文件的同一行，合并时 Git 不知道用谁的，就报冲突：

```bash
git merge feature-branch
# CONFLICT (content): Merge conflict in src/views/testboard/index.vue
```

文件里会出现冲突标记：

```
<<<<<<< HEAD
你的版本
=======
别人的版本
>>>>>>> feature-branch
```

**解决步骤：** 手动删掉 `<<<<<<<` / `=======` / `>>>>>>>` 标记，保留最终想要的代码，然后：

```bash
git add src/views/testboard/index.vue  # 标记冲突已解决
git commit -m "解决合并冲突"            # 完成合并
```

---

## 七、查看历史

### `git log` — 查看提交历史

```bash
git log                               # 完整提交历史，显示作者、日期、完整 hash
git log --oneline                     # --oneline：每条 commit 一行，简洁模式
git log --oneline -5                  # -5：只看最近 5 条
git log --graph --oneline             # --graph：显示分支线，直观看出 merge 分叉
git log --oneline --all               # --all：显示所有分支的历史
```

### `git diff` — 查看具体改动内容

```bash
git diff                              # 工作区 vs 暂存区：还没 add 的改动
git diff --staged                     # --staged：暂存区 vs 最新 commit：已 add 但没 commit 的
git diff HEAD                         # HEAD：工作区 vs 最新 commit：所有还未 commit 的改动总和
git diff main feature                 # 比较两个分支的差异
```

---

## 八、暂存工作 — 切换分支不丢改动

`git stash` 把当前未提交的改动"存"起来，工作区变干净，方便切换分支。

```bash
git stash                             # 把当前所有未提交的改动暂存，工作区变干净
git stash save "修复了一半的样式"       # save：存起来并加上描述，方便之后识别
git stash list                        # list：查看存了哪些 stash
git stash pop                         # pop：取出最近一次存的，恢复到工作区，同时删除这条 stash
git stash apply                       # apply：取出但不删除 stash（可以多次应用到不同分支）
git stash drop                        # drop：删除最近一条 stash
```

> 典型场景：在 feature 分支改了一半，突然要切到 main 修 bug。—— `git stash` → 切分支修 bug → 切回来 → `git stash pop`。

---

## 九、你项目的实际场景速查

| 你做了什么 | 用哪个命令 |
|------|------|
| 改了几个页面样式，准备提交 | `git add .` → `git commit -m "继续统一样式"` |
| 想看看改了哪些文件 | `git status` |
| 想看看具体改了什么内容 | `git diff` |
| 把本地提交推到 GitLab | `git push` |
| 别人更新了代码，拉下来 | `git pull` 或 `git pull --rebase` |
| 创建一个新功能分支 | `git checkout -b feature-xxx` |
| 改崩了想回退单个文件 | `git checkout -- 文件名` |
| 改崩了想回退整个 commit | `git reset --soft HEAD~1` |
| 改了一半要切分支 | `git stash` → 切分支 → 回来 → `git stash pop` |
| 合并时出现冲突 | 手动解决 → `git add` → `git commit` |
