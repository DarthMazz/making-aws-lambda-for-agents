# CloudFormation テンプレート設計書

**ドキュメント作成日**：2026-02-07  
**テンプレートファイル**：`template.yaml`  
**対象環境**：dev

---

## 1. 設計方針

### 1.1 ファイル構成
- **単一テンプレート戦略**：全リソースを `template.yaml` に集約
- **理由**：初期段階では複雑性が低く、単一ファイルで管理が容易

### 1.2 リソース命名規則
```
{ProjectName}-{Environment}-{ResourceType}

例：
- lambda-agents-dev-queue      （SQS）
- lambda-agents-dev-results    （S3）
- lambda-agents-dev-function   （Lambda）
- lambda-agents-dev-role       （IAM Role）
- lambda-agents-dev-dlq        （DLQ）
```

### 1.3 パラメータ化
CloudFormation の `Parameters` セクションで以下を外部化：

| パラメータ | デフォルト | 型 | 用途 |
|-----------|----------|-----|------|
| `Environment` | dev | String | 環境名（dev/staging/prod） |
| `ProjectName` | lambda-agents | String | リソース名プレフィックス |
| `LambdaMemory` | 512 | Number | Lambda メモリ |
| `LambdaTimeout` | 60 | Number | Lambda タイムアウト |
| `LogRetentionDays` | 30 | Number | CloudWatch ログ保持期間 |
| `BedrockRegion` | ap-northeast-1 | String | Bedrock API リージョン（東日本） |

---

## 2. リソース詳細設計

### 2.1 SQS Queue

#### 主要プロパティ
```yaml
QueueName: lambda-agents-dev-queue
VisibilityTimeout: 60秒（Lambda タイムアウトと同期）
MessageRetentionPeriod: 345600秒（4日）
BatchSize: 1（1件ずつ処理）
```

#### Dead Letter Queue（DLQ）
- **名前**：`lambda-agents-dev-dlq`
- **理由**：今後のエラーハンドリング拡張に対応
- **現在**：使用しない（`RedrivePolicy` は設定済み）

#### メモリ図
```
SQS Queue (lambda-agents-dev-queue)
  ↓ (batch size = 1)
Lambda Function
  ↓
Bedrock / S3
```

---

### 2.2 S3 Bucket

#### 主要プロパティ
```yaml
BucketName: lambda-agents-dev-results-{AWS::AccountId}
  # グローバル一意性のため AWS Account ID を付加

VersioningConfiguration: Suspended
  # 初期段階ではバージョン管理不要

PublicAccessBlockConfiguration: 全て有効
  # セキュリティのため パブリックアクセスをブロック
```

#### 保存仕様
- **パス例**：`s3://lambda-agents-dev-results-{ACCOUNT_ID}/{request_id}/{timestamp}.json`
- **ファイル形式**：JSON
- **アクセス権限**：Lambda ロールのみ

---

### 2.3 IAM Role & Policies

#### 信頼ポリシー
```json
{
  "Service": "lambda.amazonaws.com"
}
```

#### インラインポリシー（3つ）

**1) SQS Access**
```json
{
  "Action": [
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage",
    "sqs:GetQueueAttributes"
  ],
  "Resource": "arn:aws:sqs:*:*:lambda-agents-dev-queue"
}
```

**2) S3 Access**
```json
{
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::lambda-agents-dev-results-*",
    "arn:aws:s3:::lambda-agents-dev-results-*/*"
  ]
}
```

**3) Bedrock Access**
```json
{
  "Action": "bedrock:InvokeModel",
  "Resource": "arn:aws:bedrock:ap-northeast-1:*:foundation-model/*"
}
```

#### マネージドポリシー
- **AWSLambdaBasicExecutionRole**：CloudWatch Logs への書き込み権限

---

### 2.4 Lambda Function

#### 基本設定
```yaml
Runtime: python3.12
Handler: index.lambda_handler
Memory: 512 MB
Timeout: 60 秒
```

#### 環境変数
| 変数 | 値 | 用途 |
|------|-----|------|
| `SQS_QUEUE_URL` | CloudFormation 参照 | メッセージ受信 |
| `S3_BUCKET_NAME` | CloudFormation 参照 | 結果保存先 |
| `BEDROCK_REGION` | ap-northeast-1（パラメータ） | Bedrock API（東日本） |
| `LOG_LEVEL` | INFO | ログレベル |

#### インラインコード（Phase 1）
- **処理**：SQS メッセージを受信 → ログ出力
- **エラーハンドリング**：JSON パース エラー・例外をキャッチ
- **戻り値**：HTTP ステータスコード + JSON レスポンス

---

### 2.5 Event Source Mapping

#### 設定
```yaml
EventSourceArn: SQS Queue ARN
FunctionName: Lambda Function
Enabled: true
BatchSize: 1
MaximumBatchingWindowInSeconds: 0
FunctionResponseTypes: [ReportBatchItemFailures]
```

#### 動作フロー
```
SQS Queue
  ↓ (1件ずつ)
Event Source Mapping
  ↓ (イベント変換)
Lambda Invocation
  ↓ (成功/失敗)
SQS DeleteMessage or 再試行
```

---

### 2.6 CloudWatch Log Group

#### 設定
```yaml
LogGroupName: /aws/lambda/lambda-agents-dev-function
RetentionInDays: 30
```

#### 注意点
- **DependsOn**：Lambda 関数が作成前にログを作成しないよう`DependsOn`で明示

---

## 3. Outputs 設計

| Output | 説明 | Export | 用途 |
|--------|------|--------|------|
| `QueueURL` | SQS キュー URL | `lambda-agents-dev-QueueURL` | メッセージ送信テスト |
| `QueueArn` | SQS キュー ARN | `lambda-agents-dev-QueueArn` | 他スタック参照 |
| `BucketName` | S3 バケット名 | `lambda-agents-dev-BucketName` | 結果確認 |
| `LambdaFunctionArn` | Lambda ARN | `lambda-agents-dev-FunctionArn` | 他スタック参照 |
| `LambdaFunctionName` | Lambda 名 | `lambda-agents-dev-FunctionName` | テスト/デバッグ |
| `RoleArn` | IAM Role ARN | `lambda-agents-dev-RoleArn` | 監査/権限確認 |
| `LogGroupName` | CloudWatch ロググループ | `lambda-agents-dev-LogGroup` | ログ確認 |

---

## 4. デプロイ方法

### 4.1 Stack 作成（初回）
```bash
aws cloudformation create-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

### 4.2 Stack 更新
```bash
aws cloudformation update-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

### 4.3 Stack 削除
```bash
aws cloudformation delete-stack \
  --stack-name lambda-agents-dev \
  --region ap-northeast-1
```

### 4.4 パラメータ指定（オプション）
```bash
aws cloudformation create-stack \
  --stack-name lambda-agents-dev \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=LambdaMemory,ParameterValue=512 \
    ParameterKey=LogRetentionDays,ParameterValue=30 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

---

## 5. 検証チェックリスト

- [ ] テンプレートの YAML 構文を確認
- [ ] パラメータのデフォルト値が要件と一致
- [ ] IAM ポリシーの権限が最小化されているか
- [ ] リソース名がプレジェクト名規則に従っているか
- [ ] Output が次フェーズで参照可能か
- [ ] DependsOn が正しく設定されているか（Log Group → Lambda）

---

## 6. 今後の拡張

### Phase 2: Bedrock 統合
- Lambda コード内で Bedrock API を呼び出し
- S3 に結果を JSON で保存

### Phase 3: エラーハンドリング
- DLQ をアクティベート
- リトライロジックを実装

### Phase 4: 複数環境対応
- staging / prod 環境用に Stack を複製
- Parameter Store で環境固有の設定を管理

---

## 7. 既知の制限事項

1. **ローカルテスト**：SAM CLI 未対応（次フェーズで検討）
2. **テスト自動化**：外部テストツール未連携
3. **CI/CD パイプライン**：未実装（別スタックで対応予定）

---

## 参考資料

- [AWS CloudFormation Best Practices](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [Lambda with SQS Event Source](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/with-sqs.html)
- [Bedrock API Reference](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/what-is-bedrock.html)
