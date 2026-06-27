# Milestone 0.1: Repository & DevOps Foundation

**Phase:** 0 — Foundation & Tooling  
**Goal:** Production-ready Rails environment with strict quality gates.  
**Estimated Tasks:** 15

---

## Tasks

### 1. Initialize Rails 8.1.3 App
- [ ] Create Rails 8.1.3 application in `api` mode with PostgreSQL
- [ ] Configure database.yml for development, test, paper, production
- [ ] Verify Rails version and dependencies

### 2. Configure RuboCop
- [ ] Add `rubocop`, `rubocop-rails`, `rubocop-performance`, `rubocop-rspec` to Gemfile
- [ ] Create `.rubocop.yml` with project-specific rules
- [ ] Configure Rails/Performance/RSpec cops
- [ ] Run `rubocop --auto-gen-config` and review

### 3. Configure Brakeman
- [ ] Add `brakeman` to Gemfile (development group)
- [ ] Create `.brakeman.ignore` for false positives
- [ ] Configure to run in CI pipeline

### 4. Configure Bundler Audit
- [ ] Add `bundler-audit` to Gemfile
- [ ] Run `bundler-audit check --update`
- [ ] Configure to run in CI pipeline

### 5. Set Up GitHub Actions CI Pipeline
- [ ] Create `.github/workflows/ci.yml`
- [ ] Jobs: lint → security → test → build
- [ ] Configure Ruby version matrix
- [ ] Add PostgreSQL and Redis services
- [ ] Cache bundler and yarn dependencies

### 6. Configure Docker + Docker Compose
- [ ] Create multi-stage `Dockerfile` (builder, runtime)
- [ ] Create `docker-compose.yml` with app, db, redis, sidekiq
- [ ] Configure `.dockerignore`
- [ ] Test build locally

### 7. Add HEALTHCHECK Endpoint
- [ ] Create `/health` endpoint in Rails
- [ ] Add `HEALTHCHECK` instruction to Dockerfile
- [ ] Verify container health checks work

### 8. Configure Pre-commit Hooks
- [ ] Add `pre-commit` gem or use `husky`/`lefthook`
- [ ] Hooks: rubocop, brakeman, bundler-audit, rspec (changed files)
- [ ] Document in README

### 9. Set Up RSpec
- [ ] Add `rspec-rails`, `factory_bot_rails`, `faker`, `shoulda-matchers`
- [ ] Configure `spec/rails_helper.rb` and `spec/spec_helper.rb`
- [ ] Create `spec/support/` directory structure
- [ ] Add database_cleaner configuration

### 10. Configure SimpleCov
- [ ] Add `simplecov` to Gemfile (test group)
- [ ] Configure 80% coverage threshold
- [ ] Add coverage to CI artifacts
- [ ] Exclude generated files, vendors

### 11. Add Bullet Gem
- [ ] Add `bullet` to Gemfile (development group)
- [ ] Configure in `config/environments/development.rb`
- [ ] Enable N+1 detection, unused eager loading alerts

### 12. Set Up Rack::Attack
- [ ] Add `rack-attack` to Gemfile
- [ ] Create `config/initializers/rack_attack.rb`
- [ ] Configure throttles for API endpoints
- [ ] Add blocklist/allowlist for IPs

### 13. Configure Lograge
- [ ] Add `lograge` to Gemfile
- [ ] Configure in `config/environments/production.rb`
- [ ] Set JSON log format
- [ ] Configure custom options (request_id, user_id)

### 14. Add Error Monitoring
- [ ] Choose: `appsignal` or `sentry-rails`
- [ ] Add gem and configure DSN via credentials
- [ ] Configure error filtering and sampling
- [ ] Test error reporting in staging

### 15. Create Makefile
- [ ] Create `Makefile` with targets:
  - `setup` - install deps, setup db, seed
  - `test` - run full test suite
  - `lint` - run rubocop, brakeman, bundler-audit
  - `console` - rails console
  - `server` - start rails server
  - `worker` - start solid queue worker
  - `docker-build` - build docker image
  - `docker-up` - start docker compose

---

## Acceptance Criteria
- [ ] `make setup` works on fresh machine
- [ ] `make test` passes with >80% coverage
- [ ] `make lint` passes with zero offenses
- [ ] Docker image builds and runs health check
- [ ] CI pipeline passes on GitHub Actions
- [ ] Pre-commit hooks catch common issues

---

## Notes
- All configuration should be environment-aware
- Document any manual steps in `docs/onboarding.md`
- Consider using `direnv` for local environment management