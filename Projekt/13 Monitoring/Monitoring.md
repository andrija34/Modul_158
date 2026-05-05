# Auftrag #13 – Monitoring

---

## Beschreibung

In diesem Auftrag wurde ein vollständiges Monitoring-System eingerichtet. Prometheus sammelt Metriken von Webserver, Datenbank, DNS und System. Grafana visualisiert die Metriken in einem Dashboard. E-Mail-Alarmierung wurde konfiguriert. Ein Watchdog-Dienst überwacht kritische Prozesse und startet sie bei Ausfall automatisch neu.

---

## Architektur
crmserver.sample.ch
├── Apache2          → apache_exporter   → Prometheus → Grafana
├── MariaDB          → mysqld_exporter   → Prometheus → Alertmanager
├── System (CPU/RAM) → node_exporter     → Prometheus → E-Mail
└── Watchdog (Systemd) → Auto-Restart bei Ausfall

---

## Schritt 1 – Prometheus installieren

```bash
# Prometheus herunterladen
wget https://github.com/prometheus/prometheus/releases/download/v2.48.0/prometheus-2.48.0.linux-amd64.tar.gz
tar -xzf prometheus-2.48.0.linux-amd64.tar.gz
sudo mv prometheus-2.48.0.linux-amd64 /opt/prometheus

# Benutzer erstellen
sudo useradd -rs /bin/false prometheus

# Systemd Service erstellen
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
    --config.file=/opt/prometheus/prometheus.yml \
    --storage.tsdb.path=/opt/prometheus/data \
    --storage.tsdb.retention.time=30d \
    --web.listen-address=127.0.0.1:9090
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```


---

## Schritt 2 – Exporter installieren

### Node Exporter (System-Metriken)

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter --web.listen-address=127.0.0.1:9100
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now node_exporter
```

### MySQL Exporter (MariaDB-Metriken)

```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.0/mysqld_exporter-0.15.0.linux-amd64.tar.gz
tar -xzf mysqld_exporter-0.15.0.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.15.0.linux-amd64/mysqld_exporter /usr/local/bin/

# Datenbankbenutzer für Exporter
mysql -u root -p -e "
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporterPasswort';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;"

# Credentials Datei
echo "[client]
user=exporter
password=exporterPasswort" | sudo tee /opt/mysqld_exporter/.my.cnf
sudo chmod 600 /opt/mysqld_exporter/.my.cnf

sudo nano /etc/systemd/system/mysqld_exporter.service
```

```ini
[Unit]
Description=MySQL Exporter
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/mysqld_exporter \
    --config.my-cnf=/opt/mysqld_exporter/.my.cnf \
    --web.listen-address=127.0.0.1:9104
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now mysqld_exporter
```

### Apache Exporter

```bash
# mod_status aktivieren
sudo a2enmod status
sudo nano /etc/apache2/mods-enabled/status.conf
```

```apacheconf
<Location /server-status>
    SetHandler server-status
    Require ip 127.0.0.1
</Location>
```

```bash
wget https://github.com/Lusitaniae/apache_exporter/releases/download/v1.0.0/apache_exporter-1.0.0.linux-amd64.tar.gz
tar -xzf apache_exporter-1.0.0.linux-amd64.tar.gz
sudo mv apache_exporter-1.0.0.linux-amd64/apache_exporter /usr/local/bin/

sudo systemctl enable --now apache_exporter
```


---

## Schritt 3 – Prometheus Konfiguration

```bash
sudo nano /opt/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "/opt/prometheus/rules/*.yml"

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['127.0.0.1:9100']
        labels:
          instance: 'crmserver'

  - job_name: 'apache'
    static_configs:
      - targets: ['127.0.0.1:9117']
        labels:
          instance: 'crmserver'

  - job_name: 'mysql'
    static_configs:
      - targets: ['127.0.0.1:9104']
        labels:
          instance: 'crmserver'
```


---

## Schritt 4 – Alert Rules

```bash
sudo mkdir -p /opt/prometheus/rules
sudo nano /opt/prometheus/rules/crm-alerts.yml
```

```yaml
groups:
  - name: crm-alerts
    rules:
      # CPU Auslastung > 80%
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Hohe CPU-Auslastung auf {{ $labels.instance }}"
          description: "CPU > 80% seit 5 Minuten"

      # RAM Auslastung > 85%
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Hohe RAM-Auslastung auf {{ $labels.instance }}"

      # Disk > 80%
      - alert: DiskSpaceLow
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100 > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk-Platz kritisch auf {{ $labels.instance }}"

      # Apache down
      - alert: ApacheDown
        expr: apache_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Apache nicht erreichbar auf {{ $labels.instance }}"

      # MySQL down
      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MariaDB nicht erreichbar auf {{ $labels.instance }}"
```

---

## Schritt 5 – Alertmanager installieren

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-amd64.tar.gz
tar -xzf alertmanager-0.26.0.linux-amd64.tar.gz
sudo mv alertmanager-0.26.0.linux-amd64 /opt/alertmanager

sudo nano /opt/alertmanager/alertmanager.yml
```

```yaml
global:
  smtp_smarthost: 'smtp.sample.ch:587'
  smtp_from: 'monitoring@sample.ch'
  smtp_auth_username: 'monitoring@sample.ch'
  smtp_auth_password: 'smtpPasswort!'
  smtp_require_tls: true

route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'email-admin'

receivers:
  - name: 'email-admin'
    email_configs:
      - to: 'admin@sample.ch'
        html: '{{ template "email.html" . }}'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

```bash
sudo systemctl enable --now alertmanager
```


---

## Schritt 6 – Grafana installieren

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana

sudo systemctl enable --now grafana-server
```


---

## Schritt 7 – Grafana Dashboard einrichten

```text
URL: http://192.168.1.20:3000
Login: admin / admin
→ Passwort ändern: admin → sicheresGrafanaPasswort!

Datasource hinzufügen:
→ Configuration → Data Sources → Add data source
→ Prometheus → URL: http://localhost:9090 → Save & Test

Dashboard importieren:
→ Create → Import
→ Dashboard ID: 1860 (Node Exporter Full)
→ Dashboard ID: 7362 (MySQL Overview)
→ Dashboard ID: 3894 (Apache Overview)
```


---

## Schritt 8 – Watchdog-Dienst

```bash
sudo nano /opt/scripts/watchdog.sh
```

```bash
#!/bin/bash
# watchdog.sh – Automatischer Neustart bei Dienstausfall

LOG="/var/log/crm-watchdog.log"
EMAIL="admin@sample.ch"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG; }

check_service() {
    local SERVICE=$1
    if ! systemctl is-active --quiet $SERVICE; then
        log "ALARM: $SERVICE ist ausgefallen – Neustart wird versucht"
        systemctl restart $SERVICE
        sleep 5
        if systemctl is-active --quiet $SERVICE; then
            log "OK: $SERVICE erfolgreich neu gestartet"
            echo "$SERVICE wurde automatisch neu gestartet um $(date)" | \
                mail -s "WARNUNG: $SERVICE neu gestartet" $EMAIL
        else
            log "FEHLER: $SERVICE konnte nicht neu gestartet werden!"
            echo "$SERVICE konnte NICHT neu gestartet werden!" | \
                mail -s "KRITISCH: $SERVICE Ausfall" $EMAIL
        fi
    fi
}

while true; do
    check_service apache2
    check_service mariadb
    check_service named
    sleep 30
done
```

```bash
sudo nano /etc/systemd/system/crm-watchdog.service
```

```ini
[Unit]
Description=CRM Service Watchdog
After=network.target apache2.service mariadb.service

[Service]
Type=simple
ExecStart=/opt/scripts/watchdog.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
chmod +x /opt/scripts/watchdog.sh
sudo systemctl daemon-reload
sudo systemctl enable --now crm-watchdog

# Status prüfen
sudo systemctl status crm-watchdog
```


---

## Schritt 9 – Monitoring testen

```bash
# Apache absichtlich stoppen (Test)
sudo systemctl stop apache2
sleep 35

# Watchdog-Log prüfen
tail -5 /var/log/crm-watchdog.log
# [2025-03-24 14:32:15] ALARM: apache2 ist ausgefallen – Neustart wird versucht
# [2025-03-24 14:32:21] OK: apache2 erfolgreich neu gestartet

# Apache läuft wieder
sudo systemctl status apache2
# ● apache2.service: active (running)
```

---

## Fazit

Das vollständige Monitoring-System ist aktiv. Prometheus sammelt Metriken von System, Webserver und Datenbank. Grafana visualisiert alle Metriken in übersichtlichen Dashboards. Der Alertmanager sendet E-Mails bei kritischen Ereignissen. Der Watchdog startet ausgefallene Dienste automatisch neu und benachrichtigt den Administrator.
