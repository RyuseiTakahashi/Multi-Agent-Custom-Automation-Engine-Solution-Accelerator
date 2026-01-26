# エージェントフレームワーク アーキテクチャ解説

## 📋 目次
1. [重要な理解](#重要な理解)
2. [Azure AI Foundryの実際の使い方](#azure-ai-foundryの実際の使い方)
3. [Microsoft Agent Frameworkとは](#microsoft-agent-frameworkとは)
4. [実装詳細](#実装詳細)
5. [なぜAgent Frameworkを選んだのか](#なぜagent-frameworkを選んだのか)
6. [Cosmos DBの役割](#cosmos-dbの役割)

---

## 重要な理解

### ✅ このソリューションの実態

```
❌ 使っていないもの:
  - Azure AI Foundry Agent Service
  - Azure AI Foundry Workflow
  - Magentic

✅ 実際に使っているもの:
  - Azure AI Foundry → モデルのみデプロイ (GPT-4.1, o4-mini等)
  - Microsoft Agent Framework → エージェントとワークフローの実装
  - MCP (Model Context Protocol) → ツール統合
```

---

## Azure AI Foundryの実際の使い方

### 🎯 デプロイされているもの

```
┌─────────────────────────────────────────────┐
│  Azure AI Foundry                           │
│  (aif-macaedevfnmlt)                        │
│                                             │
│  デプロイされているリソース:                  │
│  ✅ GPT-4.1 モデル (150K TPM)               │
│  ✅ gpt-4.1-mini モデル (50K TPM)           │
│  ✅ o4-mini モデル (50K TPM)                │
│                                             │
│  使っていないリソース:                       │
│  ❌ Agent Service (エージェント機能)         │
│  ❌ Workflow (ワークフロー機能)             │
│  ❌ Prompt Flow                             │
└─────────────────────────────────────────────┘
```

**つまり**:
- Azure AI Foundryは「**AIモデルのホスティング先**」として使用
- エージェント機能は別途実装（Microsoft Agent Framework）

---

## Microsoft Agent Frameworkとは

### 📚 公式ドキュメント

[Microsoft Agent Framework Documentation](https://learn.microsoft.com/en-us/agent-framework/)

### 🎯 Agent Frameworkの概要

**Microsoft Agent Framework** は、AIエージェントを構築するためのPythonライブラリです。

```python
# requirements.txt から
azure-ai-projects==1.0.0b11       # Azure AI Foundry Projects SDK
semantic-kernel[azure]==1.32.2    # Semantic Kernel (別のフレームワーク)

# Agent Framework は azure-ai-projects に含まれる
from agent_framework import ChatAgent, HostedMCPTool
from agent_framework_azure_ai import AzureAIAgentClient
```

### 🔧 主な機能

```python
1. ChatAgent
   - エージェントの基本クラス
   - 会話履歴の管理
   - ツールの統合

2. AzureAIAgentClient
   - Azure AI Foundryとの接続
   - モデルへのアクセス
   - 認証管理

3. HostedMCPTool
   - MCPサーバーとの統合
   - 外部ツールの呼び出し

4. HostedCodeInterpreterTool
   - Pythonコード実行環境
   - データ分析機能
```

---

## 実装詳細

### 📂 コード構造

```
src/backend/v4/
├── magentic_agents/           # ⚠️ 名前は紛らわしいが Agent Framework を使用
│   ├── foundry_agent.py       # メインのエージェント実装
│   ├── proxy_agent.py         # プロキシエージェント
│   ├── common/
│   │   └── lifecycle.py       # エージェントのライフサイクル管理
│   └── models/
│       └── agent_models.py    # データモデル
│
├── orchestration/
│   └── orchestration_manager.py  # マルチエージェントオーケストレーション
│
└── common/services/
    ├── foundry_service.py     # AI Foundryとの通信
    └── mcp_service.py         # MCP Serverとの通信
```

### 🔨 FoundryAgentTemplate の実装

**ファイル**: `src/backend/v4/magentic_agents/foundry_agent.py`

```python
from agent_framework import (
    ChatAgent,
    ChatMessage,
    HostedCodeInterpreterTool,
    Role
)
from agent_framework_azure_ai import AzureAIAgentClient

class FoundryAgentTemplate(AzureAgentBase):
    """
    Microsoft Agent Framework を使用したエージェント実装

    機能:
    - Azure AI Foundryのモデルを使用
    - Azure AI Search との統合
    - MCPツールとの統合
    - Code Interpreter (オプション)
    """

    def __init__(
        self,
        agent_name: str,
        agent_description: str,
        agent_instructions: str,
        use_reasoning: bool,
        model_deployment_name: str,  # "gpt-4.1" など
        project_endpoint: str,        # AI Foundry エンドポイント
        enable_code_interpreter: bool = False,
        mcp_config: MCPConfig | None = None,
        search_config: SearchConfig | None = None,
    ):
        # AI Foundry プロジェクトクライアントを取得
        project_client = config.get_ai_project_client()

        # 親クラスの初期化
        super().__init__(
            mcp=mcp_config,
            model_deployment_name=model_deployment_name,
            project_endpoint=project_endpoint,
            agent_name=agent_name,
            agent_description=agent_description,
            agent_instructions=agent_instructions,
            project_client=project_client,
        )

        # Azure AI Search を使用するか判定
        self._use_azure_search = self._is_azure_search_requested()
```

### 🔄 エージェント実行フロー

#### **1. エージェントの作成**

```python
# src/backend/v4/magentic_agents/common/lifecycle.py

class MCPEnabledBase:
    """
    Agent Framework のベースクラス
    """

    async def open(self) -> "MCPEnabledBase":
        """エージェントの初期化"""
        # 1. Azure認証情報を取得
        self.creds = DefaultAzureCredential()

        # 2. AgentsClient を作成
        self.client = AgentsClient(
            endpoint=self.project_endpoint,
            credential=self.creds,
        )

        # 3. MCPツールを準備
        await self._prepare_mcp_tool()

        # 4. エージェントをレジストリに登録
        agent_registry.register_agent(self)

        return self
```

#### **2. MCPツールの準備**

```python
async def _prepare_mcp_tool(self) -> None:
    """
    MCP Server への接続を確立し、ツールを作成
    """
    if not self.mcp_cfg:
        return

    # MCP Server のエンドポイント
    # 例: http://ca-mcp-macaedevfnmlt.....io/mcp/
    endpoint = self.mcp_cfg.endpoint

    # HostedMCPTool を作成
    self.mcp_tool = HostedMCPTool(
        name="mcp_server",
        description="Access to HR, IT, Marketing tools",
        server_endpoint=endpoint
    )
```

#### **3. エージェントの実行**

```python
# src/backend/v4/magentic_agents/foundry_agent.py

async def run(
    self,
    user_message: str,
    thread_id: str | None = None
) -> str:
    """
    エージェントを実行
    """
    # 1. ChatAgentを作成（まだ作成されていない場合）
    if not self._agent:
        await self._create_agent()

    # 2. メッセージを送信
    response = await self._agent.chat(
        user_message=user_message,
        thread_id=thread_id
    )

    # 3. 応答を取得
    return response.content
```

#### **4. Azure AI Search との統合**

```python
async def _create_azure_search_enabled_client(self):
    """
    Azure AI Search を有効にしたエージェントを作成
    """
    # 1. AI Foundry の Connection を取得
    connections = await self.project_client.connections.list()

    search_connection = None
    for conn in connections:
        if conn.type == ConnectionType.AZURE_AI_SEARCH:
            search_connection = conn
            break

    # 2. Azure AI Search ツールを作成
    search_tool = {
        "type": "azure_ai_search",
        "connection_id": search_connection.id,
        "index_name": self.search.index_name,
        "query_type": self.search.search_query_type  # "simple" or "semantic"
    }

    # 3. エージェントを作成
    agent = await self.client.agents.create(
        model=self.model_deployment_name,
        name=self.agent_name,
        instructions=self.agent_instructions,
        tools=[search_tool]
    )

    return agent
```

---

## なぜAgent Frameworkを選んだのか

### 🎯 理由

#### **1. 柔軟性とカスタマイズ性**

```
Azure AI Foundry Agent Service (マネージド):
❌ カスタマイズの制限
❌ オーケストレーションロジックの自由度が低い
❌ ベンダーロックイン

Microsoft Agent Framework (ライブラリ):
✅ 完全な制御
✅ カスタムオーケストレーション可能
✅ 任意のPythonコードと統合可能
```

#### **2. マルチエージェントオーケストレーション**

```python
# orchestration_manager.py で実現している複雑なフロー

class OrchestrationManager:
    async def run_multi_agent_plan(self, plan):
        # 1. 依存関係の解析
        dependency_graph = self.analyze_dependencies(plan)

        # 2. 並列実行可能なエージェントを特定
        parallel_groups = self.identify_parallel_groups(dependency_graph)

        # 3. 各グループを並列実行
        for group in parallel_groups:
            tasks = [agent.run() for agent in group]
            results = await asyncio.gather(*tasks)

            # 4. 結果を検証
            validated_results = await self.validate_results(results)

            # 5. 次のグループに引き渡し
            await self.pass_to_next_group(validated_results)

        # 6. 最終結果を生成
        return await self.generate_final_result()
```

このような**複雑なオーケストレーション**は、Agent Serviceでは実現困難です。

#### **3. MCPサーバーとの深い統合**

```python
# Agent Framework は MCP を第一級市民として扱う

from agent_framework import HostedMCPTool

# MCPツールを簡単に統合
mcp_tool = HostedMCPTool(
    name="business_tools",
    server_endpoint="http://ca-mcp-macaedevfnmlt.....io/mcp/"
)

# エージェントに追加
agent = ChatAgent(
    model="gpt-4.1",
    tools=[mcp_tool]
)
```

#### **4. データベースとの直接統合**

```python
# Cosmos DBに直接アクセスして状態管理

class FoundryAgentTemplate:
    def __init__(self, memory_store: DatabaseBase):
        self.memory_store = memory_store

    async def run(self, user_message: str):
        # 1. Cosmos DB から会話履歴を取得
        history = await self.memory_store.get_conversation_history(
            session_id=self.session_id
        )

        # 2. エージェント実行
        response = await self._agent.chat(
            user_message=user_message,
            history=history
        )

        # 3. Cosmos DB に保存
        await self.memory_store.save_message(
            session_id=self.session_id,
            role="assistant",
            content=response.content
        )

        return response
```

#### **5. コスト効率**

```
Azure AI Foundry Agent Service:
- エージェント実行ごとに課金
- オーケストレーション層のコスト

Microsoft Agent Framework:
- モデル使用料のみ
- オーケストレーションは自前のコンテナで実行（Container Appsのコストのみ）
```

---

## Cosmos DBの役割

### 🎯 Cosmos DB の使用目的

```
┌─────────────────────────────────────────┐
│  Cosmos DB (cosmos-macaedevfnmlt)       │
│                                         │
│  データベース: macae                     │
│  コンテナ: memory                        │
│  パーティションキー: /session_id         │
└─────────────────────────────────────────┘
```

### 📊 保存されるデータ

#### **1. セッション管理**

```json
{
  "id": "session-uuid",
  "session_id": "session-uuid",
  "user_id": "user-principal-id",
  "team_id": "00000000-0000-0000-0000-000000000001",
  "created_at": "2026-01-15T10:00:00Z",
  "last_updated": "2026-01-15T10:05:30Z",
  "status": "active",
  "metadata": {
    "team_name": "HR Employee Onboarding Team",
    "user_email": "user@company.com"
  }
}
```

#### **2. 会話履歴**

```json
{
  "id": "message-uuid",
  "session_id": "session-uuid",
  "role": "user",
  "content": "新入社員のオンボーディングプランを作成",
  "timestamp": "2026-01-15T10:00:00Z",
  "metadata": {}
}

{
  "id": "message-uuid-2",
  "session_id": "session-uuid",
  "role": "assistant",
  "content": "オンボーディングプランを作成しています...",
  "timestamp": "2026-01-15T10:00:05Z",
  "agent_name": "HR Manager",
  "metadata": {
    "tool_calls": [
      {
        "tool": "employee_onboarding_blueprint_flat",
        "status": "completed"
      }
    ]
  }
}
```

#### **3. プラン実行履歴**

```json
{
  "id": "plan-uuid",
  "session_id": "session-uuid",
  "plan_type": "employee_onboarding",
  "status": "completed",
  "user_input": "田中太郎さんの新入社員オンボーディングプランを作成",
  "created_at": "2026-01-15T10:00:00Z",
  "completed_at": "2026-01-15T10:05:30Z",
  "agents_executed": [
    {
      "agent_id": "hr-manager-agent-uuid",
      "agent_name": "HR Manager",
      "status": "completed",
      "started_at": "2026-01-15T10:00:10Z",
      "completed_at": "2026-01-15T10:02:30Z",
      "tools_used": [
        "employee_onboarding_blueprint_flat",
        "initiate_background_check",
        "schedule_orientation_session"
      ],
      "result": "HR部分のプラン完成"
    },
    {
      "agent_id": "it-setup-agent-uuid",
      "agent_name": "IT Setup Agent",
      "status": "completed",
      "started_at": "2026-01-15T10:02:35Z",
      "completed_at": "2026-01-15T10:05:20Z",
      "tools_used": [
        "configure_laptop",
        "create_system_accounts"
      ],
      "result": "IT部分のプラン完成"
    }
  ],
  "final_result": {
    "plan_type": "新入社員オンボーディング",
    "employee": {
      "name": "田中太郎",
      "start_date": "2026-02-01",
      "role": "Software Engineer"
    },
    "pre_boarding": [...],
    "day_1": [...]
  }
}
```

#### **4. エージェントメモリ**

```json
{
  "id": "memory-uuid",
  "session_id": "session-uuid",
  "agent_id": "hr-manager-agent-uuid",
  "memory_type": "short_term",
  "content": {
    "context": "現在、田中太郎さんのオンボーディングプランを作成中",
    "current_step": "背景調査を開始",
    "dependencies": [
      {
        "step": "PC設定",
        "status": "待機中",
        "depends_on": "背景調査"
      }
    ]
  },
  "timestamp": "2026-01-15T10:01:00Z"
}

{
  "id": "memory-uuid-2",
  "session_id": "session-uuid",
  "agent_id": "hr-manager-agent-uuid",
  "memory_type": "long_term",
  "content": {
    "employee_history": [
      {
        "name": "田中太郎",
        "onboarding_date": "2026-02-01",
        "status": "プラン作成完了"
      }
    ],
    "learned_patterns": [
      "Software Engineer の標準オンボーディング期間は2週間"
    ]
  },
  "timestamp": "2026-01-15T10:05:30Z"
}
```

### 🔄 Cosmos DBの使用フロー

```python
# src/backend/common/database/cosmosdb.py

class CosmosDBService(DatabaseBase):
    """
    Cosmos DB との統合
    """

    async def save_session(self, session_data: dict):
        """セッションデータを保存"""
        container = self.client.get_database_client("macae")\
                               .get_container_client("memory")

        await container.upsert_item(session_data)

    async def get_conversation_history(
        self,
        session_id: str
    ) -> List[dict]:
        """会話履歴を取得"""
        query = """
        SELECT * FROM c
        WHERE c.session_id = @session_id
          AND c.role IN ('user', 'assistant')
        ORDER BY c.timestamp ASC
        """

        items = container.query_items(
            query=query,
            parameters=[
                {"name": "@session_id", "value": session_id}
            ],
            enable_cross_partition_query=True
        )

        return [item async for item in items]

    async def save_plan(self, plan_data: dict):
        """プラン実行結果を保存"""
        container = self.client.get_database_client("macae")\
                               .get_container_client("memory")

        await container.create_item(plan_data)
```

### 💡 なぜCosmos DBを使うのか

#### **1. グローバルスケール**
```
- 世界中のリージョンにレプリケート可能
- 低レイテンシアクセス
- 高可用性 (99.999% SLA)
```

#### **2. 柔軟なデータモデル**
```json
// スキーマレス - JSON ドキュメントを直接保存
{
  "id": "...",
  "session_id": "...",
  // 任意のフィールドを追加可能
  "custom_field": "..."
}
```

#### **3. パーティショニングによる高パフォーマンス**
```
パーティションキー: /session_id

- 各セッションのデータは同じパーティションに格納
- セッション単位の高速クエリ
- 自動スケールアウト
```

#### **4. リアルタイムアクセス**
```python
# WebSocketでリアルタイム更新を配信
async def stream_agent_progress(session_id: str):
    while True:
        # Cosmos DBから最新状態を取得
        latest_state = await cosmosdb.get_latest_state(session_id)

        # WebSocketで配信
        await websocket.send_json(latest_state)

        await asyncio.sleep(0.5)
```

---

## まとめ

### ✅ アーキテクチャの全体像

```
┌─────────────────────────────────────────────────┐
│  App Service (React フロントエンド)              │
└──────────────────┬──────────────────────────────┘
                   ↓ HTTPS
┌─────────────────────────────────────────────────┐
│  Backend API (Container Apps)                   │
│  - FastAPI                                      │
│  - Microsoft Agent Framework                    │
│    ├─ OrchestrationManager                     │
│    ├─ FoundryAgentTemplate                     │
│    └─ Proxy Agent                              │
└──────┬────────────────┬─────────────────────────┘
       ↓                ↓
┌──────────────┐  ┌──────────────────────────┐
│ Azure AI     │  │ MCP Server               │
│ Foundry      │  │ (Container Apps)         │
│              │  │ - HRService              │
│ - GPT-4.1    │  │ - TechSupportService     │
│ - o4-mini    │  │ - MarketingService       │
│ - gpt-4.1-   │  └──────────────────────────┘
│   mini       │
└──────────────┘
       ↓
┌──────────────────────────────────────────────────┐
│  Cosmos DB                                       │
│  - セッション管理                                 │
│  - 会話履歴                                       │
│  - プラン実行履歴                                 │
│  - エージェントメモリ                             │
└──────────────────────────────────────────────────┘
```

### 🎯 重要ポイント

1. **Azure AI Foundry** = モデルホスティングのみ
2. **Microsoft Agent Framework** = エージェント実装
3. **Magentic** = 使っていない（ディレクトリ名は紛らわしい）
4. **Cosmos DB** = 状態管理とメモリストア
5. **MCP Server** = ビジネスロジックとツール

### 🚀 この設計の利点

1. ✅ **完全な制御**: オーケストレーションロジックを自由にカスタマイズ
2. ✅ **コスト効率**: モデル使用料のみ、マネージドサービスの追加コストなし
3. ✅ **柔軟性**: 任意のPythonコードと統合可能
4. ✅ **スケーラビリティ**: Container Appsで自動スケール
5. ✅ **データ主権**: Cosmos DBで完全なデータ制御

---

**作成日**: 2026-01-15
**対象環境**: East US 2 (azure-foundry-multi-agent)
**関連ドキュメント**:
- `AZURE_ARCHITECTURE_DETAILED.md`
- `CONTAINER_APPS_ARCHITECTURE.md`
