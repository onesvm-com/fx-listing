---
name: listing-worker-api-design
overview: 设计一套全新的 Cloudflare Worker 版 Listing 生成服务 API，替代现有 Flask 后端，并预留同时支持 Google Gemini 与 Cloudflare Workers AI 的能力。
todos:
  - id: design-routes
    content: 细化并确认 /v1 下各资源与动作型路由（:generate-title 等）的最终列表和命名
    status: pending
  - id: design-models
    content: 确定 D1 表结构与 TypeScript 接口字段，使之覆盖现有 JSON 字段且更规范
    status: pending
  - id: design-llm-provider
    content: 定义并固定 LLMProvider 接口以及 Gemini / Workers AI 的具体输入输出结构
    status: pending
  - id: design-error-format
    content: 敲定统一错误码与响应格式，覆盖参数错误、前置条件不满足和内部错误场景
    status: pending
  - id: design-security
    content: 决定是否增加 API Key 鉴权或其他访问控制机制，并设计其在 Worker 中的实现方式
    status: pending
isProject: false
---

## Listing Cloudflare Worker API 设计方案

### 一、总体架构与目标

- **目标**：用 JavaScript/TypeScript 在 Cloudflare Workers 上重写现有 `listing` 后端，提供更规范的 REST/JSON API，用于生成和管理亚马逊 Listing（标题、五点卖点、描述），并记录历史与关键词使用情况。
- **API 风格**：重新设计（不强求兼容 Flask 版本），遵循 RESTful + JSON，统一错误格式。
- **AI 后端抽象**：通过一个统一的 "LLMProvider" 抽象同时支持：
  - Google Gemini（通过 HTTP 请求外部 API）
  - Cloudflare Workers AI（通过 `fetch` 调用 `https://api.cloudflare.com/client/v4/accounts/.../ai/run/...` 或 `@cf/*` 绑定）
- **持久化存储建议**：
  - `D1`：存 listing 历史、关键词、敏感词、prompts（结构化查询和分页更方便）。

### 二、路由设计总览

以 `/v1` 为前缀，避免将来破坏性升级：

- **Prompts 管理**
  - `GET /v1/prompts`：获取全部生成模板
  - `PUT /v1/prompts`：一次性更新全部模板
- **关键词管理**
  - `GET /v1/keywords`：分页/搜索关键词
  - `POST /v1/keywords`：批量新增关键词
  - `PATCH /v1/keywords/{id}`：更新单个关键词
  - `DELETE /v1/keywords/{id}`：删除单个关键词
- **敏感词管理**
  - `GET /v1/sensitive-words`：分页列出敏感词
  - `POST /v1/sensitive-words`：批量新增敏感词
  - `DELETE /v1/sensitive-words/{id}`：删除单个敏感词
  - `POST /v1/sensitive-words:check`：检查文本中是否含敏感词
  - `POST /v1/sensitive-words:filter`：对文本做敏感词替换
- **Listing 生成（核心）**
  - `POST /v1/listings`：创建一条新的 listing 记录（只含元信息）
  - `POST /v1/listings/{id}:generate-title`：为该 listing 生成标题
  - `POST /v1/listings/{id}:generate-bullets`：为该 listing 生成五点卖点
  - `POST /v1/listings/{id}:generate-description`：为该 listing 生成描述
- **Listing 历史与分析**
  - `GET /v1/listings`：分页查询历史记录，支持按 pid/type/时间过滤
  - `GET /v1/listings/{id}`：获取单条记录详细信息
  - `DELETE /v1/listings/{id}`：删除单条记录
  - `POST /v1/listings:batch-delete`：批量删除
  - `GET /v1/listings/{id}/analysis`：返回高亮和关键词使用统计
- **系统与调试**
  - `GET /v1/health`：运行状态和依赖检查

### 三、统一请求与响应格式

- **成功响应统一格式**：
  - 顶层：`{ "success": true, "data": T }`
- **错误响应统一格式**：
  - 顶层：`{ "success": false, "error": { "code": string, "message": string, "details"?: any } }`
- **常见错误码**：
  - `INVALID_ARGUMENT`：参数校验不通过
  - `NOT_FOUND`：资源不存在
  - `FAILED_PRECONDITION`：如在未生成标题前调用生成 bullets
  - `INTERNAL`：未预期错误

### 四、核心数据模型（TypeScript 接口草案）

- **PromptConfig**：
  - `titlePrompt: string`
  - `bulletsPrompt: string`
  - `descriptionPrompt: string`
  - `updatedAt: string`
- **Keyword**：
  - `id: string`（D1 主键或 KV key）
  - `term: string`
  - `frequency: number`
  - `createdAt: string`
  - `updatedAt: string`
- **SensitiveWord**：
  - `id: string`
  - `word: string`
  - `createdAt: string`
- **ListingStatus**：`"EMPTY" | "TITLE_ONLY" | "TITLE_BULLETS" | "COMPLETED"`
- **ListingRecord**：
  - `id: string`
  - `pid?: string`
  - `type?: string`
  - `title?: string`
  - `titleKeywords?: string[]`
  - `bullets?: string[]`
  - `bulletsKeywords?: string[]`
  - `description?: string`
  - `descriptionKeywords?: string[]`
  - `usedKeywords?: string[]`
  - `status: ListingStatus`
  - `model: string`（使用的模型名称，便于调试）
  - `createdAt: string`
  - `updatedAt: string`
- **ListingAnalysis**：
  - `title: { highlighted: string; keywords: string[]; frequencies: FrequencyItem[] }`
  - `bullets: { highlighted: string[]; keywords: string[]; frequencies: FrequencyItem[] }`
  - `description: { highlighted: string; keywords: string[]; frequencies: FrequencyItem[] }`
  - `where FrequencyItem = { phrase: string; totalCount: number; orderedCount: number; unorderedCount: number }`

### 五、AI 生成接口的详细设计

#### 1. 创建 Listing

- **路径**：`POST /v1/listings`
- **请求体**：
  - `{ "pid"?: string, "type"?: string }`
- **行为**：
  - 在 D1 中插入一条记录，状态设为 `"EMPTY"`。
- **响应**：`{ success: true, data: ListingRecord }`

#### 2. 生成标题

- **路径**：`POST /v1/listings/{id}:generate-title`
- **请求体**：
  - `{ "count"?: number, "aiProvider"?: "gemini" | "workers_ai", "model"?: string }`
- **行为**：
  - 读取全量关键词、敏感词和 PromptConfig 中的 titlePrompt。
  - 调用 `LLMProvider.generateTitle(request)`，内部根据 `aiProvider` 向 Gemini 或 Workers AI 发送请求。
  - 结果写回 listing：`title`, `titleKeywords`, `usedKeywords` 合并，`status` 至少为 `"TITLE_ONLY"`。
- **响应**：
  - `data` 中包含：
    - `listing: ListingRecord`
    - `variants: string[]`（所有备选标题）

#### 3. 生成五点卖点

- **路径**：`POST /v1/listings/{id}:generate-bullets`
- **前置条件**：该 listing 已有 `title`，否则返回 `FAILED_PRECONDITION`。
- **请求体**：与 `generate-title` 类似，增加可选参数：
  - `{ "count"?: number, "aiProvider"?: ..., "model"?: string }`
- **行为**：
  - 读取 listing 的 `title` 与当前关键词库（排除已用关键词），敏感词和 bulletsPrompt。
  - 调用 `LLMProvider.generateBullets`，得到一组 5 条左右的 bullets。
  - 更新 listing 中 `bullets`, `bulletsKeywords`, `usedKeywords`, `status` 至 `"TITLE_BULLETS"` 或更高。

#### 4. 生成描述

- **路径**：`POST /v1/listings/{id}:generate-description`
- **前置条件**：该 listing 已有 `title` 和 `bullets`，否则 `FAILED_PRECONDITION`。
- **请求体**：`{ "aiProvider"?: ..., "model"?: string }`
- **行为**：
  - 基于 title + bullets + 剩余未用关键词 + descriptionPrompt，调用 `LLMProvider.generateDescription`。
  - 使用敏感词过滤器对返回结果做替换（按配置的 `replaceWith` 字符）。
  - 更新 listing 中 `description`, `descriptionKeywords`, `usedKeywords`, `status="COMPLETED"`。

### 六、AI Provider 抽象（Gemini 与 Workers AI）

- 定义接口 `LLMProvider`：
  - `generateTitle(input: TitleInput): Promise<TitleResult>`
  - `generateBullets(input: BulletsInput): Promise<BulletsResult>`
  - `generateDescription(input: DescriptionInput): Promise<DescriptionResult>`
- 实现两个具体类：
  - `GeminiProvider`：
    - 从环境变量（如 `GEMINI_API_KEY`）读取密钥。
    - 使用 HTTP fetch 调用 Gemini REST API（如 `https://generativelanguage.googleapis.com/v1/models/...:generateContent`）。
  - `WorkersAiProvider`：
    - 通过 Worker 的环境绑定（如 `env.AI` 或 `env.CF_ACCOUNT_ID` + `CF_API_TOKEN`）调用 Cloudflare AI。
    - 将统一输入转换为模型对应的 prompt / messages 格式。
- Worker 入口中根据配置决定默认 provider，也允许每次请求通过 `aiProvider` 字段覆写。

### 七、存储设计（以 D1 为例）

- **表 `keywords`**：
  - `id TEXT PRIMARY KEY`
  - `term TEXT NOT NULL UNIQUE`
  - `frequency INTEGER NOT NULL`
  - `created_at TEXT NOT NULL`
  - `updated_at TEXT NOT NULL`
- **表 `sensitive_words`**：
  - `id TEXT PRIMARY KEY`
  - `word TEXT NOT NULL UNIQUE`
  - `created_at TEXT NOT NULL`
- **表 `prompts`**（单行表或 key-value 表）：
  - `id TEXT PRIMARY KEY`（可固定为 `"default"`）
  - `title_prompt TEXT`
  - `bullets_prompt TEXT`
  - `description_prompt TEXT`
  - `updated_at TEXT`
- **表 `listings`**：
  - `id TEXT PRIMARY KEY`
  - `pid TEXT`
  - `type TEXT`
  - `title TEXT`
  - `title_keywords TEXT`（JSON 数组字符串）
  - `bullets TEXT`（JSON 数组字符串）
  - `bullets_keywords TEXT`
  - `description TEXT`
  - `description_keywords TEXT`
  - `used_keywords TEXT`
  - `status TEXT`
  - `model TEXT`
  - `created_at TEXT`
  - `updated_at TEXT`

### 八、Cloudflare Worker 入口结构建议

- 单文件入口例如 `src/worker.ts`：
  - 导出 `fetch(request, env, ctx)`。
  - 在内部根据 `new URL(request.url).pathname` 和 `request.method` 做路由分发（或使用轻量路由库）。
  - 公共中间件：解析 JSON、捕获异常并包装为统一错误响应、记录请求日志。
  - 路由处理函数中调用：
    - 存储层封装（D1/KV）
    - AI provider 抽象
    - 文本分析和敏感词过滤工具。

### 九、后续可扩展点

- 增加简单的 API Key 鉴权（如在 `Authorization` 头携带自定义 token）。
- 增加多语言支持（支持生成 JP/DE 等 Listing）。
- 为前端提供参数化的“预览分析”接口，如只分析给定文本而不落库。

