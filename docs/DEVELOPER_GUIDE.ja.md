# TradingAgents 開発者ガイド

## 目次
1. [技術スタック](#技術スタック)
2. [アーキテクチャ詳細](#アーキテクチャ詳細)
3. [コアコンポーネント](#コアコンポーネント)
4. [データフロー](#データフロー)
5. [エージェント開発](#エージェント開発)
6. [テストとデバッグ](#テストとデバッグ)
7. [パフォーマンス最適化](#パフォーマンス最適化)
8. [トラブルシューティング](#トラブルシューティング)

---

## 技術スタック

### コア技術

| 技術 | バージョン | 用途 |
|------|-----------|------|
| Python | 3.10+ | メイン言語 |
| LangChain | latest | LLMオーケストレーション |
| LangGraph | 0.4.8+ | エージェントグラフ構築 |
| ChromaDB | 1.0.12+ | ベクトルデータベース（メモリ） |
| Pandas | 2.3.0+ | データ処理 |

### LLMプロバイダー

- **OpenAI**: GPT-4o, GPT-4o-mini, o1-preview, o4-mini
- **Anthropic**: Claude 3 Opus, Sonnet
- **Google**: Gemini 2.0 Flash

### データソース

- **FinnHub**: ニュース、インサイダー取引、センチメント
- **Yahoo Finance**: 株価、財務諸表
- **Reddit**: ソーシャルメディアセンチメント
- **Google News**: ニュース記事
- **StockStats**: テクニカル指標計算

---

## アーキテクチャ詳細

### LangGraphの構造

TradingAgentsはLangGraphの`StateGraph`を使用して、エージェント間の情報フローを管理します。

```python
from langgraph.graph import StateGraph

# グラフの構築例
workflow = StateGraph(AgentState)

# ノードの追加
workflow.add_node("market_analyst", market_analyst_node)
workflow.add_node("sentiment_analyst", sentiment_analyst_node)
workflow.add_node("trader", trader_node)

# エッジの追加（フロー制御）
workflow.add_edge("market_analyst", "trader")
workflow.add_edge("sentiment_analyst", "trader")

# 条件付きエッジ
workflow.add_conditional_edges(
    "trader",
    should_continue,
    {
        "continue": "risk_manager",
        "end": END
    }
)
```

### 状態管理

#### AgentState

全エージェント間で共有される状態です。

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    company_of_interest: str
    trade_date: str
    sender: str
    
    # アナリストレポート
    market_report: str
    fundamentals_report: str
    sentiment_report: str
    news_report: str
    
    # リサーチャー出力
    investment_plan: str
    
    # トレーダー出力
    trader_investment_plan: str
    
    # リスク管理
    risk_assessment: str
    
    # 議論状態
    investment_debate_state: InvestDebateState
    risk_debate_state: RiskDebateState
```

#### InvestDebateState

リサーチャー間の議論を管理します。

```python
class InvestDebateState(TypedDict):
    history: str              # 議論の履歴
    current_response: str     # 現在の応答
    count: int                # 議論のラウンド数
```

#### RiskDebateState

リスク管理チーム間の議論を管理します。

```python
class RiskDebateState(TypedDict):
    history: str
    current_risky_response: str      # 積極的な意見
    current_safe_response: str       # 保守的な意見
    current_neutral_response: str    # 中立的な意見
    count: int
```

---

## コアコンポーネント

### 1. TradingAgentsGraph

**ファイル**: `tradingagents/graph/trading_graph.py`

システム全体のオーケストレーターです。

#### 初期化プロセス

```python
def __init__(self, selected_analysts, debug, config):
    # 1. 設定の読み込み
    self.config = config or DEFAULT_CONFIG
    
    # 2. LLMの初期化
    self.quick_thinking_llm = self._initialize_llm("quick")
    self.deep_thinking_llm = self._initialize_llm("deep")
    
    # 3. ツールキットの作成
    self.toolkit = Toolkit(self.quick_thinking_llm)
    
    # 4. メモリの初期化
    self.bull_memory = FinancialSituationMemory("bull")
    self.bear_memory = FinancialSituationMemory("bear")
    # ... 他のメモリ
    
    # 5. グラフのセットアップ
    self.graph = self.graph_setup.setup_graph(selected_analysts)
```

#### LLM初期化

```python
def _initialize_llm(self, llm_type):
    provider = self.config["llm_provider"]
    model = self.config[f"{llm_type}_think_llm"]
    
    if provider == "openai":
        return ChatOpenAI(
            model=model,
            temperature=0.7,
            base_url=self.config["backend_url"]
        )
    elif provider == "anthropic":
        return ChatAnthropic(model=model)
    elif provider == "google":
        return ChatGoogleGenerativeAI(model=model)
```

### 2. GraphSetup

**ファイル**: `tradingagents/graph/setup.py`

グラフの構造を定義します。

#### グラフ構築の流れ

```python
def setup_graph(self, selected_analysts):
    workflow = StateGraph(AgentState)
    
    # 1. アナリストノードの追加
    for analyst in selected_analysts:
        workflow.add_node(f"{analyst}_analyst", self._create_analyst(analyst))
    
    # 2. リサーチャーノードの追加
    workflow.add_node("bull_researcher", bull_researcher_node)
    workflow.add_node("bear_researcher", bear_researcher_node)
    workflow.add_node("research_manager", research_manager_node)
    
    # 3. トレーダーノードの追加
    workflow.add_node("trader", trader_node)
    
    # 4. リスク管理ノードの追加
    workflow.add_node("risk_manager", risk_manager_node)
    
    # 5. エッジの定義
    self._add_edges(workflow, selected_analysts)
    
    # 6. グラフのコンパイル
    return workflow.compile()
```

### 3. Propagator

**ファイル**: `tradingagents/graph/propagation.py`

状態の初期化と伝播を管理します。

```python
class Propagator:
    def create_initial_state(self, company_name, trade_date):
        return {
            "messages": [("human", company_name)],
            "company_of_interest": company_name,
            "trade_date": str(trade_date),
            "investment_debate_state": InvestDebateState({
                "history": "",
                "current_response": "",
                "count": 0
            }),
            "risk_debate_state": RiskDebateState({
                "history": "",
                "current_risky_response": "",
                "current_safe_response": "",
                "current_neutral_response": "",
                "count": 0
            }),
            # 各レポートフィールドの初期化
            "market_report": "",
            "fundamentals_report": "",
            "sentiment_report": "",
            "news_report": "",
        }
```

### 4. Reflector

**ファイル**: `tradingagents/graph/reflection.py`

取引結果から学習し、メモリを更新します。

```python
class Reflector:
    def reflect_trader(self, current_state, returns_losses, trader_memory):
        # 1. 現在の状況を抽出
        situation = self._extract_current_situation(current_state)
        
        # 2. トレーダーの判断を取得
        trader_decision = current_state["trader_investment_plan"]
        
        # 3. LLMで振り返りを実行
        reflection_prompt = f"""
        状況: {situation}
        判断: {trader_decision}
        結果: {returns_losses}
        
        この判断を振り返り、何を学んだか説明してください。
        """
        
        result = self.llm.invoke([
            {"role": "system", "content": "過去の判断を分析してください"},
            {"role": "user", "content": reflection_prompt}
        ])
        
        # 4. メモリに保存
        trader_memory.add_situations([(situation, result.content)])
```

### 5. ConditionalLogic

**ファイル**: `tradingagents/graph/conditional_logic.py`

グラフ内の条件分岐を管理します。

```python
class ConditionalLogic:
    def should_continue_debate(self, state):
        """議論を続けるべきか判断"""
        debate_state = state["investment_debate_state"]
        max_rounds = self.config["max_debate_rounds"]
        
        if debate_state["count"] >= max_rounds:
            return "end_debate"
        else:
            return "continue_debate"
    
    def should_approve_trade(self, state):
        """取引を承認すべきか判断"""
        risk_assessment = state["risk_assessment"]
        
        # リスク評価を解析
        if "APPROVE" in risk_assessment:
            return "approve"
        else:
            return "reject"
```

---

## データフロー

### データ取得の統一インターフェース

**ファイル**: `tradingagents/dataflows/interface.py`

すべてのデータソースへのアクセスを統一します。

#### オンライン vs オフライン

```python
def get_YFin_data(ticker, date):
    config = get_config()
    
    if config["online_tools"]:
        # オンラインでデータを取得
        return YFinanceUtils.fetch_online(ticker, date)
    else:
        # キャッシュからデータを取得
        cache_path = os.path.join(
            config["data_cache_dir"],
            f"{ticker}_{date}.json"
        )
        if os.path.exists(cache_path):
            with open(cache_path, 'r') as f:
                return json.load(f)
        else:
            raise FileNotFoundError("キャッシュデータが見つかりません")
```

#### データキャッシング

```python
def cache_data(ticker, date, data):
    config = get_config()
    cache_dir = config["data_cache_dir"]
    os.makedirs(cache_dir, exist_ok=True)
    
    cache_path = os.path.join(cache_dir, f"{ticker}_{date}.json")
    with open(cache_path, 'w') as f:
        json.dump(data, f, indent=2)
```

### 主要なデータ取得関数

#### 1. ニュースデータ

```python
def get_finnhub_news(ticker, start_date, end_date):
    """FinnHubからニュースを取得"""
    import finnhub
    
    finnhub_client = finnhub.Client(api_key=os.getenv("FINNHUB_API_KEY"))
    news = finnhub_client.company_news(ticker, start_date, end_date)
    
    return [
        {
            "headline": item["headline"],
            "summary": item["summary"],
            "source": item["source"],
            "datetime": item["datetime"]
        }
        for item in news
    ]
```

#### 2. 株価データ

```python
def get_YFin_data(ticker, date):
    """Yahoo Financeから株価データを取得"""
    import yfinance as yf
    
    stock = yf.Ticker(ticker)
    hist = stock.history(start=date, end=date)
    
    return {
        "open": hist["Open"].iloc[0],
        "high": hist["High"].iloc[0],
        "low": hist["Low"].iloc[0],
        "close": hist["Close"].iloc[0],
        "volume": hist["Volume"].iloc[0]
    }
```

#### 3. テクニカル指標

```python
def get_stockstats_indicator(ticker, date, indicator):
    """テクニカル指標を計算"""
    from stockstats import StockDataFrame
    
    # 過去のデータを取得
    hist_data = get_YFin_data_window(ticker, date, window=30)
    
    # StockDataFrameに変換
    stock_df = StockDataFrame.retype(hist_data)
    
    # 指標を計算
    if indicator == "macd":
        return stock_df["macd"].iloc[-1]
    elif indicator == "rsi":
        return stock_df["rsi_14"].iloc[-1]
    # ... 他の指標
```

---

## エージェント開発

### エージェントの基本構造

すべてのエージェントは以下のパターンに従います：

```python
import functools

def create_agent_name(llm, memory=None, toolkit=None):
    """
    エージェント作成関数
    
    Args:
        llm: 使用するLLM
        memory: メモリシステム（オプション）
        toolkit: ツールキット（オプション）
    
    Returns:
        エージェントノード関数
    """
    
    def agent_node(state, name):
        """
        エージェントノード関数
        
        Args:
            state: 現在の状態
            name: エージェント名
        
        Returns:
            更新された状態
        """
        # 1. 状態から必要な情報を抽出
        company_name = state["company_of_interest"]
        trade_date = state["trade_date"]
        
        # 2. データの取得（必要に応じて）
        if toolkit:
            data = toolkit.get_data(company_name, trade_date)
        
        # 3. メモリから過去の経験を取得（オプション）
        past_memories = ""
        if memory:
            memories = memory.get_memories(company_name, n_matches=2)
            past_memories = "\n".join([m["recommendation"] for m in memories])
        
        # 4. プロンプトの構築
        system_prompt = f"""
        あなたは{name}です。
        役割: [役割の説明]
        タスク: [タスクの説明]
        """
        
        user_prompt = f"""
        企業: {company_name}
        日付: {trade_date}
        
        [追加のコンテキスト]
        
        過去の経験:
        {past_memories}
        """
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ]
        
        # 5. LLMの呼び出し
        result = llm.invoke(messages)
        
        # 6. 結果を状態に追加
        return {
            "messages": [result],
            "report_field_name": result.content,
            "sender": name
        }
    
    # 7. 部分適用してエージェント名を固定
    return functools.partial(agent_node, name="AgentName")
```

### 実例: Market Analyst

```python
def create_market_analyst(llm, toolkit):
    def market_analyst_node(state, name):
        company_name = state["company_of_interest"]
        trade_date = state["trade_date"]
        
        # テクニカル指標を取得
        macd = get_stockstats_indicator(company_name, trade_date, "macd")
        rsi = get_stockstats_indicator(company_name, trade_date, "rsi")
        
        system_prompt = """
        あなたはテクニカルアナリストです。
        MACD、RSI、その他のテクニカル指標を分析し、
        市場のトレンドと取引シグナルを評価してください。
        """
        
        user_prompt = f"""
        企業: {company_name}
        日付: {trade_date}
        
        テクニカル指標:
        - MACD: {macd}
        - RSI: {rsi}
        
        これらの指標に基づいて、市場分析レポートを作成してください。
        """
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ]
        
        result = llm.invoke(messages)
        
        return {
            "messages": [result],
            "market_report": result.content,
            "sender": name
        }
    
    return functools.partial(market_analyst_node, name="MarketAnalyst")
```

### 議論エージェントの実装

リサーチャーやリスク管理チームは議論を行います。

```python
def create_bull_researcher(llm, memory):
    def bull_researcher_node(state, name):
        # 議論の履歴を取得
        debate_state = state["investment_debate_state"]
        history = debate_state["history"]
        
        # アナリストレポートを取得
        market_report = state["market_report"]
        sentiment_report = state["sentiment_report"]
        
        system_prompt = """
        あなたは強気のリサーチャーです。
        投資機会を見つけ、ポジティブな側面を強調してください。
        """
        
        user_prompt = f"""
        市場レポート: {market_report}
        センチメントレポート: {sentiment_report}
        
        議論の履歴:
        {history}
        
        強気の視点から投資提案を行ってください。
        """
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ]
        
        result = llm.invoke(messages)
        
        # 議論状態を更新
        new_debate_state = debate_state.copy()
        new_debate_state["current_response"] = result.content
        new_debate_state["history"] += f"\n\nBull: {result.content}"
        new_debate_state["count"] += 1
        
        return {
            "messages": [result],
            "investment_debate_state": new_debate_state,
            "sender": name
        }
    
    return functools.partial(bull_researcher_node, name="BullResearcher")
```

---

## テストとデバッグ

### デバッグモードの使用

```python
# デバッグモードを有効化
ta = TradingAgentsGraph(debug=True, config=config)

# 実行すると各エージェントの出力が表示される
_, decision = ta.propagate("NVDA", "2024-05-10")
```

### ログの確認

```python
# 状態の履歴を確認
for date, state in ta.log_states_dict.items():
    print(f"Date: {date}")
    print(f"Market Report: {state['market_report'][:100]}...")
    print(f"Decision: {state['trader_investment_plan']}")
```

### ユニットテストの作成

```python
import unittest
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

class TestTradingAgents(unittest.TestCase):
    def setUp(self):
        config = DEFAULT_CONFIG.copy()
        config["online_tools"] = False  # テスト用にオフライン
        self.ta = TradingAgentsGraph(debug=False, config=config)
    
    def test_propagate(self):
        _, decision = self.ta.propagate("NVDA", "2024-05-10")
        self.assertIn(decision, ["BUY", "HOLD", "SELL"])
    
    def test_memory(self):
        # メモリが正しく機能するかテスト
        self.ta.reflect_and_remember({"return": 0.05})
        memories = self.ta.trader_memory.get_memories("NVDA", n_matches=1)
        self.assertGreater(len(memories), 0)

if __name__ == "__main__":
    unittest.main()
```

---

## パフォーマンス最適化

### 1. キャッシングの活用

```python
# データをキャッシュ
config["online_tools"] = False

# 初回実行でデータを取得してキャッシュ
config["online_tools"] = True
ta.propagate("NVDA", "2024-05-10")

# 2回目以降はキャッシュを使用
config["online_tools"] = False
ta.propagate("NVDA", "2024-05-10")  # 高速
```

### 2. 議論ラウンドの調整

```python
# 議論ラウンドを減らしてAPI呼び出しを削減
config["max_debate_rounds"] = 1
config["max_risk_discuss_rounds"] = 1
```

### 3. 軽量LLMの使用

```python
# テスト時は軽量モデルを使用
config["deep_think_llm"] = "gpt-4o-mini"
config["quick_think_llm"] = "gpt-4o-mini"
```

---

## トラブルシューティング

### よくあるエラーと解決方法

#### 1. API Key Error

```
Error: FINNHUB_API_KEY not found
```

**解決方法:**
```bash
export FINNHUB_API_KEY="your_api_key"
```

#### 2. Rate Limit Error

```
Error: Rate limit exceeded
```

**解決方法:**
- API呼び出し間隔を空ける
- `max_debate_rounds`を減らす
- キャッシュを使用する

#### 3. Memory Error

```
Error: ChromaDB connection failed
```

**解決方法:**
```bash
# ChromaDBを再インストール
pip uninstall chromadb
pip install chromadb
```

#### 4. Data Not Found

```
Error: No data found for ticker NVDA on 2024-05-10
```

**解決方法:**
- 取引日（平日）を指定する
- `online_tools=True`に設定する

---

## 貢献ガイドライン

### コーディング規約

- **PEP 8**に従う
- 関数には型ヒントを追加
- ドキュメント文字列を記述

```python
def create_agent(llm: ChatOpenAI, memory: Optional[FinancialSituationMemory] = None) -> Callable:
    """
    エージェントを作成します。
    
    Args:
        llm: 使用するLLM
        memory: メモリシステム（オプション）
    
    Returns:
        エージェントノード関数
    """
    pass
```

### プルリクエストの作成

1. フォークしてブランチを作成
2. 変更を実装
3. テストを追加
4. ドキュメントを更新
5. PRを作成

---

**開発を楽しんでください！質問があれば、コミュニティで聞いてください。**

