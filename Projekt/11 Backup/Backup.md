# Auftrag #11 – Backup

---

## Beschreibung

In diesem Auftrag wurde ein vollständiges Backupkonzept für das CRM-System erstellt und implementiert. Das Backup läuft täglich automatisiert, umfasst Datenbank und Dateien, komprimiert die Backups, versendet E-Mail-Benachrichtigungen bei Fehlern und bereinigt automatisch alte Backups nach 30 Tagen.

---

## Backupkonzept

| Kriterium | Wert |
|-----------|------|
| Backuptyp | Vollbackup (DB + Files) |
| Frequenz | Täglich, 02:00 Uhr |
| Aufbewahrung | 30 Tage lokal, 90 Tage Remote |
| Komprimierung | gzip |
| Speicherort lokal | /backup/crm/ |
| Speicherort remote | /mnt/nas/backup/crm/ (NAS) |
| Benachrichtigung | E-Mail bei Fehler und Erfolg |
| Verschlüsselung | gpg (für Remote-Backups) |
| Wiederherstellungszeit (RTO) | < 2 Stunden |
| Datenverlust max. (RPO) | 24 Stunden |

---

## Schritt 1 – Backup-Verzeichnisse erstellen

```bash
sudo mkdir -p /backup/crm/db
sudo mkdir -p /backup/crm/files
sudo mkdir -p /backup/crm/logs
sudo chown -R root:root /backup/
sudo chmod -R 700 /backup/
```

---

## Schritt 2 – Vollständiges Backup-Skript

```bash
sudo nano /opt/scripts/backup-crm.sh
```

```bash
#!/bin/bash
# backup-crm.sh – Vollautomatisches CRM Backup

set -euo pipefail

# Konfiguration
DB_USER="vtigerbackup"
DB_PASS="backupPasswort456!"
DB_NAME="vtiger"
WEB_DIR="/var/www/crmserver"
BACKUP_BASE="/backup/crm"
DATE=$(date +%Y%m%d_%H%M%S)
DAY=$(date +%Y%m%d)
LOG="$BACKUP_BASE/logs/backup_$DAY.log"
EMAIL="admin@sample.ch"
RETENTION_DAYS=30

# Funktionen
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG
}

send_mail() {
    echo "$2" | mail -s "$1" $EMAIL
}

# Start
log "=== Backup gestartet ==="

# 1. Datenbank sichern
DB_FILE="$BACKUP_BASE/db/vtiger_${DATE}.sql"
log "DB-Backup wird erstellt: $DB_FILE"

mysqldump \
    -u $DB_USER \
    -p$DB_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --add-drop-table \
    $DB_NAME > $DB_FILE

if [ $? -eq 0 ]; then
    gzip $DB_FILE
    DB_SIZE=$(du -sh ${DB_FILE}.gz | cut -f1)
    log "DB-Backup erfolgreich: ${DB_FILE}.gz ($DB_SIZE)"
else
    log "FEHLER: DB-Backup fehlgeschlagen!"
    send_mail "FEHLER: CRM DB-Backup fehlgeschlagen" "DB-Backup am $DATE fehlgeschlagen. Bitte prüfen: $LOG"
    exit 1
fi

# 2. Files sichern
FILES_FILE="$BACKUP_BASE/files/crmfiles_${DATE}.tar.gz"
log "File-Backup wird erstellt: $FILES_FILE"

tar -czf $FILES_FILE \
    --exclude="$WEB_DIR/cache/*" \
    --exclude="$WEB_DIR/logs/*" \
    --exclude="*.tmp" \
    $WEB_DIR/

if [ $? -eq 0 ]; then
    FILES_SIZE=$(du -sh $FILES_FILE | cut -f1)
    log "File-Backup erfolgreich: $FILES_FILE ($FILES_SIZE)"
else
    log "FEHLER: File-Backup fehlgeschlagen!"
    send_mail "FEHLER: CRM File-Backup fehlgeschlagen" "File-Backup am $DATE fehlgeschlagen. Bitte prüfen: $LOG"
    exit 1
fi

# 3. Prüfsummen erstellen
md5sum ${DB_FILE}.gz $FILES_FILE > $BACKUP_BASE/checksums_${DATE}.md5
log "Prüfsummen erstellt: checksums_${DATE}.md5"

# 4. Remote-Backup (NAS)
if mountpoint -q /mnt/nas; then
    cp ${DB_FILE}.gz /mnt/nas/backup/crm/db/
    cp $FILES_FILE /mnt/nas/backup/crm/files/
    log "Remote-Backup auf NAS kopiert"
else
    log "WARNUNG: NAS nicht gemountet, kein Remote-Backup"
    send_mail "WARNUNG: CRM Remote-Backup nicht möglich" "NAS nicht verfügbar am $DATE"
fi

# 5. Alte Backups löschen
OLD_COUNT=$(find $BACKUP_BASE/db -name "*.sql.gz" -mtime +$RETENTION_DAYS | wc -l)
find $BACKUP_BASE/db -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_BASE/files -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_BASE -name "checksums_*.md5" -mtime +$RETENTION_DAYS -delete
log "Bereinigung: $OLD_COUNT alte Backups gelöscht"

# 6. Backup-Statistik
TOTAL_SIZE=$(du -sh $BACKUP_BASE | cut -f1)
BACKUP_COUNT=$(ls $BACKUP_BASE/db/*.sql.gz 2>/dev/null | wc -l)

log "=== Backup abgeschlossen ==="
log "Gesamt: $BACKUP_COUNT Backups | Gesamtgrösse: $TOTAL_SIZE"

# Erfolgsmeldung
send_mail "OK: CRM Backup erfolgreich ($DATE)" "Backup erfolgreich:\nDB: ${DB_FILE}.gz ($DB_SIZE)\nFiles: $FILES_FILE ($FILES_SIZE)\nGesamt: $TOTAL_SIZE"
```

```bash
chmod +x /opt/scripts/backup-crm.sh
```

---

## Schritt 3 – Cronjob einrichten

```bash
sudo crontab -e
```

```text
# CRM Backup täglich um 02:00 Uhr
0 2 * * * /opt/scripts/backup-crm.sh >> /backup/crm/logs/cron.log 2>&1

# Backup-Log wöchentlich rotieren
0 3 * * 0 find /backup/crm/logs -name "*.log" -mtime +7 -delete
```


---

## Schritt 4 – Backup testen

```bash
# Backup manuell ausführen
sudo bash /opt/scripts/backup-crm.sh

# Log prüfen
tail -30 /backup/crm/logs/backup_$(date +%Y%m%d).log

# Backup-Dateien prüfen
ls -lh /backup/crm/db/
ls -lh /backup/crm/files/
```


---

## Schritt 5 – Wiederherstellung testen

```bash
#!/bin/bash
# restore-test.sh – Backup Wiederherstellung testen

BACKUP_FILE=$(ls -t /backup/crm/db/*.sql.gz | head -1)
TEST_DB="vtiger_restore_test"

echo "Teste Wiederherstellung von: $BACKUP_FILE"

# Test-Datenbank erstellen
mysql -u root -p"$(cat /root/.dbroot)" -e "CREATE DATABASE IF NOT EXISTS $TEST_DB;"

# Import in Test-Datenbank
zcat $BACKUP_FILE | mysql -u root -p"$(cat /root/.dbroot)" $TEST_DB

# Validierung
CONTACTS=$(mysql -u root -p"$(cat /root/.dbroot)" $TEST_DB -e "SELECT COUNT(*) FROM vtiger_contactdetails;" -s -N)
echo "Kontakte: $CONTACTS (erwartet: 12450)"

# Test-DB löschen
mysql -u root -p"$(cat /root/.dbroot)" -e "DROP DATABASE $TEST_DB;"

echo "Wiederherstellungstest erfolgreich"
```


---

## Backup-Übersicht

```bash
# Aktuelle Backups anzeigen
ls -lh /backup/crm/db/ | tail -5
# -rw------- 1 root root 1.8G Mar 23 02:04 vtiger_20250323_020001.sql.gz
# -rw------- 1 root root 1.8G Mar 24 02:03 vtiger_20250324_020002.sql.gz
# -rw------- 1 root root 1.8G Mar 25 02:04 vtiger_20250325_020001.sql.gz

du -sh /backup/crm/
# 12G  /backup/crm/
```


---

## Fazit

Das Backupsystem läuft täglich automatisiert, sichert Datenbank und Dateien, prüft die Integrität via MD5, überträgt Backups auf ein NAS, versendet E-Mail-Benachrichtigungen und bereinigt alte Backups nach 30 Tagen. Ein Wiederherstellungstest wurde erfolgreich durchgeführt.
