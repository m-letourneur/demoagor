# Agor Multi-Worktree Setup

This document explains how the demoagor project is configured for Agor's multi-worktree feature.

## Overview

The setup allows **multiple worktrees to run Docker Compose environments in parallel** without port conflicts. Each worktree gets dynamically calculated ports based on its `unique_id`.

## Configuration Files

### `.agor.yml` - Agor Environment Config

Uses Agor's template syntax to generate dynamic commands:

```yaml
environment:
  start: BACKEND_PORT={{add 3000 worktree.unique_id}} POSTGRES_PORT={{add 5432 worktree.unique_id}} docker compose -p demoagor-{{worktree.name}} up -d
  stop: docker compose -p demoagor-{{worktree.name}} down
  health: http://localhost:{{add 3000 worktree.unique_id}}/health
  app: http://localhost:{{add 3000 worktree.unique_id}}
  logs: docker compose -p demoagor-{{worktree.name}} logs --tail=100
```

**Template Variables:**
- `{{worktree.unique_id}}` - Numeric ID assigned to worktree (1, 2, 3, ...)
- `{{worktree.name}}` - Name of the worktree branch
- `{{add X Y}}` - Helper function to add two numbers

**Example Expansion (worktree.unique_id=5, worktree.name="initial-setup"):**
```bash
start: BACKEND_PORT=3005 POSTGRES_PORT=5437 docker compose -p demoagor-initial-setup up -d
health: http://localhost:3005/health
```

### `docker-compose.yml` - Dynamic Port Configuration

Accepts environment variables with fallback defaults:

```yaml
services:
  postgres:
    ports:
      - "${POSTGRES_PORT:-5432}:5432"  # Host port dynamic, container port fixed

  backend:
    ports:
      - "${BACKEND_PORT:-3000}:${BACKEND_PORT:-3000}"  # Both dynamic
    environment:
      PORT: ${BACKEND_PORT:-3000}  # Pass to Node.js app
```

**How it works:**
- `${BACKEND_PORT:-3000}` means "use BACKEND_PORT if set, otherwise default to 3000"
- The left side of `:` is the host port (visible externally)
- The right side of `:` is the container port (internal)
- For the backend, both sides use the same variable so the app listens on the correct port

## Port Allocation Strategy

| Worktree unique_id | Backend Port | PostgreSQL Port | Example Branch |
|--------------------|--------------|-----------------|----------------|
| 1                  | 3001         | 5433            | feature-auth   |
| 2                  | 3002         | 5434            | bugfix-login   |
| 3                  | 3003         | 5435            | refactor-api   |
| 5                  | 3005         | 5437            | initial-setup  |
| 10                 | 3010         | 5442            | experiment     |

**Base ports:**
- Backend: 3000 + unique_id
- PostgreSQL: 5432 + unique_id

## Docker Compose Project Isolation

The `-p demoagor-{{worktree.name}}` flag creates **separate Docker Compose projects**:

```bash
# Each worktree gets its own isolated project
docker compose -p demoagor-feature-1 up -d   # Separate containers
docker compose -p demoagor-feature-2 up -d   # Separate volumes
docker compose -p demoagor-main up -d        # Separate networks
```

**Benefits:**
- Containers are named uniquely (e.g., `demoagor-feature-1-backend-1`)
- Volumes are isolated (e.g., `demoagor-feature-1_postgres_data`)
- Networks don't interfere with each other
- Can stop one worktree without affecting others

## Testing Multi-Worktree Setup

### Via Agor MCP Tools (from Claude Code)

```javascript
// Agor automatically expands templates based on worktree metadata
await use_mcp_tool('mcp__agor__agor_environment_start', {
  worktreeId: '48d88ef6-...'  // Agor calculates ports and runs command
});

await use_mcp_tool('mcp__agor__agor_environment_health', {
  worktreeId: '48d88ef6-...'  // Checks dynamic URL
});
```

### Manual Testing (simulating worktrees)

```bash
# Start two "worktrees" manually
BACKEND_PORT=3001 POSTGRES_PORT=5433 docker compose -p demoagor-wt1 up -d
BACKEND_PORT=3002 POSTGRES_PORT=5434 docker compose -p demoagor-wt2 up -d

# Verify both are accessible
curl http://localhost:3001/health
curl http://localhost:3002/health

# View all containers
docker ps | grep demoagor

# Stop specific worktree
docker compose -p demoagor-wt1 down
```

## Key Design Decisions

1. **Fixed container ports, dynamic host ports**
   - PostgreSQL always listens on 5432 *inside* the container
   - But mapped to 5433, 5434, etc. on the host
   - This simplifies configuration and avoids breaking app code

2. **Backend uses dynamic port everywhere**
   - Both host and container use `${BACKEND_PORT}`
   - The Node.js app reads `process.env.PORT`
   - This ensures the healthcheck and app logic stay in sync

3. **Project name includes worktree name**
   - Makes `docker ps` output readable
   - Easy to identify which containers belong to which worktree
   - Follows Agor's naming convention (seen in agor repo itself)

4. **Base ports chosen to avoid common conflicts**
   - 3000+ for backend (avoids default 3000)
   - 5432+ for PostgreSQL (avoids default 5432)
   - Even with unique_id=1, we use 3001 to prevent base port collision

## Troubleshooting

### Port conflicts
If you see "port already in use" errors:
```bash
# Find what's using the port
lsof -i :3001

# Stop all demoagor containers
docker ps | grep demoagor | awk '{print $1}' | xargs docker stop

# Or use docker compose ls to see all projects
docker compose ls
```

### Health check failures
```bash
# Check logs for the specific project
docker compose -p demoagor-initial-setup logs backend

# Verify the backend is listening on the correct port
docker compose -p demoagor-initial-setup exec backend env | grep PORT
```

### Database connection issues
```bash
# Verify PostgreSQL is healthy
docker compose -p demoagor-initial-setup ps

# Check database connectivity
docker compose -p demoagor-initial-setup exec postgres psql -U demo -d demoagor -c "SELECT 1"
```

## References

- Agor repository: `/Users/marc/.agor/repos/agor/.agor.yml`
- Docker Compose project names: https://docs.docker.com/compose/reference/#use--p-to-specify-a-project-name
- Agor environment docs: https://agor.live
