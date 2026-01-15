# App Service (`app-macaedevfnmlt`) - 詳細解説

## 📋 目次
1. [概要](#概要)
2. [技術スタック](#技術スタック)
3. [アーキテクチャ構成](#アーキテクチャ構成)
4. [主要機能](#主要機能)
5. [バックエンドAPIとの連携](#バックエンドapiとの連携)
6. [デプロイ構成](#デプロイ構成)
7. [セキュリティ設定](#セキュリティ設定)
8. [パフォーマンス最適化](#パフォーマンス最適化)
9. [トラブルシューティング](#トラブルシューティング)

---

## 概要

**App Service (`app-macaedevfnmlt`)** は、このソリューションの**フロントエンド層**を担当するAzureリソースです。

### 🎯 役割

```
┌─────────────────────────────────────────────┐
│         App Service (Frontend)              │
│   https://app-macaedevfnmlt.azurewebsites.net│
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  Multi-Agent Planner UI               │ │
│  │  - チーム選択                         │ │
│  │  - タスク入力                         │ │
│  │  │  - リアルタイム実行監視              │ │
│  │  - 結果表示                           │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  技術: React 18 + TypeScript + Vite        │
│  デザイン: Fluent UI v9                    │
└─────────────────────────────────────────────┘
              ↓ HTTPS
┌─────────────────────────────────────────────┐
│   Backend API (Container Apps)              │
│   https://ca-macaedevfnmlt.....io           │
└─────────────────────────────────────────────┘
```

### ✅ 主な責務

1. **ユーザーインターフェース提供**
   - マルチエージェントプランナーのWebアプリケーション
   - Microsoftの設計言語に基づいたモダンUI

2. **Backend APIとの通信**
   - RESTful API呼び出し (Axios)
   - WebSocketによるリアルタイム通信

3. **ステート管理**
   - ユーザーセッション管理
   - チーム選択状態
   - タスク実行状態

4. **レスポンシブデザイン**
   - デスクトップ、タブレット、モバイル対応

---

## 技術スタック

### 📦 フロントエンドフレームワーク

#### **React 18.3.1**
```json
{
  "react": "^18.3.1",
  "react-dom": "^18.3.1",
  "react-router-dom": "^7.6.0"
}
```

**使用している理由**:
- ✅ コンポーネントベースのアーキテクチャ
- ✅ Virtual DOMによる高速レンダリング
- ✅ 豊富なエコシステム
- ✅ Concurrent Rendering (React 18)

#### **TypeScript 5.8.3**
```json
{
  "typescript": "^5.8.3"
}
```

**使用している理由**:
- ✅ 型安全性による開発時エラー検出
- ✅ IDEの自動補完サポート
- ✅ コードの可読性と保守性向上

#### **Vite 7.1.2**
```json
{
  "vite": "^7.1.2",
  "@vitejs/plugin-react": "^4.5.1"
}
```

**使用している理由**:
- ✅ 超高速な開発サーバー起動 (HMR)
- ✅ 最適化されたプロダクションビルド
- ✅ ESMベースのモジュール解決

### 🎨 UIフレームワーク

#### **Fluent UI v9**
```json
{
  "@fluentui/react-components": "^9.64.0",
  "@fluentui/react-icons": "^2.0.300"
}
```

**採用理由**:
- ✅ Microsoft公式のデザインシステム
- ✅ アクセシビリティ標準準拠 (WCAG 2.1)
- ✅ Azure Portal風の一貫したUX
- ✅ 豊富なコンポーネントライブラリ

**使用コンポーネント例**:
```typescript
- Spinner: ローディングインジケーター
- Button: アクション実行ボタン
- Input: テキスト入力
- Dialog: モーダルダイアログ
- Toast: 通知メッセージ
- Accordion: 折りたたみパネル
```

### 🔗 HTTP通信

#### **Axios 1.11.0**
```json
{
  "axios": "^1.11.0"
}
```

**使用している理由**:
- ✅ Promise ベースの HTTP クライアント
- ✅ リクエスト/レスポンスインターセプター
- ✅ 自動JSONデータ変換
- ✅ タイムアウト設定

**実装例** (src/frontend/src/api/apiClient.tsx):
```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: API_URL,
  timeout: 120000, // 2分
  headers: {
    'Content-Type': 'application/json',
  }
});

// リクエストインターセプター
apiClient.interceptors.request.use(config => {
  config.headers['x-ms-client-principal-id'] = getUserId();
  return config;
});
```

### 📝 Markdownレンダリング

#### **React Markdown**
```json
{
  "react-markdown": "^10.1.0",
  "remark-gfm": "^4.0.1",
  "rehype-prism": "^2.3.3"
}
```

**使用目的**:
- エージェントの応答をMarkdown形式で表示
- コードブロックのシンタックスハイライト
- GitHub Flavored Markdown (GFM) サポート

---

## アーキテクチャ構成

### 📂 ディレクトリ構造

```
src/frontend/
├── src/
│   ├── App.tsx                    # メインアプリケーション
│   ├── index.tsx                  # エントリーポイント
│   │
│   ├── pages/                     # ページコンポーネント
│   │   ├── HomePage.tsx           # ホーム画面
│   │   └── PlanPage.tsx           # プラン実行画面
│   │
│   ├── components/                # UIコンポーネント
│   │   ├── common/                # 共通コンポーネント
│   │   │   ├── TeamSelector.tsx  # チーム選択UI
│   │   │   └── TeamSelected.tsx  # 選択済みチーム表示
│   │   ├── content/               # コンテンツコンポーネント
│   │   │   ├── HomeInput.tsx     # タスク入力フォーム
│   │   │   ├── PlanChat.tsx      # チャットUI
│   │   │   └── streaming/        # ストリーミング表示
│   │   └── errors/                # エラー処理
│   │       └── RAIErrorCard.tsx  # RAIエラー表示
│   │
│   ├── services/                  # ビジネスロジック
│   │   ├── TeamService.tsx       # チーム管理
│   │   ├── TaskService.tsx       # タスク管理
│   │   ├── WebSocketService.tsx  # WebSocket通信
│   │   └── PlanDataService.tsx   # プランデータ管理
│   │
│   ├── hooks/                     # カスタムフック
│   │   ├── useWebSocket.tsx      # WebSocket接続
│   │   ├── useTeamSelection.tsx  # チーム選択
│   │   └── useRAIErrorHandling.tsx # エラーハンドリング
│   │
│   ├── api/                       # API通信層
│   │   ├── apiClient.tsx         # Axiosクライアント
│   │   ├── apiService.tsx        # API呼び出しロジック
│   │   └── config.tsx            # 設定管理
│   │
│   ├── models/                    # TypeScript型定義
│   │   ├── Team.tsx              # チーム型
│   │   ├── plan.tsx              # プラン型
│   │   ├── agentMessage.tsx      # エージェントメッセージ型
│   │   └── enums.tsx             # 列挙型
│   │
│   ├── coral/                     # Coralデザインシステム
│   │   ├── components/           # 共通UIコンポーネント
│   │   └── modules/              # モジュール
│   │
│   └── utils/                     # ユーティリティ
│       ├── errorUtils.tsx        # エラー処理ヘルパー
│       └── agentIconUtils.tsx    # アイコン管理
│
├── public/                        # 静的ファイル
├── package.json                   # 依存関係定義
├── vite.config.ts                 # Vite設定
├── tsconfig.json                  # TypeScript設定
└── Dockerfile                     # コンテナ化設定
```

### 🔄 アプリケーションフロー

#### **1. アプリケーション起動**
```typescript
// src/index.tsx
ReactDOM.createRoot(document.getElementById('root')!)
  .render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
  );
```

#### **2. ルーティング設定**
```typescript
// src/App.tsx
<Router>
  <Routes>
    <Route path="/" element={<HomePage />} />
    <Route path="/plan/:planId" element={<PlanPage />} />
    <Route path="*" element={<Navigate to="/" replace />} />
  </Routes>
</Router>
```

**ルート定義**:
- `/` - ホーム画面 (タスク一覧とチーム選択)
- `/plan/:planId` - プラン実行画面 (リアルタイムエージェント実行)
- `*` - 404リダイレクト

---

## 主要機能

### 1. ホーム画面 (`HomePage.tsx`)

#### **機能概要**
```
┌────────────────────────────────────────────┐
│  ┌──────────────────┐  ┌────────────────┐ │
│  │  サイドバー       │  │  メインエリア   │ │
│  │  ・チーム選択     │  │  ・タスク入力  │ │
│  │  ・タスク履歴     │  │  ・新規作成    │ │
│  └──────────────────┘  └────────────────┘ │
└────────────────────────────────────────────┘
```

#### **実装詳細**

**チーム初期化処理** (HomePage.tsx:28-92):
```typescript
useEffect(() => {
  const initTeam = async () => {
    setIsLoadingTeam(true);

    try {
      // 1. バックエンドでチーム初期化 (約20秒)
      const initResponse = await TeamService.initializeTeam();

      // 2. 初期化成功
      if (initResponse.data?.status === 'Request started successfully') {
        // 3. ユーザーのチーム一覧を取得
        const teams = await TeamService.getUserTeams();

        // 4. 初期化されたチームを選択
        const initializedTeam = teams.find(
          team => team.team_id === initResponse.data?.team_id
        );

        if (initializedTeam) {
          setSelectedTeam(initializedTeam);
          TeamService.storageTeam(initializedTeam);

          showToast(
            `${initializedTeam.name} team initialized successfully`,
            "success"
          );
        }
      }
      // 5. チーム未設定の場合
      else if (initResponse.data?.requires_team_upload) {
        setSelectedTeam(null);
        showToast(
          "Please upload a team configuration file to get started.",
          "info"
        );
      }

    } catch (error) {
      console.error('Error initializing team:', error);
      showToast("Team initialization failed.", "info");
      setSelectedTeam(null);
    } finally {
      setIsLoadingTeam(false);
    }
  };

  initTeam();
}, []);
```

**タスク入力エリア**:
```typescript
<HomeInput
  selectedTeam={selectedTeam}
  setReloadLeftList={setReloadLeftList}
/>
```

機能:
- ✅ マルチラインテキストエリア (最大5000文字)
- ✅ 送信ボタン
- ✅ チーム未選択時は無効化

**タスク履歴表示** (左サイドバー):
```typescript
<PlanPanelLeft
  selectedTeam={selectedTeam}
  reload={reloadLeftList}
  setReload={setReloadLeftList}
/>
```

機能:
- ✅ 過去に実行したタスクの一覧
- ✅ タスククリックで詳細画面へ遷移
- ✅ 実行ステータス表示 (実行中、完了、エラー)

### 2. プラン実行画面 (`PlanPage.tsx`)

#### **機能概要**
```
┌────────────────────────────────────────────┐
│  ユーザー: "新入社員のオンボーディングプランを作成" │
└────────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────────┐
│  🤖 HR Manager Agent                        │
│  "オンボーディングプランを作成しています..."  │
└────────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────────┐
│  🤖 IT Setup Agent                          │
│  "PCとアカウントをセットアップしています..."  │
└────────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────────┐
│  ✅ プラン完了                              │
│  - Day 1: IT Setup                         │
│  - Day 2-5: Training                       │
│  - Week 2: Team Integration                │
└────────────────────────────────────────────┘
```

#### **リアルタイムストリーミング**

**WebSocket接続** (hooks/useWebSocket.tsx):
```typescript
const useWebSocket = () => {
  const socketRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const wsUrl = `${API_URL.replace('http', 'ws')}/ws`;
    socketRef.current = new WebSocket(wsUrl);

    socketRef.current.onopen = () => {
      console.log('WebSocket connected');
    };

    socketRef.current.onmessage = (event) => {
      const data = JSON.parse(event.data);

      // エージェントメッセージを処理
      if (data.type === 'agent_message') {
        // UIを更新
        updateAgentMessage(data);
      }
    };

    return () => {
      socketRef.current?.close();
    };
  }, []);
};
```

**ストリーミング表示コンポーネント**:
- `StreamingUserPlan.tsx` - ユーザー入力表示
- `StreamingAgentMessage.tsx` - エージェントメッセージ表示
- `StreamingPlanResponse.tsx` - 最終結果表示
- `StreamingPlanState.tsx` - 実行状態表示

### 3. チーム管理機能

#### **チーム選択UI** (components/common/TeamSelector.tsx)

```typescript
interface TeamConfig {
  team_id: string;
  name: string;
  description: string;
  agents: Agent[];
  created_at: string;
}
```

**機能**:
1. ✅ 利用可能なチーム一覧の表示
2. ✅ チーム詳細の表示 (エージェント数、説明)
3. ✅ チームの切り替え
4. ✅ カスタムチームのアップロード

**利用可能なチーム例**:
```
1. HR Employee Onboarding Team
   - 5 agents
   - 従業員オンボーディング自動化

2. RFP Evaluation Team
   - 4 agents
   - RFP審査と評価自動化

3. Retail Customer Satisfaction Team
   - 6 agents
   - カスタマーサービス自動化

4. Marketing Press Release Team
   - 4 agents
   - プレスリリース作成自動化

5. Contract Compliance Review Team
   - 5 agents
   - 契約審査自動化
```

### 4. エラーハンドリング

#### **RAI (Responsible AI) エラー処理**

**RAIエラーカード** (components/errors/RAIErrorCard.tsx):
```typescript
interface RAIError {
  type: 'content_filter' | 'rate_limit' | 'quota_exceeded';
  message: string;
  details?: string;
}
```

**表示されるエラー**:
1. **コンテンツフィルタリング**
   ```
   ⚠️ コンテンツポリシー違反
   入力内容がAzure OpenAIのコンテンツフィルターに
   引っかかりました。表現を変えて再試行してください。
   ```

2. **レート制限**
   ```
   ⏱️ レート制限に達しました
   短時間に多くのリクエストが送信されています。
   しばらく待ってから再試行してください。
   ```

3. **クォータ超過**
   ```
   📊 クォータ超過
   月間のトークン使用量が上限に達しました。
   管理者に連絡してください。
   ```

---

## バックエンドAPIとの連携

### 🔌 API エンドポイント

#### **1. チーム管理API**

```typescript
// チーム初期化
POST /api/init_team
Request: { user_id: string }
Response: {
  status: "Request started successfully",
  team_id: string,
  message: string
}

// ユーザーのチーム一覧取得
GET /api/teams
Response: TeamConfig[]

// チームアップロード
POST /api/upload_team
Request: FormData (team_config.json)
Response: { success: boolean, team_id: string }
```

#### **2. タスク管理API**

```typescript
// 新規タスク作成
POST /api/plans
Request: {
  user_input: string,
  team_id: string,
  user_id: string
}
Response: {
  plan_id: string,
  status: "in_progress"
}

// タスク一覧取得
GET /api/plans?user_id={user_id}
Response: Plan[]

// タスク詳細取得
GET /api/plans/{plan_id}
Response: Plan
```

#### **3. WebSocket通信**

```typescript
// WebSocket接続
WS /ws

// サーバーからのメッセージ
{
  type: "agent_message",
  plan_id: string,
  agent_name: string,
  message: string,
  status: "in_progress" | "completed" | "error"
}
```

### 📡 API呼び出しフロー

```typescript
// services/apiService.tsx

export class ApiService {
  // チーム初期化
  static async initializeTeam(userId: string) {
    const response = await apiClient.post('/init_team', {
      user_id: userId
    });
    return response.data;
  }

  // タスク作成
  static async createPlan(userInput: string, teamId: string) {
    const response = await apiClient.post('/plans', {
      user_input: userInput,
      team_id: teamId,
      user_id: getUserId()
    });
    return response.data;
  }

  // タスク一覧取得
  static async getPlans() {
    const response = await apiClient.get('/plans', {
      params: { user_id: getUserId() }
    });
    return response.data;
  }
}
```

---

## デプロイ構成

### 🐳 Dockerコンテナ化

#### **Dockerfile** (src/frontend/Dockerfile)

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine

# Nginxカスタム設定
COPY nginx.conf /etc/nginx/nginx.conf

# ビルド成果物をコピー
COPY --from=builder /app/dist /usr/share/nginx/html

# 環境変数を注入するスクリプト
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

EXPOSE 3000

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

**マルチステージビルドの利点**:
- ✅ 最終イメージサイズの削減 (約50MB)
- ✅ ビルド依存関係の除外
- ✅ セキュリティ向上

#### **Nginx設定** (nginx.conf)

```nginx
server {
    listen 3000;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # SPA用のルーティング設定
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API呼び出しのプロキシ
    location /api {
        proxy_pass ${BACKEND_API_URL};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocketプロキシ
    location /ws {
        proxy_pass ${BACKEND_API_URL};
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # セキュリティヘッダー
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip圧縮
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
}
```

### 🚀 Azure App Service設定

#### **インフラ構成** (infra/main.bicep:1507-1542)

```bicep
module webSite 'modules/web-sites.bicep' = {
  name: 'app-${solutionSuffix}'
  params: {
    name: 'app-${solutionSuffix}'
    location: location
    kind: 'app,linux,container'
    serverFarmResourceId: webServerFarm.outputs.resourceId

    siteConfig: {
      linuxFxVersion: 'DOCKER|${containerRegistry}/${imageName}:${imageTag}'
      minTlsVersion: '1.2'
    }

    configs: [
      {
        name: 'appsettings'
        properties: {
          DOCKER_REGISTRY_SERVER_URL: 'https://${containerRegistry}'
          WEBSITES_PORT: '3000'
          WEBSITES_CONTAINER_START_TIME_LIMIT: '1800' // 30分
          BACKEND_API_URL: 'https://${containerApp.outputs.fqdn}'
          AUTH_ENABLED: 'false'
        }
        applicationInsightResourceId: applicationInsights.outputs.resourceId
      }
    ]

    diagnosticSettings: [
      { workspaceResourceId: logAnalyticsWorkspaceResourceId }
    ]

    vnetRouteAllEnabled: enablePrivateNetworking
    publicNetworkAccess: 'Enabled'
    e2eEncryptionEnabled: true
  }
}
```

#### **環境変数**

| 変数名 | 説明 | 値 |
|--------|------|---|
| `DOCKER_REGISTRY_SERVER_URL` | コンテナレジストリURL | `https://biabcontainerreg.azurecr.io` |
| `WEBSITES_PORT` | コンテナのポート番号 | `3000` |
| `WEBSITES_CONTAINER_START_TIME_LIMIT` | 起動タイムアウト | `1800` (30分) |
| `BACKEND_API_URL` | Backend APIのURL | `https://ca-macaedev....io` |
| `AUTH_ENABLED` | 認証の有効/無効 | `false` (デフォルト) |

#### **App Service Plan設定**

```bicep
SKU: B3 (Basic) または P1v4 (Premium)
OS: Linux
容量: 1インスタンス (デフォルト) または 3インスタンス

リソース (B3):
- CPU: 2コア
- メモリ: 3.5GB
- ストレージ: 10GB

リソース (P1v4):
- CPU: 2コア
- メモリ: 8GB
- ストレージ: 250GB
```

---

## セキュリティ設定

### 🔒 セキュリティ機能

#### **1. HTTPS強制**
```bicep
properties: {
  httpsOnly: true
  clientAffinityEnabled: false
}
```

すべてのHTTP接続は自動的にHTTPSにリダイレクトされます。

#### **2. TLS設定**
```bicep
siteConfig: {
  minTlsVersion: '1.2'
  ftpsState: 'Disabled'
}
```

TLS 1.2以上のみ許可、FTPSは無効化されています。

#### **3. セキュリティヘッダー** (Nginx設定)

```nginx
# クリックジャッキング対策
add_header X-Frame-Options "SAMEORIGIN";

# MIMEタイプスニッフィング対策
add_header X-Content-Type-Options "nosniff";

# XSS対策
add_header X-XSS-Protection "1; mode=block";

# コンテンツセキュリティポリシー
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
```

#### **4. 認証 (オプション)**

**Azure App Service認証** (Easy Auth):
```typescript
// api/config.tsx:51-75
export async function getUserInfo(): Promise<UserInfo> {
  try {
    const response = await fetch("/.auth/me");
    if (!response.ok) {
      console.log("No identity provider found.");
      return {} as UserInfo;
    }

    const payload = await response.json();
    const userInfo: UserInfo = {
      user_id: payload[0].user_claims?.find(
        claim => claim.typ === 'objectidentifier'
      )?.val || '',
      user_email: payload[0].user_id || "",
      user_first_last_name: payload[0].user_claims?.find(
        claim => claim.typ === 'name'
      )?.val || ""
    };

    return userInfo;
  } catch (e) {
    return {} as UserInfo;
  }
}
```

有効化すると、Azure ADでの認証が必須になります。

#### **5. CORS設定**

```bicep
corsPolicy: {
  allowedOrigins: [
    'https://app-${solutionSuffix}.azurewebsites.net'
  ]
  allowedMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
  allowCredentials: true
}
```

フロントエンドのドメインからのみアクセス可能。

---

## パフォーマンス最適化

### ⚡ 最適化手法

#### **1. コード分割** (Code Splitting)

```typescript
// React Router lazy loading
const HomePage = lazy(() => import('./pages/HomePage'));
const PlanPage = lazy(() => import('./pages/PlanPage'));

<Suspense fallback={<Spinner />}>
  <Routes>
    <Route path="/" element={<HomePage />} />
    <Route path="/plan/:planId" element={<PlanPage />} />
  </Routes>
</Suspense>
```

**効果**:
- 初期ロード時間の短縮
- 不要なコードの遅延ロード

#### **2. バンドル最適化** (Vite)

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'fluent-ui': ['@fluentui/react-components'],
          'routing': ['react-router-dom']
        }
      }
    },
    minify: 'terser',
    sourcemap: false
  }
});
```

**効果**:
- ベンダーコードの分離
- ブラウザキャッシュの活用
- 並列ダウンロード

#### **3. 画像最適化**

```typescript
// 遅延読み込み
<img
  src={imageSrc}
  loading="lazy"
  alt="Agent Icon"
/>

// WebP形式の使用
<picture>
  <source srcSet="image.webp" type="image/webp" />
  <img src="image.png" alt="..." />
</picture>
```

#### **4. メモ化** (Memoization)

```typescript
// React.memo でコンポーネントをメモ化
export const AgentMessage = React.memo(({ message }) => {
  return <div>{message}</div>;
});

// useMemo でexpensiveな計算をキャッシュ
const filteredAgents = useMemo(() => {
  return agents.filter(agent => agent.status === 'active');
}, [agents]);

// useCallback でコールバックをメモ化
const handleSubmit = useCallback(() => {
  submitTask(userInput);
}, [userInput]);
```

#### **5. Gzip圧縮**

Nginx設定で有効化:
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
```

**効果**:
- 転送量を70-80%削減
- ロード時間の短縮

### 📊 パフォーマンスメトリクス

**目標値**:
```
First Contentful Paint (FCP): < 1.8秒
Largest Contentful Paint (LCP): < 2.5秒
Time to Interactive (TTI): < 3.8秒
Total Blocking Time (TBT): < 200ms
Cumulative Layout Shift (CLS): < 0.1
```

**測定方法**:
```typescript
// src/reportWebVitals.ts
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  // Application Insightsに送信
  console.log(metric);
}

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);
```

---

## トラブルシューティング

### 🔧 よくある問題と解決策

#### **1. "No team selected" エラー**

**原因**:
- Post-deploymentスクリプト未実行
- チーム設定がアップロードされていない

**解決策**:
```bash
# プロジェクトルートで実行
bash infra/scripts/selecting_team_config_and_data.sh
```

#### **2. Backend APIに接続できない**

**原因**:
- Backend API URLの設定ミス
- CORS設定の問題

**確認方法**:
```bash
# App Serviceの環境変数を確認
az webapp config appsettings list \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent

# Backend APIの稼働確認
curl https://ca-macaedevfnmlt....io/health
```

**解決策**:
```bash
# 環境変数を再設定
az webapp config appsettings set \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent \
  --settings BACKEND_API_URL="https://ca-macaedevfnmlt....io"
```

#### **3. コンテナ起動が遅い**

**原因**:
- イメージサイズが大きい
- 起動スクリプトが重い

**確認方法**:
```bash
# App Serviceログを確認
az webapp log tail \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent
```

**解決策**:
```bash
# タイムアウトを延長
az webapp config appsettings set \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent \
  --settings WEBSITES_CONTAINER_START_TIME_LIMIT="1800"
```

#### **4. WebSocket接続エラー**

**原因**:
- WebSocket URLの設定ミス
- App Serviceのタイムアウト設定

**確認方法**:
```javascript
// ブラウザコンソールで確認
console.log('WebSocket URL:', wsUrl);
```

**解決策**:
```bash
# App Serviceの詳細設定
az webapp config set \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent \
  --web-sockets-enabled true
```

#### **5. 画面が真っ白になる**

**原因**:
- JavaScriptエラー
- ビルド成果物の破損

**確認方法**:
```bash
# ブラウザの開発者ツールでエラー確認
F12 → Console タブ
```

**解決策**:
```bash
# App Serviceを再起動
az webapp restart \
  --name app-macaedevfnmlt \
  --resource-group azure-foundry-multi-agent

# または、イメージを再デプロイ
azd deploy
```

---

## まとめ

### ✅ App Serviceの役割

**App Service (`app-macaedevfnmlt`)** は:

1. **ユーザーインターフェース**
   - React 18 + TypeScript製のSPA
   - Fluent UI v9によるモダンデザイン
   - レスポンシブ対応

2. **Backend APIとの連携**
   - RESTful API (Axios)
   - WebSocketによるリアルタイム通信
   - エージェント実行状態の可視化

3. **セキュアな運用**
   - HTTPS強制、TLS 1.2以上
   - セキュリティヘッダー
   - Azure AD認証対応

4. **高パフォーマンス**
   - コード分割、バンドル最適化
   - Gzip圧縮
   - Nginxによる高速配信

5. **スケーラブル**
   - Dockerコンテナ化
   - App Service Planによる柔軟なスケーリング
   - Application Insightsによる監視

### 🎯 次のステップ

追加で知りたいことはありますか？例えば:

1. **カスタマイズ方法**
   - UIのカスタマイズ
   - 新しいページの追加
   - 新しいチームの作成

2. **ローカル開発環境のセットアップ**
   - 開発サーバーの起動
   - デバッグ方法
   - ホットリロード

3. **デプロイメント**
   - CI/CDパイプライン
   - カスタムドメインの設定
   - SSL証明書の設定

---

**作成日**: 2026-01-15
**対象リソース**: `app-macaedevfnmlt` (East US 2)
**関連ドキュメント**: `AZURE_ARCHITECTURE_DETAILED.md`
