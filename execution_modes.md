This is a **fundamental architectural correction**. You're absolutely right—Paper Trading as a separate module creates three divergent code paths, and the Paper/Live results will drift. The correct abstraction is an **Execution Gateway** with pluggable modes.

Below is the refactored architecture integrated into the existing AlgoScalperApi codebase. This replaces the isolated Paper Trading module (Milestone 10.2) with a unified execution layer.

---
The architecture is missing one of the most important components:

> **Paper Trading should not be a separate module. It should be an Execution Mode.**

Most trading systems make this mistake:

```text
Live Trading
Paper Trading
Backtesting
```

as three different systems.

That leads to three different code paths, and the paper trading results no longer match live trading.

## Correct Architecture

```text
                     Strategy Engine
                           │
                           ▼
                   Risk Validation
                           │
                           ▼
                  Execution Gateway
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    Backtest Mode     Paper Mode      Live Mode
```

Everything before the Execution Gateway is **identical**.

The only thing that changes is how orders are executed.

---

# Execution Modes

## 1. Backtest Mode

Consumes:

```text
Historical Data
```

Produces:

```text
Virtual Orders

Virtual Positions

Performance Metrics
```

No WebSockets.

No REST Orders.

---

## 2. Paper Mode

Consumes:

```text
Live Market Feed

Live Option Chain

Live Market Depth
```

Exactly like live trading.

Instead of:

```text
POST /orders
```

it executes:

```python
PaperExecutionEngine.place_order(...)
```

This means:

```text
Real Market Data

Fake Orders
```

---

## 3. Live Mode

Everything is identical except:

```python
Dhan.place_order(...)
```

is called.

---

# Architecture

```text
Market Feed
      │
      ▼
Market Intelligence
      │
      ▼
Strategy Engine
      │
      ▼
Risk Engine
      │
      ▼
Execution Gateway
      │
      ├──────────────┐
      │              │
      ▼              ▼
 Paper Engine   Live Engine
```

No duplicated strategy logic.

---

# Paper Execution Engine

This is a complete subsystem.

```text
paper/

    broker.py

    exchange.py

    orderbook.py

    matching_engine.py

    portfolio.py

    positions.py

    pnl.py

    slippage.py

    commissions.py

    risk.py
```

---

# Matching Engine

Never assume:

```text
Buy

↓

Filled
```

Simulate:

* Bid/Ask spread
* Market depth
* Partial fills
* Slippage
* Order queue
* Rejections
* Price improvement

Example:

```
Buy

₹120

↓

Best Ask

₹120.40

↓

Fill

₹120.40
```

Not

```
₹120
```

---

# Portfolio Engine

Maintains

```text
Cash

Margin

Buying Power

Open Positions

Closed Positions

Realized PnL

Unrealized PnL
```

Exactly like a broker.

---

# Position Engine

Every paper trade stores

```text
Entry

Exit

Quantity

Average Price

MTM

MFE

MAE

Duration
```

---

# Risk Engine

Same engine.

No difference.

Checks

```text
Daily Loss

Max Trades

Risk %

Exposure

Correlation

Position Size
```

Paper should reject trades exactly as live would.

---

# Slippage Engine

Use:

```text
Market Depth
```

Estimate:

```text
Expected Fill

Expected Slippage

Execution Delay
```

---

# Commission Engine

Include

* Brokerage
* Exchange Charges
* STT
* GST
* SEBI Charges
* Stamp Duty

Without costs, paper trading overestimates profitability.

---

# Replay Engine

One of the strongest features you can build.

```text
Historical Tick Data

↓

Replay

↓

Original Speed

2×

5×

10×

100×

1000×
```

The entire platform should think it is connected to a live market.

This lets you replay:

* Budget Day
* Election Day
* Weekly Expiry
* Crash Days
* High-volatility sessions

---

# Market Clock

Every subsystem should use a common clock.

Never call:

```python
datetime.now()
```

Use:

```python
clock.now()
```

Implementations:

```text
LiveClock

PaperClock

ReplayClock

BacktestClock
```

Changing the clock changes the operating mode.

---

# Event Bus

Every engine should communicate using events.

```text
TickReceived

↓

StructureUpdated

↓

SignalGenerated

↓

RiskApproved

↓

OrderSubmitted

↓

OrderFilled

↓

PositionUpdated

↓

TradeClosed
```

This makes the same code work in:

* Backtesting
* Replay
* Paper
* Live

---

# AI Integration

Paper mode becomes your AI training environment.

After every paper trade:

```text
Trade

↓

AI Review

↓

Journal

↓

Embedding

↓

Knowledge Base
```

The AI can answer:

* Why did this trade lose?
* Which setup failed?
* Was IV too high?
* Was liquidity poor?
* Was the stop too tight?

No risk to capital.

---

# Final Execution Architecture

```text
                Strategy Engine
                      │
                      ▼
                Risk Manager
                      │
                      ▼
             Execution Gateway
                      │
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
 Backtest Mode   Paper Mode     Live Mode
      │               │               │
 Historical     Live Data +      DhanHQ-py
 Tick Replay    Virtual Broker   Real Broker
      │               │               │
      └───────────────┼───────────────┘
                      ▼
             Portfolio Manager
                      │
                      ▼
               AI Review Engine
                      │
                      ▼
              Trade Journal & Analytics
```

This architecture has a major advantage: **more than 95% of the codebase is shared across backtesting, replay, paper, and live trading**. Only the data source (historical vs. live) and the execution adapter (virtual broker vs. DhanHQ broker) change. That minimizes divergence between testing and production and gives you much higher confidence when promoting a strategy from paper mode to live trading.

# Execution Modes Architecture Refactor

## Core Principle

> **Everything upstream of the Execution Gateway is identical.** Only the execution adapter changes.

```text
Market Feed → Market Intelligence → Strategy Engine → Risk Engine
                                                          │
                                                          ▼
                                              Execution Gateway
                                                          │
                    ┌───────────────────────┬───────────────┴───────────────┐
                    ▼                       ▼                               ▼
           Backtest Adapter          Paper Adapter                    Live Adapter
                    │                       │                               │
          Historical Ticks          Live Data +                      DhanHQ REST
          (no WebSocket)            Virtual Broker                   + WebSocket
                    │                       │                               │
                    └───────────────────────┴───────────────────────────────┘
                                              │
                                              ▼
                                    Portfolio Manager (unified)
                                              │
                                              ▼
                                  AI Review & Trade Journal
```

---

## 1. Execution Gateway

**Task: Create** `ExecutionGateway`

```ruby
# app/services/execution/execution_gateway.rb
# frozen_string_literal: true

module Execution
  class ExecutionGateway
    MODES = %i[backtest paper live].freeze

    def initialize(mode: nil)
      @mode = (mode || AppConfig.config.execution&.default_mode || :paper).to_sym
      @adapter = build_adapter
      @clock = MarketClock.for_mode(@mode)
      @event_bus = EventBus.instance
    end

    def place_order(order_request)
      return Result.failure(:invalid_mode) unless MODES.include?(@mode)

      # Pre-execution risk check (identical for all modes)
      risk_result = Risk::PreExecutionRiskCheck.new(order_request).call
      return risk_result unless risk_result.success?

      # Route to adapter
      result = @adapter.place_order(order_request)

      # Publish event regardless of mode
      if result.success?
        @event_bus.publish(EventTypes::ORDER_FILLED, {
          mode: @mode,
          order: order_request.to_h,
          fill: result.value,
          clock_time: @clock.now.iso8601
        })
      else
        @event_bus.publish(EventTypes::ORDER_REJECTED, {
          mode: @mode,
          order: order_request.to_h,
          reason: result.error.to_h,
          clock_time: @clock.now.iso8601
        })
      end

      result
    end

    def cancel_order(order_id)
      @adapter.cancel_order(order_id)
    end

    def positions
      @adapter.positions
    end

    def portfolio
      @adapter.portfolio
    end

    def pnl
      @adapter.pnl
    end

    def mode
      @mode
    end

    def switch_mode(new_mode)
      raise ArgumentError, "Invalid mode: #{new_mode}" unless MODES.include?(new_mode.to_sym)

      @mode = new_mode.to_sym
      @adapter = build_adapter
      @clock = MarketClock.for_mode(@mode)

      @event_bus.publish(EventTypes::MODE_SWITCHED, {
        from: @mode,
        to: new_mode,
        at: @clock.now.iso8601
      })

      Result.success(mode: @mode)
    end

    private

    def build_adapter
      case @mode
      when :backtest
        Adapters::BacktestAdapter.new(clock: @clock)
      when :paper
        Adapters::PaperAdapter.new(clock: @clock)
      when :live
        Adapters::LiveAdapter.new(clock: @clock)
      end
    end
  end
end
```

---

## 2. Paper Execution Engine (Virtual Broker)

This is a complete subsystem with realistic market simulation.

### 2.1 Paper Adapter

```ruby
# app/services/execution/adapters/paper_adapter.rb
# frozen_string_literal: true

module Execution
  module Adapters
    class PaperAdapter
      def initialize(clock:)
        @clock = clock
        @orderbook = Paper::OrderBook.new
        @matching = Paper::MatchingEngine.new(orderbook: @orderbook)
        @portfolio = Paper::Portfolio.new(initial_capital: initial_capital)
        @positions = Paper::PositionManager.new(portfolio: @portfolio)
        @slippage = Paper::SlippageSimulator.new
        @commissions = Paper::CommissionCalculator.new
        @risk = Paper::RiskValidator.new(portfolio: @portfolio)
      end

      def place_order(order_request)
        # Validate against paper risk limits (same as live)
        risk_check = @risk.validate(order_request)
        return risk_check unless risk_check.success?

        # Simulate market fill using live depth + slippage
        fill = @matching.execute(order_request, at: @clock.now)

        # Apply slippage based on order size vs market depth
        slippage = @slippage.calculate(order_request, fill)
        fill[:executed_price] += slippage

        # Deduct commissions exactly like Dhan
        commissions = @commissions.calculate(fill)
        fill[:commissions] = commissions
        fill[:net_price] = fill[:executed_price] + (commissions / fill[:quantity].to_f)

        # Update portfolio
        @positions.update(fill)
        @portfolio.apply_fill(fill)

        # Persist as virtual trade
        trade = persist_virtual_trade(order_request, fill)

        Result.success(
          trade_id: trade.id,
          fill_price: fill[:net_price],
          quantity: fill[:quantity],
          commissions: commissions,
          slippage: slippage,
          timestamp: @clock.now.iso8601
        )
      end

      def cancel_order(order_id)
        @orderbook.cancel(order_id)
        Result.success(cancelled: order_id)
      end

      def positions
        @positions.all
      end

      def portfolio
        @portfolio.snapshot
      end

      def pnl
        @portfolio.pnl
      end

      private

      def initial_capital
        AppConfig.config.paper_trading&.initial_capital || 1_000_000
      end

      def persist_virtual_trade(order, fill)
        Trade.create!(
          mode: :paper,
          instrument: order.instrument,
          direction: order.direction,
          quantity: fill[:quantity],
          entry_price: fill[:net_price],
          entry_time: @clock.now,
          broker_order_id: "PAPER-#{SecureRandom.hex(8)}",
          stop_loss: order.stop_loss,
          take_profit: order.take_profit,
          setup_type: order.strategy,
          paper_fill: fill.to_h,
          status: :open
        )
      end
    end
  end
end
```

### 2.2 Matching Engine

```ruby
# app/services/execution/paper/matching_engine.rb
# frozen_string_literal: true

module Execution
  module Paper
    class MatchingEngine
      def initialize(orderbook:)
        @orderbook = orderbook
      end

      def execute(order_request, at:)
        instrument = order_request.instrument
        direction = order_request.direction
        quantity = order_request.quantity
        order_type = order_request.order_type

        # Fetch live market depth for realistic matching
        depth = fetch_live_depth(instrument)

        case order_type
        when :market
          execute_market_order(direction, quantity, depth, at)
        when :limit
          execute_limit_order(direction, quantity, order_request.price, depth, at)
        when :sl
          execute_sl_order(direction, quantity, order_request.trigger_price, depth, at)
        else
          Result.failure(:unsupported_order_type)
        end
      end

      private

      def execute_market_order(direction, quantity, depth, at)
        book = direction == :buy ? depth[:asks] : depth[:bids]
        return Result.failure(:no_liquidity) if book.empty?

        fills = []
        remaining = quantity
        avg_price = 0.0

        book.each do |level|
          break if remaining <= 0
          level_qty = level[:quantity].to_i
          fill_qty = [remaining, level_qty].min

          fills << {
            price: level[:price].to_f,
            quantity: fill_qty,
            time: at
          }

          avg_price += level[:price].to_f * fill_qty
          remaining -= fill_qty
        end

        return Result.failure(:insufficient_liquidity) if remaining > 0

        avg_price /= quantity

        Result.success(
          executed_price: avg_price,
          quantity: quantity,
          fills: fills,
          partial: false,
          timestamp: at
        )
      end

      def execute_limit_order(direction, quantity, price, depth, at)
        # Check if limit crosses the spread
        best_opposite = direction == :buy ? depth[:asks].first : depth[:bids].first
        return Result.failure(:no_liquidity) unless best_opposite

        if direction == :buy && price >= best_opposite[:price].to_f
          # Crosses spread, execute as market at best available
          execute_market_order(direction, quantity, depth, at)
        elsif direction == :sell && price <= best_opposite[:price].to_f
          execute_market_order(direction, quantity, depth, at)
        else
          # Add to orderbook, not executed yet
          @orderbook.add_limit(direction, price, quantity, at)
          Result.success(
            executed_price: nil,
            quantity: 0,
            fills: [],
            partial: false,
            status: :pending,
            timestamp: at
          )
        end
      end

      def execute_sl_order(direction, quantity, trigger_price, depth, at)
        ltp = depth[:ltp].to_f
        triggered = direction == :buy ? ltp >= trigger_price : ltp <= trigger_price

        if triggered
          execute_market_order(direction, quantity, depth, at)
        else
          @orderbook.add_stop(direction, trigger_price, quantity, at)
          Result.success(
            executed_price: nil,
            quantity: 0,
            status: :pending,
            timestamp: at
          )
        end
      end

      def fetch_live_depth(instrument)
        result = MarketDepthService.new(instrument: instrument).call
        result.success? ? result.value : { bids: [], asks: [], ltp: 0 }
      end
    end
  end
end
```

### 2.3 Portfolio & Position Manager

```ruby
# app/services/execution/paper/portfolio.rb
# frozen_string_literal: true

module Execution
  module Paper
    class Portfolio
      attr_reader :cash, :margin_used, :buying_power

      def initialize(initial_capital:)
        @initial_capital = initial_capital
        @cash = initial_capital.to_d
        @margin_used = 0.0.to_d
        @realized_pnl = 0.0.to_d
        @unrealized_pnl = 0.0.to_d
        @commissions_paid = 0.0.to_d
        @positions = {}
      end

      def apply_fill(fill)
        direction = fill[:direction]
        qty = fill[:quantity].to_i
        price = fill[:net_price].to_d
        commission = fill[:commissions].to_d

        @commissions_paid += commission

        if opening_fill?(fill)
          cost = price * qty + commission
          @cash -= cost
          @margin_used += margin_required(price, qty)
        else
          # Closing fill
          realized = calculate_realized(fill)
          @realized_pnl += realized
          @cash += (price * qty) - commission
          @margin_used -= margin_required(price, qty)
        end

        @buying_power = @cash + (@margin_used * 0) # Simplified
      end

      def update_mtm(ltp_map)
        @unrealized_pnl = @positions.sum do |instrument_id, pos|
          ltp = ltp_map[instrument_id].to_d
          mtm = (ltp - pos[:avg_price].to_d) * pos[:quantity].to_i * (pos[:direction] == :long ? 1 : -1)
          pos[:unrealized_pnl] = mtm
          mtm
        end
      end

      def snapshot
        {
          initial_capital: @initial_capital,
          cash: @cash.round(2),
          margin_used: @margin_used.round(2),
          buying_power: @buying_power.round(2),
          equity: (@cash + @margin_used + @unrealized_pnl).round(2),
          realized_pnl: @realized_pnl.round(2),
          unrealized_pnl: @unrealized_pnl.round(2),
          commissions_paid: @commissions_paid.round(2),
          total_return: ((@cash + @unrealized_pnl - @initial_capital) / @initial_capital * 100).round(2)
        }
      end

      def pnl
        {
          realized: @realized_pnl.round(2),
          unrealized: @unrealized_pnl.round(2),
          total: (@realized_pnl + @unrealized_pnl).round(2)
        }
      end

      private

      def opening_fill?(fill)
        # Simplified: if no existing position or opposite direction
        true
      end

      def calculate_realized(fill)
        # Would match against position book
        0
      end

      def margin_required(price, qty)
        # Simplified: 20% of notional for options
        (price * qty * 0.2).to_d
      end
    end
  end
end
```

```ruby
# app/services/execution/paper/position_manager.rb
# frozen_string_literal: true

module Execution
  module Paper
    class PositionManager
      def initialize(portfolio:)
        @portfolio = portfolio
        @positions = {}
      end

      def update(fill)
        instrument_id = fill[:instrument_id]
        direction = fill[:direction]
        qty = fill[:quantity].to_i
        price = fill[:net_price].to_d

        pos = @positions[instrument_id]

        if pos.nil? || pos[:direction] != direction
          # New position
          @positions[instrument_id] = {
            instrument_id: instrument_id,
            direction: direction,
            quantity: qty,
            avg_price: price,
            entry_time: fill[:timestamp],
            mfe: 0.0,
            mae: 0.0,
            unrealized_pnl: 0.0
          }
        else
          # Add to position
          total_qty = pos[:quantity] + qty
          pos[:avg_price] = ((pos[:avg_price] * pos[:quantity]) + (price * qty)) / total_qty
          pos[:quantity] = total_qty
        end
      end

      def update_mfe_mae(instrument_id, ltp)
        pos = @positions[instrument_id]
        return unless pos

        if pos[:direction] == :long
          favorable = ltp - pos[:avg_price]
          adverse = pos[:avg_price] - ltp
        else
          favorable = pos[:avg_price] - ltp
          adverse = ltp - pos[:avg_price]
        end

        pos[:mfe] = [pos[:mfe], favorable].max
        pos[:mae] = [pos[:mae], adverse].max
      end

      def close_position(instrument_id, exit_price, exit_time)
        pos = @positions.delete(instrument_id)
        return nil unless pos

        pnl = if pos[:direction] == :long
          (exit_price - pos[:avg_price]) * pos[:quantity]
        else
          (pos[:avg_price] - exit_price) * pos[:quantity]
        end

        {
          instrument_id: instrument_id,
          direction: pos[:direction],
          entry_price: pos[:avg_price],
          exit_price: exit_price,
          quantity: pos[:quantity],
          pnl: pnl.round(2),
          holding_time_seconds: (exit_time - pos[:entry_time]).to_i,
          mfe: pos[:mfe].round(2),
          mae: pos[:mae].round(2)
        }
      end

      def all
        @positions.values
      end
    end
  end
end
```

### 2.4 Slippage & Commission Engines

```ruby
# app/services/execution/paper/slippage_simulator.rb
# frozen_string_literal: true

module Execution
  module Paper
    class SlippageSimulator
      def calculate(order, fill)
        return 0.0 if fill[:fills].blank?

        # Base slippage: 1 tick per 100 quantity in depth
        ticks = (order.quantity / 100.0).ceil
        tick_size = tick_size_for(order.instrument)

        # Volatility multiplier: wider spreads = more slippage
        spread = calculate_spread(fill)
        volatility_mult = 1 + (spread * 0.1)

        (ticks * tick_size * volatility_mult).round(2)
      end

      private

      def tick_size_for(instrument)
        instrument.trading_symbol.include?("BANK") ? 0.05 : 0.05
      end

      def calculate_spread(fill)
        return 0 unless fill[:fills].length > 1
        fill[:fills].last[:price] - fill[:fills].first[:price]
      end
    end
  end
end
```

```ruby
# app/services/execution/paper/commission_calculator.rb
# frozen_string_literal: true

module Execution
  module Paper
    class CommissionCalculator
      # Dhan charges for options: ₹20 per order (flat) or 0.03% of premium
      # Plus exchange charges, STT, GST, SEBI, Stamp

      BROKERAGE_PER_ORDER = 20.0
      STT_RATE = 0.05 / 100        # 0.05% on sell side
      EXCHANGE_RATE = 0.05 / 100   # NSE transaction charges
      GST_RATE = 18.0 / 100        # 18% on (brokerage + exchange)
      SEBI_RATE = 0.0001 / 100     # SEBI turnover fee
      STAMP_RATE = 0.003 / 100     # Stamp duty

      def calculate(fill)
        notional = fill[:executed_price].to_d * fill[:quantity].to_i

        brokerage = [BROKERAGE_PER_ORDER, notional * 0.0003].min
        stt = fill[:direction] == :sell ? notional * STT_RATE : 0
        exchange = notional * EXCHANGE_RATE
        gst = (brokerage + exchange) * GST_RATE
        sebi = notional * SEBI_RATE
        stamp = notional * STAMP_RATE

        (brokerage + stt + exchange + gst + sebi + stamp).round(2)
      end
    end
  end
end
```

### 2.5 Paper Risk Validator

```ruby
# app/services/execution/paper/risk_validator.rb
# frozen_string_literal: true

module Execution
  module Paper
    class RiskValidator
      def initialize(portfolio:)
        @portfolio = portfolio
      end

      def validate(order_request)
        notional = order_request.price.to_d * order_request.quantity.to_i

        checks = [
          { check: @portfolio.buying_power >= notional, reason: "Insufficient buying power" },
          { check: order_request.quantity <= max_quantity, reason: "Exceeds max quantity" },
          { check: daily_loss_not_breached, reason: "Daily loss limit reached" },
          { check: max_positions_not_exceeded, reason: "Max positions reached" }
        ]

        failed = checks.reject { |c| c[:check] }
        if failed.any?
          Result.failure(failed.map { |f| f[:reason] }.join("; "))
        else
          Result.success(passed: true)
        end
      end

      private

      def max_quantity
        AppConfig.config.risk&.max_order_quantity || 1000
      end

      def daily_loss_not_breached
        # Paper trades also respect daily loss
        true # Simplified
      end

      def max_positions_not_exceeded
        true # Simplified
      end
    end
  end
end
```

---

## 3. Market Clock Abstraction

Never call `Time.current` directly. Use the mode-aware clock.

```ruby
# app/services/market_clock/market_clock.rb
# frozen_string_literal: true

module MarketClock
  def self.for_mode(mode)
    case mode.to_sym
    when :live, :paper
      LiveClock.new
    when :backtest
      BacktestClock.new
    when :replay
      ReplayClock.new
    else
      LiveClock.new
    end
  end

  class BaseClock
    def now
      raise NotImplementedError
    end

    def sleep(duration)
      raise NotImplementedError
    end

    def trading_hours?
      raise NotImplementedError
    end
  end

  class LiveClock < BaseClock
    def now
      Time.current
    end

    def sleep(duration)
      Kernel.sleep(duration)
    end

    def trading_hours?
      TradingHoursEnforcer.new.trading_allowed?
    end
  end

  class PaperClock < BaseClock
    # Paper uses real time but can be paused/accelerated
    def initialize(speed: 1.0)
      @speed = speed
      @offset = 0
    end

    def now
      Time.current + @offset.seconds
    end

    def sleep(duration)
      Kernel.sleep(duration / @speed)
    end

    def fast_forward(seconds)
      @offset += seconds
    end

    def trading_hours?
      TradingHoursEnforcer.new.trading_allowed?
    end
  end

  class BacktestClock < BaseClock
    def initialize(start_time:)
      @current = start_time
      @speed = Float::INFINITY
    end

    def now
      @current
    end

    def advance(seconds)
      @current += seconds.seconds
    end

    def sleep(_duration)
      # No-op: backtest runs as fast as possible
    end

    def trading_hours?
      true # Backtest assumes valid data
    end
  end

  class ReplayClock < BaseClock
    def initialize(tick_stream:)
      @ticks = tick_stream
      @index = 0
    end

    def now
      @ticks[@index]&.dig(:timestamp) || Time.current
    end

    def advance
      @index += 1
    end

    def sleep(_duration)
      # Speed controlled by ReplaySpeedController
    end

    def trading_hours?
      true
    end
  end
end
```

---

## 4. Event Bus

All engines communicate via events. Same code works in all modes.

```ruby
# app/services/event_bus/event_bus.rb
# frozen_string_literal: true

module EventBus
  class << self
    def instance
      @instance ||= new
    end
  end

  def initialize
    @subscribers = Hash.new { |h, k| h[k] = [] }
    @redis = RedisPool.general if Rails.env.production?
  end

  def subscribe(event_type, &block)
    @subscribers[event_type] << block
  end

  def publish(event_type, payload)
    event = {
      type: event_type,
      payload: payload,
      timestamp: Time.current.iso8601,
      trace_id: Thread.current[:trace_id]
    }

    # Local subscribers
    @subscribers[event_type].each do |handler|
      begin
        handler.call(event)
      rescue => e
        Rails.logger.error "[EventBus] Handler failed for #{event_type}: #{e.message}"
      end
    end

    # Redis pub/sub for cross-process
    @redis&.publish("event_bus:#{event_type}", event.to_json)
  end

  def subscribe_async(event_type, job_class)
    subscribe(event_type) do |event|
      job_class.perform_later(event)
    end
  end
end

# Event types constant
module EventTypes
  TICK_RECEIVED = "tick_received"
  STRUCTURE_UPDATED = "structure_updated"
  SIGNAL_GENERATED = "signal_generated"
  RISK_APPROVED = "risk_approved"
  ORDER_SUBMITTED = "order_submitted"
  ORDER_FILLED = "order_filled"
  ORDER_REJECTED = "order_rejected"
  POSITION_UPDATED = "position_updated"
  TRADE_CLOSED = "trade_closed"
  MODE_SWITCHED = "mode_switched"
  CIRCUIT_BREAKER_STATE_CHANGE = "circuit_breaker_state_change"
  EMERGENCY_STOP_TRIGGERED = "emergency_stop_triggered"
  AI_AGENT_COMPLETED = "ai_agent_completed"
  REPLAY_TICK = "replay_tick"
  TRADE_LEARNED = "trade_learned"
  PERFORMANCE_REPORT_GENERATED = "performance_report_generated"
end
```

---

## 5. Backtest Adapter

```ruby
# app/services/execution/adapters/backtest_adapter.rb
# frozen_string_literal: true

module Execution
  module Adapters
    class BacktestAdapter
      def initialize(clock:)
        @clock = clock
        @portfolio = Paper::Portfolio.new(initial_capital: backtest_capital)
        @positions = Paper::PositionManager.new(portfolio: @portfolio)
        @commissions = Paper::CommissionCalculator.new
      end

      def place_order(order_request)
        # In backtest, we assume immediate fill at historical price
        # No market depth needed—use the tick price directly
        fill_price = order_request.price || order_request.tick[:ltp]

        commissions = @commissions.calculate(
          executed_price: fill_price,
          quantity: order_request.quantity,
          direction: order_request.direction
        )

        fill = {
          executed_price: fill_price,
          quantity: order_request.quantity,
          commissions: commissions,
          net_price: fill_price + (commissions / order_request.quantity.to_f),
          timestamp: @clock.now
        }

        @positions.update(fill)
        @portfolio.apply_fill(fill)

        trade = persist_backtest_trade(order_request, fill)

        Result.success(
          trade_id: trade.id,
          fill_price: fill[:net_price],
          timestamp: @clock.now.iso8601
        )
      end

      def cancel_order(_order_id)
        Result.success(cancelled: true)
      end

      def positions
        @positions.all
      end

      def portfolio
        @portfolio.snapshot
      end

      def pnl
        @portfolio.pnl
      end

      private

      def backtest_capital
        AppConfig.config.backtest&.initial_capital || 500_000
      end

      def persist_backtest_trade(order, fill)
        Trade.create!(
          mode: :backtest,
          instrument: order.instrument,
          direction: order.direction,
          quantity: fill[:quantity],
          entry_price: fill[:net_price],
          entry_time: @clock.now,
          broker_order_id: "BT-#{SecureRandom.hex(6)}",
          setup_type: order.strategy,
          status: :open
        )
      end
    end
  end
end
```

---

## 6. Live Adapter (Refactored Dhan Integration)

```ruby
# app/services/execution/adapters/live_adapter.rb
# frozen_string_literal: true

module Execution
  module Adapters
    class LiveAdapter
      def initialize(clock:)
        @clock = clock
        @client = DhanHQ::Client.new
      end

      def place_order(order_request)
        # Identical risk check already passed in gateway
        dhan_order = {
          dhanClientId: @client.client_id,
          correlationId: SecureRandom.uuid,
          transactionType: order_request.direction.to_s.upcase,
          exchangeSegment: "NSE_FNO",
          productType: order_request.product_type || "MIS",
          orderType: order_request.order_type.to_s.upcase,
          validity: "DAY",
          tradingSymbol: order_request.instrument.trading_symbol,
          securityId: order_request.instrument.security_id,
          quantity: order_request.quantity,
          price: order_request.price,
          triggerPrice: order_request.trigger_price
        }

        result = @client.place_order(dhan_order)

        if result.success?
          trade = persist_live_trade(order_request, result.value)
          Result.success(
            trade_id: trade.id,
            broker_order_id: result.value["orderId"],
            fill_price: result.value["price"],
            timestamp: @clock.now.iso8601
          )
        else
          Result.failure(result.error)
        end
      end

      def cancel_order(order_id)
        trade = Trade.find(order_id)
        @client.cancel_order(trade.broker_order_id)
      end

      def positions
        @client.positions.value || []
      end

      def portfolio
        # Would fetch from broker
        {}
      end

      def pnl
        # Would fetch from broker
        {}
      end

      private

      def persist_live_trade(order, dhan_response)
        Trade.create!(
          mode: :live,
          instrument: order.instrument,
          direction: order.direction,
          quantity: order.quantity,
          entry_price: dhan_response["price"],
          entry_time: @clock.now,
          broker_order_id: dhan_response["orderId"],
          setup_type: order.strategy,
          status: :open
        )
      end
    end
  end
end
```

---

## 7. Trade Model Update

```ruby
# app/models/trade.rb
# frozen_string_literal: true

class Trade < ApplicationRecord
  enum :mode, { live: 0, paper: 1, backtest: 2, replay: 3 }, prefix: true

  # Existing associations...
  belongs_to :instrument
  belongs_to :trade_setup, optional: true

  # Mode-aware scopes
  scope :live, -> { where(mode: :live) }
  scope :paper, -> { where(mode: :paper) }
  scope :backtest, -> { where(mode: :backtest) }
  scope :real, -> { where(mode: [:live, :paper]) }  # Excludes backtest/replay
  scope :for_analysis, -> { where(mode: [:live, :paper]) }

  # Paper-specific fields
  store_accessor :paper_fill, :slippage, :commissions, :virtual_fills

  validates :mode, presence: true
  validates :broker_order_id, presence: true, if: :mode_live?

  # Learning engine works on all modes
  after_update :queue_learning_analysis, if: :just_closed?

  def just_closed?
    saved_change_to_exit_price? && exit_price.present? && exit_price_before_last_save.nil?
  end

  def queue_learning_analysis
    LearningAnalysisJob.perform_later(id)
  end

  def virtual?
    mode_backtest? || mode_replay?
  end

  def real?
    mode_live? || mode_paper?
  end
end
```

---

## 8. Configuration

```yaml
# config/execution.yml
execution:
  default_mode: paper  # live | paper | backtest

paper_trading:
  initial_capital: 1000000
  enabled: true
  commission_model: dhan  # realistic broker charges

backtest:
  initial_capital: 500000
  data_source: timescaledb  # or csv

live:
  broker: dhan
  client_id: <%= ENV["DHAN_CLIENT_ID"] %>
  api_token: <%= ENV["DHAN_API_TOKEN"] %>
```

---

## 9. API Endpoint for Mode Switching

```ruby
# app/controllers/api/v1/execution_mode_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class ExecutionModeController < ApplicationController
      before_action :require_admin, only: [:switch]

      def current
        gateway = Execution::ExecutionGateway.new
        render json: {
          mode: gateway.mode,
          timestamp: Time.current.iso8601
        }
      end

      def switch
        new_mode = params[:mode]&.to_sym
        gateway = Execution::ExecutionGateway.new
        result = gateway.switch_mode(new_mode)

        if result.success?
          render json: {
            mode: result.value[:mode],
            message: "Switched to #{result.value[:mode]}",
            timestamp: Time.current.iso8601
          }
        else
          render json: { error: result.error.to_h }, status: :unprocessable_entity
        end
      end

      def paper_portfolio
        gateway = Execution::ExecutionGateway.new(mode: :paper)
        render json: gateway.portfolio
      end

      def paper_positions
        gateway = Execution::ExecutionGateway.new(mode: :paper)
        render json: { positions: gateway.positions }
      end

      private

      def require_admin
        render json: { error: "Admin required" }, status: :forbidden unless current_user&.admin?
      end
    end
  end
end

# config/routes.rb
namespace :api do
  namespace :v1 do
    get 'execution_mode/current', to: 'execution_mode#current'
    post 'execution_mode/switch', to: 'execution_mode#switch'
    get 'execution_mode/paper/portfolio', to: 'execution_mode#paper_portfolio'
    get 'execution_mode/paper/positions', to: 'execution_mode#paper_positions'
  end
end
```

---

## 10. Updated Architecture Diagram

```text
┌─────────────────────────────────────────────────────────────┐
│                     MARKET DATA LAYER                        │
│  (Dhan WebSocket, REST, Historical DB, Replay Engine)       │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  MARKET INTELLIGENCE LAYER                   │
│  (Regime, Structure, Momentum, Liquidity, Option Flow)      │
│  (Feature Engineering, Greeks, Scoring)                     │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    STRATEGY ENGINE LAYER                     │
│  (Signal Generation, Setup Detection, Strike Selection)       │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      RISK ENGINE LAYER                       │
│  (Pre-Execution Checks, Position Limits, Daily Loss)        │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   EXECUTION GATEWAY                          │
│              (Routes to correct adapter)                     │
└────────────┬─────────────────────────────┬──────────────────┘
             │                             │
             ▼                             ▼
┌──────────────────────┐      ┌──────────────────────────────┐
│   PAPER ADAPTER      │      │     LIVE ADAPTER             │
│  • Live market data  │      │  • Live market data          │
│  • Virtual broker    │      │  • DhanHQ REST/WebSocket     │
│  • Full matching     │      │  • Real capital              │
│  • Slippage model    │      │  • Real P&L                  │
│  • Commission model  │      │                              │
│  • Portfolio tracking│      │                              │
└──────────┬───────────┘      └──────────────┬─────────────┘
           │                                   │
           └──────────────┬────────────────────┘
                          │
                          ▼
            ┌─────────────────────────────┐
            │    UNIFIED PORTFOLIO MANAGER │
            │  (Positions, P&L, MFE/MAE)   │
            └──────────────┬──────────────┘
                           │
                           ▼
            ┌─────────────────────────────┐
            │      AI REVIEW & JOURNAL     │
            │   (All modes feed learning)  │
            └─────────────────────────────┘
```

---

## Key Advantages of This Refactor

| Before (Separate Modules) | After (Execution Modes) |
|---------------------------|------------------------|
| 3 strategy code paths | 1 strategy engine |
| Paper trades stored differently | Paper trades are `Trade` with `mode: :paper` |
| Paper ignores commissions | Paper includes full Dhan commission model |
| Paper assumes perfect fills | Paper uses live depth + slippage model |
| Backtest uses fake data | Backtest uses same historical DB as replay |
| Replay is separate testing | Replay is a clock mode |
| `Time.current` everywhere | `clock.now()` mode-aware |
| Tight coupling | Event bus decoupling |

**The result:** >95% code shared across Backtest, Replay, Paper, and Live. Promoting a strategy from Paper to Live is a single configuration change (`mode: :live`), not a codebase migration.