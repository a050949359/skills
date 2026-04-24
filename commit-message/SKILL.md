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
5. 依照輸出規則產生指令

## 輸出規則（強制）

- 所有 git 指令必須放在 ```bash code block 內
- 禁止在 code block 外單獨出現任何指令或檔案路徑
- 說明文字寫在 code block 上方，以 # 註解補充說明
- 不自動執行任何 git 操作，使用者自行複製執行

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
- body 條列項目使用 `-`，不用 `*`
- 若變更跨多個模組，body 條列每個模組的變更摘要
- 若無法明確判斷 scope，省略不寫，不要猜測
- breaking change 在 footer 加上 `BREAKING CHANGE: 說明`

## git add 規則

- 永遠列出明確的檔案路徑，包含從專案根目錄起的相對路徑
- 禁止使用 `git add .` 或 `git add -A`
- 從 git diff 的結果推斷哪些檔案屬於此次變更

## 拆分邏輯

### 判斷是否需要拆分

當變更同時包含以下任兩種以上，應拆分：
- 不同 type（例如 feat + fix）
- 不同 scope（例如 auth 模組 + db 模組）
- 不相關的檔案群組

### 情況 A：每個檔案只屬於單一群組

```bash
# Commit 1
git add src/user/avatar.ts src/user/upload.ts
git commit -m "feat(user): add profile picture upload"

# Commit 2
git add db/migrations/001_users.sql
git commit -m "fix(db): correct migration script for users table"
```

### 情況 B：同一檔案內混有不相關的變更

```bash
# src/api/handler.ts 同時包含 feat 與 fix 的變更
# 請先執行以下指令，逐一選擇要納入 Commit 1 的 hunk：
git add -p src/api/handler.ts

# 完成後執行：
git commit -m "feat(api): add user fetch endpoint"

# 再次執行，選擇剩餘的 hunk：
git add -p src/api/handler.ts

# 完成後執行：
git commit -m "fix(api): handle null response from gateway"
```

## 單一 commit 範例輸出

```bash
git add src/auth/GoogleOAuthProvider.tsx src/api/token.ts
git commit -m "feat(auth): add OAuth2 login with Google

- add GoogleOAuthProvider component
- implement token exchange endpoint
- store refresh token in httpOnly cookie"
```