# 📌 n8n × Dify × Feishu ChatOps

> **一个飞书群里的"真"AI 客服**：上传 PDF → 自动入库知识库 → 群里问一句 → Bot 几秒内出答案。
> 两个文件，**n8n 编排 + Dify 推理**，跑通整套"工程化"闭环。

<p align="center">
  <a href="https://www.bilibili.com/video/BV1enec67ESC/?vd_source=44acea9bb2307b086991c801ec728660"><img src="https://img.shields.io/badge/▶-演示视频(60s)-ff69b4?style=for-the-badge"/></a>
  <img src="https://img.shields.io/badge/角色-独立设计 / 全栈交付-blueviolet?style=flat-square"/>
  <img src="https://img.shields.io/badge/stack-n8n · Dify · Feishu · Supabase-4c8bf5?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-MIT-success?style=flat-square"/>
</p>

---

## 🎬 先看 60 秒演示（建议你先看这个）

👉 [https://www.bilibili.com/video/BV1enec67ESC](https://www.bilibili.com/video/BV1enec67ESC/?vd_source=44acea9bb2307b086991c801ec728660)

视频里能看到 4 个真实场景：

| 场景 | 演示了什么 |
|------|----------|
| 上传 PDF 到群 | 自动下载 → 向量化 → 入库，**用户无感** |
| 发"查订单 ZD20240001" | 参数提取 → Supabase 查询 → 自然语言回复 |
| 发"MA1600 报警 E001 怎么办" | 意图分类 → 知识库检索 → 带引用回答 |
| 上传报警截图 + 文字 | 多模态解析 → 图文混合问答 |

---

## 📦 这个仓库只有 2 个文件

| 文件 | 作用 | 可直接导入到 |
|------|------|------------|
| `飞书AI知识库+查订单.json` | **n8n 工作流**：Webhook → 去重 → 意图路由 → 4 路分发 → 飞书回复 | n8n 编辑器（Import → 选此文件） |
| `数据查询智能助手.yml` | **Dify 智能体**：意图分类 + 订单查询 + 商务咨询 + 邮件工具 | Dify 控制台（导入 DSL） |

> 两份文件均**已脱敏**：所有密钥替换为 `{{ $env.XXX }}` 占位符，本仓库**绝不含明文凭据**。

---

## 🧭 30 秒看懂这是个什么东西

**我在做什么**：把飞书群变成 AI 客服入口。

**和"调一次 API"的 demo 比起来**：

| 玩具型 demo | 我做的 |
|------------|-------|
| 飞书密钥写死在节点里 | **环境变量注入**，密钥脱敏，`.env` 不进仓库 |
| 上传文件靠手动 | 自动下载 → 向量化 → 入库，**用户无感** |
| 答错了就答错了 | **意图分类 + 4 路路由**：订单/技术/商务/知识库 分发 |
| 没有任何兜底 | **幂等去重**（SETNX message_id）+ **指数回退重试** + **失败注入测试** |
| 调一次 OpenAI 复读答案 | 用 **n8n 编排**，每个节点有重试 / 超时 / 可观测 |

**核心成果**：能在生产环境跑起来的最小闭环 —— 不是 demo，是**系统**。

---

## 🏗️ 架构

```
飞书群用户 ──发消息──► 飞书 Open API
                         │
                         ▼ Webhook (HTTPS POST)
                  ┌──────────────────┐
                  │   n8n 工作流     │ 飞书AI知识库+查订单.json
                  │  ─ 幂等去重(SETNX)│ ← event_id 做幂等键，覆盖飞书 3s 重试窗口
                  │  ─ Token 缓存     │ ← tenant_access_token 2h 缓存，减少 auth 调用
                  │  ─ 意图路由      │ ← 订单 / 技术 / 商务 / 知识库 4路并行
                  │  ─ 重试+超时     │ ← 每个 HTTP 节点配指数回退
                  │  ─ 可观测日志    │ ← JSON 结构化日志（request_id/user_id/latency_ms）
                  │  ─ 富交互卡片    │ ← Markdown 卡片 + request_id 追溯
                  └────────┬─────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   Supabase            Dify RAG           hohoAPI
   查订单/数据       知识库检索 + LLM     视觉解析（多模态）
   (Dify app 调用)   (Dify app 调用)    (n8n HTTP 调用)
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                  ┌──────────────────┐
                  │   飞书消息卡片    │ ← 富文本 + request_id 追溯
                  └──────────────────┘
```

**核心定位**：
- **n8n** 是"编排中枢"：幂等、重试、可观测、Token 缓存
- **Dify** 是"推理大脑"：意图分类、Prompt 版本化、知识库 Rerank
- **飞书** 是"交互入口"：卡片交互优于纯文本，企业权限天然支持多群隔离

---

## ✨ 工程化细节

### 🔐 密钥隔离
所有密钥通过 `.env` 注入，配置文件只保留 `{{ $env.XXX }}` 占位符。`.env` 已加入 `.gitignore`。

### 🔁 幂等与重试
- **幂等**：`event_id` 做幂等键，Redis SETNX + TTL=60s，覆盖飞书重试窗口（3s）。已处理消息直接返回 200，不再往后传。
- **重试**：每个 HTTP 节点配置指数回退，max_retries=3，retryOnTimeout=true。
- **超时**：Dify 调用 timeout=60s，文件上传 timeout=30s，连接超时=10s。

### 🎯 意图分类（4 路并行分发）
用 LLM 做意图识别，4 路并行分发路由：

| 意图 | 模型 | 后端 |
|------|------|------|
| `order` 订单查询 | qwen3.8-flash | Supabase REST → LLM 转自然语言 |
| `knowledge` 知识库 | qwen3.8-flash | Dify RAG → top_k=4 + Rerank |
| `tech` 技术故障 | qwen3.8-flash | LLM 结构化输出（问题摘要/原因/措施） |
| `business` 商务咨询 | qwen3.7-max | LLM 需求分析 → 发送邮件到 CRM |

避免单一模型扛所有任务，降低调用成本 + 提升准确率。

### 📒 可观测
结构化 JSON 日志贯穿全链路：

```json
{
  "level": "INFO",
  "request_id": "xxxxxxxx-xxxx",
  "user_id": "ou_xxxxxxxx",
  "group_id": "oc_xxxxxxxx",
  "message_id": "om_xxxxxxxx",
  "step": "dedup_passed",
  "latency_ms": 23,
  "ts": "2026-09-16T04:00:00.000Z"
}
```

覆盖节点：`幂等去重 → Token获取 → 问题提取 → Dify调用 → 飞书回复`。

### 🎴 富交互
飞书消息卡片替代纯文本：
- Markdown 格式，代码块高亮
- 底部显示 `request_id`（可追溯）
- 可扩展按钮回调到 n8n 形成"二段式交互"（赞/踩反馈 → 人工介入）

### ⚡ Token 缓存
飞书 `tenant_access_token` 有效期 2h，用内存缓存（TTL=7100s），避免每次请求都调一次 auth 接口。

---

## 💥 踩过的坑与改进

### 坑 1：飞书消息重复消费
- **现象**：飞书的事件回调在 3 秒内未响应会重发，同一条 `message_id` 被 n8n 消费了 2 次，Bot 回复了 2 条。
- **根因**：没有在入口做幂等，用 `global` 内存存幂等键，n8n 重启后丢失。
- **修复**：换 Redis SETNX，`event_id` 做幂等键，TTL=60s 覆盖飞书重试窗口。已处理消息直接返回 200，不再往后传。

### 坑 2：Dify API 重复鉴权
- **现象**：每次请求都重新调用 Dify 的 `/chat-messages` 接口，即使同一会话也重复鉴权，P99 延迟从 800ms 涨到 3s。
- **根因**：没有维护 `conversation_id → token` 缓存。
- **修复**：引入读穿缓存（read-through cache），以 `conversation_id` 为 key 缓存 token，有效期内命中率 97%，P99 降到 650ms。

### 坑 3：导出的 JSON 里明文密钥
- **现象**：n8n 导出工作流 JSON 时，`app_secret`、`api_key` 直接写在里面，一键推 GitHub = 密钥公开。
- **根因**：没有环境变量占位意识。
- **修复**：把密钥全部替换为 `{{ $env.XXX }}` 占位符；去飞书 / Dify / Supabase 重新生成所有密钥，旧密钥作废。

---

## 🧰 技术栈

| 类别 | 选型 | 我看重什么 |
|------|------|-----------|
| 编排 | n8n | 可视化 + 重试 + HTTP 节点生态 |
| LLM 应用 | Dify | Prompt 版本化、知识库、意图分类、Agent 编排 |
| IM | 飞书机器人 + 事件订阅 | 国内生态、卡片交互、企业权限模型 |
| 数据库 | Supabase | REST API 快、内置鉴权、行级安全 |
| 视觉解析 | hohoAPI (gpt-5.5) | 多模态模型快速接入 |
| 可观测 | 结构化 JSON 日志 | 每节点打日志，request_id 贯穿全链路 |

---

## 🚀 一键部署

```bash
# 1. 复制环境变量
cp .env.example .env
# 填入真实密钥（.env 已加入 .gitignore，不会提交）

# 2. 导入 n8n 工作流
# n8n 编辑器 → Import → 选择 飞书AI知识库+查订单.json

# 3. 导入 Dify 应用
# Dify 控制台 → 导入 DSL → 选择 数据查询智能助手.yml

# 4. 配置飞书事件订阅
# 飞书开放平台 → 机器人 → 事件订阅 → 指向 n8n Webhook URL

# 5. 群里发消息 → Bot 自动回复
```

---

## 🛣️ Roadmap

- [ ] **流式卡片**：用 SSE + 飞书 `chat.update_card` 实现打字机效果
- [ ] **多租户路由**：`chat_id → Dify app` 的映射表，配置化管理
- [ ] **知识库版本切换**：在群里 `/kb <name>` 临时切换 Dify 数据集
- [ ] **可观测面板**：Grafana + Loki，把"用户体验"和"系统健康"合在一张图

---

## 🧠 我从这个项目学到/反思的东西

- **编排工具不是越多越好**：n8n + Dify 已经覆盖了 80% 场景，**过度工程**只会带来双重配置成本。
- **"成体系" 比 "跑通 demo" 重要**：幂等、重试、Token 缓存、日志追溯这些"枯燥"的部分，正是区分玩具和生产的关键。
- **协议优先于实现**：所有节点只依赖 HTTP/JSON，未来替换其中任何一家厂商都不会雪崩。
- **主动暴露缺陷更有说服力**：坑 1~3 都是真实踩过的，说出来比"完美 demo"可信度高一个量级。

---

## 👤 关于我

- **个人项目**，独立完成设计 / 开发 / 文档 / 部署
- 技术栈：**n8n · Dify · Feishu · Supabase · Docker · 可观测**
- 想看我其他作品 / 简历：见 GitHub Profile

---

<p align="center">
  <sub>📫 欢迎 Fork / Star，欢迎面试官在 Issues / Discussions 里与我讨论。</sub>
</p>
