# L8 — Procédure d'Intégration d'un Nouveau Service dans Traefik
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Public :** Administrateurs système  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Introduction

Cette procédure détaille les étapes pour **intégrer un nouveau service web** dans le reverse proxy Traefik, afin de l'exposer en HTTPS de manière sécurisée (port 4433).

**Services déjà intégrés (exemples) :**
- GLPI (https://glpi.iris.a3n.fr:4433)
- Nextcloud (https://cloud.iris.a3n.fr:4433)
- Outline (https://wiki.iris.a3n.fr:4433)
- Grafana (https://grafana.iris.a3n.fr:4433)

---

## 2. Prérequis

### 2.1 Informations nécessaires

Avant de commencer, rassembler les informations suivantes :

| Information | Exemple | Commentaire |
|:---|:---|:---|
| **Nom du service** | `mon-app` | Nom court, sans espaces |
| **Image Docker** | `mon-app:latest` | Nom de l'image + tag |
| **Port interne** | `8080` | Port d'écoute du service dans le conteneur |
| **Sous-domaine souhaité** | `mon-app.iris.a3n.fr` | URL d'accès externe |
| **Réseau Docker** | `traefik-network` | Réseau partagé avec Traefik |

### 2.2 Vérifications

**Vérifier que Traefik est opérationnel :**
```bash
docker ps | grep traefik
# Doit afficher le conteneur traefik en état "Up"
```

**Vérifier que le réseau `traefik-network` existe :**
```bash
docker network ls | grep traefik-network
# Si absent : docker network create traefik-network
```

---

## 3. Étape 1 — Créer l'Entrée DNS

### 3.1 Ajouter l'enregistrement DNS

**Sur le serveur DNS interne (ou fichier hosts si test) :**

**Exemple avec DNS interne :**
```bash
# Ajouter dans la zone DNS iris.a3n.fr
mon-app.iris.a3n.fr.  IN  A  10.10.10.10
```

**Exemple avec /etc/hosts (pour test uniquement) :**
```bash
# Sur les postes clients
echo "10.10.10.10  mon-app.iris.a3n.fr" | sudo tee -a /etc/hosts
```

### 3.2 Tester la résolution DNS

```bash
# Depuis un poste client
nslookup mon-app.iris.a3n.fr

# Résultat attendu :
Server:  openldap
Address:  10.10.10.1

Name:    mon-app.iris.a3n.fr
Address:  10.10.10.10
```

---

## 4. Étape 2 — Créer le Fichier docker-compose.yml

### 4.1 Structure du fichier

**Créer le dossier du service :**
```bash
mkdir -p /opt/iris-services/mon-app
cd /opt/iris-services/mon-app
```

**Créer le fichier `docker-compose.yml` :**
```yaml
version: '3.8'

services:
  mon-app:
    container_name: mon-app
    image: mon-app:latest
    restart: always
    ports:
      - "8080:8080"  # Port interne (optionnel si uniquement via Traefik)
    environment:
      # Variables d'environnement du service
      APP_ENV: production
      APP_URL: https://mon-app.iris.a3n.fr:4433
    volumes:
      - ./volumes/mon-app-data:/data
    networks:
      - traefik-network  #  IMPORTANT : Réseau partagé avec Traefik
    labels:
      # ========== CONFIGURATION TRAEFIK ==========
      - "traefik.enable=true"
      
      # Règle de routage (nom de domaine)
      - "traefik.http.routers.mon-app.rule=Host(`mon-app.iris.a3n.fr`)"
      
      # Point d'entrée HTTPS
      - "traefik.http.routers.mon-app.entrypoints=websecure"
      
      # Activer TLS (HTTPS)
      - "traefik.http.routers.mon-app.tls=true"
      
      # Port interne du service
      - "traefik.http.services.mon-app.loadbalancer.server.port=8080"
      
      # Middleware sécurité (headers HTTP)
      - "traefik.http.routers.mon-app.middlewares=security-headers@file"

networks:
  traefik-network:
    external: true
```

### 4.2 Explications des labels Traefik

| Label | Valeur | Description |
|:---|:---|:---|
| `traefik.enable` | `true` | Active la détection par Traefik |
| `traefik.http.routers.mon-app.rule` | `Host(\`mon-app.iris.a3n.fr\`)` | Règle de routage (nom de domaine) |
| `traefik.http.routers.mon-app.entrypoints` | `websecure` | Point d'entrée HTTPS (port 4433) |
| `traefik.http.routers.mon-app.tls` | `true` | Active TLS/SSL |
| `traefik.http.services.mon-app.loadbalancer.server.port` | `8080` | Port interne du conteneur |
| `traefik.http.routers.mon-app.middlewares` | `security-headers@file` | Middleware headers sécurité |

---

## 5. Étape 3 — Déployer le Service

### 5.1 Créer les volumes (si nécessaire)

```bash
mkdir -p volumes/mon-app-data
```

### 5.2 Démarrer le conteneur

```bash
cd /opt/iris-services/mon-app
docker compose up -d
```

### 5.3 Vérifier les logs

```bash
docker compose logs -f mon-app
```

**Résultat attendu :**
- Conteneur démarré sans erreur
- Service accessible sur son port interne

---

## 6. Étape 5 — Configuration Firewall (Sécurité)

### 6.1 Bloquer l'accès direct au port interne

**Objectif :** Forcer le passage par Traefik (HTTPS) uniquement.

**Ajouter règle iptables sur le serveur :**
```bash
# Autoriser HTTPS depuis utilisateurs (VLAN 20, 30)
iptables -A INPUT -p tcp --dport 4433 -s 10.10.20.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 4433 -s 10.10.30.0/24 -j ACCEPT

# Bloquer accès direct au port interne (8080) depuis utilisateurs
iptables -A INPUT -p tcp --dport 8080 -s 10.10.20.0/24 -j DROP
iptables -A INPUT -p tcp --dport 8080 -s 10.10.30.0/24 -j DROP

# Admin VLAN → accès complet
iptables -A INPUT -s 10.10.99.0/24 -j ACCEPT

# Sauvegarder les règles
netfilter-persistent save
```

### 6.2 Vérifier que l'accès direct est bloqué

**Depuis un poste client (VLAN 20) :**
```bash
nc -zv 10.10.10.10 8080
# Résultat attendu : timeout ou connexion refusée
```

**Depuis un poste admin (VLAN 99) :**
```bash
nc -zv 10.10.10.10 8080
# Résultat attendu : réponse du service
```

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
