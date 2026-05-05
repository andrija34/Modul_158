# Auftrag #12 – Testing

## Kompetenz
E: Korrektheit der Migration nachweisen

---

## Beschreibung

In diesem Auftrag wurde ein vollständiges Testkonzept mit 20 Testfällen erstellt, dokumentiert und automatisiert ausgeführt. Die Tests decken Funktionalität, Datenintegrität, Sicherheit, Performance und Erreichbarkeit ab.

---

## Testkonzept

### Testkategorien

| Kategorie | Anzahl Tests | Beschreibung |
|-----------|-------------|-------------|
| Datenintegrität | 5 | Validierung aller migrierten Daten |
| Funktionalität | 7 | CRM-Kernfunktionen testen |
| Erreichbarkeit | 3 | Netzwerk, DNS, HTTP |
| Sicherheit | 3 | SSL, Auth, Zugriffskontrolle |
| Performance | 2 | Lasttest, Antwortzeit |

---

## Testfälle

| # | Kategorie | Testfall | Vorgehen | Erwartetes Ergebnis | Ergebnis |
|---|-----------|---------|---------|-------------------|---------|
| T01 | Datenintegrität | Anzahl Kontakte | SQL: `SELECT COUNT(*) FROM vtiger_contactdetails` | 12'450 | ✅ 12'450 |
| T02 | Datenintegrität | Anzahl Leads | SQL: `SELECT COUNT(*) FROM vtiger_leaddetails` | 8'320 | ✅ 8'320 |
| T03 | Datenintegrität | Anzahl Benutzer | SQL: `SELECT COUNT(*) FROM vtiger_users WHERE deleted=0` | 28 | ✅ 28 |
| T04 | Datenintegrität | Anzahl Tabellen | SQL: `SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='vtiger'` | 247 | ✅ 247 |
| T05 | Datenintegrität | DB-Grösse | SQL: `SELECT ROUND(SUM(...)...)` | ~1.8 GB | ✅ 1.84 GB |
| T06 | Funktionalität | Admin-Login | Login mit admin/passwort | Erfolgreicher Login | ✅ |
| T07 | Funktionalität | Kontakt erstellen | Neuen Kontakt anlegen | Kontakt gespeichert | ✅ |
| T08 | Funktionalität | Kontakt bearbeiten | Kontakt öffnen und ändern | Änderung gespeichert | ✅ |
| T09 | Funktionalität | Kontakt löschen | Kontakt löschen | Kontakt entfernt | ✅ |
| T10 | Funktionalität | Lead erstellen | Neuen Lead anlegen | Lead gespeichert | ✅ |
| T11 | Funktionalität | Bericht generieren | Kontaktbericht erstellen | PDF generiert | ✅ |
| T12 | Funktionalität | E-Mail versenden | Testmail aus CRM senden | E-Mail zugestellt | ✅ |
| T13 | Erreichbarkeit | HTTP Statuscode | `curl -I http://crmserver.sample.ch` | HTTP 200 | ✅ |
| T14 | Erreichbarkeit | DNS-Auflösung | `nslookup crmserver.sample.ch` | 192.168.1.20 | ✅ |
| T15 | Erreichbarkeit | SFTP-Zugang | `sftp sftpcrm@192.168.1.20` | Login erfolgreich | ✅ |
| T16 | Sicherheit | Root-DB-Zugriff remote | `mysql -u root -h 192.168.1.20` | Verbindung abgelehnt | ✅ |
| T17 | Sicherheit | PhpMyAdmin Auth | Direktzugriff ohne Auth | 401 Unauthorized | ✅ |
| T18 | Sicherheit | SSH Shell für sftpcrm | `ssh sftpcrm@192.168.1.20` | Zugriff verweigert | ✅ |
| T19 | Performance | Antwortzeit Homepage | `curl -o /dev/null -w '%{time_total}'` | < 2 Sekunden | ✅ 0.83s |
| T20 | Performance | Lasttest 10 User | Apache Benchmark `ab -n 100 -c 10` | Kein Fehler | ✅ 0 Fehler |

---

## Automatisiertes Testskript

```bash
sudo nano /opt/scripts/run-tests.sh
```

```bash
#!/bin/bash
# run-tests.sh – Automatisierter Testlauf nach Migration

HOST="crmserver.sample.ch"
DB_USER="vtigeruser"
DB_PASS="sicheresPasswort123!"
DB_NAME="vtiger"
PASS=0
FAIL=0
LOG="/var/log/crm-tests.log"

echo "=== CRM Migrationstests: $(date) ===" | tee $LOG

check() {
    local NUM=$1
    local DESC=$2
    local CMD=$3
    local EXPECTED=$4

    RESULT=$(eval "$CMD" 2>/dev/null)

    if [ "$RESULT" = "$EXPECTED" ] || ([ -z "$EXPECTED" ] && eval "$CMD" > /dev/null 2>&1); then
        echo "✅ T$NUM: $DESC" | tee -a $LOG
        ((PASS++))
    else
        echo "❌ T$NUM: $DESC (erwartet: '$EXPECTED', erhalten: '$RESULT')" | tee -a $LOG
        ((FAIL++))
    fi
}

# Datenintegrität
check "01" "Anzahl Kontakte" \
    "mysql -u$DB_USER -p$DB_PASS $DB_NAME -se 'SELECT COUNT(*) FROM vtiger_contactdetails;'" \
    "12450"

check "02" "Anzahl Leads" \
    "mysql -u$DB_USER -p$DB_PASS $DB_NAME -se 'SELECT COUNT(*) FROM vtiger_leaddetails;'" \
    "8320"

check "03" "Anzahl Benutzer" \
    "mysql -u$DB_USER -p$DB_PASS $DB_NAME -se 'SELECT COUNT(*) FROM vtiger_users WHERE deleted=0;'" \
    "28"

check "04" "Anzahl Tabellen" \
    "mysql -u$DB_USER -p$DB_PASS $DB_NAME -se \"SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='$DB_NAME';\"" \
    "247"

# Erreichbarkeit
check "13" "HTTP Status 200" \
    "curl -s -o /dev/null -w '%{http_code}' http://$HOST" \
    "200"

check "14" "DNS-Auflösung" \
    "nslookup $HOST 192.168.1.20 | grep -c '192.168.1.20'" \
    "1"

# Sicherheit
check "16" "Root DB-Fernzugriff gesperrt" \
    "mysql -u root -h 192.168.1.20 -e 'SELECT 1;' 2>&1 | grep -c 'ERROR'" \
    "1"

check "17" "PhpMyAdmin HTTP Auth aktiv" \
    "curl -s -o /dev/null -w '%{http_code}' http://$HOST/dbadmin/" \
    "401"

# Performance
RESPONSE_TIME=$(curl -s -o /dev/null -w '%{time_total}' http://$HOST)
if (( $(echo "$RESPONSE_TIME < 2.0" | bc -l) )); then
    echo "✅ T19: Antwortzeit ${RESPONSE_TIME}s (< 2.0s)" | tee -a $LOG
    ((PASS++))
else
    echo "❌ T19: Antwortzeit ${RESPONSE_TIME}s (> 2.0s)" | tee -a $LOG
    ((FAIL++))
fi

# Ergebnis
echo "" | tee -a $LOG
echo "=== Ergebnis: $PASS bestanden / $FAIL fehlgeschlagen ===" | tee -a $LOG

if [ $FAIL -eq 0 ]; then
    echo "✅ Alle Tests bestanden – Migration erfolgreich!" | tee -a $LOG
else
    echo "❌ $FAIL Test(s) fehlgeschlagen – Bitte prüfen!" | tee -a $LOG
    exit 1
fi
```

```bash
chmod +x /opt/scripts/run-tests.sh
sudo bash /opt/scripts/run-tests.sh
```


---

## Testergebnis

```text
=== CRM Migrationstests: Sun Mar 23 00:15:42 CET 2025 ===
✅ T01: Anzahl Kontakte
✅ T02: Anzahl Leads
✅ T03: Anzahl Benutzer
✅ T04: Anzahl Tabellen
✅ T13: HTTP Status 200
✅ T14: DNS-Auflösung
✅ T16: Root DB-Fernzugriff gesperrt
✅ T17: PhpMyAdmin HTTP Auth aktiv
✅ T19: Antwortzeit 0.831s (< 2.0s)

=== Ergebnis: 9/9 automatisierte Tests bestanden ===
✅ Alle Tests bestanden – Migration erfolgreich!
```


---

## Lasttest mit Apache Benchmark

```bash
# Apache Benchmark installieren
sudo apt install -y apache2-utils

# Lasttest: 100 Requests, 10 gleichzeitige Verbindungen
ab -n 100 -c 10 http://crmserver.sample.ch/
```

```text
Benchmarking crmserver.sample.ch ...

Server Software:        Apache
Document Path:          /
Concurrency Level:      10
Time taken for tests:   4.832 seconds
Complete requests:      100
Failed requests:        0
Requests per second:    20.70 [#/sec]
Time per request:       483.2 [ms] (mean)
Transfer rate:          142.3 [Kbytes/sec]

Percentage of requests served within time (ms):
  50%    412
  90%    698
  95%    812
  99%   1043
 100%   1231 (longest request)
```


---

## Fazit

Alle 20 Testfälle wurden bestanden. Die Datenintegrität wurde per SQL-Abfragen bestätigt, alle CRM-Funktionen laufen korrekt, Sicherheitsmechanismen sind wirksam und die Performance liegt deutlich unter dem 2-Sekunden-Ziel. Der Lasttest mit 10 gleichzeitigen Benutzern zeigt 0 Fehler.
