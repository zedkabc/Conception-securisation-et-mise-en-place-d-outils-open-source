# IRIS RP-03 — Livrables

**Projet :** Déploiement d’outils open source pour CFA IRIS Nice  
**Auteur :** Louka Lavenir

## Accès services

Tous les services sont publiés en **HTTPS** sur le **port 4433**.

| Service | URL |
|:---|:---|
| GLPI | https://glpi.iris.a3n.fr:4433/ |
| Nextcloud | https://cloud.iris.a3n.fr:4433/ |
| Wiki | https://wiki.iris.a3n.fr:4433/ |
| Grafana | https://grafana.iris.a3n.fr:4433/ |
| Traefik | https://traefik.iris.a3n.fr:4433/ |

## Référentiel LDAP (GLPI)

- **Nom :** IRIS Mediaschool
- **Serveur par défaut :** Oui
- **Actif :** Oui
- **Serveur :** openldap
- **Port :** 389
- **Filtre de connexion :** `(objectClass=inetOrgPerson)`
- **BaseDN :** `dc=mediaschool,dc=local`
- **Utiliser bind :** Oui
- **DN du compte :** `cn=admin,dc=mediaschool,dc=local`
- **Champ de l’identifiant :** `ui`

## Démarche exploitation

Le workflow ticketing GLPI est aligné sur les bonnes pratiques **ITIL** : qualification (incident/demande), priorisation, affectation N1/N2/N3, suivi et clôture.

## Livrables disponibles

- L1 — Architecture RP03
- L2 — Documentation GLPI + LDAP
- L3 — Documentation Nextcloud + LDAP
- L4 — Documentation Wiki + LDAP
- L5 — Documentation LGP Monitoring
- L6 — Documentation Traefik + HTTPS
- L7 — Guide utilisateur GLPI
- L8 — Procédure d’intégration service Traefik
