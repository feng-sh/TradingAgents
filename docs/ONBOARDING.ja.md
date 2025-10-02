# TradingAgents 新人向けオンボーディングガイド

## 目次
1. [プロジェクト概要](#プロジェクト概要)
2. [アーキテクチャ](#アーキテクチャ)
3. [環境構築](#環境構築)
4. [プロジェクト構造](#プロジェクト構造)
5. [主要コンポーネント](#主要コンポーネント)
6. [開発ワークフロー](#開発ワークフロー)
7. [よくある質問](#よくある質問)

---

## プロジェクト概要

### TradingAgentsとは？

TradingAgentsは、LLM（大規模言語モデル）を活用したマルチエージェント金融取引フレームワークです。実際の取引会社の構造を模倣し、複数の専門エージェントが協力して市場分析と取引判断を行います。

### 主な特徴

- **マルチエージェントシステム**: アナリスト、リサーチャー、トレーダー、リスク管理チームなど、専門化されたエージェントが協働
- **LangGraphベース**: 柔軟性とモジュール性を確保した設計
- **リアルタイムデータ統合**: FinnHub、Yahoo Finance、Reddit、Google Newsなどからデータを取得
- **メモリ機能**: 過去の取引から学習し、意思決定を改善
- **研究目的**: 学術研究とバックテスト用に設計（金融アドバイスではありません）

### 研究論文

このプロジェクトは以下の論文に基づいています：
- **タイトル**: TradingAgents: Multi-Agents LLM Financial Trading Framework
- **著者**: Yijia Xiao, Edward Sun, Di Luo, Wei Wang
- **arXiv**: [2412.20138](https://arxiv.org/abs/2412.20138)

---

## アーキテクチャ

### システム全体図

```
┌─────────────────────────────────────────────────────────────┐
│                     TradingAgentsGraph                       │
│                    (メインオーケストレーター)                  │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐      ┌──────────────┐     ┌──────────────┐
│ Analyst Team │      │Research Team │     │ Trader Agent │
│              │      │              │     │              │
│ - Market     │──────▶│ - Bull      │─────▶│ - Decision  │
│ - Sentiment  │      │ - Bear      │     │   Making    │
│ - News       │      │ - Debate    │     │             │
│ - Fundamentals│      │             │     │             │
└──────────────┘      └──────────────┘     └──────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │ Risk Mgmt    │
                                          │              │
                                          │ - Aggressive │
                                          │ - Conservative│
                                          │ - Neutral    │
                                          └──────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │Portfolio Mgr │
                                          │ (最終承認)    │
                                          └──────────────┘
```

### エージェントの役割

#### 1. アナリストチーム (Analyst Team)
- **Market Analyst**: テクニカル指標（MACD、RSI等）を分析
- **Sentiment Analyst**: ソーシャルメディアの感情分析
- **News Analyst**: ニュースとマクロ経済指標の監視
- **Fundamentals Analyst**: 企業の財務諸表と業績評価

#### 2. リサーチャーチーム (Researcher Team)
- **Bull Researcher**: 強気の視点で分析
- **Bear Researcher**: 弱気の視点で分析
- **Research Manager**: 議論を調整し、投資計画を作成

#### 3. トレーダー (Trader Agent)
- アナリストとリサーチャーのレポートを統合
- 取引提案（BUY/HOLD/SELL）を作成

#### 4. リスク管理チーム (Risk Management Team)
- **Aggressive Debator**: リスクを取る視点
- **Conservative Debator**: 慎重な視点
- **Neutral Debator**: 中立的な視点
- **Risk Manager**: リスク評価と最終判断

---

## 環境構築

### 必要な環境

- **Python**: 3.10以上（推奨: 3.13）
- **OS**: Linux、macOS、Windows（WSL推奨）
- **メモリ**: 最低8GB（推奨: 16GB以上）

### インストール手順

#### 1. リポジトリのクローン

```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
```

#### 2. 仮想環境の作成

**Condaを使用する場合:**
```bash
conda create -n tradingagents python=3.13
conda activate tradingagents
```

**venvを使用する場合:**
```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
# または
venv\Scripts\activate  # Windows
```

#### 3. 依存関係のインストール

```bash
pip install -r requirements.txt
```

#### 4. 環境変数の設定

必要なAPIキーを設定します：

```bash
# FinnHub API（無料プランで利用可能）
export FINNHUB_API_KEY="your_finnhub_api_key"

# OpenAI API
export OPENAI_API_KEY="your_openai_api_key"

# オプション: Google Gemini API
export GOOGLE_API_KEY="your_google_api_key"

# オプション: Anthropic Claude API
export ANTHROPIC_API_KEY="your_anthropic_api_key"
```

**永続的に設定する場合（Linux/macOS）:**
```bash
echo 'export FINNHUB_API_KEY="your_key"' >> ~/.bashrc
echo 'export OPENAI_API_KEY="your_key"' >> ~/.bashrc
source ~/.bashrc
```

#### 5. 動作確認

```bash
python main.py
```

---

## プロジェクト構造

```
TradingAgents/
├── tradingagents/              # メインパッケージ
│   ├── agents/                 # エージェント実装
│   │   ├── analysts/           # アナリストエージェント
│   │   │   ├── fundamentals_analyst.py
│   │   │   ├── market_analyst.py
│   │   │   ├── news_analyst.py
│   │   │   └── social_media_analyst.py
│   │   ├── researchers/        # リサーチャーエージェント
│   │   │   ├── bull_researcher.py
│   │   │   └── bear_researcher.py
│   │   ├── trader/             # トレーダーエージェント
│   │   │   └── trader.py
│   │   ├── risk_mgmt/          # リスク管理エージェント
│   │   │   ├── aggresive_debator.py
│   │   │   ├── conservative_debator.py
│   │   │   └── neutral_debator.py
│   │   ├── managers/           # マネージャーエージェント
│   │   │   ├── research_manager.py
│   │   │   └── risk_manager.py
│   │   └── utils/              # ユーティリティ
│   │       ├── agent_states.py
│   │       ├── agent_utils.py
│   │       └── memory.py
│   ├── dataflows/              # データ取得・処理
│   │   ├── finnhub_utils.py    # FinnHubデータ
│   │   ├── yfin_utils.py       # Yahoo Financeデータ
│   │   ├── reddit_utils.py     # Redditデータ
│   │   ├── googlenews_utils.py # Google Newsデータ
│   │   ├── stockstats_utils.py # テクニカル指標
│   │   ├── interface.py        # 統一インターフェース
│   │   └── config.py           # データフロー設定
│   ├── graph/                  # グラフ構造
│   │   ├── trading_graph.py    # メイングラフ
│   │   ├── setup.py            # グラフセットアップ
│   │   ├── propagation.py      # 状態伝播
│   │   ├── reflection.py       # 振り返り機能
│   │   ├── signal_processing.py # シグナル処理
│   │   └── conditional_logic.py # 条件分岐ロジック
│   └── default_config.py       # デフォルト設定
├── cli/                        # コマンドラインインターフェース
│   ├── main.py                 # CLIメイン
│   ├── models.py               # データモデル
│   └── utils.py                # CLIユーティリティ
├── assets/                     # 画像・リソース
├── main.py                     # エントリーポイント
├── requirements.txt            # 依存関係
├── pyproject.toml              # プロジェクト設定
└── README.md                   # プロジェクトREADME
```

---

## 主要コンポーネント

### 1. TradingAgentsGraph (`tradingagents/graph/trading_graph.py`)

システム全体を統括するメインクラスです。

**主要メソッド:**
- `__init__()`: グラフとエージェントの初期化
- `propagate(company_name, trade_date)`: 取引判断の実行
- `reflect_and_remember(returns_losses)`: 過去の判断から学習
- `process_signal(full_signal)`: シグナルの処理

**使用例:**
```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

# グラフの初期化
ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy())

# 取引判断の実行
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

### 2. エージェント (`tradingagents/agents/`)

各エージェントは特定の役割を持ち、LLMを使用して分析を行います。

**エージェントの共通パターン:**
```python
def create_agent(llm, memory=None):
    def agent_node(state, name):
        # 1. 状態から必要な情報を取得
        company_name = state["company_of_interest"]
        
        # 2. メモリから過去の経験を取得（オプション）
        if memory:
            past_memories = memory.get_memories(situation, n_matches=2)
        
        # 3. LLMにプロンプトを送信
        messages = [
            {"role": "system", "content": "システムプロンプト"},
            {"role": "user", "content": "ユーザープロンプト"}
        ]
        result = llm.invoke(messages)
        
        # 4. 結果を状態に追加して返す
        return {
            "messages": [result],
            "report_field": result.content,
            "sender": name
        }
    
    return functools.partial(agent_node, name="AgentName")
```

### 3. データフロー (`tradingagents/dataflows/`)

外部APIからデータを取得し、エージェントに提供します。

**主要な関数:**
- `get_finnhub_news()`: ニュースデータの取得
- `get_YFin_data()`: 株価データの取得
- `get_reddit_company_news()`: Redditの投稿取得
- `get_stockstats_indicator()`: テクニカル指標の計算

**使用例:**
```python
from tradingagents.dataflows.interface import get_YFin_data

# 株価データの取得
data = get_YFin_data("NVDA", "2024-05-10")
```

### 4. メモリシステム (`tradingagents/agents/utils/memory.py`)

ChromaDBを使用して、過去の判断と結果を保存・検索します。

**主要メソッド:**
- `add_situations(situations)`: 新しい経験を追加
- `get_memories(query, n_matches)`: 類似の経験を検索

### 5. 設定管理 (`tradingagents/default_config.py`)

システム全体の設定を管理します。

**主要な設定項目:**
```python
DEFAULT_CONFIG = {
    "llm_provider": "openai",           # LLMプロバイダー
    "deep_think_llm": "o4-mini",        # 深い思考用LLM
    "quick_think_llm": "gpt-4o-mini",   # 高速思考用LLM
    "max_debate_rounds": 1,             # 議論の最大ラウンド数
    "online_tools": True,               # オンラインツールの使用
}
```

---

## 開発ワークフロー

### 基本的な開発フロー

#### 1. 機能ブランチの作成
```bash
git checkout -b feature/your-feature-name
```

#### 2. コードの編集

#### 3. テストの実行
```bash
# 簡単なテスト
python main.py

# CLIでのテスト
python -m cli.main
```

#### 4. コミットとプッシュ
```bash
git add .
git commit -m "Add: 機能の説明"
git push origin feature/your-feature-name
```

### 新しいエージェントの追加方法

#### ステップ1: エージェントファイルの作成

`tradingagents/agents/your_category/your_agent.py`:
```python
import functools

def create_your_agent(llm, memory=None):
    def agent_node(state, name):
        # エージェントのロジックを実装
        company_name = state["company_of_interest"]
        
        messages = [
            {"role": "system", "content": "あなたの役割"},
            {"role": "user", "content": f"{company_name}を分析してください"}
        ]
        
        result = llm.invoke(messages)
        
        return {
            "messages": [result],
            "your_report_field": result.content,
            "sender": name
        }
    
    return functools.partial(agent_node, name="YourAgent")
```

#### ステップ2: `__init__.py`に追加

`tradingagents/agents/__init__.py`:
```python
from .your_category.your_agent import create_your_agent

__all__ = [
    # ... 既存のエージェント
    "create_your_agent",
]
```

#### ステップ3: グラフに統合

`tradingagents/graph/setup.py`でエージェントをグラフに追加します。

### カスタム設定の使用

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

# カスタム設定の作成
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4o"
config["max_debate_rounds"] = 3
config["online_tools"] = False  # キャッシュデータを使用

# カスタム設定でグラフを初期化
ta = TradingAgentsGraph(debug=True, config=config)
```

---

## よくある質問

### Q1: APIコストを抑えるには？

**A:** 以下の方法でコストを削減できます：
- `o4-mini`と`gpt-4o-mini`を使用（テスト用）
- `max_debate_rounds`を1に設定
- `online_tools=False`でキャッシュデータを使用

```python
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "o4-mini"
config["quick_think_llm"] = "gpt-4o-mini"
config["max_debate_rounds"] = 1
config["online_tools"] = False
```

### Q2: デバッグモードとは？

**A:** `debug=True`で初期化すると、各エージェントの出力がリアルタイムで表示されます：

```python
ta = TradingAgentsGraph(debug=True, config=config)
```

### Q3: 異なるLLMプロバイダーを使用するには？

**A:** 設定で`llm_provider`を変更します：

```python
# Google Geminiを使用
config["llm_provider"] = "google"
config["backend_url"] = "https://generativelanguage.googleapis.com/v1"
config["deep_think_llm"] = "gemini-2.0-flash"
config["quick_think_llm"] = "gemini-2.0-flash"

# Anthropic Claudeを使用
config["llm_provider"] = "anthropic"
config["deep_think_llm"] = "claude-3-opus-20240229"
config["quick_think_llm"] = "claude-3-sonnet-20240229"
```

### Q4: オフラインでテストできますか？

**A:** はい、`online_tools=False`に設定すると、キャッシュされたデータを使用します：

```python
config["online_tools"] = False
```

ただし、初回実行時はオンラインでデータを取得する必要があります。

### Q5: エラー「API rate limit exceeded」が出た場合は？

**A:** 以下の対策を試してください：
1. APIキーの使用量を確認
2. `max_debate_rounds`を減らす
3. 実行間隔を空ける
4. 無料プランの場合は有料プランへのアップグレードを検討

### Q6: メモリシステムはどのように機能しますか？

**A:** ChromaDBを使用して、過去の取引判断と結果を保存します。類似の市場状況で過去の経験を参照し、より良い判断を行います。

### Q7: 本番環境で使用できますか？

**A:** このフレームワークは**研究目的**で設計されています。実際の取引には使用しないでください。パフォーマンスは多くの要因に依存し、金融アドバイスではありません。

---

## 次のステップ

1. **チュートリアルを実行**: `main.py`を実行して基本的な動作を確認
2. **CLIを試す**: `python -m cli.main`でインタラクティブなインターフェースを体験
3. **コードを読む**: 各エージェントの実装を確認
4. **カスタマイズ**: 独自のエージェントや設定を追加
5. **コミュニティに参加**: [Discord](https://discord.com/invite/hk9PGKShPK)や[GitHub](https://github.com/TauricResearch/)で質問や議論

---

## 参考リンク

- **GitHub**: https://github.com/TauricResearch/TradingAgents
- **論文**: https://arxiv.org/abs/2412.20138
- **Discord**: https://discord.com/invite/hk9PGKShPK
- **公式サイト**: https://tauric.ai/

---

**ようこそTradingAgentsへ！質問があれば、いつでもチームに聞いてください。**

