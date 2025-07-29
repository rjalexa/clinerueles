# AGENTS.md - AI Agent Development Rules

## System Context

You are an AI agent assisting with modern containerized development. This document defines mandatory rules for consistent, secure, and maintainable software development practices.

## Core Principles

### 1. Container-First Development
- **ALWAYS** use containers for all applications
- **NEVER** run local processes for production code
- **USE** `docker compose` (modern syntax without hyphen)
- **FORBIDDEN**: `docker-compose` (deprecated), `version` attribute in compose files

### 2. Package Management Standards

#### Python Projects
- **MANDATORY**: Astral UV with `pyproject.toml`
- **FORBIDDEN**: `pip` for any package operations
- **COMMANDS**: `uv add`, `uv sync`, `uv remove`

#### React/Frontend Projects
- **MANDATORY**: `pnpm` package manager
- **FORBIDDEN**: `npm` for any operations
- **COMMANDS**: `pnpm add`, `pnpm install`, `pnpm lint`

## Pre-Build Quality Gates

### Python Quality Checks
Before building any Python container, execute:
```bash
ruff format .
ruff check --fix .
```

### Frontend Quality Checks
Before building any React container, execute:
```bash
pnpm lint --fix
# TypeScript checking is mandatory
```

## Dockerfile Standards

### Build Philosophy
- Dockerfiles contain **BUILD** instructions ONLY
- Runtime configuration belongs in Docker Compose ONLY
- Use multi-stage builds for minimal final images
- Use smallest supported base images (e.g., `python:3.12-slim`, `alpine`)
- Remove all build tools/caches in final stage

### Syntax Requirements
- **UPPERCASE**: `FROM` and `AS` keywords
- **Example**: `FROM node:18-alpine AS builder`
- **Linting**: Must pass `hadolint` with no errors or warnings

## Project Structure

### Directory Layout
```
project-name/
├── .env                    # Root environment variables ONLY
├── .env.example           # Environment template
├── docker/                # All Docker configurations
│   ├── docker-compose.yml
│   └── Dockerfile.*
├── frontend/              # React TypeScript application
│   ├── src/
│   │   ├── components/    # Reusable components
│   │   ├── pages/        # Page components
│   │   ├── hooks/        # Custom React hooks
│   │   ├── services/     # API services
│   │   ├── store/        # State management
│   │   ├── types/        # TypeScript definitions
│   │   └── utils/        # Utility functions
│   └── package.json
├── src/                   # Backend application code
│   ├── main.py           # Entry point
│   ├── config.py         # Configuration
│   └── modules/          # Feature modules
├── tests/                # Test suites
└── data/                 # Runtime data (not in git)
```

### Environment Configuration
- **SINGLE** `.env` file at root level ONLY
- **FORBIDDEN**: Subdirectory `.env` files
- **Backend vars**: `API_KEY`, `DATABASE_HOST`
- **Frontend vars**: `VITE_` prefix (e.g., `VITE_API_URL`)
- **Docker vars**: `COMPOSE_` prefix

## Development Workflow

### Container Operations

#### Rebuild Single Service
```bash
docker compose down SERVICE_NAME
docker compose up -d SERVICE_NAME --build
```

#### Full Stack Rebuild
```bash
cd docker && docker compose up -d --build
```

#### View Logs
```bash
docker compose logs SERVICE_NAME -f
```

## Frontend Development Standards

### React Requirements
- **TypeScript** for all components
- **Tailwind CSS** for styling
- **Vite** as build tool
- **Functional components** with hooks
- **Proper prop typing** with TypeScript
- **Error boundaries** implementation

### Code Organization
- Components: PascalCase naming
- Functions: camelCase naming
- Imports order: React → third-party → local

## Testing Requirements

### Test Execution
All tests run in containers:
```bash
# Backend tests
docker compose exec backend python -m pytest tests/

# Frontend tests
docker compose exec frontend pnpm test

# E2E tests
docker compose exec e2e pnpm test:e2e
```

### Test Structure
- Unit tests: `/tests/unit/`
- Integration tests: `/tests/integration/`
- E2E tests: `/tests/e2e/`

## Security & Best Practices

### Forbidden Practices
- Data files in git repositories
- Secrets in code
- Running apps outside containers
- Using deprecated commands
- Environment variables in subdirectories

### Mandatory Practices
- Lint before build
- Type checking for TypeScript
- Container-only deployments
- Root `.env` configuration
- Proper `.dockerignore` files

## Quick Reference Commands

### Package Management
```bash
# Python (UV)
uv add package_name
uv sync
uv remove package_name

# React (pnpm)
pnpm add package_name
pnpm add -D package_name  # dev dependency
pnpm remove package_name
```

### Docker Operations
```bash
# Start development
cd docker && docker compose up -d --build

# Rebuild specific service
docker compose down SERVICE && docker compose up -d SERVICE --build

# Execute in container
docker compose exec SERVICE COMMAND

# Cleanup
docker compose down && docker system prune -f
```

## Deployment Checklist

Before deployment, ensure:
1. ✓ All tests pass in containers
2. ✓ Linting and type checking complete
3. ✓ Environment variables configured
4. ✓ Database migrations applied
5. ✓ Security scan completed

## Error Handling

When encountering issues:
1. Check container logs: `docker compose logs SERVICE_NAME`
2. Verify environment variables are properly set
3. Ensure all quality checks pass before build
4. Confirm using correct package managers (UV/pnpm)

## Agent Instructions

When generating code or commands:
- **ALWAYS** assume containerized environment
- **NEVER** suggest local process execution
- **USE** modern Docker Compose syntax
- **ENFORCE** UV for Python, pnpm for React
- **REQUIRE** quality checks before builds
- **MAINTAIN** single root `.env` configuration

## Compliance

**Non-compliance with these rules requires immediate correction.**