現在の実装ベースでは、主な機能は次の5系統です。  
ツール定義は [server.py](C:/Users/stell/source/repos/tradingview-mcp/src/tradingview_mcp/server.py) にあります。

**できること**
- 市場スクリーニング  
  `top_gainers`, `top_losers`, `bollinger_scan`, `rating_filter`, `volume_breakout_scanner`, `smart_volume_scanner`
- 個別銘柄のテクニカル分析  
  `coin_analysis`, `multi_timeframe_analysis`, `volume_confirmation_analysis`, `advanced_candle_pattern`, `consecutive_candles_scan`
- AIっぽい総合判断  
  `multi_agent_analysis`, `combined_analysis`
- ニュースとセンチメント  
  `market_sentiment`, `financial_news`
- 価格とバックテスト  
  `yahoo_price`, `market_snapshot`, `backtest_strategy`, `compare_strategies`, `walk_forward_backtest_strategy`
- EGX 専用分析  
  `egx_market_overview`, `egx_sector_scan`, `egx_sector_scanner`, `egx_index_analysis`, `egx_stock_screener`, `egx_trade_plan`, `egx_fibonacci_retracement`

**どう使うか**
- Codex をこのリポジトリで開き直します。  
  project-scoped の `.codex/config.toml` を読ませるためです。
- ふつうは自然文で頼めば十分です。  
  Codex が必要な MCP ツールを自動で呼びます。
- 精度を上げたいときは、次の4つを明示するとよいです。  
  `symbol`, `exchange`, `timeframe`, `limit`
- Yahoo 系は `AAPL`, `BTC-USD`, `SPY` のような Yahoo 記法を使います。  
  TradingView 系は `KUCOIN`, `BINANCE`, `NASDAQ`, `NYSE`, `EGX` などの `exchange` と組み合わせます。

**すぐ使える例**
- 「いまの市場全体を見せて」  
  `market_snapshot`
- 「AAPL を日足で詳しく分析して」  
  `coin_analysis(symbol="AAPL", exchange="NASDAQ", timeframe="1D")`
- 「KuCoin の15分足で上昇率上位3つを出して」  
  `top_gainers(exchange="KUCOIN", timeframe="15m", limit=3)`
- 「BTC-USD を2年、全戦略で比較して」  
  `compare_strategies(symbol="BTC-USD", period="2y")`
- 「NVDA の Reddit センチメントとニュースを見たい」  
  `market_sentiment(symbol="NVDA")` と `financial_news(symbol="NVDA")`
- 「TSLA をテクニカル、ニュース、センチメント込みで総合評価して」  
  `combined_analysis(symbol="TSLA", exchange="NASDAQ", timeframe="1D")`

**向いている用途**
- 売買候補の絞り込み
- 個別銘柄の短時間チェック
- 相場の地合い確認
- 戦略のざっくり比較
- EGX の専用分析

**向いていない用途**
- 発注や自動売買
- ブローカー接続
- 厳密な研究用バックテスト基盤  
  あくまで軽量な分析ツール寄りです。

**コツ**
- Codex には自然文で頼めば大丈夫です。
- ただし、精度を上げるなら `symbol`、`exchange`、`timeframe`、`limit` を入れるのが一番効きます。
- TradingView 系は `BTCUSDT` のような記法が安全です。
- Yahoo 系は `BTC-USD` や `AAPL` のような記法が安全です。
- 時間足は `5m`, `15m`, `1h`, `4h`, `1D`, `1W` を使うと安定です。

**まず使う定番**
- 「今の市場全体をざっくり見せて」
- 「S&P500、NASDAQ、VIX、BTC、ETH をまとめて見せて」
- 「AAPL の現在価格を教えて」
- 「BTC-USD の現在価格と直近の変化を教えて」
- 「KuCoin の15分足で上昇率上位5つを出して」
- 「KuCoin の15分足で下落率上位5つを出して」

**スクリーニング系**
- 「BINANCE の 1h で top gainers を10件見せて」
- 「KUCOIN の 15m で top losers を10件見せて」
- 「KUCOIN の 4h で Bollinger squeeze を BBW 0.04 以下で10件探して」
- 「KUCOIN の 5m で Bollinger rating が 3 の銘柄を5件出して」
- 「KUCOIN の 15m で volume breakout を5件探して」
- 「KUCOIN の smart volume scanner で volume ratio 2.5 以上、price change 3%以上を10件見せて」
- 「NASDAQ の 1D で上昇率上位10銘柄を見せて」

**個別分析系**
- 「BTCUSDT を KUCOIN の 15m で詳しく分析して」
- 「ETHUSDT を BINANCE の 1h でテクニカル分析して」
- 「AAPL を NASDAQ の 1D で詳しく分析して」
- 「TSLA を 4h で multi timeframe analysis して」
- 「BTCUSDT を KUCOIN の 15m で volume confirmation analysis して」
- 「SOLUSDT に連続陽線やローソク足パターンが出ているか見て」

**総合判断系**
- 「TSLA を NASDAQ の 1D で technical + sentiment + news の combined analysis して」
- 「BTCUSDT を KUCOIN の 1h で combined analysis して」
- 「NVDA を NASDAQ の 1D で multi agent analysis して」
- 「ETHUSDT を BINANCE の 15m で multi agent analysis して、最終判断を教えて」

**ニュース・センチメント系**
- 「NVDA の Reddit sentiment を見せて」
- 「BTC の market sentiment を crypto カテゴリで見せて」
- 「AAPL の関連ニュースを10件見せて」
- 「BTC の金融ニュースを crypto カテゴリで5件見せて」
- 「TSLA の sentiment と news を並べて要約して」

**バックテスト系**
- 「BTC-USD を 2年、rsi 戦略でバックテストして」
- 「AAPL を 1年、supertrend 戦略でバックテストして」
- 「ETH-USD を 2年、1h 足で macd 戦略をバックテストして」
- 「BTC-USD を 2年、全戦略で比較してランキングして」
- 「SPY を 2年、bollinger と supertrend を比較して」
- 「BTC-USD を 2年、walk-forward backtest で overfitting を見て」
- 「AAPL を 2年、ema_cross の walk-forward backtest をして robustness を教えて」

**EGX 向け**
- 「EGX の market overview を見せて」
- 「EGX の 1D で top gainers と most active を見せて」
- 「EGX の banks セクターをスキャンして」
- 「EGX の sector scanner で強いセクター順に見せて」
- 「EGX:COMI をトレードプラン付きで分析して」
- 「EGX の COMI を Fibonacci retracement で見て」

**そのままコピペしやすい形**
- `KUCOIN の 15m で上昇率上位5銘柄を見せて`
- `BTCUSDT を KUCOIN の 1h で詳しく分析して`
- `TSLA を NASDAQ の 1D で combined analysis して`
- `BTC-USD を 2年、全戦略で比較して`
- `AAPL の現在価格と関連ニュースを見せて`

必要なら次に、  
「短期トレード向けプロンプト集」  
「株向けプロンプト集」  
「バックテスト専用プロンプト集」  
のどれかに絞って作れます。