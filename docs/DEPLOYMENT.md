# Handoff Document - Ready for Deployment

**日付**：2026-02-08  
**プロジェクト**：AWS Lambda + SQS + API Gateway + Bedrock + S3 統合  
**ステータス**：✓ Phase 1.1 実装完了 / デプロイ準備完了

---

## 1. 完了した工程

### ✓ Planner
- 要件定義書作成・更新（docs/REQUIREMENTS.md）
- Phase 1.1：API Gateway 統合要件追加

### ✓ Architect  
- CloudFormation テンプレート設計（docs/ARCHITECTURE.md）
- API Gateway リソース設計追加
- リソース命名規則・パラメータ化戦略定義
- IAM 権限設計・Output 定義

### ✓ Coder
- Lambda 関数実装（Phase 1：メッセージ表示）
- API Gateway 統合実装（Phase 1.1：HTTP リクエスト対応）
- CloudFormation テンプレート実装・更新（template.yaml）
- イベント形式の判定・分岐処理実装

### ✓ Reviewer
- Lambda コード品質確認 → **PASS**
- API Gateway 統合コード確認 → **PASS**
- CloudFormation 構文・ベストプラクティス確認 → **PASS**
- Parameter/Output 設計確認 → **PASS**

### ✓ Security
- IAM 権限最小化確認 → **PASS**
- S3 セキュリティ設定確認 → **PASS**
- API Gateway アクセス設定確認 → **PASS**
- Bedrock アクセス権限確認 → **PASS**
- ログ・監査設定確認 → **PASS**

---

## 2. 成果物一覧

| ファイル | 説明 | 状態 |
|---------|------|------|
| [docs/REQUIREMENTS.md](../docs/REQUIREMENTS.md) | 要件定義書 | ✓ 完成 |
| [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) | テンプレート設計書 | ✓ 完成 |
| [docs/api/openapi.yaml](../docs/api/openapi.yaml) | OpenAPI 3.0.0 仕様書 | ✓ 完成 |
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
├── Resources（13個）
│   ├── SQS Queue
│   ├── SQS DLQ
│   ├── S3 Bucket
│   ├── IAM Role + Policies（3個）
│   ├── Lambda Function（インラインコード）
│   ├── Event Source Mapping
│   ├── CloudWatch Log Group
│   ├── API Gateway REST API
│   ├── API Gateway Resource (/v1/agents)
│   ├── API Gateway Method (POST)
│   ├── API Gateway Deployment
│   ├── API Gateway Stage
│   └── Lambda Permission（API Gateway）
│
└── Outputs（9個）
    ├── QueueURL
    ├── QueueArn
    ├── BucketName
    ├── LambdaFunctionArn
    ├── LambdaFunctionName
    ├── RoleArn
    ├── LogGroupName
    ├── ApiEndpoint
    └── ApiId
```

---

## 2a. API Gateway 統合仕様（フェーズ1.1）

### 2a.1 エンドポイント

```
https://{API_ID}.execute-api.ap-northeast-1.amazonaws.com/{Stage}/v1/agents
```

**パラメータ**：
- `{API_ID}`：CloudFormation Stack のOutput `ApiId` から取得
- `{Stage}`：Environment パラメータ（デフォルト: `dev`）

### 2a.2 リクエスト形式

**HTTP メソッド**: `POST`

**Content-Type**: `application/json`

**リクエストボディ**:
```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "prompt": "AWSとは何ですか？",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0"
}
```

**フィールド説明**:
- `request_id`（必須）: リクエストの一意識別子（UUID）
- `prompt`（必須）: 処理対象のプロンプト・質問
- `model_id`（オプション）: Bedrock モデルID（省略時のデフォルト: Claude 3 Sonnet）

### 2a.3 レスポンス形式

**成功時（HTTP 200）**:
```json
{
  "message": "Message processed successfully via API Gateway",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-02-08T12:34:56.789000"
}
```

**バリデーションエラー（HTTP 400）**:
```json
{
  "error": "Missing required fields: request_id, prompt"
}
```

**サーバーエラー（HTTP 500）**:
```json
{
  "error": "Internal server error"
}
```

### 2a.4 使用例

**cURL を使用**:
```bash
API_ENDPOINT="https://{API_ID}.execute-api.ap-northeast-1.amazonaws.com/dev/v1/agents"

curl -X POST "$API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -d '{
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "prompt": "AWSとは何ですか？"
  }'
```

**Python を使用**:
```python
import requests
import json
from uuid import uuid4

api_endpoint = "https://{API_ID}.execute-api.ap-northeast-1.amazonaws.com/dev/v1/agents"

payload = {
    "request_id": str(uuid4()),
    "prompt": "AWSとは何ですか？",
    "model_id": "anthropic.claude-3-sonnet-20240229-v1:0"
}

response = requests.post(api_endpoint, json=payload)
print(json.dumps(response.json(), indent=2))
```

**AWS CLI を使用**:
```bash
API_ENDPOINT="https://{API_ID}.execute-api.ap-northeast-1.amazonaws.com/dev/v1/agents"

aws apigateway test-invoke-method \
  --rest-api-id {API_ID} \
  --resource-id {RESOURCE_ID} \
  --http-method POST \
  --body '{
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "prompt": "AWSとは何ですか？"
  }' \
  --region ap-northeast-1
```

### 2a.5 CloudFormation Stack 出力から API エンドポイントを取得

```bash
# Stack 出力の確認
aws cloudformation describe-stacks \
  --stack-name lambda-agents-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`]' \
  --region ap-northeast-1

# 期待される出力
[
    {
        "OutputKey": "ApiEndpoint",
        "OutputValue": "https://abcd1234.execute-api.ap-northeast-1.amazonaws.com/dev/v1/agents"
    }
]
```

### 2a.6 OpenAPI 仕様の確認

詳細な API 仕様は [docs/api/openapi.yaml](../docs/api/openapi.yaml) を参照してください。

#### OpenAPI ファイルの構成

```yaml
openapi: 3.0.0
info:
  title: Lambda Agents API
  version: 1.1.0
paths:
  /v1/agents:
    post:
      # リクエスト・レスポンス仕様
      requestBody: ...
      responses:
        '200': ...
        '400': ...
        '500': ...
components:
  schemas:
    AgentRequest: ...     # リクエストスキーマ
    SuccessResponse: ...  # 成功レスポンス
    ErrorResponse: ...    # エラーレスポンス
```

#### Swagger UI でのドキュメント閲覧（オプション）

オンラインの Swagger Editor で OpenAPI 仕様を確認：

1. [Swagger Editor](https://editor.swagger.io/) を開く
2. File → Import URL で以下を指定：
   ```
   https://raw.githubusercontent.com/{username}/{repo}/main/docs/api/openapi.yaml
   ```
   または、ローカルファイルをコピー・ペースト

#### JSON スキーマの検証

リクエストボディが OpenAPI スキーマに準拠しているか確認する場合：

```python
import json
import yaml
from jsonschema import validate, ValidationError

# OpenAPI ファイルを読み込み
with open('docs/api/openapi.yaml', 'r', encoding='utf-8') as f:
    openapi_spec = yaml.safe_load(f)

# AgentRequest スキーマを取得
agent_request_schema = openapi_spec['components']['schemas']['AgentRequest']

# リクエストボディをテスト
test_request = {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "prompt": "AWSとは何ですか？"
}

try:
    validate(instance=test_request, schema=agent_request_schema)
    print("✓ リクエストは有効です")
except ValidationError as e:
    print(f"✗ バリデーションエラー: {e.message}")
```

---

## 3. AWS CLI セットアップ（前提条件）

### 3.1 AWS CLI インストール（Linux）

#### Ubuntu / Debian
```bash
# システムパッケージを更新
sudo apt-get update

# Python 3 と pip をインストール
sudo apt-get install -y python3 python3-pip

# AWS CLI v2 をインストール（推奨）
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# インストール確認
aws --version
```

#### Amazon Linux / CentOS / RHEL
```bash
# AWS CLI v2 をインストール
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# インストール確認
aws --version
```

### 3.2 AWS 認証情報の設定

#### 方法1: インタラクティブ設定（推奨）
```bash
aws configure
```

以下の情報を入力：
```
AWS Access Key ID [None]: <YOUR_ACCESS_KEY_ID>
AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
Default region name [None]: ap-northeast-1
Default output format [None]: json
```

#### 方法2: 環境変数を使用
```bash
export AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>
export AWS_DEFAULT_REGION=ap-northeast-1
```

#### 方法3: IAM ロール（EC2 から実行する場合）
```bash
# EC2 インスタンスプロファイルで自動認証
# AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY は不要
```

### 3.3 認証情報の確認

```bash
# 現在の認証ユーザーを確認
aws sts get-caller-identity
```

**期待される出力**：
```json
{
    "UserId": "AIDAI...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

### 3.4 CloudFormation 実行権限の確認

以下の権限が必要です：

**CloudFormation**
- cloudformation:CreateStack
- cloudformation:UpdateStack
- cloudformation:DeleteStack
- cloudformation:DescribeStacks

**SQS**
- sqs:CreateQueue
- sqs:DeleteQueue
- sqs:GetQueueAttributes
- sqs:SetQueueAttributes

**S3**
- s3:CreateBucket
- s3:DeleteBucket
- s3:GetBucketPolicy
- s3:PutBucketPolicy
- s3:PutPublicAccessBlock

**Lambda**
- lambda:CreateFunction
- lambda:DeleteFunction
- lambda:GetFunction
- lambda:UpdateFunctionCode
- lambda:CreateEventSourceMapping
- lambda:DeleteEventSourceMapping

**IAM**
- iam:CreateRole
- iam:DeleteRole
- iam:PutRolePolicy
- iam:DeleteRolePolicy
- iam:GetRole

**CloudWatch Logs**
- logs:CreateLogGroup
- logs:DeleteLogGroup
- logs:TagLogGroup

**確認コマンド**：
```bash
# ユーザーの権限を確認（IAM コンソール経由推奨）
# または以下を実行してアクセス可能を確認
aws s3 ls
aws sqs list-queues --region ap-northeast-1
aws lambda list-functions --region ap-northeast-1
```

---

## 4. デプロイ手順

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

#### 1a) Stack 更新（既存）
```bash
cd /Users/yo4taka/garage/repos/github/darthmazz/making-aws-lambda-for-agents

aws cloudformation update-stack \
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

**期待される出力**：`CREATE_COMPLETE` / `CREATE_IN_PROGRESS` / `UPDATE_COMPLETE` / `UPDATE_IN_PROGRESS`

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

## 5. 検証チェックリスト

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

## 6. Phase 2 準備事項（Bedrock 統合）

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

## 8. トラブルシューティング

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

## 9. サポート・問い合わせ

不明な点がある場合は、以下のドキュメントを参照してください：

- [AWS Lambda 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/)
- [Amazon SQS 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/sqs/)
- [Amazon Bedrock 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/)
- [CloudFormation 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/)

---

## 10. 次のステップ

1. **デプロイ実行**（このドキュメント内のコマンド実行）
2. **検証テスト**実施
   - SQS メッセージ送信・Lambda 実行確認
   - API Gateway エンドポイント呼び出し確認
3. **Phase 2 計画**：Bedrock 統合スケジュール策定
   - 認証・認可の実装（API Key / OAuth）
   - Bedrock API 呼び出し実装
   - S3 への結果保存実装
4. **Git コミット**：テンプレート・ドキュメントを main ブランチにマージ

---

**All Clear - Phase 1.1 Ready for Production Deployment ✓**
