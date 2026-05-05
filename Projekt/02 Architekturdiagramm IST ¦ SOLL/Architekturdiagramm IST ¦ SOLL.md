# Auftrag #2 – Architekturdiagramm IST/SOLL

## Kompetenz
A: Release-Situation analysieren

---

## Beschreibung

Es wurde eine vollständige IST-Analyse des bestehenden CRM-Systems durchgeführt. Anschliessend wurde ein SOLL-Diagramm mit der neuen Zielarchitektur erstellt und Migrationsvarianten bewertet.

---

## IST-Analyse

### Systemübersicht

| Komponente | Version | IP | Betriebssystem |
|------------|---------|-----|----------------|
| Webserver (Apache) | 2.4.29 | 192.168.1.10 | Ubuntu 18.04 |
| PHP | 7.2 | – | – |
| MySQL | 5.7 | 192.168.1.10 | – |
| Vtiger CRM | 7.1 | – | – |

### Mengengerüst Datenbank

```sql
-- Anzahl Tabellen
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'vtiger';

-- Datenbankgrösse
SELECT table_schema AS 'DB', 
ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)' 
FROM information_schema.tables 
WHERE table_schema = 'vtiger';
```

| Metrik | Wert |
|--------|------|
| Anzahl Tabellen | 247 |
| Datenbankgrösse | 1.8 GB |
| Anzahl Datensätze (Kontakte) | 12'450 |
| Anzahl Datensätze (Leads) | 8'320 |

---

## SOLL-Architektur

### Systemübersicht

| Komponente | Version | IP | Betriebssystem |
|------------|---------|-----|----------------|
| Webserver (Apache) | 2.4.58 | 192.168.1.20 | Ubuntu 24.04 LTS |
| PHP | 8.2 | – | – |
| MariaDB | 10.11 | 192.168.1.20 | – |
| Vtiger CRM | 7.5 | – | – |

---

## Migrationsvarianten

### Variante A – Upgrade auf neuste Vtiger Version
- Vtiger 7.1 → 7.5
- Bewährte Plattform, bekannte Oberfläche
- Datenmigration direkt möglich
- **Empfehlung:** Ja, geringeres Risiko

### Variante B – Wechsel auf OpenERP/Odoo
- Komplett neue Plattform
- Hoher Migrationsaufwand
- Schulungsaufwand für Benutzer
- **Empfehlung:** Nein, zu hohes Risiko im gegebenen Zeitrahmen

---

## Bewertung & Entscheid

Variante A wurde gewählt. Die Datenstruktur von Vtiger bleibt kompatibel, der Migrationsaufwand ist überschaubar und das Risiko eines Datenverlusts ist minimal.

---

## Testfälle als Nachweis

| Testfall | Erwartetes Ergebnis |
|----------|-------------------|
| Login CRM | Erfolgreicher Login |
| Kontakte anzeigen | Alle 12'450 Kontakte vorhanden |
| Leads anzeigen | Alle 8'320 Leads vorhanden |
| E-Mail senden | E-Mail wird korrekt versendet |
| Bericht erstellen | Bericht wird korrekt generiert |
