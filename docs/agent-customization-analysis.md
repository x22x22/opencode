# OpenCode 自定义主 Agent 和子 Agent 支持分析报告

## 结论

**本项目完全支持自定义主 agent（primary agent）和子 agent（subagent）。** 两种类型均可通过 Markdown 文件、配置文件或 CLI 命令创建，无需修改源代码。

---

## Agent 模式说明

OpenCode 将 agent 分为三种模式：

| 模式 | 说明 |
|------|------|
| `primary` | 主 agent，用户可在 TUI/CLI 中直接选择并对话 |
| `subagent` | 子 agent，由主 agent 通过 `task` 工具调用，不出现在主选择界面 |
| `all` | 兼容两种角色，既可作为主 agent 使用，也可被其他 agent 调用 |

源码位置：`packages/opencode/src/agent/agent.ts` 第 31 行

```ts
mode: z.enum(["subagent", "primary", "all"]),
```

---

## 内置 Agent 列表

项目预置了以下内置 agent：

| Agent 名称 | 模式 | 是否隐藏 | 说明 |
|-----------|------|---------|------|
| `build` | primary | 否 | 默认主 agent，支持完整工具集 |
| `plan` | primary | 否 | 规划模式，禁止编辑操作 |
| `general` | subagent | 否 | 通用子 agent，适合多步骤并行任务 |
| `explore` | subagent | 否 | 代码探索子 agent，快速搜索文件和代码 |
| `compaction` | primary | 是 | 上下文压缩（内部使用） |
| `title` | primary | 是 | 会话标题生成（内部使用） |
| `summary` | primary | 是 | 会话摘要生成（内部使用） |

---

## 自定义 Agent 的方式

### 方式一：Markdown 文件（推荐）

在以下目录中放置 `.md` 文件即可创建自定义 agent。文件名即为 agent 名称：

**项目级（仅对当前项目生效）：**
```
<项目根目录>/.opencode/agent/<agent名称>.md
<项目根目录>/.opencode/agents/<agent名称>.md
```

**全局级（对所有项目生效）：**
```
~/.config/opencode/agent/<agent名称>.md
~/.config/opencode/agents/<agent名称>.md
```

**Markdown 文件格式（YAML frontmatter + 系统提示词）：**

```markdown
---
description: 这个 agent 的使用说明（用于 @ 自动补全提示）
mode: all          # primary | subagent | all
model: anthropic/claude-sonnet-4-5   # 可选，指定模型
temperature: 0.7   # 可选
color: "#FF5733"   # 可选，TUI 中显示的颜色
hidden: false      # 可选，是否在 @ 菜单中隐藏
steps: 50          # 可选，最大 agentic 迭代次数
permission:        # 可选，工具权限控制
  bash: allow
  edit: ask
---

你是一个专业的 TypeScript 代码审查专家。

你的职责是...
```

#### 真实示例

本项目 `.opencode/agent/` 目录中已有以下自定义 agent：

- **docs.md** — 文档写作专家 agent（默认模式）
- **translator.md** — 翻译 agent（`mode: subagent`，专用子 agent）
- **triage.md** — GitHub issue 分类 agent（`mode: primary`，隐藏）
- **duplicate-pr.md** — PR 重复检测 agent（`mode: primary`，隐藏）

---

### 方式二：配置文件 `opencode.jsonc`

在 `opencode.jsonc` 的 `agent` 字段中内联定义或覆盖 agent 配置：

```jsonc
{
  "agent": {
    // 覆盖内置 agent
    "build": {
      "temperature": 0.3,
      "permission": {
        "bash": "ask"
      }
    },
    // 新建自定义 agent（需配合 Markdown 文件或 prompt 字段）
    "my-reviewer": {
      "mode": "subagent",
      "description": "代码审查子 agent",
      "model": "anthropic/claude-haiku-4-5",
      "steps": 20
    }
  },
  // 设置默认主 agent
  "default_agent": "build"
}
```

配置项说明（源码：`config.ts` 第 573-660 行）：

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `model` | string | 使用的模型，格式 `provider/model` |
| `variant` | string | 模型变体 |
| `temperature` | number | 温度参数 |
| `top_p` | number | Top-P 参数 |
| `prompt` | string | 系统提示词 |
| `description` | string | agent 描述 |
| `mode` | string | `primary` / `subagent` / `all` |
| `hidden` | boolean | 是否隐藏 |
| `color` | string | 显示颜色 |
| `steps` | number | 最大迭代步数 |
| `permission` | object | 工具权限规则 |
| `disable` | boolean | 是否禁用此 agent |
| `options` | object | 自定义扩展选项 |

---

### 方式三：Mode 目录（主 agent 快捷方式）

放置在 `mode/` 或 `modes/` 目录中的 `.md` 文件会被自动识别为主 agent（强制 `mode: primary`）：

```
<项目根目录>/.opencode/modes/<agent名称>.md
~/.config/opencode/modes/<agent名称>.md
```

这等同于使用 `mode: primary` 的 agent 文件。

---

### 方式四：CLI 命令（交互式创建）

使用 `opencode agent create` 命令，通过 AI 自动生成 agent 配置：

```bash
# 交互式
opencode agent create

# 非交互式（全参数）
opencode agent create \
  --description "TypeScript 代码审查专家" \
  --mode subagent \
  --tools "bash,read,grep,glob" \
  --path .opencode/agent \
  --model anthropic/claude-haiku-4-5
```

该命令会：
1. 调用 LLM 生成系统提示词和配置
2. 让用户选择工具集和模式
3. 将结果写入 Markdown 文件

---

## 主 Agent 的选择与切换

### 默认主 agent

在 `opencode.jsonc` 中通过 `default_agent` 指定默认主 agent（必须是 `primary` 或 `all` 模式且非隐藏）：

```jsonc
{
  "default_agent": "plan"
}
```

若未设置，默认使用 `build` agent。

### TUI 中切换

在 TUI 界面中，默认快捷键：
- `<leader>a`：查看 agent 列表
- `Tab`：切换到下一个 agent
- `Shift+Tab`：切换到上一个 agent

### 在 @ 菜单中调用子 agent

输入 `@agent名称` 可以在当前对话中指定子 agent 执行某个任务。

---

## 子 Agent 调用机制

主 agent 通过内置的 `task` 工具调用子 agent（源码：`session/prompt.ts`）：

```
主 agent → task 工具 → 子 agent（独立 session）→ 返回结果
```

- 子 agent 拥有独立的权限规则（`permission`）
- 子 agent 的系统提示词可以完全自定义
- 子 agent 可以使用不同的模型
- 对 `task` 工具的调用可以通过权限系统控制（`permission.task: deny/allow/ask`）

`general` 和 `explore` 是最典型的内置子 agent，主 agent 在需要并行探索或执行多步骤任务时会自动调用它们。

---

## 权限控制

每个自定义 agent 都可以设置细粒度的工具权限，格式如下：

```jsonc
{
  "permission": {
    "*": "allow",           // 默认允许所有
    "bash": "ask",          // 执行 bash 需要用户确认
    "edit": "deny",         // 禁止编辑文件
    "read": {
      "*": "allow",
      "*.env": "ask"        // 读取 .env 文件需确认
    },
    "external_directory": {
      "*": "ask",
      "/tmp/*": "allow"     // 允许访问 /tmp
    }
  }
}
```

权限值：`allow`（允许）/ `deny`（拒绝）/ `ask`（询问用户）

---

## 配置加载优先级

配置按以下顺序合并（后者覆盖前者）：

1. 全局配置目录（`~/.config/opencode/`）中的 `opencode.jsonc`
2. 全局 `agent/` 目录中的 Markdown 文件
3. 项目 `.opencode/opencode.jsonc`
4. 项目 `.opencode/agent/` 目录中的 Markdown 文件
5. 环境变量 `OPENCODE_CONFIG_CONTENT`（最高优先级）

---

## 总结

| 功能 | 是否支持 |
|------|---------|
| 自定义主 agent（primary） | ✅ 完全支持 |
| 自定义子 agent（subagent） | ✅ 完全支持 |
| 覆盖内置 agent 配置 | ✅ 完全支持 |
| 禁用内置 agent | ✅ 支持（`disable: true`） |
| 自定义 agent 的系统提示词 | ✅ 支持 |
| 自定义 agent 的模型 | ✅ 支持 |
| 自定义 agent 的权限 | ✅ 支持 |
| AI 自动生成 agent | ✅ 支持（`opencode agent create`） |
| 设置默认主 agent | ✅ 支持（`default_agent`） |
| 主 agent 切换（TUI） | ✅ 支持 |
| 子 agent 被主 agent 调用 | ✅ 支持（通过 `task` 工具） |
| 全局 / 项目级别 agent | ✅ 支持（两个层级） |
