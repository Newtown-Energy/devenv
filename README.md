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

### Choosing the Site Design

The stack runs neems-core's default site design. To run another, set
`NEEMS_SITE_DESIGN` when bringing it up:

```bash
NEEMS_SITE_DESIGN=site-2 docker compose up -d
```

It reaches neems-api, neems-data and neems-rtac-sim together, which must agree.
Changing it takes this `up -d`, which recreates the containers; a `restart`
reuses the old environment. Bring the stack up without it to go back to the
default.

### Running Against a Real RTAC

The stack follows the simulated RTAC (`neems-rtac-sim`) by default. It can
follow a real controller on a test bench instead. The controller is not
reachable directly: the path is the tinc VPN, then an SSH tunnel through a
machine that sits on the controller's network.

```
neems-data container -> host.docker.internal:5020 -> SSH tunnel -> RTAC
                                                     (over the VPN)
```

**The collector is closed-loop.** Pointed at a real RTAC it writes commands to
it as well as reading it. Only do this against a controller that is not
operating a live site.

#### One-time setup

1. Install tinc and get this machine onto the VPN. It needs its own node name,
   address and key pair in tinc's config directory, the host files of the
   nodes it connects to, and its own host file handed to whoever administers
   those nodes.
2. Get an SSH login on the machine that sits on the RTAC's network.
3. Copy `env.example` to `.env` and fill it in. `.env` is git-ignored; host
   names, addresses and key paths stay out of this repository.

#### Each session

```bash
./bin/dosh vpn          # start tinc; leave it running in its own terminal
./bin/dosh rtac-real    # open the tunnel and point neems-data at the RTAC
./bin/dosh rtac-status  # which RTAC neems-data follows, and whether it answers
./bin/dosh rtac-sim     # back to the simulator, and close the tunnel
```

`rtac-real` and `rtac-sim` recreate only `neems-data`, and keep the site design
the stack is running. A plain `docker compose up -d` also
puts it back on the simulator, since the override lasts for that one command.
`rtac-tunnel` and `rtac-tunnel-down` open and close the tunnel on its own.

#### When it doesn't connect

- **`vpn` says it couldn't write a pid file.** Homebrew's `tincd` defaults to a
  directory that doesn't exist. `vpn` passes `--pidfile` to avoid this; do the
  same if you start `tincd` by hand.
- **The tunnel won't open.** Check the VPN is up by pinging another node, then
  that `ssh` to `REAL_RTAC_SSH` works on its own.
- **`rtac-status` reports no Modbus answer.** The tunnel is fine and nothing
  answered at the far end. Check `REAL_RTAC_TARGET`, and that the RTAC is
  configured as a Modbus *server*: neems-data is the client and connects to
  it. An RTAC set up to poll NEEMS as a client will never answer.
- **neems-data connects but the readings are nonsense.** Something else is at
  that address, or `REAL_RTAC_SLAVE_ID` or the register map doesn't match.
- **On Linux the container can't reach the tunnel.** Containers there can't
  see the host's loopback. Set `REAL_RTAC_BIND` as `env.example` describes.

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
