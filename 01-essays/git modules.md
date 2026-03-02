```text
克隆一个包含子模块的 Git 项目，最常用的方式：

# 一步到位：克隆仓库并初始化所有子模块
git clone --recurse-submodules <repo-url>
如果已经克隆了但忘了拉子模块：

# 初始化并拉取所有子模块
git submodule update --init --recursive
几个常用的后续操作：

# 更新所有子模块到远程最新
git submodule update --remote --merge

# 查看子模块状态
git submodule status

# 对所有子模块执行同一个命令（比如切到 main 分支）
git submodule foreach git checkout main
--recursive 参数很关键，它会处理子模块嵌套子模块的情况（子
```

```text
git clone --recurse-submodules <repo-url>
```