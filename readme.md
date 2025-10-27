# 🧩 Shopware 6 Plugin Konzept: „Lastschriftverwaltung“

## 1️⃣ Zielsetzung

Das Plugin dient zur Verwaltung und Übertragung von Bestellungen mit der Zahlart **Lastschriftverfahren**.  
Mitarbeiter sollen Lastschriften prüfen, exportieren oder direkt an Banken übertragen können.  
Doppelte Ausführungen sollen verhindert und der Zahlungsstatus automatisiert gepflegt werden.

---

## 2️⃣ Funktionsumfang

| Bereich | Funktion |
|----------|-----------|
| **Filterung & Anzeige** | Auflistung aller Bestellungen mit Zahlart „Lastschriftverfahren“ |
| **Details** | Anzeige von IBAN, BIC, Kontoinhaber, Bestellnummer, Bestelldatum, Rechnungsnummer, Rechnungsdatum, Rechnungsbetrag |
| **Statusverwaltung** | Trennung zwischen *offenen* und *abgeschlossenen* Lastschriften |
| **Aktionen** | Auswahl (Checkbox) und Sammelübertragung an Bank oder Export als SEPA-XML |
| **Doppelschutz** | Bereits übertragene Lastschriften sind deaktiviert / ausgegraut |
| **Markierung als bezahlt** | Optional nach Übertragung oder über bestehenden Bankenabgleich |
| **Such- & Filterfunktionen** | Freitextsuche + Filter nach Datum, Rechnungsstatus etc. |
| **Pagination & Tabs** | Zwei Tabs (offen / abgeschlossen) mit Seitennavigation |
| **Schnittstellenwahl** | Option: EBICS / FinAPI / Tink / XML-Export |
| **Logging** | Alle Aktionen werden protokolliert (Wer, wann, welche Transaktion) |

---

## 3️⃣ Technisches Konzept

### 3.1 Datenbasis

**Bestehende Tabellen:**
- `order`
- `order_transaction`
- `order_customer`
- `customer`
- `custom_fields` (für IBAN, BIC, Kontoinhaber)

**Neue Tabelle:** `swag_direct_debit_log`

| Feld | Typ | Beschreibung |
|-------|------|---------------|
| `id` | UUID | Primärschlüssel |
| `order_id` | UUID | Verknüpfung zur Bestellung |
| `iban` | VARCHAR(34) | IBAN des Kunden |
| `bic` | VARCHAR(11) | BIC des Kunden |
| `account_holder` | VARCHAR(255) | Kontoinhaber |
| `amount` | DECIMAL(10,2) | Betrag |
| `status` | ENUM(`open`, `transferred`, `paid`) | Status |
| `transfer_method` | ENUM(`ebics`, `finapi`, `tink`, `xml`) | Übertragungsart |
| `transferred_at` | DATETIME | Übertragungszeitpunkt |
| `created_by` | VARCHAR(255) | Benutzer |
| `log_entry` | TEXT | Rückmeldungen / Fehler |

---

### 3.2 Backend-UI (Administration)

- Pfad: **Kunden > Lastschriften**
- Tabs: *Offene Lastschriften*, *Abgeschlossene Lastschriften*
- Suchmaske: Filter nach Datum, Kunde, Betrag, Status
- Grid mit:
  - Checkbox | Bestellnr | Datum | Kunde | IBAN | BIC | Betrag | Status | Übertragungsart | Übertragen am
- Aktionen:
  - „Ausgewählte an Bank übertragen“
  - „SEPA-XML erstellen“
  - „Als bezahlt markieren“

---

### 3.3 Business Logic

#### (1) Ermittlung der Lastschrift-Bestellungen
- Filterung: `payment_method.name = 'Lastschrift'`
- Join mit `order`, `customer`, `order_transaction`
- Nach Status sortieren

#### (2) Übertragungsoptionen
- Plugin-Einstellungen:
  - `default_transfer_method` (EBICS / FinAPI / Tink / XML)
  - Zugangsdaten für Schnittstellen
  - `auto_mark_paid` (bool)

#### (3) Übertragungsvorgang
1. Validierung (nicht doppelt)
2. Erstellung SEPA-XML
3. API-Aufruf / Datei-Export
4. Logging
5. Rechnungsstatus optional aktualisieren

#### (4) Manueller Export
- SEPA pain.008.001.02-Datei generieren
- Speicherung in `/var/export/`

---

### 3.4 Schnittstellen & Libraries

| Anbieter | Typ | Library / API | Beschreibung |
|-----------|------|---------------|--------------|
| EBICS | Direktbank | `digitick/ebics` | Sichere Bankanbindung |
| FinAPI | REST-API | [finapi.io](https://finapi.io) | Moderne Schnittstelle |
| Tink | REST-API | [tink.com](https://tink.com) | Sandbox verfügbar |
| XML | Lokal | SEPA pain.008 | Kein externes System nötig |

---

### 3.5 Rechte & Sicherheit

- Nur Benutzer mit `lastschrift.manage`
- IBAN/BIC verschlüsselt speichern
- API-Credentials sicher in `system_config`
- Logging aller Aktionen

---

### 3.6 Erweiterbarkeit

- Bankabgleich (automatisch)
- CSV-Export
- Cronjob für Abgleich
- Notifikationen

---

## 4️⃣ Ablaufdiagramm

```plaintext
[Admin öffnet Lastschriftmodul]
        |
        v
[Abfrage: offene Lastschriften]
        |
        v
[Anzeige Datagrid]
        |
        +--> [Auswahl Checkboxen]
               |
               v
         [Übertragen an Bank]
               |
        +--> [API-Aufruf / XML-Erzeugung]
               |
        +--> [Erfolg -> Status 'übertragen']
               |
        +--> [Optional: Rechnung als bezahlt markieren]
```

---

## 5️⃣ Entwicklungsaufwand (Schätzung)

| Phase | Beschreibung | Aufwand |
|--------|---------------|----------|
| Architektur & Datenmodell | Tabellen, Entitäten, Settings | 1–2 Tage |
| Backend-UI | Vue.js-Admin-Modul | 2–3 Tage |
| Business-Logik | XML-Export, Validierung | 2 Tage |
| API-Anbindung | EBICS / FinAPI / Tink | 2–4 Tage |
| Tests & Sicherheit | Unit Tests, Logging | 1–2 Tage |
| **Gesamt** | **8–12 Arbeitstage** |

---

## 6️⃣ Nächste Schritte

### 6.1 Übertragungsart auswählen
- Start: **XML-Export (pain.008.001.02)**
- Später: EBICS, FinAPI, Tink

---

### 6.2 Grundstruktur des Plugins

```plaintext
src/SwagDirectDebit/
├── SwagDirectDebit.php
├── Resources/
│   ├── config/system.xml
│   ├── migration/MigrationXXXXCreateDirectDebitLog.php
│   ├── snippets/de-DE.json
│   ├── snippets/en-GB.json
│   └── app/administration/module/lastschrift/
│       ├── index.js
│       ├── page/lastschrift-list/
│       │   └── index.js
│       └── component/lastschrift-item/
│           └── index.js
├── Entity/DirectDebitLog/
│   ├── DirectDebitLogDefinition.php
│   ├── DirectDebitLogEntity.php
│   └── DirectDebitLogCollection.php
├── Service/
│   ├── DirectDebitService.php
│   ├── SepaXmlGenerator.php
│   └── ApiTransferService.php
└── Controller/Admin/DirectDebitController.php
```

---

### 6.3 To-Do-Reihenfolge für die Entwicklung

1. Plugin-Skeleton erstellen  
2. Migration & Entität  
3. Administration-Modul  
4. Business-Logik  
5. System-Einstellungen  
6. Logging & Sicherheit  
7. Tests & Qualitätssicherung  
8. Optionale Erweiterungen  

---

### 6.4 System-Einstellungen

**Datei:** `Resources/config/system.xml`

**Parameter:**
- `transferMethod` (`xml`, `ebics`, `finapi`, `tink`)
- `autoMarkPaid` (bool)
- API-Credentials (falls benötigt)

**Verwendung:**
- Zugriff in Services per `SystemConfigService`  
  ```php
  $method = $this->systemConfigService->get('SwagDirectDebit.config.transferMethod');
  ```

---

### 6.5 Logging & Sicherheit

- IBAN/BIC mit **Shopware Encryption Service** verschlüsseln  
- Aktionen in **`swag_direct_debit_log`** protokollieren  
- Berechtigung via **ACL-Rolle `lastschrift.manage`**
- Validierung & Exception-Handling für Fehlermeldungen im Admin-UI

---

### 6.6 Tests & Qualitätssicherung

- **Unit-Tests für:**
  - `DirectDebitService`
  - `SepaXmlGenerator`
  - `ApiTransferService` (mit Mocks)
- **Validierung:** SEPA-XML gegen XSD-Schema prüfen  
- **Admin-UI-Tests:** Grid-Filter, Checkboxen, Aktionen testen

---

### 6.7 Optionale Erweiterungen

- **API-Integration:** FinAPI, Tink, EBICS (Live oder Sandbox)  
- **Automatischer Bankabgleich:** Matching eingehender Zahlungen  
- **Cronjob:** Regelmäßige Prüfung offener Lastschriften  
- **Benachrichtigungen:** E-Mail oder Admin-Hinweise bei neuen Lastschriften
