# Hello Module

> A complete, working example module for the PeerMesh Core module system. Clone it, customize it, deploy it.

## What Is This?

The Hello Module is a reference implementation that demonstrates every integration point in the Core module system. It serves a static "Hello from PeerMesh" greeting page via nginx, but the real value is in the structure: the annotated manifest, the lifecycle hooks, the Traefik routing, the health checks, and the dashboard widget.

Use this repository as a starting point when building your own Core modules. Copy the files, replace the placeholders (marked with `# CUSTOMIZE:` comments), and you have a working module.

**This module is expected to evolve rapidly** as the Core module system matures. Check the [CHANGELOG](CHANGELOG.md) for version history and the compatibility table below for supported Core versions.

## Quick Start

Five commands to get the Hello Module running in your Core:

```bash
# 1. Clone into your Core modules directory
cd /path/to/docker-lab/modules
git clone https://github.com/peermesh/hello-module.git hello-module

# 2. Configure
cd hello-module
cp .env.example .env
# Edit .env -- set DOMAIN to your actual domain
nano .env

# 3. Start
docker compose up -d

# 4. Verify
./hooks/health.sh

# 5. Visit
# https://hello-module.yourdomain.com/
```

### Important: Directory Placement

This module **must** be placed at `modules/hello-module/` inside your Core installation. The `module.json` `$schema` path and the `docker-compose.yml` `extends.file` path are both relative to this location:

- `$schema` resolves to `../../foundation/schemas/module.schema.json`
- `extends.file` resolves to `../../foundation/docker-compose.base.yml`

If you place the module elsewhere, these paths will break and compose will fail to start.

### Prerequisites

- Core foundation stack running (Traefik, socket-proxy)
- Docker Engine 24+ with Compose V2 plugin
- A domain with DNS pointing to your Core host

## File Structure

```
hello-module/
  module.json                    # Module manifest (all sections annotated)
  docker-compose.yml             # Nginx service extending _service-lite
  .env.example                   # Environment variable template
  secrets-required.txt           # Secret files list (shows pattern)
  README.md                      # This file
  CHANGELOG.md                   # Version history
  CONTRIBUTING.md                # Contribution guidelines
  LICENSE                        # MIT License
  hooks/                         # Lifecycle scripts
    install.sh                   # Validates config, checks dependencies
    start.sh                     # Starts compose, waits for health
    stop.sh                      # Graceful shutdown
    uninstall.sh                 # Cleanup (preserves data by default)
    health.sh                    # JSON/text health report
  dashboard/                     # Dashboard UI components
    HelloStatusWidget.html       # Status widget for dashboard
  html/                          # Static content served by nginx
    index.html                   # "Hello from PeerMesh" greeting page
  tests/                         # Module-level tests
    smoke-test.sh                # Automated verification
```

## Customization Guide

Every value you should change when creating your own module is marked with a `# CUSTOMIZE:` comment. Here is the complete list:

| File | Placeholder | What to Change |
|------|-------------|----------------|
| `module.json` | `id`, `name`, `description` | Your module's identity |
| `module.json` | `author` | Your name and contact |
| `module.json` | `repository` | Your module's repository URL |
| `module.json` | `tags` | Relevant tags for your module |
| `module.json` | `provides.events` | Events your module emits |
| `module.json` | `dashboard.displayName` | Name shown in dashboard |
| `module.json` | `config.properties` | Your module's configuration schema |
| `docker-compose.yml` | `image` | Your application's Docker image (pin the digest) |
| `docker-compose.yml` | `container_name` | Your module's container name |
| `docker-compose.yml` | `environment` | Your module's environment variables |
| `docker-compose.yml` | `volumes` | Your application's data volumes |
| `docker-compose.yml` | `healthcheck.test` | Health check command for your application |
| `docker-compose.yml` | Traefik labels | Your module's subdomain and routing rules |
| `docker-compose.yml` | `extends.service` | Resource profile (`_service-lite`, `_service-standard`, `_service-heavy`) |
| `.env.example` | `DOMAIN` | Your deployment domain |
| `.env.example` | `HELLO_MODULE_*` | Your module's configuration variables |
| `hooks/*.sh` | `MODULE_NAME` | Your module name |
| `hooks/*.sh` | `CONTAINER_NAME` | Your container name |
| `html/index.html` | Page content | Your application's UI |

## How Modules Work

A Core module integrates with the foundation through four mechanisms:

### 1. Module Manifest (`module.json`)

The manifest is the contract between your module and the foundation. It declares:

- **Identity**: module ID, version, description
- **Requirements**: database connections, dependent modules
- **Capabilities**: events emitted, services provided
- **Lifecycle hooks**: scripts for install, start, stop, health
- **Dashboard**: routes, widgets, icons
- **Configuration**: typed properties with environment variable mapping

### 2. Compose Integration (`docker-compose.yml`)

The compose file uses `extends` to inherit resource limits from foundation base patterns:

```yaml
services:
  my-service:
    extends:
      file: ../../foundation/docker-compose.base.yml
      service: _service-lite    # 256MB memory, 0.5 CPU
```

Available profiles: `_service-lite` (256MB), `_service-standard` (512MB), `_service-heavy` (1GB).

### 3. Traefik Routing

Traefik labels on your service define HTTPS routing:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-module.rule=Host(`my-module.${DOMAIN}`)"
  - "traefik.http.routers.my-module.entrypoints=websecure"
  - "traefik.http.routers.my-module.tls.certresolver=letsencrypt"
  - "traefik.http.services.my-module.loadbalancer.server.port=80"
```

Your service must join the `proxy-external` network so Traefik can reach it.

### 4. Lifecycle Hooks

Five hook scripts manage the module lifecycle:

| Hook | Purpose | Called When |
|------|---------|------------|
| `install.sh` | Validate config, create directories | First setup |
| `start.sh` | Start services, wait for healthy | Module activation |
| `stop.sh` | Graceful shutdown | Module deactivation |
| `uninstall.sh` | Clean up resources | Module removal |
| `health.sh` | Report health as JSON or text | Periodic checks |

**Important**: Lifecycle hooks are not auto-invoked by `launch_docker_lab_core.sh module enable`. Currently, `module enable` only runs `docker compose up -d`. Run hooks manually, or use them as documentation for future CLI integration.

## How to Add a Database

The `docker-compose.yml` contains a commented-out PostgreSQL example. To add a database:

1. Uncomment the `hello-module-db` service block
2. Uncomment the `db-internal` network
3. Uncomment the secrets section
4. Create the secret file:
   ```bash
   mkdir -p secrets
   openssl rand -base64 32 > secrets/hello_module_db_password
   chmod 600 secrets/hello_module_db_password
   ```
5. Add `DATABASE_URL` to your application's environment

## How to Add Dashboard Components

The `dashboard/HelloStatusWidget.html` demonstrates a minimal status widget. For more advanced examples:

- **Config panels**: See `modules/backup/dashboard/BackupConfigPanel.html`
- **Full pages**: See `modules/backup/dashboard/BackupPage.html`
- **Status widgets with charts**: See `modules/backup/dashboard/BackupStatusWidget.html`

Note: The current dashboard does not dynamically load widget HTML at runtime. The widget file documents the pattern for when that capability is implemented.

## How to Add Module Dependencies

Declare dependencies in `module.json`:

```json
{
  "requires": {
    "connections": [
      {
        "type": "database",
        "providers": ["postgres"],
        "required": true,
        "alias": "my-module-db"
      }
    ],
    "modules": ["pki"]
  }
}
```

The foundation resolves dependencies before starting your module.

## Testing Your Module

### Smoke Test

After starting the module, run the automated smoke test:

```bash
./tests/smoke-test.sh
```

This verifies:
- Container is running
- HTTP endpoint responds with 200
- Response contains expected content
- Health hook returns valid output
- Docker health check passes

### Health Check

Run the health hook directly:

```bash
# Human-readable output
./hooks/health.sh

# Machine-readable JSON (for dashboard integration)
./hooks/health.sh json
```

Expected JSON output:

```json
{
  "status": "healthy",
  "timestamp": "2026-02-26T12:00:00+00:00",
  "module": "hello-module",
  "checks": {
    "containerRunning": true,
    "http": "ok",
    "nginx": "ok"
  },
  "uptime": "2026-02-26T11:00:00Z",
  "messages": []
}
```

### Manual Verification

```bash
# Check container status
docker ps --filter name=hello-module

# Check logs
docker compose logs -f

# Test HTTP from inside the container
docker exec hello-module wget -qO- http://127.0.0.1/

# Validate compose configuration
docker compose config -q
```

## Production Readiness Checklist

Before deploying a module based on this example to production:

- [ ] **Image pinned**: Docker image uses a digest (`@sha256:...`), not just a tag
- [ ] **Secrets managed**: All credentials in file-based secrets, never in environment variables or compose files
- [ ] **Resource limits set**: Appropriate `_service-*` profile selected, or custom limits defined
- [ ] **Health check defined**: Both Docker health check (in compose) and lifecycle health hook
- [ ] **Security hardened**: `no-new-privileges:true` and `cap_drop: ALL` unless specific capabilities needed
- [ ] **Network isolated**: Only joins networks the module actually needs
- [ ] **Logging configured**: JSON file driver with rotation (inherited from `_service-*` profile)
- [ ] **Domain configured**: `.env` has correct `DOMAIN` value
- [ ] **Smoke test passes**: `./tests/smoke-test.sh` exits with code 0
- [ ] **Documentation updated**: README reflects your module's actual configuration

## Troubleshooting

### Module does not start

**Symptom**: `docker compose up -d` fails or container exits immediately.

**Check**:
```bash
docker compose logs
```

**Common causes**:
- Foundation stack not running (Traefik, socket-proxy)
- `pmdl_proxy-external` network does not exist
- Image pull failed (check network connectivity)

### Page not accessible via HTTPS

**Symptom**: `https://hello-module.yourdomain.com` returns 404 or connection refused.

**Check**:
```bash
# Verify Traefik sees the service
docker exec traefik wget -qO- http://localhost:8080/api/http/routers 2>/dev/null | grep hello-module
```

**Common causes**:
- `DOMAIN` not set in `.env`
- DNS not pointing to the Core host
- Container not on `proxy-external` network
- Traefik labels have a typo

### Health check fails

**Symptom**: `docker ps` shows container as `(unhealthy)`.

**Check**:
```bash
docker inspect --format='{{json .State.Health}}' hello-module | python3 -m json.tool
```

**Common causes**:
- nginx not serving on port 80 (custom config overrode default)
- `wget` not available in container (not using Alpine image)
- Health check `start_period` too short for slow startup

### Cannot reach the module locally

**Symptom**: `curl http://localhost` does not reach the module.

**Explanation**: The module does not expose ports to the host by default. Traffic goes through Traefik. To access directly:
```bash
# From inside the container
docker exec hello-module wget -qO- http://127.0.0.1/

# Or temporarily add port mapping
# In docker-compose.yml, add under the hello-module service:
#   ports:
#     - "8080:80"
```

## Version Compatibility

| Hello Module | Core | Foundation | Notes |
|--------------|------------|------------|-------|
| 1.0.0 | v7.39.0+ | 1.0.0+ | Initial release |

This module uses the `_service-lite` base pattern and the `module.json` schema from Core foundation. If the schema changes in a future Core version, a new major version of this module will be released.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on reporting issues and submitting improvements.

## License

MIT License. See [LICENSE](LICENSE) for details.
