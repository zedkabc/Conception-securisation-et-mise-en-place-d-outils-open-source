# L2 — Documentation Installation et Configuration GLPI + LDAP
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Service :** GLPI (Helpdesk & Ticketing) — **FOCUS PRINCIPAL**  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

## 1. Introduction

GLPI (Gestion Libre de Parc Informatique) est un système de helpdesk et de gestion d'incidents open source utilisé par de nombreuses entreprises et administrations françaises.

**Dans le cadre du projet RP-03, GLPI constitue le service principal** permettant de centraliser et gérer les demandes d'assistance informatique des étudiants et enseignants de l'école IRIS Nice.

### 1.1 Objectifs

- Mettre en place un système de ticketing professionnel
- Centraliser les demandes d'assistance (pas de demandes perdues ou oubliées)
- Tracer tous les incidents et leur résolution
- Classifier les tickets par niveau d'intervention (N1/N2/N3)
- Aligner le cycle de traitement sur les pratiques ITIL (incident, demande, escalade, clôture)
- Intégrer l'authentification avec l'OpenLDAP existant

### 1.2 Pourquoi GLPI ?

**Comparaison avec les alternatives du marché :**

| Critère | GLPI | ServiceNow | Jira Service Management | ManageEngine |
|:---|:---|:---|:---|:---|
| **Coût licence** | 0 € (Open Source) | 50 000 € - 500 000 €/an | 20-50 €/agent/mois | 13-60 €/technicien/mois |
| **Souveraineté données** |  Auto-hébergé |  Cloud (US) |  Cloud ou auto-hébergé |  Cloud ou auto-hébergé |
| **Inventaire intégré** |  Natif |  Natif (payant) |  Module séparé (Assets) |  Natif (version Enterprise) |
| **Éditeur français** |  Teclib' (France) |  ServiceNow Inc (US) |  Atlassian (Australie) |  Zoho (Inde) |
| **Conformité RGPD** |  Totale (données en France) |  Cloud Act |  Selon hébergement |  Selon hébergement |
| **Communauté francophone** |  Très large |  Limitée |  Bonne |  Limitée |

**Verdict :** GLPI est le seul outil qui réunit **helpdesk + inventaire dans une interface unifiée, sans frais de licence, tout en étant open source et français.**

---

## 2. Prérequis

### 2.1 Système d'exploitation

- **Serveur :** Debian 12 (ou Ubuntu 22.04 LTS)
- **RAM minimum :** 4 Go (recommandé : 8 Go)
- **Disque :** 50 Go minimum (pour base de données + pièces jointes)
- **CPU :** 2 vCPU minimum

### 2.2 Logiciels requis

- **Docker** : version 24.0+
- **Docker Compose** : version 2.20+
- **OpenLDAP** : opérationnel (RP-02)
- **Traefik** : reverse proxy configuré (voir L6)

### 2.3 Réseau

- **VLAN :** Serveur hébergé sur VLAN 10 (10.10.10.0/24)
- **DNS :** Entrée DNS `glpi.iris.a3n.fr` pointant vers l'IP du serveur
- **Port externe :** 4433 (HTTPS via Traefik)

---

## 3. Installation GLPI avec Docker Compose

### 3.1 Structure des fichiers

```
/home/iris/sisr/GLPI
├── docker-compose.yml
├── glpi-data/            # Données GLPI (pièces jointes, plugins)
├── mysql-data/           # Base de données MariaDB
├── backup.sh             # Sauvegardes
├── restore.sh                 
```

### 3.2 Fichier docker-compose.yml

```yaml
version: '3.9'

services:
  db:
    image: mariadb:10.6
    container_name: glpi-db
    hostname: glpi-db
    restart: always
    networks:
      - glpi
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  glpi:
    image: diouxx/glpi:latest
    container_name: glpi-app
    hostname: glpi-app
    restart: always
    networks:
      - glpi
      - admin_proxy
      - openldap_ldap_net
    depends_on:
      - db
    # ports:
    #   - "80:80"
    environment:
      DB_HOST: glpi-db
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
    volumes:
      - glpi_files:/var/www/html/glpi/files
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

volumes:
  db_data:
  glpi_files:

networks:
  glpi:
  admin_proxy:
    external: true
  openldap_ldap_net:
    external: true
```

** Points importants :**
- **Pas de section `ports:`** — Traefik route le trafic via le réseau Docker interne. Publier le port 8080 sur l'hôte créerait un conflit avec les autres services.
- **Deux réseaux :** `glpi-network` (interne, pour communiquer avec MariaDB) et `traefik-network` (externe, partagé avec Traefik).
- **Label `traefik.docker.network`** — Nécessaire quand un conteneur est sur plusieurs réseaux. Sans ce label, Traefik peut essayer de router via le mauvais réseau → erreur 502 Bad Gateway.
- **Variables d'environnement** — Les mots de passe sont définis dans un fichier `.env` (voir section 3.3).

### 3.3 Fichier .env (Sécurité)

**Créer le fichier `/home/iris/sisr/GLPI/.env` :**

```env
# Mots de passe base de données GLPI
YSQL_ROOT_PASSWORD=""
MYSQL_DATABASE=""
MYSQL_USER=""
MYSQL_PASSWORD=""
GLPI_VERSION=10.0.15
DOMAIN_NAME=iris.a3n.fr
```

** Sécurité :**
- Ce fichier ne doit **jamais** être commité dans un repository Git (ajouter `.env` au `.gitignore`)
- Permissions restrictives : `chmod 600 .env` (lecture/écriture root uniquement)
- Utiliser des mots de passe forts (minimum 16 caractères, alphanumériques + symboles)

### 3.4 Déploiement

```bash
# Créer le réseau Traefik (si pas déjà fait)
docker network create traefik-network

# Se placer dans le dossier GLPI
cd /home/iris/sisr/GLPI

# Créer les dossiers de volumes
mkdir -p mysql_data glpi_data

# Créer le fichier .env avec les mots de passe (voir section 3.3)
nano .env

# Démarrer les conteneurs
docker compose up -d

# Vérifier les logs
docker compose logs -f GLPI

# Attendre que GLPI soit prêt (environ 2-3 minutes au premier démarrage)
```

### 3.5 Accès initial

**URL :** https://glpi.iris.a3n.fr:4433 (via Traefik)

**Installation wizard GLPI :**

1. **Choix langue :** Français
2. **Accepter licence :** GPL v3
3. **Installer / Mettre à jour :** Installer
4. **Vérifications système :** Tout doit être vert
5. **Connexion base de données :**
   - Serveur SQL : `glpi-db`
   - Utilisateur SQL : `glpi_user`
   - Mot de passe SQL : `[voir fichier .env sécurisé]`
6. **Sélection base :** `glpi_db`
7. **Initialisation base :** Continuer
8. **Étape suivante :** Continuer
9. **Fin installation :** Connexion

**Comptes par défaut :**
- **Super-Admin :** glpi / glpi
- **Admin :** tech / tech
- **Post-only :** post-only / postonly

** SÉCURITÉ CRITIQUE :** Changer immédiatement tous les mots de passe par défaut après première connexion.

---

## 4. Configuration Initiale

### 4.1 Changement des mots de passe

**Se connecter avec :** glpi / glpi

**Administration → Utilisateurs :**
1. Sélectionner l'utilisateur `glpi`
2. Onglet **Mot de passe**
3. Nouveau mot de passe : `[mot_de_passe_fort_admin]`
4. Confirmer

**Répéter pour :** tech, post-only, normal

### 4.2 Configuration générale

**Configuration → Générale :**

**Onglet Configuration Générale :**
- Nom de l'instance : `GLPI IRIS Nice - Helpdesk`
- URL de l'application : `https://glpi.iris.a3n.fr:4433`
- Fuseau horaire : `Europe/Paris`
- Langue par défaut : `Français`

**Onglet Configuration d'affichage :**
- Format de date : `JJ-MM-AAAA`
- Format d'heure : `HH:MM`

**Onglet Restrictions d'accès :**
- Activer restriction par IP : Non (authentification LDAP suffit)

### 4.3 Suppression des fichiers d'installation

```bash
# Se connecter au conteneur GLPI
docker exec -it glpi bash

# Supprimer le fichier install
rm -f /var/www/html/glpi/install/install.php

# Quitter le conteneur
exit
```

---

## 5. Configuration LDAP (OpenLDAP)

### 5.1 Ajout du serveur LDAP

**Configuration → Authentification → Annuaire LDAP → Ajouter**

**Informations principales :**
- **Nom :** IRIS Mediaschool
- **Serveur par défaut :** Oui
- **Actif :** Oui

**Informations de connexion :**
- **Serveur :** `openldap`
- **Port :** `389`
- **Filtre de connexion :** `(objectClass=inetOrgPerson)`
- **BaseDN :** `dc=mediaschool,dc=local`
- **RootDN (utilisateur de liaison) :** `cn=admin,dc=mediaschool,dc=local`
- **Mot de passe (de liaison) :** `[mot_de_passe_compte_service_ldap]`
- **Utiliser bind :** Oui

**Utiliser TLS :** Non

**Informations de recherche :**
- **Attribut de connexion :** `ui`
- **Champ de l'identifiant :** `ui`
- **Champ du nom :** `sn`
- **Champ du prénom :** `givenname`
- **Champ de l'email :** `mail`
- **Champ du téléphone :** `telephonenumber`

**Filtre de recherche des utilisateurs :**
```ldap
(objectClass=inetOrgPerson)
```
*Ce filtre exclut les comptes désactivés.*

**Filtre de recherche des groupes :**
```ldap
(objectClass=group)
```

### 5.2 Test de connexion LDAP

**Configuration → Authentification → Annuaire LDAP → Tester**

**Saisir :**
- Identifiant : `[un_compte_LDAP_test]`
- Mot de passe : `[mot_de_passe_LDAP]`

**Résultat attendu :**
```
Connexion réussie
Utilisateur trouvé : Prénom Nom (email@iris.a3n.fr)
```

### 5.3 Import des utilisateurs depuis LDAP

**Administration → Utilisateurs → Liaison annuaire LDAP**

1. **Sélectionner l'annuaire :** OpenLDAP IRIS Nice
2. **Rechercher dans LDAP :**
   - Filtre : `*` (tous les utilisateurs)
   - Résultat : Liste des utilisateurs LDAP
3. **Sélectionner les utilisateurs à importer**
4. **Actions → Importer**

**Résultat :** Les utilisateurs LDAP sont créés dans GLPI avec leurs informations (nom, prénom, email).

### 5.4 Synchronisation automatique

**Configuration → Authentification → Annuaire LDAP → Éditer**

**Règles de liaison LDAP :**
- Activer l'importation automatique : Oui
- Synchroniser les champs lors de la connexion : Oui
- Synchroniser les groupes : Oui

**Planification de synchronisation (via cron) :**
```bash
# Ajouter dans crontab du serveur
0 2 * * * docker exec glpi php /var/www/html/glpi/front/ldap.php
```
*Synchronise tous les jours à 2h du matin.*

---

## 6. Configuration des Profils Utilisateurs

### 6.1 Profils par défaut GLPI

| Profil | Droits | Usage |
|:---|:---|:---|
| **Super-Admin** | Tous les droits (config, utilisateurs, base) | Administrateur système |
| **Admin** | Gestion tickets, inventaire, config limitée | Responsable technique |
| **Technicien** | Gestion tickets, consultation inventaire | Technicien support |
| **Hotliner** | Création tickets, traitement niveau 1 | Support N1 |
| **Observateur** | Consultation uniquement | Manager, auditeur |
| **Self-Service** | Création tickets uniquement | Utilisateur final |

### 6.2 Mapping Groupes LDAP → Profils GLPI

**Administration → Règles → Règles d'affectation d'entité et de droits**

**Règle 1 — Admins LDAP → Super-Admin GLPI**
- **Nom :** Attribution Super-Admin pour groupe Admins
- **Critères :**
  - `Groupe LDAP` / `est` / `CN=Admins,OU=Groups,dc=mediaschool,dc=local`
- **Actions :**
  - `Profil` / `Attribuer` / `Super-Admin`
  - `Entité` / `Attribuer` / `Root entity`
  - `Récursif` / `Oui`

**Règle 2 — Enseignants LDAP → Technicien GLPI**
- **Nom :** Attribution Technicien pour groupe Enseignants
- **Critères :**
  - `Groupe LDAP` / `est` / `CN=Enseignants,OU=Groups,dc=mediaschool,dc=local`
- **Actions :**
  - `Profil` / `Attribuer` / `Technicien`
  - `Entité` / `Attribuer` / `Root entity`
  - `Récursif` / `Oui`

**Règle 3 — Étudiants LDAP → Self-Service GLPI**
- **Nom :** Attribution Self-Service pour groupe Étudiants
- **Critères :**
  - `Groupe LDAP` / `est` / `CN=Etudiants,OU=Groups,dc=mediaschool,dc=local`
- **Actions :**
  - `Profil` / `Attribuer` / `Self-Service`
  - `Entité` / `Attribuer` / `Root entity`
  - `Récursif` / `Non`

**Ordre d'exécution des règles :** 1 → 2 → 3 (du plus restrictif au plus permissif)

---

## 7. Configuration des Catégories de Tickets

### 7.1 Arborescence des catégories

**Assistance → Catégories de tickets**

**Catégories principales (niveau 1) :**
1. **Problème réseau**
2. **Problème login / compte**
3. **Problème matériel**
4. **Demande d'installation**
5. **Sécurité**
6. **Autre**

**Sous-catégories (niveau 2) :**

**1. Problème réseau**
- Pas de connexion WiFi
- Pas de connexion filaire
- Lenteur réseau
- VLAN incorrect
- Problème IP (DHCP)

**2. Problème login / compte**
- Impossible de se connecter
- Mot de passe oublié
- Compte bloqué
- Droits insuffisants
- Création de compte

**3. Problème matériel**
- PC ne démarre pas
- Écran cassé / défaillant
- Clavier / souris défaillant
- Problème imprimante
- Autre périphérique

**4. Demande d'installation**
- Installation logiciel
- Installation matériel
- Accès à un service
- Configuration poste

**5. Sécurité**
- Faille détectée
- Accès non autorisé
- Virus / malware
- Incident de sécurité

**6. Autre**
- Demande d'information
- Suggestion d'amélioration

### 7.2 Configuration d'une catégorie

**Exemple : Catégorie "Problème réseau"**

**Informations principales :**
- **Nom :** Problème réseau
- **Commentaire :** Tous les problèmes liés au réseau (WiFi, filaire, VLAN)
- **Visible dans l'interface simplifiée :** Oui
- **Catégorie pour les modèles de ticket :** Oui

---

## 8. Workflow de Ticketing N1/N2/N3

### 8.1 Logique de classification

Le système de niveaux (N1/N2/N3) permet de **router les tickets vers les bonnes compétences** :

![Texte alternatif](Logigramme.png)

### 8.2 Cycle de vie d'un ticket

**États GLPI :**

1. **Nouveau (New)** — Ticket créé, en attente d'assignation
2. **En cours (traitement) (Processing - Assigned)** — Ticket assigné à un technicien
3. **En cours (planifié) (Processing - Planned)** — Intervention planifiée
4. **En attente (Waiting)** — Bloqué (en attente utilisateur ou ressource externe)
5. **Résolu (Solved)** — Solution appliquée, en attente validation utilisateur
6. **Clos (Closed)** — Ticket validé et fermé définitivement

**Workflow standard :**
```
Nouveau → En cours (Assigned) → Résolu → Clos
         ↓ (si bloqué)
         En attente → En cours (Assigned)
```

**Réouverture automatique :**
Si un utilisateur répond à un ticket **Résolu**, il repasse automatiquement en **En cours**.

---

## 9. Configuration des Notifications

### 9.1 Configuration SMTP

**Configuration → Notifications → Configuration des suivis par courriels**

**Onglet Configuration du courriel :**
- **Administrateur email :** `louka.lavenir@mediaschool.me`
- **Nom de l'administrateur :** `LAVENIR Louka` 

**Onglet Configuration du serveur de messagerie :**
- **Méthode d'envoi :** SMTP
- **Hôte SMTP :** `smtp.gmail.com`
- **Port SMTP :** `587` (STARTTLS)

**Tester l'envoi :**
- **Configuration → Notifications → Envoyer un courriel de test**
- Saisir une adresse email de test
- Vérifier réception

## 10. Interface Self-Service (Utilisateurs)

### 10.1 Accès interface simplifiée

**URL :** https://glpi.iris.a3n.fr:4433

**Connexion :**
- Identifiant : `[compte_LDAP_etudiant]`
- Mot de passe : `[mot_de_passe_LDAP]`

**Interface simplifiée affichée automatiquement pour le profil Self-Service.**

### 10.2 Créer un ticket

**Accueil → Créer un ticket**

**Formulaire :**
1. **Titre :** Description courte du problème
2. **Catégorie :** Choisir dans la liste (ex : "Problème réseau")
3. **Urgence :** 
   - Très basse
   - Basse
   - Moyenne (par défaut)
   - Haute
   - Très haute
4. **Description :** Description détaillée du problème
5. **Pièces jointes :** Captures d'écran, logs, etc.

**Soumettre le ticket.**

### 10.3 Suivre ses tickets

**Accueil → Mes tickets**

**Vue :**
- Liste de tous les tickets créés par l'utilisateur
- Statut (Nouveau, En cours, Résolu, Clos)
- Numéro, Titre, Date de création, Date de mise à jour

**Cliquer sur un ticket :**
- Voir détails complets
- Historique des actions
- Ajouter un suivi (message au technicien)
- Ajouter une pièce jointe

---

## 11. Interface Technicien

### 11.1 Connexion technicien

**URL :** https://glpi.iris.a3n.fr:4433

**Connexion :**
- Identifiant : `[compte_LDAP_enseignant]`
- Mot de passe : `[mot_de_passe_LDAP]`

**Interface complète GLPI affichée (pas interface simplifiée).**

### 11.2 Vue des tickets à traiter

**Assistance → Tickets**

**Vues disponibles :**
- **Tous les tickets** — Liste complète
- **Mes tickets** — Tickets assignés au technicien connecté
- **Tickets non attribués** — En attente d'assignation
- **Tickets de mon groupe** — Tickets du groupe du technicien

**Filtres :**
- Statut : Nouveau, En cours, En attente
- Catégorie
- Urgence
- Date de création / mise à jour

### 11.3 Traiter un ticket

**Cliquer sur un ticket → Vue détaillée**

**Actions disponibles :**

1. **Prendre en charge**
   - Cliquer sur **Attribuer à moi-même**
   - Statut passe de "Nouveau" à "En cours (traitement)"

2. **Ajouter un suivi**
   - Onglet **Suivi**
   - Cliquer sur **Ajouter un suivi**
   - Saisir message (description de l'intervention, questions à l'utilisateur)
   - Cocher **Envoyer notification** pour informer le demandeur

3. **Changer le statut**
   - Statut → Sélectionner "En attente" si bloqué
   - Ajouter un suivi expliquant pourquoi (ex : "En attente de validation enseignant")

4. **Résoudre le ticket**
   - Onglet **Solution**
   - Type de solution : 
     - Résolu
     - Résolu (avec approbation)
   - Description de la solution appliquée
   - Cliquer sur **Ajouter une solution**
   - Statut passe à "Résolu"
   - Email envoyé au demandeur

5. **Clore le ticket**
   - Si le demandeur valide → Statut passe automatiquement à "Clos" après 7 jours
   - Ou clôture manuelle si validation immédiate

### 11.4 Statistiques technicien

**Assistance → Statistiques**

**Métriques disponibles :**
- Nombre de tickets traités (par période)
- Temps moyen de résolution
- Tickets par catégorie
- Tickets par urgence
- Satisfaction utilisateur (si enquête activée)

---

## 12. Interface Administrateur

### 12.1 Accès complet

**Connexion avec compte Super-Admin.**

**Sections accessibles :**
- **Administration** — Utilisateurs, groupes, profils, règles
- **Configuration** — LDAP, notifications, SLA, entités
- **Outils** — Logs, base de données, plugins
- **Plugins** — Installer / gérer les extensions GLPI

### 12.2 Gestion des utilisateurs

**Administration → Utilisateurs**

**Actions :**
- Voir liste complète (locaux + LDAP)
- Éditer un utilisateur (email, téléphone, profil)
- Désactiver un compte
- Réinitialiser mot de passe (comptes locaux uniquement)

**Pour utilisateurs LDAP :**
- Synchronisation automatique depuis AD (pas de modification manuelle)

### 12.3 Extraction de rapports

**Outils → Rapports**

**Rapports disponibles :**
- Tickets par technicien
- Tickets par catégorie
- Temps de résolution moyen
- SLA respectés / dépassés
- Charge de travail par groupe

---

**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026
