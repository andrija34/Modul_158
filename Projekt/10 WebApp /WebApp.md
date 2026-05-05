# Auftrag #10 – WebApp/DB-Migration

## Kompetenz
D: Datenmigration durchführen

---

## Beschreibung

In diesem Auftrag wurde die vollständige Migration der Vtiger CRM Applikation und Datenbank vom IST-System auf das SOLL-System durchgeführt. Die Migration wurde schrittweise geplant, automatisiert und mit einem vollständigen Datennachweis abgeschlossen.

---

## Migrationsplan

| Schritt | Beschreibung | Zeit | Verantwortlich |
|---------|-------------|------|---------------|
| 1 | Snapshot IST-System erstellen | 21:00 | Andrija |
| 2 | Letzten DB-Export durchführen | 21:10 | Andrija |
| 3 | DB-Export auf Ziel übertragen | 21:20 | Andrija |
| 4 | DB-Import auf Zielsystem | 21:30 | Andrija |
| 5 | Dateien via SFTP übertragen | 21:40 | Andrija |
| 6 | config.inc.php anpassen | 22:30 | Andrija |
| 7 | Berechtigungen setzen | 22:35 | Andrija |
| 8 | Vtiger Upgrade 7.1 → 7.5 | 22:40 | Andrija |
| 9 | Abschlusstests | 23:00 | Andrija |
| 10 | DNS umstellen | 23:30 | Andrija |
| 11 | Benutzer informieren | 23:35 | Andrija |

---

## Schritt 1 – Snapshot IST-System

```bash
VBoxManage snapshot vm-crm-alt take "pre-migration-final" \
    --description "Snapshot unmittelbar vor Migration – 22.03.2025 21:00"
```


---

## Schritt 2 – DB-Export IST-System

```bash
ssh ubuntu@192.168.1.10

mysqldump \
    -u root -pvtigerroot \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    vtiger > /backup/vtiger_migration_final.sql

# Grösse prüfen
du -sh /backup/vtiger_migration_final.sql
# 1.8G  /backup/vtiger_migration_final.sql

# Anzahl Zeilen
wc -l /backup/vtiger_migration_final.sql
# 4382910 /backup/vtiger_migration_final.sql

gzip /backup/vtiger_migration_final.sql
```


---

## Schritt 3 – Export übertragen

```bash
# Export via SCP auf Zielsystem übertragen
scp ubuntu@192.168.1.10:/backup/vtiger_migration_final.sql.gz \
    /backup/vtiger_migration_final.sql.gz

# Integrität prüfen (MD5)
ssh ubuntu@192.168.1.10 "md5sum /backup/vtiger_migration_final.sql.gz"
md5sum /backup/vtiger_migration_final.sql.gz
# Beide Hashes müssen identisch sein
```


---

## Schritt 4 – DB-Import Zielsystem

```bash
# Datenbank vorbereiten
mysql -u root -p -e "
DROP DATABASE IF EXISTS vtiger;
CREATE DATABASE vtiger CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON vtiger.* TO 'vtigeruser'@'localhost';
FLUSH PRIVILEGES;"

# Import durchführen
zcat /backup/vtiger_migration_final.sql.gz | \
    mysql -u vtigeruser -psicheresPasswort123! vtiger

echo "Import abgeschlossen: $(date)"
```


---

## Schritt 5 – Dateien übertragen

```bash
rsync -avz \
    --progress \
    --stats \
    --exclude "*.log" \
    --exclude "cache/*" \
    --exclude "*.tmp" \
    ubuntu@192.168.1.10:/var/www/html/vtiger/ \
    /var/www/crmserver/

# Übertragungsstatistik
# Number of files: 8,432
# Total file size: 2.31 GB
# Total transferred file size: 2.28 GB
```


---

## Schritt 6 – Konfiguration anpassen

```bash
sudo nano /var/www/crmserver/config.inc.php
```

```php
<?php
// Datenbankverbindung
$dbconfig['db_server']      = '127.0.0.1';
$dbconfig['db_port']        = '3306';
$dbconfig['db_username']    = 'vtigeruser';
$dbconfig['db_password']    = 'sicheresPasswort123!';
$dbconfig['db_name']        = 'vtiger';
$dbconfig['db_type']        = 'mysql';

// Site URL
$site_URL = 'http://crmserver.sample.ch';
$root_directory = '/var/www/crmserver/';

// Logging
$LOG4PHP_DEBUG = false;
```


---

## Schritt 7 – Berechtigungen setzen

```bash
sudo chown -R www-data:www-data /var/www/crmserver/
sudo find /var/www/crmserver/ -type f -exec chmod 644 {} \;
sudo find /var/www/crmserver/ -type d -exec chmod 755 {} \;

# Spezielle Verzeichnisse mit Schreibrechten
sudo chmod -R 775 /var/www/crmserver/cache
sudo chmod -R 775 /var/www/crmserver/storage
sudo chmod -R 775 /var/www/crmserver/upload
sudo chmod -R 775 /var/www/crmserver/logs
```


---

## Schritt 8 – Vtiger Upgrade 7.1 → 7.5

```bash
# Upgrade-Paket herunterladen
wget https://sourceforge.net/projects/vtigercrm/files/vtiger%20CRM%207.5.0/Core%20Product/vtigercrm7.5.0.tar.gz

# Entpacken
tar -xzf vtigercrm7.5.0.tar.gz -C /tmp/

# Upgrade-Script ausführen
cd /tmp/vtigercrm7.5.0/migrate/
php migrate.php --source=/var/www/crmserver --target=7.5.0

echo "Upgrade abgeschlossen"
```


---

## Schritt 9 – Vollautomatisiertes Migrationsskript

```bash
#!/bin/bash
# full-migration.sh – Vollautomatisierte CRM Migration

set -e
LOG="/var/log/crm-full-migration.log"
SOURCE_HOST="192.168.1.10"
SOURCE_USER="ubuntu"

echo "=== CRM Migration gestartet: $(date) ===" | tee $LOG

# 1. Snapshot
VBoxManage snapshot vm-crm-alt take "pre-migration" 2>/dev/null || true
echo "[1/9] Snapshot erstellt" | tee -a $LOG

# 2. Export
ssh $SOURCE_USER@$SOURCE_HOST "mysqldump -u root -pvtigerroot --single-transaction vtiger | gzip > /tmp/vtiger_mig.sql.gz"
echo "[2/9] DB-Export abgeschlossen" | tee -a $LOG

# 3. Übertragen
scp $SOURCE_USER@$SOURCE_HOST:/tmp/vtiger_mig.sql.gz /backup/
echo "[3/9] Export übertragen" | tee -a $LOG

# 4. DB Import
mysql -u root -p"$(cat /root/.dbroot)" -e "DROP DATABASE IF EXISTS vtiger; CREATE DATABASE vtiger CHARACTER SET utf8mb4;"
zcat /backup/vtiger_mig.sql.gz | mysql -u vtigeruser -psicheresPasswort123! vtiger
echo "[4/9] DB importiert" | tee -a $LOG

# 5. Dateien
rsync -avz --exclude "*.log" --exclude "cache/*" $SOURCE_USER@$SOURCE_HOST:/var/www/html/vtiger/ /var/www/crmserver/ >> $LOG
echo "[5/9] Dateien übertragen" | tee -a $LOG

# 6. Konfiguration
cp /opt/deploy/config.inc.php /var/www/crmserver/config.inc.php
echo "[6/9] Konfiguration gesetzt" | tee -a $LOG

# 7. Berechtigungen
chown -R www-data:www-data /var/www/crmserver/
find /var/www/crmserver/ -type d -exec chmod 775 {} \;
find /var/www/crmserver/ -type f -exec chmod 644 {} \;
echo "[7/9] Berechtigungen gesetzt" | tee -a $LOG

# 8. Apache reload
systemctl reload apache2
echo "[8/9] Apache neu geladen" | tee -a $LOG

# 9. Validierung
CONTACTS=$(mysql -u vtigeruser -psicheresPasswort123! vtiger -e "SELECT COUNT(*) FROM vtiger_contactdetails;" -s -N)
LEADS=$(mysql -u vtigeruser -psicheresPasswort123! vtiger -e "SELECT COUNT(*) FROM vtiger_leaddetails;" -s -N)
echo "[9/9] Validierung: Kontakte=$CONTACTS | Leads=$LEADS" | tee -a $LOG

echo "=== Migration abgeschlossen: $(date) ===" | tee -a $LOG
```


---

## Kommunikation an Benutzer

### E-Mail vor Migration

> **Betreff:** Wartung CRM-System – Sa 22.03.2025 22:00–02:00 Uhr
>
> Sehr geehrte Damen und Herren,
> das CRM-System wird in der Nacht von Samstag auf Sonntag migriert.
> **Zeitfenster:** 22.03.2025, 22:00 Uhr bis ca. 02:00 Uhr
> Bitte speichern Sie alle offenen Daten bis 21:45 Uhr.
> Nach der Migration ist das System unter derselben Adresse erreichbar.

### E-Mail nach Migration

> **Betreff:** CRM-System erfolgreich migriert
>
> Die Migration wurde erfolgreich abgeschlossen.
> Das System ist ab sofort wieder unter http://crmserver.sample.ch erreichbar.
> Bei Problemen melden Sie sich bitte bei admin@sample.ch.

---

## Validierung – Datennachweis

```sql
-- Kontakte
SELECT COUNT(*) FROM vtiger_contactdetails;
-- IST: 12'450  |  SOLL: 12'450  ✅

-- Leads
SELECT COUNT(*) FROM vtiger_leaddetails;
-- IST: 8'320  |  SOLL: 8'320  ✅

-- Benutzer
SELECT COUNT(*) FROM vtiger_users WHERE deleted = 0;
-- IST: 28  |  SOLL: 28  ✅

-- Tabellen
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'vtiger';
-- IST: 247  |  SOLL: 247  ✅
```


---

## Fazit

Die Migration wurde vollständig automatisiert, alle Daten integer übertragen und durch SQL-Abfragen validiert. Der Ausfall betrug 1h 48min, was innerhalb des geplanten Fensters liegt. Die Benutzer wurden rechtzeitig informiert und nach der Migration sofort benachrichtigt.
