# Agor Environment Demo

[![CI](https://github.com/m-letourneur/demoagor/actions/workflows/ci.yml/badge.svg)](https://github.com/m-letourneur/demoagor/actions/workflows/ci.yml)
[![CD](https://github.com/m-letourneur/demoagor/actions/workflows/cd.yml/badge.svg)](https://github.com/m-letourneur/demoagor/actions/workflows/cd.yml)

This project demonstrates **Agor's Environment feature** - the ability to start, stop, and manage Docker Compose environments per worktree.

## What is Agor Environment?

Agor can automatically manage development environments for each worktree, giving you:

- **Isolated environments** - Each worktree gets its own containers and ports
- **Parallel execution** - Run multiple worktrees simultaneously without conflicts
- **Dynamic port allocation** - Ports calculated using `worktree.unique_id`
- **Automatic lifecycle** - Start/stop environments via Agor UI or MCP tools
- **Health monitoring** - Real-time health checks
- **Log streaming** - View container logs directly
- **Easy access** - One-click to open the running app

## Architecture

This demo includes:

- **PostgreSQL** (port 5432) - Database service
- **Express Backend** (port 3000) - Node.js API with health endpoint
- **Docker Compose** - Container orchestration
- **.agor.yml** - Agor environment configuration

## How It Works

### 1. Environment Configuration (`.agor.yml`)

```yaml
environment:
  # Dynamic ports using worktree unique_id (e.g., worktree 1: 3001, 5433)
  start: BACKEND_PORT={{add 3000 worktree.unique_id}} POSTGRES_PORT={{add 5432 worktree.unique_id}} docker compose -p demoagor-{{worktree.name}} up -d
  stop: docker compose -p demoagor-{{worktree.name}} down
  health: http://localhost:{{add 3000 worktree.unique_id}}/health
  app: http://localhost:{{add 3000 worktree.unique_id}}
  logs: docker compose -p demoagor-{{worktree.name}} logs --tail=100
```

**Key features:**
- **Dynamic ports** - `{{add 3000 worktree.unique_id}}` calculates unique ports per worktree
- **Project isolation** - `-p demoagor-{{worktree.name}}` creates separate Docker Compose projects
- **Template syntax** - Agor expands variables at runtime (e.g., `worktree.unique_id`, `worktree.name`)

**Port allocation example:**
- Worktree `unique_id=1`: Backend on `3001`, PostgreSQL on `5433`
- Worktree `unique_id=2`: Backend on `3002`, PostgreSQL on `5434`
- Worktree `unique_id=5`: Backend on `3005`, PostgreSQL on `5437`

### 2. Using Agor MCP Tools

From within a Claude Code session, you can:

```javascript
// Start the environment
await use_mcp_tool('mcp__agor__agor_environment_start', {
  worktree_name: 'initial-setup'
});

// Check health
await use_mcp_tool('mcp__agor__agor_environment_health', {
  worktree_name: 'initial-setup'
});

// View logs
await use_mcp_tool('mcp__agor__agor_environment_logs', {
  worktree_name: 'initial-setup'
});

// Open the app in browser
await use_mcp_tool('mcp__agor__agor_environment_open_app', {
  worktree_name: 'initial-setup'
});

// Stop the environment
await use_mcp_tool('mcp__agor__agor_environment_stop', {
  worktree_name: 'initial-setup'
});
```

### 3. Manual Testing

**Single worktree (default ports):**

```bash
# Start the environment
docker compose up -d

# Check health
curl http://localhost:3000/health

# Test the API
curl http://localhost:3000/api/hello

# View logs
docker compose logs -f

# Stop the environment
docker compose down
```

**Multiple worktrees in parallel:**

```bash
# Start worktree 1 (ports 3001, 5433)
BACKEND_PORT=3001 POSTGRES_PORT=5433 docker compose -p demoagor-feature-1 up -d

# Start worktree 2 (ports 3002, 5434)
BACKEND_PORT=3002 POSTGRES_PORT=5434 docker compose -p demoagor-feature-2 up -d

# Test both environments
curl http://localhost:3001/health  # Worktree 1
curl http://localhost:3002/health  # Worktree 2

# Stop specific worktree
docker compose -p demoagor-feature-1 down

# View all running projects
docker compose ls
```

## API Endpoints

- `GET /` - Welcome message and endpoint list
- `GET /health` - Health check (required by Agor)
- `GET /api/hello` - Example API endpoint that queries the database

## Next Steps

To extend this demo:

1. **Add more services** - Redis, frontend, etc.
2. **Use dynamic ports** - Leverage worktree IDs for port isolation
3. **Add initialization** - Database migrations, seed data
4. **Multi-worktree** - Test with multiple worktrees running simultaneously

## Verifying Multi-Worktree Support

To test that multiple worktrees can run in parallel:

```bash
# Clean up any existing containers
docker compose down

# Simulate worktree 1 (unique_id=1)
BACKEND_PORT=3001 POSTGRES_PORT=5433 docker compose -p demoagor-wt1 up -d

# Simulate worktree 2 (unique_id=2)
BACKEND_PORT=3002 POSTGRES_PORT=5434 docker compose -p demoagor-wt2 up -d

# Wait for services to be healthy
sleep 10

# Verify both are running
curl http://localhost:3001/health  # Should return healthy
curl http://localhost:3002/health  # Should return healthy

# List all containers
docker ps | grep demoagor

# Clean up
docker compose -p demoagor-wt1 down
docker compose -p demoagor-wt2 down
```

## Benefits of Agor Environments

- 🚀 **Fast context switching** - Start/stop environments without manual commands
- 🔒 **Isolation** - Each worktree is independent with unique ports
- 🔄 **Parallel development** - Work on multiple branches simultaneously
- 👁️ **Visibility** - Health and logs at a glance
- 🤖 **AI-native** - Agents can manage environments programmatically
- 🔧 **No conflicts** - Dynamic ports prevent collision between worktrees

## CI/CD Pipeline

This project includes automated CI/CD workflows using GitHub Actions.

### Continuous Integration (CI)

**Workflow:** `.github/workflows/ci.yml`

**Triggers:**
- Push to any branch
- Pull request to any branch

**Jobs:**
1. **Lint** - Code quality checks (placeholder for future linting)
2. **Build & Test** - Multi-version Node.js testing
   - Tests Node 18 and 20
   - Builds Docker images with layer caching
   - Starts Docker Compose environment
   - Runs health checks and API tests
   - Cleans up after tests
3. **Docker Security** - Vulnerability scanning with Trivy
   - Scans for CRITICAL and HIGH severity vulnerabilities
   - Uploads results to GitHub Security tab
4. **Summary** - Aggregates results and reports status

**Key Features:**
- 🚀 **Matrix builds** - Tests against multiple Node versions
- 📦 **Docker layer caching** - Faster builds using GitHub cache
- 🔒 **Security scanning** - Automated vulnerability detection
- ✅ **Health checks** - Validates services are running correctly
- 📊 **PR comments** - Automatic summary in GitHub UI

### Continuous Deployment (CD)

**Workflow:** `.github/workflows/cd.yml`

**Triggers:**
- Push to `main` branch (after PR merge)

**Jobs:**
1. **Build & Push** - Publishes Docker images to GitHub Container Registry
   - Builds multi-platform images (amd64, arm64)
   - Tags with branch name, commit SHA, and `latest`
   - Uses Docker layer caching for speed

**Published Images:**
```bash
ghcr.io/m-letourneur/demoagor/backend:latest
ghcr.io/m-letourneur/demoagor/backend:main-<sha>
```

**Using Published Images:**
```bash
# Pull the latest image
docker pull ghcr.io/m-letourneur/demoagor/backend:latest

# Update docker-compose.yml to use published image
# services:
#   backend:
#     image: ghcr.io/m-letourneur/demoagor/backend:latest
```

### Dependency Management

**Configuration:** `.github/dependabot.yml`

Dependabot automatically:
- Updates npm dependencies (weekly on Mondays)
- Updates Docker base images (weekly on Mondays)
- Updates GitHub Actions versions (weekly on Mondays)
- Groups minor/patch updates together
- Creates PRs with proper labels

### Running CI Locally

**Quick test:**
```bash
# Run the same checks as CI
docker compose up -d
sleep 10
curl http://localhost:3000/health
curl http://localhost:3000/api/hello
docker compose down -v
```

**Full CI simulation:**
```bash
# Test multiple Node versions
for version in 18 20; do
  echo "Testing with Node $version"
  docker run --rm -v "$PWD/backend":/app -w /app node:$version npm ci
done

# Build and test with docker-compose
docker compose up -d
docker compose ps
curl -f http://localhost:3000/health
docker compose down -v
```

### Workflow Customization

**Add linting:**
1. Install ESLint or Prettier in `backend/package.json`
2. Update the "Lint" job in `.github/workflows/ci.yml`

**Add deployment:**
1. Uncomment the `deploy` job in `.github/workflows/cd.yml`
2. Add deployment commands (kubectl, ssh, terraform, etc.)
3. Configure GitHub Environment secrets

**Modify security scans:**
- Edit Trivy severity levels in `.github/workflows/ci.yml`
- Add additional security tools (Snyk, Grype, etc.)

## Learn More

- [Agor Documentation](https://agor.live)
- [Environment Configuration Reference](https://github.com/agor-ai/agor)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
