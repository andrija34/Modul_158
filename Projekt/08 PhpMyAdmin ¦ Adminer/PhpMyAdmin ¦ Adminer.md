# Auftrag #8 – PhpMyAdmin/Adminer

---

## Beschreibung

In diesem Auftrag wurde PhpMyAdmin installiert und für den Produktionseinsatz abgesichert. Zusätzlich wurden vollständige Export- und Importskripte erstellt und erfolgreich für die Datenmigration eingesetzt.

---

## Schritt 1 – PhpMyAdmin installieren

```bash
sudo apt update
sudo apt install -y phpmyadmin

# Während der Installation:
# Web server to configure: apache2 (mit Leertaste auswählen, dann Enter)
# Configure database for phpmyadmin: Yes
# MySQL application password: <sicheresPhpMyAdminPasswort>
```


---

## Schritt 2 – PhpMyAdmin in Apache einbinden

```bash
# Symlink erstellen
sudo ln -s /usr/share/phpmyadmin /var/www/crmserver/dbadmin

# Alternatively: Include in Apache Konfiguration
sudo nano /etc/apache2/sites-available/crmserver.conf
```

```apacheconf
# PhpMyAdmin Alias
Alias /dbadmin /usr/share/phpmyadmin

<Directory /usr/share/phpmyadmin>
    Options SymLinksIfOwnerMatch
    DirectoryIndex index.php
    AllowOverride All

    # IP-Beschränkung (nur aus internem Netz)
    Require ip 192.168.1.0/24
    Require ip 127.0.0.1

    # HTTP Basic Auth zusätzlich
    AuthType Basic
    AuthName "Database Administration"
    AuthUserFile /etc/apache2/.htpasswd-dbadmin
    Require valid-user
</Directory>
```

```bash
# htpasswd Datei erstellen
sudo htpasswd -c /etc/apache2/.htpasswd-dbadmin dbadmin
# Passwort: sicheresDbAdminPasswort789!

sudo systemctl reload apache2
```


---

## Schritt 3 – PhpMyAdmin Konfiguration anpassen

```bash
sudo nano /etc/phpmyadmin/config.inc.php
```

```php
<?php
// Blowfish Secret (min. 32 Zeichen)
$cfg['blowfish_secret'] = 'xK8mN2pQ5rT7vW9yA1bC3dF6gH0jL4nP';

// Session Timeout (30 Minuten)
$cfg['LoginCookieValidity'] = 1800;

// Upload Limit
$cfg['ExecTimeLimit'] = 300;
$cfg['UploadDir'] = '/tmp/phpmyadmin-uploads';
$cfg['SaveDir'] = '/tmp/phpmyadmin-saves';

// Server Konfiguration
$cfg['Servers'][1]['host'] = '127.0.0.1';
$cfg['Servers'][1]['auth_type'] = 'cookie';
$cfg['Servers'][1]['AllowNoPassword'] = false;

// Sicherheit: Root-Login verbieten
$cfg['Servers'][1]['AllowRoot'] = false;
```


---

## Schritt 4 – PhpMyAdmin testen

```bash
# Zugriff testen
curl -I http://crmserver.sample.ch/dbadmin/
# HTTP/1.1 401 Unauthorized  (HTTP Auth aktiv)

# Nach Login: Datenbanken prüfen
# http://192.168.1.20/dbadmin/
# Login: vtigeruser / sicheresPasswort123!
```


---

## Schritt 5 – Datenbankexport-Skript

```bash
sudo nano /opt/scripts/db-export.sh
```

```bash
#!/bin/bash
# db-export.sh – Vollständiger Datenbankexport mit Validierung

set -e

DB_USER="vtigerbackup"
DB_PASS="backupPasswort456!"
DB_NAME="vtiger"
DATE=$(date +%Y%m%d_%H%M%S)
EXPORT_DIR="/backup/exports"
EXPORT_FILE="$EXPORT_DIR/${DB_NAME}_${DATE}.sql"
LOG="/var/log/db-export.log"

mkdir -p $EXPORT_DIR

echo "[$(date)] Export gestartet: $EXPORT_FILE" >> $LOG

# Export durchführen
mysqldump \
    -u $DB_USER \
    -p$DB_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --add-drop-table \
    $DB_NAME > $EXPORT_FILE

# Exportgrösse prüfen
SIZE=$(du -sh $EXPORT_FILE | cut -f1)
echo "[$(date)] Export abgeschlossen: $SIZE" >> $LOG

# Komprimieren
gzip $EXPORT_FILE
echo "[$(date)] Komprimiert: ${EXPORT_FILE}.gz" >> $LOG

# Anzahl Zeilen prüfen
LINES=$(zcat ${EXPORT_FILE}.gz | wc -l)
echo "[$(date)] Zeilen im Export: $LINES" >> $LOG

echo "[$(date)] Export erfolgreich: ${EXPORT_FILE}.gz ($SIZE)" | tee -a $LOG
```

```bash
chmod +x /opt/scripts/db-export.sh
sudo bash /opt/scripts/db-export.sh
```


---

## Schritt 6 – Datenbankimport-Skript

```bash
sudo nano /opt/scripts/db-import.sh
```

```bash
#!/bin/bash
# db-import.sh – Datenbankimport mit Validierung

set -e

DB_USER="vtigeruser"
DB_PASS="sicheresPasswort123!"
DB_NAME="vtiger"
IMPORT_FILE=$1
LOG="/var/log/db-import.log"

if [ -z "$IMPORT_FILE" ]; then
    echo "Verwendung: $0 <backup.sql.gz>"
    exit 1
fi

if [ ! -f "$IMPORT_FILE" ]; then
    echo "Fehler: Datei nicht gefunden: $IMPORT_FILE"
    exit 1
fi

echo "[$(date)] Import gestartet: $IMPORT_FILE" >> $LOG

# Datenbank leeren (falls vorhanden)
mysql -u root -p"$(cat /root/.dbroot)" -e "DROP DATABASE IF EXISTS $DB_NAME; CREATE DATABASE $DB_NAME CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Berechtigungen neu setzen
mysql -u root -p"$(cat /root/.dbroot)" -e "GRANT ALL PRIVILEGES ON $DB_NAME.* TO '$DB_USER'@'localhost'; FLUSH PRIVILEGES;"

# Import durchführen
if [[ $IMPORT_FILE == *.gz ]]; then
    zcat $IMPORT_FILE | mysql -u $DB_USER -p$DB_PASS $DB_NAME
else
    mysql -u $DB_USER -p$DB_PASS $DB_NAME < $IMPORT_FILE
fi

# Validierung
TABLES=$(mysql -u $DB_USER -p$DB_PASS $DB_NAME -e "SHOW TABLES;" | wc -l)
CONTACTS=$(mysql -u $DB_USER -p$DB_PASS $DB_NAME -e "SELECT COUNT(*) FROM vtiger_contactdetails;" -s -N)
LEADS=$(mysql -u $DB_USER -p$DB_PASS $DB_NAME -e "SELECT COUNT(*) FROM vtiger_leaddetails;" -s -N)

echo "[$(date)] Import abgeschlossen" >> $LOG
echo "[$(date)] Tabellen: $TABLES | Kontakte: $CONTACTS | Leads: $LEADS" | tee -a $LOG
```

```bash
chmod +x /opt/scripts/db-import.sh
sudo bash /opt/scripts/db-import.sh /backup/exports/vtiger_20250322_220000.sql.gz
```


---

## Fazit

PhpMyAdmin wurde installiert, abgesichert (HTTP Auth + IP-Beschränkung + Root-Login deaktiviert) und erfolgreich getestet. Die Export- und Importskripte führen eine vollständige Validierung der Daten durch und werden für die Datenmigration eingesetzt.
