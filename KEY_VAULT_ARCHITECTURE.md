# Azure Key Vault Architecture - Multi-Agent Custom Automation Engine

## 概要 (Overview)

このドキュメントでは、Multi-Agent Custom Automation Engine ソリューションにおける Azure Key Vault (`kv-macaedev2vhh3`) の役割と使用方法について詳細に解説します。

ユーザーの認識は正しいです：**Managed Identity がリソース間の認証に使用されています**。しかし、Key Vault は Managed Identity で対応できない特定のシークレット管理のために使用されています。

---

## Key Vault の役割

### なぜ Key Vault が必要なのか？

このソリューションでは、以下の2つの認証メカニズムが併用されています：

| 認証方式 | 用途 | 対象リソース |
|---------|------|------------|
| **Managed Identity** | リソース間の認証 | Cosmos DB, Storage Account, AI Foundry, Container Apps |
| **Key Vault** | API キーの安全な保管 | Azure AI Search API Key |

**重要なポイント：**
- Cosmos DB、Storage Account、AI Foundry などは Managed Identity でアクセス可能
- Azure AI Search は、このデプロイ構成では API キー認証を使用
- API キーを環境変数にハードコードするのではなく、Key Vault で安全に管理

---

## Key Vault に保存されているシークレット

### 1. Azure AI Search API Key

**シークレット名:** `AzureAISearchAPIKey`

**用途:** Azure AI Search サービスへのアクセス認証

**保存される値:** Azure AI Search サービスのプライマリキー

#### Bicep 構成 (`infra/main.bicep` lines 1801-1806)

```bicep
secrets: [
  {
    name: 'AzureAISearchAPIKey'
    value: searchService.outputs.primaryKey
  }
]
```

---

## アクセスパターン: Key Vault からシークレットを取得する流れ

### ステップバイステップのフロー

```
1. Container App (Backend API) 起動
   ↓
2. Managed Identity を使用して Key Vault に認証
   ↓
3. Key Vault から 'AzureAISearchAPIKey' シークレットを取得
   ↓
4. 環境変数 AZURE_AI_SEARCH_API_KEY に設定
   ↓
5. Backend コードが環境変数を読み取る
   ↓
6. Azure AI Search サービスへの API 呼び出しに使用
```

### Container App での Key Vault 参照設定

#### Backend Container App (`infra/main.bicep` lines 1310-1332, 1372-1378)

```bicep
// 環境変数の定義
{
  name: 'AZURE_AI_SEARCH_API_KEY'
  secretRef: 'azure-ai-search-api-key'  // Container App のシークレット参照
}

// Container App のシークレット定義
secrets: [
  {
    name: 'azure-ai-search-api-key'
    keyVaultUrl: keyvault.outputs.secrets[0].uriWithVersion  // Key Vault URL
    identity: userAssignedIdentity.outputs.resourceId        // Managed Identity
  }
]
```

**解説:**
1. `AZURE_AI_SEARCH_API_KEY` 環境変数が `azure-ai-search-api-key` シークレットを参照
2. `azure-ai-search-api-key` シークレットは Key Vault から値を取得
3. Managed Identity (`userAssignedIdentity`) を使用して Key Vault に認証

---

## Backend コードでの使用

### 設定の読み込み (`src/backend/common/config/app_config.py` line 97)

```python
class AppConfig:
    def __init__(self):
        # Key Vault から取得された API キーを環境変数から読み取る
        self.AZURE_AI_SEARCH_API_KEY = self._get_optional("AZURE_AI_SEARCH_API_KEY")
        self.AZURE_AI_SEARCH_ENDPOINT = self._get_optional("AZURE_AI_SEARCH_ENDPOINT")
```

### Azure AI Search への接続

Backend API は以下のように Azure AI Search に接続します：

```python
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential

# 環境変数から API キーとエンドポイントを取得
api_key = config.AZURE_AI_SEARCH_API_KEY
endpoint = config.AZURE_AI_SEARCH_ENDPOINT

# SearchClient の作成
search_client = SearchClient(
    endpoint=endpoint,
    index_name="your-index-name",
    credential=AzureKeyCredential(api_key)
)
```

---

## Key Vault のセキュリティ設定

### RBAC (Role-Based Access Control)

#### Managed Identity の権限 (`infra/main.bicep` lines 1794-1800)

```bicep
roleAssignments: [
  {
    principalId: userAssignedIdentity.outputs.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionIdOrName: 'Key Vault Administrator'
  }
]
```

**付与される権限:**
- **Key Vault Administrator**: シークレット、キー、証明書の完全な管理権限
- Container Apps はこの Managed Identity を使用して Key Vault にアクセス

### セキュリティ機能

#### 1. RBAC 認証の有効化 (`infra/main.bicep` line 1773)

```bicep
enableRbacAuthorization: true
```

**利点:**
- アクセスポリシーではなく Azure RBAC を使用
- より細かい権限制御が可能
- Azure AD との統合

#### 2. Soft Delete (論理削除) (`infra/main.bicep` lines 1774-1775)

```bicep
enableSoftDelete: true
softDeleteRetentionInDays: 7
```

**利点:**
- 誤って削除されたシークレットを7日間復元可能
- セキュリティインシデント時のリカバリが可能

#### 3. ネットワークアクセス制御 (`infra/main.bicep` lines 1766-1792)

```bicep
publicNetworkAccess: enablePrivateNetworking ? 'Disabled' : 'Enabled'
networkAcls: {
  defaultAction: 'Allow'
}

// Private Endpoint (オプション)
privateEndpoints: enablePrivateNetworking
  ? [
      {
        name: 'pep-${keyVaultName}'
        customNetworkInterfaceName: 'nic-${keyVaultName}'
        privateDnsZoneGroup: {
          privateDnsZoneGroupConfigs: [
            { privateDnsZoneResourceId: avmPrivateDnsZones[dnsZoneIndex.keyVault]!.outputs.resourceId }
          ]
        }
        service: 'vault'
        subnetResourceId: virtualNetwork!.outputs.backendSubnetResourceId
      }
    ]
  : []
```

**セキュリティレベル:**
- **基本構成**: パブリックアクセス有効、すべてのネットワークから許可
- **Private Networking 有効時**: Private Endpoint 経由のみアクセス可能

#### 4. 監視とログ記録 (`infra/main.bicep` line 1776)

```bicep
diagnosticSettings: enableMonitoring ? [{ workspaceResourceId: logAnalyticsWorkspaceResourceId }] : []
```

**監視内容:**
- シークレットアクセスログ
- 認証試行ログ
- Key Vault 操作の監査ログ

#### 5. Azure デプロイとの統合 (`infra/main.bicep` lines 1770-1772)

```bicep
enableVaultForDeployment: true
enableVaultForDiskEncryption: true
enableVaultForTemplateDeployment: true
```

**利点:**
- ARM テンプレートデプロイ時に Key Vault のシークレットを参照可能
- VM のディスク暗号化で使用可能

---

## Managed Identity と Key Vault の関係

### なぜ両方が必要なのか？

| リソース | 認証方式 | 理由 |
|---------|---------|------|
| **Cosmos DB** | Managed Identity のみ | Azure RBAC サポート、API キー不要 |
| **Storage Account** | Managed Identity のみ | Azure RBAC サポート、アクセスキー不要 |
| **AI Foundry** | Managed Identity のみ | Azure RBAC サポート、API キー不要 |
| **Key Vault** | Managed Identity | シークレットへのアクセス認証 |
| **Azure AI Search** | API Key (Key Vault に保存) | このデプロイ構成では API キー認証を使用 |

### アーキテクチャ図

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container App (Backend API)                   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │            User Assigned Managed Identity                 │   │
│  │  ClientId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                      │
└────────────────────────────┼──────────────────────────────────────┘
                             │
                             │ Managed Identity で認証
                             ↓
                  ┌──────────────────────┐
                  │   Azure Key Vault    │
                  │  kv-macaedev2vhh3    │
                  │                      │
                  │  Secrets:            │
                  │  • AzureAISearchAPIKey│
                  └──────────────────────┘
                             │
                             │ API Key を返す
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              Backend API が API Key を使用して接続               │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ API Key で認証
                             ↓
                  ┌──────────────────────┐
                  │  Azure AI Search     │
                  │  search-macaedev...  │
                  │                      │
                  │  Indexes:            │
                  │  • Documents         │
                  │  • Embeddings        │
                  └──────────────────────┘
```

---

## Key Vault の SKU とパフォーマンス

### SKU の選択 (`infra/main.bicep` line 1765)

```bicep
sku: enableScalability ? 'premium' : 'standard'
```

| SKU | 用途 | 機能 |
|-----|------|------|
| **Standard** | 通常のシークレット管理 | ソフトウェアベースの暗号化 |
| **Premium** | エンタープライズセキュリティ | HSM (Hardware Security Module) サポート |

**このソリューションでは:**
- `enableScalability` フラグにより SKU を選択
- 基本デプロイでは Standard SKU
- エンタープライズ環境では Premium SKU を推奨

---

## セキュリティベストプラクティス

### 1. シークレットのローテーション

**推奨:**
- Azure AI Search API キーを定期的にローテーション
- Key Vault のバージョン管理機能を使用
- 古いバージョンは7日間の Soft Delete 期間後に完全削除

**実装例:**

```bash
# 新しい API キーを Key Vault に追加（新しいバージョンが作成される）
az keyvault secret set \
  --vault-name kv-macaedev2vhh3 \
  --name AzureAISearchAPIKey \
  --value "<new-api-key>"

# Container App は自動的に新しいバージョンを使用
# （再起動が必要な場合があります）
```

### 2. アクセス監査

**監視すべきログ:**
- `SecretGet`: シークレット取得操作
- `VaultGet`: Key Vault メタデータアクセス
- 失敗した認証試行

**Log Analytics クエリ例:**

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where OperationName == "SecretGet"
| project TimeGenerated, CallerIPAddress, identity_claim_appid_g, ResultSignature
| order by TimeGenerated desc
```

### 3. Managed Identity のスコープ制限

**現在の構成:**
- Managed Identity に "Key Vault Administrator" ロールを付与
- 必要最小限の権限は "Key Vault Secrets User"

**推奨される権限:**

```bicep
roleAssignments: [
  {
    principalId: userAssignedIdentity.outputs.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionIdOrName: 'Key Vault Secrets User'  // 読み取り専用
  }
]
```

### 4. Private Endpoint の使用

**エンタープライズ環境では:**
- `enablePrivateNetworking = true` を設定
- Key Vault への Public アクセスを無効化
- VNet 内からのみアクセス可能

---

## トラブルシューティング

### シークレットが取得できない場合

#### 1. Managed Identity の権限を確認

```bash
# Managed Identity のロール割り当てを確認
az role assignment list \
  --assignee <managed-identity-client-id> \
  --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.KeyVault/vaults/kv-macaedev2vhh3
```

#### 2. Key Vault のアクセスログを確認

```bash
# Key Vault の診断ログを確認
az monitor diagnostic-settings show \
  --resource /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.KeyVault/vaults/kv-macaedev2vhh3
```

#### 3. Container App のシークレット設定を確認

```bash
# Container App のシークレットを確認
az containerapp secret list \
  --name ca-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent
```

#### 4. 環境変数が正しく設定されているか確認

```bash
# Container App の環境変数を確認
az containerapp show \
  --name ca-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent \
  --query "properties.template.containers[0].env"
```

---

## まとめ

### Key Vault の役割

1. **Azure AI Search API Key の安全な保管**
   - Managed Identity で認証できないリソースの API キーを管理
   - 環境変数へのハードコードを回避

2. **Managed Identity との連携**
   - Container Apps が Managed Identity を使用して Key Vault にアクセス
   - Key Vault から取得した API キーを使用して Azure AI Search に接続

3. **セキュリティ機能**
   - RBAC による細かいアクセス制御
   - Soft Delete によるシークレット保護
   - 監査ログによるアクセス追跡
   - オプションで Private Endpoint サポート

### アーキテクチャの利点

| 利点 | 説明 |
|------|------|
| **セキュリティ** | API キーを環境変数やコードにハードコードしない |
| **管理性** | シークレットのローテーションが容易 |
| **監査性** | すべてのアクセスがログに記録される |
| **拡張性** | 新しいシークレットを簡単に追加可能 |
| **ゼロトラスト** | Managed Identity による認証 + RBAC |

### ユーザーの認識の確認

ユーザーの理解は**完全に正しいです**：

✅ **Managed Identity がリソース間の認証に使用されている**
- Cosmos DB: Managed Identity
- Storage Account: Managed Identity
- AI Foundry: Managed Identity

✅ **Key Vault は補完的な役割**
- Azure AI Search API Key のような、Managed Identity で対応できないシークレットを管理
- Managed Identity を使用して Key Vault 自体にアクセス

このアーキテクチャは、Azure のベストプラクティスに従ったセキュアな設計です。

---

## 関連ドキュメント

- [AZURE_ARCHITECTURE_DETAILED.md](./AZURE_ARCHITECTURE_DETAILED.md) - 全体的な Azure アーキテクチャ
- [CONTAINER_APPS_ARCHITECTURE.md](./CONTAINER_APPS_ARCHITECTURE.md) - Container Apps の詳細
- [AGENT_FRAMEWORK_ARCHITECTURE.md](./AGENT_FRAMEWORK_ARCHITECTURE.md) - Agent Framework の実装

---

## 参考リンク

- [Azure Key Vault のベストプラクティス](https://learn.microsoft.com/ja-jp/azure/key-vault/general/best-practices)
- [Azure Key Vault RBAC ガイド](https://learn.microsoft.com/ja-jp/azure/key-vault/general/rbac-guide)
- [Container Apps での Key Vault 参照](https://learn.microsoft.com/ja-jp/azure/container-apps/manage-secrets)
- [Managed Identity の概要](https://learn.microsoft.com/ja-jp/azure/active-directory/managed-identities-azure-resources/overview)
