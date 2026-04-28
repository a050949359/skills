---
name: api-status
description: >
  API 開發狀態與安全機制追蹤 skill（Laravel 專用）。輸入「as」時啟用。
  從 project-status.yml 讀取專案設定，api-status.yml 存在則顯示現狀，不存在則建立。
---

# API Status Skill

## 用途

追蹤 Laravel 專案每支 API 的開發進度與安全機制覆蓋狀況，結果儲存於 `api-status.yml`。
`project-status.yml` 是必要前提，提供路徑設定與專案資訊。

---

## 前提檢查

執行任何流程前，先確認 `project-status.yml` 存在：

- 存在 → 讀取 `api_status.submodule_path`，決定 `api-status.yml` 路徑：
  - 不為空 → `{submodule_path}/api-status.yml`
  - 為空字串 → 專案根目錄 `api-status.yml`
- 不存在 → 輸出提醒後停止：
  ```
  找不到 project-status.yml，請先執行「ps local」或「ps .gitmodules」初始化專案。
  ```

---

## 觸發邏輯

```
輸入「as」
  ├── project-status.yml 不存在 → 輸出提醒，停止
  ├── api-status.yml 不存在 → 進入【建立】
  └── api-status.yml 存在 → 進入【確認現狀】

輸入「/api/xxx 做完了」或「更新 XXX 狀態」→ 進入【更新單支 API】
```

---

## 【建立】

1. 從 `project-status.yml` 讀取：
   - `api_status.submodule_path`（路徑）
   - `project.structure.middlewares`（自訂 middleware 清單參考）
2. 掃描 `app/Http/Middleware/`，讀取 `@security-field` 標註，建立欄位清單：
   - 有標註 → 加入 `security_fields`
   - 無標註且不在 `ignore_middleware` → 輸出警告
   - 列於 `ignore_middleware` → 靜默略過
3. 讀取 `routes/api.php`，列出所有路由、方法、handler
4. 逐一讀取對應 Controller，判斷各安全欄位值（`true` / `false` / `unknown`）
5. 依下方格式規範產生 `api-status.yml`，不自行發明欄位
6. 輸出摘要

---

## 【確認現狀】

1. 讀取 `api-status.yml`
2. 輸出摘要：總數統計、安全機制缺漏清單、進行中清單

---

## 【更新單支 API】

1. 讀取 `api-status.yml`，找到對應路由
2. 重新讀取對應 Controller，補齊所有 `security` 欄位
3. `status` 改為 `complete`
4. 更新 `last_updated` 與 `summary`
5. 輸出更新後的條目

---

## 安全欄位來源

### 內建欄位（硬編碼偵測）

| 欄位 | 偵測條件 |
|------|----------|
| `auth` | middleware 含 `auth:sanctum`、`auth:api`、`jwt` |
| `throttle` | middleware 含 `throttle:` |
| `form_request` | Controller 方法參數型別繼承自 `FormRequest` |
| `admin_only` | middleware 含 `EnsureAdmin`、`role:admin`、`can:admin` |

### 自訂欄位（PHPDoc 標註）

```php
/**
 * @security-field ensure_verified
 * @security-desc 信箱驗證確認
 */
class EnsureEmailVerified {}
```

缺少 `@security-field` 且不在 `ignore_middleware` 時輸出：
```
⚠️ 以下自訂 middleware 未登記 @security-field，已略過不納入追蹤：
- EnsureEmailVerified
請補上 @security-field 標註，或加入 ignore_middleware 清單以略過警告。
```

---

## api-status.yml 格式規範

**唯一合法格式，不得新增規範外欄位。**

```yaml
last_updated: 2026-04-28T14:32:00
project: my-app
ignore_middleware:
  - LogSensitiveAction
security_fields:
  auth: auth middleware（sanctum / jwt）
  throttle: rate limiting
  form_request: FormRequest 輸入驗證
  admin_only: 管理員限定
  ensure_verified: 信箱驗證確認
summary:
  total: 8
  complete: 5
  in_progress: 3
routes:
  - method: POST
    path: /api/auth/login
    handler: AuthController@login
    security:
      auth: false
      throttle: true
      form_request: true
      admin_only: false
      ensure_verified: false
    status: complete
    notes: throttle:5,1

  - method: DELETE
    path: /api/admin/users/{id}
    handler: AdminUserController@destroy
    security:
      auth: true
      throttle: false
      form_request: false
      admin_only: true
      ensure_verified: unknown
    status: in_progress
    notes: FormRequest 尚未建立
```

**security 值**：

| 值 | 說明 |
|----|------|
| `true` | 已套用 |
| `false` | 未套用 |
| `unknown` | 無法從程式碼確定，需人工確認 |

**status 值**：

| 值 | 說明 |
|----|------|
| `complete` | 開發與安全機制皆完整 |
| `in_progress` | 開發中，部分安全機制待補 |

---

## 輸出摘要格式

```
## API 狀態摘要

共 8 支 API｜完成 5｜進行中 3

### ⚠️ 安全機制未完整
- DELETE /api/admin/users/{id}：缺 throttle、form_request

### ⚠️ 以下自訂 middleware 未登記 @security-field，已略過不納入追蹤
- EnsureEmailVerified

### 🚧 進行中
- DELETE /api/admin/users/{id}（FormRequest 尚未建立）

### ✅ 完成
- POST /api/auth/login
- POST /api/articles
```

---

## 輸出語言

所有輸出一律使用**繁體中文**。
`api-status.yml` 的 value 中英混用，以精簡為主。
