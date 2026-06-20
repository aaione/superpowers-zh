# 适用于 Kimi Code 的 Superpowers (Superpowers for Kimi Code)

使用 [Kimi Code](https://github.com/MoonshotAI/kimi-code) 配合 Superpowers 的完整指南。

## 安装

Superpowers 已在 Kimi Code 的插件市场中提供。

打开插件管理器：

```text
/plugins
```

进入 `Marketplace` > `Superpowers` 并安装它。

你也可以从本仓库安装：

```text
/plugins install https://github.com/obra/superpowers
```

若要针对尚未发布的 `dev` 版本进行验证，请显式锁定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code 会将插件变更应用到新会话。在安装、更新、启用、禁用或重新加载插件之后，请使用 `/new` 开启一个全新会话。

## 工作原理

Kimi 插件的 manifest 位于 `.kimi-plugin/plugin.json`。

该 manifest 做三件事：

1. 将 Kimi Code 指向现有的 `skills/` 目录。
2. 通过 `sessionStart.skill` 在会话开始时加载 `using-superpowers`。
3. 通过 `skillInstructions` 提供 Kimi 专用的工具映射。

Kimi Code 直接从本仓库读取 Superpowers skills。没有复制 skills、符号链接、hooks 或额外的运行时依赖。

## 工具映射

Skills 描述的是动作，而非硬编码某个运行时的工具名。在 Kimi Code 上它们解析为：

- "Ask the user" / "ask clarifying questions"（询问用户 / 提出澄清问题） -> `AskUserQuestion`
- "Create a todo" / "mark complete in todo list"（创建 todo / 在 todo 列表中标记完成） -> `TodoList`
- "Dispatch a subagent"（派发 subagent） -> `Agent`
- "Invoke a skill"（调用一个 skill） -> Kimi Code 原生的 `Skill` 工具
- "Read a file" / "write a file" / "edit a file"（读文件 / 写文件 / 编辑文件） -> `Read`、`Write`、`Edit`
- "Run a shell command"（运行 shell 命令） -> `Bash`
- "Search file contents"（搜索文件内容） -> `Grep`
- "Find files by path or pattern"（按路径或模式查找文件） -> `Glob`
- "Fetch a URL"（获取 URL） -> `FetchURL`
- "Search the web"（搜索网络） -> `WebSearch`

## 更新

使用 Kimi Code 的插件管理器：

```text
/plugins
```

从中选择 Superpowers 进行更新。更新后请使用 `/new` 开启全新会话。

## 故障排查

### 插件未加载

1. 运行 `/plugins info superpowers` 并查看诊断信息。
2. 确保插件已启用。
3. 安装或更新后请使用 `/new` 开启全新会话。

### 直接从 GitHub 安装使用了旧版本

当存在 GitHub 发布版时，Kimi Code 会为裸仓库 URL 安装最新的 GitHub 发布版。要在下一次 Superpowers 发布之前测试未发布的变更，请显式安装分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### Skills 未触发

1. 确认 `/plugins info superpowers` 显示插件已启用。
2. 使用 `/new` 开启全新会话。
3. 尝试验收 prompt：`Let's make a react todo list`。一个正常工作的安装应在编写代码之前加载 `brainstorming`。
