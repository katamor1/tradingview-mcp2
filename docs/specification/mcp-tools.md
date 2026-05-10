# MCP Tool 仕様

この文書は `src/tradingview_mcp/server.py` の `@mcp.tool()` と `@mcp.resource()` から確認できる公開インターフェースを整理したものです。現行実装では MCP tool が 27 件、MCP resource が 1 件あります。

## 公開 Tool 一覧

| 分類 | Tool / Resource | Handler | 主な委譲先 |
| --- | --- | --- | --- |
| Screener | `top_gainers` | `server.py:84` | `screener_service.fetch_trending_analysis()` |
| Screener | `top_losers` | `server.py:100` | `screener_service.fetch_trending_analysis()` |
| Screener | `bollinger_scan` | `server.py:111` | `screener_service.fetch_bollinger_analysis()` |
| Screener | `rating_filter` | `server.py:128` | `screener_service.fetch_trending_analysis()` |
| Asset analysis | `coin_analysis` | `server.py:148` | `screener_service.analyze_coin()` |
| Pattern | `consecutive_candles_scan` | `server.py:167` | `screener_service.scan_consecutive_candles()` |
| Pattern | `advanced_candle_pattern` | `server.py:194` | `screener_service.fetch_multi_timeframe_patterns()` / fallback |
| Volume | `volume_breakout_scanner` | `server.py:242` | `scanner_service.volume_breakout_scan()` |
| Volume | `volume_confirmation_analysis` | `server.py:267` | `scanner_service.volume_confirmation_analyze()` |
| Volume | `smart_volume_scanner` | `server.py:281` | `scanner_service.smart_volume_scan()` |
| Multi-agent | `multi_agent_analysis` | `server.py:307` | `multi_agent_service.run_multi_agent_analysis()` |
| EGX | `egx_market_overview` | `server.py:327` | `egx_service.get_egx_market_overview()` |
| EGX | `egx_sector_scan` | `server.py:340` | `egx_service.scan_egx_sector()` |
| EGX | `egx_sector_scanner` | `server.py:355` | `egx_service.run_egx_sector_scanner()` |
| EGX | `egx_index_analysis` | `server.py:377` | `egx_service.analyze_egx_index()` |
| EGX | `egx_stock_screener` | `server.py:391` | `egx_service.screen_egx_stocks()` |
| EGX | `egx_trade_plan` | `server.py:412` | `egx_service.generate_egx_trade_plan()` |
| EGX | `egx_fibonacci_retracement` | `server.py:424` | `egx_service.analyze_egx_fibonacci()` |
| Multi-timeframe | `multi_timeframe_analysis` | `server.py:440` | `screener_service.run_multi_timeframe_analysis()` |
| Sentiment/news | `market_sentiment` | `server.py:455` | `sentiment_service.analyze_sentiment()` |
| Sentiment/news | `financial_news` | `server.py:467` | `news_service.fetch_news_summary()` |
| Sentiment/news | `combined_analysis` | `server.py:479` | `coin_analysis()` + sentiment/news services |
| Backtest | `backtest_strategy` | `server.py:522` | `backtest_service.run_backtest()` |
| Backtest | `compare_strategies` | `server.py:554` | `backtest_service.compare_strategies()` |
| Backtest | `walk_forward_backtest_strategy` | `server.py:572` | `backtest_service.walk_forward_backtest()` |
| Yahoo Finance | `yahoo_price` | `server.py:605` | `yahoo_finance_service.get_price()` |
| Yahoo Finance | `market_snapshot` | `server.py:615` | `yahoo_finance_service.get_market_snapshot()` |
| Resource | `exchanges://list` (`exchanges_list`) | `server.py:625` | `coinlist/*.txt` |

## 共通仕様

### timeframe

多くの tool は `sanitize_timeframe()` で timeframe を正規化します。

| 入力 | 正規化後 |
| --- | --- |
| `5m` | `5m` |
| `15m` | `15m` |
| `1h` | `1h` |
| `4h` | `4h` |
| `1d` / `1D` | `1D` |
| `1w` / `1W` | `1W` |
| `1m` / `1M` | `1M` |

空文字や未対応値は、tool ごとの既定 timeframe に置き換わります。対応 timeframe は実装上 `5m`, `15m`, `1h`, `4h`, `1D`, `1W`, `1M` です。

### exchange

`sanitize_exchange()` を通る tool では、exchange は小文字へ正規化され、未対応値は tool ごとの既定 exchange に置き換わります。対応 exchange は `core/utils/validators.py` の `EXCHANGE_SCREENER` に定義されています。

現行の主な対応市場は次の通りです。

- Crypto: `all`, `huobi`, `kucoin`, `coinbase`, `gateio`, `binance`, `bitfinex`, `bitget`, `bybit`, `okx`
- Stocks: `bist`, `egx`, `nasdaq`, `nyse`, `bursa`, `myx`, `klse`, `ace`, `leap`, `hkex`, `hk`, `hsi`, `asx`

### 戻り値とエラー

- 成功時は `dict` または `list[dict]` を返します。
- 失敗時は多くのサービスが `{"error": "..."}` を返します。例外を送出するのではなく、MCP tool の戻り値としてエラー辞書を返す設計です。
- 一部のスキャン処理は外部 API 例外を捕捉し、失敗した銘柄やバッチをスキップします。その場合、空配列や件数 0 の正常レスポンスになることがあります。
- `indicators` や詳細分析フィールドは TradingView 側の指標セットに依存し、完全固定ではありません。

### 根拠表記

各 tool の根拠行は `server.py` の handler 開始行です。戻り値の詳細は handler ではなく委譲先 service で構築されることが多いため、仕様の更新時は handler と service の両方を確認してください。

## Screener Tools

### `top_gainers(exchange="KUCOIN", timeframe="15m", limit=25) -> list[dict]`

指定 exchange/timeframe の上昇率上位を返します。内部では `fetch_trending_analysis()` を呼び、`changePercent` の降順結果を使います。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `symbol`, `changePercent`, `indicators`。
- 根拠: `server.py:84`, `screener_service.fetch_trending_analysis()`。

### `top_losers(exchange="KUCOIN", timeframe="15m", limit=25) -> list[dict]`

指定 exchange/timeframe の下落率上位を返します。`fetch_trending_analysis()` の結果を `changePercent` 昇順に並べ替えます。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `symbol`, `changePercent`, `indicators`。
- 根拠: `server.py:100`, `screener_service.fetch_trending_analysis()`。

### `bollinger_scan(exchange="KUCOIN", timeframe="4h", bbw_threshold=0.04, limit=50) -> list[dict]`

Bollinger Band Width が閾値以下の銘柄を検出します。低 BBW は squeeze 候補として扱われます。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "4h")`。
- `bbw_threshold`: サーバー側で丸めなし。サービス層の BBW filter に渡されます。
- `limit`: 1 から 100 に丸め。
- 主要戻り値: `symbol`, `changePercent`, `indicators`。
- 根拠: `server.py:111`, `screener_service.fetch_bollinger_analysis()`。

### `rating_filter(exchange="KUCOIN", timeframe="5m", rating=2, limit=25) -> list[dict]`

Bollinger Band rating で銘柄を絞り込みます。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "5m")`。
- `rating`: -3 から 3 に丸め。`-3=Strong Sell`, `-2=Sell`, `-1=Weak Sell`, `1=Weak Buy`, `2=Buy`, `3=Strong Buy`。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `symbol`, `changePercent`, `indicators`。
- 根拠: `server.py:128`, `screener_service.fetch_trending_analysis(filter_type="rating")`。

## Asset Analysis And Pattern Tools

### `coin_analysis(symbol, exchange="KUCOIN", timeframe="15m") -> dict`

単一銘柄のテクニカル分析を返します。TradingView の指標を取得し、価格、RSI、MACD、移動平均、Bollinger Bands、ATR、出来高、サポート/レジスタンス、market sentiment などをまとめます。

- `symbol`: 必須。`exchange:symbol` 形式でなければサービス側で exchange と組み合わせます。
- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- 主要戻り値: `symbol`, `exchange`, `timeframe`, `timestamp`, `price_data`, `timeframe_context`, `rsi`, `macd`, `sma`, `ema`, `bollinger_bands`, `atr`, `volume_analysis`, `support_resistance`, `market_structure`, `market_sentiment`。
- エラー例: tradingview-ta 未導入、対象銘柄データなし、metrics 計算不能、分析例外。
- 根拠: `server.py:148`, `screener_service.analyze_coin()`。

### `consecutive_candles_scan(exchange="KUCOIN", timeframe="15m", pattern_type="bullish", candle_count=3, min_growth=2.0, limit=20) -> dict`

連続した bullish/bearish candle pattern を検出します。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- `pattern_type`: サーバー層で丸めなし。サービス層は `bullish` / `bearish` 前提で評価します。
- `candle_count`: 2 から 5 に丸め。
- `min_growth`: 0.5 から 20.0 に丸め。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `exchange`, `timeframe`, `pattern_type`, `candle_count`, `min_growth`, `total_found`, `data`。
- 根拠: `server.py:167`, `screener_service.scan_consecutive_candles()`。

### `advanced_candle_pattern(exchange="KUCOIN", base_timeframe="15m", pattern_length=3, min_size_increase=10.0, limit=15) -> dict`

複数 timeframe データが使える場合は tradingview-screener による multi-timeframe candle pattern を返し、利用できない場合は single-timeframe fallback を使います。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `base_timeframe`: `sanitize_timeframe(base_timeframe, "15m")`。
- `pattern_length`: 2 から 4 に丸め。
- `min_size_increase`: 5.0 から 50.0 に丸め。
- `limit`: 1 から 30 に丸め。
- symbol list が空の場合: `{"error": "No symbols found for exchange: ...", "exchange": ...}`。
- multi-timeframe 成功時の主要戻り値: `exchange`, `base_timeframe`, `pattern_length`, `min_size_increase`, `method`, `total_found`, `data`。
- fallback 時の主要戻り値: `exchange`, `base_timeframe`, `pattern_length`, `min_size_increase`, `method`, `total_found`, `data`。
- 根拠: `server.py:194`, `screener_service.fetch_multi_timeframe_patterns()`, `scan_advanced_candle_patterns_single_tf()`。

### `multi_timeframe_analysis(symbol, exchange="KUCOIN") -> dict`

Weekly, Daily, 4H, 1H, 15m の整合性を分析します。

- `symbol`: 必須。`exchange:symbol` 形式でない場合は `EXCHANGE:SYMBOL` に変換。
- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- 主要戻り値: `symbol`, `exchange`, `analysis_type`, `timeframes`, `alignment`, `recommendation`。
- エラー例: tradingview-ta 未導入。
- 根拠: `server.py:440`, `screener_service.run_multi_timeframe_analysis()`。

## Volume Tools

### `volume_breakout_scanner(exchange="KUCOIN", timeframe="15m", volume_multiplier=2.0, price_change_min=3.0, limit=25) -> list[dict]`

通常より大きな出来高と価格変化を同時に満たす銘柄を検出します。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- `volume_multiplier`: 1.5 から 10.0 に丸め。
- `price_change_min`: 1.0 から 20.0 に丸め。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `symbol`, `changePercent`, `volume_ratio`, `volume_strength`, `current_volume`, `breakout_type`, `indicators`。
- 根拠: `server.py:242`, `scanner_service.volume_breakout_scan()`。

### `volume_confirmation_analysis(symbol, exchange="KUCOIN", timeframe="15m") -> dict`

単一銘柄について、出来高が価格変化を確認しているかを分析します。

- `symbol`: 必須。
- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- 主要戻り値: `symbol`, `price_data`, `volume_analysis`, `technical_indicators`, `signals`, `overall_assessment`。
- エラー例: 対象データなし、指標データなし、分析例外。
- 根拠: `server.py:267`, `scanner_service.volume_confirmation_analyze()`。

### `smart_volume_scanner(exchange="KUCOIN", min_volume_ratio=2.0, min_price_change=2.0, rsi_range="any", limit=20) -> list[dict]`

出来高 breakout、価格変化、RSI 条件を組み合わせたスキャナーです。

- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `min_volume_ratio`: 1.2 から 10.0 に丸め。
- `min_price_change`: 0.5 から 20.0 に丸め。
- `rsi_range`: サーバー層で丸めなし。サービス層で `oversold`, `overbought`, `neutral`, `any` の意図で評価。
- `limit`: 1 から 30 に丸め。
- 主要戻り値: `volume_breakout_scanner` と同系の各要素に `trading_recommendation` を追加。
- 根拠: `server.py:281`, `scanner_service.smart_volume_scan()`。

## Sentiment, News, And Combined Analysis

### `multi_agent_analysis(symbol, exchange="KUCOIN", timeframe="15m") -> dict`

Technical, Sentiment, Risk の 3 観点を議論形式でまとめ、consensus を返します。実装上は LLM 呼び出しではなく、TradingView 指標から作るヒューリスティック分析です。

- `symbol`: 必須。`exchange:symbol` 形式でない場合は `EXCHANGE:SYMBOL` に変換。
- `exchange`: `sanitize_exchange(exchange, "KUCOIN")`。
- `timeframe`: `sanitize_timeframe(timeframe, "15m")`。
- 主要戻り値: `framework_name`, `target`, `timeframe`, `agents_debate`, `consensus`。
- エラー例: 対象データなし、metrics 計算不能。
- 根拠: `server.py:307`, `multi_agent_service.run_multi_agent_analysis()`。

### `market_sentiment(symbol, category="all", limit=20) -> dict`

Reddit JSON API から投稿を集め、キーワードベースでセンチメントを集計します。

- `symbol`: 必須。
- `category`: `crypto`, `stocks`, `all` 想定。未対応値はサービス側の subreddit 選択に依存。
- `limit`: サーバー層で丸めなし。
- 主要戻り値: `symbol`, `sentiment_score`, `sentiment_label`, `posts_analyzed`, `bullish_count`, `bearish_count`, `neutral_count`, `top_posts`, `sources`, `timestamp`。
- 外部依存: Reddit JSON API。取得失敗時は投稿 0 件として集計されます。
- 根拠: `server.py:455`, `sentiment_service.analyze_sentiment()`。

### `financial_news(symbol=None, category="stocks", limit=10) -> dict`

RSS フィードから金融ニュースを取得し、必要に応じて symbol で絞り込みます。

- `symbol`: 任意。`None` の場合はカテゴリ内のニュース全体。
- `category`: `crypto`, `stocks`, `all` 想定。
- `limit`: サーバー層で丸めなし。
- 主要戻り値: `symbol`, `category`, `count`, `feedparser_available`, `items`, `timestamp`。
- 外部依存: `feedparser` と RSS フィード。
- 根拠: `server.py:467`, `news_service.fetch_news_summary()`。

### `combined_analysis(symbol, exchange="NASDAQ", timeframe="1D") -> dict`

`coin_analysis`、`market_sentiment` 相当、`financial_news` 相当を統合し、technical と sentiment の一致度を `confluence` として返します。

- `symbol`: 必須。
- `exchange`: サーバー層では丸めず、`coin_analysis()` 呼び出し先で正規化。
- `timeframe`: サーバー層では丸めず、`coin_analysis()` 呼び出し先で正規化。
- `exchange` が `BINANCE`, `KUCOIN`, `BYBIT` の場合は sentiment/news category を `crypto`、それ以外は `stocks` とします。
- 主要戻り値: `symbol`, `exchange`, `timeframe`, `technical`, `sentiment`, `news`, `confluence`。
- `confluence`: `signals_agree`, `confidence`, `recommendation`。
- 根拠: `server.py:479`。

## EGX Tools

### `egx_market_overview(timeframe="1D", limit=10) -> dict`

EGX 全体の overview を返します。上昇上位、下落上位、出来高上位、market stats を含みます。

- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- `limit`: 1 から 20 に丸め。
- 主要戻り値: `exchange`, `timeframe`, `total_analyzed`, `top_gainers`, `top_losers`, `most_active`, `market_stats`。
- エラー例: tradingview-ta 未導入、EGX symbol list 空、データ取得なし。
- 根拠: `server.py:327`, `egx_service.get_egx_market_overview()`。

### `egx_sector_scan(sector="", timeframe="1D", limit=20) -> dict`

EGX の sector 別スキャンです。`sector` が空の場合は利用可能 sector 一覧と usage を返します。

- `sector`: 空なら一覧表示。指定時は `core/data/egx_sectors.py` の sector 定義に基づきます。
- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `available_sectors`, `usage`、または `exchange`, `sector`, `timeframe`, `total_stocks`, `sector_avg_change`, `sector_sentiment`, `data`。
- 根拠: `server.py:340`, `egx_service.scan_egx_sector()`。

### `egx_sector_scanner(timeframe="1D", top_n_sectors=5, top_n_stocks=3, min_stock_score=60) -> dict`

EGX sector rotation scanner です。sector をランキングし、上位 sector の銘柄候補と rotation signals を返します。

- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- `top_n_sectors`: 1 から 18 に丸め。
- `top_n_stocks`: 1 から 10 に丸め。
- `min_stock_score`: 0 から 100 に丸め。
- 主要戻り値: `exchange`, `timeframe`, `total_sectors`, `total_stocks_scanned`, `weighted_market_view`, `sector_heatmap`, `sector_top_picks`, `rotation_signals`, `disclaimer`。
- 根拠: `server.py:355`, `egx_service.run_egx_sector_scanner()`。

### `egx_index_analysis(index="EGX30", timeframe="1D", limit=30) -> dict`

指定 EGX index の構成銘柄を分析します。

- `index`: `EGX30`, `EGX70`, `EGX100`, `SHARIAH33`, `EGX35LV`, `TAMAYUZ`。
- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- `limit`: 1 から 100 に丸め。
- 主要戻り値: `index`, `index_name`, `description`, `timeframe`, `index_stats`, `sector_breakdown`, `top_gainers`, `top_losers`, `all_stocks`。
- index 不正時は `available_indices` と `usage` を含むエラー。
- 根拠: `server.py:377`, `egx_service.analyze_egx_index()`。

### `egx_stock_screener(timeframe="1D", min_score=55, index_filter="", limit=20) -> dict`

EGX 銘柄の ranking engine です。stock score と trade setup を使って候補を抽出します。

- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- `min_score`: 0 から 100 に丸め。
- `index_filter`: 空なら全 EGX。指定時は EGX index 名で絞り込み。
- `limit`: 1 から 50 に丸め。
- 主要戻り値: `source`, `timeframe`, `min_score`, `total_scanned`, `total_passed`, `grade_distribution`, `qualified_trades`, `qualified_count`, `watchlist`, `execution_rules`。
- 根拠: `server.py:391`, `egx_service.screen_egx_stocks()`。

### `egx_trade_plan(symbol, timeframe="1D") -> dict`

EGX 個別銘柄の trade plan を返します。

- `symbol`: 必須。EGX symbol を想定。
- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- 主要戻り値: 実装上は `generate_egx_trade_plan()` の `output` 辞書。価格、sector、score、setup、entry、stop、targets、risk/reward、execution guidance、disclaimer 系の情報を含みます。
- エラー例: tradingview-ta 未導入、データなし、指標計算不能、setup 生成不能。
- 根拠: `server.py:412`, `egx_service.generate_egx_trade_plan()`。

### `egx_fibonacci_retracement(symbol, lookback="52W", timeframe="1D") -> dict`

EGX 個別銘柄の Fibonacci retracement / extension を分析します。

- `symbol`: 必須。
- `lookback`: `strip().upper()` されます。`1M`, `3M`, `6M`, `52W`, `ALL` 想定。
- `timeframe`: `sanitize_timeframe(timeframe, "1D")`。
- 主要戻り値: `symbol`, `sector`, `timeframe`, `lookback_period`, `price`, `change_pct`, `swing_high`, `swing_low`, `swing_range_pct`, `swing_source`, `trend`, `trend_reasoning`, `retracement_levels`, `extension_levels`, `price_position`, `context`, `interpretation`, `disclaimer`。
- 根拠: `server.py:424`, `egx_service.analyze_egx_fibonacci()`。

## Backtest Tools

### `backtest_strategy(symbol, strategy, period="1y", initial_capital=10000.0, commission_pct=0.1, slippage_pct=0.05, interval="1d", include_trade_log=False, include_equity_curve=False) -> dict`

Yahoo Finance の OHLCV を使い、単一 strategy をバックテストします。

- `symbol`: Yahoo Finance symbol。例: `AAPL`, `BTC-USD`, `THYAO.IS`, `^GSPC`。
- `strategy`: `rsi`, `bollinger`, `macd`, `ema_cross`, `supertrend`, `donchian`。
- `period`: `1mo`, `3mo`, `6mo`, `1y`, `2y`。
- `interval`: `1d` または `1h`。
- `initial_capital`, `commission_pct`, `slippage_pct`: サーバー層で丸めなし。
- `include_trade_log`: true なら `trade_log` を含めます。
- `include_equity_curve`: true なら `equity_curve` を含めます。
- 主要戻り値: `symbol`, `strategy`, `period`, `interval`, `timeframe`, `candles_analyzed`, `date_from`, `date_to`, `initial_capital`, `final_capital`, `total_return_pct`, `buy_and_hold_return_pct`, `metrics`, `trades`, `commission_pct`, `slippage_pct`, `data_source`, `disclaimer`, `timestamp`。
- エラー例: strategy 不正、period 不正、interval 不正、データ取得失敗、bar 数不足。
- 根拠: `server.py:522`, `backtest_service.run_backtest()`。

### `compare_strategies(symbol, period="1y", initial_capital=10000.0, interval="1d") -> dict`

6 strategy を同一 symbol/period/interval で比較し、leaderboard を返します。

- `symbol`: Yahoo Finance symbol。
- `period`: `1mo`, `3mo`, `6mo`, `1y`, `2y`。
- `interval`: `1d` または `1h`。
- 内部手数料と slippage はサービス層既定値の 0.1 と 0.05。
- 主要戻り値: `symbol`, `period`, `interval`, `timeframe`, `candles_analyzed`, `date_from`, `date_to`, `initial_capital`, `commission_pct`, `slippage_pct`, `buy_and_hold_return_pct`, `winner`, `ranking`, `disclaimer`, `timestamp`。
- 根拠: `server.py:554`, `backtest_service.compare_strategies()`。

### `walk_forward_backtest_strategy(symbol, strategy, period="2y", initial_capital=10000.0, commission_pct=0.1, slippage_pct=0.05, n_splits=3, train_ratio=0.7, interval="1d") -> dict`

train/test split を複数 fold で作り、未使用データ上の strategy 挙動を評価します。

- `symbol`: Yahoo Finance symbol。
- `strategy`: `rsi`, `bollinger`, `macd`, `ema_cross`, `supertrend`, `donchian`。
- `period`: `1mo`, `3mo`, `6mo`, `1y`, `2y`。
- `n_splits`: サービス層で 2 から 10 の範囲を要求。範囲外はエラー。
- `train_ratio`: サービス層で 0.5 から 0.9 の範囲を要求。範囲外はエラー。
- `interval`: `1d` または `1h`。
- 主要戻り値: `symbol`, `strategy`, `strategy_label`, `period`, `interval`, `timeframe`, `total_candles`, `n_splits`, `train_ratio`, `date_from`, `date_to`, `avg_train_return_pct`, `avg_test_return_pct`, `robustness_score`, `verdict`, `oos_total_trades`, `oos_win_rate_pct`, `oos_sharpe_ratio`, `oos_max_drawdown_pct`, `oos_total_return_pct`, `buy_and_hold_return_pct`, `folds`, `initial_capital`, `commission_pct`, `slippage_pct`, `data_source`, `disclaimer`, `timestamp`。
- 根拠: `server.py:572`, `backtest_service.walk_forward_backtest()`。

## Yahoo Finance Tools

### `yahoo_price(symbol) -> dict`

Yahoo Finance Chart API から単一 symbol の quote を取得します。

- `symbol`: 必須。Yahoo Finance が対応する symbol。
- 主要戻り値: `symbol`, `price`, `previous_close`, `change`, `change_pct`, `currency`, `exchange`, `market_state`, `52w_high`, `52w_low`, `source`, `timestamp`。
- エラー時: `symbol`, `error`, `source`。
- 外部依存: Yahoo Finance Chart API。直接接続失敗時は proxy opener を使う経路があります。
- 根拠: `server.py:605`, `yahoo_finance_service.get_price()`。

### `market_snapshot() -> dict`

主要 index、crypto、FX、ETF の snapshot を Yahoo Finance から取得します。

- 入力なし。
- 主要戻り値: `indices`, `crypto`, `fx`, `etfs`, `timestamp`。
- 個別 symbol の失敗は、各 quote の `error` として表現されます。
- 根拠: `server.py:615`, `yahoo_finance_service.get_market_snapshot()`。

## MCP Resource

### `exchanges://list -> str` (`exchanges_list`)

`src/tradingview_mcp/coinlist/*.txt` から利用可能 exchange 名を一覧化します。

- 成功時: `Available exchanges: ...` 形式の文字列。
- coinlist directory を読めない場合: common exchanges の fallback 文字列。
- 現行 coinlist: `ACE`, `ALL`, `ASX`, `BINANCE`, `BIST`, `BITFINEX`, `BITGET`, `BURSA`, `BYBIT`, `COINBASE`, `EGX`, `GATEIO`, `HK`, `HKEX`, `HSI`, `HUOBI`, `KLSE`, `KUCOIN`, `LEAP`, `MYX`, `NASDAQ`, `NYSE`, `OKX`。
- 根拠: `server.py:625`, `coinlist/*.txt`。
