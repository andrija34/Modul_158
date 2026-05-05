# Auftrag #4 – DNS

---

## Beschreibung

In diesem Auftrag wurde die Namensauflösung für das CRM-System eingerichtet. Zunächst wurde eine lokale Lösung via `/etc/hosts` umgesetzt. Anschliessend wurde ein vollständiger DNS-Server mit BIND9 eingerichtet, der die Auflösung für die gesamte Testumgebung übernimmt. Die DNS TTL wurde vor der Migration bewusst gesenkt, um einen schnellen Wechsel zu ermöglichen.

---

## Ausgangslage

Das bestehende System war unter `crmserver.sample.ch` erreichbar. Für die Testumgebung musste die Namensauflösung lokal eingerichtet werden, damit der neue Server bereits unter demselben Hostnamen erreichbar ist, ohne den produktiven DNS zu verändern.

---

## Schritt 1 – Lokale Namensauflösung via /etc/hosts

Die schnellste Lösung für die Testumgebung war ein Eintrag in der lokalen hosts-Datei:

```bash
sudo nano /etc/hosts
```

```text
# CRM Testumgebung
192.168.1.10    crmserver-alt.sample.ch
192.168.1.20    crmserver.sample.ch
192.168.1.20    www.crmserver.sample.ch
```

```bash
# Auflösung testen
ping -c 3 crmserver.sample.ch
# PING crmserver.sample.ch (192.168.1.20): 56 data bytes
# 64 bytes from 192.168.1.20: icmp_seq=0 ttl=64 time=0.312 ms

# Hostname auflösen
getent hosts crmserver.sample.ch
# 192.168.1.20    crmserver.sample.ch
```


---

## Schritt 2 – DNS-Server mit BIND9

Für eine produktionsnahe Lösung wurde BIND9 als DNS-Server eingerichtet:

### Installation

```bash
sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils

# Dienst aktivieren
sudo systemctl enable --now named

# Status prüfen
sudo systemctl status named
```


---

### BIND9 Konfiguration – named.conf.options

```bash
sudo nano /etc/bind/named.conf.options
```

```text
options {
    directory "/var/cache/bind";

    # Nur aus dem lokalen Netzwerk akzeptieren
    allow-query { localhost; 192.168.1.0/24; };
    allow-recursion { localhost; 192.168.1.0/24; };

    # Forwarding an öffentliche DNS
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;
    listen-on { any; };
};
```

---

### BIND9 Konfiguration – named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

```text
# Forward Zone
zone "sample.ch" {
    type master;
    file "/etc/bind/db.sample.ch";
};

# Reverse Zone
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};
```

---

### Zone File – db.sample.ch

```bash
sudo nano /etc/bind/db.sample.ch
```

```text
$TTL    60
@   IN  SOA ns1.sample.ch. admin.sample.ch. (
            2025032401  ; Serial (JJJJMMTTNN)
            3600        ; Refresh
            900         ; Retry
            604800      ; Expire
            60 )        ; Negative Cache TTL

; Nameserver
@       IN  NS      ns1.sample.ch.

; A-Records
ns1         IN  A       192.168.1.20
crmserver   IN  A       192.168.1.20
www         IN  CNAME   crmserver.sample.ch.
```

---

### Reverse Zone File – db.192.168.1

```bash
sudo nano /etc/bind/db.192.168.1
```

```text
$TTL    60
@   IN  SOA ns1.sample.ch. admin.sample.ch. (
            2025032401
            3600
            900
            604800
            60 )

@       IN  NS  ns1.sample.ch.
20      IN  PTR crmserver.sample.ch.
```

---

### BIND9 Syntax prüfen und neu starten

```bash
# Syntax prüfen
sudo named-checkconf
sudo named-checkzone sample.ch /etc/bind/db.sample.ch
# zone sample.ch/IN: loaded serial 2025032401
# OK

sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
# OK

# Dienst neu starten
sudo systemctl restart named
```


---

## Schritt 3 – DNS testen

```bash
# Forward Lookup
nslookup crmserver.sample.ch 192.168.1.20
# Server:   192.168.1.20
# Address:  192.168.1.20#53
# Name:    crmserver.sample.ch
# Address: 192.168.1.20

# Reverse Lookup
nslookup 192.168.1.20 192.168.1.20
# 20.1.168.192.in-addr.arpa  name = crmserver.sample.ch.

# dig Test
dig @192.168.1.20 crmserver.sample.ch
# ;; ANSWER SECTION:
# crmserver.sample.ch.  60  IN  A  192.168.1.20

# CNAME Test
dig @192.168.1.20 www.sample.ch
# ;; ANSWER SECTION:
# www.sample.ch.  60  IN  CNAME  crmserver.sample.ch.
```


---

## Schritt 4 – TTL für Migration senken

48 Stunden vor der Migration wurde die TTL auf 60 Sekunden gesenkt, damit der DNS-Wechsel nach der Migration innerhalb von 60 Sekunden für alle Clients wirksam ist:

```bash
# TTL in der Zone anpassen
sudo nano /etc/bind/db.sample.ch
# $TTL von 604800 auf 60 setzen

# Serial erhöhen
# 2025032001 → 2025032002

# Dienst neu laden
sudo rndc reload
sudo rndc flush

# TTL prüfen
dig @192.168.1.20 crmserver.sample.ch | grep TTL
# crmserver.sample.ch.  60  IN  A  192.168.1.20
```


---

## Fazit

Die Namensauflösung funktioniert sowohl lokal über `/etc/hosts` als auch über den BIND9-DNS-Server korrekt. Forward- und Reverse-Lookups wurden erfolgreich getestet. Die TTL wurde rechtzeitig gesenkt, um den DNS-Wechsel während der Migration zu beschleunigen.
