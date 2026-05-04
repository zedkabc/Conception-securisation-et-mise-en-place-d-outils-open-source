# L3 — Documentation Nextcloud + LDAP (Service Secondaire)
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Service :** Nextcloud (Cloud interne, collaboration)  
**Statut :** Service secondaire (travail en binôme)  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Introduction

Nextcloud est une plateforme open source de stockage et de collaboration de fichiers, alternative auto-hébergée à Google Drive ou Microsoft OneDrive.

**Dans le cadre du projet RP-03, Nextcloud constitue un service secondaire** permettant aux enseignants et étudiants de partager des fichiers, calendriers et contacts de manière centralisée et sécurisée.

---

## 2. Pourquoi Nextcloud ?

**Comparaison avec les alternatives :**

| Critère | Nextcloud | Google Drive | OneDrive | Dropbox |
|:---|:---|:---|:---|:---|
| **Souveraineté données** |  Auto-hébergé |  Cloud US |  Cloud US |  Cloud US |
| **Coût** | 0 € (Open Source) | 6-18 €/utilisateur/mois | 5-12,5 €/utilisateur/mois | 10-20 €/utilisateur/mois |
| **Fonctionnalités collaboratives** |  Fichiers, Agenda, Contacts, Talk, Édition |  Complètes |  Complètes |  Fichiers uniquement |
| **Conformité RGPD** |  Totale (données en France) |  Cloud Act |  Cloud Act |  Cloud Act |
| **Personnalisation** |  Modulaire (extensions) |  Figé |  Figé |  Figé |

**Verdict :** Nextcloud garantit la souveraineté des données tout en offrant des fonctionnalités collaboratives complètes, sans coût récurrent.

---

## 3. Installation Nextcloud avec Docker Compose

### 3.1 Structure des fichiers

```
/home/iris/sisr/nextcloud/
├── docker-compose.yml
```

### 3.2 Fichier docker-compose.yml

```yaml
services:
  db:
    image: mariadb:latest
    container_name: nextcloud-db
    hostname: nextcloud-db
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - nextcloud_network

  app:
    image: nextcloud:latest
    container_name: nextcloud-app
    restart: always
    environment:
      MYSQL_HOST: nextcloud-db
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - nextcloud_data:/var/www/html
    depends_on:
      - db
    networks:
      - admin_proxy
      - nextcloud_network
    expose:
      - "80"
    labels:
      traefik.enable: "true"
      traefik.docker.network: admin_proxy

      traefik.http.middlewares.NEXTCLOUD-https-redirect.redirectregex.regex: "^http://cloud\\.${DOMAIN_NAME}(.*)"
      traefik.http.middlewares.NEXTCLOUD-https-redirect.redirectregex.replacement: "https://cloud.${DOMAIN_NAME}:4433$$1"
      traefik.http.middlewares.NEXTCLOUD-https-redirect.redirectregex.permanent: "true"

      traefik.http.routers.NEXTCLOUD-http.entrypoints: web
      traefik.http.routers.NEXTCLOUD-http.rule: "Host(`cloud.${DOMAIN_NAME}`)"
      traefik.http.routers.NEXTCLOUD-http.middlewares: NEXTCLOUD-https-redirect
      traefik.http.routers.NEXTCLOUD-http.service: NEXTCLOUD-svc

      traefik.http.routers.NEXTCLOUD.entrypoints: websecure
      traefik.http.routers.NEXTCLOUD.rule: "Host(`cloud.${DOMAIN_NAME}`)"
      traefik.http.routers.NEXTCLOUD.tls: "true"
      traefik.http.routers.NEXTCLOUD.tls.certresolver: letsencrypt
      traefik.http.routers.NEXTCLOUD.service: NEXTCLOUD-svc
      traefik.http.routers.NEXTCLOUD.middlewares: NEXTCLOUD-security-headers

      traefik.http.services.NEXTCLOUD-svc.loadbalancer.server.port: "80"

      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.stsSeconds: "31536000"
      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.stsIncludeSubdomains: "true"
      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.stsPreload: "true"
      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.forceSTSHeader: "true"
      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.contentTypeNosniff: "true"
      traefik.http.middlewares.NEXTCLOUD-security-headers.headers.browserXssFilter: "true"

  collabora:
    image: collabora/code:latest
    container_name: nextcloud-collabora
    restart: always
    environment:
      aliasgroup1: "https://cloud.${DOMAIN_NAME}:4433"
      extra_params: "--o:ssl.termination=true --o:ssl.enable=false"
      dictionaries: "fr_FR en_US"
      server_name: "office.${DOMAIN_NAME}:4433"
    cap_add:
      - MKNOD
    depends_on:
      - app
    networks:
      - admin_proxy
      - nextcloud_network
    expose:
      - "9980"
    labels:
      traefik.enable: "true"
      traefik.docker.network: admin_proxy

      traefik.http.middlewares.COLLABORA-https-redirect.redirectregex.regex: "^http://office\\.${DOMAIN_NAME}(.*)"
      traefik.http.middlewares.COLLABORA-https-redirect.redirectregex.replacement: "https://office.${DOMAIN_NAME}:4433$$1"
      traefik.http.middlewares.COLLABORA-https-redirect.redirectregex.permanent: "true"

      traefik.http.routers.COLLABORA-http.entrypoints: web
      traefik.http.routers.COLLABORA-http.rule: "Host(`office.${DOMAIN_NAME}`)"
      traefik.http.routers.COLLABORA-http.middlewares: COLLABORA-https-redirect
      traefik.http.routers.COLLABORA-http.service: COLLABORA-svc

      traefik.http.routers.COLLABORA.entrypoints: websecure
      traefik.http.routers.COLLABORA.rule: "Host(`office.${DOMAIN_NAME}`)"
      traefik.http.routers.COLLABORA.tls: "true"
      traefik.http.routers.COLLABORA.tls.certresolver: letsencrypt
      traefik.http.routers.COLLABORA.service: COLLABORA-svc

      traefik.http.services.COLLABORA-svc.loadbalancer.server.port: "9980"

volumes:
  db_data:
  nextcloud_data:

networks:
  admin_proxy:
    external: true
  nextcloud_network:
    driver: bridge
```

### 3.3 Fichier .env (Sécurité)

**Créer le fichier ` /home/iris/sisr/nextcloud/.env` :**

```env
# Mots de passe Nextcloud
MYSQL_ROOT_PASSWORD=""
MYSQL_DATABASE=""
MYSQL_USER="
MYSQL_PASSWORD=""
DOMAIN_NAME=iris.a3n.fr
```

** Sécurité :** Ne jamais commiter ce fichier dans Git (ajouter `.env` au `.gitignore`). Permissions : `chmod 600 .env`.

### 3.4 Déploiement

```bash
# Créer le réseau Traefik (si pas déjà fait)
docker network create traefik-network

# Créer les dossiers
cd /home/iris/sisr/nextcloud/
mkdir -p volumes/nextcloud-data volumes/postgres-data backup

# Créer le fichier .env
nano .env

# Lancer les conteneurs
docker compose up -d

# Vérifier
docker compose logs -f nextcloud
```

**Accès :** https://cloud.iris.a3n.fr:4433
      NEXTCLOUD_TRUSTED_DOMAINS: cloud.iris.a3n.fr
      OVERWRITEPROTOCOL: HTTP
      OVERWRITECLIURL: https://cloud.iris.a3n.fr:4433
    volumes:
      - ./volumes/nextcloud-data:/var/www/html
      - ./volumes/nextcloud-apps:/var/www/html/custom_apps
      - ./volumes/nextcloud-config:/var/www/html/config
    depends_on:
      - postgres-nextcloud
    networks:
      - nextcloud-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(`cloud.iris.a3n.fr`)"
      - "traefik.http.routers.nextcloud.entrypoints=web"
      - "traefik.http.routers.nextcloud.tls=true"
      - "traefik.http.services.nextcloud.loadbalancer.server.port=80"

networks:
  nextcloud-network:
    driver: bridge
```

### 3.3 Déploiement

```bash
cd /opt/iris-services/nextcloud/
mkdir -p volumes/{nextcloud-data,nextcloud-apps,nextcloud-config,postgres-data} backup
docker compose up -d
docker compose logs -f nextcloud
```
---

## 4. Configuration LDAP (OpenLDAP)

**Paramètres → LDAP integration**

**Configuration serveur :**
- Serveur : `ldap://openldap`
- Port : 389
- Base DN : `dc=mediaschool,dc=local`
- Bind DN : `cn=admin,dc=mediaschool,dc=local`
- Mot de passe : `[mot_de_passe_service_ldap]`

**Filtre utilisateurs :**
```ldap
(objectClass=inetOrgPerson)
```

**Mapping attributs :**
- Nom affiché : `displayName`
- Email : `mail`
- Quota : par groupe (Admin/Enseignants/Étudiants)

---

## 5. Configuration Quotas

**Paramètres → Utilisateurs**

| Groupe AD | Quota Nextcloud |
|:---|:---|
| Étudiants | 5 Go |
| Enseignants | 10 Go |
| Admins | Illimité |

---

## 6. Fonctionnalités Activées

**Applications installées :**
-  Fichiers (core)
-  Calendrier
-  Contacts
-  Talk (visioconférence interne)
-  OnlyOffice / Collabora (édition collaborative documents)

**Partages configurés :**
- Espace SISR (groupe Enseignants + Étudiants SISR)
- Espace SLAM (groupe Enseignants + Étudiants SLAM)
- Ressources pédagogiques (lecture seule pour Étudiants)

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
