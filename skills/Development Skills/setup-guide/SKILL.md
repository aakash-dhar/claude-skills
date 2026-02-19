---
name: setup-guide
description: Generates a complete local development setup guide by analyzing the project, checking prerequisites, running setup steps, and verifying everything works. Use when the user says "help me set up this project", "how do I run this locally?", "setup guide", "onboard me", "I just cloned this repo", "dev environment setup", or "getting started".
allowed-tools: Read, Grep, Glob, Bash
---

# Setup Guide Skill

When generating a setup guide, follow this structured process. Don't just read the README — actually verify each step works on the current system.

## 1. Detect the Project Stack

Scan the project to understand what needs to be installed:

### Language & Runtime
```bash
# Node.js
cat package.json 2>/dev/null | grep -E '"node"|"engines"'
cat .nvmrc .node-version 2>/dev/null

# Python
cat .python-version 2>/dev/null
cat pyproject.toml 2>/dev/null | grep -E "python_requires|requires-python"
cat runtime.txt 2>/dev/null
cat setup.py 2>/dev/null | grep "python_requires"

# Go
cat go.mod 2>/dev/null | head -3

# Ruby
cat .ruby-version 2>/dev/null
cat Gemfile 2>/dev/null | grep "ruby"

# Java
cat pom.xml 2>/dev/null | grep -E "<java.version>|<maven.compiler"
cat build.gradle 2>/dev/null | grep -E "sourceCompatibility|targetCompatibility"

# Rust
cat rust-toolchain.toml rust-toolchain 2>/dev/null

# PHP
cat composer.json 2>/dev/null | grep -E '"php"'
```

### Package Manager
```bash
# Node.js — detect package manager
ls package-lock.json 2>/dev/null && echo "npm"
ls yarn.lock 2>/dev/null && echo "yarn"
ls pnpm-lock.yaml 2>/dev/null && echo "pnpm"
ls bun.lockb 2>/dev/null && echo "bun"

# Python
ls requirements.txt Pipfile pyproject.toml poetry.lock 2>/dev/null

# Ruby
ls Gemfile.lock 2>/dev/null

# PHP
ls composer.lock 2>/dev/null
```

### Database & Services
```bash
# Docker services
cat docker-compose.yml docker-compose.yaml compose.yml compose.yaml 2>/dev/null | grep -E "image:|postgres|mysql|mongo|redis|elasticsearch|rabbitmq|kafka"

# Environment variables that hint at services
cat .env.example .env.sample .env.template 2>/dev/null | grep -E "DATABASE|REDIS|MONGO|ELASTIC|RABBIT|KAFKA|S3|SMTP"

# Prisma / ORM
cat prisma/schema.prisma 2>/dev/null | head -20
cat src/db/config* 2>/dev/null
```

### Infrastructure Tools
```bash
# Check for required CLI tools
ls Makefile 2>/dev/null && echo "make"
ls Dockerfile 2>/dev/null && echo "docker"
ls docker-compose.yml compose.yml 2>/dev/null && echo "docker-compose"
ls .terraform/ terraform/ 2>/dev/null && echo "terraform"
ls k8s/ kubernetes/ 2>/dev/null && echo "kubectl"
ls .github/workflows/ 2>/dev/null && echo "github-actions"
```

## 2. Check Current System

Verify what's already installed and what's missing:

### System Info
```bash
# OS
uname -a
cat /etc/os-release 2>/dev/null | head -5

# Shell
echo $SHELL
```

### Required Tools Check
```bash
# Common tools — check versions
node --version 2>/dev/null || echo "❌ Node.js not installed"
npm --version 2>/dev/null || echo "❌ npm not installed"
yarn --version 2>/dev/null || echo "❌ yarn not installed"
pnpm --version 2>/dev/null || echo "❌ pnpm not installed"
python3 --version 2>/dev/null || echo "❌ Python not installed"
pip3 --version 2>/dev/null || echo "❌ pip not installed"
go version 2>/dev/null || echo "❌ Go not installed"
ruby --version 2>/dev/null || echo "❌ Ruby not installed"
java --version 2>/dev/null || echo "❌ Java not installed"
rustc --version 2>/dev/null || echo "❌ Rust not installed"
php --version 2>/dev/null || echo "❌ PHP not installed"
docker --version 2>/dev/null || echo "❌ Docker not installed"
docker compose version 2>/dev/null || echo "❌ Docker Compose not installed"
git --version 2>/dev/null || echo "❌ Git not installed"
make --version 2>/dev/null | head -1 || echo "❌ make not installed"
```

## 3. Generate Step-by-Step Setup

Based on discovery, generate a complete setup guide:

### Guide Structure
```markdown
# [Project Name] — Local Development Setup

## Prerequisites

Everything you need before starting:

| Tool | Required Version | Your Version | Status |
|------|-----------------|-------------|--------|
| Node.js | >= 18.0.0 | v20.10.0 | ✅ |
| npm | >= 9.0.0 | 10.2.0 | ✅ |
| Docker | >= 24.0.0 | 24.0.7 | ✅ |
| PostgreSQL | 15.x | — | ⚠️ Via Docker |

### Install Missing Prerequisites

[Provide install commands for each missing tool, specific to their OS]
```

### Stack-Specific Setup Steps

#### Node.js / TypeScript Project
```markdown
## Step 1: Clone and Install

git clone <repo-url>
cd <project-name>

# Install dependencies (uses [detected package manager])
npm install          # or yarn / pnpm install

## Step 2: Environment Configuration

# Copy the example environment file
cp .env.example .env

# Required variables to fill in:
# DATABASE_URL    → Your local PostgreSQL connection string
# REDIS_URL       → Your local Redis connection string
# JWT_SECRET      → Any random string for local dev (e.g., "dev-secret-change-me")
# [list each required variable with explanation and example value]

## Step 3: Start Infrastructure

# Start database and other services
docker compose up -d

# Verify services are running
docker compose ps

## Step 4: Database Setup

# Run migrations
npm run db:migrate

# Seed development data (optional)
npm run db:seed

## Step 5: Start Development Server

npm run dev

# Server should be running at http://localhost:3000
```

#### Python / Django Project
```markdown
## Step 1: Clone and Set Up Virtual Environment

git clone <repo-url>
cd <project-name>

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
# or: poetry install / pipenv install

## Step 2: Environment Configuration

cp .env.example .env
# Fill in required variables...

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

python manage.py migrate
python manage.py createsuperuser  # Create admin account
python manage.py loaddata fixtures/dev_data.json  # Optional seed data

## Step 5: Start Development Server

python manage.py runserver

# Server should be running at http://localhost:8000
# Admin panel at http://localhost:8000/admin
```

#### Go Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

go mod download

## Step 2: Environment Configuration

cp .env.example .env
# Fill in required variables...

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

go run cmd/migrate/main.go up
# or: make migrate-up

## Step 5: Build and Run

go run cmd/server/main.go
# or: make run

# Server should be running at http://localhost:8080
```

#### Ruby on Rails Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

bundle install

## Step 2: Environment Configuration

cp .env.example .env
# Fill in required variables...

# Install JavaScript dependencies (if applicable)
yarn install

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

rails db:create
rails db:migrate
rails db:seed

## Step 5: Start Development Server

rails server
# or: bin/dev (if using Foreman/Procfile)

# Server should be running at http://localhost:3000
```

#### Java / Spring Boot Project
```markdown
## Step 1: Clone and Build

git clone <repo-url>
cd <project-name>

# Maven
./mvnw clean install -DskipTests

# Gradle
./gradlew build -x test

## Step 2: Environment Configuration

cp src/main/resources/application-local.yml.example src/main/resources/application-local.yml
# Fill in required properties...

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

# Flyway migrations run automatically on startup
# or: ./mvnw flyway:migrate

## Step 5: Start Development Server

# Maven
./mvnw spring-boot:run -Dspring.profiles.active=local

# Gradle
./gradlew bootRun --args='--spring.profiles.active=local'

# Server should be running at http://localhost:8080
# Swagger UI at http://localhost:8080/swagger-ui.html
```

#### PHP / Laravel Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

composer install
npm install

## Step 2: Environment Configuration

cp .env.example .env
php artisan key:generate

# Fill in required variables...

## Step 3: Start Infrastructure

docker compose up -d
# or use Laravel Sail: ./vendor/bin/sail up -d

## Step 4: Database Setup

php artisan migrate
php artisan db:seed

## Step 5: Start Development Server

php artisan serve
npm run dev  # Vite dev server for frontend assets

# Server should be running at http://localhost:8000
```

#### Rust Project
```markdown
## Step 1: Clone and Build

git clone <repo-url>
cd <project-name>

cargo build

## Step 2: Environment Configuration

cp .env.example .env
# Fill in required variables...

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

# Using sqlx
sqlx database create
sqlx migrate run

# Using diesel
diesel setup
diesel migration run

## Step 5: Run Development Server

cargo run
# or: cargo watch -x run (for auto-reload, requires cargo-watch)

# Server should be running at http://localhost:8080
```

#### Mobile (React Native) Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

npm install

# iOS specific
cd ios && pod install && cd ..

## Step 2: Environment Configuration

cp .env.example .env
# Fill in required variables...

## Step 3: Start Backend (if included)

docker compose up -d
# or connect to staging API

## Step 4: Run on Simulator

# iOS
npx react-native run-ios

# Android
npx react-native run-android

# Expo
npx expo start
```

#### Flutter Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

flutter pub get

## Step 2: Environment Configuration

cp lib/config/env.example.dart lib/config/env.dart
# Fill in required variables...

## Step 3: Run on Simulator

# List available devices
flutter devices

# Run on a device
flutter run -d <device-id>

# Run in Chrome (web)
flutter run -d chrome
```

#### Monorepo / Turborepo Project
```markdown
## Step 1: Clone and Install Dependencies

git clone <repo-url>
cd <project-name>

# Install all workspace dependencies
npm install  # or pnpm install / yarn install

## Step 2: Environment Configuration

# Root level
cp .env.example .env

# Per-package (check each app)
cp apps/web/.env.example apps/web/.env
cp apps/api/.env.example apps/api/.env
# Fill in required variables...

## Step 3: Start Infrastructure

docker compose up -d

## Step 4: Database Setup

npm run db:migrate --workspace=packages/database
npm run db:seed --workspace=packages/database

## Step 5: Start All Services

npm run dev  # Starts all workspaces via Turbo

# Web: http://localhost:3000
# API: http://localhost:4000
# Docs: http://localhost:3001
```

## 4. Verification Checklist

After setup, verify everything works:
```markdown
## Verify Your Setup

Run these checks to confirm everything is working:

### Infrastructure
- [ ] `docker compose ps` — all services show "Up"
- [ ] Database is accessible: `npm run db:check` or connect with your DB client
- [ ] Redis is accessible: `redis-cli ping` returns "PONG"

### Application
- [ ] Dev server starts without errors
- [ ] Homepage loads at http://localhost:[PORT]
- [ ] API health check returns OK: `curl http://localhost:[PORT]/api/health`
- [ ] Can log in with seed user credentials (if applicable)

### Development Tools
- [ ] Tests pass: `npm test`
- [ ] Linter passes: `npm run lint`
- [ ] Hot reload works: make a small change and verify it updates
- [ ] Debugger works: set a breakpoint and verify it pauses

### Common First-Run Issues
| Problem | Solution |
|---------|----------|
| Port already in use | Kill the process: `lsof -i :3000` then `kill -9 <PID>` |
| Database connection refused | Check Docker is running: `docker compose ps` |
| Module not found | Delete node_modules and reinstall: `rm -rf node_modules && npm install` |
| Permission denied | Check file permissions: `chmod +x scripts/*.sh` |
| Env variable missing | Compare your .env against .env.example |
| Migration failed | Reset DB: `npm run db:reset` or `docker compose down -v && docker compose up -d` |
| Python version mismatch | Use pyenv: `pyenv install 3.11 && pyenv local 3.11` |
| Node version mismatch | Use nvm: `nvm install && nvm use` |
| Docker out of space | Clean up: `docker system prune -a` |
| Seed data missing | Run seed command: check package.json for seed script |
```

## 5. IDE & Editor Setup
```markdown
## Recommended Editor Setup

### VS Code
Install these extensions for the best experience:
- [List extensions detected from .vscode/extensions.json or project type]

Recommended settings (`.vscode/settings.json`):
- Format on save: enabled
- Default formatter: [Prettier/Black/gofmt/rustfmt based on project]
- ESLint/Pylint auto-fix on save

### JetBrains (IntelliJ / WebStorm / PyCharm)
- Import code style from `.editorconfig` or project settings
- Enable [relevant plugins]
- Configure run configurations for dev server and tests

### Vim / Neovim
- LSP config for [project language]
- Formatter integration
- Test runner keybinds
```

## 6. Useful Commands Reference
```markdown
## Daily Commands

| Task | Command |
|------|---------|
| Start dev server | `npm run dev` |
| Run tests | `npm test` |
| Run single test | `npm test -- --grep "test name"` |
| Lint code | `npm run lint` |
| Format code | `npm run format` |
| Start infrastructure | `docker compose up -d` |
| Stop infrastructure | `docker compose down` |
| View logs | `docker compose logs -f [service]` |
| Create migration | `npm run db:migrate:create -- --name=add_users_table` |
| Run migrations | `npm run db:migrate` |
| Reset database | `npm run db:reset` |
| Open DB shell | `docker compose exec db psql -U postgres` |
| Open Redis CLI | `docker compose exec redis redis-cli` |
| Build for production | `npm run build` |
| Check bundle size | `npm run analyze` |
```

## 7. Access & Credentials
```markdown
## Development Accounts & Access

### Local Development
| Service | URL | Credentials |
|---------|-----|-------------|
| App | http://localhost:3000 | — |
| API | http://localhost:3000/api | — |
| DB Admin (pgAdmin/Adminer) | http://localhost:5050 | admin@local.dev / admin |
| Redis Commander | http://localhost:8081 | — |
| Mailhog (email testing) | http://localhost:8025 | — |
| Swagger/API Docs | http://localhost:3000/api/docs | — |

### Seed User Accounts (if applicable)
| Role | Email | Password |
|------|-------|----------|
| Admin | admin@example.com | password123 |
| User | user@example.com | password123 |

### External Service Credentials
Obtain these from your team lead or the shared vault:
- [ ] Stripe test API key
- [ ] AWS credentials (for S3 local testing)
- [ ] OAuth client ID/secret (Google, GitHub, etc.)

Note: Never use production credentials locally.
```

## 8. Next Steps After Setup
```markdown
## You're Set Up! Now What?

### Get Familiar
1. **Read** `docs/architecture.md` or run `/explain-codebase` for an overview
2. **Explore** the codebase starting from `src/api/routes/` (follow a request through the system)
3. **Run** the test suite to see how the code is tested
4. **Check** the project board for "good first issue" tickets

### Your First Task
Pick a small, well-defined task to build familiarity:
- Fix a typo in the UI
- Add a missing unit test
- Update an outdated dependency
- Improve an error message

### Key People
| Area | Contact |
|------|---------|
| Architecture questions | @[tech-lead] |
| Deployment issues | @[devops-lead] |
| Product questions | @[product-manager] |
| General help | #[team-channel] on Slack |
```

## Output Format

Structure the setup guide as:

1. **Prerequisites table** — tools needed with version check results
2. **Install missing tools** — commands for anything not found
3. **Step-by-step setup** — numbered steps from clone to running
4. **Verification checklist** — confirm everything works
5. **Troubleshooting** — common issues and fixes
6. **Useful commands** — daily reference table
7. **Next steps** — what to do after setup is complete

## Adaptation Rules

- **Auto-detect everything** — don't ask the user what stack they're using, scan the project
- **Verify each step** — actually run commands to check if things work
- **OS-aware** — detect the user's OS and provide correct commands (macOS/Linux/Windows)
- **Skip what's done** — if a tool is already installed and correct version, say ✅ and move on
- **Flag blockers** — if something critical is missing (Docker not installed, wrong Node version), highlight it immediately before continuing
- **Include timings** — estimate how long the full setup takes ("~15 minutes on a fast connection")

## Summary

End every setup guide with:
1. **Setup status** — ✅ Ready / ⚠️ Needs attention / ❌ Blocked
2. **What's working** — services confirmed running
3. **What needs manual action** — credentials to obtain, accounts to create
4. **Estimated time** — how long the remaining setup should take
5. **First command to run** — the one command to start developing
