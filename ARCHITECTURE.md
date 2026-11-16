# ARCHITECTURE.md — omniroute

> 本文件回答「这个仓库有哪些模块、入口在哪、改动影响谁」。
> 详细规范与编码约定见 `AGENTS.md`（765 行，改代码前必读）。
> 最后校准：2026-11-17。

---

## 1. 一句话定位

统一 AI 路由网关：356 个 provider 抽象成单一 OpenAI 兼容接口，内置 RTK+Caveman 压缩、
自动故障转移、MCP/A2A 协议、桌面端与 PWA。

```
仓库      luyuehm/omniroute (private)
版本      3.8.51
运行时    Node.js（type: module）+ Next.js
包管理    bun（部分脚本）/ npm
```

---

## 2. 顶层结构

```
omniroute/
├── src/                  ★ Next.js 应用与业务域
├── open-sse/             ★ 流式推理引擎（provider 适配层，独立子包）
├── packages/browser-pool/ 浏览器池
├── electron/             桌面端
├── public/               PWA 静态资源
├── config/               运行时配置
├── skills/               内置 Skill
├── scripts/              构建/评测/性能/文档生成脚本
├── docs/                 文档
├── docker/ contrib/      部署与贡献
├── examples/ tests/
└── AGENTS.md             765 行编码规范（**必读**）
```

---

## 3. 核心模块

### src/（应用层）

| 目录 | 职责 |
|---|---|
| `app/` | Next.js 路由（含 `(dashboard)` 分组、错误码页 400/401/403/408/429/500/502/503、`api/`、`a2a/`、`auth/`） |
| `domain/` | **业务核心**：路由决策与策略（见 §4） |
| `server/` | 服务端逻辑 |
| `lib/` `shared/` | 公共库 |
| `store/` `models/` | 状态与数据模型 |
| `sse/` `mitm/` | 流式响应、中间人代理 |
| `middleware/` `hooks/` | 请求中间件与 React hooks |
| `i18n/` | 国际化 |

### open-sse/（引擎层，规模最大）

```
executors/    111 个 ts —— 各 provider 的执行器（真正发请求的地方）
services/     232 个 ts —— 服务实现（最大头）
config/        58 个 ts —— 配置与 provider 定义
handlers/      25 个 ts —— 请求处理器
mcp-server/   MCP 协议服务端
transformer/  请求/响应格式转换
index.ts      对外入口
```

**判据**：加新 provider → `open-sse/executors/` + `config/`；改流式解析 → `transformer/`。

---

## 4. src/domain 业务域（路由核心）

| 文件 | 职责 |
|---|---|
| `pipeline.ts` | **主流水线**，请求经过的完整链路 |
| `policyEngine.ts` | 策略引擎（总控） |
| `fallbackPolicy.ts` | 故障转移策略 |
| `lockoutPolicy.ts` | 锁定/熔断策略 |
| `degradation.ts` | 降级策略 |
| `comboResolver.ts` | 组合路由解析 |
| `tagRouter.ts` | 按 tag 路由 |
| `modelAvailability.ts` | 模型可用性判定 |
| `providerExpiration.ts` | provider 过期管理 |
| `quotaCache.ts` | 配额缓存 |
| `costRules.ts` | 成本规则 |
| `connectionModelRules.ts` | 连接模型规则 |
| `configAudit.ts` | 配置审计 |
| `assessment/` | 评估子域 |
| `persistence/` | 持久化 |

**改路由行为必看**：`pipeline.ts` → `policyEngine.ts` → 具体策略文件。

---

## 5. 关键数据流

```
请求    OpenAI 兼容请求 → src/app/api → middleware → domain/pipeline
                          ↓
决策    policyEngine（选 provider）
         ├→ modelAvailability 判定可用
         ├→ tagRouter / comboResolver 选路
         ├→ quotaCache / costRules 判定额度
         └→ fallbackPolicy 准备备选链
                          ↓
执行    open-sse/executors/<provider> → 上游 provider
                          ↓
转换    open-sse/transformer 统一成 OpenAI 格式
                          ↓
返回    流式 SSE → 客户端
                          ↓
异常    degradation 降级 → lockoutPolicy 熔断 → fallbackPolicy 换 provider
```

---

## 6. 已知陷阱

### ① provider 数量巨大（356），改一处要检查同类

`open-sse/executors/` 111 个执行器 + `config/` 58 个配置。修改公共行为（如认证头构造、
错误码映射）时，不能只改一个 provider，需评估全部执行器。

### ② 打包体积与内存

`dev` 脚本带 `--max-old-space-size=8192`。构建/开发需要至少 8GB 堆，小内存机器会 OOM。

### ③ open-sse 是独立子包

有独立 `package.json`。改它之后若不走构建流程，`src/` 引用到的仍是旧产物。

### ④ 双运行时（electron + Next.js）

`electron/` 桌面端与 `src/` Web 端共享逻辑。改共享模块要验证两个运行时。

### ⑤ 端口不统一

不同组件默认端口不同（`8080` inspector proxy、`3000` 主服务、`9223` CDP 相关）。
部署时注意别撞端口。

---

## 7. 验证门禁

```bash
npm run build            # 构建
npm test                 # 测试（若存在）

# 性能/路由评测（改路由逻辑后必跑）
npm run eval:router
npm run bench:routing-events
npm run bench:compression
```

**lint 口径**：以仓库既有 CI 为准，历史基线债不作为本次改动的阻塞项。

---

## 8. 改动影响面

| 改动 | 影响 |
|---|---|
| `domain/pipeline.ts` | 全部请求的必经路径，影响面最大 |
| `domain/*Policy.ts` | 路由/降级/熔断行为，直接影响可用性 |
| `open-sse/executors/*` | 单个 provider 行为，需评估同类 |
| `open-sse/transformer/*` | 影响所有 provider 的格式转换 |
| `src/app/api/*` | 对外 API 契约 |
| `electron/` | 仅桌面端 |

---

## 9. 相关文档

- `AGENTS.md` — 765 行编码规范与约定（**必读**）
- `docs/` — 架构与专题文档
- `scripts/router-eval/` — 路由评测框架
- `scripts/docs/` — 文档自动生成
