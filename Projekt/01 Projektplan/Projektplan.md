# Auftrag #1 – Projektplan

## Kompetenz
C: Umstellung und Schritte planen

---

## Beschreibung

In diesem Auftrag wurde ein vollständiger Projektplan für die CRM-Migration von `crmserver.sample.ch` erstellt. Das Ziel ist die Migration auf ein neues OS mit neuem Web/DB-Server bei minimalem Ausfall.

---

## Projektplan

### Milestones

| Milestone | Beschreibung | Datum |
|-----------|-------------|-------|
| M1 | Projektstart & IST-Analyse abgeschlossen | 03.03.2025 |
| M2 | Testumgebung aufgebaut | 10.03.2025 |
| M3 | Zielsystem konfiguriert | 17.03.2025 |
| M4 | Migration durchgeführt | 24.03.2025 |
| M5 | Tests abgeschlossen & Abnahme | 31.03.2025 |

### Aufgaben & Aufwände

| Aufgabe | Verantwortlich | Aufwand (h) |
|---------|---------------|-------------|
| IST-Analyse | Andrija | 4h |
| Architekturdiagramm | Andrija | 3h |
| Testumgebung aufbauen | Andrija | 5h |
| DNS konfigurieren | Andrija | 2h |
| Webserver installieren | Andrija | 3h |
| PHP installieren | Andrija | 2h |
| MySQL installieren | Andrija | 3h |
| PhpMyAdmin einrichten | Andrija | 2h |
| SFTP einrichten | Andrija | 2h |
| Migration durchführen | Andrija | 6h |
| Backup einrichten | Andrija | 3h |
| Testing | Andrija | 4h |
| Monitoring einrichten | Andrija | 3h |
| Deployment | Andrija | 3h |

### Critical Path
IST-Analyse → Architekturdiagramm → Testumgebung → Zielsystem → Migration → Tests → Deployment


---

## Kommunikationsplan

| Stakeholder | Kanal | Frequenz | Inhalt |
|-------------|-------|----------|--------|
| Kunde | E-Mail | Wöchentlich | Statusbericht |
| Kunde | Meeting | Bei Milestone | Abnahme & Freigabe |
| Benutzer | E-Mail | 3 Tage vor Migration | Wartungsankündigung |
| Benutzer | E-Mail | Nach Migration | Bestätigung & neue Infos |

---

## Migrationsfenster

Das System wird Mo–Sa aktiv genutzt. Der Ausfall soll minimal sein.

| Datum | Zeitfenster | Beschreibung |
|-------|-------------|-------------|
| Sa, 22.03.2025 | 22:00 – 02:00 | Hauptmigration (4h Fenster) |
| So, 23.03.2025 | 08:00 – 10:00 | Reserve / Rollback falls nötig |

**Rollback-Plan:** Falls die Migration fehlschlägt, wird innerhalb von 30 Minuten auf den alten Server zurückgewechselt. DNS TTL wurde 48h vorher auf 60 Sekunden gesetzt.

---

## Fazit

Der Projektplan wurde vollständig erstellt, mit allen Milestones, Aufgaben und Aufwänden. Das Migrationsfenster wurde bewusst auf Samstagabend gelegt, um den Betrieb minimal zu stören. Die Planung wurde laufend kontrolliert und bei Bedarf angepasst.
