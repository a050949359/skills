---
name: laravel-ci-cd
description: >
  Laravel CI/CD 規範 skill。輸入「cicd」時啟用。
  涵蓋測試撰寫規範、GitHub Actions workflow、PHPStan 設定、PR Checklist。
---

# Laravel CI/CD Skill

## 用途

提供 Laravel 專案的測試撰寫規範與 GitHub Actions 自動化流程設定，
確保每次 PR 都經過安全與品質檢查。

---

## 測試撰寫規範

### Feature Test 必要覆蓋項目

每支重要 API 的 Feature Test 必須涵蓋以下四個面向：

| 面向 | 測試內容 |
|------|----------|
| 權限邊界 | 未登入應得 401，非擁有者應得 403 |
| 輸入驗證 | 必填欄位缺漏、格式錯誤應得 422 |
| 資料正確性 | 回傳資料與資料庫一致 |
| 效能 | 使用 `assertQueryCount()` 確認無 N+1 |

### 測試結構範例

```php
class ArticleApiTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function guest_cannot_create_article(): void
    {
        $response = $this->postJson('/api/articles', ['title' => 'Test']);
        $response->assertStatus(401);
    }

    /** @test */
    public function owner_can_update_own_article(): void
    {
        $user = User::factory()->create();
        $article = Article::factory()->for($user)->create();

        $this->actingAs($user)
            ->putJson("/api/articles/{$article->id}", ['title' => 'Updated'])
            ->assertOk()
            ->assertJsonPath('data.title', 'Updated');
    }

    /** @test */
    public function other_user_cannot_update_article(): void
    {
        $owner = User::factory()->create();
        $other = User::factory()->create();
        $article = Article::factory()->for($owner)->create();

        $this->actingAs($other)
            ->putJson("/api/articles/{$article->id}", ['title' => 'Hacked'])
            ->assertStatus(403);
    }

    /** @test */
    public function article_index_has_no_n_plus_one(): void
    {
        Article::factory()->count(10)->create();
        $this->actingAs(User::factory()->create());

        \DB::enableQueryLog();
        $this->getJson('/api/articles')->assertOk();
        $queryCount = count(\DB::getQueryLog());
        $this->assertLessThanOrEqual(3, $queryCount);
    }
}
```

---

## GitHub Actions 工作流程

```yaml
# .github/workflows/laravel.yml
name: Laravel CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: laravel_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, pdo, pdo_mysql
          coverage: xdebug

      - name: Install dependencies
        run: composer install --no-progress --prefer-dist

      - name: Copy .env
        run: cp .env.example .env.testing

      - name: Generate key
        run: php artisan key:generate --env=testing

      - name: Run migrations
        run: php artisan migrate --env=testing --force

      - name: PHPStan 靜態分析
        run: ./vendor/bin/phpstan analyse --level=5

      - name: Composer Unused 檢查
        run: ./vendor/bin/composer-unused

      - name: PHPUnit 測試
        run: ./vendor/bin/phpunit --coverage-text
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: laravel_test
          DB_USERNAME: root
          DB_PASSWORD: password
```

---

## 各工具設定

### PHPStan（phpstan.neon）

```neon
parameters:
  level: 5
  paths:
    - app
  excludePaths:
    - app/Console/Kernel.php
  ignoreErrors:
    - '#Unsafe usage of new static#'
```

推薦等級：開發初期 level 3，穩定後升至 level 5，目標 level 8。

### Composer Unused

```bash
composer require --dev icanhazstring/composer-unused
./vendor/bin/composer-unused
```

---

## PR Checklist

在 `.github/pull_request_template.md` 加入：

```markdown
## Security Checklist

- [ ] 新 API 路由已加上 `auth:sanctum`
- [ ] Controller 層有權限二次檢查（user_id / isAdmin）
- [ ] 輸入透過 FormRequest 驗證，含 NoMaliciousPattern
- [ ] Model 已定義 `$fillable`
- [ ] 無 N+1 查詢（已加 eager loading）
- [ ] 錯誤回傳不含 exception trace
- [ ] Feature Test 已覆蓋新功能的權限與驗證邊界
```

---

## 輸出語言

所有輸出一律使用**繁體中文**。
