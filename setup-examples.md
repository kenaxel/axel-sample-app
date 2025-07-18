# 詳細セットアップガイド

## 1. Googleスプレッドシートの設定

### スプレッドシートの作成

1. Google Sheetsで新しいスプレッドシートを作成
2. シート名を「キーワード」に変更
3. 以下のヘッダーを設定：

```
A1: キーワード
B1: カテゴリ
C1: 処理日
```

### サンプルデータ

```
A2: WordPress プラグイン
B2: テクノロジー
C2: (空白)

A3: SEO対策 方法
B3: マーケティング
C3: (空白)

A4: 在宅ワーク 効率化
B4: ライフスタイル
C4: (空白)
```

### スプレッドシートIDの取得

URLから以下の部分を取得：
```
https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]/edit
```

## 2. Google Cloud Console設定

### プロジェクトの作成

1. [Google Cloud Console](https://console.cloud.google.com/)にアクセス
2. 新しいプロジェクトを作成
3. プロジェクト名: `n8n-seo-automation`

### API有効化

```bash
# Google Sheets API
gcloud services enable sheets.googleapis.com

# Google Drive API（オプション）
gcloud services enable drive.googleapis.com
```

### OAuth2認証情報の作成

1. 「認証情報」→「認証情報を作成」→「OAuth クライアント ID」
2. アプリケーションの種類: Webアプリケーション
3. 承認済みのリダイレクトURI: `https://your-n8n-instance.com/rest/oauth2-credential/callback`

## 3. OpenAI API設定

### APIキーの取得

1. [OpenAI Platform](https://platform.openai.com/)にログイン
2. 「API Keys」→「Create new secret key」
3. キーを安全に保存

### 使用量制限の設定

```json
{
  "monthly_limit": 100,
  "usage_alerts": true,
  "alert_threshold": 80
}
```

## 4. WordPress設定

### REST API有効化

`wp-config.php`に追加：

```php
// WordPress REST API有効化
add_filter('rest_enabled', '__return_true');

// 認証なしでのアクセスを制限
add_filter('rest_authentication_errors', function($result) {
    if (!empty($result)) {
        return $result;
    }
    if (!is_user_logged_in()) {
        return new WP_Error('rest_not_logged_in', 'You are not currently logged in.', array('status' => 401));
    }
    return $result;
});
```

### アプリケーションパスワードの生成

1. WordPress管理画面→「ユーザー」→「プロフィール」
2. 「アプリケーションパスワード」セクション
3. 新しいパスワードを生成：`n8n-automation`

### 必要な権限

ユーザーに以下の権限が必要：
- `edit_posts`
- `publish_posts`
- `upload_files`
- `edit_published_posts`

## 5. n8n設定例

### 環境変数

```bash
# .env ファイル
OPENAI_API_KEY=sk-your-openai-api-key
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
WORDPRESS_URL=https://yoursite.com
WORDPRESS_USERNAME=your-username
WORDPRESS_PASSWORD=your-app-password
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
```

### クレデンシャル設定

#### Google OAuth2

```json
{
  "clientId": "{{ $env.GOOGLE_CLIENT_ID }}",
  "clientSecret": "{{ $env.GOOGLE_CLIENT_SECRET }}",
  "scope": "https://www.googleapis.com/auth/spreadsheets"
}
```

#### OpenAI API

```json
{
  "apiKey": "{{ $env.OPENAI_API_KEY }}"
}
```

#### WordPress API

```json
{
  "url": "{{ $env.WORDPRESS_URL }}/wp-json/wp/v2/",
  "username": "{{ $env.WORDPRESS_USERNAME }}",
  "password": "{{ $env.WORDPRESS_PASSWORD }}"
}
```

#### SMTP設定

```json
{
  "host": "{{ $env.SMTP_HOST }}",
  "port": "{{ $env.SMTP_PORT }}",
  "secure": false,
  "auth": {
    "user": "{{ $env.SMTP_USER }}",
    "pass": "{{ $env.SMTP_PASS }}"
  }
}
```

## 6. 記事生成プロンプトの詳細設定

### システムプロンプト

```javascript
const systemPrompt = `あなたは経験豊富なSEOライターです。以下の要件に従って記事を作成してください：

## 記事の要件
1. タイトル：キーワードを含む魅力的なタイトル（32文字以内）
2. メタディスクリプション：検索結果に表示される説明文（160文字以内）
3. 構造化された見出し：H1, H2, H3を適切に使用
4. 本文：2000文字以上の詳細な内容
5. 内部リンク：関連記事への提案
6. 画像説明：メイン画像のalt属性用説明

## SEO要件
- キーワード密度：1-2%
- 読みやすさ：小学6年生レベル
- 構造化データ：FAQスキーマ対応
- 関連キーワード：LSIキーワードを含む

## 出力形式
必ずJSON形式で出力してください：
{
  "title": "記事タイトル",
  "meta_description": "メタディスクリプション",
  "content": "HTML形式の記事本文",
  "image_description": "メイン画像の説明",
  "tags": ["タグ1", "タグ2", "タグ3"],
  "internal_links": ["関連記事URL1", "関連記事URL2"],
  "faq": [
    {
      "question": "よくある質問1",
      "answer": "回答1"
    }
  ]
}`;
```

### ユーザープロンプト

```javascript
const userPrompt = `
キーワード: {{ $json.keyword }}
カテゴリ: {{ $json.category }}
対象読者: {{ $json.target_audience || '一般的なWebユーザー' }}
記事の目的: {{ $json.purpose || '情報提供・問題解決' }}

上記の情報に基づいて、検索エンジンで上位表示される高品質なSEO記事を作成してください。
`;
```

## 7. 画像生成プロンプトの詳細設定

### DALL-E 3プロンプト

```javascript
const imagePrompt = `
{{ $json.image_description }}

スタイル要件:
- 高品質でプロフェッショナル
- ブログ記事のメイン画像として適切
- 明るく魅力的な色合い
- 文字やテキストは含まない
- 16:9の横長比率
- 現代的でクリーンなデザイン
- {{ $json.category }}カテゴリに適したテーマ

技術仕様:
- 解像度: 1024x1024以上
- ファイル形式: PNG
- 色空間: sRGB
- 品質: 最高品質
`;
```

## 8. エラーハンドリングの設定

### 共通エラーハンドリング

```javascript
// エラーハンドリング用のコード
function handleError(error, context) {
  const errorInfo = {
    timestamp: new Date().toISOString(),
    context: context,
    error: error.message,
    stack: error.stack,
    input_data: $input.all()
  };
  
  // ログに記録
  console.error('Workflow Error:', errorInfo);
  
  // エラー通知用のデータを返す
  return [{
    json: {
      status: 'error',
      error: error.message,
      context: context,
      timestamp: errorInfo.timestamp
    }
  }];
}
```

### リトライ設定

```json
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000
}
```

## 9. 監視とログ設定

### ワークフロー実行ログ

```javascript
// 実行ログの設定
const executionLog = {
  workflow_id: '{{ $workflow.id }}',
  execution_id: '{{ $execution.id }}',
  start_time: new Date().toISOString(),
  input_data: $input.all(),
  node_name: '{{ $node.name }}',
  status: 'running'
};

console.log('Execution Log:', executionLog);
```

### 成功率の監視

```javascript
// 成功率計算用のコード
function calculateSuccessRate(executions) {
  const total = executions.length;
  const successful = executions.filter(e => e.status === 'success').length;
  return {
    total: total,
    successful: successful,
    success_rate: (successful / total * 100).toFixed(2) + '%'
  };
}
```

## 10. 運用開始前のチェックリスト

### 必須チェック項目

- [ ] Google Sheets APIの認証完了
- [ ] OpenAI APIキーの設定と動作確認
- [ ] WordPress REST APIの有効化
- [ ] SMTP設定の動作確認
- [ ] スプレッドシートの形式確認
- [ ] テスト実行の成功確認
- [ ] エラー通知の動作確認
- [ ] 料金制限の設定
- [ ] バックアップ設定
- [ ] 監視設定

### 推奨設定

- [ ] 実行時間の制限設定
- [ ] 同時実行数の制限
- [ ] リソース使用量の監視
- [ ] 定期的なログのクリーンアップ
- [ ] セキュリティ監査の実施