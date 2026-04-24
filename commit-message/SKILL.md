---
name: commit-message
description: 自動依據 git diff 產生 commit message 與完整 git 指令。
  使用時機：使用者說「幫我寫 commit」、「產生 commit message」、
  「commit 這些變更」時，一定要讀取這個 skill。
  也適用於開發流程中完成某個功能、修 bug、重構後需要提交的時機。
---

# Commit Message 產生 Skill

## 執行步驟

1. 執行 `git diff --staged` 取得已暫存的變更
2. 若暫存區為空，改執行 `git diff HEAD` 取得未暫存變更
3. 分析變更內容，判斷變更類型與影響範圍
4. 判斷是否需要拆分（見下方拆分邏輯）
5. 直接輸出完整 git 指令，不詢問使用者

## 輸出原則

- 只輸出指令，不自動執行任何 git 操作
- 使用者複製指令後自行決定何時執行
- 遇到需要人工判斷的情況（如 `git add -p`）才說明原因

## Commit Message 格式（Conventional Commits）

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

### Type 對照表

| type | 使用時機 |
|------|----------|
| feat     | 新增功能 |
| fix      | 修正 bug |
| refactor | 重構，不影響功能 |
| docs     | 文件變更 |
| style    | 格式調整，不影響邏輯 |
| test     | 新增或修改測試 |
| chore    | 建置流程、依賴套件更新 |
| perf     | 效能優化 |

### 規則

- subject 用祈使句，英文開頭小寫，不加句號
- subject 不超過 72 字元
- 若變更跨多個模組，body 條列每個模組的變更摘要
- breaking change 在 footer 加上 `BREAKING CHANGE: 說明`

## 拆分邏輯

### 判斷是否需要拆分

當變更同時包含以下任兩種以上，應拆分：
- 不同 type（例如 feat + fix）
- 不同 scope（例如 auth 模組 + db 模組）
- 不相關的檔案群組

### 情況 A：每個檔案只屬於單一群組

直接輸出每個 commit 的完整指令：

```bash
# Commit 1
git add src/user/avatar.ts src/user/upload.ts
git commit -m "feat(user): add profile picture upload"

# Commit 2
git add db/migrations/001_users.sql
git commit -m "fix(db): correct migration script for users table"
```

### 情況 B：同一檔案內混有不相關的變更

該檔案需要 patch 模式，說明原因並給出指引：

```bash
# src/api/handler.ts 同時包含 feat 與 fix 的變更
# 請先執行以下指令，逐一選擇要納入 Commit 1 的 hunk：
git add -p src/api/handler.ts

# 完成後執行：
git commit -m "feat(api): add user fetch endpoint"

# 再次執行以下指令，選擇剩餘的 hunk：
git add -p src/api/handler.ts

# 完成後執行：
git commit -m "fix(api): handle null response from gateway"
```

## 單一 commit 範例輸出

```bash
git add .
git commit -m "feat(auth): add OAuth2 login with Google

- add GoogleOAuthProvider component
- implement token exchange endpoint
- store refresh token in httpOnly cookie"
```

```bash
git add .
git commit -m "fix(api): handle null response from payment gateway

Prevent crash when gateway returns empty body on timeout."
```