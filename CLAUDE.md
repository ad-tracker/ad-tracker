# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a monorepo containing three interconnected projects for YouTube webhook ingestion:

- **youtube-webhook-ingestion-go/** - Go backend service (main webhook receiver, API server, enricher worker, renewal service)
- **youtube-webhook-admin-ui/** - React admin interface for managing subscriptions and viewing webhook data
- **youtube-webhook-ingestion-deploy/** - Docker Compose deployment configuration with Swag reverse proxy

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

# Run with verbose output and race detection
go test -v -race -coverprofile=coverage.out ./...

# Run tests for specific package
go test -v ./internal/db/repository

# Run a specific test
go test -v ./internal/db/repository -run TestWebhookEventRepository_CreateWebhookEvent
```

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
npm run test              # Watch mode
npm run test:ui           # With UI
npm run test:coverage     # With coverage report
npm run test:ci           # CI mode (no watch, with coverage)
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
