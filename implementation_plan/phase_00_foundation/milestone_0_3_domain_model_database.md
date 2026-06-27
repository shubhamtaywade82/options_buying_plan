# Milestone 0.3: Domain Model & Database Schema

**Phase:** 0 — Foundation & Tooling  
**Goal:** Core domain entities that all engines will consume and produce.  
**Estimated Tasks:** 20

---

## Tasks

### 1. Create Instruments Table
- [ ] Generate migration: `rails g migration CreateInstruments`
- [ ] Columns:
  - `exchange_segment` (string, index) - NSE, BSE, NFO, BFO
  - `security_id` (string, unique, index) - DhanHQ security ID
  - `trading_symbol` (string, index) - e.g., NIFTY24JAN25000CE
  - `expiry` (date, index) - option expiry date
  - `strike` (decimal, precision: 10, scale: 2, index)
  - `option_type` (string) - CE, PE, XX (for futures/equity)
  - `instrument_type` (string) - OPTIDX, OPTSTK, FUTIDX, FUTSTK, EQ
  - `lot_size` (integer)
  - `tick_size` (decimal, precision: 10, scale: 2)
  - `underlying_symbol` (string, index) - NIFTY, BANKNIFTY
  - `is_active` (boolean, default: true)
  - `metadata` (jsonb) - flexible extra fields
- [ ] Add composite index: `[:exchange_segment, :security_id]`
- [ ] Add composite index: `[:underlying_symbol, :expiry, :option_type, :strike]`

### 2. Create Market Ticks Table (TimescaleDB Hypertable)
- [ ] Generate migration: `rails g migration CreateMarketTicks`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `timestamp` (timestamptz, not null)
  - `price` (decimal, precision: 15, scale: 2, not null)
  - `volume` (bigint)
  - `bid_price` (decimal, precision: 15, scale: 2)
  - `ask_price` (decimal, precision: 15, scale: 2)
  - `bid_size` (integer)
  - `ask_size` (integer)
  - `oi` (bigint) - open interest
  - `metadata` (jsonb) - raw tick data
- [ ] Convert to hypertable: `SELECT create_hypertable('market_ticks', 'timestamp', chunk_time_interval => INTERVAL '1 day')`
- [ ] Create indexes: `[:instrument_id, :timestamp DESC]`
- [ ] Enable compression: `ALTER TABLE market_ticks SET (timescaledb.compress, timescaledb.compress_segmentby = 'instrument_id')`
- [ ] Add compression policy: `SELECT add_compression_policy('market_ticks', INTERVAL '7 days')`

### 3. Create Candles Table
- [ ] Generate migration: `rails g migration CreateCandles`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `timeframe` (string, not null) - 1m, 5m, 15m, 30m, 1h, 1d
  - `timestamp` (timestamptz, not null) - candle open time
  - `open` (decimal, precision: 15, scale: 2)
  - `high` (decimal, precision: 15, scale: 2)
  - `low` (decimal, precision: 15, scale: 2)
  - `close` (decimal, precision: 15, scale: 2)
  - `volume` (bigint)
  - `oi` (bigint) - open interest at close
  - `vwap` (decimal, precision: 15, scale: 2)
  - `trade_count` (integer)
  - `metadata` (jsonb)
- [ ] Convert to hypertable with 1-day chunks
- [ ] Composite index: `[:instrument_id, :timeframe, :timestamp DESC]`
- [ ] Unique constraint: `[:instrument_id, :timeframe, :timestamp]`

### 4. Create Option Chain Snapshots Table
- [ ] Generate migration: `rails g migration CreateOptionChainSnapshots`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null) - underlying instrument
  - `snapshot_time` (timestamptz, not null)
  - `strike` (decimal, precision: 10, scale: 2, not null)
  - `expiry` (date, not null)
  - `option_type` (string, not null) - CE, PE
  - `ce_oi` (bigint) - call open interest
  - `pe_oi` (bigint) - put open interest
  - `ce_volume` (bigint)
  - `pe_volume` (bigint)
  - `iv` (decimal, precision: 8, scale: 4) - implied volatility
  - `delta` (decimal, precision: 8, scale: 4)
  - `gamma` (decimal, precision: 8, scale: 4)
  - `theta` (decimal, precision: 8, scale: 4)
  - `vega` (decimal, precision: 8, scale: 4)
  - `bid` (decimal, precision: 15, scale: 2)
  - `ask` (decimal, precision: 15, scale: 2)
  - `spread` (decimal, precision: 15, scale: 2)
  - `ltp` (decimal, precision: 15, scale: 2)
  - `metadata` (jsonb)
- [ ] Composite index: `[:instrument_id, :snapshot_time DESC, :strike, :option_type]`
- [ ] Convert to hypertable

### 5. Create Market Depth Snapshots Table
- [ ] Generate migration: `rails g migration CreateMarketDepthSnapshots`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `timestamp` (timestamptz, not null)
  - `bid_levels` (jsonb) - array of [price, size] pairs (5 levels)
  - `ask_levels` (jsonb)
  - `bid_imbalance` (decimal, precision: 8, scale: 4)
  - `ask_imbalance` (decimal, precision: 8, scale: 4)
  - `total_bid_volume` (bigint)
  - `total_ask_volume` (bigint)
  - `spread` (decimal, precision: 15, scale: 2)
  - `mid_price` (decimal, precision: 15, scale: 2)
  - `metadata` (jsonb)
- [ ] Convert to hypertable
- [ ] Index: `[:instrument_id, :timestamp DESC]`

### 6. Create Market Regimes Table
- [ ] Generate migration: `rails g migration CreateMarketRegimes`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `timeframe` (string, not null) - 30m, 15m, 5m
  - `regime_type` (string, not null) - trending_up, trending_down, ranging, expanding, compressing, reversing
  - `regime_score` (decimal, precision: 5, scale: 2) - 0-100
  - `adx_value` (decimal, precision: 8, scale: 2)
  - `atr_value` (decimal, precision: 15, scale: 2)
  - `ema_alignment` (string) - bullish, bearish, mixed
  - `detected_at` (timestamptz, not null)
  - `valid_until` (timestamptz)
  - `metadata` (jsonb) - supporting indicators
- [ ] Index: `[:instrument_id, :timeframe, :detected_at DESC]`

### 7. Create Market Structures Table
- [ ] Generate migration: `rails g migration CreateMarketStructures`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `timeframe` (string, not null)
  - `structure_type` (string) - HH, HL, LH, LL, BOS, CHOCH, FVG, OB
  - `price_level` (decimal, precision: 15, scale: 2)
  - `timestamp` (timestamptz, not null) - formation time
  - `confirmed_at` (timestamptz)
  - `is_active` (boolean, default: true)
  - `strength` (decimal, precision: 5, scale: 2) - 0-100
  - `metadata` (jsonb) - swing points, volume, etc.
- [ ] Index: `[:instrument_id, :timeframe, :timestamp DESC]`

### 8. Create Trade Setups Table
- [ ] Generate migration: `rails g migration CreateTradeSetups`
- [ ] Columns:
  - `instrument_id` (bigint, foreign key, not null)
  - `setup_type` (string) - ORB, trend_follow, pullback, liquidity_sweep, breakout, reversal, momentum_continuation, range_expansion
  - `direction` (string) - long, short
  - `confidence` (decimal, precision: 5, scale: 2) - 0-100
  - `entry_price` (decimal, precision: 15, scale: 2)
  - `stop_loss` (decimal, precision: 15, scale: 2)
  - `targets` (jsonb) - array of target prices
  - `features` (jsonb) - all engine outputs at detection time
  - `detected_at` (timestamptz, not null)
  - `expires_at` (timestamptz)
  - `status` (string) - detected, validated, executed, expired, cancelled
  - `metadata` (jsonb)
- [ ] Index: `[:instrument_id, :detected_at DESC]`

### 9. Create Trade Scores Table
- [ ] Generate migration: `rails g migration CreateTradeScores`
- [ ] Columns:
  - `trade_setup_id` (bigint, foreign key, not null)
  - `component_scores` (jsonb) - {context: 85, regime: 90, structure: 75, ...}
  - `total_score` (decimal, precision: 5, scale: 2) - weighted sum
  - `threshold` (decimal, precision: 5, scale: 2) - default 80
  - `passed` (boolean)
  - `calculated_at` (timestamptz, not null)
  - `metadata` (jsonb) - weight breakdown
- [ ] Index: `[:trade_setup_id, :calculated_at DESC]`

### 10. Create Orders Table
- [ ] Generate migration: `rails g migration CreateOrders`
- [ ] Columns:
  - `order_id` (string, unique, not null) - our internal ID
  - `broker_order_id` (string, index) - DhanHQ order ID
  - `trade_setup_id` (bigint, foreign key)
  - `instrument_id` (bigint, foreign key, not null)
  - `status` (string) - pending, submitted, partial, filled, cancelled, rejected, expired
  - `side` (string) - buy, sell
  - `quantity` (integer, not null)
  - `price` (decimal, precision: 15, scale: 2) - limit price
  - `order_type` (string) - limit, market, sl, sl_m
  - `product_type` (string) - intraday, carryforward, co, oco
  - `filled_quantity` (integer, default: 0)
  - `avg_fill_price` (decimal, precision: 15, scale: 2)
  - `submitted_at` (timestamptz)
  - `filled_at` (timestamptz)
  - `cancelled_at` (timestamptz)
  - `rejection_reason` (text)
  - `metadata` (jsonb) - broker response, retry count
- [ ] Index: `[:instrument_id, :submitted_at DESC]`

### 11. Create Positions Table
- [ ] Generate migration: `rails g migration CreatePositions`
- [ ] Columns:
  - `position_id` (string, unique, not null)
  - `order_id` (bigint, foreign key) - entry order
  - `instrument_id` (bigint, foreign key, not null)
  - `entry_price` (decimal, precision: 15, scale: 2)
  - `quantity` (integer, not null)
  - `side` (string) - long, short
  - `unrealized_pnl` (decimal, precision: 15, scale: 2, default: 0)
  - `realized_pnl` (decimal, precision: 15, scale: 2, default: 0)
  - `stop_loss` (decimal, precision: 15, scale: 2)
  - `targets` (jsonb)
  - `status` (string) - open, closing, closed, stopped_out
  - `opened_at` (timestamptz, not null)
  - `closed_at` (timestamptz)
  - `metadata` (jsonb) - greeks at entry, regime, etc.
- [ ] Index: `[:instrument_id, :status, :opened_at DESC]`

### 12. Create Trades Table (Closed Positions)
- [ ] Generate migration: `rails g migration CreateTrades`
- [ ] Columns:
  - `position_id` (string, not null)
  - `instrument_id` (bigint, foreign key, not null)
  - `entry_time` (timestamptz, not null)
  - `exit_time` (timestamptz, not null)
  - `entry_price` (decimal, precision: 15, scale: 2)
  - `exit_price` (decimal, precision: 15, scale: 2)
  - `quantity` (integer)
  - `side` (string)
  - `gross_pnl` (decimal, precision: 15, scale: 2)
  - `net_pnl` (decimal, precision: 15, scale: 2) - after fees
  - `mfe` (decimal, precision: 15, scale: 2) - max favorable excursion
  - `mae` (decimal, precision: 15, scale: 2) - max adverse excursion
  - `holding_time_seconds` (integer)
  - `outcome` (string) - win, loss, breakeven
  - `exit_reason` (string) - target, stop_loss, time, manual, emergency
  - `fees` (decimal, precision: 15, scale: 2)
  - `slippage` (decimal, precision: 15, scale: 2)
  - `metadata` (jsonb)
- [ ] Index: `[:instrument_id, :entry_time DESC]`

### 13. Create Trade Features Table
- [ ] Generate migration: `rails g migration CreateTradeFeatures`
- [ ] Columns:
  - `trade_id` (bigint, foreign key, not null)
  - `entry_features` (jsonb) - all feature values at entry
  - `exit_features` (jsonb) - feature values at exit
  - `regime_at_entry` (string)
  - `regime_at_exit` (string)
  - `greeks_at_entry` (jsonb) - delta, gamma, theta, vega
  - `greeks_at_exit` (jsonb)
  - `market_structure_at_entry` (jsonb)
  - `metadata` (jsonb)
- [ ] Index: `[:trade_id]`

### 14. Create AI Analyses Table
- [ ] Generate migration: `rails g migration CreateAiAnalyses`
- [ ] Columns:
  - `analysis_type` (string) - setup_validation, market_analysis, trade_review, journal, research
  - `related_type` (string) - TradeSetup, Trade, Position, MarketRegime
  - `related_id` (bigint)
  - `model_used` (string) - gpt-4, llama3, etc.
  - `prompt` (text)
  - `response` (text)
  - `latency_ms` (integer)
  - `tokens_used` (integer)
  - `cost_usd` (decimal, precision: 10, scale: 6)
  - `confidence` (decimal, precision: 5, scale: 2)
  - `created_at` (timestamptz, not null)
  - `metadata` (jsonb)
- [ ] Index: `[:related_type, :related_id, :created_at DESC]`

### 15. Create Learning Records Table
- [ ] Generate migration: `rails g migration CreateLearningRecords`
- [ ] Columns:
  - `strategy_name` (string, not null)
  - `time_window` (string) - daily, weekly, monthly, all_time
  - `window_start` (date, not null)
  - `window_end` (date, not null)
  - `expectancy` (decimal, precision: 8, scale: 4)
  - `win_rate` (decimal, precision: 5, scale: 2)
  - `avg_rr` (decimal, precision: 8, scale: 4) - average risk/reward
  - `profit_factor` (decimal, precision: 8, scale: 4)
  - `sharpe_ratio` (decimal, precision: 8, scale: 4)
  - `max_drawdown` (decimal, precision: 15, scale: 2)
  - `sample_size` (integer)
  - `total_pnl` (decimal, precision: 15, scale: 2)
  - `metadata` (jsonb) - per-regime breakdown, etc.
- [ ] Unique index: `[:strategy_name, :time_window, :window_start, :window_end]`

### 16. Add Composite Indexes on All Time-Series Tables
- [ ] Review all hypertable indexes
- [ ] Add `pg_trgm` extension for fuzzy symbol matching
- [ ] Create GIN indexes on jsonb metadata columns where queried

### 17. Create Database Migration Rollback Tests
- [ ] Add test helper for migration up/down
- [ ] Test each migration rolls back cleanly
- [ ] Verify no data loss on rollback (where applicable)

### 18. Seed Instrument Master Data
- [ ] Create `db/seeds/instruments.rb`
- [ ] Load NIFTY, BANKNIFTY, SENSEX, FINNIFTY instruments
- [ ] Include all active expiries and strikes (ATM ± 20)
- [ ] Run via `rails db:seed`

### 19. Add Database Consistency Gem
- [ ] Add `database_consistency` to Gemfile
- [ ] Configure to check foreign keys, null constraints, enum consistency
- [ ] Run in CI pipeline

---

## Acceptance Criteria
- [ ] All 15 migrations run without errors
- [ ] `rails db:schema:dump` produces clean schema.rb
- [ ] TimescaleDB hypertables created and compression policies active
- [ ] Seed data loads 4 indices with ~2000 instruments each
- [ ] All foreign keys reference correctly
- [ ] `database_consistency` passes with zero errors
- [ ] Rollback tests pass for all migrations

---

## Notes
- Run `rails db:migrate` after each migration to catch errors early
- Use `structure.sql` instead of `schema.rb` for TimescaleDB features
- Consider partitioning strategy for very large tables
- Document schema in `docs/database_schema.md` with ERD