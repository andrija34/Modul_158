# Auftrag #1 – Projektplan

## Kompetenz
C: Umstellung und Schritte planen

---

## Beschreibung

In diesem Auftrag wurde ein vollständiger Projektplan für die CRM-Migration von `crmserver.sample.ch` erstellt. Das bestehende System läuft auf Ubuntu 18.04 mit Vtiger CRM 7.1 und soll auf eine neue Infrastruktur mit Ubuntu 24.04 und Vtiger CRM 7.5 migriert werden. Der Projektplan deckt alle Phasen der Migration ab und berücksichtigt, dass das System Mo–Sa aktiv genutzt wird. Ein Ausfall soll so kurz wie möglich gehalten werden.

---

## Ausgangslage

Das bestehende CRM-System wird von rund 25 Mitarbeitenden täglich genutzt. Es enthält über 12'000 Kundenkontakte, 8'000 Leads sowie historische Verkaufsdaten. Ein unkontrollierter Ausfall würde den Geschäftsbetrieb erheblich beeinträchtigen. Deshalb wurde die Migration sorgfältig geplant und in klar definierte Phasen aufgeteilt.

---

## Projektphasen

| Phase | Beschreibung | Dauer |
|-------|-------------|-------|
| 1 – Planung | Projektplan, Architekturdiagramm, Testkatalog | 1 Woche |
| 2 – Umgebung | VMs aufsetzen, Netzwerk, Snapshots | 1 Woche |
| 3 – Zielsystem | Webserver, PHP, DB, Dienste einrichten | 1 Woche |
| 4 – Migration | Datenmigration, Konfiguration anpassen | 1 Woche |
| 5 – Tests | Testkonzept durchführen, Abnahme | 1 Woche |

---

## Milestones

| Milestone | Beschreibung | Datum | Status |
|-----------|-------------|-------|--------|
| M1 | Kick-off & Projektplan genehmigt | 03.03.2025 | ✅ |
| M2 | IST-Analyse & Architekturdiagramm abgeschlossen | 07.03.2025 | ✅ |
| M3 | Testumgebung vollständig aufgebaut | 14.03.2025 | ✅ |
| M4 | Zielsystem konfiguriert & getestet | 21.03.2025 | ✅ |
| M5 | Migration erfolgreich durchgeführt | 22.03.2025 | ✅ |
| M6 | Abnahme durch Kunde & Projektabschluss | 31.03.2025 | ✅ |

---

## Aufgaben & Aufwände

| # | Aufgabe | Verantwortlich | Aufwand (h) | Abhängigkeit |
|---|---------|---------------|-------------|-------------|
| 1 | IST-Analyse durchführen | Andrija | 4h | – |
| 2 | Architekturdiagramm erstellen | Andrija | 3h | #1 |
| 3 | Testumgebung aufbauen | Andrija | 5h | #2 |
| 4 | DNS konfigurieren | Andrija | 2h | #3 |
| 5 | Webserver installieren & konfigurieren | Andrija | 3h | #3 |
| 6 | PHP installieren & konfigurieren | Andrija | 2h | #5 |
| 7 | MariaDB installieren & konfigurieren | Andrija | 3h | #3 |
| 8 | PhpMyAdmin einrichten | Andrija | 2h | #7 |
| 9 | SFTP einrichten | Andrija | 2h | #3 |
| 10 | Datenmigration durchführen | Andrija | 6h | #5 #6 #7 #9 |
| 11 | Backup einrichten | Andrija | 3h | #10 |
| 12 | Testing durchführen | Andrija | 4h | #10 |
| 13 | Monitoring einrichten | Andrija | 3h | #10 |
| 14 | Deployment automatisieren | Andrija | 3h | #12 |
| **Total** | | | **45h** | |

---

## Critical Path

Der kritische Pfad der Migration verläuft wie folgt. Eine Verzögerung in einem dieser Schritte würde das gesamte Projekt verzögern:
IST-Analyse
→ Architekturdiagramm
→ Testumgebung aufbauen
→ Webserver + PHP + MariaDB
→ Datenmigration
→ Testing
→ Deployment & Abnahme

Alle anderen Aufgaben (DNS, PhpMyAdmin, SFTP, Backup, Monitoring) sind parallel durchführbar und liegen nicht auf dem kritischen Pfad.

---

## Kommunikationsplan

Damit alle Beteiligten jederzeit informiert sind, wurde ein Kommunikationsplan erstellt:

| Stakeholder | Kanal | Frequenz | Inhalt |
|-------------|-------|----------|--------|
| Auftraggeber | E-Mail | Wöchentlich | Statusbericht mit aktuellem Stand |
| Auftraggeber | Meeting | Bei jedem Milestone | Abnahme & Freigabe für nächste Phase |
| Endbenutzer | E-Mail | 1 Woche vor Migration | Ankündigung Wartungsfenster |
| Endbenutzer | E-Mail | 3 Tage vor Migration | Erinnerung mit genauen Zeiten |
| Endbenutzer | E-Mail | Nach Migration | Bestätigung & neue Server-URL |
| Technisches Team | Slack | Täglich | Kurzupdate & offene Punkte |

---

## Migrationsfenster

Das System wird Mo–Sa aktiv genutzt. Deshalb wurde das Migrationsfenster bewusst auf Samstagabend gelegt:

| Zeitpunkt | Aktion |
|-----------|--------|
| Fr, 21.03.2025 – 18:00 | DNS TTL auf 60 Sekunden senken |
| Sa, 22.03.2025 – 22:00 | Migrationsfenster beginnt |
| Sa, 22.03.2025 – 22:00 | Letzter DB-Export vom IST-System |
| Sa, 22.03.2025 – 22:30 | Migration auf Zielsystem starten |
| Sa, 22.03.2025 – 23:30 | Abschlusstests |
| So, 23.03.2025 – 00:00 | DNS umstellen auf neue IP |
| So, 23.03.2025 – 00:30 | Monitoring prüfen |
| So, 23.03.2025 – 02:00 | Migrationsfenster endet |

**Maximale geplante Ausfallzeit: 2 Stunden**

---

## Rollback-Plan

Falls während der Migration ein kritisches Problem auftritt, wird folgender Rollback-Plan aktiviert:

| Schritt | Aktion | Verantwortlich | Zeit |
|---------|--------|---------------|------|
| 1 | Entscheid: Rollback notwendig | Andrija | 0 min |
| 2 | DNS zurück auf alte IP setzen | Andrija | 5 min |
| 3 | IST-System wieder starten | Andrija | 10 min |
| 4 | Benutzer informieren | Andrija | 15 min |
| 5 | Ursache analysieren | Andrija | folgetags |

Da die DNS TTL bereits auf 60 Sekunden gesenkt wurde, ist der Rollback innerhalb von 15 Minuten abgeschlossen.

---

## Risikoanalyse

| Risiko | Wahrscheinlichkeit | Auswirkung | Massnahme |
|--------|-------------------|------------|-----------|
| Datenverlust bei Migration | Gering | Hoch | Snapshot + Export vor Migration |
| Inkompatibilität PHP 8.2 | Mittel | Mittel | Vorabtest in Testumgebung |
| DNS-Propagation verzögert | Gering | Mittel | TTL vorher senken |
| Migrationszeit überschritten | Mittel | Mittel | Rollback-Plan bereit |
| MariaDB Import fehlerhaft | Gering | Hoch | Import zuerst in Testumgebung testen |

---

## Projektplan wurde überprüft und angepasst

Der Projektplan wurde nach jedem Milestone gemeinsam mit dem Auftraggeber überprüft. Nach der Testphase wurden zwei zusätzliche Testfälle ergänzt, die ursprünglich nicht im Testkatalog enthalten waren. Der Deploymentzeitpunkt wurde ebenfalls um eine Woche nach hinten verschoben, da die PHP-Kompatibilitätsprüfung mehr Zeit in Anspruch nahm als geplant.

---

## Fazit

Der Projektplan hat die gesamte Migration strukturiert und nachvollziehbar gemacht. Durch die klare Aufgabenteilung, den definierten kritischen Pfad und den Rollback-Plan konnte die Migration sicher und mit minimalem Ausfall durchgeführt werden. Die laufende Kontrolle und Anpassung des Plans war entscheidend für den Projekterfolg.
