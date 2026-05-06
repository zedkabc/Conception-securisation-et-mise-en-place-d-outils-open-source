# L4 — Documentation Wiki Outline (Service Secondaire)
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Service :** Outline (Wiki interne, base de connaissances)  
**Statut :** Service secondaire (travail en binôme)  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Introduction

Outline est un wiki moderne et collaboratif, alternative open source à Notion ou Confluence.

**Dans le cadre du projet RP-03, Outline constitue un service secondaire** permettant de centraliser la documentation technique de l'école IRIS Nice (infrastructure, procédures, guides).

---

## 2. Pourquoi Outline ?

**Comparaison avec les alternatives :**

| Critère | Outline | Notion | Confluence | MediaWiki | Dokuwiki | BookStack |
|:---|:---|:---|:---|:---|:---|:---|
| **Souveraineté données** |  Auto-hébergé |  Cloud |  Cloud |  Auto-hébergé |  Auto-hébergé |  Auto-hébergé |
| **Interface moderne** |  Oui |  Oui |  Complexe |  Datée |  Simple |  Moderne |
| **Co-édition temps réel** |  Oui |  Oui |  Oui |  Non |  Non |  Non |
| **Authentification LDAP** |  OIDC Natif via Keycloak (SSO) |  Non |  Oui |  Oui |  Oui |  Oui |
| **Coût à l'échelle** |  Indépendant |  Lié au nombre d'utilisateurs |  Lié au nombre d'utilisateurs |  Indépendant |  Indépendant |  Indépendant |

---

## 3. Installation Outline avec Docker Compose

### 3.1 Structure des fichiers

```
/home/iris/sisr/outline-main/
├── docker-compose.yml
├── .env
└── backup/
```

### 3.2 Fichier docker-compose.yml

```yaml
version: "3.9"

services:
  outline:
    image: docker.getoutline.com/outlinewiki/outline:latest
    env_file:
      - ./.env
    expose:
      - "3000"
    volumes:
      - storage-data:/var/lib/outline/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - admin_proxy
    labels:
      traefik.enable: "true"
      traefik.docker.network: admin_proxy

      # Middleware redirect HTTP → HTTPS
      traefik.http.middlewares.outline-https-redirect.redirectscheme.scheme: https
      traefik.http.middlewares.outline-https-redirect.redirectscheme.permanent: "true"

      # Router HTTP → redirect HTTPS
      traefik.http.routers.outline-http.entrypoints: web
      traefik.http.routers.outline-http.rule: Host(`wiki.iris.a3n.fr`)
      traefik.http.routers.outline-http.middlewares: outline-https-redirect
      traefik.http.routers.outline-http.service: outline-svc

      # Router HTTPS avec TLS Let's Encrypt
      traefik.http.routers.outline.entrypoints: websecure
      traefik.http.routers.outline.rule: Host(`wiki.iris.a3n.fr`)
      traefik.http.routers.outline.tls: "true"
      traefik.http.routers.outline.tls.certresolver: letsencrypt
      traefik.http.routers.outline.service: outline-svc

      # Service
      traefik.http.services.outline-svc.loadbalancer.server.port: "3000"

      # Headers HSTS
      traefik.http.routers.outline.middlewares: outline-security-headers
      traefik.http.middlewares.outline-security-headers.headers.stsSeconds: "31536000"
      traefik.http.middlewares.outline-security-headers.headers.stsIncludeSubdomains: "true"
      traefik.http.middlewares.outline-security-headers.headers.forceSTSHeader: "true"

  redis:
    image: redis:7
    expose:
      - "6379"
    volumes:
      - ./redis.conf/redis.conf:/redis.conf
    command: ["redis-server", "/redis.conf"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3
    restart: unless-stopped
    networks:
      - admin_proxy

  postgres:
    image: postgres:16
    expose:
      - "5432"
    volumes:
      - database-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD", "pg_isready", "-d", "${POSTGRES_DB}", "-U", "${POSTGRES_USER}"]
      interval: 30s
      timeout: 20s
      retries: 3
    restart: unless-stopped
    # SECURITE: PostgreSQL reçoit UNIQUEMENT ses 3 variables — pas le .env complet
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    networks:
      - admin_proxy

volumes:
  storage-data:
  database-data:

networks:
  admin_proxy:
    external: true
```

### 3.3 Fichier .env (Sécurité)

```env
# Outline Wiki — Configuration générée par setup.sh
# NE PAS COMMITTER CE FICHIER (il est dans .gitignore)
# Régénérer avec: make setup

# ── Base de données ───────────────────────────────────────────────
POSTGRES_USER=""
POSTGRES_PASSWORD=""
POSTGRES_DB=""
DATABASE_URL=""

# ── Application ───────────────────────────────────────────────────
DOMAIN_NAME=iris.a3n.fr
URL=https://wiki.iris.a3n.fr:4433
WEB_CONCURRENCY=1
PGSSLMODE=disable
DEFAULT_LANGUAGE=fr_FR

# ── Secrets ───────────────────────────────────────────────────────
SECRET_KEY=""
UTILS_SECRET=""

# ── Redis ─────────────────────────────────────────────────────────
REDIS_URL=redis://:e40e937380b68dea862b1526635cc17309572c0e2cb9f13b@redis:6379

# ── HTTPS ─────────────────────────────────────────────────────────
FORCE_HTTPS=true

# ── Stockage ──────────────────────────────────────────────────────
FILE_STORAGE=local
FILE_STORAGE_LOCAL_ROOT_DIR=/var/lib/outline/data
FILE_STORAGE_UPLOAD_MAX_SIZE=262144000

# ── Rate limiter ──────────────────────────────────────────────────
RATE_LIMITER_ENABLED=true
RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60

# ── Email (Gmail App Password) ────────────────────────────────────
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_SECURE=true
SMTP_DISABLE_STARTTLS=false
SMTP_USERNAME=""
SMTP_PASSWORD=""
SMTP_FROM_EMAIL=""
SMTP_REPLY_EMAIL=""

# ── Debug ─────────────────────────────────────────────────────────
LOG_LEVEL=info
DEBUG=http
ENABLE_UPDATES=false
```

### 3.4 Déploiement

```bash
cd /home/iris/sisr/outline-main/
docker compose up -d
docker compose logs -f outline
```

**Accès initial :** https://wiki.iris.a3n.fr:4433

---

## 5. Droits d'accès par Groupe AD

**Mapping groupes AD → Rôles Outline :**

| Groupe Keycloak | Rôle Outline | Droits |
|:---|:---|:---|
| Admins | Admin | Lecture / Écriture / Suppression / Configuration |
| Enseignants | Éditeur | Lecture / Écriture |
| Étudiants | Lecteur | Lecture / Écriture |

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
