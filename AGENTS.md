# AI Agent Development Guidelines

This document provides critical guidelines for AI agents (like Claude Code) working on this project.

## Docker-First Development

**CRITICAL RULE: ALL development commands MUST be executed inside the appropriate Docker container.**

### Why Docker-First?

This project uses Docker containers to ensure:
- Consistent development environments across all contributors (human and AI)
- Proper dependency management and isolation
- Correct versions of all tools (Rust, Node.js, etc.)
- Reproducible builds and tests

### Container Architecture

This project uses the following containers:

1. **neems-api** (Rust backend)
   - Location: `neems-core/neems-api/`
   - Purpose: API server, data layer, TypeScript generation
   - Access: `docker compose exec neems-api bash`

2. **neems-react** (React frontend)
   - Location: `neems-react/`
   - Purpose: Frontend application
   - Access: `docker compose exec neems-react bash`

3. **neems-db** (SQLite database)
   - Location: `neems-core/neems-api/`
   - Purpose: Database server
   - Access: `docker compose exec neems-db bash`

### Correct Development Workflow

#### WRONG - Running commands on host machine:
```bash
# ❌ NEVER DO THIS
cd neems-core
cargo test generate_typescript_types
cargo build
npm install
```

#### RIGHT - Running commands inside containers:
```bash
# ✅ ALWAYS DO THIS
docker compose exec neems-api cargo test generate_typescript_types
docker compose exec neems-api cargo build
docker compose exec neems-react npm install
```

### Common Tasks and Correct Commands

#### TypeScript Generation from Rust

**IMPORTANT: TypeScript types are automatically generated!**

The neems-api container automatically watches for Rust file changes and regenerates TypeScript types in real-time. You typically don't need to run generation manually.

The auto-generation process:
- Watches `neems-api/src` and `neems-data/src` directories
- Automatically runs when any Rust file changes
- Outputs to `neems-react/src/types/generated/` (mounted via Docker)
- Runs in the background alongside the API server

```bash
# Manual generation (only needed for testing/debugging)
docker compose exec neems-api cargo test --features test-staging generate_typescript_types -- --nocapture
```

To verify auto-generation is working:
```bash
# Check container logs for TypeScript generation activity
docker compose logs neems-api | grep "generate"
```

#### Rust Development
```bash
# Run tests
docker compose exec neems-api cargo test

# Build the API
docker compose exec neems-api cargo build

# Check code
docker compose exec neems-api cargo check

# Format code
docker compose exec neems-api cargo fmt
```

#### React Development
```bash
# Install dependencies
docker compose exec neems-react npm install

# Run tests
docker compose exec neems-react npm test

# Build
docker compose exec neems-react npm run build

# Generate types (wrapper script)
docker compose exec neems-react npm run generate-types
```

#### Database Operations
```bash
# Access database
docker compose exec neems-db sqlite3 /data/neems.db

# Run migrations
docker compose exec neems-api diesel migration run
```

### When to Start/Stop Containers

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f neems-api
docker compose logs -f neems-react

# Rebuild containers after Dockerfile changes
docker compose up -d --build
```

### Exception: Host Machine Commands

The ONLY commands that should run on the host machine are:
- `docker compose` commands for container orchestration
- Git operations (`git add`, `git commit`, `git push`, etc.)
- File operations using AI tools (Read, Write, Edit)
- Text search and file globbing (Grep, Glob)

### Debugging Container Issues

If containers aren't running:
```bash
# Check container status
docker compose ps

# Start containers
docker compose up -d

# Check logs for errors
docker compose logs neems-api
docker compose logs neems-react
```

If you need to rebuild:
```bash
# Rebuild specific service
docker compose up -d --build neems-api

# Rebuild all services
docker compose up -d --build
```

### File Permissions

Files created inside containers may have different ownership. If you encounter permission issues:
```bash
# Fix ownership (run on host)
sudo chown -R $USER:$USER neems-core/
sudo chown -R $USER:$USER neems-react/
```

## TypeScript Generation System

The project uses `ts-rs` to automatically generate TypeScript definitions from Rust types.

### How It Works

1. Rust structs are annotated with `#[derive(TS)]` and `#[ts(export)]`
2. The generation script (`neems-core/neems-api/src/generate_types.rs`) exports all types
3. TypeScript files are written to `neems-react/src/types/generated/`

### Running Type Generation

**Always run inside the neems-api container:**
```bash
docker compose exec neems-api cargo test generate_typescript_types -- --nocapture
```

### Auto-Generated Files

The following TypeScript files are auto-generated and should NEVER be manually edited:
- `neems-react/src/types/generated/*.ts`

Each file includes a header:
```typescript
// This file was generated by [ts-rs](https://github.com/Aleph-Alpha/ts-rs). Do not edit this file manually.
```

### When to Regenerate Types

Regenerate TypeScript types after:
- Adding/modifying Rust structs with `#[ts(export)]`
- Changing API request/response structures
- Updating database models
- Modifying any Rust type that's exported to TypeScript

## General Best Practices

1. **Always check container status** before running commands
2. **Use `docker compose exec`** for all build/test/development commands
3. **Read docker-compose.yml** to understand service dependencies
4. **Check Dockerfiles** to understand what tools are available in each container
5. **Never assume** tools are installed on the host machine
6. **When in doubt**, ask the user which container to use

## Quick Reference

| Task | Container | Command |
|------|-----------|---------|
| Generate TypeScript types | neems-api | `docker compose exec neems-api cargo test generate_typescript_types` |
| Run Rust tests | neems-api | `docker compose exec neems-api cargo test` |
| Build Rust API | neems-api | `docker compose exec neems-api cargo build` |
| Install npm packages | neems-react | `docker compose exec neems-react npm install` |
| Run React dev server | neems-react | `docker compose exec neems-react npm start` |
| Access database | neems-db | `docker compose exec neems-db sqlite3 /data/neems.db` |
| View API logs | - | `docker compose logs -f neems-api` |
| Restart services | - | `docker compose restart` |

---

**Remember: When working on this project, think "Docker first" for all development operations.**
