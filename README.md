# AWS Lambda for Agents

Python 3.12 で AWS Lambda を構築し、SQS メッセージを受け取って Bedrock に問い合わせ、結果を S3 に保存するシステム。

---

## 概要

```
┌─────────────┐
│  SQS Queue  │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│  Lambda (Python  │
│     3.12)        │
└──────┬───────────┘
       │
       ├────────────────┐
       │                │
       ▼                ▼
┌────────────┐    ┌──────────┐
│  Bedrock   │    │   S3     │
│   API      │    │ (Results)│
└────────────┘    └──────────┘
```

### 特徴

- **IaC（Infrastructure as Code）**：CloudFormation で全リソース管理
- **セキュリティ第一**：IAM 権限最小化、S3 パブリックアクセスブロック
- **本番対応**：CloudWatch Logs（30日保持）、DLQ 対応
- **パラメータ化**：環境に応じた柔軟な設定

---

## クイックスタート

### 前提条件

- AWS アカウント
- AWS CLI がインストール済み
- AWS 認証情報が設定済み

### デプロイ

```bash
# リポジトリをクローン
git clone https://github.com/darthmazz/making-aws-lambda-for-agents.git
cd making-aws-lambda-for-agents

# Stack を作成
aws cloudformation create-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# 作成完了を確認
aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus'
```

詳細なデプロイ手順は [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) を参照してください。

---

## ファイル構成

```
.
├── README.md                    # このファイル
├── template.yaml                # CloudFormation テンプレート
└── docs/
    ├── REQUIREMENTS.md          # 要件定義書
    ├── ARCHITECTURE.md          # テンプレート設計書
    └── DEPLOYMENT.md            # デプロイ手順書
```

---

## ドキュメント

| ドキュメント | 対象者 | 内容 |
|------------|--------|------|
| [REQUIREMENTS.md](docs/REQUIREMENTS.md) | PM / 要件定義者 | 機能・インフラ要件 |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | アーキテクト / デザイナー | テンプレート設計 |
| [DEPLOYMENT.md](docs/DEPLOYMENT.md) | DevOps / 運用者 | デプロイ方法・トラブルシューティング |

---

## AWS リソース

### 自動作成されるリソース

| リソース | 名前 | 用途 |
|---------|------|------|
| **SQS Queue** | `lambda-agents-dev-queue` | メッセージキュー |
| **SQS DLQ** | `lambda-agents-dev-dlq` | Dead Letter Queue（エラー処理用） |
| **S3 Bucket** | `lambda-agents-dev-results-{ACCOUNT_ID}` | 結果保存先 |
| **Lambda Function** | `lambda-agents-dev-function` | メイン処理 |
| **IAM Role** | `lambda-agents-dev-role` | Lambda 実行権限 |
| **CloudWatch Log Group** | `/aws/lambda/lambda-agents-dev-function` | ログ保存（30日） |

### IAM 権限

Lambda 実行ロールに以下の権限を自動付与：

- `sqs:ReceiveMessage, DeleteMessage, GetQueueAttributes` → SQS キュー
- `s3:GetObject, PutObject, ListBucket` → S3 バケット
- `bedrock:InvokeModel` → Bedrock API
- `logs:CreateLogGroup, CreateLogStream, PutLogEvents` → CloudWatch Logs

---

## Phase 1 ステータス

### 実装済み

- [x] CloudFormation テンプレート
- [x] SQS → Lambda イベントマッピング
- [x] Lambda 関数（メッセージ表示のみ）
- [x] S3 バケット・IAM 権限
- [x] CloudWatch Logs 統合
- [x] 要件定義書・設計書・デプロイ手順書

### Phase 2 予定（Bedrock 統合）

- [ ] Bedrock API インテグレーション
- [ ] S3 への結果保存処理
- [ ] エラーハンドリング拡張
- [ ] ローカルテスト環境（SAM/Moto）

---

## 開発ワークフロー

本プロジェクトは以下のマルチエージェント構造で管理されます：

```
Planner (要件定義)
    ↓
Architect (設計)
    ↓
Coder (実装)
    ↓
Reviewer (コード品質確認)
    ↓
Security (セキュリティ確認)
    ↓
Deployment (デプロイ)
```

各ステップの詳細は [.github/agents/](/.github/agents/) を参照してください。

---

## トラブルシューティング

### Lambda が SQS メッセージを受け取らない

```bash
# Event Source Mapping の状態確認
aws lambda list-event-source-mappings \
  --function-name lambda-agents-dev-function \
  --region us-east-1
```

→ `State: Disabled` の場合は有効化してください

### S3 への書き込みが失敗

IAM ロールの S3 権限を確認：

```bash
aws iam get-role-policy \
  --role-name lambda-agents-dev-role \
  --policy-name S3Access
```

### Bedrock API にアクセスできない

- 使用リージョンで Bedrock が利用可能か確認
- モデル ID が正しいか確認（[Bedrock ドキュメント](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/model-ids.html)）

---

## 料金見積もり

無料枠内で運用できますが、Bedrock 利用時は従量課金が発生します。

| サービス | 無料枠 | 費用 |
|---------|--------|------|
| Lambda | 100万リクエスト/月 | ✓ 無料 |
| SQS | 100万リクエスト/月 | ✓ 無料 |
| S3 | 5GB/月 | 少額 |
| CloudWatch Logs | 5GB/月 | 少額 |
| **Bedrock** | なし | **従量課金** |

**推奨**：AWS Cost Explorer で監視してください

---

## FAQ

**Q: Phase 1 ではなぜ Bedrock を呼び出さないのか？**  
A: 初期段階で機能を明確に分離し、SQS → Lambda パイプラインが正常に動作することを確認するため。

**Q: DLQ は使用されないが削除してもいいか？**  
A: Phase 2 のエラーハンドリング拡張に備えて保持推奨。削除する場合は `RedrivePolicy` の記述も削除してください。

**Q: 複数環境（staging/prod）に対応できるか？**  
A: はい。`Environment` パラメータを変更して Stack を複製できます。

**Q: ローカルで Lambda をテストできるか？**  
A: Phase 2 で SAM CLI を導入予定。現在はクラウド環境でのテストを推奨。

---

## サポート・ドキュメント

- [AWS Lambda ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/)
- [Amazon SQS ドキュメント](https://docs.aws.amazon.com/ja_jp/sqs/)
- [Amazon Bedrock ドキュメント](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/)
- [CloudFormation ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/)

---

## ライセンス

MIT License

---

## 作成者

GitHub Copilot (Orchestrator Agent)

**作成日**：2026-02-07  
**バージョン**：0.1
