# Auftrag #5 – Webserver

---

## Beschreibung

In diesem Auftrag wurde Apache2 in der neusten verfügbaren Version installiert und vollständig konfiguriert. Ein VirtualHost für das CRM-System wurde eingerichtet. ModSecurity wurde als Web Application Firewall aktiviert. Unnötige Module wurden deaktiviert und Performance-Optimierungen vorgenommen.

---

## Ausgangslage

Das IST-System nutzt Apache 2.4.29 ohne SSL, ohne WAF und mit einer minimalen Standardkonfiguration. Auf dem neuen System soll Apache 2.4.58 mit ModSecurity, SSL und einem sauber konfigurierten VirtualHost laufen.

---

## Schritt 1 – Apache installieren

```bash
sudo apt update
sudo apt install -y apache2

# Version prüfen
apache2 -v
# Server version: Apache/2.4.58 (Ubuntu)
# Server built:   2024-01-13T00:00:00

# Dienst aktivieren
sudo systemctl enable --now apache2

# Status prüfen
sudo systemctl status apache2
```


---

## Schritt 2 – Verzeichnisstruktur erstellen

```bash
# Webroot erstellen
sudo mkdir -p /var/www/crmserver
sudo chown -R www-data:www-data /var/www/crmserver
sudo chmod -R 755 /var/www/crmserver

# Log-Verzeichnis
sudo mkdir -p /var/log/apache2/crm
```

---

## Schritt 3 – VirtualHost konfigurieren

```bash
sudo nano /etc/apache2/sites-available/crmserver.conf
```

```apacheconf
<VirtualHost *:80>
    ServerName crmserver.sample.ch
    ServerAlias www.crmserver.sample.ch
    DocumentRoot /var/www/crmserver

    # Logging
    ErrorLog /var/log/apache2/crm/error.log
    CustomLog /var/log/apache2/crm/access.log combined

    # Verzeichnis-Konfiguration
    <Directory /var/www/crmserver>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Sicherheits-Header
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # Server-Informationen verstecken
    ServerTokens Prod
    ServerSignature Off
</VirtualHost>
```

```bash
# VirtualHost aktivieren
sudo a2ensite crmserver.conf

# Standard-Site deaktivieren
sudo a2dissite 000-default.conf

# Apache neu laden
sudo systemctl reload apache2
```

---

## Schritt 4 – Module aktivieren/deaktivieren

```bash
# Benötigte Module aktivieren
sudo a2enmod rewrite      # URL Rewriting
sudo a2enmod headers      # HTTP Header setzen
sudo a2enmod deflate      # Komprimierung
sudo a2enmod expires      # Cache-Control Header
sudo a2enmod ssl          # SSL/TLS
sudo a2enmod security2    # ModSecurity WAF

# Unnötige Module deaktivieren
sudo a2dismod status      # Server Status
sudo a2dismod autoindex   # Verzeichnis-Listing
sudo a2dismod userdir     # Benutzerverzeichnisse

sudo systemctl restart apache2
```


---

## Schritt 5 – ModSecurity einrichten

```bash
# ModSecurity installieren
sudo apt install -y libapache2-mod-security2

# Konfiguration aktivieren
sudo cp /etc/modsecurity/modsecurity.conf-recommended \
        /etc/modsecurity/modsecurity.conf

# ModSecurity in Enforcement Mode setzen
sudo nano /etc/modsecurity/modsecurity.conf
```

```text
# Zeile ändern:
# SecRuleEngine DetectionOnly
SecRuleEngine On

# Request Body Limit
SecRequestBodyLimit 13107200
SecRequestBodyNoFilesLimit 131072

# Response Body
SecResponseBodyAccess On
SecResponseBodyLimit 524288

# Logging
SecAuditLog /var/log/apache2/modsecurity_audit.log
SecAuditLogParts ABIJDEFHZ
```

```bash
# OWASP CRS installieren
sudo apt install -y modsecurity-crs

sudo systemctl restart apache2

# ModSecurity Status prüfen
sudo tail -f /var/log/apache2/modsecurity_audit.log
```


---

## Schritt 6 – Performance Optimierung

```bash
sudo nano /etc/apache2/mods-available/deflate.conf
```

```apacheconf
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/xml
    AddOutputFilterByType DEFLATE text/css text/javascript
    AddOutputFilterByType DEFLATE application/javascript application/json
    DeflateCompressionLevel 6
</IfModule>
```

```bash
sudo nano /etc/apache2/mods-available/expires.conf
```

```apacheconf
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresByType image/jpeg "access plus 1 month"
    ExpiresByType image/png "access plus 1 month"
    ExpiresByType text/css "access plus 1 week"
    ExpiresByType application/javascript "access plus 1 week"
</IfModule>
```

```bash
sudo systemctl reload apache2
```


---

## Schritt 7 – Apache testen

```bash
# Konfiguration prüfen
sudo apache2ctl configtest
# Syntax OK

# VirtualHost prüfen
sudo apache2ctl -S
# VirtualHost configuration:
# *:80    crmserver.sample.ch (/etc/apache2/sites-enabled/crmserver.conf)

# HTTP Test
curl -I http://crmserver.sample.ch
# HTTP/1.1 200 OK
# Server: Apache
# X-Content-Type-Options: nosniff
# X-Frame-Options: SAMEORIGIN
```


---

## Fazit

Apache 2.4.58 läuft korrekt mit einem sauber konfigurierten VirtualHost. ModSecurity schützt die Anwendung vor Web-Angriffen. Unnötige Module wurden deaktiviert, Performance-Optimierungen wurden vorgenommen und Sicherheits-Header wurden gesetzt.
