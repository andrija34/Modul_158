# Auftrag #3 – Umgebung aufbauen/einrichten

## Kompetenz
B: Software in Testumgebung in Betrieb nehmen

---

## Beschreibung

In diesem Auftrag wurde eine vollständige Testumgebung mit zwei VMs aufgebaut. Das IST-System wurde aus dem gelieferten OVA-Export eingebunden. Das neue Zielsystem wurde von Grund auf aufgebaut und konfiguriert. Beide Systeme wurden verglichen und die Migration realitätsnah vorbereitet. Snapshots sichern jeden relevanten Stand.

---

## Übersicht Testumgebung

| VM | Rolle | OS | IP | RAM | CPU | Disk |
|----|-------|----|----|-----|-----|------|
| vm-crm-alt | IST-System (Quelle) | Ubuntu 18.04 | 192.168.1.10 | 4 GB | 2 vCPU | 80 GB |
| vm-crm-neu | SOLL-System (Ziel) | Ubuntu 24.04 | 192.168.1.20 | 8 GB | 4 vCPU | 120 GB |

Beide VMs laufen im selben privaten Netzwerk und können miteinander kommunizieren. Der Zugriff von aussen erfolgt über SSH.

---

## IST-System einbinden

### OVA Import

```bash
# OVA-Export des Kunden importieren
VBoxManage import crmserver-export.ova \
    --vsys 0 \
    --vmname "vm-crm-alt" \
    --memory 4096 \
    --cpus 2

# VM starten
VBoxManage startvm vm-crm-alt --type headless

# Status prüfen
VBoxManage showvminfo vm-crm-alt | grep -E "State|IP"
```

---

### IST-System prüfen

```bash
# SSH Verbindung
ssh ubuntu@192.168.1.10

# Betriebssystem prüfen
lsb_release -a
# Ubuntu 18.04.6 LTS

# Laufende Dienste prüfen
systemctl list-units --type=service --state=running | grep -E "apache|mysql|php"

# Offene Ports prüfen
ss -tlnp | grep -E "80|443|3306"

# Disk-Nutzung prüfen
df -h

# Speichernutzung
free -h
```

---

## Zielsystem aufbauen

### Ubuntu 24.04 installieren

```bash
# Cloud Image herunterladen
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# VM erstellen
VBoxManage createvm --name "vm-crm-neu" --ostype Ubuntu_64 --register
VBoxManage modifyvm "vm-crm-neu" --memory 8192 --cpus 4
VBoxManage modifyvm "vm-crm-neu" --nic1 bridged --bridgeadapter1 eth0

# VM starten
VBoxManage startvm vm-crm-neu --type headless
```

---

### System aktualisieren und Grundkonfiguration

```bash
# SSH Verbindung zum neuen System
ssh ubuntu@192.168.1.20

# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Zeitzone setzen
sudo timedatectl set-timezone Europe/Zurich

# Hostname setzen
sudo hostnamectl set-hostname crmserver-neu

# Grundpakete installieren
sudo apt install -y curl wget git unzip htop net-tools ufw fail2ban

# Firewall konfigurieren
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status
```

---

### Fail2Ban konfigurieren

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

---

## Snapshots

Snapshots wurden nach jedem wichtigen Schritt erstellt, um jederzeit auf einen funktionierenden Zustand zurückgehen zu können:

```bash
# Snapshot IST-System vor Migration
VBoxManage snapshot vm-crm-alt take "vor-migration" \
    --description "Stand vor Migrationsbeginn – 22.03.2025 21:00"

# Snapshot SOLL-System nach Grundkonfiguration
VBoxManage snapshot vm-crm-neu take "grundkonfiguration" \
    --description "Ubuntu 24.04 Grundkonfiguration abgeschlossen"

# Snapshot SOLL-System nach Webserver + PHP + DB
VBoxManage snapshot vm-crm-neu take "dienste-installiert" \
    --description "Apache, PHP 8.2, MariaDB installiert und konfiguriert"

# Snapshot SOLL-System nach Migration
VBoxManage snapshot vm-crm-neu take "nach-migration" \
    --description "CRM erfolgreich migriert – noch vor DNS-Umstellung"

# Alle Snapshots anzeigen
VBoxManage snapshot vm-crm-neu list
```

---

## Vergleich IST vs. SOLL

| Kriterium | IST-System | SOLL-System |
|-----------|-----------|------------|
| Betriebssystem | Ubuntu 18.04 (EOL) | Ubuntu 24.04 LTS |
| Webserver | Apache 2.4.29 | Apache 2.4.58 |
| PHP | 7.2 (EOL) | 8.2 |
| Datenbank | MySQL 5.7 | MariaDB 10.11 |
| CRM | Vtiger 7.1 | Vtiger 7.5 |
| SSL | Nein | Let's Encrypt |
| Backup | Manuell | Automatisiert (täglich) |
| Monitoring | Keines | Prometheus + Grafana |
| Firewall | Keine | UFW + Fail2Ban |
| WAF | Keine | ModSecurity |
| RAM | 4 GB | 8 GB |
| CPU | 2 vCPU | 4 vCPU |

---

## Automatisiertes Setup-Skript

```bash
#!/bin/bash
# setup.sh – Automatisches Grundsetup des Zielsystems

set -e
LOG="/var/log/crm-setup.log"

echo "=== Setup gestartet: $(date) ===" | tee -a $LOG

# System aktualisieren
apt update && apt upgrade -y >> $LOG 2>&1
echo "[OK] System aktualisiert" | tee -a $LOG

# Zeitzone
timedatectl set-timezone Europe/Zurich
echo "[OK] Zeitzone gesetzt" | tee -a $LOG

# Grundpakete
apt install -y curl wget git unzip htop net-tools ufw fail2ban >> $LOG 2>&1
echo "[OK] Grundpakete installiert" | tee -a $LOG

# Firewall
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
echo "[OK] Firewall konfiguriert" | tee -a $LOG

echo "=== Setup abgeschlossen: $(date) ===" | tee -a $LOG
```

```bash
chmod +x setup.sh
sudo bash setup.sh
```

---

## Fazit

Die Testumgebung wurde vollständig aufgebaut. Das IST-System wurde eingebunden und analysiert. Das Zielsystem wurde von Grund auf konfiguriert und ist für die Migration bereit. Snapshots sichern jeden wichtigen Stand. Das automatisierte Setup-Skript ermöglicht eine reproduzierbare Umgebung.
