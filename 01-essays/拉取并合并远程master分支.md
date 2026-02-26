我要将远程分支的 master 拉下来然后合并到当前分支中，如何做？

拉取远程 master 并合并到当前分支，执行以下命令：

```bash
git fetch origin master
git merge origin/master
```

如果想要更干净的提交历史，可以用 rebase：

```bash
git fetch origin master
git rebase origin/master
```