# L8 — Procédure d'Intégration d'un Nouveau Service dans Traefik
**Projet :** RP-03 — Déploiement d'outils open source pour CFA IRIS Nice  
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

## 6. Étape 4 — Vérifier l'Intégration Traefik

### 6.1 Vérifier que Traefik détecte le service

**Accéder au Dashboard Traefik :**
- URL : https://traefik.iris.a3n.fr:4433
- Se connecter avec compte admin

**Vérifier dans l'onglet "HTTP Routers" :**
- Routeur `mon-app@docker` doit apparaître
- Règle : `Host(\`mon-app.iris.a3n.fr\`)`
- Statut : Vert (actif)

### 6.2 Tester l'accès HTTPS

**Depuis un poste client (VLAN 20 ou 30) :**
```bash
curl -I https://mon-app.iris.a3n.fr:4433

# Résultat attendu :
HTTP/2 200
server: nginx
strict-transport-security: max-age=31536000; includeSubDomains
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
x-xss-protection: 1; mode=block
```

**Ouvrir dans un navigateur :**
- URL : https://mon-app.iris.a3n.fr:4433
- Certificat SSL accepté (via GPO)
- Service accessible

---

## 7. Étape 5 — Configuration Firewall (Sécurité)

### 7.1 Bloquer l'accès direct au port interne

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

### 7.2 Vérifier que l'accès direct est bloqué

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

## 8. Étape 6 — Configuration LDAP (Optionnel)

**Si le service supporte l'authentification LDAP :**

**Paramètres LDAP unifiés (OpenLDAP) :**
- **Nom :** `IRIS Mediaschool`
- **Serveur LDAP :** `openldap`
- **Port :** `389`
- **Base DN :** `dc=mediaschool,dc=local`
- **Bind DN :** `cn=admin,dc=mediaschool,dc=local`
- **Mot de passe :** `[mot_de_passe_service_ldap]`
- **Filtre utilisateurs :** `(objectClass=inetOrgPerson)`
- **Champ de l'identifiant :** `ui`

**Mapping groupes LDAP → Rôles applicatifs :**
- `CN=Admins,OU=Groups,dc=mediaschool,dc=local` → Admin
- `CN=Enseignants,OU=Groups,dc=mediaschool,dc=local` → Utilisateur avancé
- `CN=Etudiants,OU=Groups,dc=mediaschool,dc=local` → Utilisateur standard

---

## 9. Étape 7 — Tests de Validation

### 9.1 Checklist de validation

- [ ] Service accessible via HTTPS (https://mon-app.iris.a3n.fr:4433)
- [ ] Certificat SSL accepté par les navigateurs (GPO)
- [ ] Redirection HTTP → HTTPS automatique
- [ ] Headers de sécurité présents (curl -I)
- [ ] Accès direct au port interne bloqué (depuis VLAN 20/30)
- [ ] Accès HTTPS autorisé depuis VLAN 20/30
- [ ] Dashboard Traefik affiche le routeur
- [ ] Authentification LDAP fonctionnelle (si applicable)

### 9.2 Tests de charge (optionnel)

```bash
# Test avec Apache Bench
ab -n 1000 -c 10 https://mon-app.iris.a3n.fr:4433/

# Vérifier les temps de réponse dans les logs Traefik
docker compose -f /opt/iris-services/traefik/docker-compose.yml logs traefik | grep "mon-app"
```

---

## 10. Étape 8 — Documentation

### 10.1 Mettre à jour la documentation

**Ajouter le service dans :**
- **L1 — Schéma d'architecture** (mise à jour du schéma)
- **Wiki Outline** (page dédiée au service)
- **Procédures internes** (installation, configuration, utilisation)

### 10.2 Informer les utilisateurs

**Communiquer l'URL d'accès :**
- Email aux utilisateurs
- Affichage sur le portail intranet (si existant)
- Ajout dans la documentation utilisateur

---

## 11. Troubleshooting

### Problème 1 — Traefik ne détecte pas le service

**Cause :** Le conteneur n'est pas sur le réseau `traefik-network`.

**Solution :**
```bash
# Vérifier le réseau du conteneur
docker inspect mon-app | grep NetworkMode

# Ajouter le réseau si absent
docker network connect traefik-network mon-app
docker compose restart mon-app
```

### Problème 2 — Erreur 404 Not Found

**Cause :** La règle de routage ne correspond pas au nom de domaine.

**Solution :**
- Vérifier le label `traefik.http.routers.mon-app.rule`
- Vérifier la résolution DNS (`nslookup mon-app.iris.a3n.fr`)

### Problème 3 — Erreur 502 Bad Gateway

**Cause :** Le port interne est incorrect ou le service ne répond pas.

**Solution :**
```bash
# Vérifier le port du service
docker compose logs mon-app

# Vérifier que le service écoute bien sur le port déclaré
docker exec mon-app netstat -tuln | grep 8080

# Corriger le label si nécessaire
traefik.http.services.mon-app.loadbalancer.server.port=8080
```

### Problème 4 — Certificat SSL non accepté

**Cause :** GPO non appliquée ou certificat expiré.

**Solution :**
```bash
# Forcer mise à jour GPO sur les postes clients
gpupdate /force

# Vérifier la validité du certificat
openssl x509 -in /opt/iris-services/traefik/certs/iris.a3n.fr.crt -text -noout | grep "Not After"
```

---

## 12. Exemple Complet — Service "PhpMyAdmin"

### 12.1 docker-compose.yml

```yaml
version: '3.8'

services:
  phpmyadmin:
    container_name: phpmyadmin
    image: phpmyadmin:latest
    restart: always
    environment:
      PMA_HOST: mariadb-glpi
      PMA_PORT: 3306
      PMA_ARBITRARY: 1
    networks:
      - traefik-network
      - glpi-network  # Pour accéder à MariaDB de GLPI
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.phpmyadmin.rule=Host(`pma.iris.a3n.fr`)"
      - "traefik.http.routers.phpmyadmin.entrypoints=web"
      - "traefik.http.routers.phpmyadmin.tls=true"
      - "traefik.http.services.phpmyadmin.loadbalancer.server.port=80"
      - "traefik.http.routers.phpmyadmin.middlewares=security-headers@file"

networks:
  traefik-network:
    external: true
  glpi-network:
    external: true
```

### 12.2 Déploiement

```bash
cd /opt/iris-services/phpmyadmin
docker compose up -d
```

### 12.3 Accès

**URL :** https://pma.iris.a3n.fr:4433  
**Accès réservé :** VLAN 99 (Administration uniquement)

---

## 13. Conclusion

Cette procédure garantit une **intégration sécurisée et standardisée** de tout nouveau service web dans l'infrastructure IRIS Nice.

**Points clés :**
1. Toujours passer par Traefik (pas d'accès direct)
2. HTTP obligatoire (certificat SSL)
3. Headers de sécurité activés
4. Firewall configuré (restriction réseau)
5. Documentation mise à jour

**Prochaines étapes :**
- Automatiser avec Ansible (déploiement one-click)
- Mettre en place monitoring (Prometheus/Grafana)
- Configurer sauvegardes automatiques

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026  
**Version :** 1.0


