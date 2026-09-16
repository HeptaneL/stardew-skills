# Stardew Skills

在 Claude 中与《星露谷物语》角色进行持续对话的 Skill 集合。

角色定义负责"是谁在说话、怎么说"，MCP 工具负责读取真实存档状态，两者配合让角色的回应能和当前的游戏进度对得上——而不是凭空生成一段台词。

## 效果

<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/3ed5f33c-1617-4c74-bada-5bbaab392fb3" />

<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/83e091dc-c1e7-4f86-b053-79f865418da9" />


## 依赖

这套东西需要两个项目配合，缺一不可：

### 1. HelloStardew — 游戏内 Mod

C# 编写的 SMAPI Mod，在游戏里跑一个本地 HTTP 服务，把当前存档状态（时间、天气、位置、关系、物品等）暴露出来。

https://github.com/HeptaneL/HelloStardew

> 没装 / 没进存档 → 读不到任何数据，Skill 无法判断"现在是什么情况"。

### 2. stardew-mcp-server — MCP Server

MCP Server，连上上面那个 HTTP 服务，把存档数据包装成 Claude 可以直接调用的工具。

https://github.com/HeptaneL/stardew-mcp-server

> 没装 → Claude 拿不到工具，只能靠编。

数据流向：

```
星露谷存档
    │
    │  HelloStardew (C# Mod)  ── 本地 HTTP
    ▼
stardew-mcp-server
    │
    │  MCP
    ▼
Claude  ──  结合 character/ 与 skills/ 生成回应
```

## 在 Claude 中使用

### 步骤 1：装 Mod

按 [HelloStardew](https://github.com/HeptaneL/HelloStardew) 的说明安装，然后**进入游戏并读取存档**。Mod 的 HTTP 服务只有在存档加载后才有数据。

### 步骤 2：接 MCP Server

[stardew-mcp-server](https://github.com/HeptaneL/stardew-mcp-server) 是 Python 写的，server 名注册为 `stardew`。

用 CLI 添加：

```bash
claude mcp add stardew --transport stdio \
  --env STARDEW_API_URL=http://127.0.0.1:8788 \
  --env NO_PROXY=localhost,127.0.0.1,::1 \
  --env no_proxy=localhost,127.0.0.1,::1 \
  -- <仓库路径>/.venv/bin/python <仓库路径>/src/stardew_mcp_server/server.py
```

或者写进项目根目录的 `.mcp.json`：

```json
{
  "mcpServers": {
    "stardew": {
      "type": "stdio",
      "command": "<仓库路径>/.venv/bin/python",
      "args": ["<仓库路径>/src/stardew_mcp_server/server.py"],
      "env": {
        "STARDEW_API_URL": "http://127.0.0.1:8788",
        "NO_PROXY": "localhost,127.0.0.1,::1",
        "no_proxy": "localhost,127.0.0.1,::1"
      }
    }
  }
}
```

环境变量说明：

| 变量 | 作用 |
| --- | --- |
| `STARDEW_API_URL` | 指向 HelloStardew Mod 暴露的 HTTP 服务地址，默认端口 `8788` |
| `NO_PROXY` / `no_proxy` | 让 MCP Server 访问本地地址时绕过系统代理 |

> `NO_PROXY` 两行别省。开了代理的环境里，不加这个会连不上 `127.0.0.1`。

### 步骤 3：开始对话

直接用自然语言说话即可，例如：

```
Haley，今天想不想出去拍点照片？
```

Claude 会先调用工具确认当前处境（日期、天气、所在地点、与 Haley 的关系与好感度等），再结合 `character/haley.md` 与 `skills/spouse/SKILL.md` 决定怎么回应。

如果回应像是"刚编的"，先查一下存档是不是没加载——工具会返回 `no_save_loaded`。

### 可用工具

| 工具 | 用途 |
| --- | --- |
| `check_health` | 服务是否在线、存档是否已加载 |
| `get_current_state` | 时间、地点、金钱、体力、天气、技能、背包 |
| `get_current_date` | 当前季节 / 日期 / 星期 |
| `get_household` | 农夫、农场、配偶、宠物、孩子 |
| `get_relationship` | 与村民的好感度、心数、是否结婚 |
| `get_recent_activity` | 最近三个游戏日的活动记录 |
| `get_todays_events` | 今天的生日 / 节日 / 钓鱼赛 / 书商 |
| `get_week_birthdays` | 本周村民生日 |
| `get_birthdays_on_day` | 查询指定季节 + 日期的生日 |
| `get_events_on_day` | 查询指定季节 + 日期的全部事件 |
| `get_month_calendar` | 整个季节的日历 |
| `get_recent_events` | 今天前后若干天的事件 |

## 目录结构

```
character/    角色定义（现在有 haley.md）
skills/       Skill 定义（现在有 spouse/）
memory/       对话记忆（按角色分文件，现在有 haley.md）
mcp/          预留：MCP 相关配置
assets/       截图资源
```

新增角色：在 `character/` 下加一个 `<名字>.md`，写清身份、性格、关系与边界。
新增场景：在 `skills/` 下加一个目录，放 `SKILL.md`。

## 说明

截图放在 `assets/` 目录下，文件名分别为 `screenshot-1.png` 与 `screenshot-2.png`（也可以用别的名字，记得同步改上面的路径）。
