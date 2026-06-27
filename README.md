# PlayerLedger

玩家儲值紀錄查詢工具。提供後台介面讓管理員快速查詢玩家的儲值歷程與明細。

## 架構

```
PlayerLedger/
├── PlayerLedgerFrontend/   # Next.js 前端
└── PlayerLedgerBackend/    # Golang API Server
```

| 層級 | 技術 |
|------|------|
| 前端 | Next.js |
| 後端 | Go (Golang) |

## 快速開始

### 後端

```bash
cd PlayerLedgerBackend
go mod download
go run main.go
```

### 前端

```bash
cd PlayerLedgerFrontend
npm install
npm run dev
```

## 功能

- 查詢玩家儲值紀錄
- 依條件篩選（玩家 ID、日期區間、金額等）
- 儲值明細檢視
