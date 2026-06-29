# PlayerLedger

> 玩家儲值紀錄查詢工具 — 提供 CMS 後台讓管理員查詢玩家的儲值歷程與明細。

本 repository 為 **Monorepo**，包含前後端兩個獨立部署的子專案。本文件聚焦於兩者的**架構設計 overview**；細節規格見各子專案的 `docs/specs/` 與 `docs/adr/`。

```
PlayerLedger/
├── PlayerLedgerFrontend/   # Next.js 16 BFF + CMS 介面（ECS Fargate）
├── PlayerLedgerBackend/    # Go (Gin) API Server（PostgreSQL + Redis）
└── infra/                  # 基礎設施
```

| 子專案 | 技術 | 角色 | 文件 |
|--------|------|------|------|
| **Frontend** | Next.js 16 · App Router · Tailwind v4 · shadcn/ui | BFF + CMS UI | [README](https://github.com/Yintc123/PlayerLedgerFrontend) · [specs](https://github.com/Yintc123/PlayerLedgerFrontend/tree/main/docs/specs) · [adr](https://github.com/Yintc123/PlayerLedgerFrontend/tree/main/docs/adr) |
| **Backend** | Go 1.25 · Gin · GORM · PostgreSQL 15 · Redis 7 | REST API | [README](https://github.com/Yintc123/PlayerLedgerBackend) · [specs](https://github.com/Yintc123/PlayerLedgerBackend/tree/main/docs/specs) · [adr](https://github.com/Yintc123/PlayerLedgerBackend/tree/main/docs/adr) |

---

## 共用設計原則

兩端共享一套契約與紀律，這是兩個架構能並行開發的前提：

- **SDD（Schema-Driven Development）** — 後端 `schema/openapi.yaml`（OpenAPI 3.1）是前後端唯一契約；任何 API 變更先改 schema，再調整測試與實作，不對不存在的端點做猜測性實作。
- **TDD（Test-Driven Development）** — Red → Green → Refactor；無對應測試不得新增功能程式碼，spec 內已附測試清單。
- **共用 token 模型** — 前端 ADR-010 對齊後端 ADR-007，雙方對 JWT claims、refresh rotation 的理解一致。

### 系統脈絡

瀏覽器永遠只與前端 BFF 通訊；**JWT 不進瀏覽器**，僅以 HttpOnly Cookie 攜帶 sessionId，token 存於 Redis session。

```
使用者瀏覽器
  │  Cookie: __Host-sid（HttpOnly / Secure / SameSite）
  ▼
CloudFront ─▶ API Gateway/ALB ─▶ ┌─────────────────────────┐
                                 │  前端 BFF（Next.js）      │  ← 架構設計 A
                                 └───────────┬─────────────┘
                                Bearer JWT（由 BFF 注入）
                                             ▼
                  API Gateway/ALB ─▶ ┌─────────────────────────┐
                                     │  後端 API（Gin）          │  ← 架構設計 B
                                     └───────────┬─────────────┘
                                     ┌───────────┴───────────┐
                                     ▼                       ▼
                              RDS PostgreSQL          ElastiCache Redis
```

---

## 架構設計 A — 前端 BFF（PlayerLedgerFrontend）

> 📖 子專案文件：[PlayerLedgerFrontend/README.md](https://github.com/Yintc123/PlayerLedgerFrontend)

定位為 **Backend-for-Frontend**：不只是 UI，更是瀏覽器與後端之間的安全閘道。瀏覽器拿不到任何後端 token，所有跨域呼叫一律經 BFF 代理。

### 分層

```
src/
├── app/
│   ├── (auth)/         # 登入、註冊頁
│   ├── (cms)/          # 受保護的 CMS 畫面（SSR decode role 注入）
│   └── api/[...path]/  # catch-all proxy：注入 Bearer、X-Forwarded-For append
├── lib/
│   ├── session/        # Redis session、cookie、client-session、token refresh（mutex + CAS）
│   ├── auth/           # login / logout / refresh / decode-token
│   ├── players/        # 玩家查詢（search / get）
│   ├── topups/         # 儲值紀錄（list / get / create / update）
│   ├── logger/         # pino + PII redact
│   └── observability/  # CloudWatch EMF metrics
├── proxy.ts            # 路由層守門：CSRF Origin 檢查、CSP nonce、PUBLIC_PATHS、rate limit
└── schema/             # 由 OpenAPI 產生的型別
```

### 設計重點

| 面向 | 設計 |
|------|------|
| **Session 邊界** | sessionId 存 HttpOnly Cookie（`__Host-` prefix / Secure / SameSite=Strict）；token 與使用者狀態存 Redis，TTL = min(8h, abs_exp − now) |
| **Token Refresh** | `getValidAccessToken()` 內含 Redis mutex + Lua CAS；多請求並行時只有一個觸發後端 refresh，其餘 bounded polling 等待（100ms × 9s）|
| **CSRF 防護** | `proxy.ts` 在路由層攔截 POST/PATCH/DELETE，驗證 Origin 白名單；SameSite cookie 為第二道防線 |
| **限流** | 登入採帳號層 lockout（5 次失敗鎖 15 分，fail-closed）；其餘採 IP 層（fail-open）|
| **安全標頭** | HSTS / CSP（nonce via proxy header）/ X-Frame-Options / COOP / CORP 等 |
| **渲染策略** | Server-first（RSC + SSR）避免 waterfall；UI 可見欄位由後端資料驅動（PII 遮罩在後端執行）|
| **可觀測性** | pino（自動 redact token/password/PII）→ CloudWatch；EMF metrics；OTel + X-Ray；前端 telemetry 端點 `/api/client-errors`、`/api/vitals`、`/api/csp-report` |
| **部署** | 多階段 Dockerfile（standalone、非 root UID 1001、tini）→ CloudFront → API Gateway/ALB → ECS Fargate |

> 完整 ADR 共 22 篇，涵蓋 BFF route 結構、session API、token mutex、header forwarding、CSRF、health probe 等決策。

---

## 架構設計 B — 後端 API（PlayerLedgerBackend）

> 📖 子專案文件：[PlayerLedgerBackend/README.md](https://github.com/Yintc123/PlayerLedgerBackend)

定位為**無狀態 REST API**：JWT（HS256）驗證，所有 session/token/限流狀態存 Redis，支援多副本水平擴展與 graceful shutdown。

### 分層（依賴單向，無循環）

```
internal/
├── handler/      # HTTP handler（E2E 測試層）— 嚴格對應 OpenAPI request/response
├── service/      # 業務邏輯（Unit 測試層）— repository 以 interface 注入
├── repository/   # 資料存取（Integration 測試層）— 真實 DB
├── model/        # GORM entity
├── dto/          # API response DTO
├── apperr/       # Domain error（統一 HandleError 對應 HTTP 狀態碼）
└── pagination/   # 分頁 helper（cursor / offset）

pkg/   # 可重用基礎建設：jwt、redis（Lua script）、ratelimit、audit、logger（Zap）、metrics…
```

### 設計重點

| 面向 | 設計 |
|------|------|
| **雙使用者系統** | JWT `utype` claim 區分 `cms` / `member`，`RequireOwnership` middleware 強制隔離；CMS 三層角色 admin/user/viewer（ADR-003）|
| **Token 安全** | Access 15m 短期；Refresh 採 family-based rotation + replay detection，偵測重放即廢整個 family；10s grace window 容忍網路重試；絕對 TTL 防無限延期（ADR-007）|
| **錯誤處理** | `HandleError` 以 `errors.As` 偵測 `validator.ValidationErrors` → 400 + `details[]`，handler 零負擔（ADR-004/005）|
| **資料完整性** | 禁止 AutoMigrate，改用 golang-migrate 版本化；金融紀錄不可刪除（DepositRecord 無 deleted_at），狀態轉換嚴格驗證（ADR-001）|
| **稽核** | app log 與 audit log 分離；所有安全事件（登入、token 換發、session 撤銷、admin 操作）寫獨立 sink，best-effort 不阻塞主流程 |
| **無狀態 / 高可用** | session/token/限流計數存 Redis；graceful shutdown（SIGTERM → 排空 → 依序關閉 HTTP/DB/Redis/logger）|
| **可觀測性** | Zap 結構化 JSON；Prometheus `/metrics`；X-Ray tracing；`/health`(liveness) 與 `/health/ready`(readiness) 解耦 |
| **部署** | Dockerfile / Lambda 變體 → ECS / K8s + RDS PostgreSQL + ElastiCache Redis |

### 資料模型

| 模型 | 說明 | 重點欄位 |
|------|------|---------|
| **CMSUser** | 後台操作人員 | `username`(unique) · `password_hash`(bcrypt) · `role`(admin/user/viewer) · 軟刪除；至少保留一名 admin |
| **Member** | 玩家（查詢唯讀）| `display_name` · `external_id` · `email`/`phone`（PII，依角色遮罩）· `status`(active/frozen/closed) |
| **DepositRecord** | 儲值交易（金融，不可刪除）| `amount`(>0) · `currency`(TWD) · `payment_method` · `status`(狀態機) · `operator_id`/`operator_ip` |

### API 端點總覽

Base path `/api`，完整契約見 [`schema/openapi.yaml`](https://github.com/Yintc123/PlayerLedgerBackend/blob/main/schema/openapi.yaml)。

| 網域 | 端點 |
|------|------|
| **Auth** | `register` · `login` · `refresh` · `logout` · `sessions`（列出 / 撤銷 / 全裝置登出）|
| **玩家查詢** | `GET /cms/players`（cursor 搜尋）· `GET /cms/players/{id}` |
| **儲值紀錄** | `POST/GET /cms/deposit-records` · `GET/PATCH /cms/deposit-records/{id}` · `GET /me/deposit-records` |
| **CMS 使用者** | `GET /cms/users` · `GET/PATCH/DELETE /cms/users/{id}` · `PATCH /cms/users/me` |

---

## 快速開始

需求：Go 1.25+、Node.js、PostgreSQL 15+、Redis 7+。

```bash
# 後端
cd PlayerLedgerBackend && go mod download && go run main.go   # http://localhost:8080

# 前端
cd PlayerLedgerFrontend && npm install && npm run dev          # http://localhost:3000
```

環境變數請參考各子專案的 `.env.example`；測試見各自 README。
