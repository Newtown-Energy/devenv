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
