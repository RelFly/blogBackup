---
title: AI Skill 开发实战指南——让大模型真正理解并操作你的业务系统
date: 2026-05-06 16:00:00
tags:
- ai skill
- AI Agent
- MCP
categories:
- AI
---

### 前言

如果你是一名研发工程师，想让 AI Agent（如 Claude、GPT 等）能够**真正理解并操作你的业务系统**，你需要了解 **Skill（技能）** 与 **MCP（Model Context Protocol，模型上下文协议）** 这两个核心概念。本文基于业界实践与源码级分析，梳理一套企业级 Skill + MCP 的开发范式。

<!-- more -->

### 一、什么是 Skill？——给 AI 的岗位说明书

如果把大模型比作一个极其聪明但刚入职的全科大学生，那么 **Skill** 就是你发给他的特定岗位操作手册。

#### 为什么需要 Skill？

| 问题 | Skill 的解法 |
|------|-------------|
| 大模型上下文窗口有限且昂贵 | **按需动态加载**对应业务逻辑，而非全量塞入 Prompt |
| 大模型不懂你们公司的规则 | 通过 SOP 强制约束行为，防止幻觉 |
| 不同场景需要不同的能力集 | 每个 Skill 职责单一，互不干扰 |

#### Skill 目录结构

```
my-business-skill/
├── SKILL.md          # 【必填】核心大脑：触发词 + 业务 SOP
├── mcp.json          # 【按需】MCP 服务连接配置
├── references/       # 【选填】知识库：错误码映射、业务规则等
│   └── error_codes.md
├── scripts/          # 【选填】本地脚本
└── assets/           # 【选填】静态物料：模板、图片
```

#### SKILL.md 核心结构

```yaml
---
name: my-assistant           # 全局唯一标识
version: 1.0.0               # 版本号，用于灰度控制
author: "Dev-Team"
description: "当用户要求预约会议、查询列表或取消时触发"
trigger_keywords: ["预约会议", "我的会议", "取消会议"]
---
```

> **加载原理**：用户输入 → Agent 扫描所有 Skill 头部 → 关键词/语义命中 → 将 `SKILL.md` **完整注入**大模型上下文 → 任务结束后释放。

**触发优先级**：
1. **关键词匹配**（最高）— `trigger_keywords` 精准全匹配
2. **语义匹配**（兜底）— 基于 `description` 计算相似度

---

### 二、Skill + MCP：大脑与手脚的组合架构

在企业级开发中，AI 需要与后端微服务交互，此时引入 **MCP（Model Context Protocol）**：

```
用户输入 → [路由层] 命中 Skill
                ↓
        SKILL.md 注入上下文（大脑）
                ↓
        大模型判断意图 → 决定调用哪个工具
                ↓
        MCP Client → MCP Server（手脚）
                ↓
        后端微服务 / 第三方 API
```

**分工明确**：

| 层 | 角色 | 类比 |
|---|------|------|
| **SKILL.md** | 业务 SOP、意图识别、交互逻辑 | 指挥官 |
| **MCP Server** | 封装 API 为标准化工具 | 执行者 |
| **mcp.json** | 配置连接地址和传输方式 | 电话簿 |

#### mcp.json 的两种传输方式

**方式一：stdio（本地子进程）**

```json
{
  "mcpServers": {
    "my-tool": {
      "command": "/usr/local/bin/my-server",
      "args": ["--debug"],
      "env": { "APP_SECRET": "xxx" }
    }
  }
}
```

适合本地环境，MCP 客户端将 Server 作为子进程拉起，通过 stdin/stdout 通信。

**方式二：SSE（远程 HTTP）**

```json
{
  "mcpServers": {
    "my-tool": {
      "url": "https://api.yourcompany.com/mcp/sse",
      "headers": { "Authorization": "Bearer TOKEN" }
    }
  }
}
```

适合企业级生产环境，支持多实例负载均衡、集中鉴权。

---

### 三、SKILL.md 编写实战

以下是包含 CRUD 四个场景的 Skill SOP 示例：

```markdown
# 业务管理助手 (SOP)

## 核心工作流

### 场景 A：创建 (Create)
- [条件]：用户要求新建/预约
- [动作]：检查【主题】和【时间】是否齐备，缺失则反问。
        调用 MCP 工具 `create_resource`。
- [结果]：输出成功链接。

### 场景 B：查询 (Query)
- [条件]：用户询问"今天有什么"
- [动作]：调用 MCP 工具 `get_list`。
- [结果]：将 JSON 渲染为 Markdown 表格返回。

### 场景 C：修改 (Modify)
- [条件]：用户要求变更
- [动作]：调用 `get_list` 定位目标 ID → 询问新参数 → 调用 `update_resource`。

### 场景 D：删除 (Cancel) — ⚠️ 高危操作
- [条件]：用户意图为删除
- [动作]：必须向用户复述关键信息并**二次确认**！
        获得明确同意后才调用 `delete_resource`。
- [结果]：告知已删除。
```

#### 编写红线清单

| 规则 | 说明 | 反模式 |
|------|------|--------|
| **不要重复写出入参** | MCP 握手时已暴露 InputSchema | 在 references 里再抄一遍 JSON Schema |
| **references 只放业务规则** | 错误码映射、权限边界、跨系统联动 | 放通用技术文档 |
| **高危操作强制确认** | 删除/修改前必须二次确认 | 直接调用不问用户 |
| **时间参数用 ISO 8601** | 如 `2026-04-01T14:30:00+08:00` | 用 Unix 时间戳（模型算不准） |

---

### 四、MCP 服务端开发规范

#### Schema 编写规则

大模型完全依赖你定义的 Schema 来理解如何调用工具：

```json
{
  "type": "object",
  "properties": {
    "subject": {
      "type": "string",
      "description": "主题",
      "minLength": 1,
      "maxLength": 100
    },
    "start_time": {
      "type": "string",
      "description": "开始时间，ISO 8601 格式",
      "format": "date-time"
    },
    "status": {
      "type": "string",
      "enum": ["active", "cancelled", "completed"]
    },
    "duration": {
      "type": "number",
      "default": 60,
      "minimum": 15,
      "maximum": 480
    }
  },
  "required": ["subject", "start_time"]
}
```

**核心原则**：

| 要点 | 做法 | 原因 |
|------|------|------|
| Description 写精准 | 不要写"时间"，要写"ISO 8601 格式的开始时间" | 模型会自由发挥 |
| 用 Enum 限制选项 | 固定枚举值防止模型编造非法值 | 减少幻觉 |
| 必填项用 required | 缺失时模型自动反问用户 | 避免空值调用失败 |
| 返回语义化字符串 | `"status": "cancelled"` > `"status": 3` | 模型无需查表理解 |

#### Content Block 返回格式

MCP 协议规定的标准返回类型：

```json
// 文本
{ "type": "text", "text": "创建成功！ID: 888-999" }

// 图片
{ "type": "image", "data": "base64...", "mimeType": "image/png" }

// 错误（isError: true 让模型知道需要纠错）
{ "isError": true, "content": [{ "type": "text", "text": "时长超限，请选择 15-480 分钟" }] }
```

> **重要**：错误返回时**不要只返回错误码**，要返回自然语言修复建议。这样大模型能自动引导用户修正输入。

---

### 五、安全与最佳实践

#### 5.1 鉴权隔离——Token 绝不进 Prompt

```
❌ 反模式：InputSchema 中定义 user_token 参数
   → 模型可能向用户索要 Token → 泄露到日志

✅ 正确方式：
   用户请求 → 网关拦截 → 从会话提取凭证
             → 动态注入 HTTP Header / Context
             → MCP Server 从 ctx 提取 → 对模型全程隐身
```

#### 5.2 高危操作双层防御

```
第一道防线（Skill 层）
  SKILL.md 写死："删除前必须复述信息并获得用户确认"

第二道防线（MCP 层）
  Handler 检查上下文中是否有 human_confirm 标识
  无标识直接拒绝 → 即使模型跳过确认也不会执行
```

#### 5.3 数据量控制——防失焦

- **裁剪字段**：只返回当前场景需要的字段，别返回 50 个无用字段
- **分页限制**：超过 5 条时提示"展示前 N 条，使用 offset 查询更多"
- **原因**：撑爆 Token 窗口 + 触发"中间注意力丢失"效应

#### 5.4 企业级高阶架构

| 模式 | 解决的问题 | 做法 |
|------|-----------|------|
| **编排层 (Facade)** | 多步原子工具导致模型迷失 | 封装成宏指令，一次调用完成复杂流程 |
| **API Gateway** | 静态配置无法处理多租户鉴权 | 动态注入凭证、统一审计、限流熔断 |
| **Hub & Spoke 路由** | Context Window 爆栈 | 总网关分发 → 按需加载子域 Skill |

---

### 六、验证闭环

开发完成后必须经过三阶段验证：

```
阶段 1：MCP 工具独立测试（手脚）
  → 用 CLI/Postman 直接发 JSON-RPC 请求
  → 验证：参数校验、鉴权链路、错误信息自然语言化

阶段 2：Skill 流程模拟（大脑）
  → 把 SKILL.md 粘贴到大模型控制台 System Prompt
  → 扮演刁难用户，验证 SOP 是否被严格执行

阶段 3：端到端全链路整合
  → 在实际客户端加载 Skill，打通完整链路
  → 构造异常场景（Token 过期、权限降级、数据不存在）
```

---

### 七、版本管理与灰度发布

| 规范 | 要求 |
|------|------|
| **语义化版本** | 新增可选参数 → 小版本；删除必填字段 → 大版本 |
| **向下兼容** | 新增参数必须有默认值，老 Skill 不感知也能工作 |
| **灰度策略** | 新版先下发给 10% 白名单用户，观察成功率后再全量 |
| **快速回滚** | 网关一键切回旧版 Skill/MCP，保证 Session 向下兼容 |

---

### 八、总结

| 维度 | 核心要点 |
|------|---------|
| **Skill 定位** | 大模型的业务 SOP 操作手册，按需注入上下文 |
| **MCP 定位** | 将后端 API 封装为大模型可调用的标准化工具 |
| **编写原则** | SOP 清晰、高危确认、不用重复写 Schema、时间用 ISO 8601 |
| **安全红线** | Token 不进 Prompt、双层防御、数据量控制 |
| **落地路径** | 先写 Skill（大脑）→ 再造 MCP（手脚）→ 三阶段验证 → 灰度发布 |

> 一句话：**Skill 告诉大模型"做什么"，MCP 帮大模型"做到"。两者配合，才能让 AI 真正融入你的业务系统。**

### 参考资料

- [Model Context Protocol 官方文档](https://modelcontextprotocol.io)
- [Zoom Skills 开源项目](https://github.com/zoom/skills)
- [Microsoft MCP Gateway](https://github.com/microsoft/mcp-gateway)
- [IBM Developer - MCP Architecture Patterns](https://developer.ibm.com/articles/mcp-architecture-patterns-ai-systems/)
