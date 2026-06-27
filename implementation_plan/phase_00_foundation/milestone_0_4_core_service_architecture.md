# Milestone 0.4: Core Service Architecture

**Phase:** 0 — Foundation & Tooling  
**Goal:** Clean, testable service layer with dependency injection.  
**Estimated Tasks:** 16

---

## Tasks

### 1. Create Service Directory Structure
- [ ] Create `app/services/` with subdirectories matching engine taxonomy:
  ```
  app/services/
  ├── base_service.rb
  ├── result.rb
  ├── errors/
  │   ├── validation_error.rb
  │   ├── broker_error.rb
  │   ├── data_error.rb
  │   └── risk_error.rb
  ├── engines/
  │   ├── market_context_engine.rb
  │   ├── market_regime_engine.rb
  │   ├── market_structure_engine.rb
  │   ├── momentum_engine.rb
  │   ├── liquidity_engine.rb
  │   ├── option_intelligence_engine.rb
  │   ├── strike_selection_engine.rb
  │   ├── trade_scoring_engine.rb
  │   ├── risk_validation_engine.rb
  │   ├── execution_engine.rb
  │   └── position_management_engine.rb
  ├── repositories/
  │   ├── instrument_repository.rb
  │   ├── tick_repository.rb
  │   ├── candle_repository.rb
  │   └── option_chain_repository.rb
  ├── gateway/
  │   ├── ai_gateway.rb
  │   └── broker_gateway.rb
  ├── calculations/
  │   ├── statistics.rb
  │   ├── indicators/
  │   └── options/
  ├── validators/
  └── learning/
  ```

### 2. Implement BaseService
- [ ] Create `app/services/base_service.rb`
- [ ] Interface: `call(*args, **kwargs) -> Result`
- [ ] Built-in error handling with `ServiceError` hierarchy
- [ ] Logging: start, success, failure with timing
- [ ] Transaction management option
- [ ] Dry-rb `do` notation support

### 3. Implement Result Object Pattern
- [ ] Create `app/services/result.rb`
- [ ] Methods: `success?`, `failure?`, `value`, `error`, `errors`
- [ ] Factory methods: `Result.success(value)`, `Result.failure(error)`
- [ ] Support for multiple errors
- [ ] Monadic `and_then`, `or_else` chaining

### 4. Create ServiceError Hierarchy
- [ ] Base: `ServiceError < StandardError`
- [ ] `ValidationError` - input validation failures
- [ ] `BrokerError` - DhanHQ API errors (with error codes)
- [ ] `DataError` - missing/invalid market data
- [ ] `RiskError` - risk rule violations
- [ ] Each error carries: code, message, context hash, severity

### 5. Implement InstrumentRepository
- [ ] Create `app/services/repositories/instrument_repository.rb`
- [ ] Methods:
  - `find_by_symbol(symbol)` - exact match
  - `find_by_security_id(security_id)`
  - `find_atm_strikes(underlying, expiry, count: 5)` - ATM ± N
  - `find_by_underlying_and_expiry(underlying, expiry)`
  - `active_underlyings` - distinct underlying symbols
  - `expiries_for(underlying)` - sorted upcoming expiries
- [ ] Cache frequent lookups in Redis (TTL: 1 hour)
- [ ] Handle instrument refresh from DhanHQ

### 6. Implement TickRepository
- [ ] Create `app/services/repositories/tick_repository.rb`
- [ ] Methods:
  - `latest(instrument_id, limit: 100)`
  - `range(instrument_id, from:, to:)`
  - `aggregate_to_candles(instrument_id, timeframe, from:, to:)`
  - `count_gaps(instrument_id, from:, to:, expected_interval:)`
- [ ] Use TimescaleDB continuous aggregates where possible
- [ ] Batch insert via `COPY` for ingestion jobs

### 7. Implement CandleRepository
- [ ] Create `app/services/repositories/candle_repository.rb`
- [ ] Methods:
  - `latest(instrument_id, timeframe, limit: 500)`
  - `range(instrument_id, timeframe, from:, to:)`
  - `vwap_range(instrument_id, timeframe, from:, to:)`
  - `previous_close(instrument_id, timeframe, as_of:)`
  - `opening_range(instrument_id, minutes: 15)`
- [ ] Efficient time-range queries using hypertable indexes
- [ ] Support for multiple timeframes in single query

### 8. Implement OptionChainRepository
- [ ] Create `app/services/repositories/option_chain_repository.rb`
- [ ] Methods:
  - `latest_snapshot(underlying, expiry)`
  - `snapshot_at(underlying, expiry, timestamp)`
  - `strike_range(underlying, expiry, from_strike:, to_strike:)`
  - `atm_strikes(underlying, expiry, count: 5)`
  - `greeks_for(underlying, expiry, strike, option_type)`
  - `oi_change(underlying, expiry, from:, to:)`
- [ ] Optimized for strike-range queries

### 9. Create Engines Directory
- [ ] Create `app/engines/` for deterministic trading engines
- [ ] Each engine: single responsibility, stateless, pure functions where possible
- [ ] Base engine interface: `analyze(context) -> EngineOutput`
- [ ] Engine output: structured data class with `to_h` for serialization

### 10. Create Gateway Directory
- [ ] Create `app/gateway/` for external abstractions
- [ ] `AiGateway` - multi-provider LLM routing
- [ ] `BrokerGateway` - DhanHQ REST + WebSocket abstraction
- [ ] Gateway pattern: hide external API complexity

### 11. Implement EventBus
- [ ] Create `app/services/event_bus.rb` using Redis pub/sub
- [ ] Channels: `market.ticks`, `market.candles`, `orders.updates`, `positions.updates`, `engine.signals`
- [ ] Publish: `EventBus.publish(channel, payload)`
- [ ] Subscribe: `EventBus.subscribe(channel) { |msg| ... }`
- [ ] JSON serialization with schema versioning
- [ ] Dead letter queue for failed handlers

### 12. Add dry-container for Dependency Injection
- [ ] Add `dry-container` and `dry-auto_inject` to Gemfile
- [ ] Create `app/container.rb` registering all services
- [ ] Example registrations:
  - `instrument_repository` -> `InstrumentRepository.new`
  - `tick_repository` -> `TickRepository.new`
  - `market_context_engine` -> `MarketContextEngine.new`
  - `ai_gateway` -> `AiGateway.new`
- [ ] Use `Import` in services for explicit dependencies

### 13. Create Calculations Library
- [ ] Create `app/lib/calculations/statistics.rb` module
- [ ] Functions:
  - `sma(values, period)`
  - `ema(values, period, previous_ema: nil)`
  - `std_dev(values, period)`
  - `percentile(values, p)` - p in 0..100
  - `z_score(value, mean, std_dev)`
  - `percentile_rank(values, value)`
  - `correlation(series_a, series_b)`
  - `covariance(series_a, series_b)`
- [ ] Pure functions, no side effects, fully tested

### 14. Add Validators
- [ ] Create `app/services/validators/` directory
- [ ] `InputValidator` - dry-validation schemas for engine inputs
- [ ] `MarketDataValidator` - check data freshness, continuity
- [ ] `OrderValidator` - validate order parameters before submission
- [ ] `RiskValidator` - pre-trade risk checks

### 15. Document Service Boundaries
- [ ] Create `docs/architecture.md` with:
  - Service layer diagram
  - Dependency graph (container registrations)
  - Data flow: Market Data → Engines → Strategy → Scoring → Risk → Execution
  - Event bus channels and payloads
  - Testing strategies for each layer

### 16. Add Interface Tests
- [ ] Create `spec/services/shared_examples/base_service.rb`
- [ ] Shared examples for: success/failure handling, logging, timing
- [ ] Contract tests for each repository interface
- [ ] Contract tests for each engine interface

---

## Acceptance Criteria
- [ ] All services load without circular dependencies
- [ ] Container resolves all dependencies in test environment
- [ ] `BaseService.call` returns `Result` object consistently
- [ ] EventBus publishes/subscribes work in test and development
- [ ] Statistics module functions pass mathematical accuracy tests
- [ ] Repository interfaces tested with real database
- [ ] `docs/architecture.md` is current and accurate

---

## Notes
- Keep services small (< 100 lines); decompose if larger
- Prefer composition over inheritance
- Use `dry-monads` for Result chaining if needed
- All external I/O (DB, Redis, HTTP) behind repository/gateway interfaces
- Engine interfaces must be deterministic for replay testing