# Newtown Energy Development Environment

This repository contains files for orchestrating a local development environment using Docker Compose.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Directory Structure

The development environment requires the following directory structure with all repositories as siblings:

```
/parent-directory/
├── devenv/       (this repository - contains docker-compose.yml)
├── neems-core/   (backend services)
└── neems-react/  (frontend application)
```

### Repositories

- **/devenv** - [repository](https://github.com/Newtown-Energy/devenv) - Contains docker-compose.yml
- **/neems-core** - [repository](https://github.com/Newtown-Energy/neems-core) - Contains neems-data/Dockerfile and neems-api/Dockerfile
- **/neems-react** - [repository](https://github.com/Newtown-Energy/neems-react) - Contains Dockerfile

## Getting Started

1. Clone all three repositories as siblings in the same parent directory
2. Navigate to the devenv directory
3. Run `docker compose up` to start all services

The following services will be available:
- **neems-api**: http://localhost:8000
- **neems-react**: http://localhost:5173

## Default Credentials

For testing the application, use the following default admin credentials:
- **Email**: `superadmin@example.com`
- **Password**: `admin`

## Development Workflow

All services are configured with **live reload** for rapid development:

### Making Changes

- **neems-react**: Edit files in `../neems-react/src/` - Vite HMR will automatically update the browser
- **neems-api**: Edit files in `../neems-core/neems-api/` - cargo-watch will detect changes and rebuild/restart the API
- **neems-data**: Edit files in `../neems-core/neems-data/` - cargo-watch will detect changes and rebuild/restart the service

### How it Works

- Source code is mounted as volumes into the containers
- Build artifacts (Rust `target/`, Node `node_modules/`) are stored in named Docker volumes for faster rebuilds
- Changes to your local files are immediately reflected in the running containers

### First Build

The first time you run `docker compose up`, Rust services will take several minutes to:
1. Install cargo-watch
2. Download and compile dependencies
3. Build the application

Subsequent changes will be much faster as dependencies are cached.
