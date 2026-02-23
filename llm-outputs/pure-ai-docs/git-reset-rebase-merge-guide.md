# Git Reset、Rebase、Merge 完全指南

> 本文档总结了 Git 中三个核心概念：Reset、Rebase 和 Merge 的使用方法和最佳实践。

## 目录

-   [Git 三个区域](#git-%E4%B8%89%E4%B8%AA%E5%8C%BA%E5%9F%9F)
    
-   [Git Reset](#git-reset)
    
-   [Git Rebase vs Merge](#git-rebase-vs-merge)
    
-   [合并多个提交](#%E5%90%88%E5%B9%B6%E5%A4%9A%E4%B8%AA%E6%8F%90%E4%BA%A4)
    
-   [最佳实践](#%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)
    

---

## Git 三个区域

理解 Git 的三个区域是掌握 Reset、Rebase、Merge 的基础：

```text
工作区 →(add)→ 暂存区 →(commit)→ 本地仓库
```

1.  **工作区** (Working Directory) - 你实际看到的文件
    
2.  **暂存区** (Staging Area/Index) - `git add` 后的文件
    
3.  **本地仓库** (Local Repository) - `git commit` 后的提交
    

---

## Git Reset

`git reset` 用于**撤销**更改，有三种模式，从温和到危险。

### 三种模式对比

| 模式  | 撤销 commit | 撤销 add | 保留代码改动 | 安全性 |
| --- | --- | --- | --- | --- |
| `--soft` | ✅   | ❌   | ✅ 暂存区 | 安全  |
| `--mixed` (默认) | ✅   | ✅   | ✅ 工作区 | 安全  |
| `--hard` | ✅   | ✅   | ❌ 全部丢失 | 危险  |

### 1\. `git reset --soft` （最温和）

只撤销提交，**保留**所有更改在暂存区

```bash
git reset --soft HEAD~1  # 撤销最近 1 次提交
```

-   C 提交被撤销
    
-   **代码改动保留在暂存区**（绿色，已 staged）
    
-   相当于"撤销 commit，但不撤销 add"
    

**使用场景**：提交信息写错了，想重新提交

```bash
git reset --soft HEAD~1
git commit -m "正确的提交信息"
```

### 2\. `git reset --mixed` 或 `git reset` （默认）

撤销提交，并**从暂存区移除**，但保留文件内容

```bash
git reset HEAD~1  # 默认是 --mixed
```

-   C 提交被撤销
    
-   **代码改动保留在工作区**（红色，未 staged）
    
-   需要重新 `git add`
    

**使用场景**：想重新整理要提交的内容

### 3\. `git reset --hard` （最危险）

**彻底删除**所有更改！

```bash
git reset --hard HEAD~1
```

-   C 提交被撤销
    
-   **所有代码改动都丢失**
    
-   回到 B 的状态，像 C 从未存在过
    

⚠️ **使用前要确认！改动的代码会永久丢失！**

### 其他常用用法

```bash
# 撤销暂存区的文件（不撤销提交）
git reset 文件名
git reset HEAD 文件名

# 回到指定提交
git reset --hard abc123f  # 回到 commit abc123f

# 保留更改但回到某个提交
git reset --soft abc123f  # 所有更改都在暂存区
```

---

## Git Rebase vs Merge

### 核心区别

-   **Merge（合并）**：保留完整的提交历史
    
-   **Rebase（变基）**：重写提交历史，使线性更清晰
    

### 图解说明

假设有这样的分支结构：

```text
A --- B --- C (main)
      \
       D --- E (feature)
```

#### 使用 Merge

```bash
git checkout main
git merge feature
```

结果：

```text
A --- B --- C -------- M (main)
      \              /
       D --- E (feature)
```

-   创建了一个**新的合并提交 M**
    
-   保留了真实的开发历史
    
-   分支图有分叉
    

#### 使用 Rebase

```bash
git checkout feature
git rebase main
```

结果：

```text
A --- B --- C --- D' --- E' (main)
```

-   把 D 和 E 的提交**复制**一份，依次应用到 C 之后
    
-   历史变成**线性**的
    
-   看起来像是在 main 上直接开发的
    

### 优缺点对比

|     | Merge | Rebase |
| --- | --- | --- |
| **历史真实性** | ✅ 保留真实历史 | ❌ 重写历史 |
| **历史清晰度** | ❌ 有分叉，复杂 | ✅ 线性，清晰 |
| **安全性** | ✅ 不会丢失提交 | ⚠️ 可能冲突复杂 |
| **适用场景** | 公共分支、保留历史 | 个人分支、清理历史 |

### 使用建议

**用 Merge**：

-   ✅ 公共分支（如 main/master）
    
-   ✅ 需要保留完整的开发历史
    
-   ✅ 团队协作时
    

**用 Rebase**：

-   ✅ 个人功能分支
    
-   ✅ 准备合并前清理历史
    
-   ✅ 想要线性历史
    

### ⚠️ 重要警告

**绝对不要对已经推送的公共提交用 rebase！**

这会重写历史，导致其他人的仓库出问题。如果已经推送，建议使用 `git revert` 来撤销更改：

```bash
git revert HEAD  # 创建新提交来撤销上一次提交
```

---

## 合并多个提交

### 方法一：使用 reset --soft（简单直接）

**适合场景**：快速合并最近的多个提交

```bash
# 1. 回退 10 个提交，但保留所有更改在暂存区
git reset --soft HEAD~10

# 2. 重新提交（把所有改动合并成一个提交）
git commit -m "合并后的提交信息"
```

**原理**：

-   `HEAD~10` 表示倒数第 10 个提交
    
-   `--soft` 只撤销提交，不改代码，所有更改都在暂存区
    
-   最后重新 commit，就把所有改动打包成一个了
    

**实际例子**：

假设你有这样的提交历史：

```text
abc1234 (HEAD) feat: 添加功能C
def5678 feat: 添加功能B
ghi9012 feat: 添加功能A
```

你想合并这 3 个提交：

```bash
git reset --soft HEAD~3
git commit -m "feat: 添加完整功能模块"
```

结果：

```text
xyz7890 (HEAD) feat: 添加完整功能模块
```

3 个提交变成 1 个了！

### 方法二：使用交互式 rebase（更灵活）

**适合场景**：需要选择性合并或重新排序

```bash
# 1. 进入交互式 rebase 模式（编辑最近 11 个提交）
git rebase -i HEAD~11

# 2. 会打开编辑器，看到类似这样的内容：
pick abc1234 第1个提交
pick def5678 第2个提交
pick ghi9012 第3个提交
...
pick jkl3456 第10个提交

# 3. 把第 2-10 个提交的 "pick" 改成 "squash" 或 "s"：
pick abc1234 第1个提交
squash def5678 第2个提交
squash ghi9012 第3个提交
...
squash jkl3456 第10个提交

# 4. 保存退出后，会再次打开编辑器让你编辑合并后的提交信息
```

**rebase 的优点**：

-   可以选择性地合并某些提交
    
-   可以重新排序提交
    
-   可以修改提交信息
    

---

## 最佳实践

### 1\. 执行前先检查

```bash
# 查看最近的提交
git log --oneline -10

# 查看更详细的信息
git log -10

# 查看图形化历史
git log --graph --oneline --all
```

### 2\. 安全第一

```bash
# 在执行危险操作前，先创建备份分支
git branch backup-branch

# 如果出错了，可以从备份恢复
git reset --hard backup-branch
```

### 3\. 已推送的提交

-   ❌ 不要使用 `reset --hard` + `push --force`
    
-   ✅ 使用 `git revert` 来撤销
    
-   ✅ 或者在团队协商后谨慎使用 force push
    

### 4\. 工作流程建议

```bash
# 开发新功能时
git checkout -b feature-xxx
# ... 开发和提交 ...
git commit -m "feat: xxx"

# 准备合并前，更新主分支
git checkout main
git pull origin main

# 变基你的功能分支（使历史线性）
git checkout feature-xxx
git rebase main

# 合并到主分支
git checkout main
git merge feature-xxx

# 推送
git push origin main
```

---

## 命令速查表

```bash
# Reset 相关
git reset --soft HEAD~1    # 撤销 commit，保留暂存
git reset HEAD~1           # 撤销 commit + add，保留工作区
git reset --hard HEAD~1    # 撤销所有，丢失更改
git reset --soft abc123f   # 回到指定提交，保留暂存

# Merge 相关
git merge feature-branch   # 合并分支
git merge --no-ff feature  # 创建合并提交（即使是快进）

# Rebase 相关
git rebase main            # 变基到 main
git rebase -i HEAD~5       # 交互式变基最近 5 个提交
git pull --rebase          # 拉取时使用 rebase

# 合并提交
git reset --soft HEAD~N && git commit -m "新信息"  # 合并 N 个提交
git rebase -i HEAD~N       # 交互式合并

# 安全撤销
git revert HEAD            # 创建新提交来撤销上一次提交
git checkout -- 文件名     # 撤销工作区文件的更改
```

---

## 相关资源

-   [Git 官方文档 - git-reset](https://git-scm.com/docs/git-reset)
    
-   [Git 官方文档 - git-rebase](https://git-scm.com/docs/git-rebase)
    
-   [Git 官方文档 - git-merge](https://git-scm.com/docs/git-merge)
    
-   [Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials)
    

---

**文档创建时间**：2026-02-07 **最后更新**：2026-02-07