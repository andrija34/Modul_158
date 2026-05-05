# Auftrag #3 – Umgebung aufbauen/einrichten

## Kompetenz
B: Software in Testumgebung in Betrieb nehmen

---

## Beschreibung

Es wurde eine vollständige Testumgebung aufgebaut. Das IST-System wurde eingebunden, die neue Zielumgebung aufgebaut und verglichen. Snapshots sichern jeden Stand.

---

## Testumgebung

### VMs

| VM | Rolle | OS | IP | RAM | CPU |
|----|-------|----|----|-----|-----|
| vm-crm-alt | IST-System | Ubuntu 18.04 | 192.168.1.10 | 4 GB | 2 vCPU |
| vm-crm-neu | SOLL-System | Ubuntu 24.04 | 192.168.1.20 | 8 GB | 4 vCPU |

---

## IST-System einbinden

```bash
# VM Import (OVA Export vom Kunden)
VBoxManage import crmserver-export.ova

# VM starten
VBoxManage startvm vm-crm-alt --type headless

# SSH Verbindung prüfen
ssh ubuntu@192.168.1.10
```

---

## Zielsystem aufbauen

```bash
# Ubuntu 24.04 installieren
# (via ISO / Cloud Image)

# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Grundpakete installieren
sudo apt install -y curl wget git unzip
```

---

## Snapshots

```bash
# Snapshot IST-System vor Migration
VBoxManage snapshot vm-crm-alt take "vor-migration" --description "Stand vor Migration"

# Snapshot SOLL-System nach Grundkonfiguration
VBoxManage snapshot vm-crm-neu take "grundkonfiguration" --description "Grundkonfiguration abgeschlossen"
```

---

## Vergleich IST vs. SOLL

| Kriterium | IST | SOLL |
|-----------|-----|------|
| OS | Ubuntu 18.04 | Ubuntu 24.04 LTS |
| Webserver | Apache 2.4.29 | Apache 2.4.58 |
| PHP | 7.2 | 8.2 |
| Datenbank | MySQL 5.7 | MariaDB 10.11 |
| Sicherheit | Keine WAF | ModSecurity aktiv |
| Backup | Manuell | Automatisiert |

---

## Automatisierung

```bash
#!/bin/bash
# setup.sh – Automatisches Setup des Zielsystems

apt update && apt upgrade -y
apt install -y apache2 php8.2 mariadb-server phpmyadmin
systemctl enable apache2 mariadb
echo "Setup abgeschlossen"
```
