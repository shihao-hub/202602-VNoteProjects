## Claude Code 命令

### `/init`

让 Claude Code 把整个文件夹下面的所有文件通读一遍，生成一个 CLAUDE.md 文件。

Claude Code 会分析当前文件夹，把它学到的关于项目的知识都保存到 CLAUDE.md 文件中。

**后续和 Claude Code 的所有对话都会带上这个文件作为上下文。**

这个文件有助于帮助 AI 更快地理解整个项目（我们也可以手动修改这个文件）。

### `/compact {prompt}`

用来压缩对话的上下文（{prompt}可选），Claude Code 会将之前对话中的无关紧要的内容排除掉，

这样可以有效提高 AI 的专注力（注意，不要总是新开窗口，上下文详细才能更准确地回答问题！）

### `/clear`

清除跟 AI 的对话记录（为什么不新建对话？）

### `think < think hard < think harder < ultrathink`

官方支持的，控制模型思考长度的方法。

在开始一些比较困难的推理任务之前，可以加上这些思考的提示词！

### `!`

把对话窗口切换成命令行模式，这样可以执行一些临时的命令行命令

优点之一：执行命令的结果和过程是包含在上下文中的，AI 可以从对话历史中获取信息。

### `#`

进入记忆模式，接下来我们输入的话都会被 Claude Code 作为文件的形式记录下来，变成 AI 的长期记忆。

### `~/.claude/CLAUDE.md（区别于 ./claude/CLAUDE.md 和 ./CLAUDE.md）`

用户级别的记忆，对所有的项目都生效，比如：

```txt
- 用中文回答用户的问题
```

### `/ide（重要命令）`

ide 集成，不过 ide 中需要安装 Claude Code 插件，这样就可以把 Claude Code 和 ide 打通了。

1.  在 ide 中选中的代码，Claude Code 这边都能感知到！（实践发现需要 ide 的处于打开状态且工作目录和 Claude Code 目录一致）
    
2.  Claude Code 代码修改时，ide 中也会出来页面对比，然后再 Claude Code 中选择是否接受
    

### `claude -p {prompt}`

非交互模式，开启临时的一次性对话（后台执行，结束响应）

## 把 MCP 安装到 Claude Code 中

详细内容从视频 5:30 开始看

### context7

用来帮助 AI 查找最新代码文档的 MCP Server

```bash
# {name} 自己取，但是依旧起 context7 即可
# -- npx 及其后面的内容是 mcp 启动参数，每个 mcp server 有自己的启动参数
# --scope user 是将 mcp server 设置成 user 级别，而非当前 project 级别
# 有的 mcp server 还需要配置 API Key
claude mcp add {name} --scope user -- npx @upstash/context7-mcp 
```

安装完成后，重启 Claude Code，输入 /mcp 即可看到 context7（此时还有其余三个，那是智谱帮忙安装的好像）

输入 prompt：`tailwind4 如何配置？使用 context7`，即可使用 context7 去查找。

> 我推测，mcp server 就是个公开的网络服务，供各个 AI 使用！

context7 可以针对性查找最新代码文档，所以有了它的帮助，可以让 AI 将旧版本代码改造成新版本代码！

通过主动查找最新代码文档嵌入上下文思考！

prompt 举例：`把当前目录的项目改造成 tailwind v4（使用 context7）`

### 删除 MCP

退出 Claude Code，执行 `claude mcp remove {name}` 即可

### 远程调用 MCP

sse/streamable http 协议的 mcp server 可以通过命令被远程调用（无需本地安装），具体调用方式需阅读对应 mcp server 的文档

## 数据库 MCP

### neon

> 具体查看技术爬爬虾的往期视频

```bash
claude add mcp {name} -- npx @neondatabase/mcp-server-neon start {API_KEY}
```

prompt: `我在 neon 上有哪些表`

预取结果：调用 neon 并告诉我在 neon 数据库中有哪些表

## 参考视频

```txt
【最火AI编程Claude Code详细攻略，一期视频精通】 https://www.bilibili.com/video/BV1XGbazvEuh/?share_source=copy_web&vd_source=66b627ae8b4ac6377f3b4eaa01e690b1
```

## 权限控制

### `/permissions`

可以自定义一些规则，Allow | Deny | Workspace 等，

比如将 Bash(git commit: \*) | mcp\_\_neon 规则添加进 Allow，Claude Code 调用时就不需要征求人的同意。

### `claude --dangerously-skip-permissions`

赋予 Claude Code 最高的权限，它可以使用任意的工具，执行任意的命令！

## 自定义命令

Claude Code 的斜线命令除了内置的外，我们还可以定义自己的命令：

打开项目目录，找到 .claude 目录，新建 commands 目录，在这里面就可以以文件的形式来自定义命令了， 文件的名字就是命令的名字，举个例子：

code\_review.md（自然语言编写内容即可）

```markdown
对比这个分支：$ARGUMENTS，与 main 分支的差异，并且提出你的 review 意见。
```

文本也可以传递参数，如 $ARGUMENTS 作为传入参数的占位符。

### 全局自定义命令

在 `~/.claude` 目录中做同样的操作即可

### 使用与总结

关于自定义命令，目前看来就是减少重复操作的自然语言，我也可以这样：

```txt
阅读 @outline.md 文档并调研，进行需求分析（requirements.md）和需求设计（design.md），并进行需求拆分（tasks.md）
```

## Hook

让 Claude Code 在工作过程中的某个特定节点，执行某些特定操作。

比如：写代码时用户经常执行 `npx prettier --check .` 命令去检查当面目录下面的代码格式是否正确， 那么现在我想要 AI 在写完代码后也执行这个命令应该这样做：

打开 .claude 目录，新建 settings.json/settings.local.json 文件（local 优先级更高）：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --check ."
          }
        ]
      }
    ]
  }
}
```

当工具调用完成后（PostToolUse），执行 command

### 更多 Hook Events

Claude Code 官方文档中还列举了很多的触发事件！

可以参考官方文档，配置更多 hook 来辅助开发工作

> 显然，这自然更适合插件开发

## Sub Agent

类似编程里面的子线程，可以让 Claude Code 在后台开启多个子任务来并行执行，加快任务的执行效率。

每个 Sub Agent 都专注于一个小功能，这样就提高了结果的可预测性！

正确使用 Sub Agent 拆解复杂任务，可以提高任务的成功率。

### `/agents`

这个命令执行后，可以创建我们自己的 Sub Agent（Create new agent）

下一步可以选择 `Generate with Claude`，让 Claude Code 为我们生成一个。

直接使用自然语言填上这个 agent 的描述即可（1. 可以执行什么样的任务 2. 希望达成一个什么样的结果）

举例（多个）：

`你是代码审核大师，请帮我比较 git 分支之间的代码差异，提出审核意见`

`你是一个天气预报大师，你用联网工具查询天气`

再下一步选择赋予这个 Sub Agent 什么样的工具权限（可以跳过）

至此，Sub Agent 创建完成！

#### 使用 Sub Agent

实际上，AI 会根据你的描述，**自动拆分任务**（这是很厉害的能力），然后交给指定的 Sub Agent（并行）

这个过程中，每个 Sub Agent 拿到的还是精简的上下文，这样还能提高 Sub Agent 的专注度和结果准确性（AI 也是当上领导了）

## plan

`/plan`

这 /plan该命令将 Claude Code 置于 计划模式，指示 Claude 使用只读操作分析您的代码库， 并在进行更改之前创建计划。这非常适合以下情况：

-   探索不熟悉的代码库
    
-   安全地规划复杂的变革
    
-   审查代码而无需担心被修改
    

在计划模式下，克劳德会分析并提出计划，然后您可以退出计划模式开始实施更改。 您可以使用 ExitPlanMode提示 Claude 开始编写代码的工具。

更多详情请参见[“常用工作流程”](https://code.claude.com/docs/en/common-workflows)页面。

## skills

官方文档的[常用工作流程](https://code.claude.com/docs/en/common-workflows)的 Create custom skills and commands...

完整参考信息，请参阅[技能文档](https://code.claude.com/docs/en/skills)

> 英文文档终究拗口，即使翻译也拗口，这种情况建议继续搜索相关教学视频...

## GitHub 集成

GitHub CLI 命令行工具（github 搜 cli）：

Claude Code 可以借助这个工具，执行对 GitHub 上面的所有操作。

安装完成 github cli 后，执行：`gh repo list`，可以看到自己有哪些项目。

> 注意，似乎只要下载 github cli，无需任何配置，Claude Code 就能自动根据 prompt 使用？

### 全自动修复 Issue

prompt：`查看 issue #1 内容，并且进行修复，创建一个新的修复分支 bugfix001，然后推送到 Github。`

预期结果：

Claude Code 自动完成了 issue 的修改，并创建分支和提交代码

> 这么强？

## 找回历史对话

执行 `/resume` 命令，可以找到历史话题，除此之外，选择话题后按两次 esc，可以跳转到具体某一句话的前面！

但是需要注意的是，这只能回退对话内容，代码不会回退的。

github 上有个开源工具 ccundo，但是可能是 nodejs 版本问题，我无法使用且这个项目 star 还不够多，有待观望。

折中的方案是，让 ai 修改完代码后，自动 git commit，这样就可以随时回退了。

值得注意的是，ccundo 回退是从某个节点开始，往后的对话和代码全部回退，无法再找回。

## 导出对话内容

`/export` -> Copy to clipboard | Save to file

这样我们可以持久化，也可以发送给其他 AI 进行交叉验证。

### QA

导出的对话内容，可以给另一台电脑的 Claude Code 导入吗？等价于同步功能...

## Claudia

基于 Claude Code 打造的开源桌面可视化应用，但是这玩意只能自己源码编译...

当然了，这时候社区大概率会出现：基于 Claudia 的有安装包的项目...

> 当然了，可视化应用我并不需要（这个应用也挺好用的）

本节视频位置：19:17 前后

## 总结

更多使用案例可以参考官方文档，文档也提供了简体中文版。

这个视频的内容几乎涵盖了 Claude Code 的大部分功能了。