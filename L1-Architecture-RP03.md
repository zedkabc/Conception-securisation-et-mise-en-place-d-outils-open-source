# L1 — Schéma d'Architecture des Services Open Source
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS  Mediaschool Nice  
**Focus Principal :** GLPI (Helpdesk & Ticketing)  
**Services Secondaires :** Nextcloud, Outline, LGP (Monitoring), Traefik  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Architecture Réseau Existante (RP-01)

L'infrastructure réseau de l'école IRIS Nice est déjà opérationnelle depuis le projet RP-01. Les services applicatifs déployés dans RP-03 s'appuient sur cette base.

### 1.1 Équipements réseau

| Équipement | Modèle | Rôle | Caractéristiques clés |
|:---|:---|:---|:---|
| **Switches** | Cisco Catalyst 2960-S | Commutation / segmentation VLAN | Layer 2 manageable, PoE+, support 802.1Q, port-security |
| **Points d'accès WiFi** | Cisco Catalyst C9105AXI-E | Accès sans fil sécurisé | WiFi 6 (802.11ax), PoE alimenté par le 2960-S, support WPA2/WPA3-Enterprise, 802.1X |
| **Routeurs** | Cisco CISCO1941W-E/K9 | Routage inter-VLAN / passerelle | Routeur ISR G2, modules intégrés, ACLs, NAT |

### 1.2 Schéma infrastructure réseau

![Schéma infrastructure réseau IRIS Nice](schema-reseau-iris.png)

**Légende :**
- **Internet** → Routeur Cisco 1941 (Gateway)
- **Switch Catalyst 2960-S** avec Spanning Tree Protocol (STP) pour prévention des boucles
- **VLAN 10** (Serveurs) : Virtualisation, RADIUS, DNS/DHCP, Services applicatifs (GLPI, Nextcloud, Wiki, Monitoring, Traefik)
- **VLAN 20** (Postes Étudiants) : Postes filaires salle TP
- **VLAN 30** (WiFi Étudiants) : Access Point C9105 WiFi 6
- **VLAN 99** (Administration) : Management switches/routeurs/AP

### 1.3 Plan de segmentation VLAN

| VLAN | ID | Réseau | Usage |
|:---|:---|:---|:---|
| **Serveurs** | 10 | 10.10.10.0/24 | Serveur de virtualisation, RADIUS, DNS/DHCP, **Services applicatifs (GLPI, Nextcloud, Wiki, Monitoring, Traefik)** |
| **Postes étudiants** | 20 | 10.10.20.0/24 | Postes filaires salle TP |
| **WiFi étudiants** | 30 | 10.10.30.0/24 | Accès WiFi via AP C9105 |
| **Administration** | 99 | 10.10.99.0/24 | Management switches/routeurs/AP |

**Point clé :** Tous les services applicatifs (GLPI, Nextcloud, Outline, Grafana, Traefik) sont hébergés sur le **VLAN 10 (Serveurs)**, isolés des postes utilisateurs.

---

## 2. Architecture Applicative (RP-03)

### 2.1 Vue d'ensemble

L'architecture applicative repose sur une **approche conteneurisée (Docker)** avec un **reverse proxy centralisé (Traefik)** qui expose tous les services en HTTPS sur le port 4433.

**Note importante :** Le déploiement est désormais actif en HTTPS (port 4433), avec redirection HTTP vers HTTPS.

**Service Principal :** GLPI (Helpdesk & Ticketing)  
**Services Secondaires :** Nextcloud, Outline, LGP (Monitoring), Traefik

**Authentification unifiée :** Tous les services s'authentifient via **LDAP** sur l'OpenLDAP existant.

### 2.2 URLs d'accès (via Traefik)

| Service | URL | Authentification | Port interne |
|:---|:---|:---|:---|
| **GLPI (Helpdesk)** | https://glpi.iris.a3n.fr:4433 | LDAP | 80 |
| **Nextcloud (Cloud)** | https://cloud.iris.a3n.fr:4433 | LDAP | 80 |
| **Outline (Wiki)** | https://wiki.iris.a3n.fr:4433 | OIDC via bridge LDAP | 3000 |
| **Grafana (Monitoring)** | https://grafana.iris.a3n.fr:4433 | LDAP | 3000 |

**Certificat HTTPS :** Auto-signé avec CN=*.iris.a3n.fr, distribué via GPO aux postes clients.

---

## 3. Intégration OpenLDAP (LDAP)

### 3.1 Configuration LDAP unifiée

Tous les services utilisent la même configuration LDAP pour se connecter à l'OpenLDAP :

**Paramètres LDAP :**
- **Serveur LDAP :** `ldap://openldap:389`
- **Base DN :** `dc=mediaschool,dc=local`
- **Bind DN :** `cn=admin,dc=mediaschool,dc=local`
- **Filtre utilisateurs :** `(objectClass=inetOrgPerson)`
- **Groupes AD utilisés :**
  - `CN=Admins,OU=Groups,dc=mediaschool,dc=local` → Administrateurs
  - `CN=Enseignants,OU=Groups,dc=mediaschool,dc=local` → Enseignants
  - `CN=Etudiants,OU=Groups,dc=mediaschool,dc=local` → Étudiants

### 3.2 Mapping groupes AD → Rôles applicatifs

| Service | Groupe AD | Rôle applicatif |
|:---|:---|:---|
| **GLPI** | Admins | Super-Admin |
| **GLPI** | Enseignants | Technicien |
| **GLPI** | Étudiants | Utilisateur |
| **Nextcloud** | Admins | Admin |
| **Nextcloud** | Enseignants | Utilisateur (quota élevé) |
| **Nextcloud** | Étudiants | Utilisateur (quota standard) |
| **Outline** | Admins | Admin |
| **Outline** | Enseignants | Éditeur |
| **Outline** | Étudiants | Lecteur |
| **Grafana** | Admins | Admin |
| **Grafana** | Enseignants | Éditeur |
| **Grafana** | Étudiants | Viewer |

**Avantage :** Un seul compte AD par utilisateur pour accéder à tous les services. Pas de gestion de comptes locaux multiples.

---

## 4. Services Secondaires

### 4.1 Nextcloud (Cloud interne)

**Rôle :** Stockage et collaboration de fichiers pour enseignants et étudiants.

**Fonctionnalités activées :**
- Stockage de fichiers avec quotas par groupe AD
- Calendrier partagé
- Contacts
- Édition collaborative (OnlyOffice ou Collabora)
- Partages par groupe AD (ex : espace SISR, espace SLAM)

**Authentification :** LDAP (groupes AD)  
**Quotas :**
- Étudiants : 5 Go
- Enseignants : 50 Go
- Admin : Illimité

### 4.2 Outline (Wiki interne)

**Rôle :** Documentation technique centralisée.

**Arborescence initiale :**
- Infrastructure (RP-01)
- OpenLDAP (RP-02)
- Services applicatifs (RP-03)
- Procédures
- Guides utilisateurs

**Authentification :** LDAP (groupes AD)  
**Droits :**
- Admin → Lecture/Écriture/Suppression
- Enseignants → Lecture/Écriture
- Étudiants → Lecture seule

### 4.3 LGP Stack (Monitoring)

**Rôle :** Supervision de l'infrastructure et des services.

**Composants :**
- **Prometheus** — Collecte des métriques (CPU, RAM, disque, réseau)
- **Loki** — Centralisation des logs
- **Grafana** — Visualisation (dashboards)
- **Exporters** — Node Exporter, Blackbox Exporter, SNMP Exporter

**Métriques surveillées :**
- CPU > 90% → Alerte
- Disque > 85% → Alerte
- Service down → Alerte critique
- SNMP (switches Cisco) → Bande passante, erreurs

**Dashboards Grafana :**
- Vue d'ensemble infrastructure
- État des services (GLPI, Nextcloud, Outline)
- Performances réseau (SNMP Cisco)
- Logs applicatifs (Loki)

### 4.4 Traefik (Reverse Proxy)

**Rôle :** Point d'entrée unique pour tous les services en HTTPS.

**Fonctionnalités :**
- Routage automatique par nom de domaine
- Génération certificats SSL
- Injection headers de sécurité
- Logs d'accès centralisés
- Dashboard d'administration

**Avantages vs Nginx :**
- Configuration automatique (détecte les conteneurs Docker)
- Let's Encrypt intégré (ACME)
- Support TCP/UDP natif
- Dashboard intégré
- Moins d'erreurs humaines (pas de config manuelle par service)

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026  
**Version :** 1.0


