# Project Context: ansible-role-firecrawl

**Type:** Ansible Role / Infrastructure Automation  
**Target:** Debian 12 (Bookworm), Debian 13 (Trixie), Ubuntu 22.04 (Jammy), Ubuntu 24.04 (Noble)  
**Service:** [Firecrawl](https://firecrawl.dev) Self-hosted Web Scraper, Crawler, and LLM Extraction Stack via Docker Compose

---

## Overview

Deploys the standalone Firecrawl self-hosted stack consisting of:
- **Firecrawl API** (`:3002`)
- **Firecrawl Worker** (background crawl queues)
- **Playwright Microservice** (`:3000`)
- **Redis** (`:6379`)
- **PostgreSQL / NuQ Database** (`:5432`)
- Optional **Nginx** reverse proxy with TLS termination

---

## Repository Layout

```
.
├── AGENTS.md                         # AI source of truth
├── CLAUDE.md                         # Claude Code instructions
├── README.md                         # Human documentation
├── ansible.cfg                       # Inventory and SSH settings (wires shared-inventory)
├── requirements.yml                  # Collection dependencies
├── inventory/
│   ├── hosts.yml                     # firecrawl_servers group
│   └── group_vars/                   # all/ and firecrawl_servers/
├── playbooks/
│   ├── site.yml                      # Full stack playbook
│   └── deploy.yml                    # Rapid role deployment playbook
└── roles/
    └── firecrawl/                    # Role implementation
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── meta/main.yml
        ├── tasks/
        │   ├── main.yml
        │   ├── deploy.yml
        │   └── nginx.yml
        └── templates/
            ├── docker-compose.yml.j2
            ├── env.j2
            └── nginx-site.conf.j2
```

---

## Common Commands

```bash
# Syntax check
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check
ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml --syntax-check

# Dry-run / Check mode
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff

# Full deployment
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# Rapid deploy / configuration update
ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml
```
