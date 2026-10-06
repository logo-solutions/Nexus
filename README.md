# Nexus — Private Container Registry & Artifact Repository

![Version](https://img.shields.io/badge/Nexus-3.96.3-blue)
![License](https://img.shields.io/badge/License-Community-green)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

Sonatype Nexus 3 self-hosted instance for the ecosystem: private container registry (Docker, OCI) and artifact repository for build pipelines.

**Architecture**: Nexus stores all build artifacts and container images for the factice CI/CD pipeline. Serves as the single source of truth for container digests and SBOM manifests.

## Features

- ✅ **Private Container Registry** — push/pull Docker images without public exposure
- ✅ **Docker Hosted Repository** — native Docker v2 protocol support (port 8082)
- ✅ **Artifact Storage** — raw-candidat, raw-release for SBOM + manifests
- ✅ **Repository Provisioning** — API-driven repo creation (no UI clicking)
- ✅ **Immutable Releases** — prevent overwrite once promoted

## Quick Start

```bash
# Prerequisites
docker compose --version   # v2.x required
docker --version          # 24.0+

# Deploy
cd /Volumes/logousb/SSD/Projects/Nexus
docker compose up -d

# Verify
curl -u admin:admin123 http://localhost:8081/service/metrics/healthcheck
curl http://localhost:8082/v2/                    # docker registry endpoint (401 auth required)

# UI access
# http://localhost:8081 (user: admin, pass: admin123)
```

## Ports

| Port | Service | Purpose |
|------|---------|---------|
| **8081** | UI + API | Web interface, REST API |
| **8082** | Docker Registry (HTTP) | Push/pull container images |
| **5001–5004** | Docker Repositories | Reserved for factice repos (candidat, release, proxy, all-in-one) |

**Note**: Ports 5001–5004 are blocked by Colima SSH tunnel. Use port 8082 for Docker operations.

## Installation & Configuration

### Prerequisites

- **Docker Engine** 24.0+
- **Docker Compose** 2.x
- **Disk Space** — minimum 20 GB for artifact storage
- **RAM** — minimum 2 GB allocated to Docker (4 GB recommended)

### Step 1: Clone & Initialize

```bash
git clone https://github.com/logo-solutions/Nexus.git
cd Nexus

# Create data directory (NAS HDD6)
mkdir -p /Volumes/HDD6/03-Travail-Actif/nexus/nexus-data
chmod 755 /Volumes/HDD6/03-Travail-Actif/nexus/nexus-data
```

### Step 2: Deploy Stack

```bash
# Using the provided docker-compose.yml
docker compose up -d

# Check logs
docker compose logs -f nexus

# Wait for startup (2–3 minutes)
docker compose ps
# STATUS should be "healthy" after initialization
```

### Step 3: First Login

```bash
# Default credentials
USER: admin
PASS: admin123

# Access UI
open http://localhost:8081

# Change password immediately
# Settings → Security → Users → admin → Change Password
```

### Step 4: Create Docker Hosted Repository

```bash
# Via API (recommended, idempotent)
curl -X POST \
  -u admin:admin123 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "docker-candidat",
    "format": "docker",
    "type": "hosted",
    "online": true,
    "storage": {
      "blobStoreName": "default",
      "strictContentTypeValidation": true
    },
    "docker": {
      "v1Enabled": false,
      "httpPort": 5001
    }
  }' \
  http://localhost:8081/service/rest/v1/repositories/docker/hosted

# Or via UI
# Repositories → Create → Docker (Hosted)
# name: docker-candidat, HTTP: 5001, Storage: default, Strict validation: ✓
```

### Step 5: Create Service Accounts

For CI/CD pipelines (factice runner):

```bash
# Create account with limited permissions
curl -X POST \
  -u admin:admin123 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "svc-build-factice-app",
    "password": "GENERATED_PASSWORD",
    "roles": ["nx-repository-view-docker-*-edit"]
  }' \
  http://localhost:8081/service/rest/v1/security/users

# Store password in CI secret (NEXUS_BUILD_PASSWORD)
```

## Usage in CI/CD

### Push Artifact to Nexus (publish-candidate.sh)

```bash
# Environment variables
export NEXUS_DOCKER_CANDIDAT="localhost:8082"
export NEXUS_USER="svc-build-factice-app"
export NEXUS_PASSWORD="..."  # from CI secret
export VERSION="0.1.0"
export BUILD="42"
export COMMIT="abc123def"
export IMAGE="factice:abc123def"
export APP_DIR="app/"

# Run publish script
scripts/nexus/publish-candidate.sh

# Result: image, SBOM, manifest pushed to Nexus
```

### Pull from Nexus

```bash
docker login localhost:8082 -u svc-build-factice-app -p <PASSWORD>
docker pull localhost:8082/factice.app/factice:0.1.0-42@sha256:abc123...
```

## Backup & Recovery

### Backup

```bash
# Via Docker
docker exec nexus tar czf /nexus-data/nexus-backup-$(date +%Y%m%d).tar.gz \
  --exclude="instance" --exclude="log" /nexus-data

# Or via filesystem
tar czf /backup/nexus-$(date +%Y%m%d).tar.gz \
  /Volumes/HDD6/03-Travail-Actif/nexus/nexus-data
```

### Recovery

```bash
# Stop container
docker compose down

# Restore data
tar xzf /backup/nexus-YYYYMMDD.tar.gz -C /

# Restart
docker compose up -d
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Port 8081/8082 already in use** | `lsof -nP -iTCP:8081` → kill process or change docker-compose ports |
| **Nexus not responding after restart** | Check logs: `docker compose logs nexus` (2–3 min startup) |
| **Docker login fails (401 Unauthorized)** | Verify user exists + password correct: `curl -u user:pass http://localhost:8081/service/metrics/healthcheck` |
| **Repository not showing in API** | Check status: `curl -u admin:admin123 http://localhost:8081/service/rest/v1/repositories` |
| **Disk space full** | Monitor `/Volumes/HDD6/03-Travail-Actif/nexus/nexus-data` size; delete old artifacts via API or UI |

## API Reference

```bash
# List repositories
curl -u admin:admin123 http://localhost:8081/service/rest/v1/repositories | jq

# List artifacts in docker-candidat
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/search?repository=docker-candidat | jq

# Get repository details
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories/docker/hosted/docker-candidat | jq
```

## Performance & Tuning

### JVM Memory

```yaml
# docker-compose.yml
environment:
  - INSTALL4J_ADD_VM_PARAMS=-Xms1200m -Xmx1200m -XX:MaxDirectMemorySize=1g
```

Adjust `-Xmx` based on available RAM:
- 2 GB machine: `-Xmx512m`
- 4 GB machine: `-Xmx1200m`
- 8+ GB machine: `-Xmx2g`

### Blob Store Configuration

Default blob store is sufficient for <100 GB artifacts. For larger deployments, add separate blob stores per repository type.

## Contributing

Report issues or suggest improvements via [GitHub Issues](https://github.com/logo-solutions/factice/issues) (factice repo).

## License

Sonatype Nexus Community Edition. See [nexus_directeur.pptx](nexus_directeur.pptx) for design docs.

---

**Deployment**: `/Volumes/logousb/SSD/Projects/Nexus/` (Mac Mini)  
**Data**: `/Volumes/HDD6/03-Travail-Actif/nexus/nexus-data` (NAS HDD6)  
**Related**: [factice CI/CD](../factice/), [cmdb](../cmdb/)  
**Status**: Active, production
