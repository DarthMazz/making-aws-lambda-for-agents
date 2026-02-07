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
- **フェーズ2以降**：Bedrock へのコール実装

### 2.3 メモリ・タイムアウト設定
- **メモリ**：512 MB（提案値。Bedrock コール時に調整可能）
- **タイムアウト**：60 秒（提案値。処理内容に応じて調整）

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
- API Gateway / 外部エンドポイント
- VPC / NAT Gateway

---

## 10. 次のステップ

1. **Architect**：CloudFormation テンプレート設計
2. **Coder**：Lambda 関数実装 + template.yaml 作成
3. **Tester**：（不要）
4. **Reviewer**：コード・テンプレート確認
5. **Security**：IAM 権限・Bedrock アクセスの確認

---

## 11. 用語定義

| 用語 | 定義 |
|------|-----|
| **TBD** | 後で決定予定 |
| **Event Source Mapping** | SQS イベントを Lambda にマッピング |
| **Bedrock** | AWS の生成 AI サービス |
| **CloudFormation Stack** | AWS リソース管理の単位 |
