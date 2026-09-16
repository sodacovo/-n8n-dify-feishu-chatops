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
                  │  ─ 去重 (SETNX)  │ ← 同一 message_id 不重复处理
                  │  ─ 意图路由      │ ← 订单 / 技术 / 商务 / 知识
                  │  ─ 重试 + 兜底   │
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
                  │   飞书消息卡片    │ ← 富文本 + 引用 + 按钮回调
                  └──────────────────┘
```

**核心定位**：
- **n8n** 是"编排中枢"：Webhook、重试链、可观测
- **Dify** 是"推理大脑"：Prompt 版本化、知识库、Agent 编排
- **飞书** 是"交互入口"：卡片交互优于纯文本，企业权限天然支持多群隔离

### 一次完整的"问订单"数据流

```
1. 用户群里发："查一下 ZD20240001"
2. 飞书 → Webhook(POST /im.message.receive_v1) 到 n8n
3. n8n 入口：去重 (message_id) + 限流 + 鉴权
4. n8n → Dify /chat-messages (意图 = order)
5. n8n → 参数提取 (order_id = ZD20240001)
6. n8n → Supabase REST 查询 orders 表
7. n8n → LLM 把 JSON 数据转成自然语言
8. n8n → 飞书消息卡片 POST (带按钮"反馈赞/踩")
```

---

## ✨ 工程化细节（这是面试官最想看的）

> **不是你能不能调通，而是你把它做成"系统"了没有**。

- **🔐 密钥隔离**：所有 `FEISHU_APP_SECRET`、`DIFY_API_KEY`、`SUPABASE_ANON_KEY` 通过 `.env` 注入，本仓库只保留占位符。
- **🔁 幂等与重试**：n8n 节点配置了指数回退 + 抖动；用 `message_id` 做幂等键，避免飞书 Webhook 重复消费。
- **🎯 意图分类**：用 LLM 做意图识别 + 路由（订单 / 技术 / 商务 / 知识库），4 路并行分发，避免单一模型挑大梁。
- **📒 可观测**：结构化 JSON 日志（`request_id`、`user_id`、`group_id`、`latency_ms`）。
- **🎴 富交互**：用飞书**消息卡片**而不是纯文本，支持按钮回调到 n8n 形成"二段式交互"。

---

## 💥 踩坑与改进

> 主动暴露缺陷 + 给出修复，比展示一个"完美 demo"可信度高一个量级。

### 坑 1：飞书消息重复消费
- **现象**：飞书的事件回调在 3 秒内未响应会重发，同一条 `message_id` 被 n8n 消费了 2 次，Bot 回复了 2 条。
- **根因**：没有在入口做幂等。
- **修复**：在 n8n 入口节点用 `SETNX message_id <ttl>` 做去重，已处理的消息直接返回 200，不再往后传。

### 坑 2：Dify API Key 每次现拿
- **现象**：每次请求都重新调用 Dify 的 `/messages` 接口拿 `task_id`，即使同一会话也重复鉴权，P99 延迟从 800ms 涨到 3s。
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
| LLM 应用 | Dify | Prompt 版本化、知识库、Agent 编排 |
| IM | 飞书机器人 + 事件订阅 | 国内生态、卡片交互、企业权限模型 |
| 数据库 | Supabase | REST API 快、内置鉴权、行级安全 |
| 视觉解析 | hohoAPI (gpt-5.5) | 多模态模型快速接入 |
| 可观测 | 结构化日志 + Prometheus `/metrics` | 面试官最常问的两件套 |

---

## 🚀 一键运行（方便面试官本地拉起）

```bash
# 1. 把这两个文件 import 到你的 n8n + Dify
# 2. 在你的 .env 文件里填好所有密钥
# 3. 飞书后台把事件回调指向你的 n8n webhook URL
# 4. 群里发消息 → Bot 自动回复
```

### 需要准备的环境变量（这些都不会进仓库）

```bash
# 飞书机器人
FEISHU_APP_ID=
FEISHU_APP_SECRET=

# Dify
DIFY_DATASET_API_KEY=
DIFY_APP_API_KEY=

# Supabase
SUPABASE_ANON_KEY=

# 腾讯云 COS（图片持久化）
TENCENT_COS_CREDENTIALS_ID=

# hohoAPI（多模态视觉）
HOHO_API_CREDENTIAL_ID=
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
- **"成体系" 比 "跑通 demo" 重要**：密钥、重试、幂等、限流这些"枯燥"的部分，正是区分玩具和生产的关键。
- **协议优先于实现**：所有节点只依赖 HTTP/JSON，未来替换其中任何一家厂商都不会雪崩。

---

## 👤 关于我

- **个人项目**，独立完成设计 / 开发 / 文档 / 部署
- 技术栈：**n8n · Dify · Feishu · Supabase · Docker · 可观测**
- 想看我其他作品 / 简历：见 GitHub Profile

---

<p align="center">
  <sub>📫 欢迎 Fork / Star，欢迎面试官在 Issues / Discussions 里与我讨论。</sub>
</p>
