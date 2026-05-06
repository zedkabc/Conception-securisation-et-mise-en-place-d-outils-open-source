# L6 — Documentation Traefik + HTTPS (Reverse Proxy)
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Service :** Traefik (Reverse Proxy centralisé)  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Introduction

Traefik est un reverse proxy moderne et automatisé, conçu pour les architectures conteneurisées (Docker, Kubernetes).

**Dans le cadre du projet RP-03, Traefik constitue le point d'entrée unique** pour tous les services applicatifs (GLPI, Nextcloud, Outline, Grafana), en HTTPS sécurisé (port 4433).

---

## 2. C'est Quoi un Reverse Proxy ?

Le reverse proxy permet de **rediriger l'URL saisie vers le bon service**. C'est un intermédiaire entre le client et le serveur.

**Avantages :**
- Le serveur n'est **pas directement exposé** sur Internet
- Le reverse proxy est le **point d'entrée unique**
- On voit son IP, **pas celle du serveur**
- **Sécurité renforcée** (isolation des services)

---

## 3. Pourquoi Traefik (vs Nginx / Caddy) ?

**Benchmark complet :**

| Critère | Traefik | Nginx | Caddy |
|:---|:---|:---|:---|
| **Configuration automatique** |  Détecte les conteneurs Docker |  Config manuelle par service |  Semi-automatique |
| **Let's Encrypt intégré (ACME)** |  Oui |  Non (certbot externe) |  Oui |
| **Support TCP/UDP natif** |  Oui |  Limité |  Limitée |
| **Dashboard intégré** |  Oui |  Non |  Non |
| **Middlewares intégrés** |  Oui (auth, rate limiting, compression) |  Via modules |  Via plugins |
| **Erreurs humaines** |  Minimales (config automatique) |  Risque élevé (fichiers manuels) |  Modéré |

**Verdict :** Traefik est **le meilleur choix pour une architecture Docker** grâce à son auto-découverte des conteneurs, son intégration ACME, et son dashboard intégré.

---

## 3bis. HTTPS actif — Port 4433

L’infrastructure est désormais publiée en **HTTPS** avec un point d’entrée unique sur le **port 4433**.
Le port 80 peut rester ouvert uniquement pour rediriger automatiquement les clients vers HTTPS.

**Option A : Certificat auto-signé distribué par GPO**

1. **Générer le certificat wildcard** : `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout iris.a3n.fr.key -out iris.a3n.fr.crt -subj "/CN=*.iris.a3n.fr"`
2. **Distribuer via GPO** : Computer Configuration → Public Key Policies → Trusted Root Certification Authorities → Import `iris.a3n.fr.crt`
3. **Décommenter les sections HTTPS dans Traefik** : entrypoint `websecure`, section `tls`, redirection HTTP → HTTPS
4. **Redémarrer Traefik** : `docker compose restart traefik`

**Option B : Let's Encrypt si accès Internet ouvert**

1. **Ouvrir le port 80 sur le routeur** vers le serveur Traefik (pour validation HTTP-01 challenge)
2. **Configurer un enregistrement DNS public** : `*.iris.a3n.fr` → IP publique du routeur
3. **Activer le certificateResolver ACME dans traefik.toml** :
   ```toml
   [certificatesResolvers.letsencrypt.acme]
  email = "admin@a3n.fr"
  storage = "/acme.json"
  [certificatesResolvers.letsencrypt.acme.dnsChallenge]
    provider = "infomaniak"
    delayBeforeCheck = 10
   ```
4. **Modifier les labels des services** : `traefik.http.routers.<service>.tls.certresolver=letsencrypt`
5. **Redémarrer Traefik** : Let's Encrypt génère automatiquement les certificats

**Option C : Certificat interne via PKI d'établissement**

1. **Installer l'autorité de certification interne** sur le serveur d'infrastructure
2. **Générer une demande de certificat** (CSR) : `openssl req -new -newkey rsa:2048 -nodes -keyout iris.a3n.fr.key -out iris.a3n.fr.csr`
3. **Soumettre la CSR à ADCS** et obtenir le certificat signé
4. **Copier le certificat** dans `/opt/iris-services/traefik/certs/`
5. **Vérifier la publication HTTPS sur Traefik**

**Conclusion :** La configuration est active en HTTPS sur le port 4433.

---

## 4. Installation Traefik avec Docker Compose

### 4.1 Structure des fichiers

```
/home/iris/admin/
├── docker-compose.yml
├── systeme
    ├── traefik_data
        ├── traefik.toml
```

### 4.2 Fichier docker-compose.yml

```yaml
version: "3.7"

services:
  traefik:
    image: traefik:v3.6
    container_name: traefik
    hostname: traefik
    restart: always
    ports:
      - "80:80"
      - "443:443"
    env_file:
      - .env
    networks:
      proxy:
        ipv4_address: 172.100.10.10
      #- metric
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./systeme/traefik_data/traefik.toml:/etc/traefik/traefik.toml
      - ./systeme/traefik_data/acme.json:/acme.json
      - ./systeme/traefik_data/configurations:/configurations
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.traefik-secure.entrypoints=web"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik.${DOMAIN_NAME}`)"
      - "traefik.http.routers.traefik-secure.service=api@internal"
      - "traefik.http.routers.traefik-secure.middlewares=user-auth@file"

  portainer:
    image: portainer/portainer-ce:sts
    container_name: portainer
    hostname: portainer
    networks:
      proxy:
          ipv4_address: 172.100.10.20
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./systeme/portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      # Portainer en HTTP sur portainer.iris.a3n.fr:8080
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.routers.portainer.rule=Host(`portainer.${DOMAIN_NAME}`)"
      - "traefik.http.routers.portainer.service=portainer"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

networks:
  proxy:
    ipam:
      driver: default
      config:
        - subnet: "172.100.10.0/24"% 
```

**Note :** Le service utilisateur est publié en HTTPS via le mapping `4433:443`.

### 4.3 Fichier traefik.toml (Configuration Statique)

```toml
[log]
  level = "INFO"

[api]
  dashboard = true
  debug = true

[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.websecure]
    address = ":443"

[certificatesResolvers.letsencrypt.acme]
  email = "admin@a3n.fr"
  storage = "/acme.json"
  [certificatesResolvers.letsencrypt.acme.dnsChallenge]
    provider = "infomaniak"
    delayBeforeCheck = 10

[providers]
  [providers.docker]
    endpoint = "unix:///var/run/docker.sock"
    exposedByDefault = false

  [providers.file]
    directory = "/configurations"
    watch = true
```

**Points importants :**
- **Entrypoint `web` sur le port 80** uniquement pour la redirection.
- **Entrypoint `websecure` sur le port 443** pour les services HTTPS.
- **Redirection HTTP → HTTPS** active.
- **Section TLS** active.

## 5. Déploiement Traefik

### 5.1 Créer le réseau Docker

```bash
# Créer le réseau externe partagé par tous les services
docker network create traefik-network
```

### 5.2 Démarrer Traefik

```bash
cd /opt/iris-services/traefik/
docker compose up -d
docker compose logs -f traefik
```

---

## 6. Intégration d'un Service dans Traefik

### 6.1 Exemple : GLPI

**Dans le `docker-compose.yml` de GLPI, ajouter les labels :**

```yaml
services:
  glpi:
    # ... (config existante)
    networks:
      - glpi
      - admin_proxy
      - openldap_ldap_net
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=admin_proxy"

      # Middleware redirect HTTP → HTTPS
      - "traefik.http.middlewares.glpi-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.glpi-https-redirect.redirectscheme.permanent=true"

      # Router HTTP → redirect HTTPS
      - "traefik.http.routers.glpi-http.entrypoints=web"
      - "traefik.http.routers.glpi-http.rule=Host(`glpi.${DOMAIN_NAME}`)"
      - "traefik.http.routers.glpi-http.middlewares=glpi-https-redirect"
      - "traefik.http.routers.glpi-http.service=glpi-svc"

      # Router HTTPS avec TLS Let's Encrypt
      - "traefik.http.routers.glpi.entrypoints=websecure"
      - "traefik.http.routers.glpi.rule=Host(`glpi.${DOMAIN_NAME}`)"
      - "traefik.http.routers.glpi.tls=true"
      - "traefik.http.routers.glpi.tls.certresolver=letsencrypt"
      - "traefik.http.routers.glpi.service=glpi-svc"
      - "traefik.http.routers.glpi.middlewares=secureHeaders@file"

      # Service
      - "traefik.http.services.glpi-svc.loadbalancer.server.port=80"

networks:
  glpi:
  admin_proxy:
    external: true
  openldap_ldap_net:
    external: true
```

** Points critiques :**
- **Deux réseaux :** Le service doit être sur son réseau interne (pour communiquer avec sa base de données) ET sur `traefik-network` (pour être accessible via Traefik)
- **Label `traefik.docker.network`** : Quand un conteneur est sur plusieurs réseaux, ce label indique à Traefik quel réseau utiliser. Sans ce label, Traefik peut essayer de router via le mauvais réseau → erreur 502 Bad Gateway
- **Pas de section `ports:`** : Le service n'expose aucun port sur l'hôte. Seul Traefik expose le port 4433 pour les utilisateurs. Cela évite les conflits de ports et renforce la sécurité (les services ne sont accessibles que via Traefik)

**Redémarrer le conteneur :**
```bash
docker compose restart glpi
```

**Traefik détecte automatiquement le service et le rend accessible via HTTPS.**

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
