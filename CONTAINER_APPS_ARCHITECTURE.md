# Container Apps アーキテクチャ詳細解説

## 📋 目次
1. [2つのContainer Appsの違い](#2つのcontainer-appsの違い)
2. [リクエストフロー全体像](#リクエストフロー全体像)
3. [Backend API (ca-macaedevfnmlt)](#backend-api-ca-macaedevfnmlt)
4. [MCP Server (ca-mcp-macaedevfnmlt)](#mcp-server-ca-mcp-macaedevfnmlt)
5. [なぜ2つに分離されているのか](#なぜ2つに分離されているのか)
6. [通信プロトコル](#通信プロトコル)
7. [具体的な実行例](#具体的な実行例)

---

## 2つのContainer Appsの違い

### 📊 比較表

| 項目 | **ca-macaedevfnmlt**<br>(Backend API) | **ca-mcp-macaedevfnmlt**<br>(MCP Server) |
|------|--------------------------------------|----------------------------------------|
| **役割** | マルチエージェントオーケストレーション | 外部ツール・機能の提供 |
| **技術** | Python FastAPI | Python FastMCP |
| **ポート** | 8000 | 9000 |
| **CPU** | 2.0 | 2.0 |
| **メモリ** | 4.0Gi | 4.0Gi |
| **主な責務** | ・エージェント管理<br>・タスク分解<br>・実行オーケストレーション<br>・結果集約 | ・HR機能<br>・データアクセス<br>・ビジネスロジック<br>・外部システム統合 |
| **接続先** | ・AI Foundry<br>・Cosmos DB<br>・AI Search<br>・MCP Server | ・(特定のデータソース)<br>・(ビジネスシステム) |
| **App Serviceから** | 直接呼び出される | Backend経由で呼び出される |

---

## リクエストフロー全体像

### 🔄 エンドツーエンドのフロー

```
┌─────────────────────────────────────────────────────────────┐
│  1. ユーザー                                                  │
│     "新入社員のオンボーディングプランを作成"                  │
└──────────────────────┬──────────────────────────────────────┘
                       ↓ HTTPSリクエスト
┌─────────────────────────────────────────────────────────────┐
│  2. App Service (フロントエンド)                              │
│     app-macaedevfnmlt.azurewebsites.net                      │
│                                                              │
│     POST /api/plans                                          │
│     {                                                        │
│       "user_input": "新入社員の...",                         │
│       "team_id": "hr-team-id"                                │
│     }                                                        │
└──────────────────────┬──────────────────────────────────────┘
                       ↓ HTTPS API呼び出し
┌─────────────────────────────────────────────────────────────┐
│  3. Backend API (Container Apps)                             │
│     ca-macaedevfnmlt.....eastus2.azurecontainerapps.io       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ FastAPI Application                                    │ │
│  │ ├─ Router: /api/plans                                  │ │
│  │ ├─ OrchestrationManager                                │ │
│  │ │   ├─ タスク分析                                      │ │
│  │ │   ├─ エージェント選択                                │ │
│  │ │   └─ 実行計画作成                                    │ │
│  │ │                                                       │ │
│  │ └─ FoundryService                                      │ │
│  │     └─ Azure AI Foundry APIを呼び出し                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│     ↓ AI Foundryへ                    ↓ MCPサーバーへ      │
│                                                              │
└──────┬───────────────────────────────────────┬──────────────┘
       ↓                                       ↓
┌──────────────────┐              ┌──────────────────────────┐
│ 4. AI Foundry    │              │ 5. MCP Server            │
│                  │              │    (Container Apps)      │
│ GPT-4.1モデルで  │              │    ca-mcp-macaedev...    │
│ エージェント実行 │◄─────ツール─┤                          │
│                  │    呼び出し  │  ┌────────────────────┐ │
│ "HR Manager"が  │              │  │ FastMCP Server     │ │
│ MCPツールを要求 │──────────────►│  │                    │ │
│                  │              │  │ ├─ HRService      │ │
│                  │              │  │ │   ├─ employee_  │ │
│                  │              │  │ │   │   onboarding│ │
│                  │              │  │ │   └─ background_│ │
│                  │              │  │ │       check     │ │
│                  │              │  │ │                 │ │
│                  │              │  │ ├─ TechSupport   │ │
│                  │              │  │ │   ├─ configure_ │ │
│                  │              │  │ │   │   laptop    │ │
│                  │              │  │ │   └─ create_    │ │
│                  │              │  │ │       accounts  │ │
│                  │              │  │ │                 │ │
│                  │              │  │ └─ Marketing     │ │
│                  │              │  │     └─ ...       │ │
│                  │              │  └────────────────────┘ │
└──────────────────┘              └──────────────────────────┘
       ↓                                       ↓
       │                                       │
       └───────────────┬───────────────────────┘
                       ↓ 結果を返す
┌─────────────────────────────────────────────────────────────┐
│  6. Backend API                                              │
│     - 各エージェントの結果を集約                              │
│     - 最終プランを生成                                        │
│     - Cosmos DBに保存                                        │
└──────────────────────┬──────────────────────────────────────┘
                       ↓ WebSocketでストリーミング
┌─────────────────────────────────────────────────────────────┐
│  7. App Service (フロントエンド)                              │
│     - リアルタイムで進捗を表示                                │
│     - 最終結果をMarkdownでレンダリング                        │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  8. ユーザー                                                  │
│     完成したオンボーディングプランを確認                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Backend API (ca-macaedevfnmlt)

### 🎯 役割: **マルチエージェントのオーケストレーター**

Backend APIは、**指揮者（Conductor）** のような役割を果たします。

### 📂 技術スタック

```python
# src/backend/app.py
FastAPI アプリケーション
├─ CORS Middleware
├─ Health Check Middleware
├─ Application Insights統合
└─ v4 Router
    ├─ /api/plans (タスク管理)
    ├─ /api/teams (チーム管理)
    ├─ /api/init_team (チーム初期化)
    └─ /ws (WebSocket)
```

### 🔧 主要コンポーネント

#### **1. OrchestrationManager** (オーケストレーションマネージャー)
**ファイル**: `src/backend/v4/orchestration/orchestration_manager.py`

```python
class OrchestrationManager:
    """
    マルチエージェント実行のオーケストレーション
    """
    async def run_plan(self, user_input: str, team_config: dict):
        # 1. タスク分析
        plan = await self.analyze_task(user_input)

        # 2. エージェント選択
        agents = self.select_agents(plan, team_config)

        # 3. 実行順序の決定
        execution_order = self.determine_execution_order(agents)

        # 4. エージェントを順次実行
        for agent in execution_order:
            result = await self.execute_agent(agent)
            plan.add_result(result)

        # 5. 結果の検証と集約
        final_result = await self.validate_and_aggregate(plan)

        return final_result
```

#### **2. FoundryService** (AI Foundry連携サービス)
**ファイル**: `src/backend/v4/common/services/foundry_service.py`

```python
class FoundryService:
    """
    Azure AI Foundryとの通信を担当
    """
    async def create_agent(self, agent_config: dict):
        # AI Foundryでエージェントを作成
        return await self.client.agents.create(
            name=agent_config["name"],
            instructions=agent_config["instructions"],
            model=agent_config["model"],
            tools=agent_config["tools"]
        )

    async def run_agent(self, agent_id: str, user_message: str):
        # エージェントを実行
        run = await self.client.agents.create_and_run(
            agent_id=agent_id,
            messages=[{"role": "user", "content": user_message}]
        )

        # 結果を取得
        return await self.get_run_result(run.id)
```

#### **3. MCPService** (MCP Server連携サービス)
**ファイル**: `src/backend/v4/common/services/mcp_service.py`

```python
class MCPService(BaseAPIService):
    """
    MCP Serverとの通信を担当
    """
    def __init__(self, base_url: str):
        # MCP Server のエンドポイント
        # 例: http://ca-mcp-macaedevfnmlt.....io
        super().__init__(base_url)

    async def health(self):
        # MCP Serverのヘルスチェック
        return await self.get_json("health")

    async def invoke_tool(self, tool_name: str, payload: dict):
        # MCPツールを呼び出し
        # 例: tool_name = "employee_onboarding_blueprint_flat"
        return await self.post_json(f"tools/{tool_name}", json=payload)
```

**使用例**:
```python
# Backend APIがMCPツールを呼び出す
mcp_service = MCPService.from_app_config()

# HR オンボーディングブループリントを取得
blueprint = await mcp_service.invoke_tool(
    tool_name="employee_onboarding_blueprint_flat",
    payload={
        "employee_name": "田中太郎",
        "start_date": "2026-02-01",
        "role": "Software Engineer"
    }
)
```

### 🔌 Backend APIのエンドポイント

#### **タスク管理API**
```python
# 新規タスク作成
POST /api/plans
Request:
{
  "user_input": "新入社員のオンボーディングプランを作成",
  "team_id": "00000000-0000-0000-0000-000000000001",
  "user_id": "user-principal-id"
}

Response:
{
  "plan_id": "uuid",
  "status": "in_progress",
  "created_at": "2026-01-15T10:00:00Z"
}

# タスク一覧取得
GET /api/plans?user_id={user_id}

# タスク詳細取得
GET /api/plans/{plan_id}
```

#### **チーム管理API**
```python
# チーム初期化
POST /api/init_team
Request:
{
  "user_id": "user-principal-id"
}

Response:
{
  "status": "Request started successfully",
  "team_id": "00000000-0000-0000-0000-000000000001",
  "message": "Team initialized successfully"
}

# ユーザーのチーム一覧取得
GET /api/teams?user_id={user_id}

# チームアップロード
POST /api/upload_team
Request: FormData (team_config.json)
```

#### **WebSocket**
```python
# WebSocket接続
WS /ws

# サーバーからのメッセージ例
{
  "type": "agent_message",
  "plan_id": "uuid",
  "agent_name": "HR Manager",
  "message": "オンボーディングプランを作成中...",
  "status": "in_progress",
  "progress": 30
}

{
  "type": "agent_message",
  "plan_id": "uuid",
  "agent_name": "IT Setup Agent",
  "message": "PCとアカウントをセットアップ中...",
  "status": "in_progress",
  "progress": 60
}

{
  "type": "plan_completed",
  "plan_id": "uuid",
  "result": "...",
  "status": "completed"
}
```

### 💾 データ保存

```python
# Cosmos DBに保存されるデータ
{
  "id": "plan-uuid",
  "session_id": "session-uuid",
  "user_id": "user-principal-id",
  "team_id": "00000000-0000-0000-0000-000000000001",
  "user_input": "新入社員のオンボーディングプランを作成",
  "status": "completed",
  "created_at": "2026-01-15T10:00:00Z",
  "completed_at": "2026-01-15T10:05:30Z",
  "agents_executed": [
    {
      "name": "HR Manager",
      "status": "completed",
      "result": "..."
    },
    {
      "name": "IT Setup Agent",
      "status": "completed",
      "result": "..."
    }
  ],
  "final_result": "完成したオンボーディングプラン..."
}
```

---

## MCP Server (ca-mcp-macaedevfnmlt)

### 🎯 役割: **ツールとビジネスロジックの提供者**

MCP Serverは、**専門家集団（Subject Matter Experts）** のような役割を果たします。

### 📂 技術スタック

```python
# src/mcp_server/mcp_server.py
FastMCP Server
├─ HRService (HR関連ツール)
├─ TechSupportService (IT関連ツール)
├─ MarketingService (マーケティング関連ツール)
├─ ProductService (製品関連ツール)
└─ DataToolService (データアクセスツール)
```

### 🔧 MCP (Model Context Protocol) とは？

**MCP** は、AIエージェントが外部ツールを呼び出すための標準プロトコルです。

```
┌─────────────────────────────────────────┐
│  AI Agent (in Azure AI Foundry)         │
│  "新入社員の背景調査を開始してください"  │
└──────────────────┬──────────────────────┘
                   ↓ MCPプロトコル
┌─────────────────────────────────────────┐
│  MCP Server                             │
│  "initiate_background_check" ツール      │
│  を実行します                            │
└──────────────────┬──────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│  ビジネスシステム / データベース        │
│  実際の背景調査プロセスを開始           │
└─────────────────────────────────────────┘
```

### 🛠️ 提供されているツール (Services)

#### **1. HRService** (人事関連ツール)
**ファイル**: `src/mcp_server/services/hr_service.py`

```python
@mcp.tool(tags={"HR"})
async def employee_onboarding_blueprint_flat(
    employee_name: str,
    start_date: str,
    role: str
) -> dict:
    """
    新入社員オンボーディングのブループリントを返す

    返り値の例:
    {
      "version": "1.0",
      "intent": "employee_onboarding",
      "employee": {
        "name": "田中太郎",
        "start_date": "2026-02-01",
        "role": "Software Engineer"
      },
      "steps": [
        {
          "id": "bg_check",
          "domain": "HR",
          "action": "背景調査を開始",
          "tool": "initiate_background_check",
          "required": true
        },
        {
          "id": "configure_laptop",
          "domain": "TECH_SUPPORT",
          "action": "PCのセットアップ",
          "tool": "configure_laptop",
          "required": true
        },
        ...
      ]
    }
    """

@mcp.tool(tags={"HR"})
async def initiate_background_check(
    employee_name: str,
    check_type: str = "standard"
) -> dict:
    """
    背景調査を開始

    実際にはビジネスシステムAPIを呼び出す
    """
    return {
      "status": "initiated",
      "employee": employee_name,
      "check_id": "BGC-2026-001",
      "estimated_completion": "2026-01-20"
    }

@mcp.tool(tags={"HR"})
async def schedule_orientation_session(
    employee_name: str,
    date: str
) -> dict:
    """
    オリエンテーションセッションをスケジュール
    """
    return {
      "status": "scheduled",
      "employee": employee_name,
      "session_date": date,
      "location": "Conference Room A"
    }
```

#### **2. TechSupportService** (IT関連ツール)
**ファイル**: `src/mcp_server/services/tech_support_service.py`

```python
@mcp.tool(tags={"TECH_SUPPORT"})
async def configure_laptop(
    employee_name: str,
    laptop_model: str = "Dell Latitude 5540"
) -> dict:
    """
    ラップトップを設定
    """
    return {
      "status": "configured",
      "employee": employee_name,
      "laptop_model": laptop_model,
      "asset_tag": "LAPTOP-2026-042",
      "os": "Windows 11 Pro"
    }

@mcp.tool(tags={"TECH_SUPPORT"})
async def create_system_accounts(
    employee_name: str,
    role: str
) -> dict:
    """
    システムアカウントを作成
    """
    return {
      "status": "created",
      "employee": employee_name,
      "accounts": [
        {
          "system": "Active Directory",
          "username": "t.tanaka",
          "email": "t.tanaka@company.com"
        },
        {
          "system": "Office 365",
          "username": "t.tanaka@company.com"
        }
      ]
    }
```

#### **3. MarketingService** (マーケティング関連ツール)
**ファイル**: `src/mcp_server/services/marketing_service.py`

```python
@mcp.tool(tags={"MARKETING"})
async def generate_press_release_outline(
    topic: str,
    target_audience: str
) -> dict:
    """
    プレスリリースのアウトラインを生成
    """
    return {
      "outline": {
        "headline": "...",
        "subheadline": "...",
        "body": [
          "Introduction",
          "Key Points",
          "Quotes",
          "Call to Action"
        ]
      }
    }
```

#### **4. DataToolService** (データアクセスツール)
**ファイル**: `src/mcp_server/services/data_tool_service.py`

```python
@mcp.tool(tags={"DATA"})
async def query_customer_data(
    customer_id: str
) -> dict:
    """
    顧客データを取得
    (Azure AI Searchやデータベースにアクセス)
    """
    return {
      "customer_id": customer_id,
      "name": "...",
      "email": "...",
      "orders": [...]
    }
```

### 🌐 MCPサーバーのエンドポイント

```python
# ヘルスチェック
GET /health
Response:
{
  "status": "healthy",
  "version": "1.0",
  "services": ["HR", "TECH_SUPPORT", "MARKETING", "DATA"]
}

# ツール一覧取得
GET /tools
Response:
{
  "tools": [
    {
      "name": "employee_onboarding_blueprint_flat",
      "domain": "HR",
      "description": "新入社員オンボーディングのブループリント"
    },
    {
      "name": "initiate_background_check",
      "domain": "HR",
      "description": "背景調査を開始"
    },
    ...
  ]
}

# ツール実行
POST /tools/{tool_name}
Request:
{
  "employee_name": "田中太郎",
  "start_date": "2026-02-01",
  "role": "Software Engineer"
}

Response:
{
  "result": {
    "version": "1.0",
    "intent": "employee_onboarding",
    ...
  }
}
```

---

## なぜ2つに分離されているのか

### 🎯 アーキテクチャ設計の理由

#### **1. 関心の分離 (Separation of Concerns)**

```
┌───────────────────────────────────────┐
│  Backend API                          │
│  責務: エージェントのオーケストレーション│
│  - タスク分析                         │
│  - エージェント管理                    │
│  - 実行フロー制御                      │
│  - 結果集約                           │
└───────────────────────────────────────┘

┌───────────────────────────────────────┐
│  MCP Server                           │
│  責務: ドメイン固有のビジネスロジック  │
│  - HR業務処理                         │
│  - IT業務処理                         │
│  - データアクセス                      │
│  - 外部システム統合                    │
└───────────────────────────────────────┘
```

**メリット**:
- ✅ それぞれが独立して開発・テスト可能
- ✅ コードの可読性と保守性の向上
- ✅ チーム分業が容易

#### **2. スケーラビリティ**

```
高負荷時の動作:

Backend API: 3レプリカ
  ├─ レプリカ1 (タスク1を処理中)
  ├─ レプリカ2 (タスク2を処理中)
  └─ レプリカ3 (タスク3を処理中)
       ↓ 全てが同じMCP Serverを利用
MCP Server: 5レプリカ
  ├─ レプリカ1 (HRツール処理)
  ├─ レプリカ2 (ITツール処理)
  ├─ レプリカ3 (HRツール処理)
  ├─ レプリカ4 (マーケティングツール処理)
  └─ レプリカ5 (データアクセス処理)
```

**メリット**:
- ✅ それぞれ独立してスケールアウト可能
- ✅ ボトルネックに応じたリソース配分
- ✅ コスト最適化

#### **3. 再利用性 (Reusability)**

```
MCP Serverは複数のシステムから利用可能:

┌─────────────────┐
│  Backend API 1  │─┐
└─────────────────┘ │
                    │
┌─────────────────┐ │    ┌──────────────┐
│  Backend API 2  │─┼───►│  MCP Server  │
└─────────────────┘ │    └──────────────┘
                    │
┌─────────────────┐ │
│  Other System   │─┘
└─────────────────┘
```

**メリット**:
- ✅ 一度実装したツールを複数のシステムで再利用
- ✅ ビジネスロジックの一元管理
- ✅ 開発効率の向上

#### **4. セキュリティと隔離**

```
┌─────────────────────────────────────┐
│  Backend API                        │
│  - エージェントロジック              │
│  - セッション管理                    │
│  - 認証・認可                        │
└─────────────────────────────────────┘
          ↓ 制御されたAPI呼び出し
┌─────────────────────────────────────┐
│  MCP Server                         │
│  - 機密データアクセス                │
│  - 外部システム統合                  │
│  - ビジネスルール実行                │
└─────────────────────────────────────┘
```

**メリット**:
- ✅ MCP Serverを内部ネットワークに隔離可能
- ✅ アクセス制御の細かい設定
- ✅ セキュリティ境界の明確化

#### **5. 開発と更新の独立性**

```
Backend APIの更新:
- エージェントオーケストレーションロジックの改善
- 新しいAIモデルへの対応
→ MCP Serverに影響なし

MCP Serverの更新:
- 新しいHRツールの追加
- データアクセスロジックの改善
→ Backend APIに影響なし
```

**メリット**:
- ✅ それぞれ独立してデプロイ可能
- ✅ ダウンタイムの最小化
- ✅ 段階的な機能追加

---

## 通信プロトコル

### 📡 Backend API ⇄ MCP Server

#### **HTTPベースのJSON-RPC**

```python
# Backend APIからMCP Serverへのリクエスト
POST http://ca-mcp-macaedevfnmlt.....io/tools/employee_onboarding_blueprint_flat
Headers:
  Content-Type: application/json
  Authorization: Bearer {token} (認証有効時)

Body:
{
  "employee_name": "田中太郎",
  "start_date": "2026-02-01",
  "role": "Software Engineer"
}

# MCP Serverからのレスポンス
Status: 200 OK
Body:
{
  "result": {
    "version": "1.0",
    "intent": "employee_onboarding",
    "employee": {
      "name": "田中太郎",
      "start_date": "2026-02-01",
      "role": "Software Engineer"
    },
    "steps": [...]
  }
}
```

#### **エラーハンドリング**

```python
# エラーレスポンス例
Status: 400 Bad Request
Body:
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "employee_name is required",
    "details": "..."
  }
}

Status: 500 Internal Server Error
Body:
{
  "error": {
    "code": "TOOL_EXECUTION_FAILED",
    "message": "Failed to execute tool",
    "details": "..."
  }
}
```

---

## 具体的な実行例

### 🎬 シナリオ: 新入社員オンボーディング

#### **Step 1: ユーザーがタスクを入力**

```
ユーザー: "田中太郎さんの新入社員オンボーディングプランを作成してください。
          開始日は2026年2月1日、役職はSoftware Engineerです。"
```

#### **Step 2: App Service → Backend API**

```python
# App ServiceがBackend APIを呼び出し
POST https://ca-macaedevfnmlt.....io/api/plans
{
  "user_input": "田中太郎さんの新入社員オンボーディング...",
  "team_id": "00000000-0000-0000-0000-000000000001",  # HR Teamのデプロイメント
  "user_id": "user-principal-id"
}
```

#### **Step 3: Backend APIがタスクを分析**

```python
# OrchestrationManagerが動作
1. タスク分析
   - 意図: 新入社員オンボーディング
   - 必要なエージェント: HR Manager, IT Setup Agent

2. チーム設定を読み込み
   - HR Employee Onboarding Team
   - 5つのエージェント構成

3. 実行計画を作成
```

#### **Step 4: Backend API → MCP Server (ブループリント取得)**

```python
# Backend APIがMCPツールを呼び出し
POST http://ca-mcp-macaedevfnmlt.....io/tools/employee_onboarding_blueprint_flat
{
  "employee_name": "田中太郎",
  "start_date": "2026-02-01",
  "role": "Software Engineer"
}

# MCP Serverからのレスポンス
{
  "result": {
    "version": "1.0",
    "intent": "employee_onboarding",
    "steps": [
      {
        "id": "bg_check",
        "domain": "HR",
        "action": "背景調査を開始",
        "tool": "initiate_background_check"
      },
      {
        "id": "configure_laptop",
        "domain": "TECH_SUPPORT",
        "action": "PCのセットアップ",
        "tool": "configure_laptop"
      },
      {
        "id": "create_accounts",
        "domain": "TECH_SUPPORT",
        "action": "システムアカウントを作成",
        "tool": "create_system_accounts"
      },
      {
        "id": "orientation",
        "domain": "HR",
        "action": "オリエンテーションをスケジュール",
        "tool": "schedule_orientation_session"
      }
    ]
  }
}
```

#### **Step 5: Backend API → AI Foundry (エージェント実行)**

```python
# HR Manager エージェントを実行
FoundryService.run_agent(
  agent_id="hr-manager-agent",
  instructions="""
  以下のブループリントに従って、新入社員オンボーディングプランを作成してください。
  {blueprint}
  """,
  tools=[
    "initiate_background_check",
    "schedule_orientation_session"
  ]
)

# AI Foundry内で実行される処理:
1. GPT-4.1モデルがブループリントを理解
2. MCPツール "initiate_background_check" を呼び出し
   → Backend API経由で MCP Serverにリクエスト
   → MCP Serverが実行してレスポンス
3. MCPツール "schedule_orientation_session" を呼び出し
   → Backend API経由で MCP Serverにリクエスト
   → MCP Serverが実行してレスポンス
4. 結果を統合してHR部分のプランを生成
```

#### **Step 6: Backend API → AI Foundry (次のエージェント実行)**

```python
# IT Setup エージェントを実行
FoundryService.run_agent(
  agent_id="it-setup-agent",
  instructions="""
  以下のブループリントに従って、IT関連のセットアップを実行してください。
  {blueprint}
  """,
  tools=[
    "configure_laptop",
    "create_system_accounts"
  ]
)

# AI Foundry内で実行される処理:
1. GPT-4.1モデルがブループリントを理解
2. MCPツール "configure_laptop" を呼び出し
3. MCPツール "create_system_accounts" を呼び出し
4. IT部分のプランを生成
```

#### **Step 7: Backend APIが結果を集約**

```python
# OrchestrationManagerが結果を集約
final_result = {
  "plan_type": "新入社員オンボーディング",
  "employee": {
    "name": "田中太郎",
    "start_date": "2026-02-01",
    "role": "Software Engineer"
  },
  "pre_boarding": [
    {
      "task": "背景調査",
      "status": "開始済み",
      "check_id": "BGC-2026-001",
      "estimated_completion": "2026-01-20"
    },
    {
      "task": "PCセットアップ",
      "status": "完了",
      "laptop_model": "Dell Latitude 5540",
      "asset_tag": "LAPTOP-2026-042"
    },
    {
      "task": "システムアカウント作成",
      "status": "完了",
      "accounts": [
        {"system": "Active Directory", "username": "t.tanaka"},
        {"system": "Office 365", "email": "t.tanaka@company.com"}
      ]
    }
  ],
  "day_1": [
    {
      "task": "オリエンテーション",
      "status": "スケジュール済み",
      "session_date": "2026-02-01",
      "location": "Conference Room A"
    }
  ]
}

# Cosmos DBに保存
await cosmosdb.save_plan(final_result)
```

#### **Step 8: Backend API → App Service (結果を返す)**

```python
# WebSocketでリアルタイム更新を送信
{
  "type": "plan_completed",
  "plan_id": "uuid",
  "result": final_result,
  "status": "completed"
}
```

#### **Step 9: App Serviceがユーザーに表示**

```markdown
# 新入社員オンボーディングプラン

**従業員**: 田中太郎
**開始日**: 2026年2月1日
**役職**: Software Engineer

## 事前準備 (Pre-boarding)
- ✅ 背景調査開始 (ID: BGC-2026-001、完了予定: 2026年1月20日)
- ✅ PCセットアップ完了 (Dell Latitude 5540、Asset: LAPTOP-2026-042)
- ✅ システムアカウント作成完了
  - Active Directory: t.tanaka
  - Office 365: t.tanaka@company.com

## 初日 (Day 1)
- 📅 オリエンテーション (Conference Room A)
```

---

## まとめ

### ✅ 2つのContainer Appsの役割分担

```
┌─────────────────────────────────────────────────┐
│  App Service (app-macaedevfnmlt)                │
│  役割: フロントエンド (ユーザーインターフェース) │
│  技術: React + TypeScript                       │
└──────────────────┬──────────────────────────────┘
                   ↓ HTTPS API呼び出し
┌─────────────────────────────────────────────────┐
│  Backend API (ca-macaedevfnmlt)                 │
│  役割: エージェントオーケストレーター            │
│  技術: FastAPI + Azure Agent Framework          │
│  責務:                                          │
│   - タスク分析                                  │
│   - エージェント管理                             │
│   - 実行フロー制御                               │
│   - 結果集約                                    │
└──────────────────┬──────────────────────────────┘
                   ↓ HTTP JSON-RPC
┌─────────────────────────────────────────────────┐
│  MCP Server (ca-mcp-macaedevfnmlt)              │
│  役割: ツールとビジネスロジックの提供者          │
│  技術: FastMCP                                  │
│  責務:                                          │
│   - HR業務処理                                  │
│   - IT業務処理                                  │
│   - データアクセス                               │
│   - 外部システム統合                             │
└─────────────────────────────────────────────────┘
```

### 🎯 リクエストフローの要点

1. **App Service** がユーザーからのタスクを受け取る
2. **Backend API** にAPI呼び出し
3. **Backend API** がタスクを分析し、エージェントを選択
4. **AI Foundry** でエージェント実行
5. エージェントが必要に応じて **MCP Server** のツールを呼び出し
6. **MCP Server** がビジネスロジックを実行して結果を返す
7. **Backend API** が結果を集約
8. **App Service** にWebSocketでリアルタイム配信
9. ユーザーに結果を表示

### 🚀 なぜ分離されているか

1. **関心の分離**: オーケストレーションとビジネスロジックを分離
2. **スケーラビリティ**: それぞれ独立してスケールアウト可能
3. **再利用性**: MCPツールを複数のシステムで再利用
4. **セキュリティ**: 明確なセキュリティ境界
5. **独立性**: それぞれ独立して開発・デプロイ可能

---

**作成日**: 2026-01-15
**対象環境**: East US 2 (azure-foundry-multi-agent)
**関連ドキュメント**:
- `AZURE_ARCHITECTURE_DETAILED.md`
- `APP_SERVICE_DETAILED.md`
