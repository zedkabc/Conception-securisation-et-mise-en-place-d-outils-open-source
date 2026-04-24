# L7 — Guide Utilisateur GLPI
**Projet :** RP-03 — Déploiement d'outils open source pour CFA IRIS Nice  
**Service :** GLPI (Helpdesk & Ticketing)  
**Public :** Étudiants et Enseignants de l'école IRIS Nice  
**Auteur :** Louka Lavenir  
**Date :** 20 mars 2026

---

##  Introduction

Bienvenue sur **GLPI**, votre nouveau système de helpdesk pour déclarer et suivre vos demandes d'assistance informatique.
Le traitement des tickets suit une logique **ITIL** : qualification, priorisation, affectation, résolution et clôture.

**Pourquoi GLPI ?**
-  **Plus aucune demande perdue** — Tout est tracé dans un ticket
-  **Suivi en temps réel** — Vous savez où en est votre demande
-  **Priorisation** — Les urgences sont traitées en premier
-  **Historique complet** — Toutes les interventions sont enregistrées

---

##  Se Connecter à GLPI

### URL d'accès

**Adresse :** https://glpi.iris.a3n.fr:4433

### Identifiants

**Utilisez votre compte LDAP habituel :**
- **Identifiant :** Votre login Windows (ex : `j.dupont`)
- **Mot de passe :** Votre mot de passe Windows

**Pas de nouveau compte à créer !** Si vous pouvez vous connecter à votre PC, vous pouvez vous connecter à GLPI.

---

##  Interface Simplifiée (Étudiants)

Quand vous vous connectez avec un compte étudiant, vous voyez **l'interface simplifiée** (Self-Service).

**Menu principal :**
-  **Créer un ticket** — Déclarer un problème ou une demande
-  **Mes tickets** — Voir tous vos tickets en cours
-  **Notifications** — Recevoir les mises à jour par email

---

##  Créer un Ticket (Déclarer un Problème)

### Étape 1 — Accéder au formulaire

**Cliquer sur :**  **Créer un ticket**

### Étape 2 — Remplir le formulaire

**1. Titre**
- Description courte du problème (1 ligne)
- **Exemples :**
  - "Impossible de me connecter au WiFi"
  - "Mot de passe oublié"
  - "PC ne démarre plus"

**2. Catégorie**
- Choisir la catégorie qui correspond à votre problème :
  -  **Problème réseau** (WiFi, connexion filaire, lenteur)
  -  **Problème login / compte** (mot de passe, connexion, droits)
  -  **Problème matériel** (PC, écran, clavier, imprimante)
  -  **Demande d'installation** (logiciel, accès à un service)
  -  **Sécurité** (faille, virus, accès non autorisé)
  -  **Autre** (si aucune catégorie ne correspond)

**3. Urgence**
- Choisir le niveau d'urgence :
  -  **Très haute** — Problème bloquant (impossible de travailler)
  -  **Haute** — Problème important mais contournable
  -  **Moyenne** — Problème gênant (par défaut)
  -  **Basse** — Problème mineur
  -  **Très basse** — Simple question ou demande d'info

**4. Description**
- Décrire le problème en détail :
  - **Que se passe-t-il ?** (symptôme)
  - **Depuis quand ?** (date/heure)
  - **Avez-vous fait des manipulations ?** (ce que vous avez essayé)
  - **Message d'erreur ?** (si applicable)

**Exemple de bonne description :**
```
Depuis ce matin 9h, je ne peux plus me connecter au WiFi sur mon PC portable.
J'ai essayé de redémarrer mon PC mais ça ne fonctionne toujours pas.
Message d'erreur : "Impossible de se connecter au réseau IRIS-Student"
Mon téléphone se connecte sans problème au même WiFi.
```

**5. Pièces jointes (optionnel)**
- Vous pouvez joindre des captures d'écran, logs, etc.
- **Cliquer sur :**  **Ajouter un fichier**

### Étape 3 — Soumettre le ticket

**Cliquer sur :**  **Soumettre le ticket**

** Confirmation par email**
Vous recevrez un email de confirmation avec :
- Le numéro de votre ticket
- Un lien pour suivre son évolution

---

##  Suivre Mes Tickets

### Voir la liste de mes tickets

**Cliquer sur :**  **Mes tickets**

**Vous voyez :**
-  **Nouveau** — Ticket créé, en attente d'assignation
-  **En cours** — Un technicien s'en occupe
-  **En attente** — Bloqué (en attente de votre réponse ou d'une ressource)
-  **Résolu** — Solution appliquée, en attente de votre validation
-  **Clos** — Ticket terminé et validé

### Voir les détails d'un ticket

**Cliquer sur le numéro du ticket** (ex : #12345)

**Vous voyez :**
- Le détail complet de votre demande
- Les suivis du technicien (ce qu'il a fait, questions posées)
- L'historique de toutes les actions

### Ajouter un suivi (répondre au technicien)

**Si le technicien vous pose une question ou demande des infos :**

1. **Cliquer sur :**  **Ajouter un suivi**
2. Saisir votre message
3. **Cliquer sur :**  **Envoyer**

**Le technicien recevra une notification par email.**

### Valider la résolution

**Quand le ticket passe en statut "Résolu" :**
- Vous recevez un email avec la solution appliquée
- **Si le problème est résolu :** Ne rien faire → Le ticket sera automatiquement clos après 7 jours
- **Si le problème persiste :** Répondre au ticket en ajoutant un suivi → Le ticket repasse en "En cours"

---

##  Comprendre les Niveaux d'Intervention (N1/N2/N3)

Votre ticket est automatiquement routé vers le bon niveau de support selon la catégorie :

###  Niveau 1 (N1) — Support Basique
**Exemples :**
- Mot de passe oublié
- Création de compte
- Demande d'information
- Installation logiciel standard

**Délai de résolution :** 2-4 heures

###  Niveau 2 (N2) — Support Technique
**Exemples :**
- Problème réseau (WiFi, VLAN, IP)
- Problème matériel (PC, écran, imprimante)
- Configuration système

**Délai de résolution :** 24 heures

###  Niveau 3 (N3) — Expertise
**Exemples :**
- Problème de sécurité (faille, virus)
- Bug applicatif
- Évolution / développement
- Incident critique

**Délai de résolution :** 48 heures

** Bon à savoir :** Si le ticket est complexe, il peut être **escaladé** (passé à un niveau supérieur). Vous serez notifié par email.

---

##  Notifications Email

**Vous recevez un email automatiquement quand :**
-  Votre ticket est créé
-  Un technicien est assigné
-  Le technicien ajoute un suivi (message, question)
-  Votre ticket est résolu
-  Votre ticket est clos

** Vérifiez votre boîte email régulièrement !**

---

##  Questions Fréquentes (FAQ)

### Combien de temps avant que mon ticket soit pris en charge ?

**Ça dépend de l'urgence et de la catégorie :**
- Problème bloquant (Très haute urgence) : **< 1 heure**
- Problème important (Haute urgence) : **< 4 heures**
- Problème moyen (Moyenne urgence) : **< 24 heures**
- Demande d'info (Basse urgence) : **< 48 heures**

### Puis-je créer plusieurs tickets en même temps ?

**Oui !** Si vous avez plusieurs problèmes différents, créez **un ticket par problème**.

**Pourquoi ?**
- Chaque ticket est traité indépendamment
- Meilleure traçabilité
- Évite les confusions

### Que faire si mon problème est urgent ?

1. **Créer un ticket** avec urgence "Très haute"
2. **Bien détailler** le problème et l'impact (ex : "Je ne peux pas travailler")
3. **Ajouter une pièce jointe** (capture d'écran, message d'erreur)

**Si c'est VRAIMENT critique (ex : serveur en panne, tout le monde bloqué) :**
- Contacter directement le responsable technique en plus du ticket

### Mon ticket est en attente, que faire ?

**Statut "En attente" signifie :**
- Le technicien attend une information de votre part → **Répondre au ticket**
- Le technicien attend une ressource externe (validation, matériel) → **Patienter**

**Vérifiez les suivis** pour voir ce qui est attendu.

### Puis-je annuler un ticket ?

**Oui, si le problème est résolu de lui-même ou n'est plus d'actualité :**
1. Ouvrir le ticket
2. Ajouter un suivi : "Problème résolu, vous pouvez clore le ticket"
3. Le technicien clôturera le ticket

---

##  Bonnes Pratiques

###  À FAIRE

- **Donner un titre clair** — "Impossible de me connecter au WiFi" plutôt que "Problème"
- **Choisir la bonne catégorie** — Aide le routage automatique
- **Décrire précisément** — Symptôme, contexte, ce que vous avez essayé
- **Ajouter des captures d'écran** — Une image vaut mille mots
- **Répondre rapidement** — Si le technicien vous pose une question
- **Valider la résolution** — Confirmer que le problème est résolu

###  À ÉVITER

- **Titre vague** — "Ça marche pas", "Problème", "Urgent"
- **Pas de détails** — "Mon PC ne fonctionne pas" (quoi exactement ?)
- **Regrouper plusieurs problèmes** — Créer un ticket par problème
- **Oublier de répondre** — Si le technicien pose une question
- **Créer des doublons** — Un seul ticket par problème

---

##  Besoin d'Aide ?

**Si vous avez des difficultés avec GLPI :**
-  **Email :** louka.lavenir@mediaschool.me
-  **Créer un ticket** avec catégorie "Autre" et titre "Problème avec GLPI"

---

##  Résumé en 5 Étapes

1. **Se connecter** — https://glpi.iris.a3n.fr:4433 avec votre compte Windows
2. **Créer un ticket** — Titre, catégorie, urgence, description détaillée
3. **Recevoir confirmation** — Email avec numéro de ticket
4. **Suivre l'évolution** — Menu "Mes tickets"
5. **Valider la résolution** — Confirmer que le problème est résolu (ou relancer si besoin)

---

** Bon usage de GLPI et bon courage dans vos études !**

---

**Auteur :** Louka Lavenir  
**Support IRIS Nice**  
**Date :** 20 mars 2026  
**Version :** 1.0


