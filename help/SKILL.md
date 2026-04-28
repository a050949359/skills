---
name: help
description: >
  指令提示 skill。輸入「指令」時啟用，列出所有可用指令與說明。
---

# Help Skill

## 用途

列出所有可用指令與簡短說明，方便使用者快速查閱。

---

## 輸出格式

```
## 可用指令

### 專案管理
| 指令 | 說明 |
|------|------|
| ps | 已有設定時直接恢復專案狀態 |
| ps local | 第一次使用，相關檔案放專案根目錄 |
| ps .gitmodules | 第一次使用，使用 submodule 管理相關檔案 |
| ps close | 功能完成，更新 project-status.yml |
| ps save | 開發中斷，記錄當前進度 |
| ps log | 記錄技術決策與問題解法至 dev-context.yml |

### API 狀態
| 指令 | 說明 |
|------|------|
| as | 確認所有 API 開發狀態與安全機制覆蓋狀況 |

### 程式碼品質
| 指令 | 說明 |
|------|------|
| cr | 審查程式碼是否符合 Laravel 安全規範 |
| cicd | 測試撰寫規範、GitHub Actions、PR Checklist |

### Git
| 指令 | 說明 |
|------|------|
| cm | 依據 git diff 產生 Conventional Commits 格式的 commit 指令 |

### 其他
| 指令 | 說明 |
|------|------|
| 指令 | 顯示此說明 |
```

---

## 輸出語言

所有輸出一律使用**繁體中文**。
