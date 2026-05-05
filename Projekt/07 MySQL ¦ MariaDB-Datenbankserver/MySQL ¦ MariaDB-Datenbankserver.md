# Auftrag #7 – MySQL/MariaDB-Datenbankserver

---

## Beschreibung

In diesem Auftrag wurde MariaDB 10.11 als Ersatz für MySQL 5.7 installiert. Die Datenbank wurde vollständig konfiguriert und abgesichert. Der Zugriff wurde auf spezifische Benutzer und IP-Adressen beschränkt. Performance-Optimierungen wurden basierend auf dem verfügbaren RAM und den Anforderungen von Vtiger CRM vorgenommen.

---

## Ausgangslage IST

```bash
# MySQL Version IST-System
mysql --version
# mysql  Ver 14.14 Distrib 5.7.42

# Aktueller Root-Zugriff
mysql -u root -p -e "SELECT user, host FROM mysql.user;"
# root   localhost
# root   %           <-- Sicherheitsproblem: Root von überall erreichbar
# vtigeruser localhost

# DB-Grösse IST
mysql -u root -p -e "
SELECT table_schema AS 'DB',
ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'MB'
FROM information_schema.tables
WHERE table_schema = 'vtiger';"
# vtiger  1843.72
```

---

## Schritt 1 – MariaDB installieren

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client

# Version prüfen
mariadb --version
# mariadb  Ver 15.1 Distrib 10.11.6-MariaDB

# Dienst aktivieren
sudo systemctl enable --now mariadb

# Status prüfen
sudo systemctl status mariadb
```


---

## Schritt 2 – MariaDB absichern

```bash
sudo mysql_secure_installation
```

Folgende Einstellungen wurden vorgenommen:

| Frage | Antwort | Begründung |
|-------|---------|------------|
| Set root password? | Yes | Sicheres Root-Passwort setzen |
| Remove anonymous users? | Yes | Anonym-Zugriff entfernen |
| Disallow root login remotely? | Yes | Root nur lokal erlauben |
| Remove test database? | Yes | Test-DB entfernen |
| Reload privilege tables? | Yes | Änderungen sofort wirksam |


---

## Schritt 3 – Datenbank und Benutzer erstellen

```sql
-- MariaDB aufrufen
sudo mysql -u root -p

-- Datenbank erstellen mit korrektem Zeichensatz
CREATE DATABASE vtiger
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Applikationsbenutzer erstellen (nur von localhost)
CREATE USER 'vtigeruser'@'localhost'
    IDENTIFIED BY 'sicheresPasswort123!';

-- Berechtigungen zuweisen (nur auf vtiger-DB)
GRANT ALL PRIVILEGES ON vtiger.* TO 'vtigeruser'@'localhost';

-- Backup-Benutzer erstellen (nur SELECT, LOCK)
CREATE USER 'vtigerbackup'@'localhost'
    IDENTIFIED BY 'backupPasswort456!';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER
    ON vtiger.* TO 'vtigerbackup'@'localhost';

-- Änderungen aktivieren
FLUSH PRIVILEGES;

-- Benutzer prüfen
SELECT user, host, plugin FROM mysql.user;
```


---

## Schritt 4 – MariaDB Konfiguration optimieren

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
[mysqld]
# Netzwerk
bind-address            = 127.0.0.1
port                    = 3306

# Zeichensatz
character-set-server    = utf8mb4
collation-server        = utf8mb4_unicode_ci

# Performance (basierend auf 8 GB RAM)
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 4
innodb_log_file_size    = 512M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method     = O_DIRECT

# Verbindungen
max_connections         = 300
max_connect_errors      = 100000
wait_timeout            = 600
interactive_timeout     = 600

# Query Cache
query_cache_type        = 1
query_cache_size        = 128M
query_cache_limit       = 4M

# Slow Query Log
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 2
log_queries_not_using_indexes = 1

# Binlog (für Replikation / Point-in-Time Recovery)
log_bin                 = /var/log/mysql/mysql-bin.log
expire_logs_days        = 7
max_binlog_size         = 100M
```

```bash
sudo systemctl restart mariadb

# Konfiguration prüfen
sudo mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
# innodb_buffer_pool_size  4294967296 (= 4 GB)
```


---

## Schritt 5 – Datenbankzugriff testen

```bash
# Login als vtigeruser testen
mysql -u vtigeruser -psicheresPasswort123! vtiger -e "SHOW TABLES;" | head -10

# Root-Fernzugriff versuchen (muss scheitern)
mysql -u root -h 192.168.1.20 -p
# ERROR 1130 (HY000): Host '...' is not allowed to connect to this MariaDB server

# Backup-User testen
mysql -u vtigerbackup -pbackupPasswort456! vtiger -e "SELECT COUNT(*) FROM vtiger_contactdetails;"
# 12450
```


---

## Schritt 6 – MariaDB Performance prüfen

```bash
# mysqltuner installieren (Analyse-Tool)
sudo apt install -y mysqltuner

sudo mysqltuner --user root --pass <passwort>
```

```text
>>  MySQLTuner 2.x
[OK] Currently running supported MariaDB version 10.11.6
[OK] Operating on 64-bit architecture
[OK] InnoDB buffer pool size (4.0G) is above critical threshold
[OK] Maximum reached memory usage: 4.8G
[OK] Slow queries: 0% (0/1K)
[OK] Highest connection usage: 8% (24/300)
[!!] Table cache hit rate: 23% (400 open / 1K opened)
     Add table_open_cache = 800 to [mysqld]
```

---

## Fazit

MariaDB 10.11 wurde erfolgreich installiert, abgesichert und optimiert. Der Zugriff ist auf localhost beschränkt, Root-Fernzugriff ist deaktiviert und dedizierte Benutzer wurden für Applikation und Backup erstellt. Die Performance-Konfiguration ist auf die verfügbaren Ressourcen abgestimmt.
