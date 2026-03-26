# Secure Web API 项目文档

## 1. 项目概述

Secure Web API 是一个安全的网页搜索与内容抓取服务，设计目标是为大语言模型（LLM）提供干净、结构化的网页内容。服务提供两个核心接口：搜索和抓取，中间通过 Redis 会话白名单连接，确保只有经过安全过滤的 URL 才能被抓取。

### 核心特性

- 多层安全过滤：SSRF 防护、PII 检测、域名黑白名单、查询关键词拦截
- 会话级 URL 白名单：搜索结果写入 Redis，抓取时校验，防止任意 URL 访问
- 内容提取输出 Markdown：保留标题、链接、列表等结构，适合 LLM 消费
- 可插拔搜索引擎：接口化设计，支持 Mock/Google/自定义引擎切换
- 生产级基础设施：JWT 鉴权、请求限流、安全头、优雅关闭、健康检查

### 技术指标

| 指标 | 数值 |
|---|---|
| 源文件 | 21 个 |
| 测试文件 | 14 个 |
| 测试用例 | 104 个 |
| TypeScript 严格模式 | 开启 |
| Node.js 版本 | 22+ |
| 运行时模块系统 | ESM |

---

## 2. 技术栈

### 2.1 运行时与构建

| 技术 | 版本 | 说明 |
|---|---|---|
| Node.js | 22 | 运行时，使用原生 fetch（无需 axios） |
| TypeScript | 5.7+ | 严格模式，ES2022 target，Node16 模块解析 |
| ESM | — | 全项目使用 ES Module（`"type": "module"`） |

### 2.2 生产依赖

#### Web 框架

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **express** | 4.21+ | HTTP 服务框架 | Node.js 生态最成熟的 Web 框架，中间件生态丰富，社区活跃，文档完善。对比 Fastify：Express 学习成本更低，中间件兼容性更好；对比 Koa：Express 内置路由，不需要额外引入 |

#### 安全与鉴权

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **helmet** | 8.1+ | HTTP 安全响应头 | 一行代码设置 15+ 个安全头（X-Content-Type-Options、X-Frame-Options 等），Express 官方推荐。对比手动设置：helmet 覆盖全面且持续跟踪新的安全标准 |
| **cors** | 2.8+ | 跨域资源共享控制 | Express 官方 CORS 中间件，支持按域名/方法/头部精细控制。对比手动设置 Access-Control 头：cors 库处理了预检请求（OPTIONS）、凭证模式等边界情况 |
| **jsonwebtoken** | 9.0+ | JWT 签发与验证 | Node.js JWT 实现的事实标准（npm 周下载量 1800 万+），支持 HS256/RS256 等多种算法。对比 jose：jsonwebtoken API 更简洁直观，适合服务端验证场景；jose 更适合需要 JWE 加密的场景 |
| **express-rate-limit** | 8.3+ | 请求限流 | Express 官方推荐的限流中间件，支持滑动窗口、按 IP 限流、自定义存储后端。对比手动实现：覆盖了 X-RateLimit 标准头、多种时间窗口算法、内存/Redis 存储切换 |

#### 数据与存储

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **ioredis** | 5.4+ | Redis 客户端 | Node.js 最成熟的 Redis 客户端，支持 Cluster、Sentinel、Pipeline、Lua 脚本。对比 node-redis（官方客户端）：ioredis 的 Pipeline API 更符合直觉，TypeScript 类型支持更好，重连策略更灵活 |

#### 请求校验与配置

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **zod** | 3.24+ | 运行时 Schema 校验 | TypeScript-first 的校验库，Schema 定义即类型推导，API 链式调用直观。对比 joi：Zod 是 TypeScript 原生设计，类型推导零成本；joi 需要额外的 @types 且类型推导不完整。对比 ajv（JSON Schema）：Zod 的 DSL 更简洁，不需要写冗长的 JSON Schema 对象 |
| **js-yaml** | 4.1+ | YAML 配置文件解析 | 轻量的 YAML 解析器。对比 JSON 配置：YAML 支持注释、多行字符串，配置文件可读性更好 |

#### 内容提取

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **@mozilla/readability** | 0.5+ | 网页正文提取 | Firefox 阅读模式使用的同一算法，提取精度高。对比自行解析：Readability 经过 Firefox 数十亿页面的验证，处理各种复杂 HTML 结构的鲁棒性远超手写规则 |
| **cheerio** | 1.0+ | HTML DOM 操作 | 服务端 jQuery 语法的 HTML 解析器，作为 Readability 失败时的 fallback。对比 jsdom：cheerio 不执行 JS、不加载外部资源，性能快 10x+，适合纯 DOM 操作；jsdom 仅在 Readability 需要完整 DOM 时使用 |
| **jsdom** | 29.0+ | DOM 环境模拟 | 为 Readability 提供 `window.document` 环境。Readability 需要完整的 DOM API，cheerio 不提供，所以必须用 jsdom |
| **turndown** | 7.2+ | HTML 转 Markdown | 成熟的 HTML-to-Markdown 转换库，支持自定义规则。对比自行实现：turndown 正确处理了嵌套列表、代码块、表格、链接等复杂结构，手写覆盖这些边界情况工作量巨大 |

#### 日志

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **pino** | 9.6+ | 结构化日志 | Node.js 最快的日志库（比 winston 快 5x），JSON 格式输出适合日志采集系统（ELK、Datadog）。对比 winston：pino 零依赖、性能更优、JSON 原生；winston 功能更多但性能开销大 |

### 2.3 开发依赖

| 库 | 版本 | 用途 | 选型理由 |
|---|---|---|---|
| **vitest** | 4.1+ | 测试框架 | 与 Vite 生态集成，原生支持 ESM 和 TypeScript，无需 babel 配置。对比 jest：jest 对 ESM 支持不完善（需要实验性 flag），TypeScript 需要 ts-jest 转译；vitest 零配置即可运行 |
| **supertest** | 7.2+ | HTTP 接口测试 | 在不启动真实服务器的情况下测试 Express 路由，API 简洁。行业标准选择 |
| **tsx** | 4.19+ | 开发运行时 | 直接运行 TypeScript 文件，无需先编译。对比 ts-node：tsx 基于 esbuild，启动速度快 10x+ |
| **@vitest/coverage-v8** | 4.1+ | 代码覆盖率 | V8 原生覆盖率，无需 istanbul 插桩，精度高 |

### 2.4 依赖关系选型决策树

```
需要 Web 框架？
  → Express（成熟稳定，中间件生态）

需要校验请求参数？
  → TypeScript 项目 → Zod（类型推导最好）
  → JavaScript 项目 → joi 或 ajv

需要提取网页正文？
  → 有结构化文章 → Readability（Firefox 级精度）
  → Readability 失败 → Cheerio（轻量 DOM 操作）
  → 需要完整 DOM API → jsdom（仅给 Readability 用）

需要输出格式？
  → 给 LLM 用 → Markdown（turndown 转换）
  → 给前端展示 → 保留 HTML

需要 Redis 客户端？
  → 需要 Pipeline/Cluster → ioredis
  → 简单 get/set → node-redis 也行

需要日志？
  → 高性能 + JSON → pino
  → 功能丰富 + 多 transport → winston

需要测试？
  → ESM + TypeScript → vitest（零配置）
  → CommonJS 项目 → jest 也行
```

---

## 3. 项目结构

```
secure-web-api/
├── config/
│   ├── server.yaml              # 生产配置
│   └── server.dev.yaml          # 开发配置
├── src/
│   ├── index.ts                 # 入口：组装模块、启动 Express
│   ├── config/
│   │   ├── loader.ts            # YAML + 环境变量 + 校验
│   │   └── __tests__/
│   ├── shared/
│   │   ├── types.ts             # ServerConfig、SearchResult
│   │   ├── redis.ts             # Redis 单例连接
│   │   └── logger.ts            # Pino 日志
│   ├── search/                  # 搜索引擎层（策略模式）
│   │   ├── types.ts             # SearchEngine 接口
│   │   ├── mock.ts              # Mock 引擎
│   │   ├── google.ts            # Google 引擎（含重试）
│   │   ├── cached-google.ts     # 缓存装饰器
│   │   └── __tests__/
│   ├── filter/                  # 安全过滤层
│   │   ├── types.ts             # QueryFilter、UrlFilter 接口
│   │   ├── query-guard.ts       # 查询守卫
│   │   ├── url-filter.ts        # URL 过滤器
│   │   ├── composite.ts         # 组合过滤器
│   │   └── __tests__/
│   ├── pool/                    # 会话白名单
│   │   ├── url-pool.ts          # Redis Set + TTL
│   │   ├── url-normalize.ts     # URL 归一化
│   │   └── __tests__/
│   ├── fetch/                   # HTTP 抓取 + 内容提取
│   │   ├── fetcher.ts           # 抓取：重定向、超时、大小限制
│   │   ├── extractor.ts         # 提取：Readability → Cheerio → Turndown
│   │   └── __tests__/
│   ├── routes/                  # 路由处理
│   │   ├── search.ts            # POST /api/search
│   │   ├── fetch.ts             # POST /api/fetch
│   │   └── __tests__/
│   └── middleware/              # Express 中间件
│       ├── auth.ts              # JWT 鉴权
│       ├── rate-limit.ts        # 请求限流
│       └── __tests__/
├── openapi.yaml                 # OpenAPI 3.0 规范
├── Dockerfile                   # 多阶段构建
├── docker-compose.yaml          # 服务编排
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

---

## 4. 架构设计

### 4.1 请求处理管线

```
请求 → helmet → cors → json解析 → 全局限流 → 搜索限流 → JWT鉴权 → 路由 → 错误兜底
```

中间件注册顺序决定了执行顺序：
1. **helmet** — 添加安全响应头（不拦截请求）
2. **cors** — 跨域检查（拦截不允许的源）
3. **express.json()** — 解析 JSON body（最大 100KB）
4. **全局限流** — 60 次/分钟（超限返回 429）
5. **搜索限流** — /api/search 额外 10 次/分钟
6. **健康检查** — /health 在鉴权前注册，不需要 token
7. **JWT 鉴权** — /api/* 路径需要 Bearer token
8. **业务路由** — 搜索和抓取
9. **错误兜底** — 未捕获异常返回 JSON（不是 HTML）

### 4.2 搜索流程（POST /api/search）

```
请求 { query, maxResults?, sessionId? }
  │
  ├─ Step 1: Zod 参数校验
  │   query: string, 1-1000 字符
  │   maxResults: int, 1-20, 默认 5
  │   sessionId: UUID 格式, 可选（自动生成）
  │
  ├─ Step 2: 查询内容过滤（QueryGuard）
  │   ├─ 关键词黑名单：case-insensitive 子串匹配
  │   │   例："confidential" 拦截 "show me confidential docs"
  │   └─ PII 正则检测：
  │       ├─ 18位身份证号：\b\d{17}[\dXx]\b
  │       ├─ 11位手机号：\b1[3-9]\d{9}\b
  │       ├─ 邮箱：\b[\w.+-]{1,100}@[\w-]{1,50}\.[\w.]{1,20}\b
  │       └─ API Key：\b(AKIA|sk-|ghp_|gho_)[\w+/=-]{1,200}\b
  │
  ├─ Step 3: 调用搜索引擎
  │   SearchEngine.search(query, maxResults)
  │   ├─ MockSearchEngine: 5条硬编码结果
  │   ├─ GoogleSearchEngine: Google API, 分页(max 30), 重试(429/5xx)
  │   └─ CachedSearchEngine: Redis缓存装饰器, SHA256 key, 可配TTL
  │
  ├─ Step 4: URL 过滤 ← 【第一次 URL 过滤】
  │   对搜索返回的每个 URL 执行 4 项检查：
  │   ├─ SSRF: 拒绝内网 IP（127.x/10.x/172.16.x/192.168.x/169.254.x/100.64.x/0.0.0.0/localhost/::1）
  │   ├─ 域名白名单: 非空时只允许列表内域名（支持 *.example.com 通配符）
  │   ├─ 域名黑名单: 匹配则拒绝
  │   └─ 路径拒绝: 正则匹配 pathname（如 /admin、\.env）
  │   → 不通过的 URL 从结果中移除，不返回给用户
  │
  ├─ Step 5: 写入会话白名单
  │   URL 归一化（移除默认端口、排序参数、去 fragment）
  │   → Redis SADD + EXPIRE 30分钟
  │
  └─ 返回 { success, sessionId, results[] }
```

### 4.3 抓取流程（POST /api/fetch）

```
请求 { url, sessionId }
  │
  ├─ Step 1: Zod 参数校验
  │   url: 合法 URL 格式
  │   sessionId: 非空字符串
  │
  ├─ Step 2: 会话白名单检查
  │   URL 归一化 → Redis SISMEMBER
  │   → 不在白名单: 403（必须先搜索）
  │
  ├─ Step 3: URL 二次过滤 ← 【第二次 URL 过滤】
  │   与搜索时完全相同的 4 项检查（SSRF/白名单/黑名单/路径）
  │   目的: 防止搜索后管理员更新了黑名单，旧白名单中的 URL 仍被访问
  │   → 不通过: 403
  │
  ├─ Step 4: HTTP 抓取（fetchUrl）
  │   ├─ Accept 头: 只接受文本类型（无 */*）
  │   ├─ 超时: 15秒 (AbortSignal.timeout)
  │   ├─ 重定向: 手动处理，同源才跟随，跨域拒绝，最多 5 次
  │   ├─ Content-Type 白名单: text/html, xhtml+xml, xml, text/plain, json
  │   │   → 图片/PDF 等在读取 body 前直接拒绝（body.cancel() 释放连接）
  │   └─ 大小限制: 流式读取，超过 5MB 立即停止
  │
  ├─ Step 5: 内容提取（extractContent）
  │   ├─ MIME 精确判断: split(';')[0].trim()
  │   ├─ text/html 或 xhtml → 提取:
  │   │   ├─ 路径 A: Readability
  │   │   │   JSDOM → Readability.parse() → article.content(HTML)
  │   │   │   → removeNoise(40+ CSS选择器清理噪声)
  │   │   │   → turndown(HTML → Markdown)
  │   │   │   → normalizeWhitespace
  │   │   ├─ 路径 B: Cheerio fallback（Readability 失败时）
  │   │   │   cheerio.load → 移除噪声 → 尝试 article/main/.content 等
  │   │   │   → $.html(el) → turndown → Markdown
  │   │   └─ 路径 C: body fallback
  │   │       $.html('body') → turndown → Markdown
  │   └─ 其他类型（text/plain, json）→ body.trim() 直接返回
  │
  ├─ Step 6: 会话续期（非致命）
  │   Redis EXPIRE 刷新 30分钟 TTL（滑动窗口）
  │   失败只记 warn 日志，不影响返回
  │
  ├─ Step 7: 截断
  │   内容超过 50,000 字符时截断 + "[Content truncated]"
  │
  └─ 返回 { success, content(Markdown), contentType }
```

### 4.4 噪声移除详细清单

内容提取时移除的 HTML 元素（40+ CSS 选择器）：

| 类别 | 选择器 | 移除目标 |
|---|---|---|
| 基础 | script, style, nav, header, footer, aside, iframe, noscript | JS/CSS/导航/页头页脚/侧栏/iframe |
| 图片媒体 | img, picture, figure, svg, video, audio, canvas, source, embed, object | 所有图片、视频、音频、SVG、Canvas |
| 广告 | [class\*="ad-"], [class\*="ads-"], [class\*="advert"], [id\*="ad-"], [id\*="ads-"], [id\*="advert"], [class\*="sponsor"], [id\*="sponsor"], [data-ad], [data-ads], [data-ad-slot], .ad, .ads, #ad, #ads | class/id/data 属性匹配的广告 |
| 社交 | [class\*="social"], [class^="share-"], [class\*=" share-"], [class\*="sharing"] | 分享按钮 |
| Cookie | [class\*="cookie"], [id\*="cookie"], [class\*="consent"], [id\*="consent"] | Cookie/GDPR 横幅 |
| 评论 | [class\*="comment"], [id\*="comment"], #disqus_thread | 评论区 |
| 表单 | form | 所有表单 |
| 弹窗 | [class\*="modal"], [id\*="modal"], [class\*="popup"], [id\*="popup"], [class^="overlay-"], [class\*=" overlay-"], [id\*="overlay"] | 模态框/弹窗 |
| 推荐 | [class\*="related"], [class\*="recommended"], [class\*="newsletter"], [id\*="newsletter"] | 相关文章/订阅区 |

### 4.5 设计模式

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **策略模式** | SearchEngine 接口 | Mock/Google/Brave 可切换，路由层不关心具体实现 |
| **装饰器模式** | CachedSearchEngine | 包装任意 SearchEngine 实例，透明添加缓存能力 |
| **组合模式** | CompositeQueryFilter/CompositeUrlFilter | 多个过滤器链式执行 |
| **依赖注入** | createSearchRouter(deps) / createFetchRouter(deps) | 路由工厂函数接收依赖，便于测试 mock |
| **单例模式** | getRedis() | Redis 客户端全局唯一实例 |
| **中间件管线** | Express middleware chain | 请求按注册顺序经过每层中间件 |

---

## 5. 安全体系

### 5.1 安全层次总览

| 层 | 机制 | 防护目标 |
|---|---|---|
| 传输层 | helmet 安全头 | XSS、点击劫持、MIME 嗅探 |
| 访问层 | JWT 鉴权 + CORS | 未授权访问、跨域滥用 |
| 流量层 | 请求限流 | DDoS、API 配额耗尽 |
| 输入层 | Zod 校验 + QueryGuard | 参数注入、PII 泄露、恶意查询 |
| URL 层 | DefaultUrlFilter（双重过滤） | SSRF、访问受限域名/路径 |
| 会话层 | UrlPool 白名单 | 任意 URL 抓取 |
| 网络层 | 重定向控制 + 超时 + 大小限制 | 跨域重定向 SSRF、资源耗尽 |
| 内容层 | Content-Type 白名单 + 噪声移除 | 二进制下载、XSS 残留 |

### 5.2 两次 URL 过滤的区别

| | 第一次（搜索时） | 第二次（抓取时） |
|---|---|---|
| 位置 | routes/search.ts | routes/fetch.ts |
| 方法 | filterUrls() 批量 | check() 单个 |
| 检查内容 | SSRF + 域名白/黑名单 + 路径拒绝 | 完全相同 |
| 目的 | 写入白名单前剔除危险 URL | 防止配置更新后旧白名单 URL 仍可访问 |

---

## 6. 配置

### 6.1 配置文件（config/server.yaml）

```yaml
server:
  port: 3100
  host: 0.0.0.0

redis:
  url: redis://redis:6379

pool:
  ttlSeconds: 1800              # 会话白名单 30 分钟过期

search:
  engine: google                # mock | google
  google:
    apiKey: "..."
    cx: "..."
  cacheTtlSeconds: 300          # 搜索结果缓存 5 分钟

auth:
  jwtSecret: "..."              # >=32 字符

fetch:
  timeoutMs: 15000              # 抓取超时 15 秒
  maxSizeBytes: 5242880         # 响应最大 5MB

filter:
  query:
    keywordBlacklist: []        # 搜索关键词黑名单
    piiPatterns: [...]          # PII 正则列表
  url:
    blockPrivateIPs: true       # SSRF 防护
    domainBlacklist: []         # 域名黑名单
    domainWhitelist: []         # 域名白名单（空=全部允许）
    pathDenyPatterns: []        # 路径拒绝正则
```

### 6.2 环境变量覆盖

| 环境变量 | 覆盖配置项 | 说明 |
|---|---|---|
| GOOGLE_API_KEY | search.google.apiKey | 优先级高于配置文件 |
| GOOGLE_CX | search.google.cx | 优先级高于配置文件 |
| JWT_SECRET | auth.jwtSecret | 优先级高于配置文件 |
| REDIS_URL | redis.url | 仅在 getRedis() 中作为 fallback |
| CONFIG_PATH | — | 配置文件路径，默认 config/server.yaml |
| LOG_LEVEL | — | 日志级别，默认 info |
| NODE_ENV | — | production 时日志为 JSON 格式 |

### 6.3 启动校验

- `engine: google` 时必须提供 apiKey 和 cx，否则启动失败
- `auth.jwtSecret` 如果配置了但长度 < 32 字符，启动失败
- Redis 启动时 ping 检查，不可达则 exit(1)

---

## 7. API 接口

### 7.1 POST /api/search

**请求：**
```json
{
  "query": "Node.js best practices",
  "maxResults": 5,
  "sessionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**成功响应（200）：**
```json
{
  "success": true,
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "results": [
    { "title": "...", "url": "https://...", "snippet": "..." }
  ]
}
```

**错误响应：** 400（参数/查询拒绝）、401（鉴权）、429（限流）、502（引擎故障）

### 7.2 POST /api/fetch

**请求：**
```json
{
  "url": "https://nodejs.org/en/guides",
  "sessionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**成功响应（200）：**
```json
{
  "success": true,
  "content": "## Guide Title\n\nParagraph with [link](https://...)...",
  "contentType": "text/html; charset=utf-8"
}
```

**错误响应：** 400（参数）、401（鉴权）、403（白名单/过滤拒绝）、429（限流）、502（抓取/提取失败）

### 7.3 GET /health

**成功响应（200）：**
```json
{
  "status": "ok",
  "engine": "google",
  "checks": { "redis": "ok" }
}
```

**降级响应（503）：**
```json
{
  "status": "degraded",
  "engine": "google",
  "checks": { "redis": "error" }
}
```

完整 OpenAPI 3.0 规范见 `openapi.yaml`。

---

## 8. 部署

### 8.1 Docker 部署

```bash
docker compose up --build
```

docker-compose.yaml 定义两个服务：
- **api-server**：Node.js 应用，端口 3100，依赖 Redis 健康检查通过后启动
- **redis**：Redis 7 Alpine，持久化到 docker volume

Dockerfile 采用多阶段构建：
- Stage 1（builder）：安装全部依赖 + TypeScript 编译
- Stage 2（production）：仅安装生产依赖 + 复制编译产物，安装 CA 证书

### 8.2 本地开发

```bash
npm install
npm run dev        # tsx 直接运行 TypeScript
npm test           # vitest 运行测试
npm run build      # tsc 编译
npm start          # 运行编译产物
```

---

## 9. 扩展指南

### 9.1 添加新搜索引擎

1. 创建 `src/search/brave.ts`，实现 `SearchEngine` 接口
2. `src/shared/types.ts` 中 `engine` 类型加 `'brave'`
3. `src/index.ts` 加 else if 分支
4. `src/config/loader.ts` 加环境变量覆盖
5. CachedSearchEngine 可直接复用

### 9.2 添加新过滤器

1. 实现 `QueryFilter` 或 `UrlFilter` 接口
2. 用 `CompositeQueryFilter` / `CompositeUrlFilter` 组合多个过滤器
3. 注入到路由工厂函数

### 9.3 添加新中间件

在 `src/index.ts` 中按需注册，注意顺序：安全头 → 限流 → 鉴权 → 路由 → 错误处理

---

## 10. 已知局限

| 局限 | 说明 | 后续方案 |
|---|---|---|
| 限流不跨实例 | express-rate-limit 默认内存存储 | 切换为 rate-limit-redis |
| 无熔断器 | 搜索引擎持续不可用时每次请求都重试 | 引入 circuit breaker |
| 无 Prometheus metrics | 缺少请求延迟、错误率等监控指标 | 集成 prom-client |
| User-Agent 硬编码 | 部分网站可能拒绝 | 可配置化 |
| Google API 新项目不可用 | 2026-01-20 起新项目被封锁 | 替换为 Brave/DuckDuckGo/SearXNG |
