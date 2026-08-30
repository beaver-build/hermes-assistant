# assistant — Hermes 个人日程助手

一个 [Hermes Agent](https://hermes-agent.nousresearch.com/) 的 profile（预配置好的助手）：你用自然语言说「明天下午三点开会」「记一下这个想法」，它就通过滴答清单官方 MCP 帮你写进[滴答清单](https://dida365.com/)，也能查日程、改任务、对账进度。

安装分三步：**装 Hermes → 装这个 profile → 填滴答清单的 token**。全程在终端（macOS 的「终端」、Windows 的 PowerShell）里操作，复制粘贴即可。

---

## 第 1 步：安装 Hermes Agent

先看看有没有装过。在终端里输入：

```bash
hermes --version
```

- 能看到版本号（需要 **0.20.0 及以上**）→ 跳到第 2 步。
- 提示「command not found」或版本太低 → 按下面安装。

### macOS / Linux

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

装完后**关掉终端重新打开**（或执行 `source ~/.zshrc`），再运行 `hermes --version` 确认。

### Windows

在 PowerShell 里执行：

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

### 不想用命令行？

官网有 macOS / Windows 桌面版安装包：<https://hermes-agent.nousresearch.com/>。装好后桌面版自带终端，后面的命令在里面执行即可。

> 遇到问题先跑 `hermes doctor`，它会检查依赖并给出修复建议。完整安装文档：<https://hermes-agent.nousresearch.com/docs/getting-started/installation>

---

## 第 2 步：安装这个 profile

### 2.1 安装

```bash
hermes profile install github.com/beaver-build/hermes-assistant --alias
```

命令会先展示这个 profile 包含什么（人格文件、一个 skill、一个 MCP 配置），按回车确认。`--alias` 会顺便生成一个快捷命令 `assistant`，等价于 `hermes -p assistant`。

### 2.2 初始化（选模型等）

```bash
hermes -p assistant setup
```

这是一个交互式向导，会问你：

| 向导问题 | 怎么选 |
|---|---|
| **LLM 提供商 / 模型** | 你有哪家的 API key 就选哪家（OpenAI、Anthropic、DeepSeek、OpenRouter……都行）。没有的话选 Nous Portal 走网页登录。 |
| **工具 / 记忆 / 会话重置等** | 不确定就一路回车用默认值，之后随时可以重新运行 `setup` 改。 |

这个 profile 不绑定任何模型，模型好坏直接影响日程解析的准确度，建议选当前较强的模型。

---

## 第 3 步：填滴答清单的 token

助手是通过滴答清单**官方 MCP**（`https://mcp.dida365.com`）读写你的清单的，需要一个 API Token。

### 3.1 生成 token

1. 用浏览器打开滴答清单网页版 <https://dida365.com/> 并登录。
2. 点击**左上角头像 → 设置 → 账户**。
3. 找到 **API Token**，点击生成，把生成的 token 复制下来（只显示一次，丢了就重新生成）。

> 官方说明（英文版 TickTick，中文版滴答清单界面相同）：<https://help.ticktick.com/articles/7438129581631995904>

### 3.2 写进 Hermes

```bash
hermes -p assistant mcp add dida365 --url https://mcp.dida365.com --auth header
```

- 提示「Server 'dida365' already exists. Overwrite?」→ 输入 `y`（profile 里预置了这个 MCP，这里只是补 token）。
- 提示「API key / Bearer token」→ 粘贴刚才的 token（输入时不会显示，粘贴完直接回车）。
- 成功后它会连一次滴答清单并列出发现的工具。

token 只会保存在你本机的 `~/.hermes/profiles/assistant/.env` 里，不会进入任何仓库。

<details>
<summary>不想用命令、想手动编辑文件？</summary>

安装后 profile 目录里有一份 `.env.EXAMPLE`，复制成 `.env` 并填入：

```
MCP_DIDA365_API_KEY=你的token
```

文件路径可以用 `hermes -p assistant config env-path` 查到。
</details>

### 3.3 验证

```bash
hermes -p assistant mcp test dida365
```

看到「连接成功」和一串 `list_projects / create_task …` 工具名就说明通了。

---

## 开始使用

```bash
assistant            # 等价于 hermes -p assistant
```

试试对它说：

- 「明天上午十点和老王开会」
- 「记个想法：把周报改成自动生成」
- 「今天还有什么安排」

### 接到微信 / Telegram / Discord 等聊天软件（可选）

这个 profile 本身不绑定聊天平台。想在手机上用，再跑一次 `hermes -p assistant setup` 选择聊天平台，或参考官方文档：<https://hermes-agent.nousresearch.com/docs/user-guide/profiles>

### 定时任务（可选）

助手默认假定有两个定时任务：每天凌晨把没做完的事顺延到今天、每天早上推送当日日程。这些**不随 profile 分发**，需要自己创建，约定和 prompt 写法见 [`skills/productivity/personal-assistant-intake/references/automation.md`](skills/productivity/personal-assistant-intake/references/automation.md)，用 `hermes -p assistant cron` 管理。

---

## 常见问题

**装完 `hermes` 命令找不到？** 重新打开终端；还不行就跑安装脚本末尾提示的 `source` 命令。

**`mcp test` 连接失败？** 大概率 token 粘错了或已失效：回滴答清单重新生成，再跑一次 3.2 的命令。

**想换模型？** `hermes -p assistant setup` 重新选，或 `hermes -p assistant model`。

**想删掉重来？** 你的滴答清单数据不受影响。注意 Hermes 删除后会给这个名字留一个标记，直接重装会"看不见"，所以按这个顺序：

```bash
hermes profile delete assistant
hermes profile create assistant
hermes profile install github.com/beaver-build/hermes-assistant --name assistant --force --alias
```

## 安装之后就是你的了

这个 profile 是一个**起点**，不是需要跟着仓库升级的软件。装完后：

- `SOUL.md`、skill、配置都在 `~/.hermes/profiles/assistant/` 里，想怎么改就怎么改；Hermes 自己也会在使用中调整它们。
- **不要**跑 `hermes profile update assistant` 来"升级"——它会用仓库版本覆盖你改过的 `SOUL.md` 和 `personal-assistant-intake` skill（其他文件和 `config.yaml` 不动）。这个仓库的新版本是给新用户的新起点，不是给你的补丁。
- 想试试仓库的新版本：装成另一个 profile，`hermes profile install github.com/beaver-build/hermes-assistant --name assistant2`。

## 包含内容

| 路径 | 说明 |
|---|---|
| `SOUL.md` | 人格与工作规则（入库规则、时间约定、回复格式） |
| `config.yaml` | 只含 dida365 MCP 配置，token 通过 `MCP_DIDA365_API_KEY` 注入 |
| `skills/productivity/personal-assistant-intake/` | 滴答清单复杂操作与定时任务约定；Hermes 自带 skill 不重复打包，安装后自动就位 |
| `assets/avatar.png` | 头像 |

---

## 维护者说明

本仓库是一次性起点，不与任何已安装的实例同步。发布新版本时：把值得分发的 `SOUL.md` / `personal-assistant-intake` 改动复制进来，去掉所有个人信息（名字、本机路径、邮箱、聊天平台、时区、具体钟点、个人事项和生活细节例子），用 `git grep` 反查确认无残留，再改 `distribution.yaml` 的 `version`，commit → tag → push。`config.yaml` 只含 `mcp_servers`；不要加入 cron、`.env`、memories、Hermes 自带 skill 或 `skills/.bundled_manifest`。
