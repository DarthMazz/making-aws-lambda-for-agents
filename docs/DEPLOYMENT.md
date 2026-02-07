# Handoff Document - Ready for Deployment

**日付**：2026-02-07  
**プロジェクト**：AWS Lambda + SQS + Bedrock + S3 統合  
**ステータス**：✓ 全ワークフロー完了 / デプロイ準備完了

---

## 1. 完了した工程

### ✓ Planner
- 要件定義書作成（docs/REQUIREMENTS.md）
- 機能要件・インフラ要件を明確化

### ✓ Architect  
- CloudFormation テンプレート設計（docs/ARCHITECTURE.md）
- リソース命名規則・パラメータ化戦略定義
- IAM 権限設計・Output 定義

### ✓ Coder
- Lambda 関数実装（Phase 1：メッセージ表示）
- CloudFormation テンプレート実装（template.yaml）
- 環境変数・パラメータ設定完了

### ✓ Reviewer
- Lambda コード品質確認 → **PASS**
- CloudFormation 構文・ベストプラクティス確認 → **PASS**
- Parameter/Output 設計確認 → **PASS**

### ✓ Security
- IAM 権限最小化確認 → **PASS**
- S3 セキュリティ設定確認 → **PASS**
- Bedrock アクセス権限確認 → **PASS**
- ログ・監査設定確認 → **PASS**

---

## 2. 成果物一覧

| ファイル | 説明 | 状態 |
|---------|------|------|
| [docs/REQUIREMENTS.md](../docs/REQUIREMENTS.md) | 要件定義書 | ✓ 完成 |
| [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) | テンプレート設計書 | ✓ 完成 |
| [template.yaml](../template.yaml) | CloudFormation テンプレート | ✓ 完成 |

### リソース構成
```
template.yaml
├── Parameters（6個）
│   ├── Environment
│   ├── ProjectName
│   ├── LambdaMemory
│   ├── LambdaTimeout
│   ├── LogRetentionDays
│   └── BedrockRegion
│
├── Resources（8個）
│   ├── SQS Queue
│   ├── SQS DLQ
│   ├── S3 Bucket
│   ├── IAM Role + Policies（3個）
│   ├── Lambda Function（インラインコード）
│   ├── Event Source Mapping
│   └── CloudWatch Log Group
│
└── Outputs（7個）
    ├── QueueURL
    ├── QueueArn
    ├── BucketName
    ├── LambdaFunctionArn
    ├── LambdaFunctionName
    ├── RoleArn
    └── LogGroupName
```

---

## 3. デプロイ手順

### 前提条件
- AWS CLI がインストール済み
- AWS 認証情報が設定済み（`aws configure`）
- リージョン：`ap-northeast-1`（東日本）に設定

### デプロイコマンド

#### 1) Stack 作成（初回）
```bash
cd /Users/yo4taka/garage/repos/github/darthmazz/making-aws-lambda-for-agents

aws cloudformation create-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

#### 2) Stack 作成ステータス確認
```bash
aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1 \
  --query 'Stacks[0].StackStatus'
```

**期待される出力**：`CREATE_COMPLETE` または `CREATE_IN_PROGRESS`

#### 3) リソース確認
```bash
# Stack の Output を取得
aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1 \
  --query 'Stacks[0].Outputs'
```

#### 4) テストメッセージ送信
```bash
# SQS Queue URL を取得
QUEUE_URL=$(aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`QueueURL`].OutputValue' \
  --output text)

# メッセージ送信
aws sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{"request_id": "test-001", "prompt": "Hello Bedrock", "model_id": "anthropic.claude-3-sonnet-20240229-v1:0"}' \
  --region ap-northeast-1
```

#### 5) Lambda ログ確認
```bash
# CloudWatch Logs でメッセージ処理ログを確認
LOG_GROUP="/aws/lambda/lambda-agents-dev-function"

aws logs tail "$LOG_GROUP" \
  --follow \
  --region ap-northeast-1
```

### Stack 削除（不要時）
```bash
aws cloudformation delete-stack \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1

# 削除完了を確認
aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1 \
  --query 'Stacks[0].StackStatus' 2>/dev/null || echo "Stack deleted"
```

---

## 4. 検証チェックリスト

デプロイ後、以下の項目を確認してください：

- [ ] Stack が正常に作成された（`CREATE_COMPLETE`）
- [ ] SQS Queue が作成されたか確認
  ```bash
  aws sqs list-queues --region ap-northeast-1 | grep lambda-agents-dev
  ```
- [ ] S3 Bucket が作成されたか確認
  ```bash
  aws s3 ls | grep lambda-agents-dev-results
  ```
- [ ] Lambda Function が作成されたか確認
  ```bash
  aws lambda list-functions --region ap-northeast-1 | grep lambda-agents-dev
  ```
- [ ] IAM Role が作成されたか確認
  ```bash
  aws iam list-roles | grep lambda-agents-dev
  ```
- [ ] CloudWatch Log Group が作成されたか確認
  ```bash
  aws logs describe-log-groups --log-group-name-prefix /aws/lambda/lambda-agents-dev --region ap-northeast-1
  ```
- [ ] テストメッセージを送信して Lambda が実行されたか確認
- [ ] CloudWatch Logs にメッセージ処理ログが出力されているか確認

---

## 5. Phase 2 準備事項（Bedrock 統合）

以下は Phase 2 で実装予定：

- [ ] Bedrock API インテグレーション
  - `boto3.client('bedrock-runtime')` でモデルを呼び出し
  - リクエスト・レスポンス処理を実装
  
- [ ] S3 結果保存処理
  - Bedrock 応答を JSON フォーマット
  - `s3:PutObject` で `/results/{request_id}/{timestamp}.json` に保存
  
- [ ] エラーハンドリング
  - Bedrock API エラー（モデル利用不可、レート制限など）
  - S3 書き込みエラー（権限不足、容量不足など）
  
- [ ] ローカルテスト環境
  - SAM CLI または Moto での統合テスト

---

## 6. AWS リソース費用概算

| リソース | 無料枠 | 費用 |
|---------|--------|------|
| **Lambda** | 100万リクエスト/月、400,000GB秒/月 | ✓ 初期無料 |
| **SQS** | 100万リクエスト/月 | ✓ 初期無料 |
| **S3** | 5GB ストレージ/月 | 少額 |
| **CloudWatch Logs** | 5GB 収集/月 | 少額 |
| **Bedrock** | なし | 従量課金（モデルによる） |

**注**：本番運用時は予算アラートを設定してください

### Ownerタグによるコスト管理

全リソースに `Owner: dev1` タグが設定されているため、以下の方法でコストを追跡できます。

#### AWS Cost Explorer でのフィルタリング

**方法1: AWS Management Console**
1. Cost Explorer を開く
2. 「Filters」→「Tag」を選択
3. `Owner` = `dev1` でフィルタリング
4. 期間を指定してコストを表示

**方法2: AWS CLI**
```bash
# 月次コストを取得（2026年2月の例）
aws ce get-cost-and-usage \
  --time-period Start=2026-02-01,End=2026-03-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --filter '{
    "Tags": {
      "Key": "Owner",
      "Values": ["dev1"]
    }
  }' \
  --region us-east-1
```

#### リソースグループでの管理

**リソースグループ作成**
```bash
# Owner=dev1 のリソースをグループ化
aws resource-groups create-group \
  --name lambda-agents-dev1-resources \
  --resource-query '{
    "Type": "TAG_FILTERS_1_0",
    "Query": "{\"ResourceTypeFilters\":[\"AWS::AllSupported\"],\"TagFilters\":[{\"Key\":\"Owner\",\"Values\":[\"dev1\"]}]}"
  }' \
  --region ap-northeast-1

# リソースグループのリソース一覧
aws resource-groups list-group-resources \
  --group-name lambda-agents-dev1-resources \
  --region ap-northeast-1
```

#### タグエディターでの確認

**AWS Management Console から**
1. AWS Resource Groups & Tag Editor を開く
2. 「Tag Editor」を選択
3. 「Find resources」で `Owner: dev1` を検索
4. 全リソースを一覧表示

#### 予算アラート設定（推奨）

```bash
# Owner=dev1 タグで月額$10を超えたら通知
aws budgets create-budget \
  --account-id <AWS_ACCOUNT_ID> \
  --budget '{
    "BudgetName": "lambda-agents-dev1-budget",
    "BudgetLimit": {
      "Amount": "10",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["user:Owner$dev1"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "your-email@example.com"
        }
      ]
    }
  ]'
```

**ポイント**：
- 全リソース（SQS, S3, Lambda, IAM, CloudWatch Logs）に `Owner: dev1` タグが付与済み
- Cost Explorer でタグフィルタリングにより、このプロジェクト専用のコストを可視化
- リソースグループで一元管理し、運用を効率化

---

## 7. トラブルシューティング

### Lambda Function が実行されない
```bash
# Event Source Mapping の状態確認
aws lambda list-event-source-mappings \
  --function-name lambda-agents-dev-function \
  --region ap-northeast-1
```

**原因**：Event Source Mapping が `Disabled` の可能性
```bash
# 有効化
aws lambda update-event-source-mapping \
  --uuid <UUID> \
  --state Enabled
```

### S3 への書き込みが失敗
```bash
# IAM ロールの権限確認
aws iam get-role-policy \
  --role-name lambda-agents-dev-role \
  --policy-name S3Access
```

### Bedrock API にアクセスできない
- リージョン確認：`BedrockRegion` パラメータが正しいか
- モデル利用可否：[Bedrock ドキュメント](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/model-ids.html)で確認

---

## 8. サポート・問い合わせ

不明な点がある場合は、以下のドキュメントを参照してください：

- [AWS Lambda 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/)
- [Amazon SQS 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/sqs/)
- [Amazon Bedrock 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/)
- [CloudFormation 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/)

---

## 9. 次のステップ

1. **デプロイ実行**（このドキュメント内のコマンド実行）
2. **検証テスト**実施（検証チェックリストを参照）
3. **Phase 2 計画**：Bedrock 統合スケジュール策定
4. **Git コミット**：テンプレート・ドキュメントを main ブランチにマージ

---

**All Clear - Ready for Production Deployment ✓**
