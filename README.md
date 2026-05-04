# Dossier de Réalisation Professionnelle GLPI
**Projet :** Déploiement d'outils open source pour IRIS Nice  
**Auteur :** Louka Lavenir  
**Période de réalisation :** 24/11/2025 au 28/11/2025  
**Lieu :** Mediaschool — IRIS Nice  

---

## INTRODUCTION

Dans le cadre de la réalisation professionnelle, l’objectif principal est de concevoir et déployer un service de ticketing professionnel pour centraliser les demandes d’assistance informatique de l’école IRIS Nice.  

La solution retenue est **GLPI**, déployée sur infrastructure Linux conteneurisée. Le service est intégré à l’écosystème existant (Traefik, Active Directory, supervision), avec une logique d’exploitation réaliste de type service desk.

Ce dossier reprend les éléments de la documentation serveur GLPI, de la fiche RP E5, et des livrables techniques du dossier RP03, dans un format de rapport unique, structuré et soutenable pour la soutenance.

---

## SOMMAIRE

1. Contexte et enjeux  
2. Objectifs, périmètre et compétences mobilisées  
3. Architecture de la solution GLPI  
4. Mise en œuvre technique  
5. Paramétrage helpdesk et intégration annuaire  
6. Sécurisation, exploitation et maintenance  
7. Bilan professionnel et perspectives  
Bibliographie  
Annexes

---

## 1. Contexte et enjeux

Avant la mise en place de GLPI, les demandes d’assistance étaient traitées de manière informelle (oral, messages), ce qui entraînait :

- perte de demandes ;
- absence de traçabilité ;
- faible visibilité sur la charge de support ;
- absence de priorisation et d’historique exploitable.

Le projet vise donc à industrialiser le support en appliquant un cycle de ticket standard :

1. Déclaration de l’incident par l’utilisateur  
2. Qualification (catégorie, urgence, priorité)  
3. Affectation à un technicien  
4. Traitement avec suivi  
5. Résolution puis clôture

Cette démarche aligne le fonctionnement de la section BTS sur les pratiques professionnelles de support informatique.

---

## 2. Objectifs, périmètre et compétences mobilisées

### 2.1 Objectifs opérationnels

- Déployer une plateforme GLPI fonctionnelle sur serveur Linux.
- Assurer une persistance des données (tickets, utilisateurs, configuration).
- Structurer les rôles et le workflow de traitement des tickets.
- Intégrer l’authentification à l’annuaire Active Directory.
- Produire les livrables techniques et utilisateur pour exploitation.

### 2.2 Périmètre RP

- **Service principal :** GLPI (ticketing / helpdesk)  
- **Services d’écosystème :** reverse proxy Traefik, base MariaDB, annuaire AD, supervision (Grafana/Loki/Prometheus), documentation Wiki.

### 2.3 Compétences BTS SISR mobilisées

- Concevoir une solution d’infrastructure réseau  
- Installer, tester et déployer une solution  
- Exploiter, dépanner et superviser la solution

### 2.4 Ressources mobilisées

- **Matériel :** serveur Dell PowerEdge, équipements Cisco (routeur/switch/AP)  
- **Logiciels :** Docker, Docker Compose, GLPI, MariaDB, Traefik  
- **Ressources documentaires :** documentation officielle GLPI et documentation interne RP03

---

## 3. Architecture de la solution GLPI

### 3.1 Positionnement dans l’architecture

GLPI est hébergé dans le VLAN serveurs et exposé via Traefik.  
L’authentification des utilisateurs est déléguée à l’Active Directory pour éviter la gestion de comptes locaux multiples.

### 3.2 Composants techniques

| Composant | Rôle |
|:---|:---|
| GLPI (conteneur applicatif) | Interface helpdesk, gestion tickets et profils |
| MariaDB (conteneur base) | Stockage des tickets, entités, profils, historiques |
| Traefik | Publication centralisée et routage des services |
| Active Directory (LDAP) | Authentification et mapping des rôles |

### 3.3 Principes d’architecture retenus

- conteneurisation pour portabilité et maintenance ;
- isolation des services et séparation des responsabilités ;
- persistance via volumes Docker ;
- centralisation des accès via reverse proxy ;
- alignement des droits applicatifs sur les groupes annuaire.

---

## 4. Mise en œuvre technique

### 4.1 Déploiement de la stack

Le déploiement est réalisé via Docker Compose, avec un service GLPI et un service MariaDB, puis raccordement au réseau partagé de publication.

Le principe de démarrage :

1. Préparer les dossiers de volumes persistants  
2. Définir les variables sensibles dans un fichier d’environnement  
3. Démarrer la stack en mode détaché  
4. Vérifier la disponibilité applicative et la connectivité base

### 4.2 Initialisation GLPI

Après démarrage :

- initialisation via assistant d’installation ;
- configuration de la connexion base de données ;
- création de l’instance ;
- sécurisation immédiate des comptes par défaut.

### 4.3 Paramètres système importants

- fuseau horaire et paramètres régionaux ;
- configuration des notifications ;
- définition des entités/profils ;
- suppression des artefacts d’installation.

---

## 5. Paramétrage helpdesk et intégration annuaire

### 5.1 Modèle de rôles

Trois profils fonctionnels sont mis en place :

- **Utilisateur final** : création/suivi des tickets ;  
- **Technicien** : traitement, suivi, résolution ;  
- **Administrateur** : gouvernance de l’outil (règles, profils, paramètres).

### 5.2 Catégorisation et priorisation

Le catalogue de tickets est organisé pour faciliter le tri et l’assignation :

- incidents réseau ;  
- incidents d’authentification ;  
- demandes d’installation ;  
- autres demandes.

La priorité repose sur l’impact et l’urgence, permettant d’ordonner les interventions.

### 5.3 Workflow de cycle de vie

Cycle appliqué dans GLPI :

**Nouveau → Attribué / En cours → En attente → Résolu → Clos**

Ce workflow garantit la traçabilité complète et la qualité de suivi.

### 5.4 Intégration LDAP / Active Directory

L’annuaire est configuré pour :

- authentifier les utilisateurs sur compte institutionnel ;
- récupérer les attributs d’identité utiles ;
- appliquer un mapping groupes annuaire → rôles GLPI.

Bénéfices :

- réduction du risque lié aux comptes locaux ;
- cohérence des droits ;
- administration simplifiée.

---

## 6. Sécurisation, exploitation et maintenance

### 6.1 Mesures de sécurisation appliquées

- gestion des secrets hors code ;
- durcissement des comptes applicatifs ;
- segmentation réseau et exposition maîtrisée ;
- principe de moindre privilège sur les rôles.

### 6.2 Exploitation quotidienne

- supervision de disponibilité du service ;
- suivi de charge et des tickets récurrents ;
- contrôle des files d’attente et des délais de traitement ;
- revue des incidents pour amélioration continue.

### 6.3 Sauvegarde et continuité

Stratégie recommandée :

- sauvegardes régulières base + volumes ;
- tests de restauration ;
- procédure de reprise documentée.

---

## 7. Bilan professionnel et perspectives

### 7.1 Acquis techniques

- déploiement d’un service métier en environnement conteneurisé ;
- structuration d’un processus de support réaliste ;
- intégration annuaire pour authentification centralisée ;
- formalisation de livrables exploitables en contexte réel.

### 7.2 Valeur ajoutée pour l’établissement

- support centralisé et traçable ;
- meilleure communication utilisateurs/techniciens ;
- historique exploitable pour piloter l’amélioration continue ;
- base solide pour les futurs projets d’infrastructure.

### 7.3 Pistes d’évolution

- enrichissement du portail utilisateur ;
- tableaux de bord avancés de pilotage ;
- automatisation renforcée des règles d’assignation ;
- extension des scénarios de supervision et d’alerting.

---

## Bibliographie

- Documentation GLPI Serveur (référence projet)  
- Fiche descriptive RP GLPI (annexe E5)  
- Documentation officielle GLPI  
- Livrables techniques RP03 (L1 à L9)

---

## Annexes

| Référence | Titre |
|:---|:---|
| L1 | Schéma d’architecture RP03 |
| L2 | Documentation GLPI + LDAP |
| L3 | Documentation Nextcloud + LDAP |
| L4 | Documentation Wiki Outline + LDAP |
| L5 | Documentation Stack LGP Monitoring |
| L6 | Documentation Traefik HTTPS |
| L7 | Guide utilisateur GLPI |
| L8 | Procédure d’intégration de service dans Traefik |
