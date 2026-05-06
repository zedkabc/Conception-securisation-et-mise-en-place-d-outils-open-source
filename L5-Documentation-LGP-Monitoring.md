# L5 — Documentation Stack LGP (Loki-Grafana-Prometheus) — Service Secondaire
**Projet :** RP-03 — Déploiement d'outils open source pour IRIS Mediaschool Nice  
**Service :** LGP Stack (Monitoring infrastructure)  
**Statut :** Service secondaire (travail de Vincent ANDREO)  
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
/home/iris/sisr/monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
├── loki/
│   └── chunks
├── promtail/
│   └── config.yml
├── grafana/
│   └── conf
│       └── provisioning/
│           ├── datasources/
│           └── dashboards/
```

### 3.2 Fichier docker-compose.yml

```yaml
version: "3.8"

services:

  cadvisor:
    hostname: cadvisor
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    command:
      - '--docker_only=true'
      - '--housekeeping_interval=1s'
    restart: always
    networks:
      internal:
        ipv4_address: 10.14.7.10
    ports:
      - "8080:8080"

  nodeexporter:
    hostname: nodeexporter
    image: prom/node-exporter:latest
    container_name: nodeexporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    restart: always
    networks:
      internal:
        ipv4_address: 10.14.7.20
    ports:
      - "9100:9100"

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    hostname: prometheus
    volumes:
      - ./prometheus/:/etc/prometheus/
    networks:
      internal:
        ipv4_address: 10.14.7.30
    restart: always
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    hostname: grafana
    volumes:
      - grafana-storage:/var/lib/grafana
      - grafana-conf:/usr/share/grafana
    depends_on:
      - prometheus
    networks:
      internal:
        ipv4_address: 10.14.7.40
    restart: unless-stopped
    ports:
      - "3000:3000"

  loki:
    hostname: loki
    image: grafana/loki:2.6.1
    container_name: loki
    volumes:
      - ./config/loki.yaml:/etc/config/loki.yaml
      - ./loki/chunks:/loki/chunks
      - ./loki/rules:/loki/rules
      - ./loki/rules-storage:/home/loki/rules-storage
    entrypoint:
      - /usr/bin/loki
      - -config.file=/etc/config/loki.yaml
    networks:
      internal:
        ipv4_address: 10.14.7.50
    restart: unless-stopped
    ports:
      - "3100:3100"

  promtail:
    hostname: promtail
    image: grafana/promtail:2.6.1
    container_name: promtail
    volumes:
      - /var/log:/var/log
      - ./promtail/config.yml:/etc/promtail/config.yml
    command:
      - -config.file=/etc/promtail/config.yml
    networks:
      internal:
        ipv4_address: 10.14.7.60
    restart: always

  blackbox:
    hostname: blackbox
    image: prom/blackbox-exporter:v0.25.0
    container_name: blackbox
    volumes:
      - ./blackbox/config.yml:/opt/bitnami/blackbox-exporter/blackbox.yml
    networks:
      internal:
        ipv4_address: 10.14.7.70
    restart: always
    ports:
      - "9115:9115"

  alertmanager:
    hostname: alertmanager
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ./alertmanager/:/etc/alertmanager/
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    networks:
      internal:
        ipv4_address: 10.14.7.80
    restart: unless-stopped
    ports:
      - "9093:9093"

volumes:
  grafana-storage:
    external: true

  grafana-conf:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/iris/sisr/monitoring/grafana-conf

networks:
  internal:
    driver: bridge
    ipam:
      config:
        - subnet: "10.14.7.0/24"
```

---

## 4. Configuration Prometheus

### 4.1 Fichier prometheus.yml

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alerts:9093"]
      scheme: http
      timeout: 10s

scrape_configs:
  - job_name: cadvisor
    scrape_interval: 5s
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: node-exporter
    scrape_interval: 5s
    static_configs:
      - targets: ["nodeexporter:9100"]

  - job_name: blackbox
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets: ["http://example.com"]  # Remplace par ton URL cible
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115  # Adresse du blackbox exporter

  - job_name: blackbox_exporter
    static_configs:
      - targets: ["blackbox:9115"]
```

### 4.2 Fichier node_alerts.yml

```yaml
groups:

  # ── Disponibilite services (hors snmp) ────────────────────────────────────
  - name: services
    rules:
      - alert: ServiceDown
        expr: up{job!~"snmp.*"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} inaccessible"
          description: "L'instance {{ $labels.instance }} est DOWN depuis plus de 2 minutes."

  # ── CPU ───────────────────────────────────────────────────────────────────
  - name: cpu
    rules:
      - alert: CPUHighUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU eleve sur {{ $labels.instance }}"
          description: "Utilisation CPU > 85% depuis 5 min (valeur: {{ $value | printf \"%.1f\" }}%)"

      - alert: CPUCritical
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CPU CRITIQUE sur {{ $labels.instance }}"
          description: "Utilisation CPU > 95% depuis 2 min (valeur: {{ $value | printf \"%.1f\" }}%)"

  # ── RAM ───────────────────────────────────────────────────────────────────
  - name: memory
    rules:
      - alert: MemoryHighUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAM elevee sur {{ $labels.instance }}"
          description: "Utilisation RAM > 85% (valeur: {{ $value | printf \"%.1f\" }}%)"

      - alert: MemoryCritical
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "RAM CRITIQUE sur {{ $labels.instance }}"
          description: "Utilisation RAM > 95% (valeur: {{ $value | printf \"%.1f\" }}%)"

  # ── Disque ────────────────────────────────────────────────────────────────
  - name: disk
    rules:
      - alert: DiskSpaceWarning
        expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"})) * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disque plein a {{ $value | printf \"%.1f\" }}% sur {{ $labels.instance }}"
          description: "Partition {{ $labels.mountpoint }} utilisee a {{ $value | printf \"%.1f\" }}%"

      - alert: DiskSpaceCritical
        expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"})) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "DISQUE PRESQUE PLEIN sur {{ $labels.instance }}"
          description: "Partition {{ $labels.mountpoint }} utilisee a {{ $value | printf \"%.1f\" }}%"

  # ── Docker conteneurs (hors cadvisor lui-meme) ────────────────────────────
  - name: docker
    rules:
      - alert: ContainerDown
        expr: absent_over_time(container_memory_usage_bytes{name!="",name!~"POD|k8s_.*"}[5m])
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Conteneur Docker arrete : {{ $labels.name }}"
          description: "Le conteneur {{ $labels.name }} n'est plus actif depuis 5 minutes."

      - alert: ContainerCPUHigh
        expr: rate(container_cpu_usage_seconds_total{name!="",name!="cadvisor",name!~"POD|k8s_.*"}[5m]) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU eleve conteneur {{ $labels.name }}"
          description: "Le conteneur {{ $labels.name }} consomme {{ $value | printf \"%.1f\" }}% CPU depuis 5 min."

  # ── HTTP / Blackbox (conteneurs internes uniquement) ──────────────────────
  - name: http
    rules:
      - alert: HTTPEndpointDown
        expr: probe_success{job="blackbox"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint HTTP inaccessible : {{ $labels.instance }}"
          description: "La sonde HTTP sur {{ $labels.instance }} echoue depuis 5 minutes."
```

** Note sur les alertes :**
- **HighCPUUsage** : Se déclenche après 5 minutes de CPU > 90% (évite les faux positifs lors de pics courts)
- **HighMemoryUsage** : Ajoutée pour superviser la RAM (conteneurs Docker peuvent fuir de la mémoire)
- **ContainerRestarting** : Détecte les conteneurs instables (plus de 3 redémarrages en 1 heure)

---

## 5. Configuration Loki (Centralisation des Logs)

### 5.1 Fichier loki.yaml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /home/iris/sisr/monitoring/loki
  storage:
    filesystem:
      chunks_directory: /home/iris/sisr/monitoring/loki/chunks
      rules_directory: /home/iris/sisr/monitoring/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

analytics:
  reporting_enabled: false
```

** Politique de rétention :**
- **Maquette (RP-03)** : 30 jours (720h) — suffisant pour valider le fonctionnement et diagnostiquer les incidents
- **Production recommandée** : 12 mois (8760h) — conformité LCEN (conservation des logs d'accès et d'authentification)
- **Compaction automatique** : Les logs au-delà de la période de rétention sont supprimés automatiquement toutes les 2 heures

### 5.2 Dashboard Grafana "Logs Applicatifs"

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

### 6.1 Configuration LDAP (OpenLDAP)

**Fichier `grafana-conf/conf/ldap.toml` :**

```toml
# To troubleshoot and get more log info enable ldap debug logging in grafana.ini
# [log]
# filters = ldap:debug

[[servers]]
# Ldap server host (specify multiple hosts space separated)
host = "127.0.0.1"
# Default port is 389 or 636 if use_ssl = true
port = 389
# Set to true if LDAP server should use an encrypted TLS connection (either with STARTTLS or LDAPS)
use_ssl = false
# If set to true, use LDAP with STARTTLS instead of LDAPS
start_tls = false
# The value of an accepted TLS cipher. By default, this value is empty. Example value: ["TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"])
# For a complete list of supported ciphers and TLS versions, refer to: https://go.dev/src/crypto/tls/cipher_suites.go
# Starting with Grafana v11.0 only ciphers with ECDHE support are accepted for TLS 1.2 connections.
tls_ciphers = []
# This is the minimum TLS version allowed. By default, this value is empty. Accepted values are: TLS1.1 (only for Grafana v10.4 or older), TLS1.2, TLS1.3.
min_tls_version = ""
# set to true if you want to skip ssl cert validation
ssl_skip_verify = false
# set to the path to your root CA certificate or leave unset to use system defaults
# root_ca_cert = "/path/to/certificate.crt"
# Authentication against LDAP servers requiring client certificates
# client_cert = "/path/to/client.crt"
# client_key = "/path/to/client.key"

# Search user bind dn
bind_dn = "cn=admin,dc=mediaschool,dc=local"
# Search user bind password
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
bind_password = 'grafana'
# We recommend using variable expansion for the bind_password, for more info https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#variable-expansion
# bind_password = '$__env{LDAP_BIND_PASSWORD}'

# Timeout in seconds (applies to each host specified in the 'host' entry (space separated))
timeout = 10

# User search filter, for example "(cn=%s)" or "(sAMAccountName=%s)" or "(uid=%s)"
search_filter = "(cn=%s)"

# An array of base dns to search through
search_base_dns = ["dc=mediaschool,dc=local"]

## For Posix or LDAP setups that does not support member_of attribute you can define the below settings
## Please check grafana LDAP docs for examples
# group_search_filter = "(&(objectClass=posixGroup)(memberUid=%s))"
# group_search_base_dns = ["ou=groups,dc=mediaschool,dc=local"]
# group_search_filter_user_attribute = "uid"

# Specify names of the ldap attributes your ldap uses
[servers.attributes]
name = "givenName"
surname = "sn"
username = "cn"
member_of = "memberOf"
email =  "email"

# Map ldap groups to grafana org roles
[[servers.group_mappings]]
group_dn = "cn=admins,ou=groups,dc=mediaschool,dc=local"
org_role = "Admin"
# To make user an instance admin  (Grafana Admin) uncomment line below
# grafana_admin = true
# The Grafana organization database id, optional, if left out the default org (id 1) will be used
# org_id = 1

[[servers.group_mappings]]
group_dn = "cn=admins,ou=groups,dc=mediaschool,dc=local"
org_role = "Editor"

[[servers.group_mappings]]
# If you want to match all (or no ldap groups) then you can use wildcard
group_dn = "*"
org_role = "Viewer"
```

### 6.2 Dashboards Pré-configurés

**Dashboards disponibles :**
1. **Vue d'ensemble infrastructure** (CPU, RAM, Disque, Réseau)
2. **État des services** (GLPI, Nextcloud, Outline up/down)
3. **Logs applicatifs** (via Loki)
4. **Performances réseau** (SNMP Cisco, si configuré)

---

**Auteur :** Vincent (repris par Louka Lavenir)  
**Date :** 20 mars 2026
