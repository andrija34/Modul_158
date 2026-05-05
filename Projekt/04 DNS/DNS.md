# Auftrag #4 – DNS

## Beschreibung

Die Namensauflösung wurde lokal über `/etc/hosts` konfiguriert und zusätzlich ein eigener DNS-Server mit BIND9 eingerichtet.

---

## Lokale Namensauflösung via hosts

```bash
sudo nano /etc/hosts
```

```text
192.168.1.20    crmserver.sample.ch
192.168.1.20    www.crmserver.sample.ch
```

```bash
# Testen
ping crmserver.sample.ch
```

---

## DNS-Server mit BIND9

```bash
# BIND9 installieren
sudo apt install -y bind9 bind9utils

# Zone konfigurieren
sudo nano /etc/bind/named.conf.local
```

```text
zone "sample.ch" {
    type master;
    file "/etc/bind/db.sample.ch";
};
```

```bash
sudo nano /etc/bind/db.sample.ch
```

```text
$TTL    604800
@   IN  SOA ns1.sample.ch. admin.sample.ch. (
            2025030401 ; Serial
            604800     ; Refresh
            86400      ; Retry
            2419200    ; Expire
            604800 )   ; Negative Cache TTL

@       IN  NS      ns1.sample.ch.
@       IN  A       192.168.1.20
ns1     IN  A       192.168.1.20
crmserver IN A      192.168.1.20
www     IN  CNAME   crmserver
```

```bash
# BIND9 neu starten
sudo systemctl restart bind9

# Namensauflösung testen
nslookup crmserver.sample.ch 192.168.1.20
dig crmserver.sample.ch @192.168.1.20
```

---

## Ergebnis

Die Namensauflösung funktioniert sowohl lokal via `/etc/hosts` als auch über den BIND9 DNS-Server korrekt.
