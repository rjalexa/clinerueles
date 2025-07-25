# Development Tooling and Workflow Rules

This document defines the mandatory development workflow, tooling requirements, and compliance rules for all projects. These rules ensure consistent development practices, reproducible builds, and standardized deployment procedures.

## Core Development Principles

### Container-First Development

**Rule 1: Always Use Containers**
- **MANDATORY**: Launch all project artifacts as containers, never as local processes
- All applications must run in containerized environments
- No direct execution of local binaries or scripts outside of containers

**Rule 2: Docker Compose Command**
- **MANDATORY**: Use `docker compose` (new syntax) for all container orchestration
- **FORBIDDEN**: Never use the deprecated `docker-compose` command (old syntax with hyphen)
- All compose operations must use the modern Docker CLI integrated command

## Package Management

### Python Development

**Rule 3: Python Package Management with Astral UV**
- **MANDATORY**: Manage all Python packages using Astral UV with `pyproject.toml`
- **MANDATORY**: Use `uv sync` for dependency synchronization
- **FORBIDDEN**: Never use `pip` for package installation or management
- All Python dependencies must be declared in `pyproject.toml` and managed through UV
- **Legacy Support**: Poetry support maintained for compatibility where existing

**Python Development Best Practices**:
- Use Astral UV for dependency management
- Follow PEP 8 coding standards
- Use type hints where applicable
- Structure modules by feature/domain

### Frontend Development

**Rule 4: React Package Management**
- **MANDATORY**: For React projects, use `pnpm` as the package manager
- **FORBIDDEN**: Never use `npm` for React project dependency management
- All React dependencies must be managed through pnpm
- **Lock File**: Use `pnpm-lock.yaml` for dependency locking

**Rule 5: Pre-Build Code Quality Validation**

**Python Projects**:
- **MANDATORY**: Before building a Python container, run the following commands and fix all issues:
  ```bash
  ruff format .
  ruff check --fix .
  ```
- **MANDATORY**: All ruff formatting and linting errors must be resolved prior to Docker build process
- Code must pass both formatting and linting checks before containerization
- Use ruff for consistent code formatting and quality enforcement

**Frontend Projects**:
- **MANDATORY**: Before building a React/frontend container, run `pnpm lint` and fix any errors
- **MANDATORY**: Run `pnpm lint --fix` to automatically fix linting issues where possible
- Code quality must be verified before containerization
- All linting errors must be resolved prior to Docker build process
- **MANDATORY**: Run TypeScript type checking before building containers

**React Development Best Practices**:
- **MANDATORY**: Use TypeScript for all React components and logic
- **MANDATORY**: Use Tailwind CSS for styling
- **MANDATORY**: Use Vite as the build tool
- **MANDATORY**: Follow strict TypeScript configuration
- **MANDATORY**: Use functional components with hooks
- **MANDATORY**: Implement proper component prop typing
- Use modern React patterns (hooks, context, suspense)
- Implement proper error boundaries
- Use consistent naming conventions (PascalCase for components, camelCase for functions)
- Organize imports: React imports first, then third-party, then local imports


## Dockerfile Authoring Rules

**Rule 11: Dockerfile = Build Only**
- **MANDATORY**: The Dockerfile must **only** describe how to **build** the image in the most efficient and smallest way possible.
- **FORBIDDEN**: Any instruction about **how to run** the container (commands, arguments, environment, ports, volumes, networks, restart policies, replicas, resource limits, etc.) must **only** appear in Docker Compose files.
- **MANDATORY**: Prefer **multi‑stage builds** to ensure the final image is minimal.
- **MANDATORY**: Use the **smallest still-supported base image** that satisfies the project’s requirements (e.g., `python:3.12-slim`, `alpine`, `distroless`, `ubi-micro`, etc.), and keep it **recent** to receive security patches.
- **MANDATORY**: Remove all build-time tools, caches and temporary files in the final stage.
- **MANDATORY**: Provide a proper `.dockerignore` to keep the build context minimal.
- **RECOMMENDED**: Use BuildKit features such as `--mount=type=cache` to cache package managers (uv/pip/pnpm) during the build.
- **RECOMMENDED**: Pin dependency versions where sensible to ensure reproducible builds.
- **RECOMMENDED**: Run as a non-root user in the final stage when possible.
- **OPTIONAL**: `ENTRYPOINT`/`CMD` can be minimal or omitted; the **effective runtime command must be set in Compose**.

## Linting Dockerfiles

**Rule 12: Dockerfiles must pass `hadolint`**
- **MANDATORY**: All Dockerfiles must be linted with **hadolint** locally and in CI.
- **MANDATORY**: The CI pipeline **must fail** on any `error` **or** `warning` severities.
- **MANDATORY**: Any rule suppression must be:
  - Declared in a **project-wide `.hadolint.yaml`** with a **short rationale**, **or**
  - Inline via `# hadolint ignore=DLXXXX` **with a reference to the issue/PR explaining why**.
- **RECOMMENDED**: Pin the hadolint version in CI for reproducibility.
- **RECOMMENDED**: Keep the exception list short and reviewed periodically.

### Sample `.hadolint.yaml`
```yaml
failure-threshold: warning   # pipeline fails on warnings and above
ignored:
  # Keep this list short, each with a comment explaining *why* it is safe to ignore.
  # - DL3008  # Example: apt-get install without --no-install-recommends (document justification)
```

### Example CI step
```yaml
- name: Lint Dockerfiles
  run: |
    hadolint $(git ls-files '*Dockerfile*') --config .hadolint.yaml
```

## Container Development Workflow

### Frontend Container Deployment

**Rule 6: Frontend Container Deployment**
- **MANDATORY**: Launch frontend applications by rebuilding their Docker image and launching with `docker compose`
- **FORBIDDEN**: Never use `pnpm run` or any local package manager commands to start frontend applications
- Frontend must always be containerized and orchestrated through Docker Compose

**Rule 7: Frontend Container Build and Test Process**
- **MANDATORY**: To build and test a new version of a frontend container always use:
  ```bash
  docker compose down FRONTENDNAME
  docker compose up -d FRONTENDNAME --build
  ```
- This ensures clean shutdown of existing containers before rebuilding
- Forces fresh build of the frontend container image
- Launches in detached mode for testing

### Development Workflow Commands

**Backend Development**:
```bash
cd docker
docker compose down backend
docker compose up -d backend --build
```

**Frontend Development**:
```bash
cd docker
docker compose down frontend
docker compose up -d frontend --build
```

**Full Stack Development**:
```bash
cd docker
docker compose down
docker compose up -d --build
```

**Development Mode Testing**:
```bash
# For backend testing
docker compose exec backend python -m src.main --help

# For frontend testing
docker compose logs frontend
```

## Environment Configuration

### Centralized Environment Management

**Rule 8: Root-Level Environment Variables Only**
- **MANDATORY**: All environment variables must be stored in the root `.env` file only
- **FORBIDDEN**: Environment variables in subdirectories (e.g., `frontend/.env`, `src/.env`)
- **MANDATORY**: Use a single `.env.example` template at the root level
- Both backend and frontend applications must read from the root `.env` file
- Docker Compose should mount the root `.env` file to all containers

**Environment Variable Naming Conventions**:
- **Backend variables**: Use standard naming (e.g., `API_KEY`, `DATABASE_HOST`)
- **Frontend variables**: Prefix with `VITE_` for Vite projects (e.g., `VITE_API_URL`)
- **Docker variables**: Use `COMPOSE_` prefix for Docker Compose settings
- **Shared variables**: Can be used by both backend and frontend (e.g., `APP_ENV`)

## Database Integration

**Rule 9: Database Configuration**
- Place database initialization scripts in `/docker/init-scripts/`
- Use environment variables for database configuration
- Implement proper connection pooling
- Follow database naming conventions
- Mount database data as Docker volumes for persistence

## Testing and Quality Assurance

**Rule 10: Testing Requirements**
- **MANDATORY**: Run all tests in containerized environments
- Follow testing framework conventions (pytest for Python, Jest for React)
- Include unit, integration, and end-to-end tests
- Test data should be minimal and focused
- Separate test types into different directories

**Testing Commands**:
```bash
# Python tests
docker compose exec backend python -m pytest tests/

# Frontend tests
docker compose exec frontend pnpm test

# End-to-end tests
docker compose exec e2e pnpm test:e2e
```

## Compliance Requirements

### Mandatory Rules

**Project Structure**:
- **MANDATORY**: All new code must follow the standardized project structure
- **MANDATORY**: Core backend logic only in `/src/` directory
- **MANDATORY**: Frontend code only in `/frontend/` directory
- **MANDATORY**: All Docker files in `/docker/` directory

**Package Management**:
- **MANDATORY**: Use UV for Python package management
- **MANDATORY**: Use pnpm for React package management
- **MANDATORY**: Dependencies declared in appropriate configuration files

**Containerization**:
- **MANDATORY**: Use Docker containers for all deployments
- **MANDATORY**: Frontend applications must be containerized
- **MANDATORY**: Use `docker compose` for orchestration

**Code Quality**:
- **MANDATORY**: Run linting before building frontend containers
- **MANDATORY**: Resolve all TypeScript errors before deployment
- **MANDATORY**: Follow language-specific coding standards

### Forbidden Practices

**Package Management**:
- **FORBIDDEN**: Using `pip` for Python package management
- **FORBIDDEN**: Using `npm` for React projects (use pnpm instead)
- **FORBIDDEN**: Installing packages outside of containerized environments

**Project Organization**:
- **FORBIDDEN**: Business logic outside `/src/` directory
- **FORBIDDEN**: Frontend code outside `/frontend/` directory
- **FORBIDDEN**: Data files committed to git
- **FORBIDDEN**: Configuration secrets in code
- **FORBIDDEN**: Environment variables in subdirectories

**Development Workflow**:
- **FORBIDDEN**: Using deprecated `docker-compose` command (use `docker compose`)
- **FORBIDDEN**: Running applications outside of containers
- **FORBIDDEN**: Direct execution of local binaries for production code

## Deployment Procedures

### Production Deployment

**Pre-Deployment Checklist**:
1. All tests passing in containerized environment
2. Linting and type checking completed
3. Environment variables properly configured
4. Database migrations applied
5. Security scan completed

**Deployment Commands**:
```bash
# Production build
docker compose -f docker-compose.prod.yml build

# Production deployment
docker compose -f docker-compose.prod.yml up -d

# Health check
docker compose -f docker-compose.prod.yml ps
```

### Development vs Production

**Development Environment**:
- Use development Docker Compose configuration
- Enable hot reloading where applicable
- Include development tools and debugging utilities
- Use development environment variables

**Production Environment**:
- Use production-optimized Docker images
- Exclude development dependencies
- Enable production optimizations
- Use production environment variables
- Implement proper logging and monitoring

## Summary

These rules ensure:
- **Consistent Development Environment**: All developers use identical containerized setups
- **Modern Tooling**: Latest Docker CLI, UV for Python, pnpm for React
- **Quality Assurance**: Mandatory linting, type checking, and testing
- **Reproducible Builds**: Standardized container build and deployment procedures
- **Security**: Centralized environment management and no secrets in code
- **Maintainability**: Clear separation of concerns and standardized workflows

**Non-compliance with these rules is not acceptable and must be corrected immediately.**

## Quick Reference

### Common Commands

**Start Development Environment**:
```bash
cd docker && docker compose up -d --build
```

**Rebuild Specific Service**:
```bash
docker compose down SERVICE_NAME
docker compose up -d SERVICE_NAME --build
```

**View Logs**:
```bash
docker compose logs SERVICE_NAME -f
```

**Execute Commands in Container**:
```bash
docker compose exec SERVICE_NAME COMMAND
```

**Clean Up**:
```bash
docker compose down
docker system prune -f
```

### Package Management Quick Reference

**Python (UV)**:
```bash
# Add dependency
uv add package_name

# Sync dependencies
uv sync

# Remove dependency
uv remove package_name
```

**React (pnpm)**:
```bash
# Install dependencies
pnpm install

# Add dependency
pnpm add package_name

# Add dev dependency
pnpm add -D package_name

# Remove dependency
pnpm remove package_name

# Run linting
pnpm lint

# Run tests
pnpm test
