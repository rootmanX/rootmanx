You are an expert Python developer specializing in trading bots, AI integration, and GUI applications. Your task is to implement the entire Trading Bot project step-by-step, following the exact specifications and roadmap provided below. Proceed phase by phase, outputting code, explanations, and tests at each step. Do not skip phases or add unrequested features. If something is unclear, ask for clarification before proceeding.

## Project Overview
Develop a Trading Bot that connects to MetaTrader 5 (MT5) terminal, uses over 10 custom indicators, has a modern GUI with PySide6 + PyQtGraph + QML / QDarkStyle, AI decision-making core via OpenRouter API, reads candle data from MT as text for AI analysis. Users can customize strategies by toggling indicators, setting parameters, managing positions (including multi-TP, trailing stop, limit orders), and writing custom prompts. Handle API limits with multiple keys, symbol variations across brokers, backtesting, profile saving, and more.

**New Features Integration**:
- **Trading Hours Management**: User sets trading hours based on their timezone (e.g., UTC offset input). Also, session-based trading (e.g., Asia: 00:00-09:00 UTC, London: 08:00-17:00 UTC, NY: 13:00-22:00 UTC). Toggle on/off in GUI to enable/disable hour filters.
- **News Filter**: Daily fetch of economic news from Investing.com via RSS feed (use feedparser library to parse https://www.investing.com/rss/economic_calendar.rss or similar). Detect high-impact news (filter by importance level in RSS tags). Pause trading 30 minutes before/after high-impact events. Toggle on/off in GUI.
- **Multi-Timeframe Trading**: Support trading/analysis on multiple timeframes (M1, M5, H1, D1, etc.) per symbol. User marks/selects TFs in GUI (checkboxes per symbol). Fetch candle data for selected TFs and include in AI prompt.
- **Live Chart Display**: Real-time charts in GUI via PyQtGraph, with options to toggle news/hour filters visually (e.g., overlay warnings on chart).
- **Modular Structure**: Design with strict modularity – separate modules for MT connection (mt_module.py), indicators (indicators/), AI (ai_core.py), strategy (strategy_engine.py), filters (filters/news_filter.py, filters/hours_filter.py), GUI (ui/), backtest (backtester.py). Use dependency injection (e.g., via dataclasses or simple config passing) for easy extension. Avoid tight coupling so developers/AI can add modules without breaking core.

Key Requirements:
- Language: Python 3.10+.
- Libraries: MetaTrader5, requests (for OpenRouter), PySide6, PyQtGraph, QDarkStyle, pandas, numpy, pytest, PyInstaller, feedparser (for Investing.com RSS), pytz (for timezones).
- Security: Encrypt API keys (use Fernet).
- Output from AI: Always JSON format for actions.
- Modular code: Classes for indicators, strategies, filters, etc. Each feature in isolated modules.
- Multi-threading: QThread for non-blocking operations (e.g., news fetch, chart updates).

## Detailed Features
1. **MT Connection**:
   - User browses MT terminal path via GUI.
   - Fetch account details (balance, equity).
   - List all tradeable symbols with live spreads (mt5.symbol_info_tick).
   - Handle symbol variations (e.g., XAUUSD vs XAUUSDb): Auto-discover via symbols_get(), mapping aliases (JSON dict: {"Gold": ["XAUUSD", "GOLD"]}), user search/select.
   - Multi-TF Support: Fetch candles for selected TFs (e.g., mt5.copy_rates_range(symbol, tf, from_date, to_date)).

2. **Indicators (Over 10 + Custom)**:
   - Built-in: SMA, EMA, RSI (period tunable), MACD, Bollinger Bands, Stochastic, ATR, Parabolic SAR, ADX, Ichimoku, Volume Profile.
   - Each as separate Python class inheriting from base Indicator, with calculate(data) method (data: pandas DataFrame of OHLCV).
   - GUI: Checkbox list to toggle, sliders/inputs for params.
   - **Custom Indicator Addition**: User uploads Python code file (via QFileDialog), validate (syntax check with compile), add to indicators list dynamically (exec in isolated namespace or import as module). Warn user of risks (e.g., no exec malicious code).

3. **AI Core (OpenRouter API)**:
   - Input: Text summary of candles (last N, e.g., 100, chunked if needed; include multi-TF summaries), indicators outputs, spreads, user strategy prompt, timezone/session info, news alerts if active.
   - Prompt Builder: Dynamic, e.g., "Analyze {symbol} on TF {tfs} (spread: {spread}pips). Candles: {summary}. Indicators: RSI={value} (period={period}). Strategy: {user_prompt}. Trading session: {current_session}. High-impact news: {news_list}. Decide: {JSON_schema}".
   - Output: Strict JSON: {"action": "buy/sell/close/hold", "symbol": str, "lot_size": float, "stop_loss": int (pips), "take_profit_levels": [int, int] (for multi-TP), "trailing_stop": bool, "trailing_step": int (pips), "order_type": "market/limit/stop", "confidence": float, "reason": str}.
   - Handle limits: Rotate multiple keys (pools: 10 for real, 20 for backtest), exponential backoff, delays (user-settable).

4. **User Strategy Customization**:
   - GUI: Prompt editor with placeholders ({rsi}, {spread}, {tf_summary}, {news_alert}).
   - Position Management: Inputs for lot size, risk %, multi-TP (e.g., 50pips:50%, 100pips:30%), trailing step, limit price.
   - Rule Engine: Simple if-then for combining indicators (AND/OR).
   - Multi-TF Marking: Per-symbol TF checkboxes (e.g., for EURUSD: H1, D1 checked).

5. **API Management**:
   - GUI Panel: Add/validate keys, set delays, separate pools for real/backtest.
   - Rotation: Round-robin, quarantine on 429 error.

6. **Profiles**:
   - Save/load strategies (indicators, params, prompt, API settings, TFs, filters toggles, timezone/session) to JSON/SQLite.
   - GUI: List with select for trade/backtest.

7. **Backtesting**:
   - GUI: Date range picker, symbol/strategy select, TF selection.
   - Use historical data from MT (copy_rates_range) for multi-TF.
   - Simulate trades, calculate metrics (profit, drawdown) with pandas; include simulated news/hour filters.
   - Separate API pool, mock AI option for speed.

8. **Advanced Orders**:
   - Multi-TP: Partial closes via order_modify.
   - Trailing Stop: Background thread monitoring positions, adjust SL.
   - Limit/Stop Orders: mt5.order_send with type.

9. **Trading Filters (New Module: filters/)**:
   - **Hours Filter**: User inputs timezone (e.g., pytz timezone selector), session toggles (Asia/London/NY/All). Check current time before trades; pause if outside. Toggle on/off in GUI.
   - **News Filter**: Daily cron-like fetch (thread) from Investing.com RSS (feedparser.parse('https://www.investing.com/rss/economic_calendar.rss')). Parse for high-impact (e.g., filter entries with 'importance: high'). Pause 30 min before/after. Include in prompt if active. Toggle on/off in GUI; display alerts on live chart.
   - Modular: Separate classes (HoursFilter, NewsFilter) with check() method returning bool (allow_trade).

10. **GUI Details**:
    - Dark theme (QDarkStyle).
    - Tabs: Dashboard (live charts via PyQtGraph with multi-TF overlays, positions, logs, filter status indicators), Strategies (checkboxes, editor, TF markers), Settings (API, MT, timezone/session, news toggle, hours toggle), Backtest (results plot).
    - Real-time updates: Spreads, candles plot, news alerts overlay, hour/session status.

11. **Other**:
    - Logging: All actions to file/UI, including filter pauses.
    - Error Handling: Reconnects, invalid JSON retry, RSS fetch failures (fallback to no news).
    - Spread Integration: Always include in AI prompt for risk calc.
    - Modularity: All features importable as modules; config-driven for easy extension (e.g., add new filter via subclass).

## JSON Schema for AI Output
{
  "action": "buy|sell|close|hold",
  "symbol": "str",
  "lot_size": "float",
  "stop_loss": "int",  // pips
  "take_profit_levels": ["array of ints"],  // e.g., [50, 100]
  "trailing_stop": "bool",
  "trailing_step": "int",  // pips
  "order_type": "market|limit|stop",
  "confidence": "float (0-1)",
  "reason": "str"
}

## Implementation Roadmap
Follow this exactly, phase by phase. Output: Code snippets, file structure, tests. After each phase, confirm completion. Emphasize modularity in code (e.g., each feature in src/{module}/).

### Phase 1: Research & Setup (1-2 weeks)
- Study docs: MetaTrader5, OpenRouter, PySide6, feedparser (RSS), pytz (timezones).
- Design: Architecture diagram (text desc), class/module outlines (e.g., filters module).
- Setup: Repo with modular folders (src/mt, src/indicators, src/ai, src/strategy, src/filters, src/ui, src/backtest), virtualenv, install libs (add feedparser, pytz), initial config.json (with timezone defaults).
- Test: Simple MT connect script + RSS parse test.

### Phase 2: Backend Core (2-3 weeks)
- MT Connection: Browse func, symbols list with spreads, mapping, multi-TF data fetch.
- Indicators: Base class + 10+ impls, custom add (exec safe).
- AI: Prompt builder (include filters/news/TF), API call with rotation/backoff, JSON parse.
- Strategy: Config parser, advanced order logic.
- Profiles: Save/load (include new filters/TF).
- API Pools: Rotation impl.
- Filters: Implement HoursFilter (pytz integration), NewsFilter (feedparser, high-impact parse).

### Phase 3: GUI Development (1-2 weeks)
- Base Window: Tabs, theme.
- Components: Checkboxes (indicators/TFs), editors, date pickers, file dialog for custom indicators, timezone selector, session checkboxes, news/hours toggles.
- Integration: Signals for updates, threads for AI/MT/news fetch/live charts.

### Phase 4: Testing & Optimization (1-2 weeks)
- Units: pytest for indicators, AI, orders, filters (mock RSS/time).
- Backtest: Full sim, metrics (with filters).
- Real Demo: Run on MT demo, check trailing/multi-TP, news pauses, multi-TF.
- Optimize: Chunking for context, caching (news/RSS).

### Phase 5: Deployment & Docs (1 week)
- Build: PyInstaller exe.
- Docs: README, user guide (strategy setup, custom indicators, filters config, modularity for devs).
- Final: Full project zip/test.

Start with Phase 1. Provide code/files as you go. End with complete, runnable project.
