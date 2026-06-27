# Milestone 0.2: Configuration & Environment Management

**Phase:** 0 — Foundation & Tooling  
**Goal:** Environment-aware, type-safe configuration using dry-rb patterns.  
**Estimated Tasks:** 12

---

## Tasks

### 1. Install dry-configurable & dry-struct
- [ ] Add `dry-configurable`, `dry-struct`, `dry-types` to Gemfile
- [ ] Run `bundle install`
- [ ] Verify versions compatible with Rails 8.1

### 2. Create AppConfig Class
- [ ] Create `app/config/app_config.rb` with `Dry::Configurable`
- [ ] Define all trading parameters with types and defaults:
  - Trading hours, symbol list, strike ranges
  - Risk limits (daily/weekly/monthly loss, max positions)
  - Broker credentials (referenced from credentials)
  - Engine thresholds (score thresholds, regime filters)
  - Data retention policies
  - AI model parameters (model, temperature, max tokens)
- [ ] Add validation rules for each setting

### 3. Set Up Environment Files
- [ ] Create `.env.example` with all required variables documented
- [ ] Create `.env.development` with local defaults
- [ ] Create `.env.paper` for paper trading environment
- [ ] Create `.env.production` template (no secrets)
- [ ] Add `.env*` to `.gitignore` (except `.env.example`)

### 4. Configure Rails Credentials
- [ ] Run `rails credentials:edit --environment=development`
- [ ] Run `rails credentials:edit --environment=production`
- [ ] Store: DhanHQ API keys, Redis URLs, AI API keys, webhook secrets
- [ ] Document credential structure in `docs/credentials.md`

### 5. Create DhanHQ Initializer
- [ ] Create `config/initializers/dhanhq.rb`
- [ ] Configure `DhanHQ.configure` with credentials from `Rails.application.credentials.dhanhq`
- [ ] Set sandbox vs production mode based on env
- [ ] Configure timeouts, retries, logging

### 6. Create Redis Initializer
- [ ] Create `config/initializers/redis.rb`
- [ ] Configure `ConnectionPool` for Redis
- [ ] Separate pools for: cache, cable, queue, pubsub
- [ ] Set pool sizes via AppConfig

### 7. Configure Solid Queue
- [ ] Run `rails solid_queue:install` (creates migration)
- [ ] Configure `config/solid_queue.yml` with:
  - Separate database (or same with different schema)
  - Worker processes and threads
  - Queue priorities (critical, default, low)
  - Recurring jobs schedule
- [ ] Add monitoring dashboard route

### 8. Configure Solid Cache
- [ ] Run `rails solid_cache:install`
- [ ] Configure `config/solid_cache.yml` with TTL policies:
  - Market ticks: 1 hour
  - Candles: 24 hours
  - Option chains: 5 minutes
  - Features: 30 seconds
  - AI responses: 10 minutes
- [ ] Set compression for large entries

### 9. Configure Solid Cable
- [ ] Run `rails solid_cable:install`
- [ ] Configure Redis adapter for Action Cable
- [ ] Set up channels for real-time dashboard updates

### 10. Add Environment Validation
- [ ] Create `config/initializers/validate_environment.rb`
- [ ] Validate all required credentials present on boot
- [ ] Fail fast with clear error messages
- [ ] Skip in test environment

### 11. Create bin/setup Script
- [ ] Create executable `bin/setup`
- [ ] Steps: bundle install, yarn install (if needed), db setup, db seed, credentials check
- [ ] Make idempotent and safe to re-run
- [ ] Output clear next steps

### 12. Document Configuration
- [ ] Create `docs/configuration.md` with:
  - All environment variables table
  - All credential keys table
  - AppConfig settings reference
  - Environment-specific overrides
  - Troubleshooting common config issues

---

## Acceptance Criteria
- [ ] `bin/setup` completes successfully on fresh clone
- [ ] Rails boots in all 4 environments without config errors
- [ ] `AppConfig.trading.daily_loss_limit` returns typed value
- [ ] Solid Queue workers start and process jobs
- [ ] Solid Cache reads/writes with correct TTL
- [ ] Action Cable connects via Solid Cable
- [ ] Missing required credential raises clear error on boot

---

## Notes
- Use `Dry::Types` for coercion (e.g., `"50000".to_i`)
- Consider `dry-validation` for complex cross-field validation
- Keep credentials out of code; use `Rails.application.credentials.dig(:dhanhq, :api_key)`
- Test paper trading config matches production structure