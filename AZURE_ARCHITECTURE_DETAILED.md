# Multi-Agent Custom Automation Engine - Azure アーキテクチャ詳細解説

## 📋 目次
1. [全体アーキテクチャ概要](#全体アーキテクチャ概要)
2. [コアコンポーネント](#コアコンポーネント)
3. [データフローと処理プロセス](#データフローと処理プロセス)
4. [AIエージェントシステム](#aiエージェントシステム)
5. [セキュリティとアイデンティティ](#セキュリティとアイデンティティ)
6. [ストレージとデータ管理](#ストレージとデータ管理)
7. [ネットワーキング](#ネットワーキング)
8. [監視とロギング](#監視とロギング)
9. [スケーラビリティと可用性](#スケーラビリティと可用性)
10. [コスト最適化](#コスト最適化)

---

## 全体アーキテクチャ概要

このソリューションは、**マルチエージェントAIシステム**を実現するためのエンタープライズグレードのアーキテクチャです。

### アーキテクチャの特徴
- 🎯 **マイクロサービスアーキテクチャ**: フロントエンド、バックエンドAPI、MCPサーバーを分離
- 🤖 **AI駆動型**: Azure AI Foundryを活用した高度なエージェントオーケストレーション
- 🔒 **セキュアバイデザイン**: Managed Identity、Key Vault、Private Endpointsによる多層防御
- 📈 **スケーラブル**: Container Apps、Cosmos DBによる自動スケーリング
- 🌐 **グローバル対応**: 複数リージョンでのデプロイメントサポート

---

## コアコンポーネント

### 1. フロントエンド層

#### **Azure App Service (Website)**
- **種類**: Linux Container App Service
- **SKU**: B3 (基本) または P1v4 (スケーラビリティ有効時)
- **役割**: ユーザーインターフェース (React/Next.js)
- **機能**:
  - マルチエージェントプランナーUIの提供
  - バックエンドAPIとの通信
  - リアルタイムチャットインターフェース
  - チーム選択とタスク管理

**技術詳細** (infra/main.bicep:1507-1542):
```bicep
- Dockerコンテナベース (Port 3000)
- TLS 1.2以上のみ対応
- Application Insightsと統合
- VNet統合サポート (Private Networking有効時)
```

**環境変数**:
- `BACKEND_API_URL`: バックエンドContainer AppのFQDN
- `AUTH_ENABLED`: 認証設定 (デフォルト: false)

---

### 2. バックエンドAPI層

#### **Azure Container Apps - Backend API**
- **役割**: マルチエージェントオーケストレーションエンジン
- **フレームワーク**: Python FastAPI + Azure Agent Framework
- **リソース**: 2.0 CPU, 4.0Gi メモリ
- **スケール設定**:
  - 最小レプリカ: 1
  - 最大レプリカ: 3 (スケーラビリティ有効時)
  - スケールトリガー: HTTP同時リクエスト100件

**技術詳細** (infra/main.bicep:1164-1380):
```bicep
主な機能:
✅ エージェントチームの管理とオーケストレーション
✅ タスクの分解と実行計画の作成
✅ 各エージェントへのタスク割り当て
✅ 実行結果の集約と検証
✅ セッション管理 (Cosmos DB)
✅ AI Searchとの統合
```

**接続先サービス**:
1. **Azure AI Foundry** - エージェント実行
2. **Cosmos DB** - セッションとメモリストレージ
3. **Azure AI Search** - ドキュメント検索
4. **Azure Storage** - データセットアクセス
5. **MCP Server** - 外部ツール統合

**環境変数** (主要なもの):
```bash
COSMOSDB_ENDPOINT: Cosmos DBエンドポイント
AZURE_OPENAI_ENDPOINT: Azure OpenAIエンドポイント
AZURE_AI_PROJECT_ENDPOINT: AI Foundryプロジェクトエンドポイント
AZURE_AI_SEARCH_ENDPOINT: AI Searchエンドポイント
MCP_SERVER_ENDPOINT: MCPサーバーエンドポイント
REASONING_MODEL_NAME: o4-mini (推論モデル)
```

---

### 3. MCP (Model Context Protocol) サーバー

#### **Azure Container Apps - MCP Server**
- **役割**: 外部ツールとエージェントの統合
- **ポート**: 9000
- **リソース**: 2.0 CPU, 4.0Gi メモリ

**機能** (docs/mcp_server.md参照):
```python
提供するツール:
1. HR関連ツール - 従業員情報取得、オンボーディング
2. プランニングツール - タスク分解、スケジューリング
3. データアクセスツール - データセットへのアクセス
4. グリーティングツール - デモ用
```

**MCPプロトコル**:
- JSON-RPCベースの通信
- SSE (Server-Sent Events) によるストリーミング
- OAuth2認証サポート (オプション)

---

## AIエージェントシステム

### Azure AI Foundry アーキテクチャ

#### **1. AI Services (Cognitive Services Account)**
- **種類**: AIServices (統合アカウント)
- **SKU**: S0
- **リージョン**: East US 2 (あなたのデプロイ)
- **認証**: Managed Identity (ローカル認証無効)

**デプロイされたモデル** (infra/main.bicep:770-926):

| モデル名 | 用途 | キャパシティ | デプロイメントタイプ |
|---------|-----|------------|------------------|
| **gpt-4.1** | 主要な対話モデル | 150 TPM | GlobalStandard |
| **gpt-4.1-mini** | 軽量タスク用 | 50 TPM | GlobalStandard |
| **o4-mini** | 推論タスク用 | 50 TPM | GlobalStandard |

**TPM (Tokens Per Minute)**: 1分あたりの処理可能トークン数

#### **2. AI Foundry Project**
- **役割**: エージェント実行環境の提供
- **API**: Agent Service API (2025-01-01-preview)
- **機能**:
  - マルチエージェントオーケストレーション
  - コンテキスト管理
  - ツール統合 (Function Calling)
  - メモリ管理

#### **3. Foundry Agent Service**
**Agent Frameworkの機能**:
```python
主要機能:
- エージェントチームの定義と管理
- タスクの自動分解とルーティング
- エージェント間のコミュニケーション
- 実行結果の検証と再試行
- コンテキストの保持と共有
```

**エージェントチームの例** (data/agent_teams/):
1. **RFP Evaluation Team**: RFP審査エージェント群
2. **HR Onboarding Team**: 従業員オンボーディングエージェント群
3. **Retail Satisfaction Team**: カスタマーサービスエージェント群
4. **Marketing Team**: マーケティングプランニングエージェント群
5. **Contract Review Team**: 契約審査エージェント群

---

## データフローと処理プロセス

### エンドツーエンドのリクエストフロー

```
┌─────────┐
│  User   │
└────┬────┘
     │ 1. HTTPSリクエスト
     ▼
┌─────────────────┐
│ App Service     │
│ (Frontend)      │
└────┬────────────┘
     │ 2. API呼び出し
     ▼
┌─────────────────────────┐
│ Container App           │
│ (Backend API)           │
│ - FastAPI               │
│ - Agent Framework       │
└────┬───────┬────────┬───┘
     │       │        │
     │       │        └──────────────┐
     │       │ 3. エージェント実行   │
     │       ▼                       ▼
     │  ┌──────────────┐      ┌──────────────┐
     │  │ AI Foundry   │      │ MCP Server   │
     │  │ - GPT-4.1    │      │ - Tools      │
     │  │ - o4-mini    │      │ - Functions  │
     │  └──────────────┘      └──────────────┘
     │       │
     │       │ 4. データ検索
     ▼       ▼
┌─────────────────┐     ┌─────────────────┐
│ Cosmos DB       │     │ AI Search       │
│ - Sessions      │     │ - Documents     │
│ - Memory        │     │ - Vector Index  │
└─────────────────┘     └─────────────────┘
                              │
                              │ 5. ソースデータ
                              ▼
                        ┌─────────────────┐
                        │ Azure Storage   │
                        │ - Datasets      │
                        │ - Documents     │
                        └─────────────────┘
```

### 処理フローの詳細

#### **フェーズ1: リクエスト受信**
1. ユーザーがフロントエンドでタスクを入力
2. App ServiceがBackend APIにHTTPS POSTリクエスト送信
3. Backend APIがリクエストを検証し、セッションIDを生成

#### **フェーズ2: タスク分析**
```python
# Backend APIの処理
1. タスクをAI Foundryに送信
2. GPT-4.1がタスクを分析
   - タスクの複雑度評価
   - 必要なエージェントの特定
   - サブタスクへの分解
3. 実行プランの作成
```

#### **フェーズ3: エージェント実行**
```python
# Agent Frameworkによるオーケストレーション
for agent in assigned_agents:
    1. エージェントにコンテキストを提供
    2. 必要なツールを割り当て (MCP Server経由)
    3. AI Searchでドキュメント検索
    4. エージェントがタスクを実行
    5. 結果をCosmos DBに保存
```

#### **フェーズ4: 結果の集約**
```python
1. 各エージェントの実行結果を収集
2. o4-mini (推論モデル) で結果を検証
3. 不整合がある場合は再実行
4. 最終結果を生成
5. フロントエンドにストリーミング返信
```

---

## セキュリティとアイデンティティ

### 1. Managed Identity (マネージドID)

#### **User-Assigned Managed Identity**
- **役割**: すべてのAzureリソース間の認証に使用
- **利点**:
  - ✅ シークレット管理不要
  - ✅ 自動的なキーローテーション
  - ✅ Azure RBACとの統合

**割り当てられたロール** (infra/main.bicep):
```bicep
AI Services:
- Azure AI User
- Azure AI Developer
- Cognitive Services OpenAI User

Cosmos DB:
- Cosmos DB SQL Data Contributor (カスタムロール)

Storage Account:
- Storage Blob Data Contributor

AI Search:
- Search Index Data Contributor

Key Vault:
- Key Vault Administrator
```

### 2. Azure Key Vault

**保存されるシークレット**:
- ✅ Azure AI Search API Key
- (将来) データベース接続文字列
- (将来) 外部APIキー

**設定** (infra/main.bicep:1759-1809):
```bicep
- SKU: Standard (または Premium でスケーラビリティ有効時)
- RBAC認証有効
- Soft Delete有効 (7日間保持)
- Private Endpoint (プライベートネットワーク有効時)
- 診断ログをLog Analyticsに送信
```

### 3. ネットワークセキュリティ

#### **パブリックアクセス設定** (デフォルト):
```
App Service: Enabled (エンドユーザーアクセス用)
Container Apps: Enabled (HTTPSのみ)
Cosmos DB: Enabled + RBAC
AI Services: Enabled + AAD認証
Storage Account: Enabled + RBAC
Key Vault: Enabled + RBAC
AI Search: Enabled (他サービスとの互換性のため)
```

#### **Private Networking有効時**:
- Virtual Network作成 (10.0.0.0/8)
- Private Endpoints for:
  - AI Services
  - Cosmos DB
  - Storage Account (Blob)
  - Key Vault
- Private DNS Zones:
  - privatelink.cognitiveservices.azure.com
  - privatelink.openai.azure.com
  - privatelink.services.ai.azure.com
  - privatelink.documents.azure.com
  - privatelink.blob.core.windows.net
  - privatelink.vaultcore.azure.net
- Azure Bastion (管理用アクセス)
- Windows VM (Jumpbox)

---

## ストレージとデータ管理

### 1. Azure Cosmos DB

#### **構成** (infra/main.bicep:1026-1111):
```bicep
API: SQL (NoSQL)
データベース名: macae
コンテナ名: memory
パーティションキー: /session_id
一貫性レベル: Session (デフォルト)
```

#### **使用目的**:
1. **セッション管理**:
   ```json
   {
     "session_id": "uuid",
     "user_id": "user_principal_id",
     "created_at": "timestamp",
     "messages": [],
     "context": {}
   }
   ```

2. **エージェントメモリ**:
   ```json
   {
     "session_id": "uuid",
     "agent_id": "agent_name",
     "memory_type": "short_term|long_term",
     "content": "...",
     "timestamp": "..."
   }
   ```

#### **スケーリング設定**:
- **デフォルト**: Serverlessモード (使用量ベース課金)
- **Redundancy有効時**:
  - プロビジョニング済みスループット
  - Zone Redundancy有効
  - 2リージョンレプリケーション (East US 2 → Central US)

### 2. Azure Storage Account

#### **構成** (infra/main.bicep:1556-1651):
```bicep
種類: StorageV2
アクセス層: Hot
最小TLS: 1.2
パブリックBlobアクセス: 無効
```

#### **Blobコンテナ**:
| コンテナ名 | 用途 | データ例 |
|-----------|------|---------|
| retail-dataset-customer | 顧客データ | CSVファイル |
| retail-dataset-order | 注文データ | JSONファイル |
| rfp-summary-dataset | RFP要約文書 | PDFファイル |
| rfp-risk-dataset | RFPリスク分析 | Markdownファイル |
| rfp-compliance-dataset | RFPコンプライアンス | DOCXファイル |
| contract-*-dataset | 契約関連文書 | 各種形式 |

#### **データ保護機能**:
- ✅ Soft Delete (9日間)
- ✅ Container Delete Retention (10日間)
- ✅ Automatic Snapshot (有効)
- ✅ Last Access Time Tracking

### 3. Azure AI Search

#### **構成** (infra/main.bicep:1665-1735):
```bicep
SKU: Basic (デフォルト) または Standard (スケーラビリティ有効時)
パーティション: 1
レプリカ: 1
認証: API Key + RBAC (AAD)
```

#### **インデックス**:
| インデックス名 | ソースコンテナ | 用途 |
|--------------|--------------|------|
| macae-retail-customer-index | retail-dataset-customer | 顧客情報検索 |
| macae-retail-order-index | retail-dataset-order | 注文履歴検索 |
| macae-rfp-summary-index | rfp-summary-dataset | RFP要約検索 |
| macae-rfp-risk-index | rfp-risk-dataset | リスク分析検索 |
| macae-rfp-compliance-index | rfp-compliance-dataset | コンプライアンス検索 |
| contract-*-index | contract-*-dataset | 契約文書検索 |

#### **AI Foundry統合**:
- Connection作成 (infra/modules/aifp-connections.bicep)
- エージェントからの直接アクセス
- セマンティック検索機能
- ベクトル検索サポート

---

## ネットワーキング

### 1. Container App Environment

#### **構成** (infra/main.bicep:1117-1158):
```bicep
Public Network Access: Enabled
Internal: false (外部アクセス許可)
Zone Redundancy: Redundancy有効時にtrue
```

#### **ワークロードプロファイル**:
- **デフォルト**: Consumption (従量課金)
- **Redundancy有効時**: D4 (専用ハードウェア、3インスタンス)

#### **ログ設定**:
- Log Analyticsと統合
- Application Insightsと統合

### 2. App Service Plan

#### **構成** (infra/main.bicep:1482-1498):
```bicep
Kind: Linux
Reserved: true (Linux専用)
SKU: B3 または P1v4
Capacity: 1 または 3
Zone Redundant: Redundancy有効時にtrue
```

### 3. Virtual Network (Private Networking有効時)

#### **サブネット構成** (infra/modules/virtualNetwork.bicep):
```
VNet CIDR: 10.0.0.0/8

サブネット:
1. AzureBastionSubnet: 10.0.1.0/26 (64 IPs)
   - Azure Bastion専用

2. administration-subnet: 10.1.0.0/24 (256 IPs)
   - 管理VM用

3. backend-subnet: 10.2.0.0/24 (256 IPs)
   - Private Endpoints用
   - AI Services, Cosmos DB, Storage, Key Vault

4. webserverfarm-subnet: 10.3.0.0/24 (256 IPs)
   - App Service VNet統合

5. container-subnet: 10.4.0.0/23 (512 IPs)
   - Container Apps専用
```

---

## 監視とロギング

### 1. Application Insights

#### **構成** (infra/main.bicep:359-374):
```bicep
種類: Web
データ保持: 365日
Workspace統合: Log Analytics Workspace
IP Masking: 有効 (プライバシー保護)
```

#### **収集データ**:
- ✅ HTTPリクエスト/レスポンス
- ✅ 依存関係の呼び出し (AI Services, Cosmos DB, etc.)
- ✅ 例外とエラー
- ✅ カスタムイベントとメトリクス
- ✅ パフォーマンスカウンター

#### **統合サービス**:
- Backend Container App
- Frontend App Service
- Container App Environment

### 2. Log Analytics Workspace

#### **構成** (infra/main.bicep:283-340):
```bicep
SKU: PerGB2018 (従量課金)
データ保持: 365日
Daily Quota: Redundancy有効時150GB
Replication: Redundancy有効時、ペアリージョンに複製
```

#### **データソース**:
- Container Apps ログ
- App Service ログ
- AI Services 診断ログ
- Cosmos DB メトリクス
- Storage Account ログ
- Key Vault 監査ログ

#### **クエリ例**:
```kusto
// エージェント実行時間の分析
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "ca-macaedev..."
| where Log_s contains "agent_execution"
| summarize avg(duration_ms) by agent_name

// エラー率の監視
AppRequests
| where Success == false
| summarize error_count=count() by bin(TimeGenerated, 1h)
| render timechart
```

### 3. Azure Monitor Alerts (設定推奨)

**推奨アラート**:
```yaml
1. Container App CPU使用率 > 80%
2. Cosmos DB RU/s 使用率 > 90%
3. Application Insights エラー率 > 5%
4. AI Services スロットリング検出
5. Storage Account 可用性 < 99.9%
```

---

## スケーラビリティと可用性

### 1. 自動スケーリング設定

#### **Container Apps**:
```yaml
Backend API:
  最小レプリカ: 1
  最大レプリカ: 3 (enableScalability=true時)
  スケールトリガー:
    - HTTP同時リクエスト: 100件
    - メトリクス: CPU使用率、メモリ使用率

MCP Server:
  最小レプリカ: 1
  最大レプリカ: 3 (enableScalability=true時)
  スケールトリガー:
    - HTTP同時リクエスト: 100件
```

#### **App Service**:
```yaml
SKU: P1v4 (enableScalability=true時)
インスタンス数: 3
自動スケール: 手動設定可能
```

### 2. 高可用性構成 (enableRedundancy=true時)

#### **リージョンペアリング**:
```
East US 2 ⇄ Central US (データレプリケーション)
```

#### **Zone Redundancy**:
- Container Apps: 3つのAvailability Zonesに分散
- Cosmos DB: Zone Redundant有効
- App Service: Zone Redundant有効

#### **データレプリケーション**:
- Cosmos DB: 2リージョンレプリケーション
- Log Analytics: ペアリージョンへのレプリケーション

### 3. SLA (Service Level Agreement)

**各サービスのSLA**:
| サービス | SLA | 構成 |
|---------|-----|------|
| Container Apps | 99.95% | Zone Redundant |
| App Service | 99.95% | Premium Plan |
| Cosmos DB | 99.999% | Multi-region writes |
| AI Services | 99.9% | Standard |
| Storage Account | 99.9% | RA-GRS |

**複合SLA計算**:
```
全体SLA ≈ 99.95% × 99.95% × 99.9% × 99.9% = 99.7%
```

---

## コスト最適化

### 1. 推定月額コスト (East US 2リージョン)

#### **基本構成** (enableScalability=false, enableRedundancy=false):
```
Container Apps (Consumption): $50-100/月
App Service (B3): $55/月
Cosmos DB (Serverless): $25-100/月 (使用量による)
AI Services (S0): $1,000-2,000/月 (使用量による)
  - GPT-4.1 (150K TPM): ~$1,500/月
  - o4-mini (50K TPM): ~$300/月
  - gpt-4.1-mini (50K TPM): ~$200/月
Storage Account: $20-50/月
AI Search (Basic): $75/月
Application Insights: $50-100/月
Log Analytics: $50-150/月

合計: $1,325 - $2,630/月
```

#### **スケーラブル構成** (enableScalability=true):
```
App Service (P1v4): $150/月
Container Apps (3 replicas): $150-300/月
AI Search (Standard): $250/月

追加コスト: +$400-600/月
```

#### **高可用性構成** (enableRedundancy=true):
```
Cosmos DB (Provisioned, Multi-region): +$500-1,000/月
Container App Environment (Dedicated): +$300-500/月
Log Analytics Replication: +$100/月

追加コスト: +$900-1,600/月
```

### 2. コスト削減のベストプラクティス

#### **開発/テスト環境**:
```bash
# デフォルト設定を使用
azd up
# または
enableScalability=false
enableRedundancy=false
enableMonitoring=false  # 監視を無効化
```

#### **本番環境**:
```bash
# スケーラビリティのみ有効
enableScalability=true
enableRedundancy=false  # 必要に応じて

# または既存のLog Analyticsを再利用
existingLogAnalyticsWorkspaceId="<resource-id>"
```

#### **AI Servicesコスト削減**:
1. **モデルキャパシティの調整**:
   ```bash
   gpt4_1ModelCapacity=50  # 150から50に削減
   gptModelCapacity=30     # 50から30に削減
   ```

2. **Serverless Cosmos DBの活用**:
   - 低トラフィック時は自動的にコスト削減
   - RU/s課金なし、使用量のみ課金

3. **Auto-pauseの活用** (将来):
   - Container Appsのidle時自動停止

### 3. リソースタグ付け

**自動タグ** (infra/main.bicep:222-245):
```yaml
azd-env-name: <environment-name>
TemplateName: MACAE
Type: WAF または Non-WAF
CreatedBy: <deployer-username>
DeploymentName: <deployment-name>
```

**コスト分析での活用**:
- Azure Cost Managementでタグ別にフィルタリング
- 環境別コスト追跡
- チャージバック/ショーバック

---

## デプロイメントパラメータ

### カスタマイズ可能なパラメータ

#### **基本設定**:
```bash
solutionName: macae (3-16文字)
location: eastus2, japaneast, etc.
azureAiServiceLocation: eastus2, japaneast, etc.
```

#### **AIモデル設定**:
```bash
# GPT-4.1
gpt4_1ModelName: gpt-4.1
gpt4_1ModelVersion: 2025-04-14
gpt4_1ModelCapacity: 150
gpt4_1ModelDeploymentType: GlobalStandard

# GPT-4.1 Mini
gptModelName: gpt-4.1-mini
gptModelVersion: 2025-04-14
gptModelCapacity: 50
gptModelDeploymentType: GlobalStandard

# Reasoning Model (o4-mini)
gptReasoningModelName: o4-mini
gptReasoningModelVersion: 2025-04-16
gptReasoningModelCapacity: 50
gptReasoningModelDeploymentType: GlobalStandard
```

#### **Well-Architected Framework設定**:
```bash
enableMonitoring: true/false       # 監視機能
enableScalability: true/false      # スケーラビリティ
enableRedundancy: true/false       # 高可用性
enablePrivateNetworking: true/false # プライベートネットワーク
```

#### **既存リソースの再利用**:
```bash
existingLogAnalyticsWorkspaceId: <resource-id>
existingAiFoundryAiProjectResourceId: <resource-id>
```

---

## トラブルシューティング

### よくある問題と解決策

#### **1. "No team selected" エラー**
**原因**: Post-deploymentスクリプト未実行

**解決策**:
```bash
bash infra/scripts/selecting_team_config_and_data.sh
```

#### **2. Agent実行エラー**
**原因**: AI Searchアクセス権限不足

**解決策**:
```bash
# AI ProjectにSearch権限を付与 (自動設定済み)
# 手動確認:
az role assignment list --assignee <ai-project-principal-id> \
  --scope /subscriptions/.../providers/Microsoft.Search/searchServices/...
```

#### **3. Container App起動失敗**
**原因**: 環境変数の設定ミス

**解決策**:
```bash
# Container Appログを確認
az containerapp logs show \
  --name ca-macaedev... \
  --resource-group azure-foundry-multi-agent \
  --follow
```

#### **4. Cosmos DBアクセスエラー**
**原因**: RBAC権限の伝播遅延

**解決策**:
```bash
# 5-10分待機後、再試行
# または手動でロール割り当てを確認
az cosmosdb sql role assignment list \
  --account-name cosmos-macaedev... \
  --resource-group azure-foundry-multi-agent
```

---

## まとめ

このアーキテクチャは、以下の要素を組み合わせた**エンタープライズグレードのマルチエージェントAIシステム**です:

### ✅ 主要な強み
1. **モジュラー設計**: 各コンポーネントが疎結合で独立してスケール可能
2. **セキュア**: Managed Identity、Private Endpoints、RBACによる多層防御
3. **スケーラブル**: Container Apps、Cosmos DBによる自動スケーリング
4. **監視可能**: Application Insights、Log Analyticsによる包括的な可観測性
5. **コスト最適化**: Serverlessオプション、使用量ベース課金

### 🎯 ユースケース
- RFP評価自動化
- カスタマーサポート自動化
- HR業務自動化
- マーケティングプランニング
- 契約審査自動化

### 📚 さらに詳しく
- [デプロイメントガイド](./docs/DeploymentGuide.md)
- [ローカル開発セットアップ](./docs/LocalDevelopmentSetup.md)
- [トラブルシューティング](./docs/TroubleShootingSteps.md)
- [MCP Serverドキュメント](./docs/mcp_server.md)

---

**作成日**: 2026-01-15
**バージョン**: 1.0
**対象リージョン**: East US 2 (azure-foundry-multi-agent)
