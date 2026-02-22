# Knowledge Base

个人知识库，存放学习笔记和技术文档。

## 项目脚本

### claude_commit_and_push.bat

自动化 Git 提交和推送脚本，通过 Claude CLI 智能处理代码变更：

```bash
claude_commit_and_push.bat
```

**功能：**
- 自动检查 `git status` 查看变更
- 智能生成符合 Conventional Commits 规范的提交信息
- 自动提交并推送到远程仓库
