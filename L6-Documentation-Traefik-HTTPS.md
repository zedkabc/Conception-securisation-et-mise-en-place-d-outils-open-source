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
3. **Activer le certificateResolver ACME dans traefik.yml** :
   ```yaml
   certificatesResolvers:
     letsencrypt:
       acme:
         email: admin@iris.a3n.fr
         storage: /acme/acme.json
         httpChallenge:
           entryPoint: web
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
/opt/iris-services/traefik/
├── docker-compose.yml
├── traefik.yml             # Configuration statique
├── dynamic/
│   └── middlewares.yml     # Middlewares (headers sécurité)
├── certs/
│   ├── iris.a3n.fr.crt     # Certificat SSL auto-signé
│   └── iris.a3n.fr.key     # Clé privée
└── acme/
    └── acme.json           # Let's Encrypt (si activé)
```

### 4.2 Fichier docker-compose.yml

```yaml
version: '3.8'

services:
  traefik:
    container_name: traefik
    image: traefik:v2.11
    restart: always
    ports:
      - "80:80"       # HTTP — redirection vers HTTPS
      - "4433:443"    # HTTPS — point d'entrée utilisateurs
      - "8080:8080"   # Dashboard Traefik (accès VLAN admin uniquement)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic:/etc/traefik/dynamic:ro
      # - ./certs:/certs:ro           # À activer avec HTTPS
      # - ./acme:/acme                # À activer avec Let's Encrypt
    networks:
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.iris.a3n.fr`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"

networks:
  traefik-network:
    external: true
```

**Note :** Le service utilisateur est publié en HTTPS via le mapping `4433:443`.

### 4.3 Fichier traefik.yml (Configuration Statique)

```yaml
# Configuration globale
global:
  checkNewVersion: true
  sendAnonymousUsage: false

# API et Dashboard
api:
  dashboard: true
  insecure: false  # Dashboard accessible uniquement via Traefik routing

# Points d'entrée (EntryPoints)
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http:
      tls: true

# Fournisseurs
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik-network  # Force Traefik à utiliser ce réseau

  file:
    directory: "/etc/traefik/dynamic"
    watch: true

# Certificats SSL
# tls:
#   stores:
#     default:
#       defaultCertificate:
#         certFile: /certs/iris.a3n.fr.crt
#         keyFile: /certs/iris.a3n.fr.key

# Logs
log:
  level: INFO

accessLog:
  filePath: "/var/log/traefik/access.log"
  bufferingSize: 100
```

**Points importants :**
- **Entrypoint `web` sur le port 80** uniquement pour la redirection.
- **Entrypoint `websecure` sur le port 443** pour les services HTTPS.
- **Redirection HTTP → HTTPS** active.
- **Section TLS** active.

### 4.4 Fichier dynamic/middlewares.yml (Headers Sécurité)

```yaml
http:
  middlewares:
    security-headers:
      headers:
        sslRedirect: true
        forceSTSHeader: true
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
        frameDeny: true
        browserXssFilter: true
        referrerPolicy: "same-origin"
        customFrameOptionsValue: "SAMEORIGIN"
```

---

## 5. Génération Certificat SSL Auto-Signé

### 5.1 Créer le certificat wildcard

```bash
# Se placer dans le dossier certs
cd /opt/iris-services/traefik/certs/

# Générer clé privée + certificat (valide 1 an)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout iris.a3n.fr.key \
  -out iris.a3n.fr.crt \
  -subj "/C=FR/ST=PACA/L=Nice/O=IRIS/CN=*.iris.a3n.fr" \
  -addext "subjectAltName=DNS:*.iris.a3n.fr,DNS:iris.a3n.fr"

# Vérifier le certificat
openssl x509 -in iris.a3n.fr.crt -text -noout
```

### 5.2 Distribution via GPO

**Pour que les postes clients acceptent le certificat auto-signé :**

1. **Copier le certificat** `iris.a3n.fr.crt` sur le serveur de gestion
2. **Créer une GPO :**
   - **Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Trusted Root Certification Authorities**
   - **Import** → Sélectionner `iris.a3n.fr.crt`
3. **Appliquer la GPO** à l'OU contenant les postes clients
4. **Forcer mise à jour GPO** sur les postes : `gpupdate /force`

**Résultat :** Les navigateurs des postes acceptent le certificat sans avertissement.

---

## 6. Déploiement Traefik

### 6.1 Créer le réseau Docker

```bash
# Créer le réseau externe partagé par tous les services
docker network create traefik-network
```

### 6.2 Démarrer Traefik

```bash
cd /opt/iris-services/traefik/
docker compose up -d
docker compose logs -f traefik
```

### 6.3 Accès Dashboard Traefik

**URL :** https://traefik.iris.a3n.fr:4433

** SÉCURITÉ :** Le dashboard doit être accessible **uniquement depuis VLAN 99 (Administration)**.

**Configuration firewall :**
```bash
# Autoriser VLAN Admin uniquement
iptables -A INPUT -p tcp --dport 8080 -s 10.10.99.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -j DROP
```

---

## 7. Intégration d'un Service dans Traefik

### 7.1 Exemple : GLPI

**Dans le `docker-compose.yml` de GLPI, ajouter les labels :**

```yaml
services:
  glpi:
    # ... (config existante)
    networks:
      - glpi-network        # Réseau interne pour MariaDB
      - traefik-network     #  IMPORTANT : Réseau partagé avec Traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.glpi.rule=Host(`glpi.iris.a3n.fr`)"
      - "traefik.http.routers.glpi.entrypoints=websecure"
      - "traefik.http.routers.glpi.tls=true"
      - "traefik.http.services.glpi.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-network"  #  IMPORTANT : Force Traefik à utiliser ce réseau
      - "traefik.http.routers.glpi.middlewares=security-headers@file"

networks:
  glpi-network:
    driver: bridge
  traefik-network:
    external: true  #  IMPORTANT : Réseau créé en dehors du docker-compose
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

## 8. Redirection HTTP → HTTPS

**Configuré automatiquement dans `traefik.yml` :**

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
```

**Effet :** Toute requête HTTP est redirigée automatiquement vers HTTPS (code 301).

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
