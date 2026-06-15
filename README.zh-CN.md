# Heptabase Local Sync Security

**隐私优先、AI 安全、把 Heptabase 持续同步成结构化 Markdown 的本地工具。**

由你决定 AI agent 到底能看到哪些卡片，其余一律进不了它的视野。

[English](README.md) · [繁體中文](README.zh-TW.md) · 简体中文 · [日本語](README.ja.md) · [한국어](README.ko.md) · [Tiếng Việt](README.vi.md) · [Español](README.es.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [العربية](README.ar.md) · [עברית](README.he.md) · [Русский](README.ru.md) · [Українська](README.uk.md)

> 非官方工具，与 Heptabase 无任何隶属或背书关系。目前仅支持 macOS。

---

## 为什么打造这个工具

一开始的目标很单纯：把 Heptabase 接上 Claude Code，让 AI agent 读我的笔记。

官方路径是 Heptabase 自家的 [CLI](https://github.com/heptameta/heptabase-cli-skills) 采 **fail-open** 机制，一旦授权，agent 就能读你整个知识库，第三方工具如 `heptabase-mcp` server 也是同样机制。如果你的知识库每张卡片都能公开就没问题，但只要你像多数人一样，把机密卡片和想给 AI 用的卡片放在一起，这就不可行。

问题的核心在于，隐私墙必须立在「AI 能读到什么」这条边界上，而这条边界落在 Heptabase 之外，在你怎么把笔记喂给 AI 的那一层。所以这类工具真正要做的，是**把机密卡片留在 AI 碰不到的地方，只把其余卡片导出成 AI 可读的 Markdown**。同步笔记只是简单的那一半。

## 命名由来

名字拆成三段，各说一件事，`heptabase` 是数据来源，`local-sync` 说明它的机制，`security` 说明它的角色。

Security 同时带着两层含义。第一层是安全：工具守在 AI 和你的知识库之间，用 fail-closed 机制决定哪些卡片能进 AI 的视野、哪些一律挡在外面，不用担心数据外泄。第二层更像一个尽责的秘书：主动在后台把笔记整理好，按你设定的位置每 15 分钟自动同步，让你打开 AI 工具就直接有内容可用，不必再手动管这件事。守门加庶务，这就是它名字的意思。

## 它做什么

Heptabase Local Sync Security 中本地运作的 Python 程序直接读你本地的 Heptabase 数据库，把选定的卡片写成 Markdown 文件，放到你指定的位置。`launchd` 每 15 分钟跑一次，让 Markdown 与笔记保持同步。AI agent 永远只读导出的 Markdown 文件夹，不碰数据库。

- **直读 live 本地数据库。** Heptabase 在 2025 年底停止提供[自动本地备份](https://support.heptabase.com/en/articles/11064116-how-does-auto-backup-work-in-heptabase)，直读 live DB 因此成为持续本地同步的可靠路径。
- **结构保真转换。** 表格、bullet／todo／toggle 列表、嵌套 section、视频，都是从 Heptabase 的 ProseMirror schema 逆向出来，转成干净 Markdown。
- **任意目的地路由。** 每张白板都能落到自己的文件夹，包含用绝对路径把某张白板直接放进另一个项目，可以通过 AI Agent 协作，决定 python 要指向你本地的哪个位置。

## fail-closed 隐私模型

一张卡片只有命中以下两个明确白名单之一才会导出，默认什么都不读。

| 来源 | 规则 |
|---|---|
| **`whitelist_whiteboards`** | 你点名的白板。只读每张白板『桌面上』的卡片，不钻子白板。要读子白板就把它的名字也加进来。 |
| **`card_map`** | `标题 -> 精准路径` 覆写层。这些标题一律同步，路径优先。 |
| **`blacklist_whiteboards`** | 这些白板上的卡片会在『读内容之前』先被扣除。黑名单优先于白名单，所以误放在两张白板上的卡片仍会被挡下。 |
| **未点名的子白板** | 卡片移进子白板后 `whiteboard_id` 会变，桌面扫描就扫不到。靠结构排除，不靠你记得设规则。 |

更重要的是，每个会碰到卡片标题或内容的查询，都被限制在白名单白板 id 或 `card_map` 标题内。非白名单卡片的标题与内容根本不会被读进内存。

由此导出两个设计原则。**结构排除优于减法排除**，查询碰不到的卡片比读进来再过滤掉更安全。工具设计让你从头到尾不必担心某张卡有没有外泄，靠结构保证，不靠你事后去检查。

## 与其他方案的对照

| | Heptabase Local Sync Security | 官方 Heptabase CLI | 其他导出工具 |
|---|---|---|---|
| 隐私模型 | fail-closed 白名单 | fail-open（整个知识库） | 全量导出 |
| 持续本地同步 | 是（`launchd`，15 分钟） | 按需读取 | 一次性导出 |
| 直读 live 本地 DB | 是 | 视情况 | 常需备份文件 |
| 结构保真 Markdown | 表格、列表、section、视频 | 视情况 | 视情况 |
| 各白板独立目的地路由 | 是，含绝对路径 | 否 | 否 |

这不是 Heptabase 的全面替代品。「比官方好」只在三个切面成立：隐私可控、持续本地同步、结构保真。受众刻意很窄：在意 AI 看到什么、重度使用 Heptabase 的 macOS 用户。如果这就是你，这工具正是为你的情境而生。

## 安装

需求：macOS、Python 3.9+、已安装 Heptabase 桌面 app。

```bash
git clone https://github.com/yyu0310/heptabase-local-sync-security.git
cd heptabase-local-sync-security
cp config.example.json config.json
```

接着编辑 `config.json`（每个字段都有 inline 注释说明）：

1. 确认 `db_path` 指向你的本地 `hepta.db`。
2. 设置 `base_output_dir` 与 `board_output_dir` 为你要写出 Markdown 的位置。
3. 在 `whitelist_whiteboards` 列出要导出的白板。
4. 在 `card_map` 加入需要精准路径覆写的标题。

先跑预览（不写入任何文件）：

```bash
python3 heptabase_sync.py --dry
```

计划看起来正确后，正式执行：

```bash
python3 heptabase_sync.py
```

### 每 15 分钟自动同步

```bash
cp com.example.heptasieve.plist ~/Library/LaunchAgents/
# 编辑刚复制的文件：填入绝对路径，并确认你的 python3 路径
launchctl load ~/Library/LaunchAgents/com.example.heptasieve.plist
```

## 搭配 AI agent 使用

这个工具附带 agent 可读的文档，你可以用对话让 AI coding agent 帮你设置，而不必手动照步骤操作：

- [`AGENTS.md`](AGENTS.md) 与 [`CLAUDE.md`](CLAUDE.md) 告诉 agent 如何理解与设置这个工具。
- [`llms.txt`](llms.txt) 是给 LLM 的文档索引。
- [`skills/setup-heptasieve/`](skills/setup-heptasieve/) 是一个 Claude Code skill，一句话带你走完整套设置。

让你的 agent 指向导出的 Markdown 文件夹，永远不要指向 `hepta.db`。这个切分正是整个工具的重点。

### 情境一：Claude Code 项目助理

你在开发一套量化交易工具，Heptabase 里有三张白板：`交易策略研究`、`系统设计笔记`、`个人财务记录`。你只把前两张加进 `whitelist_whiteboards`，工具每 15 分钟把它们同步到 `/projects/trading/heptabase-export/`。接着在项目的 CLAUDE.md 里加上这个文件夹路径，让 Claude Code 把它当作背景知识读取。财务记录那张白板从头到尾不在白名单，Claude Code 连标题都碰不到。

### 情境二：跨工具的个人记忆层

不同 AI 工具之间没有共同记忆，每次换工具都要从头说明背景。把常用的参考笔记、工作脉络、研究摘要同步成 Markdown，任何支持读取本地文件夹的 AI 工具都能直接取用，几秒内上手。哪些白板进白名单、哪些永远不进，靠设置决定，不靠信任 AI 工具的判断。

## 运作原理

架构细节见 [`ARCHITECTURE.md`](ARCHITECTURE.md)，包含数据流、`build_plan` 里的 fail-closed 顺序、读到哪些数据库表，以及修改程序时必须守住的隐私不变量。

## 限制与揭露

- **schema 脆弱。** 这依赖 Heptabase 内部数据库结构，官方改版可能让它失效。它本质上就是非官方工具。
- **直读 live DB 非官方认可。** 实务上运作良好且为只读，但你该知道这不是受支持的集成方式。
- **仅支持 macOS。** 目前的路径与 `launchd` 设置都以 macOS 为前提。

## 授权

[MIT](LICENSE)。
