# Azure VM クォータ増加リクエストガイド

## 問題
サブスクリプションで Basic VMs のクォータが 0 に設定されているため、デプロイが失敗しています。

## 解決手順

### 1. Azure Portal でクォータ増加をリクエスト

1. **Azure Portal にアクセス**: https://portal.azure.com

2. **Quotas を検索**
   - 上部の検索バーで "Quotas" と入力
   - "Quotas" サービスを選択

3. **Compute のクォータを選択**
   - "Compute" を選択
   - "Request increase" をクリック

4. **リクエスト詳細を入力**
   - **Subscription**: `generative-ai-playground (b89a7aff-ce92-48cd-96ce-7b60673dfa93)`
   - **Quota type**: `Compute-VM (cores-vCPUs) subscription limit increases`
   - **Region**: `Japan East` (または使用したいリージョン)
   - **Quota name**: `Standard BS Family vCPUs` (Basic VM用)
   - **New limit**: `10` (最低1必要、余裕を持って10を推奨)

5. **Submit** をクリック

### 2. 承認を待つ
- 通常、数時間〜1営業日で承認されます
- 緊急の場合は、サポートチケットを作成して優先対応を依頼できます

### 3. 承認後に再デプロイ

```bash
azd env set AZURE_LOCATION japaneast
azd up
```

---

## 代替案: VMを使わないデプロイメント

このソリューションは通常、Private Networking を有効にした場合のみ VM が必要です。
デフォルトでは VM は不要なはずですが、検証エラーが発生しています。

残念ながら、このクォータ問題を回避する簡単な方法はありません。
**クォータ増加リクエストが最も確実な解決策です。**

---

## リージョン選択のガイダンス

クォータリクエスト時は以下のリージョンを推奨（Azure OpenAI も利用可能）:

1. **Japan East (japaneast)** - 日本から最も近い ⭐推奨
2. **East US 2 (eastus2)** - 安定性が高い
3. **Australia East (australiaeast)** - アジア太平洋圏
4. **UK South (uksouth)** - ヨーロッパ圏

注意: `Sweden Central` はこのテンプレートでサポートされていません。
