# Git 笔记

## 一、git checkout -b dev 的含义

`git checkout`：原本用于切换分支。

`-b`：表示"创建一个新的分支"。

`dev`：是你想要创建的新分支的名称。

**这条命令的意思是：**
创建一个名为 dev 的新分支，并立即切换到该分支。

**这等价于以下两条命令的组合：**
```bash
git branch dev      # 创建 dev 分支
git checkout dev    # 切换到 dev 分支
```

## 二、如何切换到某个已存在的分支？

使用以下命令即可切换到已存在的分支（比如 main、feature/login 等）：

```bash
git checkout <分支名>
```

例如：

```bash
git checkout main
```

或者，如果你使用的是较新版本的 Git（2.23+），推荐使用更语义清晰的 `git switch` 命令：

**创建并切换到新分支：**

```bash
git switch -c dev
```

**切换到已有分支：**

```bash
git switch main
```

## 三、查看当前有哪些分支

你可以用以下命令查看本地所有分支，当前所在分支会以 `*` 标出：

```bash
git branch
```

或者查看远程和本地所有分支：

```bash
git branch -a
```