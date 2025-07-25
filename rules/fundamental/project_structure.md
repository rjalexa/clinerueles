# Standardized Project Structure

This document defines the standardized project structure for all development projects. This structure ensures maintainability, clear separation of concerns, and deployment readiness across different technology stacks.

## Root Directory Structure

```
project-name/
├── .clinerules/                    # Cline AI development rules and guidelines
├── .dockerignore                   # Docker build exclusions
├── .env                           # Environment variables (not in git)
├── .env.example                   # Environment template (in git)
├── .gitignore                     # Git exclusions
├── README.md                      # Main project documentation
├── additionaldocs/                # Supplementary documentation
├── data/                          # Runtime data directories
├── docker/                        # Docker configuration and orchestration
├── frontend/                      # Frontend application (if applicable)
├── prompts/                       # AI prompt templates (if applicable)
├── scripts/                       # Automation and utility scripts
├── src/                           # Core backend/application source code
├── tests/                         # Test suite
└── tools/                         # Development and debugging utilities
```

## Technology-Specific Configuration Files

### Python Projects
- **`pyproject.toml`**: Python project metadata, dependencies (UV/pip)
- **`uv.lock`**: UV dependency lock file (preferred)
- **`poetry.lock`**: Poetry lock file (legacy, maintain for compatibility)

### Node.js/React Projects
- **`package.json`**: Node.js project metadata and dependencies
- **`pnpm-lock.yaml`**: pnpm dependency lock file (preferred for React)

## Core Directories

### `/src/` - Backend/Core Application Source Code

**Purpose**: Contains the main backend application logic and core modules.

**Structure for Python Projects**:
```
src/
├── __init__.py                    # Package initialization
├── config.py                     # Configuration management
├── database.py                   # Database operations and schema
├── main.py                       # Main entry point
└── modules/                      # Feature-specific modules
    ├── __init__.py
    ├── auth/                     # Authentication module
    ├── api/                      # API endpoints
    └── services/                 # Business logic services
```

**Structure for Node.js Projects**:
```
src/
├── index.js                      # Main entry point
├── config/                       # Configuration files
├── controllers/                  # Request handlers
├── models/                       # Data models
├── routes/                       # API routes
├── services/                     # Business logic
└── middleware/                   # Express middleware
```

**Rules**:
- All production backend code must be in this directory
- Follow language-specific package conventions
- Each module should have a single, clear responsibility
- No business logic should exist outside this directory

### `/frontend/` - Frontend Application (Modern React TypeScript)

**Purpose**: Contains the frontend application code when the project includes a web interface.

**Structure for Modern Vite React TypeScript Tailwind Projects**:
```
frontend/
├── package.json                  # Frontend dependencies
├── pnpm-lock.yaml               # pnpm lock file
├── vite.config.ts               # Vite build configuration
├── tsconfig.json                # TypeScript configuration
├── tsconfig.node.json           # TypeScript config for Node.js
├── tailwind.config.js           # Tailwind CSS configuration
├── postcss.config.js            # PostCSS configuration
├── index.html                   # Main HTML template
├── public/                      # Static assets
│   ├── favicon.ico
│   ├── vite.svg
│   └── assets/
│       ├── images/
│       └── icons/
├── src/                         # React TypeScript source code
│   ├── main.tsx                 # React entry point
│   ├── App.tsx                  # Main App component
│   ├── vite-env.d.ts            # Vite environment types
│   ├── components/              # Reusable components
│   │   ├── ui/                  # Base UI components (buttons, inputs, etc.)
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── index.ts         # Component exports
│   │   ├── layout/              # Layout components
│   │   │   ├── Header.tsx
│   │   │   ├── Footer.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Layout.tsx
│   │   ├── forms/               # Form components
│   │   └── common/              # Shared/common components
│   ├── pages/                   # Page components
│   │   ├── Home.tsx
│   │   ├── About.tsx
│   │   ├── Dashboard.tsx
│   │   └── NotFound.tsx
│   ├── hooks/                   # Custom React hooks
│   │   ├── useApi.ts
│   │   ├── useAuth.ts
│   │   └── useLocalStorage.ts
│   ├── services/                # API services and external integrations
│   │   ├── api.ts               # API client configuration
│   │   ├── auth.ts              # Authentication service
│   │   └── types.ts             # API response types
│   ├── store/                   # State management (Zustand/Redux)
│   │   ├── authStore.ts
│   │   ├── appStore.ts
│   │   └── index.ts
│   ├── utils/                   # Utility functions
│   │   ├── constants.ts         # Application constants
│   │   ├── helpers.ts           # Helper functions
│   │   ├── formatters.ts        # Data formatters
│   │   └── validators.ts        # Validation functions
│   ├── types/                   # TypeScript type definitions
│   │   ├── api.ts               # API types
│   │   ├── auth.ts              # Authentication types
│   │   └── global.ts            # Global types
│   ├── styles/                  # Global styles and Tailwind
│   │   ├── globals.css          # Global CSS with Tailwind imports
│   │   ├── components.css       # Component-specific styles
│   │   └── utilities.css        # Custom utility classes
│   └── assets/                  # Static assets
│       ├── images/
│       ├── icons/
│       └── fonts/
├── Dockerfile                   # Frontend container definition
├── .eslintrc.cjs                # ESLint configuration
├── .prettierrc                  # Prettier configuration
├── .gitignore                   # Git ignore rules
└── README.md                    # Frontend documentation
```

**Rules for Modern React TypeScript Frontend**:
- **MANDATORY**: Use pnpm for package management
- **MANDATORY**: Use TypeScript for all React components and logic
- **MANDATORY**: Use Tailwind CSS for styling
- **MANDATORY**: Use Vite as the build tool
- **MANDATORY**: All frontend code must be containerized
- **MANDATORY**: Run linting and type checking before building containers
- **MANDATORY**: Follow strict TypeScript configuration
- **MANDATORY**: Use functional components with hooks
- **MANDATORY**: Implement proper component prop typing
- Follow component-based architecture with clear separation
- Use modern React patterns (hooks, context, suspense)
- Implement proper error boundaries
- Use consistent naming conventions (PascalCase for components, camelCase for functions)
- Organize imports: React imports first, then third-party, then local imports

### `/docker/` - Containerization

**Purpose**: All Docker-related configuration and orchestration files.

**Structure**:
```
docker/
├── docker-compose.yml             # Service orchestration
├── Dockerfile.backend            # Backend container definition
├── Dockerfile.frontend           # Frontend container definition (if applicable)
└── init-scripts/                 # Database/service initialization scripts
    ├── db-init.sql               # Database setup
    └── redis-init.conf           # Cache setup
```

**Rules**:
- All Docker files must be in this directory
- Use `docker compose` (not `docker-compose`) for orchestration
- Container builds must be reproducible and follow best practices
- Environment-specific configurations via `.env` file
- Separate Dockerfiles for different services when needed

### `/data/` - Runtime Data

**Purpose**: Persistent data storage for application runtime.

**Structure**:
```
data/
├── input/                         # Input files for processing
│   └── .gitkeep                   # Preserve empty directory in git
├── output/                        # Processing results and exports
│   └── .gitkeep
├── uploads/                       # User uploaded files
│   └── .gitkeep
└── logs/                          # Application and processing logs
    └── .gitkeep
```

**Rules**:
- **NEVER** commit actual data files to git
- Use `.gitkeep` files to preserve directory structure
- Mount these directories as Docker volumes
- Organize by data flow: input → processing → output

### `/tools/` - Development Utilities

**Purpose**: Optional development, debugging, and analysis tools.

**Structure**:
```
tools/
├── db_query.py                    # Database query utilities
├── performance_analysis.py       # Performance monitoring
├── data_migration.py             # Data migration scripts
└── debug_helpers.js              # Debugging utilities
```

**Rules**:
- These are **optional** utilities, not core application code
- Should not be required for normal application operation
- Can be excluded from production deployments
- Useful for debugging, analysis, and development workflows

### `/scripts/` - Automation Scripts

**Purpose**: Shell scripts and automation tools for common tasks.

**Structure**:
```
scripts/
├── setup.sh                      # Project setup automation
├── deploy.sh                     # Deployment automation
├── backup.sh                     # Data backup scripts
└── test.sh                       # Test execution scripts
```

**Rules**:
- Provide simplified interfaces for complex operations
- Should be executable and well-documented
- Follow shell scripting best practices
- Include error handling and validation

### `/prompts/` - AI Prompt Templates (Optional)

**Purpose**: AI prompt templates and configurations for projects using LLM integration.

**Structure**:
```
prompts/
├── system_prompt.md              # Main system prompt
├── user_prompts/                 # User interaction prompts
└── experimental/                 # Development/experimental prompts
```

**Rules**:
- Version prompts with clear naming conventions
- Document prompt changes and their effects
- Keep production and experimental prompts separate
- Include metadata about prompt performance

### `/tests/` - Test Suite

**Purpose**: Unit tests, integration tests, and test utilities.

**Structure**:
```
tests/
├── unit/                         # Unit tests
│   ├── backend/                  # Backend unit tests
│   └── frontend/                 # Frontend unit tests
├── integration/                  # Integration tests
├── e2e/                          # End-to-end tests
├── fixtures/                     # Test data and fixtures
└── utils/                        # Test utilities
```

**Rules**:
- Follow testing framework conventions (pytest, Jest, etc.)
- Mirror the `src/` directory structure for test organization
- Include unit, integration, and end-to-end tests
- Test data should be minimal and focused
- Separate test types into different directories

### `/additionaldocs/` - Supplementary Documentation

**Purpose**: Additional documentation beyond the main README.

**Structure**:
```
additionaldocs/
├── API_DOCUMENTATION.md          # API reference
├── DEPLOYMENT_GUIDE.md           # Deployment instructions
├── ARCHITECTURE.md               # System architecture
└── TROUBLESHOOTING.md            # Common issues and solutions
```

**Rules**:
- Use for detailed technical documentation
- Keep main README focused and concise
- Include implementation details and advanced usage
- Document features and architectural decisions

## Configuration Files

### Root Level Configuration

**Common Files**:
- **`.env`**: Environment variables (local, not in git)
- **`.env.example`**: Environment template (in git)
- **`.dockerignore`**: Docker build exclusions
- **`.gitignore`**: Git exclusions

**Python Projects**:
- **`pyproject.toml`**: Python project metadata, dependencies (UV/pip)
- **`uv.lock`**: UV dependency lock file (preferred)

**Node.js/React Projects**:
- **`package.json`**: Node.js project metadata and dependencies
- **`pnpm-lock.yaml`**: pnpm dependency lock file

### Environment Variables Structure

**Centralized Environment Management**:
- **MANDATORY**: All environment variables must be stored in the root `.env` file only
- **FORBIDDEN**: Environment variables in subdirectories (e.g., `frontend/.env`, `src/.env`)
- **MANDATORY**: Use a single `.env.example` template at the root level
- Both backend and frontend applications must read from the root `.env` file
- Docker Compose should mount the root `.env` file to all containers

**Template for `.env.example`**:
```bash
# API Configuration
API_KEY="your_api_key_here"
API_BASE_URL="https://api.example.com"

# Database Configuration
DATABASE_HOST="localhost"
DATABASE_USERNAME="admin"
DATABASE_PASSWORD="password"
DATABASE_NAME="project_db"

# Backend Application Settings
BACKEND_PORT="3000"
APP_ENV="development"
LOG_LEVEL="info"

# Frontend Configuration
FRONTEND_PORT="5173"
VITE_API_URL="http://localhost:3000"
VITE_APP_TITLE="My Application"
VITE_ENABLE_ANALYTICS="false"

# Docker Configuration
COMPOSE_PROJECT_NAME="project_name"
```

**Environment Variable Naming Conventions**:
- **Backend variables**: Use standard naming (e.g., `API_KEY`, `DATABASE_HOST`)
- **Frontend variables**: Prefix with `VITE_` for Vite projects (e.g., `VITE_API_URL`)
- **Docker variables**: Use `COMPOSE_` prefix for Docker Compose settings
- **Shared variables**: Can be used by both backend and frontend (e.g., `APP_ENV`)

### File Organization Rules

1. **Core Backend Logic**: Only in `/src/` directory
2. **Frontend Code**: Only in `/frontend/` directory
3. **Configuration**: Root level configuration files
4. **Data Separation**: Input, output, and logs in `/data/`
5. **Containerization**: All Docker files in `/docker/`
6. **Documentation**: Main README + supplementary in `/additionaldocs/`
7. **Utilities**: Optional tools in `/tools/`
8. **Automation**: Scripts in `/scripts/`

## Deployment Considerations

### Production Exclusions

When deploying, these directories can be excluded:
- `/tools/` (optional development utilities)
- `/tests/` (test suite)
- `/additionaldocs/` (supplementary documentation)
- `.git/` (version control)

### Required for Production

**Backend Projects**:
- `/src/` (core application)
- `/docker/` (containerization)
- Root configuration files

**Full Stack Projects**:
- `/src/` (backend application)
- `/frontend/` (frontend application)
- `/docker/` (containerization)
- `/data/` (runtime data directories)
- Root configuration files

**AI-Enhanced Projects**:
- `/prompts/` (AI templates)

## Benefits of This Structure

1. **Clear Separation**: Backend vs frontend vs utilities vs configuration
2. **Docker-Friendly**: Optimized for containerized deployment
3. **Maintainable**: Logical organization with clear responsibilities
4. **Scalable**: Easy to extend without structural changes
5. **Technology Agnostic**: Works with Python, Node.js, React, and other stacks
6. **Standard**: Follows industry best practices
7. **Clean Root**: Essential files only at project root
8. **Development-Ready**: Clear separation of development vs production code

This structure ensures consistency, maintainability, and deployment readiness across all development and production environments, regardless of the technology stack used.
