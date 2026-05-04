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
/opt/iris-services/nextcloud/
├── docker-compose.yml
├── volumes/
│   ├── nextcloud-data/        # Fichiers utilisateurs
│   ├── nextcloud-apps/        # Applications installées
│   ├── nextcloud-config/      # Configuration Nextcloud
│   └── postgres-data/         # Base de données PostgreSQL
└── backup/
```

### 3.2 Fichier docker-compose.yml

```yaml
version: '3.8'

services:
  postgres-nextcloud:
    container_name: postgres-nextcloud
    image: postgres:15
    restart: always
    environment:
      POSTGRES_DB: nextcloud_db
      POSTGRES_USER: nextcloud_user
      POSTGRES_PASSWORD: ${NC_DB_PASSWORD}
    volumes:
      - ./volumes/postgres-data:/var/lib/postgresql/data
    networks:
      - nextcloud-network

  nextcloud:
    container_name: nextcloud
    image: nextcloud:28
    restart: always
    # PAS de section ports: — Traefik gère l'accès
    environment:
      POSTGRES_HOST: postgres-nextcloud
      POSTGRES_DB: nextcloud_db
      POSTGRES_USER: nextcloud_user
      POSTGRES_PASSWORD: ${NC_DB_PASSWORD}
      NEXTCLOUD_ADMIN_USER: admin
      NEXTCLOUD_ADMIN_PASSWORD: ${NC_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: cloud.iris.a3n.fr
      OVERWRITEPROTOCOL: http
      OVERWRITECLIURL: https://cloud.iris.a3n.fr:4433
    volumes:
      - ./volumes/nextcloud-data:/var/www/html
    depends_on:
      - postgres-nextcloud
    networks:
      - nextcloud-network
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(`cloud.iris.a3n.fr`)"
      - "traefik.http.routers.nextcloud.entrypoints=web"
      - "traefik.http.services.nextcloud.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-network"

networks:
  nextcloud-network:
    driver: bridge
  traefik-network:
    external: true
```

### 3.3 Fichier .env (Sécurité)

**Créer le fichier `/opt/iris-services/nextcloud/.env` :**

```env
# Mots de passe Nextcloud
NC_DB_PASSWORD=mot_de_passe_db_complexe_ici
NC_ADMIN_PASSWORD=mot_de_passe_admin_complexe_ici
```

** Sécurité :** Ne jamais commiter ce fichier dans Git (ajouter `.env` au `.gitignore`). Permissions : `chmod 600 .env`.

### 3.4 Déploiement

```bash
# Créer le réseau Traefik (si pas déjà fait)
docker network create traefik-network

# Créer les dossiers
cd /opt/iris-services/nextcloud/
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
| Enseignants | 50 Go |
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
