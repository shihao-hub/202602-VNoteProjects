# Git Commit 合并与 Reset 原理

## 一、合并多个 Commit 的方法

### 场景：合并最近的 N 个 commit

**使用** `git reset --soft`

```bash
# 合并最后 7 个 commit 到基础 commit
git reset --soft <base-commit>
git commit -m "合并后的信息"
```

**实际案例**

```bash
# 初始状态（8 个 commit）
976b407 ← 651d85e ← 0c49f9d ← ... ← d20e8c0 (HEAD)

# 执行 reset
git reset --soft 976b407

# 结果
976b407 (HEAD)
  ↓
  [暂存区：651d85e 到 d20e8c0 的所有更改]

# 重新提交
git commit -m "feat: 完整的 Lua 注解功能"

# 最终
976b407 ← d810f9d (HEAD)
```

---

## 二、Git Reset --Soft 原理

### Git 三个区域

```text
工作区 (Working Directory)
    ↓ git add
暂存区 (Staging Area / Index)
    ↓ git commit
本地仓库 (Repository)
```

### Git Reset 三种模式对比

| 模式  | 工作区 | 暂存区 | HEAD | 说明  |
| --- | --- | --- | --- | --- |
| `--soft` | ✅ 保留 | ✅ 保留 | ⏪ 回退 | 只移动 HEAD，保留所有更改 |
| `--mixed` | ✅ 保留 | ❌ 清空 | ⏪ 回退 | 移动 HEAD + 清空暂存区（默认） |
| `--hard` | ❌ 丢弃 | ❌ 清空 | ⏪ 回退 | 危险！所有更改丢失 |

### 为什么 --soft 可以合并 commit？

**核心原理：**

1.  **只移动指针，不丢失代码**
    
    -   `--soft`: 只移动 HEAD → 更改保留在暂存区
        
2.  **暂存区 = 未提交的更改**
    
    -   Git 把所有 commit 的 diff 保存到暂存区
        
    -   相当于"撤销提交，但保留代码"
        
3.  **重新提交 = 合并所有更改**
    
    -   新 commit 包含暂存区的所有内容
        
    -   自动产生一个统一的 commit
        

**图解：**

```text
原始：A ← B ← C ← D (HEAD)

执行 git reset --soft A

结果：A (HEAD) + [暂存区包含 B+C+D 的所有代码]

新提交：A ← NEW (包含 B+C+D 的所有内容)
```

---

## 三、合并中间的 Commit（不包含最近的）

### 场景：A ← B ← C ← D (HEAD)，只合并 B+C，保留 D

**使用** `git rebase -i`**（交互式变基）**

```bash
# 1. 启动交互式变基
git rebase -i <base-commit>

# 2. 编辑器会显示：
pick 0c49f9d fix(extract): 优化匹配逻辑
pick 651d85e fix(extract): 修复空行问题
pick 7c2b3d5 feat(extract): 添加类型注解

# 3. 修改为（将第二个 pick 改为 fixup）：
pick 0c49f9d fix(extract): 优化匹配逻辑
fixup 651d85e fix(extract): 修复空行问题
pick 7c2b3d5 feat(extract): 添加类型注解

# 4. 保存退出（:wq）
```

### Rebase 命令详解

| 命令  | 说明  | 使用场景 |
| --- | --- | --- |
| `pick` | 保留此 commit | 默认，保持不变 |
| `squash` | 合并到此 commit | 合并并保留提交信息 |
| `fixup` | 合并到此 commit | 合并但丢弃提交信息 |
| `drop` | 删除此 commit | 移除不需要的提交 |
| `reword` | 修改提交信息 | 只改消息，不改内容 |
| `edit` | 停下来编辑 | 可以修改 commit 内容 |

---

## 四、快速参考卡片

### 选择合适的合并方法

#### 场景1：合并最近的 N 个 commit

-   **使用：** `git reset --soft <base>`
    
-   **示例：** 合并最后 3 个
    
    ```bash
    git reset --soft HEAD~3
    git commit -m "合并后的信息"
    ```
    
-   **优点：** 简单、快速、安全
    

#### 场景2：合并中间的 commit

-   **使用：** `git rebase -i <base>`
    
-   **示例：** A ← B ← C ← D，合并 B+C，保留 D
    
    ```bash
    git rebase -i A
    # 编辑：pick B, fixup C, pick D
    ```
    
-   **优点：** 灵活、可选择、精确
    

---

## 五、实际案例总结

### 合并前（8 个 commit）

```text
d20e8c0 refactor(extract): 简化缩进处理逻辑
43913cf fix(extract): 移除注释后的空行
357d160 fix(extract): 修复注释和代码之间多余的空行
...（更多类似的小调整）
```

### 合并后（2 个 commit）

```text
d810f9d feat(extract): 实现 Lua 类型注解自动注入功能
976b407 feat(extract): 实现 DST scripts.zip 自动解压工具
```

**代码统计：**

-   合并前：8 个 commit，反复调整
    
-   合并后：2 个 commit，功能清晰
    
-   新增 181 行代码 = 之前 7 个 commit 的总和
    

---

## 六、注意事项

⚠️ **重要提醒：**

-   不要对已推送的远程 commit 使用 reset 或 rebase
    
-   `--hard` 模式会丢失代码，慎用
    
-   操作前确保了解每个命令的作用
    

---

## 七、优点总结

**git reset --soft：**

-   ✅ 简单直观，2 条命令完成
    
-   ✅ 不会丢失代码（--soft 模式安全）
    
-   ✅ 可以重新编写提交信息
    
-   ✅ 适合整理频繁的小提交
    

**git rebase -i：**

-   ✅ 可以精确选择要合并的 commit
    
-   ✅ 可以删除、重排、修改 commit
    
-   ✅ 适合整理复杂的提交历史