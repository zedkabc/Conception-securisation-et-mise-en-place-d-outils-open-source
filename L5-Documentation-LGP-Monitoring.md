# L5 — Documentation Stack LGP (Loki-Grafana-Prometheus) — Service Secondaire
**Projet :** RP-03 — Déploiement d'outils open source pour CFA IRIS Nice  
**Service :** LGP Stack (Monitoring infrastructure)  
**Statut :** Service secondaire (travail de Vincent)  
**Auteur :** Vincent (repris par Louka Lavenir pour documentation)  
**Date :** 20 mars 2026

---

## 1. Introduction

La stack LGP (Loki-Grafana-Prometheus) est une solution open source de monitoring et d'observabilité infrastructure.

**Dans le cadre du projet RP-03, LGP constitue un service secondaire** permettant de superviser l'état de l'infrastructure IRIS Nice en temps réel (CPU, RAM, disque, réseau, services, équipements Cisco).

---

## 2. Pourquoi la Stack LGP ?

**Benchmark complet réalisé par Vincent :**

### 2.1 Métriques : Collecte et Stockage (TSDB)

| Technologie | Type | Points Forts | Points Faibles |
|:---|:---|:---|:---|
| **Prometheus (Choisi)** | Open Source | Auto-discovery Docker, requêtes PromQL puissantes | Courbe d'apprentissage du langage |
| **Zabbix** | Open Source | Excellente gestion SNMP et parc hardware | Difficile à automatiser via Ansible |
| **Datadog** | SaaS / Cloud | Zéro maintenance, UI parfaite | Coût élevé, données hors site |
| **Netdata** | Open Source | Visualisation immédiate, ultra-précis | Historique court, stockage RAM lourd |

### 2.2 Gestion des Logs : Centralisation et Analyse

| Technologie | Type | Points Forts | Points Faibles |
|:---|:---|:---|:---|
| **Loki + Promtail (Choisi)** | Open Source | Empreinte RAM minimale, corrélation native | Pas de recherche "plein texte" complexe |
| **ELK (Elasticsearch)** | Open Source | Puissance de recherche absolue | Extrêmement lourd (Java/JVM) |
| **Splunk** | Propriétaire | Capacité d'analyse massive | Prix prohibitif (licence au volume) |
| **Graylog** | Open Source | Interface de gestion très intuitive | Moins performant pour lier logs et métriques |

### 2.3 Visualisation : Dashboards et Corrélation

| Technologie | Type | Points Forts | Points Faibles |
|:---|:---|:---|:---|
| **Grafana (Choisi)** | Open Source | Multi-source, dashboards dynamiques | Demande de l'entraînement pour le design |
| **Kibana** | Open Source | Analyse de logs textuels poussée | Incompatible avec Prometheus |
| **Chronograf** | Open Source | Simple pour les séries temporelles | Écosystème plus fermé que Grafana |

### 2.4 Synthèse des Décisions

| Critère | Stack LGP | Alternatives (Moyenne) |
|:---|:---|:---|
| **Sécurité / Souveraineté** | Maximale (Auto-hébergé, 100% local) | Risquée (Si stockage Cloud/SaaS) |
| **Coût de licence** | 0 € (Totalement Open Source) | Variable (Souvent facturé au volume) |
| **Automatisation Ansible** | Native (Configuration via YAML) | Partielle (Souvent via interface web) |
| **Consommation RAM** | Faible (Environ 1 Go pour tout le stack) | Élevée (6 Go+ pour ELK ou Zabbix) |

**Verdict :** La stack LGP est la solution la plus robuste et la plus professionnelle pour valider les compétences DevOps de ce projet, avec **0€ de coût** et **souveraineté totale des données**.

---

## 3. Installation LGP avec Docker Compose

### 3.1 Structure des fichiers

```
/opt/iris-services/monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   ├── alerts.yml
│   └── targets/
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       └── dashboards/
└── volumes/
    ├── prometheus-data/
    ├── loki-data/
    ├── grafana-data/
    └── alertmanager-data/
```

### 3.2 Fichier docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    container_name: prometheus
    image: prom/prometheus:latest
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml
      - ./volumes/prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
    networks:
      - monitoring-network

  loki:
    container_name: loki
    image: grafana/loki:latest
    restart: always
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - ./volumes/loki-data:/loki
    networks:
      - monitoring-network

  promtail:
    container_name: promtail
    image: grafana/promtail:latest
    restart: always
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring-network

  grafana:
    container_name: grafana
    image: grafana/grafana:latest
    restart: always
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin_password_secure
      GF_SERVER_ROOT_URL: https://grafana.iris.a3n.fr:4433
      GF_AUTH_LDAP_ENABLED: "true"
      GF_AUTH_LDAP_CONFIG_FILE: /etc/grafana/ldap.toml
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./volumes/grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
      - loki
    networks:
      - monitoring-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.iris.a3n.fr`)"
      - "traefik.http.routers.grafana.entrypoints=web"
      - "traefik.http.routers.grafana.tls=true"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

  node-exporter:
    container_name: node-exporter
    image: prom/node-exporter:latest
    restart: always
    ports:
      - "9100:9100"
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - monitoring-network

  blackbox-exporter:
    container_name: blackbox-exporter
    image: prom/blackbox-exporter:latest
    restart: always
    ports:
      - "9115:9115"
    volumes:
      - ./prometheus/blackbox.yml:/etc/blackbox_exporter/config.yml
    networks:
      - monitoring-network

  alertmanager:
    container_name: alertmanager
    image: prom/alertmanager:latest
    restart: always
    ports:
      - "9093:9093"
    volumes:
      - ./prometheus/alertmanager.yml:/etc/alertmanager/config.yml
      - ./volumes/alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring-network

networks:
  monitoring-network:
    driver: bridge
```

---

## 4. Configuration Prometheus

### 4.1 Fichier prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - '/etc/prometheus/alerts.yml'

scrape_configs:
  # Métriques Prometheus lui-même
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Métriques serveur Linux (Node Exporter)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Tests HTTP/HTTP (Blackbox Exporter)
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://glpi.iris.a3n.fr:4433
          - https://cloud.iris.a3n.fr:4433
          - https://wiki.iris.a3n.fr:4433
          - https://grafana.iris.a3n.fr:4433
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # SNMP pour équipements Cisco (optionnel, nécessite snmp_exporter)
  # - job_name: 'cisco-switches'
  #   static_configs:
  #     - targets:
  #         - 10.10.10.1  # Switch Catalyst 2960-S
  #         - 10.10.10.2  # Routeur Cisco 1941
  #   metrics_path: /snmp
  #   params:
  #     module: [if_mib]
  #   relabel_configs:
  #     - source_labels: [__address__]
  #       target_label: __param_target
  #     - target_label: __address__
  #       replacement: snmp-exporter:9116
```

### 4.2 Fichier alerts.yml

```yaml
groups:
  - name: infrastructure_alerts
    interval: 30s
    rules:
      # Alerte CPU > 90%
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU élevé sur {{ $labels.instance }}"
          description: "CPU à {{ $value }}% sur {{ $labels.instance }}"

      # Alerte RAM > 90%
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAM élevée sur {{ $labels.instance }}"
          description: "RAM utilisée à {{ $value }}% sur {{ $labels.instance }}"

      # Alerte Disque > 85%
      - alert: HighDiskUsage
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disque plein sur {{ $labels.instance }}"
          description: "Disque {{ $labels.mountpoint }} à {{ $value }}%"

      # Alerte Service Down
      - alert: ServiceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} down"
          description: "Le service {{ $labels.job }} ne répond pas"

      # Alerte HTTP/HTTP Service Down
      - alert: HTTPServiceDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service web {{ $labels.instance }} inaccessible"
          description: "{{ $labels.instance }} ne répond pas (HTTP)"

      # Alerte Conteneur Docker redémarrant en boucle
      - alert: ContainerRestarting
        expr: increase(container_restart_count[1h]) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Conteneur {{ $labels.name }} redémarre en boucle"
          description: "Le conteneur {{ $labels.name }} a redémarré {{ $value }} fois en 1h"

      # Alerte Certificat SSL expirant (à activer quand HTTP déployé)
      # - alert: SSLCertExpiring
      #   expr: probe_ssl_earliest_cert_expiry - time() < 604800
      #   for: 1h
      #   labels:
      #     severity: warning
      #   annotations:
      #     summary: "Certificat SSL {{ $labels.instance }} expire bientôt"
      #     description: "Le certificat expire dans moins de 7 jours"
```

** Note sur les alertes :**
- **HighCPUUsage** : Se déclenche après 5 minutes de CPU > 90% (évite les faux positifs lors de pics courts)
- **HighMemoryUsage** : Ajoutée pour superviser la RAM (conteneurs Docker peuvent fuir de la mémoire)
- **ContainerRestarting** : Détecte les conteneurs instables (plus de 3 redémarrages en 1 heure)
- **SSLCertExpiring** : Préparée mais commentée — à activer quand HTTP sera déployé

---

## 5. Configuration Loki (Centralisation des Logs)

### 5.1 Fichier loki-config.yml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

# Rétention des logs
limits_config:
  retention_period: 720h    # 30 jours pour la maquette
  # En production : 8760h (12 mois) pour conformité LCEN (Loi pour la Confiance dans l'Économie Numérique)
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20

# Stockage et compaction
storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: filesystem
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

** Politique de rétention :**
- **Maquette (RP-03)** : 30 jours (720h) — suffisant pour valider le fonctionnement et diagnostiquer les incidents
- **Production recommandée** : 12 mois (8760h) — conformité LCEN (conservation des logs d'accès et d'authentification)
- **Compaction automatique** : Les logs au-delà de la période de rétention sont supprimés automatiquement toutes les 2 heures

### 5.2 Centralisation des logs Cisco (optionnel)

**Objectif :** Collecter les logs des équipements Cisco (switches, routeur) via syslog et les envoyer vers Loki.

**Option A : Via rsyslog (relai vers Loki)**

1. **Configurer rsyslog sur le serveur Docker** :

```bash
# /etc/rsyslog.d/50-cisco.conf
# Recevoir les logs Cisco en UDP 514
module(load="imudp")
input(type="imudp" port="514")

# Filtrer et envoyer vers fichier
if $fromhost-ip startswith '10.10.10.' then {
    action(type="omfile" file="/var/log/cisco/cisco.log")
}
```

2. **Configurer Promtail pour collecter ce fichier** (déjà configuré dans section 3.2)

3. **Configurer les équipements Cisco pour envoyer les logs** :

```cisco
! Sur Switch Catalyst 2960-S
enable
configure terminal
logging host 10.10.10.X       ! IP du serveur Docker
logging trap informational    ! Niveau de log
logging facility local6
exit
write memory

! Sur Routeur Cisco 1941
enable
configure terminal
logging host 10.10.10.X
logging trap informational
exit
write memory
```

**Option B : Via syslog-ng (filtrage avancé)**

Pour un filtrage plus fin (par type d'événement Cisco), utiliser syslog-ng au lieu de rsyslog.

### 5.3 Dashboard Grafana "Logs Applicatifs"

**Création du dashboard :**

1. **Accéder à Grafana** → https://grafana.iris.a3n.fr:4433
2. **Dashboards** → New Dashboard → Add visualization
3. **Datasource** → Loki
4. **Requête LogQL** :

```logql
# Logs de tous les conteneurs Docker
{job="docker"}

# Logs d'un service spécifique (ex : GLPI)
{container_name="glpi"}

# Logs d'erreur uniquement (tous services)
{job="docker"} |= "error" or |= "ERROR" or |= "Error"

# Logs avec filtrage par niveau (Loki détecte automatiquement les patterns)
{job="docker"} | json | level="error"

# Logs Cisco (si syslog configuré)
{job="syslog", source="cisco"}
```

5. **Filtres interactifs** :
   - Par service (glpi, nextcloud, outline, grafana, traefik)
   - Par niveau (info, warning, error, critical)
   - Par plage horaire (dernière heure, dernier jour, dernière semaine)

**Panels recommandés :**
- **Top 10 erreurs** (table avec compteur)
- **Timeline des logs** (graphique temporel)
- **Logs bruts** (logs stream)

---

## 6. Configuration Grafana

### 5.1 Configuration LDAP (OpenLDAP)

**Fichier `grafana/ldap.toml` :**

```toml
[[servers]]
host = "openldap"
port = 389
use_ssl = false
start_tls = false
ssl_skip_verify = false

bind_dn = "cn=admin,dc=mediaschool,dc=local"
bind_password = '[mot_de_passe_service_ldap]'

search_filter = "(ui=%s)"
search_base_dns = ["dc=mediaschool,dc=local"]

[servers.attributes]
name = "givenName"
surname = "sn"
username = "ui"
member_of = "memberOf"
email = "mail"

# Mapping groupes AD → Rôles Grafana
[[servers.group_mappings]]
group_dn = "CN=Admins,OU=Groups,dc=mediaschool,dc=local"
org_role = "Admin"

[[servers.group_mappings]]
group_dn = "CN=Enseignants,OU=Groups,dc=mediaschool,dc=local"
org_role = "Editor"

[[servers.group_mappings]]
group_dn = "CN=Etudiants,OU=Groups,dc=mediaschool,dc=local"
org_role = "Viewer"
```

### 6.2 Dashboards Pré-configurés

**Dashboards disponibles :**
1. **Vue d'ensemble infrastructure** (CPU, RAM, Disque, Réseau)
2. **État des services** (GLPI, Nextcloud, Outline up/down)
3. **Logs applicatifs** (via Loki)
4. **Performances réseau** (SNMP Cisco, si configuré)

---

## 7. Tests de Validation

- [ ] Accès Grafana avec compte AD → rôle selon groupe
- [ ] Métriques CPU/RAM/Disque visibles (Node Exporter)
- [ ] Tests HTTP (Blackbox Exporter) → services up/down
- [ ] Alertes configurées (CPU > 90%, Service down)
- [ ] Logs centralisés (Loki + Promtail)
- [ ] Dashboards exploitables par responsable technique

---

**Auteur :** Vincent (repris par Louka Lavenir)  
**Date :** 20 mars 2026  
**Version :** 1.0


