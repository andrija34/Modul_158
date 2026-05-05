# Auftrag #6 – PHP

---

## Beschreibung

In diesem Auftrag wurde PHP 8.2 installiert und vollständig konfiguriert. Da auf dem IST-System noch PHP 7.2 läuft, wurde ein schrittweises Upgrade dokumentiert und eine parallele Installation eingerichtet, um die Kompatibilität der Anwendung zu testen, bevor der definitive Wechsel erfolgt.

---

## Ausgangslage IST

```bash
# PHP Version IST-System
php -v
# PHP 7.2.24-0ubuntu0.18.04.17 (cli)

# Aktive PHP-Module
php -m
# Core, date, json, mysql, mysqli, pdo_mysql, xml, ...

# PHP Konfiguration
php -i | grep -E "memory_limit|upload_max|max_execution"
# memory_limit => 128M
# upload_max_filesize => 2M
# max_execution_time => 30
```

---

## Schritt 1 – PHP 8.2 installieren

```bash
# Ondřej Surý PPA hinzufügen (enthält aktuelle PHP Versionen)
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# PHP 8.2 und benötigte Extensions installieren
sudo apt install -y \
    php8.2 \
    php8.2-cli \
    php8.2-common \
    php8.2-mysql \
    php8.2-xml \
    php8.2-xmlrpc \
    php8.2-curl \
    php8.2-mbstring \
    php8.2-zip \
    php8.2-gd \
    php8.2-intl \
    php8.2-soap \
    php8.2-bcmath \
    php8.2-imagick \
    libapache2-mod-php8.2

# Version prüfen
php8.2 -v
# PHP 8.2.15 (cli)
```


---

## Schritt 2 – Parallele Installation (PHP 7.2 & 8.2)

Beide PHP-Versionen wurden parallel installiert, um die Vtiger-Kompatibilität zuerst zu testen:

```bash
# Beide Versionen sind verfügbar
php7.2 -v
php8.2 -v

# Für Apache: PHP 8.2 aktivieren
sudo a2dismod php7.2
sudo a2enmod php8.2
sudo systemctl restart apache2

# Für CLI: Standard setzen
sudo update-alternatives --set php /usr/bin/php8.2
sudo update-alternatives --set php-config /usr/bin/php-config8.2
sudo update-alternatives --set phpize /usr/bin/phpize8.2

# Aktive Version prüfen
php -v
# PHP 8.2.15
```


---

## Schritt 3 – PHP Konfiguration anpassen

```bash
sudo nano /etc/php/8.2/apache2/php.ini
```

```ini
; Zeitzone
date.timezone = Europe/Zurich

; Speicher und Limits
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
max_input_vars = 10000

; Upload Limits
upload_max_filesize = 64M
post_max_size = 64M

; Fehlerbehandlung (Produktion)
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php/php_errors.log
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT

; Session
session.gc_maxlifetime = 3600
session.cookie_httponly = 1
session.cookie_secure = 0

; OPcache aktivieren
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 60
```

```bash
# Log-Verzeichnis erstellen
sudo mkdir -p /var/log/php
sudo chown www-data:www-data /var/log/php

sudo systemctl restart apache2
```


---

## Schritt 4 – PHP Konfiguration testen

```bash
# PHP Info Seite erstellen (temporär)
echo "<?php phpinfo(); ?>" | sudo tee /var/www/crmserver/info.php

# Über Browser prüfen
curl http://crmserver.sample.ch/info.php | grep -E "PHP Version|memory_limit|upload_max"

# Nach Test entfernen (Sicherheit)
sudo rm /var/www/crmserver/info.php
```


---

## Schritt 5 – Vtiger Kompatibilität prüfen

Vtiger CRM 7.5 benötigt spezifische PHP-Extensions:

```bash
# Benötigte Extensions prüfen
php -m | grep -E "curl|gd|mbstring|mysql|xml|zip|soap|bcmath"

# OPcache Status prüfen
php -r "echo opcache_get_status()['opcache_enabled'] ? 'OPcache aktiv' : 'OPcache inaktiv';"
# OPcache aktiv
```

| Extension | Benötigt | Status |
|-----------|---------|--------|
| curl | ✅ | ✅ installiert |
| gd | ✅ | ✅ installiert |
| mbstring | ✅ | ✅ installiert |
| mysqli | ✅ | ✅ installiert |
| pdo_mysql | ✅ | ✅ installiert |
| xml | ✅ | ✅ installiert |
| zip | ✅ | ✅ installiert |
| soap | ✅ | ✅ installiert |
| bcmath | ✅ | ✅ installiert |
| intl | ✅ | ✅ installiert |


---

## Schritt 6 – PHP 7.2 entfernen

Nachdem die Kompatibilität bestätigt wurde, wurde PHP 7.2 entfernt:

```bash
sudo apt remove --purge php7.2* -y
sudo apt autoremove -y

# Prüfen ob wirklich entfernt
php7.2 --version
# bash: php7.2: command not found

# Nur noch PHP 8.2 aktiv
php -v
# PHP 8.2.15
```


---

## Fazit

PHP 8.2 läuft korrekt auf dem Zielsystem. Alle von Vtiger CRM 7.5 benötigten Extensions sind installiert. OPcache ist aktiv und verbessert die Performance deutlich. Die Konfiguration wurde für eine Produktionsumgebung optimiert.
