# Claude Skills 安装记录

## 安装时间
2026-02-06

## 安装命令
```powershell
npx skills add https://github.com/axtonliu/axton-obsidian-visual-skills --skill excalidraw-diagram
```

## 已安装的 Skills

### 1. excalidraw-diagram
- **源地址**: https://github.com/axtonliu/axton-obsidian-visual-skills.git
- **功能**: 生成 Excalidraw 图表
- **安装位置**: `~\.agents\skills\excalidraw-diagram`
- **支持的客户端**:
  - Amp
  - Codex
  - Gemini CLI
  - GitHub Copilot
  - Kimi Code CLI
  - OpenCode

### 2. find-skills (自动安装)
- **源地址**: https://github.com/vercel-labs/skills.git
- **功能**: 帮助发现和推荐可安装的 skills
- **安装位置**: `~\.agents\skills\find-skills`
- **支持的客户端**: 同上

## 安装说明
- **安装方式**: Symlink (符号链接)
- **安装范围**: 全局安装
- **注意事项**: Skills 运行时具有完整的代理权限，使用前需要审查

## Excalidraw Diagram Skill 功能
根据系统提示，该 skill 支持以下触发词：
- "Excalidraw"
- "画图"
- "流程图"
- "思维导图"
- "可视化"
- "diagram"
- "标准Excalidraw"
- "standard excalidraw"
- "Excalidraw动画"
- "动画图"
- "animate"

## 使用建议
安装完成后，可以通过上述触发词来调用 Excalidraw 图表生成功能。
