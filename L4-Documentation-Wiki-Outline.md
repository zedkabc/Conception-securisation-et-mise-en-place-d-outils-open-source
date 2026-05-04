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
| **Authentification LDAP** |  Non (OIDC uniquement) |  Non |  Oui |  Oui |  Oui |  Oui |
| **Coût à l'échelle** |  Indépendant |  Lié au nombre d'utilisateurs |  Lié au nombre d'utilisateurs |  Indépendant |  Indépendant |  Indépendant |

---

## 3. Installation Outline avec Docker Compose

### 3.1 Structure des fichiers

```
/opt/iris-services/outline/
├── docker-compose.yml
├── .env
├── volumes/
│   ├── outline-data/
│   ├── postgres-data/
│   └── redis-data/
└── backup/
```

### 3.2 Fichier docker-compose.yml

```yaml
services:
  outline:
    image: docker.getoutline.com/outlinewiki/outline:latest
    env_file: ./.env
    expose:
      - "3000"
    volumes:
      - storage-data:/var/lib/outline/data
    depends_on:
      - postgres
      - redis
    networks:
      - admin_proxy
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=admin_proxy"
      - "traefik.http.routers.saidmkd.entrypoints=web"
      - "traefik.http.routers.saidmkd.rule=Host(`wiki.$DOMAIN_NAME`)"
      - "traefik.http.routers.saidmkd.service=saidmkd"
      - "traefik.http.services.saidmkd.loadbalancer.server.port=3000"

  redis:
    image: redis
    env_file: ./.env
    expose:
      - "6379"
    volumes:
      - ./redis.conf:/redis.conf
    command: ["redis-server", "/redis.conf"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3

  postgres:
    image: postgres
    env_file: ./.env
    expose:
      - "5432"
    volumes:
      - database-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-d", "outline", "-U", "user"]
      interval: 30s
      timeout: 20s
      retries: 3
    environment:
      POSTGRES_USER: $POSTGRES_USER
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
      POSTGRES_DB: $POSTGRES_DB

volumes:
  storage-data:
  database-data:

networks:
  admin_proxy:
    external: true
```

### 3.3 Fichier .env (Sécurité)

```env
# Mots de passe Outline
OUTLINE_DB_PASSWORD=mot_de_passe_db_complexe_ici

# Clés secrètes Outline (générer avec openssl rand -hex 32)
OUTLINE_SECRET_KEY=clé_secrète_64_caractères_hexadécimaux
OUTLINE_UTILS_SECRET=autre_clé_secrète_64_caractères_hexadécimaux

# Credentials OIDC (à obtenir depuis Authelia)
OUTLINE_OIDC_CLIENT_ID=outline
OUTLINE_OIDC_CLIENT_SECRET=secret_généré_par_authelia
```

**Génération des clés :**

```bash
# Générer SECRET_KEY
openssl rand -hex 32

# Générer UTILS_SECRET
openssl rand -hex 32
```

### 3.4 Déploiement

```bash
cd /opt/iris-services/outline/
mkdir -p volumes/{outline-data,postgres-data,redis-data} backup
docker compose up -d
docker compose logs -f outline
```

**Accès initial :** https://wiki.iris.a3n.fr:4433

---

## 5. Droits d'accès par Groupe AD

**Mapping groupes AD → Rôles Outline :**

| Groupe AD | Rôle Outline | Droits |
|:---|:---|:---|
| Admins | Admin | Lecture / Écriture / Suppression / Configuration |
| Enseignants | Éditeur | Lecture / Écriture |
| Étudiants | Lecteur | Lecture / Écriture |

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
