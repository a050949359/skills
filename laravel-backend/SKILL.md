---
name: laravel-backend-security
description: >
  Laravel 後端開發規範與安全 skill。輸入「cr」時啟用。
  用於撰寫或審查 Laravel 程式碼（Controller、Route、Model、FormRequest），
  以及 API 設計、權限、XSS 防護、onboarding 相關任務。
---

# Laravel 後端開發規範與安全 Skill

## 用途

作為 Laravel 後端開發的規範依據，適用於以下情境：

- **Code Review**：依規範逐條審查，指出違規並提供修正範例
- **開發協助**：產生符合規範的程式碼片段（Route、Controller、FormRequest 等）
- **安全稽核**：掃描潛在安全漏洞，輸出結構化報告
- **Onboarding**：解釋規範背後的原因

---

## 輸出格式

### Code Review

```
## Code Review 結果

### ✅ 符合規範
- [條列通過項目]

### ⚠️ 需要修正
| 項目 | 問題 | 建議修正 |
|------|------|----------|
| ...  | ...  | ...      |

### 修正後範例
[可直接使用的程式碼]
```

### 安全稽核

```
## 安全稽核報告

風險等級：高 / 中 / 低
違規項目：[條列]
建議優先處理：[最高風險項目]
```

---

## 規範內容

### 1. 路由與 API 設計

- API 路由統一放於 `routes/api.php`，回傳格式為 JSON。
- 僅對外開放必要 API，不暴露未使用或測試用路由。
- 敏感操作（PUT / DELETE / POST）必須加上 `auth:sanctum` middleware，並於 Controller 層再次檢查權限。

```php
// ❌ 錯誤
Route::post('/articles', [ArticleController::class, 'store']);

// ✅ 正確
Route::middleware('auth:sanctum')->post('/articles', [ArticleController::class, 'store']);
```

---

### 2. 權限與授權

- 僅信任後端 Controller 層的權限檢查，前端僅作 UI 隱藏。
- 操作資源前必須檢查 `user_id` 或 `isAdmin()`，不可僅依賴路由保護。
- 管理員 API 必須加上 `EnsureAdmin` middleware。

```php
// ❌ 錯誤
public function update(Request $request, Article $article) {
    $article->update($request->all());
}

// ✅ 正確
public function update(Request $request, Article $article) {
    if ($article->user_id !== auth()->id() && !auth()->user()->isAdmin()) {
        abort(403);
    }
    $article->update($request->validated());
}
```

---

### 3. 輸入驗證與 XSS 防護

- 所有用戶輸入必須通過 FormRequest 驗證，並加上 `NoMaliciousPattern` 規則。
- 不允許直接渲染未過濾 HTML，前端一律用 `{{ 變數 }}` 輸出。
- 產生 SVG、圖片等動態內容時，用 `htmlspecialchars()` 處理。
- Model 必須明確定義 `$fillable`，防止 Mass Assignment。

```php
// ❌ 錯誤
class Article extends Model {}

// ✅ 正確
class Article extends Model {
    protected $fillable = ['title', 'content', 'summary'];
}
```

---

### 4. 敏感資料保護

- 密碼、Token、API Key 不得出現於 log、response body 或 git history。
- 敏感設定統一放 `.env`，程式碼中以 `config()` 或 `env()` 讀取。
- 確認 `.env` 已列於 `.gitignore`。

---

### 5. 認證、CORS 與 Rate Limiting

- 前後端分離時，統一使用 Bearer Token（Sanctum）認證。
- 跨網域需求正確設定 `config/cors.php`，僅允許可信網域。
- 登入、重置密碼、發送 OTP 等端點加上 `throttle` middleware。

```php
Route::middleware(['throttle:5,1'])->post('/login', [AuthController::class, 'login']);
```

---

### 6. 靜態資源管理

- `public/images`、`public/build` 等靜態檔案需手動部署或自動同步。
- 不將靜態資源存入資料庫或經由 API 傳遞。
- 檔案上傳功能必須驗證 MIME 類型、限制大小、儲存至非 public 路徑。

---

### 7. 套件與依賴管理

- 定期執行 `composer update` 並用 `composer-unused` 檢查未使用套件。
- 僅安裝實際需要的套件。

---

### 8. 例外與錯誤處理

- API 錯誤統一回傳格式：
```json
{
  "status": "error",
  "code": 422,
  "message": "驗證失敗",
  "errors": { "title": ["標題為必填"] }
}
```
- 不回傳詳細 exception trace 至前端（`APP_DEBUG=false` in production）。

---

### 9. 效能：避免 N+1 查詢

- 查詢關聯資料時使用 eager loading（`with()`）。

```php
// ❌ 錯誤
$articles = Article::all();
foreach ($articles as $article) {
    echo $article->user->name;
}

// ✅ 正確
$articles = Article::with('user')->get();
```

---

### 10. 測試與 CI/CD

重要 API 需有 Feature Test，覆蓋：權限邊界、輸入驗證、資料正確性、效能（無 N+1）。
測試撰寫規範、GitHub Actions workflow、PHPStan 設定、PR Checklist 請輸入 `cicd`。

---

## 能力等級指引

| 等級 | 描述 |
|------|------|
| Level 1 | 了解規範條目，能在提示下遵守 |
| Level 2 | 能主動套用規範，無需提醒 |
| Level 3 | 能在 Code Review 中發現他人違規 |
| Level 4 | 能解釋規範背後的安全原理 |
| Level 5 | 能設計自動化檢查（phpstan rules、custom middleware、CI pipeline）確保團隊合規 |

---

## 不適用情境

前端框架（Vue/React）架構問題、資料庫 schema 設計（非安全相關）、DevOps 部署流程（CI/CD 測試部分除外）。
