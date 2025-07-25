# Development Rules - Compact Version

## Container-First Development

### MANDATORY
- **Always use containers** - Never run local processes
- **Use `docker compose`** - Never use deprecated `docker-compose`
- **No `version` attribute** in Docker Compose files
- **Launch all apps via containers** - Never use local package managers to start apps

### Build & Deploy Commands
```bash
# Standard rebuild pattern
docker compose down SERVICE_NAME
docker compose up -d SERVICE_NAME --build
```

## Package Management

### Python
- **MANDATORY**: Use Astral UV with `pyproject.toml`
- **FORBIDDEN**: Never use `pip`
- **Commands**: `uv add`, `uv sync`, `uv remove`

### React/Frontend
- **MANDATORY**: Use `pnpm`
- **FORBIDDEN**: Never use `npm`
- **Commands**: `pnpm add`, `pnpm install`, `pnpm lint`

## Pre-Build Quality Checks

### Python
**MANDATORY before Docker build**:
```bash
ruff format .
ruff check --fix .
```

### Frontend
**MANDATORY before Docker build**:
```bash
pnpm lint --fix
# TypeScript checking required
```

## Dockerfile Rules

### Build-Only Focus
- **Dockerfiles = Build instructions ONLY**
- **Runtime config goes in Docker Compose ONLY**
- **Use multi-stage builds**
- **Use smallest supported base images** (e.g., `python:3.12-slim`, `alpine`)
- **Remove all build tools/caches in final stage**
- **Provide `.dockerignore`**

### Syntax
- **UPPERCASE**: `FROM` and `AS` keywords
- **Example**: `FROM node:18-alpine AS builder`

### Linting
- **MANDATORY**: Pass `hadolint` locally and in CI
- **CI must fail on errors AND warnings**
- **Suppressions need justification** in `.hadolint.yaml` or inline

## Environment Configuration

### Single .env File
- **MANDATORY**: Root `.env` file ONLY
- **FORBIDDEN**: Subdirectory .env files
- **Naming**:
  - Backend: `API_KEY`, `DATABASE_HOST`
  - Frontend: `VITE_API_URL` (Vite prefix)
  - Docker: `COMPOSE_` prefix

## General Practices

### FORBIDDEN
- Data files in git
- Secrets in code
- Running apps outside containers

## Frontend Development

### React Best Practices (MANDATORY)
- TypeScript for all components
- Tailwind CSS for styling
- Vite as build tool
- Functional components with hooks
- Proper prop typing
- Error boundaries

## Testing

### MANDATORY
- Run all tests in containers
- Test structure: unit, integration, e2e
- Commands:
  ```bash
  docker compose exec backend python -m pytest tests/
  docker compose exec frontend pnpm test
  docker compose exec e2e pnpm test:e2e
  ```

## Quick Reference

### Common Commands
```bash
# Full stack
cd docker && docker compose up -d --build

# Single service rebuild
docker compose down SERVICE && docker compose up -d SERVICE --build

# Logs
docker compose logs SERVICE -f

# Execute in container
docker compose exec SERVICE COMMAND

# Cleanup
docker compose down && docker system prune -f
```

### Package Commands
```bash
# Python (UV)
uv add/remove/sync

# React (pnpm)
pnpm add/remove/install/lint/test
```

## Deployment

### Pre-Deployment Checklist
1. Tests passing in containers
2. Linting/type checking complete
3. Environment variables configured
4. Database migrations applied
5. Security scan complete

### Production Commands
```bash
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml ps
```

## Summary of Key Rules

**ALWAYS**:
- Containers for everything
- `docker compose` (no hyphen)
- UV for Python, pnpm for React
- Lint before build
- Root .env only
- Uppercase FROM/AS in Dockerfiles

**NEVER**:
- Local processes
- `pip` or `npm`
- `version` in compose files
- Subdirectory .env files
- Lowercase from/as in Dockerfiles

**Non-compliance is not acceptable and must be corrected immediately.**