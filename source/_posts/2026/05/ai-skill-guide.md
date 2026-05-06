---
title: AI Skill + MCP 开发实战指南——从零搭建你的第一个 AI 业务助手
date: 2026-05-06 16:00:00
tags:
- ai skill
- AI Agent
- MCP
categories:
- AI
---

### 前言

你有没有试过让 ChatGPT 直接调用你们公司的内部 API？大概率会碰壁——模型不知道你们的接口长什么样，不懂业务规则，甚至会编造不存在的参数。这就是 **Skill（技能）+ MCP（Model Context Protocol）** 要解决的问题。

本文不会罗列概念定义，而是带着你从零开始，用一套完整的伪代码示例，搭建一个能真正跑通的 AI 业务助手。读完之后，你应该能够：

- 理解 Skill 的加载机制和编写规则
- 掌握 MCP 协议的核心原理与服务端开发方法
- 用 Skill + MCP 组合完成一个端到端的业务场景

我们选用的示例场景是：**一个简单的待办事项管理助手**——创建、查询、修改、删除待办，覆盖 CRUD 全流程。

<!-- more -->

---

### 第一章：为什么大模型需要"操作手册"

#### 1.1 一个真实的痛点

假设你是一家 SaaS 公司的后端开发，产品经理提了个需求：让 AI 助手帮用户管理待办事项。你的后端已经有一套 RESTful API：

```
POST   /api/todos          创建待办
GET    /api/todos          查询列表
PUT    /api/todos/:id      修改待办
DELETE /api/todos/:id      删除待办
```

最直觉的做法是把 API 文档塞进 System Prompt：

```text
系统提示词：
你是一个待办管理助手。
API 列表：
1. POST /api/todos {title, priority, due_date} → 返回 {id, title}
2. GET /api/todos?status=active → 返回 [{id, title, status}]
3. PUT /api/todos/:id {title?, priority?} → 返回更新结果
4. DELETE /api/todos/:id → 返回删除结果
...
```

这样做有四个致命问题：

| 问题 | 后果 |
|------|------|
| Prompt 太长 | 每次请求都消耗 Token，成本线性增长 |
| 规则无法强制执行 | 用户说"帮我全删了"，模型可能真去调 DELETE 循环 |
| 多个业务混在一起 | 待办、日历、通知的 API 全塞进同一个 Prompt，互相干扰 |
| 授权流程无法落地 | 企业 API 几乎都需 OAuth2.0 等鉴权，Token 不可能写死在 Prompt 里，也不能让模型向用户索取凭证 |

Skill 的核心思想就是：**不要把所有东西一次性喂给模型，而是按需加载、按场景隔离。**

#### 1.2 Skill 是什么

Skill 就是一份结构化的 Markdown 文件（`SKILL.md`），它告诉大模型：在什么场景下触发、该遵循什么流程、有哪些约束规则。

可以把 Skill 理解为大模型的 **岗位说明书 + SOP 操作手册**。它不是代码，而是一种人类可写、机器可读的"行为契约"。

#### 1.3 Skill 的加载原理

理解加载机制，才能写出高效的 Skill：

```
用户输入: "帮我查一下今天的待办"
           │
           ▼
    ┌──────────────────┐
    │  Agent 路由层     │  ← 扫描所有已注册的 Skill 头部
    └──────┬───────────┘
           │ 关键词 "待办"/"查询" 命中 todo-manager Skill
           ▼
    ┌──────────────────┐
    │  加载 SKILL.md   │  ← 将整个文件内容注入到 System Prompt
    │  注入上下文       │
    └──────┬───────────┘
           │
           ▼
    ┌──────────────────┐
    │  大模型推理       │  ← 根据 SKILL.md 中的 SOP 决定下一步动作
    │  输出工具调用     │
    └──────┬───────────┘
           │
           ▼
    ┌──────────────────┐
    │  任务结束         │  ← 释放 SKILL.md，上下文恢复原状
    └──────────────────┘
```

关键点：

- **按需注入**：只有命中的 Skill 才会进入上下文，其他 Skill 不占 Token
- **完整注入**：不是摘要，是 `SKILL.md` 的全文
- **任务级生命周期**：一个对话轮次（或一个完整任务）结束后释放

#### 1.4 Skill 的目录结构

一个标准的 Skill 长这样：

```
todo-manager-skill/
├── SKILL.md              # 【必填】核心文件：触发词 + 业务 SOP
├── mcp.json              # 【按需】MCP 服务连接配置
├── references/           # 【选填】辅助知识库
│   ├── error_codes.md    #   错误码映射表
│   └── priority_rules.md #   优先级业务规则
├── scripts/              # 【选填】本地脚本
│   └── format_date.sh
└── assets/               # 【选填】模板、图片等静态资源
```

每个文件各司其职，后面章节会逐个讲解怎么写。

---

### 第二章：SKILL.md 编写详解

#### 2.1 头部元数据（Front Matter）

`SKILL.md` 的头部使用 YAML 格式定义元数据，这是路由层用来做匹配的关键信息：

```yaml
---
name: todo-manager                    # Skill 的唯一标识符
version: 1.0.0                        # 版本号，灰度发布时用于区分新旧版
author: "DevTeam"
description: >
  当用户要求创建、查询、修改或删除待办事项时触发。
  支持按状态筛选、按优先级排序、设置截止日期。
trigger_keywords:                     # 触发关键词列表（精确匹配优先级最高）
  - "新建待办"
  - "添加待办"
  - "我的待办"
  - "查待办"
  - "改待办"
  - "删除待办"
  - "取消待办"
---
```

字段说明：

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 全局唯一，建议用 kebab-case 命名 |
| `version` | 是 | 语义化版本号，网关据此做灰度控制 |
| `description` | 是 | 用于语义匹配兜底，写得越具体匹配越准 |
| `trigger_keywords` | 否 | 精确关键词列表，命中则直接加载 |

> **触发机制有两层**：第一层是 `trigger_keywords` 的精确匹配（最快）；如果关键词没命中，路由层会用 `description` 做语义相似度计算（兜底）。所以 `description` 不要敷衍，要写清楚这个 Skill 到底覆盖哪些场景。

#### 2.2 正文：SOP 编写

正文部分是真正的操作手册，直接用 Markdown 写业务流程即可。下面是一份完整的 `SKILL.md` 示例（接在头部元数据之后）：

```markdown
# 待办管理助手

## 触发条件

当用户要求创建、查询、修改或删除待办事项时触发本 Skill。

## 核心工作流

### 场景 A：创建待办

- **用户意图**："帮我建个待办"、"加个任务：周五前完成 PPT"、"提醒我明天开会"
- **前置检查**：
  1. 标题（title）：**必填**，缺失时反问"你想给这个待办起什么名？"
  2. 截止日期（due_date）：选填，建议主动询问
  3. 优先级（priority）：选填，默认 "medium"
- **调用工具**：`create_todo`
- **成功回复**："已创建{priority}优先级待办：{title}，ID: {id}，截止日期：{due_date}"
- **失败处理**：将 MCP 返回的错误信息转述给用户

### 场景 B：查询待办

- **用户意图**："今天有什么待办"、"我的待办列表"、"看看还有啥没做完"
- **调用工具**：`list_todos`
- **默认参数**：`status=active`, `limit=5`
- **回复格式**：将返回的 JSON 渲染为表格：

  | ID | 标题 | 优先级 | 截止日期 | 状态 |
  |----|------|--------|---------|------|
  | todo_001 | 完成PPT | 高 | 2026-05-10 | 进行中 |
  | todo_002 | 代码评审 | 中 | 2026-05-08 | 进行中 |

  超过 5 条时追加提示："仅展示前 5 条，共 N 条。如需更多请告知筛选条件。"

### 场景 C：修改待办

- **用户意图**："改一下那个待办"、"把优先级调高"
- **流程**：
  1. 先调 `list_todos` 展示当前列表（帮助定位目标）
  2. 若用户未指明具体哪条，追问："你想修改哪一条？"
  3. 确认目标后，询问新的参数值
  4. 调用 `update_todo(id, new_params)`
- **约束**：一次只修改一条，不支持批量操作

### 场景 D：删除待办 —— ⚠️ 高危操作

- **用户意图**："删掉那个PPT待办"、"取消了"
- **强制流程（必须严格执行）**：
  1. 调用 `list_todos` 展示当前列表
  2. 向用户确认目标："你要删除的是「{title}」(ID: {id}) 吗？"
  3. **获得明确同意后**才调用 `delete_todo(id)`
  4. 用户说"全部删掉" → **拒绝**，回复："出于安全考虑不支持批量删除，请逐条确认"
- **成功回复**："已删除待办：「{title}」"

## 全局约束

- 所有时间参数统一使用 ISO 8601 格式
- 不确定的参数必须向用户确认，禁止编造值
- 涉及资金、权限变更、数据删除的操作均需二次确认
```

#### 2.3 编写红线清单

根据实践经验总结的规则：

| # | 规则 | 为什么 | 反例 |
|---|------|--------|------|
| 1 | **高危操作必须二次确认** | 模型可能过度顺从用户 | 用户说"删了吧"→直接调 delete |
| 2 | **时间统一用 ISO 8601** | 模型算不准 Unix 时间戳 | `"due_date": 1741234567` |
| 3 | **枚举值必须穷举（输入）+ 可读文本（输出）** | 输入端不写清模型会编；输出端返回数字模型还要查表翻译 | 输入：priority 只写了 high/low 漏了 medium；输出：`status: 2` 而非 `status: "completed"` |
| 4 | **不要在 SOP 里重复写 Schema 或参数映射** | MCP 握手时已注入 Schema，模型知道怎么传参 | SOP 中再抄 JSON Schema 或列举 "高→high" 映射 |
| 5 | **references 只放业务知识** | 技术文档对模型没有额外价值 | 放 HTTP 状态码说明、通用编码规范 |
| 6 | **每步动作要有明确的输出格式** | 不规定输出格式，模型的回复会不可控 | 只写"调用工具"没说拿到结果后怎么呈现 |

---

### 第三章：MCP 协议原理深入

如果说 Skill 是大脑（知道做什么），那 **MCP 就是手脚（负责做到）**。这一章我们从协议层面搞清楚 MCP 是怎么工作的。

#### 3.1 MCP 解决什么问题

大模型本身只能输出文本，不能直接调用你的后端 API。MCP 在中间做了一个标准化翻译层：

```
┌─────────────┐      JSON-RPC 2.0       ┌─────────────┐      HTTP/gRPC    ┌─────────────┐
│   大模型     │ ◄────────────────────► │  MCP Client  │ ◄──────────────► │  后端服务    │
│  (LLM)      │   stdin/stdout 或 SSE   │  (SDK)       │                  │  (Your API)  │
└─────────────┘                          └─────────────┘                  └─────────────┘
                                                │
                                                │ 读取配置
                                                ▼
                                         ┌─────────────┐
                                         │  mcp.json    │
                                         └─────────────┘
```

MCP 做了三件事：

1. **协议标准化**：用 JSON-RPC 2.0 统一通信格式，不管后端是什么技术栈
2. **能力发现**：Client 启动时自动向 Server 询问"你有哪些工具可用"（tools/list）
3. **Schema 透传**：把工具的输入输出定义告诉大模型，让它知道怎么调用

#### 3.2 两种传输方式详解

##### 方式一：stdio（本地子进程模式）

适用场景：开发调试、本地工具、单机部署

```json
// mcp.json
{
  "mcpServers": {
    "todo-service": {
      "command": "node",
      "args": ["/path/to/mcp-server/index.js"],
      "env": {
        "LOG_LEVEL": "debug",
        "API_BASE_URL": "http://localhost:8080"
      }
    }
  }
}
```

工作原理：

```
Agent 启动
    │
    ▼
fork 子进程: node /path/to/mcp-server/index.js
    │
    ├── stdin  ──►  发送 JSON-RPC 请求（如 tools/call）
    └── stdout ◄──  接收 JSON-RPC 响应
```

优点：简单，不需要网络配置，适合本地开发
缺点：只能在同一台机器上运行，无法负载均衡

##### 方式二：SSE（远程 HTTP 模式）

适用场景：生产环境、多实例部署、需要集中鉴权

```json
// mcp.json
{
  "mcpServers": {
    "todo-service": {
      "url": "https://api.yourcompany.com/mcp/todo/sse",
      "headers": {
        "Authorization": "Bearer ${SESSION_TOKEN}"
      }
    }
  }
}
```

工作原理：

```
Agent Client                          MCP Server (远程)
    │                                     │
    │  ─── GET /mcp/todo/sse ────►        │
    │  (建立 SSE 长连接)                   │
    │  ◄── 200 OK (text/event-stream) ──  │
    │                                     │
    │  ─── POST /mcp/todo/message ──►     │  (发送 JSON-RPC 请求)
    │  ◄── SSE event (响应数据) ──────     │
    │                                     │
    │  ... 往复通信 ...                    │
```

优点：支持多实例负载均衡、集中鉴权、独立扩缩容
缺点：需要额外的网络基础设施

> **选型建议**：本地开发和 PoC 阶段用 stdio；生产环境一律走 SSE。

#### 3.3 MCP 协议的核心消息类型

MCP 基于 JSON-RPC 2.0，以下是开发时最常打交道的消息：

##### 1. 握手阶段：能力发现

Client 启动后的第一步是询问 Server 有哪些能力：

```json
// Client → Server: 询问可用工具
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}

// Server → Client: 返回工具列表及 Schema
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "create_todo",
        "description": "创建一个新的待办事项",
        "inputSchema": {
          "type": "object",
          "properties": {
            "title": {
              "type": "string",
              "description": "待办标题，1~100 个字符",
              "minLength": 1,
              "maxLength": 100
            },
            "due_date": {
              "type": "string",
              "description": "截止日期，ISO 8601 格式，如 2026-05-10T09:00:00+08:00",
              "format": "date-time"
            },
            "priority": {
              "type": "string",
              "description": "优先级",
              "enum": ["low", "medium", "high"],
              "default": "medium"
            }
          },
          "required": ["title"]
        }
      },
      {
        "name": "list_todos",
        "description": "查询待办列表，支持按状态和优先级筛选",
        "inputSchema": {
          "type": "object",
          "properties": {
            "status": {
              "type": "string",
              "enum": ["active", "completed", "cancelled"],
              "description": "按状态筛选，不传则返回全部"
            },
            "limit": {
              "type": "integer",
              "description": "返回条数上限，默认 5，最大 20",
              "default": 5,
              "maximum": 20
            }
          }
        }
      },
      {
        "name": "update_todo",
        "description": "修改已有待办事项",
        "inputSchema": {
          "type": "object",
          "properties": {
            "id": {
              "type": "string",
              "description": "待办 ID"
            },
            "title": {
              "type": "string",
              "description": "新标题"
            },
            "priority": {
              "type": "string",
              "enum": ["low", "medium", "high"]
            },
            "status": {
              "type": "string",
              "enum": ["active", "completed"]
            }
          },
          "required": ["id"]
        }
      },
      {
        "name": "delete_todo",
        "description": "删除待办事项（⚠️ 高危操作，需二次确认）",
        "inputSchema": {
          "type": "object",
          "properties": {
            "id": {
              "type": "string",
              "description": "待办 ID"
            }
          },
          "required": ["id"]
        }
      }
    ]
  }
}
```

这段响应就是大模型理解"我能做什么"的唯一来源。Schema 写得越精准，模型调用越准确。

##### 2. 运行阶段：工具调用

当大模型决定调用某个工具时，发送 `tools/call` 请求：

```json
// Client → Server: 调用 create_todo
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "create_todo",
    "arguments": {
      "title": "完成 Q2 季度汇报 PPT",
      "due_date": "2026-05-15T18:00:00+08:00",
      "priority": "high"
    }
  }
}

// Server → Client: 成功响应（Content Block 格式）
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"id\":\"todo_001\",\"title\":\"完成 Q2 季度汇报 PPT\",\"priority\":\"high\",\"status\":\"active\",\"created_at\":\"2026-05-06T16:30:00+08:00\"}"
      }
    ],
    "isError": false
  }
}

// Server → Client: 错误响应（isError=true 让模型知道需要纠错）
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "创建失败：标题长度超过限制（最大 100 字符），当前输入 150 字符。请缩短标题后重试。"
      }
    ],
    "isError": true
  }
}
```

#### 3.4 Schema 编写的黄金法则

大模型完全依赖 Schema 来理解工具用法，以下是从实践中总结的原则：

**原则 1：description 要写成"给模型看的说明书"**

```json
// ❌ 差：太模糊
{"description": "时间"}

// ✅ 好：包含格式、含义、示例
{"description": "截止日期，ISO 8601 格式含时区，如 2026-05-10T09:00:00+08:00"}
```

**原则 2：用 enum 把选项锁死**

```json
// ❌ 差：模型可能传入 "紧急" 或 "urgengt"
{"type": "string", "description": "优先级"}

// ✅ 好：只有三个合法值
{"type": "string", "enum": ["low", "medium", "high"], "description": "优先级"}
```

**原则 3：required 引导模型反问**

```json
// ❌ 差：title 缺失时模型可能传空字符串
{"properties": {"title": {"type": "string"}}}

// ✅ 好：title 缺失时模型知道要问用户
{"properties": {"title": {"type": "string"}}, "required": ["title"]}
```

**原则 4：返回值也要语义化**

```json
// ❌ 差：模型还要去查 3 是什么意思
{"status": 3}

// ✅ 好：模型直接就能理解并呈现给用户
{"status": "cancelled"}
```

**原则 5：错误信息必须说人话**

```json
// ❌ 差：错误码对模型没有意义
{"code": 40001, "message": "INVALID_PARAMS"}

// ✅ 好：自然语言 + 修复建议，模型可以直接转述给用户
{"type": "text", "text": "优先级只能是 low/medium/high 之一，你传入的 'urgent' 不合法，请重新选择。"}
```

#### 3.5 Content Block 类型一览

MCP 协议规定了四种标准返回类型：

```json
// 类型 1：纯文本（最常用）
{ "type": "text", "text": "创建成功！ID: todo_001" }

// 类型 2：图片（base64 编码）
{
  "type": "image",
  "data": "iVBORw0KGgoAAAANSUhEUgAA...",
  "mimeType": "image/png"
}

// 类型 3：资源引用（指向外部 URL）
{
  "type": "resource",
  "resource": {
    "uri": "https://your-cdn.com/reports/q2.pdf",
    "mimeType": "application/pdf",
    "title": "Q2 季度报表"
  }
}
```

> 实际开发中 90% 的场景用 `text` 类型就够了。图片和资源引用适用于报表生成、截图回传等特殊场景。

---

### 第四章：端到端实战——搭建待办管理助手

前面讲了 Skill 和 MCP 各自的原理，这一章我们把它们串起来，用一个完整的伪代码示例演示从用户输入到最终结果的完整链路。

#### 4.1 系统架构总览

```
┌──────────────────────────────────────────────────────────┐
│                      用户                                │
│  "帮我建个高优待办，周五前完成PPT"                         │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   Agent 路由层                            │
│                                                          │
│  1. 分词 → "建"、"待办"、"高优"                           │
│  2. 匹配 trigger_keywords: "新建待办" ✓                   │
│  3. 加载 todo-manager/SKILL.md                           │
│  4. 加载 todo-service/mcp.json → 连接 MCP Server         │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                 大模型上下文                              │
│                                                          │
│  System: [SKILL.md 全文]                                 │
│  Tools:   [create_todo, list_todos, update_todo,         │
│            delete_todo — 含完整 Schema]                   │
│                                                          │
│  User: "帮我建个高优待办，周五前完成PPT"                   │
│                                                          │
│  Model 推理过程:                                          │
│    ① 命中场景 A (创建待办)                               │
│    ② 检查必填项: title="完成PPT" ✓                        │
│    │   due_date: "周五" → 需推算 → 2026-05-09             │
│    │   priority: "高优" → "high"                         │
│    ③ 输出工具调用:                                       │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                MCP 工具调用                               │
│                                                          │
│  → tools/call(create_todo, {                             │
│      title: "完成PPT",                                   │
│      due_date: "2026-05-09T18:00:00+08:00",             │
│      priority: "high"                                    │
│    })                                                    │
│                                                          │
│  ← 返回: { id: "todo_001", ... }                         │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                大模型生成回复                             │
│                                                          │
│  "已创建高优先级待办：完成PPT                              │
│   ID: todo_001                                           │
│   截止日期：2026-05-09（周五）18:00"                       │
└──────────────────────────────────────────────────────────┘
```

#### 4.2 完整交互时序（以删除场景为例）

删除是最容易出问题的场景，我们用它来展示双层防御如何协作：

```
用户: "把那个PPT待办删了"
  │
  ▼
[路由层] 命中 trigger_keywords: "删除待办"
  │
  ▼
[SKILL.md 注入] → 模型读到场景 D 的 SOP：
  "必须向用户复述关键信息并获得明确同意后才调用 delete_todo"
  │
  ▼
[模型推理] → 我不知道用户说的是哪个待办，先查一下
  │
  ▼
[tools/call] list_todos({limit: 5})
  │
  ▼
[MCP Server] 返回:
  [
    {id: "todo_001", title: "完成PPT", priority: "high"},
    {id: "todo_002", title: "代码评审", priority: "medium"}
  ]
  │
  ▼
[模型回复用户]:
  "找到以下待办：
   1. 完成PPT（高优先级）- ID: todo_001
   2. 代码评审（中优先级）- ID: todo_002
   你要删除的是「完成PPT」（ID: todo_001）吗？"
  │
  ▼
[用户]: "对，删吧"
  │
  ▼
[模型推理] → 用户已确认，调用删除
  │
  ▼
[tools/call] delete_todo({id: "todo_001"})
  │
  ▼
[MCP Server 层校验] → 检查请求上下文中的 confirm 标识 ✓
  │                      （第二道防线：即使模型跳过确认也会被拦截）
  ▼
  执行删除 → 返回成功
  │
  ▼
[模型回复]: "已删除待办：「完成PPT」"
```

#### 4.3 MCP Server 伪代码实现

下面是一个最小可用的 MCP Server 伪代码（类 JavaScript/TypeScript 风格），帮助你理解服务端的内部逻辑：

```javascript
// ============================================
// MCP Server 最小实现（伪代码）
// ============================================

const { McpServer } = require('@modelcontextprotocol/sdk');

// 1. 创建 Server 实例
const server = new McpServer({
  name: 'todo-service',
  version: '1.0.0'
});

// 2. 注册工具及其处理函数
server.tool(
  'create_todo',
  '创建一个新的待办事项',  // 这个 description 会暴露给大模型
  {
    // ↓↓ 这就是 InputSchema，决定模型怎么传参 ↓↓
    type: 'object',
    properties: {
      title: {
        type: 'string',
        description: '待办标题，1~100个字符',
        minLength: 1,
        maxLength: 100
      },
      due_date: {
        type: 'string',
        description: '截止日期，ISO 8601 格式',
        format: 'date-time'
      },
      priority: {
        type: 'string',
        enum: ['low', 'medium', 'high'],
        description: '优先级',
        default: 'medium'
      }
    },
    required: ['title']
  },

  // ↓↓ 工具被调用时执行的函数 ↓↓
  async ({ title, due_date, priority }, extra) => {

    // --- 参数校验 ---
    if (!title || title.trim().length === 0) {
      return {
        content: [{
          type: 'text',
          text: '标题不能为空，请提供一个有效的待办名称。'
        }],
        isError: true
      };
    }

    if (title.length > 100) {
      return {
        content: [{
          type: 'text',
          text: `标题超长（${title.length}/100），请缩短后重试。`
        }],
        isError: true
      };
    }

    // --- 业务逻辑：调用后端 API ---
    try {
      const result = await httpClient.post('/api/todos', {
        title: title.trim(),
        due_date: due_date || null,
        priority: priority || 'medium'
      });

      // --- 成功返回 ---
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({
            id: result.data.id,
            title: result.data.title,
            priority: result.data.priority,
            status: result.data.status,
            created_at: result.data.created_at
          })
        }],
        isError: false
      };

    } catch (error) {
      // --- 错误返回：自然语言 + 修复建议 ---
      const msg = mapErrorToHumanMessage(error);
      return {
        content: [{ type: 'text', text: msg }],
        isError: true
      };
    }
  }
);

// 3. 注册删除工具（带二次确认校验）
server.tool(
  'delete_todo',
  '删除待办事项（⚠️ 高危操作）',
  {
    type: 'object',
    properties: {
      id: { type: 'string', description: '待办 ID' }
    },
    required: ['id']
  },
  async ({ id }, extra) => {

    // ★ 第二道防线：检查上下文中是否携带确认标识
    const isConfirmed = extra.context?.metadata?.human_confirm === true;
    if (!isConfirmed) {
      return {
        content: [{
          type: 'text',
          text: '删除操作未经过用户确认，已被系统拦截。请在 Skill 流程中获得用户明确授权后再试。'
        }],
        isError: true
      };
    }

    try {
      await httpClient.delete(`/api/todos/${id}`);
      return {
        content: [{
          type: 'text',
          text: `待办 ${id} 已成功删除。`
        }],
        isError: false
      };
    } catch (error) {
      return {
        content: [{ type: 'text', text: `删除失败：${error.message}` }],
        isError: true
      };
    }
  }
);

// 4. 启动 Server（stdio 模式）
server.connect(stdioTransport);
```

对应地，`mapErrorToHumanMessage` 函数展示了错误信息应该如何转换：

```javascript
/**
 * 将后端错误码转为自然语言描述
 * 核心：让模型能直接转述给用户，而不是暴露原始错误码
 */
function mapErrorToHumanMessage(error) {
  const ERROR_MAP = {
    'NOT_FOUND':          '该待办不存在，可能已被删除。请刷新列表后重试。',
    'DUPLICATE_TITLE':    '已存在相同标题的待办，请换个名字或查看现有列表。',
    'INVALID_DATE':       '日期格式不正确，请使用 ISO 8601 格式，如 2026-05-10T09:00:00+08:00。',
    'RATE_LIMITED':       '操作过于频繁，请等待 30 秒后再试。',
    'FORBIDDEN':          '你没有权限操作此待办。',
    'QUOTA_EXCEEDED':     '待办数量已达上限（100 条），请先清理已完成的项目。'
  };

  const code = error.response?.data?.code;
  return ERROR_MAP[code] || `操作失败：${error.message}，请联系管理员。`;
}
```

#### 4.4 references 文件的正确用法

`references/` 目录不是用来放技术文档的，而是放大模型**记不住的业务知识**：

```markdown
<!-- references/priority_rules.md -->
# 优先级业务规则

## 定义
- **high**：影响项目交付或 SLA 的关键任务，必须在 24 小时内处理
- **medium**：常规工作任务，本周内处理即可
- **low**：锦上添花的优化项，有时间再做

## 自动升级规则
- 截止日期在 24 小时内且 priority 为 low 的待办，
  应自动升级为 medium 并通知用户
```

这种内容不适合硬编码进 SKILL.md（会膨胀），也不属于 Schema（不是参数定义），放在 references 中按需引用刚刚好。

---

### 第五章：安全与生产级实践

#### 5.1 第一条铁律：Token 绝不进 Prompt

这是安全最重要的一条规则。来看对比：

```text
❌ 反模式：在 InputSchema 中定义认证参数
{
  "properties": {
    "user_token": { "type": "string", "description": "用户访问令牌" }
  }
}
后果：
  1. 模型可能在对话中向用户索要 Token
  2. Token 可能被写入日志
  3. 恶意 Prompt Injection 可以诱导模型泄露 Token

✅ 正确做法：凭证在传输层透明传递
用户请求
  → API Gateway 拦截
  → 从 Session/Cookie 中提取 Token
  → 通过 HTTP Header 或 Context 动态注入
  → MCP Server 从请求上下文中获取
  → 对大模型全程不可见
```

#### 5.2 高危操作的双层防御

我们在第四章的删除示例中已经看到了这个模式，这里再抽象一层：

```
第一道防线 — Skill 层（SOP 约束）
  SKILL.md 明确写道：
  "删除前必须向用户复述信息并获得明确同意"
  → 依赖模型的"自觉"执行

第二道防线 — MCP 层（代码强校验）
  Handler 检查 context.metadata.human_confirm == true
  → 不依赖模型，代码级别硬拦截

为什么需要两层？
  → 模型可能因为 Prompt Injection、长对话遗忘等原因跳过确认步骤
  → 代码层的校验是最终的安全网
```

#### 5.3 数据量控制：防止模型"失焦"

大模型的注意力是有限资源。如果你的 `list_todos` 一次返回 200 条待办，会发生：

- Token 窗口被占满
- 模型在中间几条上"注意力丢失"，出现幻觉
- 回应质量断崖式下降

**解决方案**：

```javascript
// 服务端硬限制
async function handleListTodos(params) {
  const limit = Math.min(params.limit || 5, 20);  // 上限 20 条
  const results = await db.query(
    'SELECT id, title, priority, due_date FROM todos WHERE status = ? LIMIT ?',
    [params.status || 'active', limit]
  );

  // 如果还有更多数据，追加提示
  const hasMore = await db.count('todos', { status: params.status }) > limit;

  return {
    data: results,
    meta: hasMore ? { hint: `仅展示前 ${limit} 条，共 ${total} 条。如需更多请指定筛选条件或分页查询。` } : null
  };
}
```

#### 5.4 企业级架构演进

当你的 Skill 数量从 1 个增长到 50 个，就需要架构层面的支撑：

| 架构模式 | 解决的问题 | 核心思路 |
|---------|-----------|---------|
| **编排层 Facade** | 多步原子操作导致模型迷失 | 将"查→确认→删除"封装为一个宏指令 `confirmed_delete`，模型只需一次调用 |
| **API Gateway** | 多租户鉴权、审计、限流 | Gateway 统一注入 Token、记录操作日志、熔断异常调用 |
| **Hub & Spoke 路由** | Context Window 不够放所有 Skill | Hub 总网关根据用户意图分发到 Spoke 子域，每个子域只加载相关 Skill |

---

### 第六章：验证闭环——确保质量的三阶段测试

开发完 Skill 和 MCP Server 之后，不要急着上线。按以下三个阶段验证：

```
╔═══════════════════════════════════════════════════════════╗
║  阶段 1：MCP 工具独立测试（先验证手脚没问题）              ║
║  ─────────────────────────────────────────────────────── ║
║  方法：                                                    ║
║    用 curl/Postman 直接发 JSON-RPC 请求到 MCP Server       ║
║                                                          ║
║  检查项：                                                  ║
║    □ tools/list 能否正常返回所有工具及 Schema              ║
║    □ 正常参数能否正确执行业务逻辑                           ║
║    □ 缺少 required 参数时是否返回友好的错误信息              ║
║    □ enum 值非法时是否被拦截                               ║
║    □ 错误信息是否为自然语言（非错误码）                     ║
║    □ 删除接口无确认标识时是否被拦截                         ║
╠═══════════════════════════════════════════════════════════╣
║  阶段 2：Skill 流程模拟（验证大脑能正确指挥）               ║
║  ─────────────────────────────────────────────────────── ║
║  方法：                                                    ║
║    打开大模型控制台（如 OpenAI Playground）                ║
║    将 SKILL.md 全文粘贴到 System Prompt                    ║
║    手动模拟 Tools 的返回值                                 ║
║                                                          ║
║  测试用例：                                                ║
║    □ 正常创建："帮我建个待办：周五前交报告"                  ║
║    □ 缺参创建："建个待办" → 是否反问标题？                   ║
║    □ 正常查询："我今天有什么待办"                            ║
║    □ 删除流程："把报告那个删了" → 是否列出并确认？            ║
║    □ 批量删除诱骗："全部帮我删掉" → 是否拒绝？               ║
║    □ 歧义处理："改一下那个待办" → 是否追问哪个？             ║
╠═══════════════════════════════════════════════════════════╣
║  阶段 3：端到端全链路整合（验证整体跑通）                   ║
║  ─────────────────────────────────────────────────────── ║
║  方法：                                                    ║
║    在实际客户端加载 Skill + MCP Server，打通完整链路        ║
║                                                          ║
║  异常场景构造：                                            ║
║    □ Session 过期 / Token 失效时的行为                     ║
║    □ 后端 API 响应超时（>5s）                               ║
║    □ 目标待办已被其他人删除（并发冲突）                      ║
║    □ 网络中断后的重连与恢复                                ║
║    □ 连续快速发送多个请求                                  ║
╚═══════════════════════════════════════════════════════════╝
```

---

### 第七章：版本管理 & 灰度发布

上线之后迭代不可避免，需要一套规范的版本策略：

#### 7.1 语义化版本规则

| 变更类型 | 版本升级 | 示例 |
|---------|---------|------|
| 新增可选参数（有默认值） | 小版本 | 1.0.0 → 1.1.0 |
| 修复 bug、优化文案 | 补丁版本 | 1.0.0 → 1.0.1 |
| 删除已有参数或工具 | 大版本 | 1.0.0 → 2.0.0 |
| 修改 required 字段列表 | 大版本 | 1.0.0 → 2.0.0 |

**向下兼容是底线**：新增参数必须有默认值，旧版 Skill 不感知也能正常工作。

#### 7.2 灰度发布流程

```
v1.0.0 (稳定版，100% 用户)
        │
        ▼ 开发 v1.1.0（新增 due_date 参数）
        │
  ┌─────┴─────┐
  │  内部测试  │  开发团队 + 产品验证（1~2 天）
  └─────┬─────┘
        │
        ▼
  ┌─────┴─────┐
  │  灰度 10%  │  白名单用户，观察成功率、错误率（3~5 天）
  └─────┬─────┘
        │
        ├─ 成功率 > 98%，无严重 Bug
        ▼
  ┌─────┴─────┐
  │  全量发布  │  所有用户切换到 v1.1.0
  └───────────┘
        │
        ▼ 保留回滚能力
  网关一键切回 v1.0.0（保证 Session 级别向下兼容）
```

---

### 总结

回顾一下整套体系的核心要点：

| 维度 | 一句话总结 |
|------|-----------|
| **Skill** | 给大模型的岗位说明书，按需注入、场景隔离 |
| **SKILL.md 写法** | 元数据定义触发条件，正文直接写业务 SOP（触发条件、场景流程、全局约束） |
| **MCP** | 让大模型能调用后端 API 的标准化手脚 |
| **Schema 原则** | description 写准、enum 锁死、required 引导反问、错误说人话 |
| **安全红线** | Token 不进 Prompt、高危操作双层防御、数据量硬限制 |
| **落地路径** | 先写 Skill 定 SOP → 再造 MCP 封装工具 → 伪代码验证 → 三阶段测试 → 灰度 |

最后用一句话概括两者的关系：**Skill 告诉大模型"做什么、按什么流程做"，MCP 帮大模型"真正做到"**。缺了 Skill，大模型不知道业务规则；缺了 MCP，大模型只能纸上谈兵。两者配合，AI 才能真正融入你的业务系统。

### 参考资料

- [Model Context Protocol 官方规范](https://modelcontextprotocol.io)
- [MCP SDK (TypeScript)](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP SDK (Python)](https://github.com/modelcontextprotocol/python-sdk)
- [Microsoft MCP Gateway — 企业级 MCP 网关方案](https://github.com/microsoft/mcp-gateway)
- [IBM Developer — MCP 企业架构模式](https://developer.ibm.com/articles/mcp-architecture-patterns-ai-systems/)
