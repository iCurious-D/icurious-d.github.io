+++
date = '2025-12-15T14:38:11+08:00'
draft = false
title = 'Git 快速上手：原理、命令与底层实现'
tags = ["Git", "工程化"]
categories = ["AI 技术笔记"]
description = "Git 快速上手：包括 Git 的核心原理、开发流程、常用命令、以及底层实现。"

+++

## Git 快速上手：原理、命令与底层实现

**Git** 是现代软件开发中不可或缺的版本控制工具。
本文简单梳理了 Git 的常用命令、核心原理、开发流程、最佳实践以及底层实现。

---

### 一、Git 核心原理

#### 1. 三个区域与文件状态

Git 管理文件时涉及三个区域：

- **工作区（Working Directory）**：你正在编辑的文件，实际存在于磁盘上。
- **暂存区（Staging Area / Index）**：存放即将提交的内容，对应 `.git/index` 文件。
- **版本库（Repository / Git Directory）**：存储所有提交历史，文件以对象形式保存在 `.git/objects`。

文件状态流转：
- 未跟踪（Untracked）→ 已修改（Modified）→ 已暂存（Staged）→ 已提交（Committed）

#### 2. 指针与提交对象

Git 中核心是指针：

- **HEAD**：指向当前分支（或直接指向提交，即 detached HEAD 状态）。
- **分支（Branch）**：本质是指向提交对象的可变指针，存储在 `.git/refs/heads/`。
- **提交对象（Commit）**：包含树对象、父提交指针、作者信息等，通过父指针形成历史链。

关键操作与指针移动：
- `git commit`：创建新提交，分支指针前移。
- `git branch <name>`：创建新指针。
- `git checkout <branch>`：切换 HEAD 指向，更新工作区和暂存区。
- `git reset`：移动分支指针，可选修改暂存区和工作区。

#### 3. 合并策略

**(1) 快进合并（Fast-forward）**

当目标分支的历史完全包含当前分支时，只需将指针直接前移，无新提交产生。

> A---B  main
>      \
>       C---D  feature
> → git merge feature （在 main 上）
> A---B---C---D  main, feature

**(2) 三方合并（Three-way Merge）**

两个分支都有新提交时，Git 找到共同祖先，结合三方内容合并，若有冲突需手动解决，最终产生合并提交（两个父提交）。

**(3) 变基（Rebase）**

将当前分支的提交依次重新应用到目标分支之上，生成新提交，历史线性化。**注意：不要在公共分支上使用 rebase**。

#### 4. 远程协作原理

- 远程分支（`origin/*`）是本地存储的远程状态快照。
- `git fetch` 只更新远程分支指针，不合并。
- `git pull` = `fetch` + `merge`（或 `rebase`）。
- `git push` 推送本地提交并更新远程分支，若远程有新提交会被拒绝。

---

### 二、开发流程实战模拟

假设使用 **Feature Branch Workflow** + **Pull Request**。

```bash
# 步骤 1：克隆远程仓库
git clone https://github.com/team/project.git
cd project
# 或已有仓库，更新主分支
git switch main
git pull --ff-only

# 步骤 2：创建特性分支
git switch -c feature/login

# 步骤 3：开发、检查、提交
git status
git diff
git add -p
# 修改文件后
git add login.html login.js
git diff --cached
git commit -m "feat: add login page"

# 步骤 4：推送分支到远程
# 4.1 获取主分支新变化并整合
git fetch origin
git rebase origin/main
# 4.2 运行项目测试，通过后推送
git push -u origin feature/login

# 步骤 5：创建 Pull Request 并合并
# 在托管平台创建 PR/MR，进行评审review 后合并（通常采用 merge commit 或 squash）。

# 步骤 6：合并完成后更新本地主分支
git switch main
git pull origin main
# git pull --ff-only

# 步骤 7：删除特性分支
git branch -d feature/login          # 本地删除
git push origin --delete feature/login  # 远程删除（可选）
# git fetch --prune origin

# 步骤 8：处理合并冲突
git switch feature/login
git merge main        # 出现冲突
# 手动编辑冲突文件，删除标记
git add login.js
git commit            # 完成合并

# 步骤 9：打标签与发布
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# 步骤 10：回滚错误提交
# 公共分支推荐 revert
git revert <commit-hash>
git push origin main
# 本地未推送可用 reset
git reset --hard HEAD~1

# 步骤 11：使用 stash 临时保存工作
git stash push -m "WIP: login feature"
git switch main
# ... 处理紧急任务 ...
git switch feature/login
git stash pop
```

---

### 三、分支切换的最佳实践

切换分支时，**最好确保工作区干净**，或明确处理未提交修改。

#### 1. 检查状态
```bash
git status
```

#### 2. 处理未提交修改
- **有修改且目标分支无冲突**：使用 `git stash` 临时保存。
- **有修改且目标分支有冲突**：Git 会阻止切换，必须 stash 或 commit。
- **工作区干净**：直接切换。

#### 3. 推荐流程
```bash
git status
git stash push -m "WIP: xxx"
git switch <target-branch>
# 在目标分支工作...
git switch <original-branch>
git stash pop
```

#### 4. 新命令 `git switch` 和 `git restore`

Git 2.23 引入了两个专注命令，拆分 `checkout` 的职责：

- **`git switch`**：只管分支切换与创建。
  - `git switch <branch>` 切换分支
  - `git switch -c <branch>` 创建并切换
- **`git restore`**：只管恢复文件。
  - `git restore <file>` 丢弃工作区修改
  - `git restore --staged <file>` 取消暂存
  - `git restore --source=<commit> <file>` 从历史恢复文件

使用新命令语义更清晰，降低误操作风险。

---

### 四、Git 底层实现深度解析

Git 本质是一个**内容寻址文件系统**，底层是键值存储。

#### 1. 对象模型

**(1) Blob 对象**

存储文件内容，不含文件名或路径。内容相同则哈希相同，自动去重。

**(2) Tree 对象**

存储目录结构和文件名到 blob / 子 tree 的映射，类似于目录快照。

**(3) Commit 对象**

包含指向顶层 tree 的哈希、父提交哈希、作者信息、提交信息等。

**(4) Tag 对象**

轻量标签只创建引用；附注标签创建对象，包含额外信息，可签名。

#### 2. 内容寻址存储

- 对象文件名 = 内容 SHA-1 哈希，存储在 `.git/objects/xx/yyyy...`。
- 内容不可变，哈希保证完整性。
- 松散对象使用 zlib 压缩。
- 打包文件（packfile）使用 delta 压缩，附有 .idx 索引。

#### 3. 索引文件（暂存区）

`.git/index` 是二进制文件，存储路径到 blob 哈希的映射，代表下一次提交的快照。本质是一个**哈希表**。

#### 4. 引用系统

- 分支：`.git/refs/heads/<branch>`，指向提交。
- 远程分支：`.git/refs/remotes/<remote>/<branch>`。
- 标签：`.git/refs/tags/<tag>`。
- HEAD：符号引用，指向当前分支或直接指向提交。
- 引用过多时会打包到 `.git/packed-refs`。

#### 5. 对象打包与垃圾回收

- `git gc`：打包松散对象、清理不可达对象、压缩引用。
- packfile 对相似对象进行 delta 压缩，大幅减小体积。
- 不可达对象最终由 `git prune` 或 `git gc` 清理。

#### 6. 常用命令底层调用链

- `git add <file>`：计算文件哈希 → 写入 blob（若不存在）→ 更新索引。
- `git commit`：从索引生成 tree → 创建 commit → 更新分支引用。
- `git checkout <branch>`：读取目标 commit 的 tree → 更新索引和工作区 → 更新 HEAD。
- `git status`：比较 HEAD、索引和工作区，报告差异。

---

### 五、Git 常用命令速览

以下命令覆盖了实际项目 90% 以上的 Git 操作。

**1. 初始化、克隆与配置**

| 场景           | 命令                                               | 说明                       |
| -------------- | -------------------------------------------------- | -------------------------- |
| 初始化仓库     | `git init`                                         | 在当前目录创建 Git 仓库    |
| 克隆项目       | `git clone <url>`                                  | 获取仓库并配置远程关联     |
| 配置提交者姓名 | `git config --global user.name "Your Name"`        | 设置默认提交者姓名         |
| 配置提交者邮箱 | `git config --global user.email "you@example.com"` | 设置默认提交者邮箱         |
| 查看配置及来源 | `git config --list --show-origin`                  | 排查配置被哪里覆盖         |
| 查看远程仓库   | `git remote -v`                                    | 查看拉取与推送地址         |
| 添加远程仓库   | `git remote add origin <url>`                      | 常用于本地项目首次关联远程 |
| 修改远程地址   | `git remote set-url origin <url>`                  | 仓库迁移、HTTPS/SSH 切换   |

------

**2. 日常开发：查看、暂存、提交**

| 场景                             | 命令                                  | 说明                               |
| -------------------------------- | ------------------------------------- | ---------------------------------- |
| 查看当前状态                     | `git status`                          | 当前分支、暂存、未暂存、未跟踪文件 |
| 简洁查看状态                     | `git status -sb`                      | 日常高频                           |
| 查看尚未暂存的修改               | `git diff`                            | 工作区与暂存区比较                 |
| 查看即将提交的修改               | `git diff --cached`                   | 暂存区与 `HEAD` 比较               |
| 查看相对上次提交的全部已跟踪修改 | `git diff HEAD`                       | 工作区与 `HEAD` 比较               |
| 暂存指定文件                     | `git add <file>`                      | 精确控制提交范围                   |
| 暂存整个仓库的变化               | `git add -A`                          | 包括新增、修改、删除               |
| 按修改块暂存                     | `git add -p`                          | 将混在一起的修改拆成独立提交       |
| 提交                             | `git commit -m "feat: add login"`     | 创建本地提交                       |
| 修改最近一次提交                 | `git commit --amend`                  | 替换最近一次提交                   |
| 只修改最近一次提交信息           | `git commit --amend -m "new message"` | 前提是没有意外暂存的内容           |

典型流程：

```
git status
git diff
git add -p
git diff --cached
git commit -m "fix: handle expired token"
```

------

**3. 分支：创建、切换、删除**

| 场景                       | 命令                                      |
| -------------------------- | ----------------------------------------- |
| 查看本地分支               | `git branch`                              |
| 查看远程跟踪分支           | `git branch -r`                           |
| 查看本地与远程跟踪分支     | `git branch -a`                           |
| 查看分支跟踪关系           | `git branch -vv`                          |
| 创建并切换功能分支         | `git switch -c feature/login`             |
| 切换已有分支               | `git switch main`                         |
| 从远程分支创建本地跟踪分支 | `git switch --track origin/feature/login` |
| 重命名当前分支             | `git branch -m new-name`                  |
| 安全删除本地分支           | `git branch -d feature/login`             |
| 强制删除本地分支           | `git branch -D feature/login`             |
| 删除远程分支               | `git push origin --delete feature/login`  |

- **分支本质上是指向提交的可移动引用**，创建分支不需要复制整个项目。
- `git checkout` 同时承担切换分支、恢复文件等职责；`switch` 专门用于切换分支，`restore` 用于恢复文件。
- 带着未提交修改有时也能切换分支；如果切换会覆盖这些修改，Git 通常会阻止操作。
- `-d` 会做合并状态检查；`-D` 跳过该检查。

------

**4. 远程协作：fetch、pull、push**

| 场景                             | 命令                               | 说明                             |
| -------------------------------- | ---------------------------------- | -------------------------------- |
| 获取远程更新                     | `git fetch origin`                 | 下载对象并更新远程跟踪引用       |
| 获取更新并清理失效的远程跟踪分支 | `git fetch --prune origin`         | 清理远程已删除分支的本地跟踪引用 |
| 拉取并使用 merge 整合            | `git pull --no-rebase`             | 明确使用合并策略                 |
| 拉取并使用 rebase 整合           | `git pull --rebase`                | 将本地提交重放到上游之后         |
| 只接受快进更新                   | `git pull --ff-only`               | 分支已分叉时直接失败             |
| 首次推送并设置上游               | `git push -u origin feature/login` | 后续通常可直接 `git push`        |
| 推送当前分支                     | `git push`                         | 具体目标受上游和配置影响         |

- `fetch`：获取远程更新，通常不改变当前本地分支与工作区。
- `pull`：先获取远程更新，再根据参数和配置整合到当前分支。

**push 被拒绝怎么办？**先看报错。如果是远程有新提交导致非快进拒绝：

```
git fetch origin
git rebase origin/feature/login
# 如有冲突，解决后继续 rebase
git push
```

也可以按团队规范使用 merge。**不要把强推当作普通推送失败的默认解决方案。**

------

**5. 合并分支：merge 和 rebase**

```
# 将功能分支合入主分支
git switch main
git merge feature/login

# 在功能分支上吸收最新主分支
git switch feature/login
git fetch origin
git rebase origin/main
```

| 对比         | `merge`            | `rebase`                     |
| ------------ | ------------------ | ---------------------------- |
| 处理方式     | 将两条历史汇合     | 将提交重新应用到新基点上     |
| 已有提交身份 | 保留               | 被重放的提交通常获得新哈希   |
| 历史形态     | 保留分叉与汇合关系 | 适合形成线性历史             |
| 典型场景     | 整合共享分支       | 更新个人功能分支、整理提交   |
| 冲突处理     | 一次合并过程       | 可能在多个重放提交上分别处理 |

```
原始：
A---B---C  main
     \
      D---E  feature

merge：
A---B---C-------M
     \        /
      D------E

rebase：
A---B---C---D'---E'
```

- **Fast-forward**：当前分支是目标分支的祖先时，只移动分支指针即可，无需合并提交。
- `git merge --no-ff feature/login`：即使能够快进，也创建合并提交。
- `git merge --squash feature/login`：将合并结果放入暂存区，随后自行提交；不会记录普通 merge 的双父提交关系。
- **不要随意 rebase 他人正在基于其开发的共享历史。**

------

**6. 冲突处理**

merge 冲突处理流程：

```
git merge feature/login
git status

# 编辑冲突文件，确定最终内容，删除冲突标记

git add <resolved-file>
git merge --continue
```

放弃本次合并：

```
git merge --abort
```

rebase 冲突处理：

```
git rebase origin/main

# 编辑并解决冲突
git add <resolved-file>
git rebase --continue
```

放弃本次变基：

```
git rebase --abort
```

------

**7. 撤销操作：restore、reset、revert【重点】**

先判断：**修改有没有提交？提交有没有共享？是否要保留代码？**

| 场景                               | 命令                                                   | 结果                             |
| ---------------------------------- | ------------------------------------------------------ | -------------------------------- |
| 放弃某个文件未暂存的修改           | `git restore <file>`                                   | 用暂存区版本覆盖工作区文件       |
| 取消暂存，保留工作区修改           | `git restore --staged <file>`                          | 暂存区恢复为 `HEAD` 版本         |
| 文件的暂存和工作区都恢复到上次提交 | `git restore --source=HEAD --staged --worktree <file>` | 丢弃该文件已暂存、未暂存修改     |
| 撤销最近提交，保留为已暂存修改     | `git reset --soft HEAD~1`                              | 分支回退，暂存区和工作区保持     |
| 撤销最近提交，保留为未暂存修改     | `git reset --mixed HEAD~1`                             | 分支回退，重置暂存区，保留工作区 |
| 回退提交并重置已跟踪内容           | `git reset --hard HEAD~1`                              | 分支、暂存区、工作区都回退       |
| 撤销某次提交的效果并留下记录       | `git revert <commit>`                                  | 创建反向提交                     |

三种 reset 的核心区别：

| 模式            | 移动当前分支 | 重置暂存区 | 重置工作区 |
| --------------- | ------------ | ---------- | ---------- |
| `--soft`        | 是           | 否         | 否         |
| `--mixed`，默认 | 是           | 是         | 否         |
| `--hard`        | 是           | 是         | 是         |

- `reset`：调整分支位置，常用于尚未共享的本地历史。
- `revert`：新增提交抵消旧提交的效果，常用于已经推送、多人协作的分支。

`reset --hard` 会丢弃未提交的已跟踪修改，也可能删除妨碍恢复目标路径的未跟踪文件；执行前应明确要保留什么。

------

**8. 临时切任务：stash**

场景：功能写到一半，需要临时修线上问题。

```
git stash push -u -m "login work in progress"
git switch main

# 完成其他工作后
git switch feature/login
git stash pop
```

| 命令                          | 用途                         |
| ----------------------------- | ---------------------------- |
| `git stash list`              | 查看暂存记录                 |
| `git stash show -p`           | 查看最近一条记录的补丁       |
| `git stash apply 'stash@{0}'` | 恢复指定记录，保留记录       |
| `git stash pop`               | 恢复最近记录，成功后删除记录 |
| `git stash drop 'stash@{0}'`  | 删除指定记录                 |

- 默认 stash 不包含未跟踪文件；`-u` 包含未跟踪文件；`-a` 还包含被忽略文件。
- `apply` 保留 stash，`pop` 成功后删除。
- `pop` 发生冲突时，stash 通常会保留，需要手动处理。
- stash 更适合临时保存，不应代替长期提交与备份。

------

**9. 挑选提交：cherry-pick**

场景：某个修复已经在开发分支完成，需要单独同步到发布分支。

```
git switch release/1.0
git cherry-pick <commit>
```

多个提交可以按顺序应用：

```
git cherry-pick <commit1> <commit2>
```

发生冲突：

```
git add <resolved-file>
git cherry-pick --continue

# 或放弃
git cherry-pick --abort
```

- `merge` 整合分支历史；`cherry-pick` 应用选定提交的变更。
- 一般会生成新提交，哈希通常不同。
- 目标提交可能依赖之前的代码，不能只看“这条提交改得少”就直接搬过去。

------

**10. 整理提交：交互式 rebase**

场景：准备代码评审前，把多次零碎修正整理成有意义的提交。

```
git rebase -i HEAD~3
```

常见操作：

| 操作     | 含义                               |
| -------- | ---------------------------------- |
| `pick`   | 保留提交                           |
| `reword` | 修改提交说明                       |
| `edit`   | 暂停，修改该提交                   |
| `squash` | 合并到上一条提交，并编辑说明       |
| `fixup`  | 合并到上一条提交，通常丢弃本条说明 |
| `drop`   | 删除该提交                         |

**经典问题：如何把最近三次提交合成一次？**

运行上面的命令，第一条保留 `pick`，后两条改为 `squash` 或 `fixup`。

如果已经推送，并且团队允许改写该功能分支：

```
git push --force-with-lease
```

`--force-with-lease` 会检查远程引用是否符合预期，比 `--force` 更有保护，但**不代表可以无条件改写共享分支**。

------

**11. 查询历史与定位问题**

| 场景                             | 命令                                         |
| -------------------------------- | -------------------------------------------- |
| 简洁查看提交                     | `git log --oneline`                          |
| 查看分支提交图                   | `git log --oneline --graph --decorate --all` |
| 查看某次提交详情                 | `git show <commit>`                          |
| 查看某个文件的历史               | `git log -- <file>`                          |
| 尝试跟踪文件重命名前的历史       | `git log --follow -- <file>`                 |
| 按提交说明搜索                   | `git log --grep="login"`                     |
| 查找字符串出现次数发生变化的提交 | `git log -S "functionName" -- <file>`        |
| 查某些行最后由哪次提交修改       | `git blame -L 10,30 -- <file>`               |
| 比较两个提交或分支的最终内容     | `git diff main feature/login`                |
| 看功能分支相对共同祖先的改动     | `git diff main...feature/login`              |

要区分 `log` 与 `diff`：

| 命令             | 含义                                      |
| ---------------- | ----------------------------------------- |
| `git log A..B`   | B 可达、A 不可达的提交                    |
| `git log A...B`  | 只属于其中一侧历史的提交                  |
| `git diff A..B`  | 比较 A、B 两个端点，等价于 `git diff A B` |
| `git diff A...B` | 比较 A、B 的共同祖先与 B                  |

这是很容易混淆的面试题。

------

**12. 二分定位：bisect**

场景：过去某个版本正常，现在有 bug，但不知道哪次提交引入。

```
git bisect start
git bisect bad
git bisect good <known-good-commit>

# Git 切到一个中间提交，运行测试后标记：
git bisect good
# 或
git bisect bad

# 重复直到定位完成，最后退出：
git bisect reset
```

如果有自动检测脚本：

```
git bisect run <test-command>
```

- 使用二分法缩小范围。
- 自动测试退出码 `0` 表示正常，`1–127` 中除 `125` 外表示有问题，`125` 表示跳过。
- 需要稳定、可复现的判断标准，才容易得到可靠结果。

------

**13. 误操作恢复：reflog**

场景：误 reset、误删分支，或者 rebase 后想找回原来的提交。

```
git reflog

# 找到原来的提交，先建立恢复分支
git branch rescue <old-commit>
```

- `log` 查看提交历史。
- `reflog` 记录本地引用的移动，例如切分支、reset、rebase。
- reflog 是本地记录，不会随普通 push 同步到远程。
- **它不是万能撤销**：从未提交、被覆盖的工作区内容，不一定能找回。
- 记录会过期，不可达对象也可能被垃圾回收，不能当永久备份。

------

**14. 忽略文件与清理**

场景：依赖目录、构建产物、本地配置不应该进入版本库。

```
node_modules/
dist/
.env
*.log
```

如果文件已经被跟踪：

```
git rm --cached .env
git add .gitignore
git commit -m "chore: stop tracking local environment file"
```

- `.gitignore` 主要影响未跟踪文件，**不能让已经跟踪的文件自动停止跟踪**。
- `git rm --cached` 从索引移除文件，保留本地文件。
- 停止跟踪不会清除历史里已有的文件内容。

清理未跟踪文件：

```
git clean -nd   # 预览将删除哪些未跟踪文件和目录
git clean -fd   # 执行删除
```

`-x` 会把被忽略的文件也纳入清理范围，可能包括本地配置与依赖目录。

------

**15. 发布版本：tag**

| 场景             | 命令                                             |
| ---------------- | ------------------------------------------------ |
| 查看标签         | `git tag`                                        |
| 创建附注标签     | `git tag -a v1.2.0 -m "Release v1.2.0"`          |
| 给指定提交打标签 | `git tag -a v1.2.0 <commit> -m "Release v1.2.0"` |
| 推送指定标签     | `git push origin v1.2.0`                         |
| 查看标签内容     | `git show v1.2.0`                                |

- 分支会随新提交向前移动；标签通常固定标记一个版本。
- 轻量标签主要是引用；附注标签还保存标签者、说明等信息，并可签名。
- 普通分支推送不意味着所有标签都会同步，应明确推送所需标签。

------

**16. 多任务并行：worktree**

场景：当前功能分支保留现场，同时在另一个目录修 bug 或查看其他版本。

```
git worktree add -b hotfix/payment ../project-hotfix main
git worktree list

# 工作完成且目录状态允许时移除
git worktree remove ../project-hotfix
```

它允许一个仓库拥有多个工作目录，各自具有独立的工作区和索引，共享对象库等仓库数据。同一个分支通常不能同时在多个 worktree 中检出。