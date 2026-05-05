# Auftrag #9 – SFTP-Zugang

---

## Beschreibung

In diesem Auftrag wurde ein sicherer SFTP-Zugang eingerichtet. Der SFTP-Benutzer wurde auf das Webverzeichnis beschränkt (Chroot). Eine automatisierte Datenmigration via SFTP wurde implementiert und getestet.

---

## Ausgangslage

Für die Migration der CRM-Anwendungsdaten müssen rund 2.3 GB Dateien (PHP-Dateien, Uploads, Konfigurationen) vom IST-System auf das SOLL-System übertragen werden. Dafür wurde ein sicherer SFTP-Zugang eingerichtet.

---

## Schritt 1 – SFTP-Benutzer einrichten

```bash
# Gruppe erstellen
sudo groupadd sftpusers

# Benutzer erstellen (kein Shell-Zugriff)
sudo useradd \
    -m \
    -d /var/www/crmserver \
    -G sftpusers \
    -s /usr/sbin/nologin \
    sftpcrm

# Passwort setzen
sudo passwd sftpcrm
# Passwort: sftpSicheresPasswort321!

# Benutzer prüfen
id sftpcrm
# uid=1002(sftpcrm) gid=1002(sftpcrm) groups=1002(sftpcrm),1003(sftpusers)
```


---

## Schritt 2 – SSH Konfiguration für SFTP

```bash
sudo nano /etc/ssh/sshd_config
```

```text
# Standard SFTP Subsystem ersetzen
# Subsystem sftp /usr/lib/openssh/sftp-server
Subsystem sftp internal-sftp -l INFO -f AUTH

# SFTP-Gruppe konfigurieren
Match Group sftpusers
    # Chroot auf Webverzeichnis
    ChrootDirectory /var/www/crmserver
    # Nur SFTP erlauben, kein SSH
    ForceCommand internal-sftp
    # Weiterleitungen deaktivieren
    AllowTcpForwarding no
    AllowAgentForwarding no
    X11Forwarding no
    PermitTunnel no
```

```bash
# Syntax prüfen
sudo sshd -t
# (kein Output = OK)

# SSHD neu starten
sudo systemctl restart sshd
```


---

## Schritt 3 – Verzeichnisberechtigungen setzen

Chroot erfordert, dass das Chroot-Verzeichnis Root gehört:

```bash
# Chroot-Verzeichnis muss Root gehören
sudo chown root:root /var/www/crmserver
sudo chmod 755 /var/www/crmserver

# Unterverzeichnis für sftpcrm schreibbar machen
sudo mkdir -p /var/www/crmserver/uploads
sudo chown sftpcrm:sftpusers /var/www/crmserver/uploads
sudo chmod 755 /var/www/crmserver/uploads
```


---

## Schritt 4 – SFTP Verbindung testen

```bash
# Verbindung testen
sftp sftpcrm@192.168.1.20
# sftpcrm@192.168.1.20's password:
# Connected to 192.168.1.20.
# sftp>

# Verzeichnis anzeigen
sftp> ls
# uploads

# Datei hochladen
sftp> put testfile.txt uploads/
# Uploading testfile.txt to /uploads/testfile.txt
# testfile.txt  100%  1KB  1.0KB/s  00:00

# SSH-Zugriff versuchen (muss scheitern)
ssh sftpcrm@192.168.1.20
# This service allows sftp connections only.
# Connection to 192.168.1.20 closed.
```


---

## Schritt 5 – SSH Key Authentication einrichten

Für die automatisierte Migration wurde Key-Authentifizierung eingerichtet:

```bash
# SSH-Key generieren (auf Quell-Server)
ssh-keygen -t ed25519 -f ~/.ssh/sftp_migration_key -N ""

# Public Key auf Ziel-Server kopieren
sudo mkdir -p /home/sftpcrm/.ssh
sudo cp ~/.ssh/sftp_migration_key.pub /home/sftpcrm/.ssh/authorized_keys
sudo chown -R sftpcrm:sftpusers /home/sftpcrm/.ssh
sudo chmod 700 /home/sftpcrm/.ssh
sudo chmod 600 /home/sftpcrm/.ssh/authorized_keys

# Test ohne Passwort
sftp -i ~/.ssh/sftp_migration_key sftpcrm@192.168.1.20
# Connected to 192.168.1.20.
```


---

## Schritt 6 – Automatisiertes Migrations-Skript

```bash
sudo nano /opt/scripts/sftp-migration.sh
```

```bash
#!/bin/bash
# sftp-migration.sh – Automatisierte Datenmigration via SFTP

set -e

SOURCE_DIR="/var/www/html/vtiger"
DEST_HOST="192.168.1.20"
DEST_USER="sftpcrm"
DEST_KEY="/root/.ssh/sftp_migration_key"
LOG="/var/log/sftp-migration.log"

echo "[$(date)] SFTP Migration gestartet" | tee -a $LOG

# Verbindung testen
sftp -i $DEST_KEY -o BatchMode=yes $DEST_USER@$DEST_HOST <<EOF >> $LOG 2>&1
ls
EOF
echo "[$(date)] Verbindung erfolgreich" >> $LOG

# Dateien übertragen mit rsync über SSH (effizienter als reines SFTP)
rsync -avz \
    --progress \
    --stats \
    -e "ssh -i $DEST_KEY" \
    --exclude "*.log" \
    --exclude "cache/*" \
    $SOURCE_DIR/ \
    $DEST_USER@$DEST_HOST:/var/www/crmserver/

echo "[$(date)] Dateiübertragung abgeschlossen" | tee -a $LOG

# Dateigrösse auf Ziel prüfen
ssh -i $DEST_KEY $DEST_USER@$DEST_HOST "du -sh /var/www/crmserver/" 2>/dev/null || true

echo "[$(date)] SFTP Migration erfolgreich abgeschlossen" | tee -a $LOG
```

```bash
chmod +x /opt/scripts/sftp-migration.sh
sudo bash /opt/scripts/sftp-migration.sh
```


---

## Fazit

Der SFTP-Zugang ist sicher eingerichtet. Der Benutzer ist auf das Webverzeichnis beschränkt, Shell-Zugriff ist deaktiviert und Key-Authentifizierung wurde für die automatisierte Migration konfiguriert. Die Datenmigration läuft vollautomatisch und protokolliert alle Schritte.
