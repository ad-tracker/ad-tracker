# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a monorepo containing three interconnected projects for YouTube webhook ingestion:

- **youtube-webhook-ingestion-go/** - Go backend service (main webhook receiver, API server, enricher worker, renewal service)
- **youtube-webhook-admin-ui/** - React admin interface for managing subscriptions and viewing webhook data
- **youtube-webhook-ingestion-deploy/** - Docker Compose deployment configuration with Swag reverse proxy

## Agent Workflow Instructions

**CRITICAL: Follow this workflow for ALL code changes. This is mandatory.**

When making any code changes, follow this exact workflow:

### 1. Before Making Changes

```bash
# Check for uncommitted changes (should show clean working tree)
git status

# Ensure you're on main branch
git checkout main

# Update the default branch
git pull origin main

# Create a new feature branch
git checkout -b feature/descriptive-name
```

**Important:**
- Always commit or stash changes before switching branches
- Always create a feature branch from an up-to-date default branch
- Use descriptive branch names: `feature/add-channel-api`, `fix/webhook-parsing-bug`, `refactor/improve-error-handling`
- Never commit directly to `main` or the default branch

### 2. During Development

Make your code changes following project conventions and architecture patterns.

### 3. Before Committing

**For Go projects (youtube-webhook-ingestion-go/):**

```bash
cd youtube-webhook-ingestion-go

# Format all Go code
go fmt ./...

# Ensure dependencies are clean (if this changes files, commit them)
go mod tidy

# Check for common issues
go vet ./...

# Run static analysis (install if needed: go install honnef.co/go/tools/cmd/staticcheck@latest)
staticcheck ./...

# Run all tests with race detection (requires Docker for testcontainers)
go test -v -race ./...

# Verify all builds succeed
go build ./cmd/server
go build ./cmd/enricher
go build ./cmd/renewer
go build ./cmd/migrate
```

**Note:** If `go mod tidy` changes `go.mod` or `go.sum`, stage and commit those changes before proceeding. If tests fail due to Docker not running, start Docker and re-run the tests.

**For React projects (youtube-webhook-admin-ui/):**

```bash
cd youtube-webhook-admin-ui

# Run linting (auto-fixes some issues)
npm run lint

# Run all tests in CI mode (non-interactive, with coverage)
npm run test:ci

# Verify build succeeds
npm run build
```

**All pre-commit checks must pass before proceeding.** Fix any linting errors, test failures, or build errors before committing.

### 4. Committing and Opening PR

```bash
# Verify you're on the feature branch
git branch --show-current

# Stage all changes
git add .

# Commit with descriptive message (use clear, imperative mood)
git commit -m "Brief description of changes

Longer explanation if needed:
- Key change 1
- Key change 2
- Fixes issue #123 (if applicable)"

# Push the feature branch
git push -u origin feature/descriptive-name

# Verify push succeeded
git status

# Open a pull request using GitHub CLI (targets main by default)
gh pr create --base main --title "Brief description" --body "Detailed description of changes"
```

**Commit message guidelines:**
- Use imperative mood: "Add feature" not "Added feature"
- First line should be concise (50 chars or less)
- Add detailed explanation after a blank line if needed

### 5. After PR Creation

```bash
# Verify PR was created successfully and view details
gh pr view

# Check status of CI checks
gh pr checks

# If checks are running, watch them in real-time
gh pr checks --watch

# If a check fails, view the detailed logs for that check
gh run view --log-failed
```

**Monitoring PR checks:**
- `gh pr view` - Shows PR details, description, and status
- `gh pr checks` - Lists all checks with their current status
- `gh pr checks --watch` - Live updates as checks run (use Ctrl+C to exit)
- `gh run view --log-failed` - Shows detailed failure logs from GitHub Actions

**Required PR checks:**
- Code formatting (`go fmt`, linting)
- Dependency validation (`go mod tidy`)
- All tests passing with race detection
- Build verification
- Code coverage (must not decrease significantly)
- Static analysis (`go vet`, `staticcheck` for Go)

**If any check fails:**
1. Review the failure logs: `gh run view --log-failed`
2. Fix the issue locally (re-run the relevant pre-commit checks)
3. If `go mod tidy` made changes, stage them: `git add go.mod go.sum`
4. Commit and push the fix: `git add . && git commit -m "Fix: description" && git push`
5. Monitor checks again: `gh pr checks --watch`
6. Repeat until all checks pass

**Common check failures and fixes:**
- **Formatting check fails:** Run `go fmt ./...` or `npm run lint`, then commit changes
- **Tests fail:** Review test output with `gh run view --log-failed`, fix the code, verify locally with `go test -v ./...` or `npm run test:ci`
- **Build fails:** Verify locally with `go build ./cmd/...` or `npm run build`, fix errors, then commit
- **Dependencies check fails:** Run `go mod tidy`, commit the changes to `go.mod` and `go.sum`

### 6. Verification Checklist

Before considering work complete, verify these items:

**Pre-commit verification:**
1. Verify feature branch was created from up-to-date main branch: `git log origin/main..HEAD`
2. Verify on correct feature branch: `git branch --show-current`
3. Verify code is properly formatted: `go fmt ./...` or `npm run lint` (should show no changes)
4. Verify all tests pass locally: `go test -v -race ./...` or `npm run test:ci` (should show all green)
5. Verify dependencies are clean: `go mod tidy` (should not change files)
6. Verify static analysis passes: `go vet ./...` and `staticcheck ./...` (Go only)
7. Verify all builds succeed: `go build ./cmd/...` or `npm run build` (should complete without errors)

**Post-commit verification:**
8. Verify changes committed with clear message: `git log -1`
9. Verify PR was created successfully: `gh pr view`
10. Verify all PR CI checks are passing: `gh pr checks` (should show all green checkmarks)

**Do not consider the task complete until all verification items pass.**

### 7. Common Issues and Solutions

**"Tests are failing locally"**
- **Go tests:** Ensure Docker is running (required for testcontainers). Start Docker Desktop or `systemctl start docker`
- Check that all required environment variables are set (see "Running Services" section)
- Run specific failing test with verbose output: `go test -v ./path/to/package -run TestName`
- Review detailed test output for error messages and stack traces
- For React tests, try clearing cache: `npm run test:ci -- --clearCache`

**"PR checks failing but local tests pass"**
- Ensure branch is up-to-date with main: `git pull origin main` then resolve any conflicts and `git push`
- Check for formatting issues: `go fmt ./...` or `npm run lint` (commit any changes)
- Verify `go.mod` and `go.sum` are committed after `go mod tidy`
- Check the actual CI logs with: `gh run view --log-failed`
- Ensure you ran the exact same commands locally (e.g., `go test -v -race ./...` not just `go test ./...`)

**"Merge conflicts with main"**
```bash
# Update main branch first
git checkout main
git pull origin main

# Switch back to feature branch and merge
git checkout feature/your-branch
git merge main

# Resolve conflicts manually, then:
git add .
git commit -m "Merge main into feature branch"
git push
```

**"go mod tidy keeps changing files"**
- This is normal if dependencies were added/removed
- Review the changes with `git diff go.mod go.sum`
- Stage and commit them: `git add go.mod go.sum && git commit -m "Update dependencies"`

**"staticcheck not found"**
- Install it: `go install honnef.co/go/tools/cmd/staticcheck@latest`
- Ensure `$GOPATH/bin` or `$HOME/go/bin` is in your PATH

**"Docker permission denied (Go tests)"**
- Linux: Add user to docker group: `sudo usermod -aG docker $USER` (then logout/login)
- Ensure Docker daemon is running: `docker ps`

## Go Service (youtube-webhook-ingestion-go/)

### Build Commands

```bash
# Build all binaries
go build -o bin/server ./cmd/server
go build -o bin/enricher ./cmd/enricher
go build -o bin/renewer ./cmd/renewer
go build -o bin/migrate ./cmd/migrate

# Or use the shorthand (binaries go to project root)
go build ./cmd/server
go build ./cmd/enricher
go build ./cmd/renewer
go build ./cmd/migrate
```

### Testing

```bash
# Run all tests (requires Docker for testcontainers)
go test ./...

# Run with verbose output and race detection (recommended for pre-commit)
go test -v -race -coverprofile=coverage.out ./...

# Run tests for specific package
go test -v ./internal/db/repository

# Run a specific test
go test -v ./internal/db/repository -run TestWebhookEventRepository_CreateWebhookEvent

# View coverage report
go tool cover -html=coverage.out
```

**Note:** Integration tests use testcontainers which require Docker to be running. If Docker is not available, tests will fail with connection errors.

### Code Quality

```bash
# Format code
go fmt ./...

# Check for issues
go vet ./...

# Run static analysis
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...

# Ensure dependencies are tidy
go mod tidy
```

### Database Migrations

```bash
# Run migrations up
go run ./cmd/migrate -direction up

# Run migrations down
go run ./cmd/migrate -direction down

# With custom database URL
go run ./cmd/migrate -db "postgres://user:password@localhost:5432/youtube_webhooks?sslmode=disable" -direction up
```

**⚠️ IMPORTANT:** Always test migrations on a development database first. Create a backup before running migrations on production. Verify that rollback (down migration) works correctly before deploying to production.

### Running Services

```bash
# Set required environment variables
export DATABASE_URL="postgres://user:password@localhost:5432/youtube_webhooks?sslmode=disable"
export API_KEYS="your-api-key-here"
export DOMAIN="yourdomain.com"

# Run the webhook server
go run ./cmd/server

# Run the enricher worker (requires YOUTUBE_API_KEY and REDIS_URL)
export YOUTUBE_API_KEY="your-youtube-api-key"

# REDIS_URL supports multiple formats:
# - Simple host:port: "localhost:6379"
# - Redis URL: "redis://localhost:6379/0"
# - With password: "redis://:password@localhost:6379/0"
# - With TLS: "rediss://:password@secure-redis.example.com:6380/0"
export REDIS_URL="redis://localhost:6379"
go run ./cmd/enricher

# Run the renewal service
go run ./cmd/renewer
```

### Architecture Overview

The Go service follows a clean architecture with clear separation:

- **cmd/** - Entry points for each binary (server, enricher, renewer, migrate)
- **internal/db/** - Database layer with repository pattern, models, and connection pooling
  - **models/** - Database models (structs)
  - **repository/** - Repository interfaces and implementations (channels, videos, webhook_events, video_updates, subscriptions, enrichments)
  - Thread-safe connection pooling with pgx/v5
- **internal/handler/** - HTTP handlers for REST API and webhook endpoints
- **internal/service/** - Business logic layer (subscription service, enrichment service, parser service)
- **internal/parser/** - YouTube Atom feed XML parsing
- **internal/queue/** - Asynq job queue integration for async enrichment
- **internal/middleware/** - API key authentication middleware
- **migrations/** - SQL migration files (versioned with golang-migrate)

**Important Design Patterns:**
- Immutable audit trails: `webhook_events` and `video_updates` tables are append-only
- Content-based deduplication: SHA-256 hashing prevents duplicate webhook processing
- Event sourcing: Complete history preserved in immutable tables
- Repository pattern: Clean separation between database access and business logic
- All database operations accept `context.Context` for proper cancellation/timeout handling

## Admin UI (youtube-webhook-admin-ui/)

### Development Commands

```bash
# Install dependencies
npm install

# Start development server (http://localhost:5173)
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Linting
npm run lint

# Testing
npm run test              # Watch mode (interactive)
npm run test:ui           # With UI (interactive browser)
npm run test:coverage     # With coverage report (interactive)
npm run test:ci           # CI mode (non-interactive, with coverage - use for pre-commit checks)
```

### Architecture Overview

Modern React application with:

- **React 19** with TypeScript for type safety
- **Vite** for fast development and optimized production builds
- **TanStack Query (React Query)** for server state management and caching
- **React Router v7** for client-side routing
- **TailwindCSS** for styling
- **Vitest + React Testing Library** for comprehensive unit testing

**Key Components:**
- `src/lib/api-client.ts` - Type-safe API client with authentication
- `src/contexts/APIContext.tsx` - Global API configuration (base URL, API key)
- `src/components/ui/` - Reusable base UI components
- `src/pages/` - Page-level components for each route
- API credentials stored in session storage (cleared on tab close)

## Deployment (youtube-webhook-ingestion-deploy/)

### Docker Compose Commands

```bash
# Deploy with local databases (PostgreSQL + Valkey/Redis)
docker compose --profile local-postgres --profile local-valkey up -d

# Deploy with remote databases (set DATABASE_URL and REDIS_URL in .env)
docker compose up -d

# View logs
docker compose logs -f
docker compose logs -f webhook-service

# Restart services
docker compose restart webhook-service

# Update to latest version
docker compose pull
docker compose up -d

# Stop all services
docker compose down

# Stop and remove all data (WARNING: destroys database!)
docker compose down -v
```

### Required Environment Variables

Always configure in `.env` file:
- `DOMAIN` - Your domain name for SSL certificates
- `POSTGRES_PASSWORD` - Strong database password
- `API_KEYS` - Comma-separated API keys for authentication
- `EMAIL` - Email for Let's Encrypt notifications

Optional:
- `YOUTUBE_API_KEY` - Required for enrichment and `/api/v1/channels/from-url` endpoint
- `DATABASE_URL` - Override for remote PostgreSQL
- `REDIS_URL` - Override for remote Redis/Valkey (supports password authentication)

#### Connecting to Remote Valkey/Redis with Password

The `REDIS_URL` environment variable supports multiple connection formats, including password authentication:

**Supported formats:**
```bash
# Simple host:port (no authentication, legacy format)
REDIS_URL=localhost:6379

# Redis URL without password
REDIS_URL=redis://my-redis.example.com:6379/0

# Redis URL with password (recommended for remote servers)
REDIS_URL=redis://:mySecretPassword@redis.example.com:6379/0

# Redis with TLS and password (for secure connections)
REDIS_URL=rediss://:mySecretPassword@secure-redis.example.com:6380/0

# Using a specific database number (0-15)
REDIS_URL=redis://:password@redis.example.com:6379/5
```

**Important notes:**
- Passwords with special characters must be URL-encoded (e.g., `@` becomes `%40`, `#` becomes `%23`)
- Use `rediss://` (with double 's') for TLS-encrypted connections
- The database number is optional and defaults to `0` if not specified
- Both the webhook-service and enricher use the same `REDIS_URL` configuration

### Stack Components

- **Swag** - Reverse proxy with automatic Let's Encrypt SSL/TLS certificates
- **PostgreSQL** - Primary database for all webhook/channel/video data
- **Valkey** (Redis) - Job queue backend for async enrichment tasks
- **webhook-service** - Main API server (port 8080)
- **enricher** - Background worker for YouTube API enrichment
- **renewer** - Automatic subscription renewal service (runs every 6 hours)
- **admin-ui** - Web interface for management

## Development Workflow

### Working with Git

Always work on feature branches and open PRs:

```bash
# Create feature branch
git checkout -b feature/your-feature-name

# Make changes, commit
git add .
git commit -m "Description of changes"

# Push and create PR
git push -u origin feature/your-feature-name
```

PRs must have all checks passing before merge. The PR validation workflow runs:
- Code formatting checks (`go fmt`)
- Dependency validation (`go mod tidy`)
- All tests with race detection
- Code coverage upload
- Build verification
- Static analysis (`go vet`, `staticcheck`)

### Common Development Tasks

**Adding a new database migration:**
1. Create up/down SQL files in `youtube-webhook-ingestion-go/migrations/`
2. Follow naming: `NNNNNN_description.up.sql` and `NNNNNN_description.down.sql`
3. Test locally with `go run ./cmd/migrate -direction up`
4. Ensure rollback works: `go run ./cmd/migrate -direction down`

**Adding a new API endpoint:**
1. Add handler in `youtube-webhook-ingestion-go/internal/handler/`
2. Add route in `cmd/server/main.go`
3. Write tests in corresponding `*_test.go` file
4. Update API documentation in `docs/API.md`
5. Update admin UI if needed (`youtube-webhook-admin-ui/src/lib/api-client.ts`)

**Adding a new React component:**
1. Create component in `youtube-webhook-admin-ui/src/components/`
2. Write tests in `__tests__/` subdirectory
3. Run tests: `npm run test`
4. Ensure proper TypeScript types

## Key Dependencies

**Go:**
- `github.com/jackc/pgx/v5` - PostgreSQL driver (connection pooling)
- `github.com/golang-migrate/migrate/v4` - Database migrations
- `github.com/hibiken/asynq` - Redis-backed job queue
- `google.golang.org/api/youtube/v3` - YouTube Data API client
- `github.com/testcontainers/testcontainers-go` - Integration testing with real PostgreSQL

**React/TypeScript:**
- `@tanstack/react-query` - Server state management
- `react-router-dom` - Routing
- `tailwindcss` - Styling
- `vitest` - Testing framework
- `@testing-library/react` - Component testing utilities

## Documentation

Refer to these docs in the Go service:
- `docs/ARCHITECTURE.md` - Complete architecture, design patterns, data flows
- `docs/API.md` - REST API reference with examples
- `docs/AUTHENTICATION.md` - API key authentication guide
- `docs/database-schema.md` - Detailed schema documentation
