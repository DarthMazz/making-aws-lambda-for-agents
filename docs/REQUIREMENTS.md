# AWS Lambda + SQS + Bedrock 統合 要件定義書

**プロジェクト名**：AWS Lambda for Agents  
**バージョン**：0.1  
**作成日**：2026-02-07  
**ステータス**：初期構想段階

---

## 1. 概要

AWS Lambda（Python 3.12）で SQS メッセージを受け取り、Bedrock に問い合わせた結果を S3 に保存するシステム。

---

## 2. 機能要件

### 2.1 Lambda 処理フロー
```
SQS → Lambda → Bedrock API → S3 (結果保存)
```

### 2.2 処理内容
- **フェーズ1（初期）**：SQS メッセージ内容の表示（ログ出力）
- **フェーズ1.1（API統合）**：API Gateway 経由での直接呼び出しに対応
- **フェーズ2以降**：Bedrock へのコール実装

### 2.3 メモリ・タイムアウト設定
- **メモリ**：512 MB（提案値。Bedrock コール時に調整可能）
- **タイムアウト**：60 秒（提案値。処理内容に応じて調整）

---

## 3a. API Gateway 連携仕様（フェーズ1.1）

| 項目 | 値 |
|------|-----|
| API タイプ | REST API |
| エンドポイント | `/v1/agents` |
| HTTPメソッド | POST |
| リージョン | ap-northeast-1（東日本） |
| 認証 | なし（Phase 1.1） |
| OpenAPI 仕様 | `docs/api/openapi.yaml` を参照 |

### 3a.1 リクエスト仕様

**Content-Type**: `application/json`

**リクエストボディ**:
```json
{
  "request_id": "uuid",
  "prompt": "質問内容",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0"
}
```

**必須フィールド**:
- `request_id`: リクエストの一意識別子
- `prompt`: 処理対象の質問・プロンプト

**オプションフィールド**:
- `model_id`: Bedrock モデルID（デフォルト: Claude 3 Sonnet）

### 3a.2 レスポンス仕様

**成功時（200）**:
```json
{
  "message": "Message processed successfully via API Gateway",
  "request_id": "uuid",
  "timestamp": "2026-02-08T00:00:00.000000"
}
```

**エラー時（400/500）**:
```json
{
  "error": "エラーメッセージ"
}
```

### 3a.3 仕様例

**リクエスト例**:
```bash
curl -X POST \
  https://{APIId}.execute-api.ap-northeast-1.amazonaws.com/dev/v1/agents \
  -H "Content-Type: application/json" \
  -d '{
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "prompt": "AWSとは何ですか？"
  }'
```

---

## 3. SQS 連携仕様

| 項目 | 値 |
|------|-----|
| キューの数 | 1 個（単一） |
| メッセージ形式 | JSON |
| バッチ処理 | 1 件ずつ（バッチサイズ = 1） |
| Dead Letter Queue（DLQ） | 不要 |
| 最大受信数 | 3（デフォルト） |
| メッセージ保持期間 | 4 日（デフォルト） |

### 3.1 メッセージスキーマ（例）
```json
{
  "request_id": "uuid",
  "prompt": "質問内容",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0"
}
```

---

## 4. S3 連携仕様

| 項目 | 値 |
|------|-----|
| バケット名 | `lambda-agents-dev-results` |
| 用途 | Bedrock 応答結果の保存 |
| 保存パス | `s3://lambda-agents-dev-results/{request_id}/{timestamp}.json` |
| ファイル形式 | JSON |

### 4.1 必要な権限
- `s3:GetObject`（リード）
- `s3:PutObject`（ライト）
- `s3:ListBucket`（リスト表示）

---

## 5. Bedrock 連携仕様

| 項目 | 値 |
|------|-----|
| AWS リージョン | ap-northeast-1（東日本） |
| モデル | Claude 3 Sonnet （デフォルト）※リクエストで指定可能 |
| 必要な権限 | `bedrock:InvokeModel` |

---

## 6. IAM 権限定義

Lambda 実行ロール（`lambda-agents-dev-role`）に以下の権限を付与：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:*:*:lambda-agents-dev-queue"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::lambda-agents-dev-results",
        "arn:aws:s3:::lambda-agents-dev-results/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "arn:aws:bedrock:*:*:foundation-model/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

---

## 7. インフラストラクチャ要件

| 項目 | 値 |
|------|-----|
| 環境 | dev |
| VPC | 不要（パブリック実行） |
| CloudWatch Logs 保持期間 | 30 日 |
| IaC ツール | CloudFormation（YAML） |
| Lambda ランタイム | Python 3.12 |

---

## 8. デプロイ方式

### 8.1 CloudFormation テンプレート構成
- **ファイル名**：`template.yaml`
- **リソース**：
  - SQS Queue
  - S3 Bucket
  - IAM Role
  - Lambda Function
  - Event Source Mapping（SQS → Lambda）
  - CloudWatch Log Group

### 8.2 デプロイコマンド（案）
```bash
aws cloudformation create-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

---

## 9. 除外項目

- ローカル開発環境（SAM/LocalStack）
- ユニットテスト・統合テスト
- 本番環境（prod）
- ~~API Gateway / 外部エンドポイント~~ → Phase 1.1 で実装
- 認証・認可（Phase 2 で実装予定）
- VPC / NAT Gateway

---

## 10. 次のステップ

1. **Phase 1.1（現在）**
   - Architect：API Gateway テンプレート設計完了
   - Coder：Lambda 関数 + template.yaml 実装完了
   - Reviewer：コード・テンプレート確認
   - Security：IAM 権限・API アクセスの確認

2. **Phase 2**
   - Bedrock へのコール実装
   - S3 への結果保存
   - 認証・認可の追加（API Key / OAuth）

---

## 11. 用語定義

| 用語 | 定義 |
|------|-----|
| **TBD** | 後で決定予定 |
| **Event Source Mapping** | SQS イベントを Lambda にマッピング |
| **Bedrock** | AWS の生成 AI サービス |
| **CloudFormation Stack** | AWS リソース管理の単位 |
