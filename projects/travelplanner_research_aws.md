# AWS 2026 ベストプラクティス調査報告書

**調査日**: 2026-02-07
**タスクID**: subtask_001a
**調査担当**: 足軽1

---

## 1. Webアプリケーション向けAWSアーキテクチャのベストプラクティス

### 推奨アーキテクチャパターン

2026年のAWSモダンアプリケーション開発では、以下のアプローチが推奨される：

#### 三層アーキテクチャ（コンテナ化）
- **プレゼンテーション層**: CloudFront + S3（静的コンテンツ）
- **アプリケーション層**: ECS/EKS on Fargate または Lambda
- **データ層**: Aurora Serverless v2 / DynamoDB

#### 主要コンポーネント
| コンポーネント | 推奨サービス | 役割 |
|--------------|-------------|------|
| CDN | CloudFront | グローバル配信、エッジキャッシュ |
| ロードバランサー | Application Load Balancer (ALB) | トラフィック分散、ヘルスチェック |
| API Gateway | Amazon API Gateway | APIリクエスト管理、スロットリング |
| コンピュート | ECS Fargate / Lambda | スケーラブルな処理実行 |
| データベース | Aurora Serverless v2 | 自動スケーリングRDB |
| キャッシュ | ElastiCache for Redis | セッション管理、高速キャッシュ |

### AWS Well-Architected Framework 6つの柱

2026年も変わらず以下の6つの柱が基盤：

1. **Operational Excellence（運用上の優秀性）**: 自動化、イベント対応、日次運用の標準化
2. **Security（セキュリティ）**: データ保護、権限管理、セキュリティイベント検出
3. **Reliability（信頼性）**: 障害からの迅速な復旧、需要対応
4. **Performance Efficiency（パフォーマンス効率）**: 適切なリソース選択、ニーズに応じた調整
5. **Cost Optimization（コスト最適化）**: コスト効率の良いソリューション提供
6. **Sustainability（持続可能性）**: 環境影響の最小化、リソース効率の最大化

---

## 2. サーバーレス vs コンテナ（ECS/EKS）の選択基準

### 選択フローチャート

```
開始
  │
  ├─ Kubernetes経験・マルチクラウド必要？
  │    ├─ Yes → EKS
  │    └─ No ─┬─ シンプルさ重視・AWS専用？
  │           │    ├─ Yes → ECS
  │           │    └─ No ─┬─ リクエスト駆動・短時間実行？
  │           │           │    ├─ Yes → Lambda（サーバーレス）
  │           │           │    └─ No → ECS/EKS + Fargate
```

### 各オプションの特徴比較

| 項目 | Lambda（サーバーレス） | ECS | EKS |
|------|----------------------|-----|-----|
| **運用負荷** | 最小 | 低 | 中〜高 |
| **スケーリング** | 自動（ミリ秒単位） | 自動 | 自動（設定必要） |
| **コスト** | 実行時間課金 | EC2/Fargate課金 | コントロールプレーン$73/月 + EC2/Fargate |
| **最大実行時間** | 15分 | 無制限 | 無制限 |
| **コールドスタート** | あり | なし | なし |
| **ポータビリティ** | AWS専用 | AWS専用 | マルチクラウド対応 |

### 2026年の推奨パターン

| ユースケース | 推奨サービス | 理由 |
|-------------|-------------|------|
| API/Webhook | Lambda + API Gateway | 低コスト、自動スケール |
| Webアプリ（常時稼働） | ECS + Fargate | コールドスタートなし、シンプル |
| マイクロサービス（大規模） | EKS + Fargate | Kubernetes標準、柔軟性 |
| バッチ処理 | AWS Batch / Step Functions | ジョブ管理最適化 |
| スパイク対応 | Fargate Spot | コスト最適化 |

### Fargateの活用

FargateはECS/EKS両方で使用可能なサーバーレスコンピュートエンジン：
- EC2インスタンス管理不要
- 秒単位課金
- ECS/EKS共通の料金体系

---

## 3. コスト最適化のベストプラクティス

### 2026年の重要指標

> 調査によると、**30-40%のクラウド支出がアイドルリソース、過剰プロビジョニング、ゾンビアセットに浪費されている**

### コスト最適化の5つの柱

#### 1. ライトサイジング（適正サイズ化）
- **現状**: AWS報告によると約50%のインスタンスが過剰サイズ
- **対策**: AWS Compute Optimizer を活用して適切なサイズを特定
- **ツール**: Cost Explorer、Trusted Advisor

#### 2. Gravitonプロセッサの採用
| 項目 | 効果 |
|------|------|
| コスト削減 | 20-40% |
| パフォーマンス | 同等以上 |
| 対応サービス | EC2, ECS, EKS, Lambda, RDS |

**推奨**: 新規ワークロードはGraviton (ARM) を第一選択とせよ

#### 3. 料金モデルの最適化

| モデル | 割引率 | 適用条件 |
|--------|-------|---------|
| Reserved Instances | 最大75% | 1-3年コミット、安定稼働 |
| Savings Plans | 最大72% | コンピュート使用量コミット |
| Spot Instances | 最大90% | 中断許容ワークロード |
| On-Demand | 0% | 変動ワークロード |

#### 4. ストレージ最適化
- **S3 Intelligent-Tiering**: アクセスパターンに応じた自動階層化
- **EBS最適化**: gp3への移行（gp2比で20%コスト削減）
- **ライフサイクルポリシー**: 古いデータの自動アーカイブ

#### 5. ガバナンスと可視化
- **必須タグ**: プロジェクト、環境、オーナー
- **AWS Organizations**: タグポリシーの強制
- **Cost Explorer**: タグ別コスト分析
- **Budgets**: 予算アラート設定

### アイドルリソース対策

```
Instance Scheduler on AWS
  ├─ 開発環境: 平日9-18時のみ稼働 → 約70%削減
  ├─ テスト環境: 必要時のみ起動
  └─ 本番環境: 常時稼働（ただしスケーリング最適化）
```

---

## 4. セキュリティのベストプラクティス

### IAM（Identity and Access Management）

#### 2026年の必須対策

| 対策 | 優先度 | 詳細 |
|------|-------|------|
| ルートアカウントのアクセスキー削除 | 最高 | 完全削除必須 |
| ハードウェアMFA / FIDO2 | 最高 | ルートアカウントに必須 |
| IAMロールの使用 | 最高 | 静的キーを廃止 |
| 最小権限の原則 | 高 | 四半期ごとにレビュー |
| IAM Access Analyzer | 高 | 外部アクセス検出 |
| Service Control Policies (SCPs) | 中 | 組織全体の権限制御 |

#### 推奨ポリシー設計

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::bucket-name/*",
      "Condition": {
        "StringEquals": {"aws:PrincipalTag/Project": "TravelPlanner"}
      }
    }
  ]
}
```

### VPC（Virtual Private Cloud）

#### ゼロトラストネットワーク設計

```
インターネット
     │
     ▼
[CloudFront] ─── [WAF]
     │
     ▼
[ALB in Public Subnet]
     │
     ▼
[ECS Tasks in Private Subnet] ─── [VPC Endpoint] ─── [S3/DynamoDB]
     │
     ▼
[Aurora in Isolated Subnet]
```

#### VPCセキュリティ設計の要点

| 項目 | ベストプラクティス |
|------|------------------|
| サブネット設計 | Public/Private/Isolated の3層 |
| セキュリティグループ | 最小限のポート開放、0.0.0.0/0禁止 |
| NACL | サブネットレベルの追加防御 |
| VPC Endpoints | S3/DynamoDB等へのプライベート接続 |
| VPC Flow Logs | 全VPC/サブネットで有効化 |
| NAT Gateway | Private SubnetからのOutbound用 |

### WAF（Web Application Firewall）

#### 推奨ルール構成

| ルールセット | 目的 |
|-------------|------|
| AWS Managed Rules - Common | 一般的な攻撃パターン |
| AWS Managed Rules - Known Bad Inputs | 既知の悪意あるペイロード |
| AWS Managed Rules - SQL Database | SQLインジェクション防止 |
| Rate-based Rules | DDoS/ブルートフォース対策 |
| Geographic Restrictions | 地域制限（必要に応じて） |

#### 追加セキュリティサービス

- **Amazon GuardDuty**: 脅威検出（VPC Flow Logs分析）
- **AWS Security Hub**: セキュリティ状態の一元管理
- **AWS Config**: 設定変更の追跡・コンプライアンス
- **AWS Secrets Manager**: シークレットの安全な管理

---

## 5. 可用性・耐障害性のベストプラクティス

### 基本概念

| 概念 | 定義 | 目標 |
|------|------|------|
| High Availability (HA) | 障害時も運用継続 | 99.9%〜99.99% uptime |
| Fault Tolerance (FT) | サブシステム障害を許容 | 0ダウンタイム |

### マルチAZ設計（必須）

```
Region: ap-northeast-1 (Tokyo)
  │
  ├── AZ-a
  │     ├── ALB Node
  │     ├── ECS Task (Primary)
  │     └── Aurora Writer
  │
  ├── AZ-c
  │     ├── ALB Node
  │     ├── ECS Task (Secondary)
  │     └── Aurora Reader
  │
  └── AZ-d
        ├── ALB Node
        ├── ECS Task (Tertiary)
        └── Aurora Reader
```

### 可用性パターン別の構成

| 可用性目標 | 年間ダウンタイム | 推奨構成 |
|-----------|----------------|---------|
| 99.9% | 8.76時間 | マルチAZ + ALB |
| 99.95% | 4.38時間 | マルチAZ + Auto Scaling |
| 99.99% | 52分 | マルチリージョン + Route53 |

### 重要サービスの耐障害性設定

| サービス | 推奨構成 |
|---------|---------|
| RDS/Aurora | Multi-AZ DB Cluster（3AZ、Writer+2Reader） |
| ElastiCache | Multi-AZ with Auto-Failover |
| ECS/EKS | 複数AZにタスク分散 |
| Lambda | デフォルトで複数AZ対応 |
| S3 | デフォルトで11 9s の耐久性 |

### Static Stability（静的安定性）

障害発生前に冗長リソースを事前プロビジョニング：
- Auto Scalingの最小値を余裕を持って設定
- 障害時のスケールアウト不要で即座に対応

### 障害テスト

- **AWS Fault Injection Simulator (FIS)**:
  - ネットワーク遅延シミュレーション
  - インスタンス終了シミュレーション
  - AZ障害シミュレーション
- **Game Days**: 定期的な障害訓練

---

## 6. TravelPlannerアプリ向け推奨アーキテクチャ

上記調査結果を踏まえ、TravelPlannerアプリに推奨するアーキテクチャ：

### 推奨構成

```
[Route 53]
    │
    ▼
[CloudFront + WAF] ─── [S3: Static Assets]
    │
    ▼
[API Gateway]
    │
    ▼
[Lambda / ECS Fargate] ←── [Secrets Manager]
    │
    ├── [Aurora Serverless v2] (Multi-AZ)
    ├── [DynamoDB] (セッション/キャッシュ)
    └── [S3] (ユーザーアップロード)
```

### 選択理由

| コンポーネント | 選択 | 理由 |
|--------------|------|------|
| コンピュート | ECS Fargate | コールドスタートなし、シンプル、Graviton対応 |
| DB | Aurora Serverless v2 | 自動スケーリング、コスト効率 |
| キャッシュ | DynamoDB | サーバーレス、低遅延 |
| CDN | CloudFront | グローバル配信、WAF統合 |

### 概算月額コスト（小〜中規模）

| サービス | 月額概算 |
|---------|---------|
| ECS Fargate (2 vCPU, 4GB) x2 | $60-100 |
| Aurora Serverless v2 | $50-150 |
| CloudFront | $20-50 |
| その他（ALB, Route53等） | $30-50 |
| **合計** | **$160-350** |

※ Savings Plans適用で30-50%削減可能

---

## 参考資料

### 公式ドキュメント
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS Modern Applications](https://aws.amazon.com/modern-apps/)
- [AWS Cost Optimization](https://aws.amazon.com/aws-cost-management/cost-optimization/)
- [VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)

### 調査に使用した情報源
- [AWS Cost Optimization Complete Guide 2026](https://aws51.com/en/aws-cost-optimization-guide-2026/)
- [Top 12 AWS Cost Optimization Tools 2026](https://northflank.com/blog/aws-cost-optimization)
- [12 AWS Security Best Practices 2026](https://www.sentinelone.com/cybersecurity-101/cloud-security/aws-security-best-practices/)
- [EKS vs ECS 2026](https://squareops.com/eks-vs-ecs/)
- [AWS Container Service Selection Guide](https://docs.aws.amazon.com/decision-guides/latest/containers-on-aws-how-to-choose/choosing-aws-container-service.html)
- [High Availability and Fault Tolerance](https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/fault-tolerance-and-fault-isolation.html)

---

**報告完了**: 2026-02-07
**足軽1 拝**
