# Ansible Role: Firecrawl (Self-Hosted)

Production-ready Ansible role to install, configure, manage, and verify a self-hosted **[Firecrawl](https://firecrawl.dev)** stack via Docker Compose.

---

## Architecture

```mermaid
graph TD
    ClientReq["Client / Scrape API Request"]

    subgraph Host["Target Host (Debian / Ubuntu)"]
        Nginx["Optional Nginx Reverse Proxy (:443 / :80)"]
        API["Firecrawl API (:3002)"]
        Worker["Firecrawl Queue Worker"]
        Playwright["Playwright Microservice (:3000)"]
        Redis["Redis (:6379)"]
        Postgres["PostgreSQL DB / NuQ Queue (:5432)"]
    end

    subgraph External["External / Optional Integrations"]
        LLM["Ollama / OpenAI API"]
        Searxng["SearXNG Search Engine"]
    end

    ClientReq -->|HTTPS| Nginx
    Nginx -->|Proxy| API
    ClientReq -.->|Direct HTTP (:3002)| API

    API -->|Enqueue Jobs| Redis
    API -->|Store Metadata| Postgres
    Worker -->|Consume Queue| Redis
    Worker -->|Fetch State| Postgres
    Worker -->|Render JS Pages| Playwright
    API -->|Direct Scrapes| Playwright
    Worker -->|Extraction / Embeddings| LLM
    API -->|Search Queries| Searxng
```

The stack coordinates the following core services:
- **`api`**: REST API handling `/v1/scrape`, `/v1/crawl`, `/v1/map`, `/v1/extract`, `/v1/batch/scrape`, and healthchecks.
- **`worker`**: Background worker consuming BullMQ / NuQ queues to perform crawling and page processing.
- **`playwright-service`**: Headless browser microservice executing JavaScript rendering.
- **`redis`**: High-performance cache and job queue broker.
- **`nuq-postgres`**: PostgreSQL database for crawl metadata and auth token persistence.
- **`nginx`** *(optional)*: TLS reverse-proxy termination with WebSocket and extended timeout support.

---

## Requirements

### Supported Target Operating Systems
- Debian 12 (Bookworm), Debian 13 (Trixie)
- Ubuntu 22.04 (Jammy), Ubuntu 24.04 (Noble)

### Hardware Recommendations
- **CPU**: 4+ vCPUs recommended for concurrent headless browser execution.
- **RAM**: 8+ GB RAM recommended (minimum 4 GB).
- **Disk**: 20+ GB SSD storage for container images and queue data.

### Ansible Dependencies
- `ansible-core >= 2.15`
- `community.docker >= 4.0.0`
- `community.general >= 11.0.0`

---

## Role Variables

All defaults are defined in [`roles/firecrawl/defaults/main.yml`](roles/firecrawl/defaults/main.yml).

### Container Images & Versions
| Variable | Default | Description |
| :--- | :--- | :--- |
| `firecrawl_image` | `ghcr.io/firecrawl/firecrawl` | Firecrawl container image |
| `firecrawl_version` | `latest` | Image tag / version |
| `firecrawl_playwright_image` | `ghcr.io/firecrawl/playwright-service` | Playwright service image |
| `firecrawl_playwright_version` | `latest` | Playwright image tag |
| `firecrawl_redis_image` | `redis:7-alpine` | Redis container image |
| `firecrawl_postgres_image` | `postgres:16-alpine` | PostgreSQL container image |

### Filesystem & Networking
| Variable | Default | Description |
| :--- | :--- | :--- |
| `firecrawl_dir` | `/opt/firecrawl` | Directory where compose files and .env reside |
| `firecrawl_data_dir` | `{{ firecrawl_dir }}/data` | Local data mount directory |
| `firecrawl_api_bind_address` | `127.0.0.1` | IP address to bind the API port |
| `firecrawl_api_port` | `3002` | Port exposed by the Firecrawl API container |
| `firecrawl_public_fqdn` | `firecrawl.example.com` | Public FQDN (used by Nginx vhost) |

### Concurrency & Worker Tuning
| Variable | Default | Description |
| :--- | :--- | :--- |
| `firecrawl_num_workers_per_queue` | `8` | Worker concurrency per queue |
| `firecrawl_crawl_concurrent_requests`| `10` | Max concurrent HTTP crawl requests |
| `firecrawl_max_concurrent_jobs` | `5` | Maximum active concurrent jobs |
| `firecrawl_browser_pool_size` | `5` | Maximum active Playwright browser instances |

### AI, Extraction & Search Integrations
| Variable | Default | Description |
| :--- | :--- | :--- |
| `firecrawl_openai_api_key` | `""` | OpenAI API key for `/v1/extract` |
| `firecrawl_openai_base_url` | `""` | Custom OpenAI-compatible endpoint |
| `firecrawl_ollama_base_url` | `""` | Base URL for local Ollama instances |
| `firecrawl_searxng_endpoint` | `""` | SearXNG instance endpoint for `/v1/search` |

### Reverse Proxy (Nginx) & TLS
| Variable | Default | Description |
| :--- | :--- | :--- |
| `firecrawl_manage_nginx` | `false` | When true, installs and configures Nginx vhost |
| `acme_cert_name` | `firecrawl` | ACME certificate identifier |
| `acme_cert_fullchain_file` | `/etc/ssl/{{ acme_cert_name }}/fullchain.pem` | Path to TLS fullchain |
| `acme_cert_key_file` | `/etc/ssl/{{ acme_cert_name }}/privkey.pem` | Path to TLS private key |

---

## Quickstart

### 1. Install Collections
```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Inventory
Edit `inventory/hosts.yml` or your shared inventory:
```yaml
all:
  children:
    firecrawl_servers:
      hosts:
        firecrawl1:
          ansible_host: 10.1.0.60
```

### 3. Deploy
```bash
# Syntax check
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check

# Dry-run
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff

# Execute deployment
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

---

## API Verification Examples

### Healthcheck
```bash
curl -i http://127.0.0.1:3002/test
# Returns HTTP 200 OK
```

### Scrape a Single Page
```bash
curl -X POST http://127.0.0.1:3002/v1/scrape \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://docs.firecrawl.dev",
    "formats": ["markdown", "html"]
  }'
```

### Map Links on a Site
```bash
curl -X POST http://127.0.0.1:3002/v1/map \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com"
  }'
```

---

## Operations & Maintenance

### Checking Container Status
```bash
cd /opt/firecrawl
docker compose ps
```

### Viewing Logs
```bash
cd /opt/firecrawl
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f playwright-service
```

### Restarting Stack
```bash
cd /opt/firecrawl
docker compose restart
```

---

## License
[MIT](LICENSE)
