# Auftrag #2 – Architekturdiagramm IST/SOLL

## Kompetenz
A: Release-Situation analysieren

---

## Beschreibung

In diesem Auftrag wurde eine vollständige IST-Analyse des bestehenden CRM-Systems durchgeführt. Die vorhandene Konfiguration wurde analysiert, dokumentiert und grafisch dargestellt. Anschliessend wurde ein SOLL-Diagramm mit der neuen Zielarchitektur erstellt. Verschiedene Migrationsvarianten wurden bewertet und die beste Lösung ausgewählt.

---

## IST-Analyse

### Übersicht IST-System

Das bestehende System läuft als einzelne VM on-premise. Alle Dienste (Webserver, Datenbank, Applikation) laufen auf derselben Maschine. Es gibt keine Trennung zwischen Applikations- und Datenbankschicht.

| Komponente | Wert |
|------------|------|
| Hostname | crmserver.sample.ch |
| Betriebssystem | Ubuntu 18.04 LTS (Bionic Beaver) |
| IP-Adresse | 192.168.1.10 |
| RAM | 4 GB |
| CPU | 2 vCPU |
| Disk | 80 GB |
| Webserver | Apache 2.4.29 |
| PHP | 7.2.24 |
| Datenbank | MySQL 5.7.42 |
| CRM | Vtiger CRM 7.1.0 |
| Backup | Kein automatisiertes Backup vorhanden |
| Monitoring | Keines |
| SSL | Kein SSL-Zertifikat |


---

### Konfiguration IST – Apache

```bash
# Apache Version prüfen
apache2 -v
# Server version: Apache/2.4.29 (Ubuntu)

# Aktive VHosts anzeigen
apache2ctl -S
```

```apacheconf
# /etc/apache2/sites-enabled/000-default.conf

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/vtiger
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

```


---

### Konfiguration IST – PHP

```bash
php -v
# PHP 7.2.24-0ubuntu0.18.04.17

php -i | grep memory_limit
# memory_limit => 128M

php -i | grep upload_max
# upload_max_filesize => 2M
```


---

### Konfiguration IST – MySQL

```bash
mysql --version
# mysql  Ver 14.14 Distrib 5.7.42

mysql -u root -p -e "SHOW DATABASES;"
# vtiger, information_schema, mysql, performance_schema
```


---

### Mengengerüst Datenbank

```sql
-- Anzahl Tabellen
SELECT COUNT(*) AS anzahl_tabellen
FROM information_schema.tables
WHERE table_schema = 'vtiger';
-- Ergebnis: 247

-- Datenbankgrösse
SELECT table_schema AS 'Datenbank',
ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Grösse (MB)'
FROM information_schema.tables
WHERE table_schema = 'vtiger';
-- Ergebnis: 1843.72 MB

-- Anzahl Kontakte
SELECT COUNT(*) FROM vtiger_contactdetails;
-- Ergebnis: 12'450

-- Anzahl Leads
SELECT COUNT(*) FROM vtiger_leaddetails;
-- Ergebnis: 8'320

-- Anzahl Benutzer
SELECT COUNT(*) FROM vtiger_users WHERE deleted = 0;
-- Ergebnis: 28
```


---

### Sicherheitsprobleme IST-System

Bei der IST-Analyse wurden folgende Sicherheitsprobleme festgestellt:

| Problem | Schweregrad | Beschreibung |
|---------|-------------|-------------|
| Kein SSL | Hoch | Gesamter Traffic unverschlüsselt |
| Kein Backup | Hoch | Kein automatisiertes Backup vorhanden |
| Root DB-Zugriff | Hoch | Vtiger läuft mit Root-Datenbankbenutzer |
| Ubuntu 18.04 EOL | Hoch | Kein Sicherheitssupport mehr seit April 2023 |
| PHP 7.2 EOL | Hoch | Kein Support mehr seit November 2019 |
| Kein Monitoring | Mittel | Kein Monitoring oder Alarmierung |
| Keine Firewall | Mittel | Alle Ports offen |

---

## SOLL-Architektur

### Übersicht SOLL-System

Das neue System wird auf einer modernen Ubuntu 24.04 VM aufgebaut. Alle Komponenten werden auf aktuelle Versionen aktualisiert. SSL, Backup und Monitoring werden implementiert.

| Komponente | Wert |
|------------|------|
| Hostname | crmserver.sample.ch |
| Betriebssystem | Ubuntu 24.04 LTS (Noble Numbat) |
| IP-Adresse | 192.168.1.20 |
| RAM | 8 GB |
| CPU | 4 vCPU |
| Disk | 120 GB |
| Webserver | Apache 2.4.58 + ModSecurity |
| PHP | 8.2.x |
| Datenbank | MariaDB 10.11 |
| CRM | Vtiger CRM 7.5.0 |
| Backup | Automatisiert, täglich, 30 Tage |
| Monitoring | Prometheus + Grafana |
| SSL | Let's Encrypt |


---

## Migrationsvarianten

### Variante A – Upgrade auf Vtiger 7.5 (empfohlen)

| Kriterium | Bewertung |
|-----------|-----------|
| Aufwand | Mittel |
| Risiko | Gering |
| Kosten | Gering |
| Datenmigration | Direkt möglich |
| Schulungsaufwand | Gering (gleiche Oberfläche) |
| Empfehlung | ✅ Ja |

**Begründung:** Vtiger 7.5 ist vollständig kompatibel mit den bestehenden Daten aus Version 7.1. Die Migration erfolgt über einen direkten Import. Die Benutzer kennen die Oberfläche bereits und benötigen keine Schulung. Das Risiko eines Datenverlusts ist minimal.

---

### Variante B – Wechsel auf Odoo (nicht empfohlen)

| Kriterium | Bewertung |
|-----------|-----------|
| Aufwand | Sehr hoch |
| Risiko | Hoch |
| Kosten | Hoch |
| Datenmigration | Komplex, eigenes Mapping nötig |
| Schulungsaufwand | Hoch (neue Oberfläche) |
| Empfehlung | ❌ Nein |

**Begründung:** Ein Wechsel auf Odoo würde eine vollständige Datentransformation erfordern, da die Datenstrukturen grundlegend verschieden sind. Der Schulungsaufwand für 28 Benutzer ist erheblich. Das Risiko eines unvollständigen oder fehlerhaften Datenimports ist hoch. Diese Variante ist im gegebenen Zeitrahmen nicht realistisch umsetzbar.

---

## Entscheid

**Variante A – Upgrade auf Vtiger 7.5** wurde gewählt.

Der Entscheid wurde gemeinsam mit dem Auftraggeber getroffen und dokumentiert. Die Kriterien Aufwand, Risiko, Kosten und Benutzerfreundlichkeit haben klar für Variante A gesprochen.


---

## Testfälle als Nachweis der Migration

| # | Testfall | SQL / Befehl | Erwartetes Ergebnis |
|---|----------|-------------|-------------------|
| T01 | Anzahl Kontakte | `SELECT COUNT(*) FROM vtiger_contactdetails;` | 12'450 |
| T02 | Anzahl Leads | `SELECT COUNT(*) FROM vtiger_leaddetails;` | 8'320 |
| T03 | Anzahl Benutzer | `SELECT COUNT(*) FROM vtiger_users WHERE deleted=0;` | 28 |
| T04 | DB-Grösse | `SELECT ... SUM(data_length+index_length)...` | ~1.8 GB |
| T05 | Anzahl Tabellen | `SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='vtiger';` | 247 |

---

## Fazit

Die IST-Analyse hat mehrere kritische Sicherheitsprobleme aufgedeckt, insbesondere das veraltete Betriebssystem und die fehlenden Sicherheitsmechanismen. Das SOLL-System behebt alle identifizierten Probleme. Die Wahl von Variante A minimiert das Migrationsrisiko bei gleichzeitig hohem Sicherheitsgewinn.
