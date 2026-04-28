---
name: project-context
description: >
  專案狀態恢復與記憶 skill。輸入「ps」「ps local」「ps .gitmodules」
  「ps close」「ps save」「ps log」時啟用。
  負責建立與維護 project-status.yml 與 dev-context.yml，適用任何語言與框架的專案。
---

# Project Context Skill

## 用途

掃描並記錄專案完整快照至 `project-status.yml`。
此檔案是整個 skill 體系的基礎，供其他 skill（如 `as`）讀取專案設定與資訊。
初始化一次後，每次 `ps` 自動更新動態資訊。

`dev-context.yml` 獨立記錄技術決策與問題解法，進 git 作為專案知識庫。

---

## 觸發詞

| 觸發詞 | 說明 |
|--------|------|
| `ps` | `project-status.yml` 存在 → 進入主流程；不存在 → 輸出提醒 |
| `ps local` | 第一次初始化，相關檔案放專案根目錄 |
| `ps .gitmodules` | 第一次初始化，從 `.gitmodules` 讀取 submodule 路徑 |
| `ps close` | 功能完成，更新 `project-status.yml` |
| `ps save` | 開發中斷，記錄當前進度至 `project-status.yml` |
| `ps log` | 記錄技術決策或問題解法至 `dev-context.yml` |

**`ps` 且 `project-status.yml` 不存在時輸出**：
```
project-status.yml 不存在，請指定初始化模式：
- ps local　　　→ 相關檔案放專案根目錄
- ps .gitmodules → 使用 submodule 管理相關檔案
```

---

## 觸發詞前置處理

```
├── 「ps」
│     ├── project-status.yml 存在 → 讀取所有設定 → 進入【主流程】
│     └── 不存在 → 輸出提醒，停止
│
├── 「ps local」
│     └── api_status.submodule_path = "" → 進入【初始化】
│
└── 「ps .gitmodules」
      ├── 讀取 .gitmodules
      │     ├── 只有一個 submodule → api_status.submodule_path = 該路徑
      │     └── 多個 submodule → 取第一個路徑，輸出清單供日後修改：
      │           submodule 路徑已設定為 {第一個路徑}。
      │           若需更換，請手動修改 project-status.yml 的 api_status.submodule_path：
      │           - {path1}
      │           - {path2}
      └── 進入【初始化】
```

---

## 【初始化】

目標：掃透專案，建立完整快照，寫入 `project-status.yml`。
完成後直接輸出摘要，不需再執行 `ps`。

### 1. 基本資訊

- `git status`（分支、未提交檔案）
- `git log --oneline -5`
- 根目錄結構（2-3 層）
- 執行 `php artisan about --json` 取得框架版本、環境資訊
- 讀取 `.env`：非敏感設定（`APP_NAME`、`DB_CONNECTION`、`QUEUE_CONNECTION`、`CACHE_DRIVER`）

### 2. 套件資訊

- 執行 `composer show --direct --format=json` 取得後端套件清單
- 讀取 `package.json` 取得前端套件清單

### 3. 結構資訊

掃描以下目錄，讀取每個 `.php` 檔案的 class 名稱：

| 目錄 | 對應欄位 |
|------|----------|
| `app/Models/` | `structure.models` |
| `app/Http/Controllers/` | `structure.controllers` |
| `app/Http/Middleware/` | `structure.middlewares` |
| `app/Services/` | `structure.services` |
| `app/Jobs/` | `structure.jobs` |
| `app/Events/` | `structure.events` |
| `app/Enums/` | `structure.enums` |
| `app/Traits/` | `structure.traits` |

目錄不存在則略過，不填空陣列。

### 4. 路由資訊

執行 `php artisan route:list --json`，取得每支路由的：
- `method`、`path`、`middleware`、`handler`

### 5. 功能資訊

依照以下優先順序讀取功能描述：

1. `README.md`：取得專案功能概述
2. `docs/` 資料夾（若存在）：取得模組說明文件
3. Controller class docblock（若有）：取得模組職責描述
4. 無文件時：依 class 名稱與 public method 名稱推斷功能描述

每個 Controller / Service 產生一筆 `features` 條目：

`status` 判斷規則：
- 對應路由全部存在且有 middleware → `complete`
- 部分路由缺少 middleware 或 FormRequest → `in_progress`
- 無法判斷 → 留空字串

### 6. 寫入 project-status.yml

依下方格式規範寫入所有掃描結果：
- 未知欄位填空字串或空陣列
- 不自行發明規範外的欄位

### 7. 輸出摘要

---

## 【主流程】

每次 `ps` 時執行，更新動態資訊：

1. 讀取 `git status`（分支、未提交檔案）
2. 讀取 `git log --oneline -5`
3. 更新 `project-status.yml`（`branch`、`uncommitted`、`last_updated`）
4. 輸出摘要

---

## 【ps close】功能完成

**情境**：某功能開發完成，更新專案狀態。

1. 從對話與 `git diff HEAD` 擷取本次完成的功能
2. 更新 `project-status.yml`：
   - `features` 對應模組的 `status` 改為 `complete`
   - `structure` 補入新增的 class
   - `routes` 補入新增的路由
   - `in_progress` 清空
   - `last_updated` 更新
3. 輸出變更內容

---

## 【ps save】開發中斷

**情境**：開發到一半需要中斷，記錄當前進度。

1. 讀取 `git diff HEAD`，取得新增／修改的檔案與內容摘要
2. 從對話擷取：目前做到哪個步驟、還剩什麼沒做
3. 更新 `project-status.yml`：
   - `in_progress.feature`：進行中的功能名稱
   - `in_progress.last_action`：上次做到的步驟
   - `next`：下次要繼續的事項
   - `key_files`：本次修改的相關檔案
   - `uncommitted`：未提交的檔案清單
   - `last_updated` 更新
4. 輸出寫入內容

---

## 【ps log】記錄技術決策

**情境**：記錄技術決策或解決的問題，維護 `dev-context.yml`。

1. 從對話擷取本次的技術決策或問題解法
2. 讀取現有 `dev-context.yml`（不存在則建立）
3. 掃描現有條目，找出：
   - **可合併**：topic 相同或高度相似的條目
   - **衝突**：同一 topic 但 decision 不一致的條目
4. 輸出掃描結果：
   ```
   ### 新增條目
   - topic: Token 儲存方式
     decision: 使用 httpOnly cookie

   ### 可合併
   - 以下兩筆 topic 相似，建議合併：
     1. Token 儲存方式
     2. refresh token 存放位置

   ### 衝突
   - 無
   ```
5. 寫入（含合併結果）`dev-context.yml`

---

## 輸出格式

```
## 專案狀態摘要

專案：{name}　分支：{branch}　上次工作：{last_updated}

### 基本資訊
技術棧：{stack}　語言：{language}
DB：{env.DB_CONNECTION}　Queue：{env.QUEUE_CONNECTION}　Cache：{env.CACHE_DRIVER}

### 結構
Models：{models 條列}
Controllers：{controllers 條列}
Services：{services 條列}
Jobs：{jobs 條列}

### 路由
共 {routes 總數} 支

### 功能模組
{features 條列，含 module、description、status}

### 進行中
{in_progress.feature}
上次做到：{in_progress.last_action}

### 待辦
{next 條列}

### 未提交變更
{uncommitted 條列，若無則顯示「工作區乾淨」}

### 注意事項
{notes，若無則略過}
```

---

## project-status.yml 格式規範

**唯一合法格式，不得新增規範外欄位（如 mode、api_status_file、files 等）。**

```yaml
last_updated: 2026-04-28T14:32:00
project:
  name: my-app
  stack: Laravel 11 + Vue 3
  language: PHP 8.2
  root: /var/www/html
  env:
    APP_NAME: MyApp
    DB_CONNECTION: mysql
    QUEUE_CONNECTION: redis
    CACHE_DRIVER: redis
  frontend:
    framework: Vue 3
    path: resources/js

structure:
  models: [User, Article, Comment]
  controllers: [AuthController, ArticleController]
  services: [ArticleService, AuthService]
  jobs: [SendEmailJob, ProcessImageJob]
  events: [ArticleCreated, UserRegistered]
  enums: [ArticleStatus, UserRole]
  traits: [HasTimestamps, Uploadable]
  middlewares: [EnsureAdmin, EnsureEmailVerified]

routes:
  - method: POST
    path: /api/auth/login
    middleware: [throttle:5,1]
    handler: AuthController@login
  - method: GET
    path: /api/articles
    middleware: [auth:sanctum]
    handler: ArticleController@index

dependencies:
  backend: [laravel/sanctum, spatie/laravel-permission]
  frontend: [vue, axios, tailwindcss]

features:
  - module: Auth
    description: 使用者認證，含 OAuth2 登入、token refresh
    related: [AuthController, AuthService]
    status: complete
  - module: Article
    description: 文章 CRUD，含草稿、發布、軟刪除
    related: [ArticleController, ArticleService]
    status: in_progress

api_status:
  submodule_path: ""

branch: main
uncommitted: []
in_progress:
  feature: ""
  last_action: ""
next: []
key_files: []
notes: ""
```

| 欄位 | 必填 | 說明 |
|------|------|------|
| `last_updated` | ✅ | ISO 8601 格式 |
| `project.name` | ✅ | 專案名稱 |
| `project.stack` | ✅ | 技術棧 |
| `project.language` | ✅ | 主要語言與版本 |
| `project.root` | ✅ | 專案根目錄絕對路徑 |
| `project.env` | ✅ | 非敏感環境設定 |
| `project.frontend` | 選填 | 有前端時填入 |
| `structure` | ✅ | 各模組 class 名稱清單 |
| `routes` | ✅ | 完整路由清單 |
| `dependencies` | ✅ | backend、frontend 套件 |
| `features` | ✅ | 功能模組描述與狀態 |
| `api_status.submodule_path` | ✅ | submodule 路徑；local 模式為空字串 |
| `branch` | ✅ | 當前 git 分支 |
| `uncommitted` | ✅ | 未提交檔案，無則 `[]` |
| `in_progress` | ✅ | 進行中功能與上次動作 |
| `next` | ✅ | 下次待辦，條列 |
| `key_files` | ✅ | 本次最相關的檔案 |
| `notes` | 選填 | 重要提醒、決策紀錄 |

---

## dev-context.yml 格式規範

**唯一合法格式，不帶日期，不併入 `project-status.yml`，進 git。**

```yaml
decisions:
  - topic: Token 儲存方式
    decision: 使用 httpOnly cookie，不走 localStorage
    reason: 避免 XSS 攻擊
    related: [AuthController, EnsureTokenValid]

  - topic: CORS 設定
    decision: config/cors.php 加入 supports_credentials
    reason: Sanctum 在 subdomain 需要帶 cookie
    related: [config/cors.php]

problems:
  - topic: Sanctum CORS 在 subdomain 失效
    solution: 調整 config/cors.php，加入 supports_credentials: true
    related: [config/cors.php]
```

| 欄位 | 說明 |
|------|------|
| `decisions.topic` | 決策主題（合併依據） |
| `decisions.decision` | 決策內容 |
| `decisions.reason` | 決策原因 |
| `decisions.related` | 相關檔案或 class |
| `problems.topic` | 問題描述 |
| `problems.solution` | 解決方式 |
| `problems.related` | 相關檔案或 class |

---

## git 說明

`project-status.yml` 與 `dev-context.yml` 皆進 git。
兩份檔案不含敏感資訊（`.env` 只讀非敏感設定，不含密碼或 API key）。
`notes` 欄位請避免填入任何敏感資訊。

---

## 輸出語言

所有輸出一律使用**繁體中文**。
yml 的 value 中英混用，以精簡為主。
