# 🍸 DockTail

**Unleash your Containers as Tailscale Services**

<p align="center">
  <img src="assets/header.jpeg" alt="DockTail Header" width="100%">
</p>

Automatically expose Docker containers as Tailscale Services using label-based configuration - zero-config service mesh for your dockerized services.

```
 ┌────────────────────────────────────────────────────────┐
 │                     Docker Host                        │
 │                                                        │
 │  ┌──────────────────┐         ┌──────────────────┐     │
 │  │     DockTail     │────────▶│ Tailscale Daemon │     │
 │  │   (Container)    │  CLI    │   (Host Process) │     │
 │  └────────┬─────────┘         └────────┬─────────┘     │
 │           │                            │               │
 │           │ Docker Socket              │ Proxies to    │
 │           │ Monitoring                 │ localhost     │
 │           ▼                            ▼               │
 │  ┌──────────────────┐         ┌──────────────────┐     │
 │  │   App Container  │◀────────│  localhost:9080  │     │
 │  │   Port 80        │  Mapped │  localhost:9081  │     │
 │  │  ports: 9080:80  │◀────────│                  │     │
 │  └──────────────────┘         └──────────────────┘     │
 │                                                        │
 └────────────────────────────────────────────────────────┘
                          │
                          │ Tailscale Network
                          ▼
               ┌─────────────────────┐
               │  Tailnet Clients    │
               │  Access services:   │
               │  web.tailnet.ts.net │
               └─────────────────────┘
```

## Features

- [x] Automatically discover and advertise Docker containers as Tailscale Services
- [x] HTTP, HTTPS and TCP protocols for running services
- [x] Support Tailscale HTTPS (auto TLS certificate)
- [x] Automatically drain Tailscale service configurations on container stop
- [x] Runs entirely in a **stateless Docker container**
- [x] Tailscale Funnel support (public internet access)
- [ ] More? => Create an Issue :)

> [!WARNING]
> This project is still being developed and it is **not** yet recommended to use for mission critical services.

## Quick Start

### Admin Console Setup

Before installing the DockTail, configure your Tailscale admin console at https://login.tailscale.com/admin/services:

1. **Create service definitions** (Services → Add service):
   - Create a service for each application you want to expose
   - Example: Service name `web`, `api`, `db`, etc.
   - Note: DockTail will automatically configure and advertise these services

2. **(Optional) Configure service tags**:
   - Navigate to Access Controls
   - Add tags for service identification (e.g., `tag:homelab-service`)
   - Tag your Docker host (e.g., `tag:homelab`). **Important:** Ensure the machine running DockTail is authenticated with this tag (e.g., `tailscale up --advertise-tags=tag:homelab`).

3. **(Recommended) Enable auto-approval**:
   - Navigate to Access Controls and edit your ACL policy
   - Add auto-approvers to skip manual approval for service advertisements.
   - **Why?** Without this, every new service DockTail advertises will require manual approval in the Admin Console.
   ```json
   {
     "autoApprovers": {
       "services": {
         // Allow devices with tag:homelab to advertise services with tag:homelab-service
         // e.g. "container tag (applied with label)" <- ["approver tag (host machine)"]
         "tag:homelab-service": ["tag:homelab"]
       }
     }
   }
   ```
   - **Note:** DockTail applies `DEFAULT_SERVICE_TAGS` (default: `tag:container`) to all services unless overridden by `docktail.tags`. Ensure your ACLs match these tags.

See [Tailscale Services documentation](https://tailscale.com/kb/1552/tailscale-services) for detailed setup instructions.

### Installation

#### Option 1: Docker Compose

Create a `docker-compose.yaml`:

```yaml
version: '3.8'

services:
  docktail:
    image: ghcr.io/marvinvr/docktail:latest
    container_name: docktail
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
```

#### Option 2: Docker Run

```bash
docker run -d \
  --name docktail \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock \
  ghcr.io/marvinvr/docktail:latest
```

#### Option 3: Docker Compose with Containerized Tailscale (Sidecar)

For systems without Tailscale installed directly on the host (e.g., macOS, NAS devices), use the provided sidecar compose file:

```bash
# 1. Configure your Tailscale auth key
cp .env.example .env
# Edit .env and set TAILSCALE_AUTH_KEY

# 2. Start DockTail with containerized Tailscale
docker compose -f docker-compose.sidecar.yaml up -d
```

See `docker-compose.sidecar.yaml` for the full configuration.

**Key points:**
- Tailscale runs in a container with `network_mode: host` to access container ports on localhost
- A shared volume connects DockTail to the Tailscale socket
- State is persisted so Tailscale doesn't need to re-authenticate on restart

### Usage

**🚨 CRITICAL:** Container ports MUST be published to host. Tailscale serve only supports `localhost` proxies.

**Exception:** Containers using `network_mode: host` do NOT need port publishing since they directly use the host's network. DockTail will automatically detect host networking and skip port binding validation.

**Basic example:**
```yaml
services:
  myapp:
    image: nginx:latest
    ports:
      - "8080:80"  # REQUIRED! HOST:CONTAINER format
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=myapp"
      - "docktail.service.port=80"  # CONTAINER port (RIGHT side of "8080:80")
```

Access from any device in your tailnet:
```bash
curl http://myapp.your-tailnet.ts.net
```

**With HTTPS (auto TLS cert from Tailscale):**
```yaml
services:
  myapp:
    image: nginx:latest
    ports:
      - "8080:80"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=myapp"
      - "docktail.service.port=80"                   # Container port
      - "docktail.service.protocol=http"             # Container speaks HTTP
      - "docktail.service.service-port=443"          # Tailscale listens on 443
      - "docktail.service.service-protocol=https"    # Tailscale serves HTTPS (auto TLS!)
```

**Smart defaults (minimal config):**
```yaml
services:
  myapp:
    image: nginx:latest
    ports:
      - "8080:80"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=myapp"
      - "docktail.service.port=80"
      - "docktail.service.service-port=443"  # Port 443 → auto-defaults to HTTPS!
      # service-protocol auto-defaults to "https" (based on port 443)
      # protocol auto-defaults to "http" (TLS termination at Tailscale)
```

**Port Mapping Rules:**
- `ports:` = `"HOST:CONTAINER"` (e.g., `"8080:80"` = host 8080 → container 80)
- `docktail.service.port` = CONTAINER port (always the RIGHT side)
- Result: Tailscale → localhost:8080 → Container:80

**Understanding the 3 Ports:**
1. **Container Port** (`docktail.service.port`): The port your app listens on inside the container (e.g., 80).
2. **Published Host Port** (Docker `ports`): The port Docker maps to the host (e.g., 8080). DockTail proxies traffic to `localhost:8080`.
3. **Tailscale Service Port** (`docktail.service.service-port`): The port clients use to connect to your service (e.g., 443). This is what gets registered in the Tailscale Control Plane.

### Available Labels

#### Service Labels (Tailnet-only Access)

See [Tailscale Service documentation](https://tailscale.com/kb/1552/tailscale-services) for detailed setup instructions.

| Label                               | Required | Default                | Description                                                                               |
| ----------------------------------- | -------- | ---------------------- | ----------------------------------------------------------------------------------------- |
| `docktail.service.enable`           | Yes      | -                      | Enable DockTail for container                                                             |
| `docktail.service.name`             | Yes      | -                      | Service name (e.g., `web`, `api-v2`)                                                      |
| `docktail.service.port`             | Yes      | -                      | **CONTAINER** port (RIGHT side of `ports:`)                                               |
| `docktail.service.protocol`         | No       | Smart***               | Protocol container speaks: `http`, `https`, `https+insecure`, `tcp`, `tls-terminated-tcp` |
| `docktail.service.service-port`     | No       | Smart*                 | Port Tailscale listens on                                                                 |
| `docktail.service.service-protocol` | No       | Smart**                | Protocol Tailscale uses: `http`, `https`, `tcp`                                           |
| `docktail.tags`                     | No       | `DEFAULT_SERVICE_TAGS` | Comma-separated list of tags (e.g., `tag:web,tag:prod`)                                   |

**Smart Defaults:**
- *`service-port`: Defaults to `80`, OR `443` if `service-protocol=https`
- **`service-protocol`: Defaults to match `protocol` for TCP backends, otherwise `https` if `service-port=443`, otherwise `http`
- ***`protocol`: Defaults to `https` if container `port=443`, otherwise `http`

**Critical:** If `ports: "9080:80"`, then `docktail.service.port=80` (container port, NOT 9080)

#### Funnel Labels (Public Internet Access)

See [Tailscale Funnel documentation](https://tailscale.com/kb/1311/tailscale-funnel) for detailed setup instructions.

**Independent from serve labels**

| Label                         | Required         | Default | Description                                             |
| ----------------------------- | ---------------- | ------- | ------------------------------------------------------- |
| `docktail.funnel.enable`      | Yes (for funnel) | `false` | Enable Tailscale Funnel (public internet access)        |
| `docktail.funnel.port`        | Yes (for funnel) | -       | **CONTAINER** port (same concept as `service.port`)     |
| `docktail.funnel.funnel-port` | No               | `443`   | **PUBLIC** port (must be 443, 8443, or 10000 for HTTPS) |
| `docktail.funnel.protocol`    | No               | `https` | Protocol: `https`, `tcp`, `tls-terminated-tcp`          |

**Notes about Funnel:**
- Funnel is independent of serve - different labels, different ports, different everything
- Can run funnel alone, serve alone, or both side-by-side on the same container
- Funnel uses the **machine's hostname**, NOT service names (unlike serve)
- Public URL format: `https://<machine-hostname>.<tailnet>.ts.net:<funnel-port>`
- **⚠️ IMPORTANT**: Only ONE funnel can be active per `funnel-port` (Tailscale limitation). Multiple containers cannot share the same `funnel-port`.
- When used together with serve:
  - Serve URL: `https://<service-name>.<tailnet>.ts.net` (tailnet only, uses `service.port`)
  - Funnel URL: `https://<machine-hostname>.<tailnet>.ts.net:<funnel-port>` (public, uses `funnel.port`)
- `funnel-port` (public) can be: **443**, **8443**, or **10000** for HTTPS
- Funnel exposes your service to the **⚠️ public internet** - use with caution!

### Supported Protocols

**Service Protocols (Tailscale-facing):**
- `http`: Layer 7 HTTP forwarding
- `https`: Layer 7 HTTPS forwarding (auto TLS cert)
- `tcp`: Layer 4 TCP forwarding
- `tls-terminated-tcp`: Layer 4 with TLS termination

**Backend Protocols (Container-facing):**
- `http`: HTTP backend
- `https`: HTTPS backend with valid certificate
- `https+insecure`: HTTPS backend with self-signed/invalid certificate
- `tcp`: TCP backend
- `tls-terminated-tcp`: TCP with TLS termination

### Environment Variables

| Variable               | Default                              | Description                                                          |
| ---------------------- | ------------------------------------ | -------------------------------------------------------------------- |
| `LOG_LEVEL`            | `info`                               | Logging level (debug, info, warn, error)                             |
| `RECONCILE_INTERVAL`   | `60s`                                | State reconciliation interval                                        |
| `DOCKER_HOST`          | `unix:///var/run/docker.sock`        | Docker daemon socket                                                 |
| `TAILSCALE_SOCKET`     | `/var/run/tailscale/tailscaled.sock` | Tailscale daemon socket                                              |
| `TAILSCALE_API_KEY`    | `""` (Empty)                         | Tailscale API Key (`tskey-api-...`). If empty, API sync is disabled. |
| `TAILSCALE_TAILNET`    | `-`                                  | Tailnet ID. Defaults to `-` (the tailnet associated with the key).   |
| `DEFAULT_SERVICE_TAGS` | `tag:container`                      | Comma-separated list of default tags for services.                   |

## How It Works

1. **Container Discovery**: Monitors Docker events API for container lifecycle events (start, stop, die, restart)
2. **Label Parsing**: Extracts Tailscale service configuration from container labels
3. **Port Detection**: Queries Docker API for published host ports
4. **Configuration Generation**: Creates Tailscale service configuration proxying to `localhost:HOST_PORT`
5. **Configuration Application**: Executes Tailscale CLI commands to apply config and advertise services
6. **Stateless Operation**: Periodically reconciles state by querying Docker and Tailscale APIs
7. **Control Plane Sync (Optional)**: If `TAILSCALE_API_KEY` is provided, DockTail syncs service definitions to the Tailscale Control Plane (API). This ensures services exist globally with correct tags for ACLs.
   - **Conservative Deletion**: DockTail does **NOT** delete service definitions from the API when containers stop, preventing accidental outages in HA setups.
   - **Creation Flow**: Attempts to create new services via API. Note: If your tailnet requires pre-provisioned addresses, creation may fail, requiring manual initial creation in the Admin Console.


## Examples

### Web Application (Simple)

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=web"
      - "docktail.service.port=80"
```

### Database (Custom Port & Protocol)

```yaml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: secret
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=db"
      - "docktail.service.port=5432"
      - "docktail.service.protocol=tcp"
      - "docktail.service.service-port=5432"
      # service-protocol auto-defaults to "tcp" (matches backend protocol)
```

### API with HTTPS (Auto TLS Certificate)

```yaml
services:
  api:
    image: myapi:latest
    ports:
      - "8080:3000"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=api"
      - "docktail.service.port=3000"            # Container listens on 3000
      - "docktail.service.service-port=443"     # Tailscale listens on 443 (HTTPS)
      # service-protocol auto-defaults to "https" (based on service-port=443)
      # protocol auto-defaults to "http" (based on container port=3000, not 443)
```

Access with automatic TLS:
```bash
curl https://api.your-tailnet.ts.net  # TLS cert auto-provisioned!
```

### Container with Host Networking

```yaml
services:
  app:
    image: myapp:latest
    network_mode: host
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=app"
      - "docktail.service.port=3093"      # Port app listens on
      # No port mapping needed! Container uses host network directly
```

Access from your tailnet:
```bash
curl http://app.your-tailnet.ts.net
```

**Note:** With `network_mode: host`, the container bypasses Docker's network isolation and uses the host's network directly. No port publishing is required since the container port is already accessible on `localhost`.

### Public Website with Funnel (Internet Access)

```yaml
services:
  website:
    image: nginx:latest
    ports:
      - "8080:80"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=website"
      - "docktail.service.port=80"
      - "docktail.service.service-port=443"      # Serve HTTPS on tailnet
      - "docktail.funnel.enable=true"            # Enable public internet access
      - "docktail.funnel.port=80"                # Container port for funnel
      # funnel.protocol defaults to "https" and funnel.funnel-port defaults to "443"
```

Access from your tailnet and the public internet:
```bash
# Tailnet-only access (via serve, uses service name):
curl https://website.your-tailnet.ts.net

# Public internet access (via funnel, uses machine hostname):
curl https://your-machine-name.your-tailnet.ts.net
```

**Security Note:** Funnel exposes your service to the **public internet**. Ensure proper authentication and security measures are in place!

### Funnel with Custom Public Port

```yaml
services:
  app:
    image: myapp:latest
    ports:
      - "8080:3000"
    labels:
      - "docktail.service.enable=true"
      - "docktail.service.name=app"
      - "docktail.service.port=3000"
      - "docktail.service.service-port=443"       # Tailnet HTTPS
      - "docktail.funnel.enable=true"             # Enable funnel
      - "docktail.funnel.port=3000"               # Container port for funnel
      - "docktail.funnel.funnel-port=8443"        # Public port (443, 8443, or 10000)
      - "docktail.funnel.protocol=https"          # Funnel protocol
```

Access via custom public port:
```bash
# Tailnet (serve):
curl https://app.your-tailnet.ts.net

# Public internet (funnel):
curl https://your-machine-name.your-tailnet.ts.net:8443
```

## Testing Control Plane Sync

To manually verify the Control Plane Sync feature:

### Prerequisites

1. **Tailscale API Key:** Generate at [Tailscale Admin Console > Settings > Keys](https://login.tailscale.com/admin/settings/keys)
2. **Docker & Tailscale:** Ensure Docker is running and Tailscale is installed/authenticated on your host

### Setup & Run

1. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env and add your TAILSCALE_API_KEY
   ```

2. **Start the test stack:**
   ```bash
   docker compose -f docker-compose.console-sync.yaml up --build
   ```

3. **Verify in logs:** Watch for:
   - `Configuration loaded` with `api_sync_enabled=true`
   - `Syncing service definitions to Control Plane`
   - `Successfully synced service definition to Control Plane`

4. **Verify in Tailscale Console:**
   - Go to [Tailscale Admin Console > Services](https://login.tailscale.com/admin/services)
   - Look for the test service with configured tags

5. **Cleanup:**
   ```bash
   docker compose -f docker-compose.console-sync.yaml down
   ```

**Note:** DockTail uses a **Conservative Deletion Strategy** - stopping containers will NOT remove service definitions from the Tailscale Admin Console. You must manually delete them if needed.

## Building from Source

```bash
# Build binary
go build -o docktail .

# Build Docker image
docker build -t docktail:latest .

# Run locally
./docktail
```

## Links

- [Tailscale Services Documentation](https://tailscale.com/kb/1552/tailscale-services)
- [Tailscale Funnel Documentation](https://tailscale.com/kb/1311/tailscale-funnel)
- [Tailscale Service Configuration Reference](https://tailscale.com/kb/1589/tailscale-services-configuration-file)
- [Docker SDK for Go](https://docs.docker.com/engine/api/sdk/)

## License

AGPL v3

----
By [@marvinvr](https://marvinvr.ch)
