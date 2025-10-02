# TradingAgents クイックスタートガイド

このガイドでは、TradingAgentsを最速で動かす方法を説明します。

## 前提条件

- Python 3.10以上がインストールされていること
- 基本的なPythonの知識
- ターミナル/コマンドラインの基本操作

---

## ステップ1: リポジトリのクローン

```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
```

---

## ステップ2: 仮想環境の作成

### Condaを使用する場合

```bash
conda create -n tradingagents python=3.13
conda activate tradingagents
```

### venvを使用する場合

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
# または
venv\Scripts\activate  # Windows
```

---

## ステップ3: 依存関係のインストール

```bash
pip install -r requirements.txt
```

インストールには数分かかる場合があります。

---

## ステップ4: APIキーの設定

### 必須: FinnHub API

1. [FinnHub](https://finnhub.io/)にアクセス
2. 無料アカウントを作成
3. APIキーを取得
4. 環境変数に設定:

```bash
export FINNHUB_API_KEY="your_finnhub_api_key_here"
```

### 必須: OpenAI API

1. [OpenAI](https://platform.openai.com/)にアクセス
2. アカウントを作成
3. APIキーを取得
4. 環境変数に設定:

```bash
export OPENAI_API_KEY="your_openai_api_key_here"
```

### オプション: 他のLLMプロバイダー

**Google Gemini:**
```bash
export GOOGLE_API_KEY="your_google_api_key_here"
```

**Anthropic Claude:**
```bash
export ANTHROPIC_API_KEY="your_anthropic_api_key_here"
```

### 環境変数を永続化する（推奨）

**Linux/macOS:**
```bash
echo 'export FINNHUB_API_KEY="your_key"' >> ~/.bashrc
echo 'export OPENAI_API_KEY="your_key"' >> ~/.bashrc
source ~/.bashrc
```

**Windows (PowerShell):**
```powershell
[System.Environment]::SetEnvironmentVariable('FINNHUB_API_KEY', 'your_key', 'User')
[System.Environment]::SetEnvironmentVariable('OPENAI_API_KEY', 'your_key', 'User')
```

---

## ステップ5: 最初の実行

### 方法1: Pythonスクリプトで実行

`main.py`を編集して、コスト削減のために軽量モデルを使用します:

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

# カスタム設定（コスト削減）
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4o-mini"      # 軽量モデル
config["quick_think_llm"] = "gpt-4o-mini"     # 軽量モデル
config["max_debate_rounds"] = 1               # 議論を1ラウンドに制限
config["online_tools"] = True                 # オンラインデータを使用

# グラフの初期化
ta = TradingAgentsGraph(debug=True, config=config)

# NVIDIAの2024年5月10日の取引判断を実行
_, decision = ta.propagate("NVDA", "2024-05-10")
print(f"\n最終判断: {decision}")
```

実行:
```bash
python main.py
```

### 方法2: CLIで実行

インタラクティブなCLIを使用:

```bash
python -m cli.main
```

CLIでは以下を選択できます:
- **ティッカーシンボル**: 分析する株式（例: NVDA, AAPL, TSLA）
- **日付**: 分析日（YYYY-MM-DD形式）
- **LLMプロバイダー**: OpenAI、Google、Anthropic
- **モデル**: 使用するLLMモデル
- **研究深度**: 議論のラウンド数（1-3）

---

## ステップ6: 出力の確認

### デバッグモードの出力

`debug=True`で実行すると、各エージェントの出力がリアルタイムで表示されます:

```
================================ Human Message =================================
NVDA

================================== Ai Message ==================================
[MarketAnalyst]
テクニカル分析レポート:
- MACD: ポジティブなクロスオーバー
- RSI: 65（中立からやや買われ過ぎ）
...

================================== Ai Message ==================================
[SentimentAnalyst]
センチメント分析レポート:
- Reddit: 非常にポジティブ（スコア: 0.85）
- ニュース: 中立的（スコア: 0.55）
...

================================== Ai Message ==================================
[Trader]
FINAL TRANSACTION PROPOSAL: **BUY**
```

### 最終判断

スクリプトの最後に最終判断が表示されます:

```
最終判断: BUY
```

可能な判断:
- **BUY**: 購入推奨
- **HOLD**: 保有推奨
- **SELL**: 売却推奨

---

## ステップ7: カスタマイズ

### 異なる株式を分析

```python
# Appleを分析
_, decision = ta.propagate("AAPL", "2024-05-10")

# Teslaを分析
_, decision = ta.propagate("TSLA", "2024-05-10")

# Microsoftを分析
_, decision = ta.propagate("MSFT", "2024-05-10")
```

### 異なる日付を分析

```python
# 異なる日付で分析（取引日である必要があります）
_, decision = ta.propagate("NVDA", "2024-06-15")
_, decision = ta.propagate("NVDA", "2024-07-20")
```

### 研究深度を変更

```python
config["max_debate_rounds"] = 3  # より深い議論
config["max_risk_discuss_rounds"] = 3
```

### 異なるLLMを使用

**Google Geminiを使用:**
```python
config["llm_provider"] = "google"
config["backend_url"] = "https://generativelanguage.googleapis.com/v1"
config["deep_think_llm"] = "gemini-2.0-flash"
config["quick_think_llm"] = "gemini-2.0-flash"
```

**Anthropic Claudeを使用:**
```python
config["llm_provider"] = "anthropic"
config["deep_think_llm"] = "claude-3-sonnet-20240229"
config["quick_think_llm"] = "claude-3-sonnet-20240229"
```

---

## よくある問題と解決方法

### 問題1: APIキーエラー

```
Error: FINNHUB_API_KEY not found
```

**解決方法:**
環境変数が正しく設定されているか確認:
```bash
echo $FINNHUB_API_KEY  # Linux/macOS
echo %FINNHUB_API_KEY%  # Windows
```

空の場合は、再度設定してください。

### 問題2: モジュールが見つからない

```
ModuleNotFoundError: No module named 'langchain'
```

**解決方法:**
依存関係を再インストール:
```bash
pip install -r requirements.txt
```

### 問題3: レート制限エラー

```
Error: Rate limit exceeded
```

**解決方法:**
- 数分待ってから再実行
- `max_debate_rounds`を1に設定
- 無料プランの場合は有料プランへのアップグレードを検討

### 問題4: データが見つからない

```
Error: No data found for ticker NVDA on 2024-05-10
```

**解決方法:**
- 取引日（平日）を指定
- 最近の日付を使用（データが利用可能な日付）
- `online_tools=True`に設定されているか確認

### 問題5: 実行が遅い

**解決方法:**
- 軽量モデルを使用: `gpt-4o-mini`
- 議論ラウンドを減らす: `max_debate_rounds = 1`
- キャッシュを使用: `online_tools = False`（2回目以降）

---

## 次のステップ

### 1. ドキュメントを読む

- **[オンボーディングガイド](ONBOARDING.ja.md)**: プロジェクト全体の理解
- **[開発者ガイド](DEVELOPER_GUIDE.ja.md)**: 技術的な詳細

### 2. コードを探索

主要なファイルを確認:
- `tradingagents/graph/trading_graph.py`: メインシステム
- `tradingagents/agents/`: 各エージェントの実装
- `tradingagents/dataflows/`: データ取得ロジック

### 3. カスタマイズ

- 新しいエージェントを追加
- カスタムデータソースを統合
- 独自の取引戦略を実装

### 4. コミュニティに参加

- **Discord**: https://discord.com/invite/hk9PGKShPK
- **GitHub**: https://github.com/TauricResearch/TradingAgents
- **X (Twitter)**: https://x.com/TauricResearch

---

## 簡単なサンプルコード集

### サンプル1: 複数の株式を一括分析

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4o-mini"
config["quick_think_llm"] = "gpt-4o-mini"
config["max_debate_rounds"] = 1

ta = TradingAgentsGraph(debug=False, config=config)

tickers = ["NVDA", "AAPL", "TSLA", "MSFT", "GOOGL"]
date = "2024-05-10"

results = {}
for ticker in tickers:
    print(f"\n分析中: {ticker}...")
    _, decision = ta.propagate(ticker, date)
    results[ticker] = decision
    print(f"{ticker}: {decision}")

print("\n=== 結果サマリー ===")
for ticker, decision in results.items():
    print(f"{ticker}: {decision}")
```

### サンプル2: 時系列分析

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG
from datetime import datetime, timedelta

config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4o-mini"
config["quick_think_llm"] = "gpt-4o-mini"
config["max_debate_rounds"] = 1

ta = TradingAgentsGraph(debug=False, config=config)

ticker = "NVDA"
start_date = datetime(2024, 5, 1)
dates = [start_date + timedelta(days=i*7) for i in range(4)]  # 4週間

results = []
for date in dates:
    date_str = date.strftime("%Y-%m-%d")
    print(f"\n分析中: {ticker} on {date_str}...")
    _, decision = ta.propagate(ticker, date_str)
    results.append((date_str, decision))
    print(f"{date_str}: {decision}")

print("\n=== 時系列結果 ===")
for date_str, decision in results:
    print(f"{date_str}: {decision}")
```

### サンプル3: カスタム設定でのバッチ処理

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG
import json

def analyze_portfolio(tickers, date, config_overrides=None):
    """ポートフォリオを分析"""
    config = DEFAULT_CONFIG.copy()
    config["deep_think_llm"] = "gpt-4o-mini"
    config["quick_think_llm"] = "gpt-4o-mini"
    config["max_debate_rounds"] = 1
    
    if config_overrides:
        config.update(config_overrides)
    
    ta = TradingAgentsGraph(debug=False, config=config)
    
    results = {}
    for ticker in tickers:
        print(f"分析中: {ticker}...")
        try:
            _, decision = ta.propagate(ticker, date)
            results[ticker] = {
                "decision": decision,
                "status": "success"
            }
        except Exception as e:
            results[ticker] = {
                "decision": None,
                "status": "error",
                "error": str(e)
            }
    
    return results

# 使用例
portfolio = ["NVDA", "AAPL", "TSLA"]
date = "2024-05-10"

results = analyze_portfolio(portfolio, date)

# 結果をJSONで保存
with open("portfolio_analysis.json", "w") as f:
    json.dump(results, f, indent=2)

print("\n結果をportfolio_analysis.jsonに保存しました")
```

---

## コスト管理のヒント

### 推奨設定（低コスト）

```python
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4o-mini"      # 最も安価
config["quick_think_llm"] = "gpt-4o-mini"     # 最も安価
config["max_debate_rounds"] = 1               # 最小限の議論
config["max_risk_discuss_rounds"] = 1         # 最小限のリスク議論
config["online_tools"] = False                # キャッシュを使用（2回目以降）
```

### API呼び出し数の見積もり

1回の`propagate()`呼び出しで:
- アナリスト: 4回（market, sentiment, news, fundamentals）
- リサーチャー: 2回 × `max_debate_rounds`
- トレーダー: 1回
- リスク管理: 3回 × `max_risk_discuss_rounds`
- マネージャー: 2回

**合計（デフォルト設定）**: 約15-20回のAPI呼び出し

### コスト削減のコツ

1. **テスト時は軽量モデル**: `gpt-4o-mini`を使用
2. **議論を制限**: `max_debate_rounds = 1`
3. **キャッシュを活用**: 同じデータを再利用
4. **バッチ処理**: 複数の分析をまとめて実行

---

## トラブルシューティングチェックリスト

実行前に確認:

- [ ] Python 3.10以上がインストールされている
- [ ] 仮想環境が有効化されている
- [ ] 依存関係がインストールされている
- [ ] FINNHUB_API_KEYが設定されている
- [ ] OPENAI_API_KEYが設定されている
- [ ] インターネット接続が有効（online_tools=Trueの場合）
- [ ] 取引日（平日）を指定している

---

## サポート

問題が解決しない場合:

1. **GitHub Issues**: https://github.com/TauricResearch/TradingAgents/issues
2. **Discord**: https://discord.com/invite/hk9PGKShPK
3. **ドキュメント**: プロジェクト内の他のドキュメントを確認

---

**これでTradingAgentsを使い始める準備が整いました！楽しい分析を！** 🚀

