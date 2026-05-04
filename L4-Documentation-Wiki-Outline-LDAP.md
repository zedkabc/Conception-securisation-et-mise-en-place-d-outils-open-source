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
version: '3.8'

services:
  postgres-outline:
    container_name: postgres-outline
    image: postgres:15
    restart: always
    environment:
      POSTGRES_DB: outline_db
      POSTGRES_USER: outline_user
      POSTGRES_PASSWORD: ${OUTLINE_DB_PASSWORD}
    volumes:
      - ./volumes/postgres-data:/var/lib/postgresql/data
    networks:
      - outline-network

  redis-outline:
    container_name: redis-outline
    image: redis:7
    restart: always
    volumes:
      - ./volumes/redis-data:/data
    networks:
      - outline-network

  outline:
    container_name: outline
    image: outlinewiki/outline:latest
    restart: always
    # PAS de section ports: — Traefik gère l'accès
    environment:
      SECRET_KEY: ${OUTLINE_SECRET_KEY}
      UTILS_SECRET: ${OUTLINE_UTILS_SECRET}
      DATABASE_URL: postgres://outline_user:${OUTLINE_DB_PASSWORD}@postgres-outline:5432/outline_db
      REDIS_URL: redis://redis-outline:6379
      URL: https://wiki.iris.a3n.fr:4433
      PORT: 3000
      FORCE_HTTPS: "false"
      ENABLE_UPDATES: "true"
      WEB_CONCURRENCY: 1
      # OIDC Configuration (Authelia bridge vers AD)
      OIDC_CLIENT_ID: ${OUTLINE_OIDC_CLIENT_ID}
      OIDC_CLIENT_SECRET: ${OUTLINE_OIDC_CLIENT_SECRET}
      OIDC_AUTH_URI: https://authelia.iris.a3n.fr:4433/api/oidc/authorization
      OIDC_TOKEN_URI: https://authelia.iris.a3n.fr:4433/api/oidc/token
      OIDC_USERINFO_URI: https://authelia.iris.a3n.fr:4433/api/oidc/userinfo
      OIDC_USERNAME_CLAIM: preferred_username
      OIDC_DISPLAY_NAME: "Connexion IRIS Nice (AD)"
      OIDC_SCOPES: "openid profile email"
    volumes:
      - ./volumes/outline-data:/var/lib/outline/data
    depends_on:
      - postgres-outline
      - redis-outline
    networks:
      - outline-network
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.outline.rule=Host(`wiki.iris.a3n.fr`)"
      - "traefik.http.routers.outline.entrypoints=web"
      - "traefik.http.services.outline.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-network"

networks:
  outline-network:
    driver: bridge
  traefik-network:
    external: true
```

### 3.3 Fichier .env (Sécurité)

**Créer le fichier `/opt/iris-services/outline/.env` :**

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
