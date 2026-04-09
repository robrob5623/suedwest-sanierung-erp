# ERP Framework Patch v2 – Erweitertes Ausgangsrechnungs-Modul

**Patch-ID:** PATCH-2025-002  
**Datum:** 26. Januar 2025  
**Betrifft:** M11 – Ausgangsrechnungen & Weiterberechnung  
**Priorität:** HOCH  
**Status:** Entwurf

---

## 1. Zusammenfassung der Änderungen

Dieses Patch erweitert das Ausgangsrechnungs-Modul um folgende Kernfunktionen:

1. **Rechnungserstellung aus Angebot ODER manuell** – Flexible Erstellungsoptionen
2. **Titel/Positions-Logik identisch zu Angeboten** – Hierarchische Struktur mit Titeln und Unterpositionen
3. **Vollständige Editierbarkeit bis Festschreibung** – Alle Felder änderbar im Status ENTWURF
4. **Korrekturrechnung-Funktionalität** – Storno und Korrektur bereits festgeschriebener Rechnungen
5. **Import aus Projekt und Angebot** – Übernahme von Titeln/Positionen aus verschiedenen Quellen

---

## 2. Datenmodell-Erweiterungen

### 2.1 AUSGANGSRECHNUNG (Erweitert)

**Neue/geänderte Attribute:**

| Attribut | Typ | Pflicht | Beschreibung |
|----------|-----|---------|--------------|
| `angebot_id` | UUID | | FK → ANGEBOT (Quelle, falls aus Angebot erstellt) |
| `auftrag_id` | UUID | | FK → AUFTRAG (bleibt bestehen) |
| `erstellungsart` | ENUM | ✓ | `AUS_ANGEBOT`, `AUS_AUFTRAG`, `AUS_PROJEKT`, `MANUELL` |
| `ist_festgeschrieben` | BOOLEAN | ✓ | Default: FALSE – Nach Festschreibung keine Änderung mehr |
| `festgeschrieben_am` | TIMESTAMP | | Zeitpunkt der Festschreibung |
| `festgeschrieben_von` | UUID | | FK → MITARBEITER |
| `korrektur_zu_ar_id` | UUID | | FK → AUSGANGSRECHNUNG (bei Korrekturrechnung: Referenz auf Original) |
| `korrigiert_durch_ar_id` | UUID | | FK → AUSGANGSRECHNUNG (bei stornierter Rechnung: Verweis auf Korrektur) |
| `korrektur_grund` | TEXT | | Pflicht bei Korrektur-/Stornorechnung |
| `ursprungs_rechnungs_nr` | VARCHAR(30) | | Original-Rechnungsnummer bei Korrektur |
| `einleitung` | TEXT | | Einleitungstext (wie bei Angebot) |
| `schlusstext` | TEXT | | Schlusstext (wie bei Angebot) |
| `rabatt_prozent` | DECIMAL(5,2) | | Gesamtrabatt |
| `rabatt_betrag` | DECIMAL(10,2) | | (berechnet oder manuell) |
| `skonto_prozent` | DECIMAL(5,2) | | Skonto bei Zahlung innerhalb Frist |
| `skonto_tage` | INT | | Tage für Skontofrist |
| `skonto_betrag` | DECIMAL(12,2) | | (berechnet) |

**Erweiterte Status-ENUM:**

```sql
CREATE TYPE ar_status AS ENUM (
  'ENTWURF',           -- Vollständig editierbar
  'ERSTELLT',          -- Erstellt, noch nicht geprüft
  'IN_PRUEFUNG',       -- 4-Augen-Prinzip aktiv
  'KORREKTUR',         -- Zurück zur Bearbeitung
  'FREIGEGEBEN',       -- Freigabe erteilt
  'FESTGESCHRIEBEN',   -- NEU: Unveränderbar, aber noch nicht versendet
  'VERSENDET',         -- An Kunde gesendet
  'TEILZAHLUNG',       -- Teilzahlung eingegangen
  'BEZAHLT',           -- Vollständig bezahlt
  'UEBERFAELLIG',      -- Zahlungsfrist überschritten
  'GEMAHNT',           -- Mahnung versendet
  'STORNIERT',         -- Storniert (durch Korrekturrechnung)
  'GUTGESCHRIEBEN'     -- NEU: Vollständige Gutschrift erstellt
);
```

**Erweiterte Rechnungsart-ENUM:**

```sql
CREATE TYPE ar_rechnungsart AS ENUM (
  'ABSCHLAG',
  'TEILRECHNUNG',
  'SCHLUSSRECHNUNG',
  'KORREKTUR',           -- NEU: Korrekturrechnung
  'STORNO',              -- Vollstorno
  'GUTSCHRIFT',
  'PROFORMA'             -- NEU: Proforma-Rechnung (informativ)
);
```

---

### 2.2 AR_TITEL (NEU – Analog zu Angebot)

| Attribut | Typ | Pflicht | Beschreibung |
|----------|-----|---------|--------------|
| `ar_titel_id` | UUID | ✓ | |
| `ar_id` | UUID | ✓ | FK → AUSGANGSRECHNUNG |
| `titel_nr` | VARCHAR(10) | ✓ | z.B. "1", "2", "3" |
| `bezeichnung` | VARCHAR(500) | ✓ | Titeltext |
| `sortierung` | INT | ✓ | Reihenfolge |
| `summe_netto` | DECIMAL(12,2) | | (berechnet aus Positionen) |
| `ist_optional` | BOOLEAN | ✓ | Default: FALSE |
| `quelle_typ` | ENUM | | `ANGEBOT_TITEL`, `PROJEKT`, `KATALOG`, `MANUELL` |
| `quelle_id` | UUID | | Referenz auf Quell-Titel |

---

### 2.3 AR_POSITION (Erweitert)

**Neue/geänderte Attribute:**

| Attribut | Typ | Pflicht | Beschreibung |
|----------|-----|---------|--------------|
| `ar_titel_id` | UUID | | FK → AR_TITEL (optional, für Titel-Zuordnung) |
| `parent_pos_id` | UUID | | FK → AR_POSITION (für Unterpositionen) |
| `hierarchie_ebene` | INT | ✓ | Default: 0 (0=Hauptposition, 1=Unterposition, etc.) |
| `ist_titel` | BOOLEAN | ✓ | Default: FALSE – TRUE wenn Sammelposition |
| `ist_optional` | BOOLEAN | ✓ | Default: FALSE |
| `ist_alternativ` | BOOLEAN | ✓ | Default: FALSE |
| `gewerk_id` | UUID | | FK → GEWERK_KATALOG |
| `katalog_position_id` | UUID | | FK → POSITION_KATALOG |
| `quelle_typ` | ENUM | | `ANGEBOT_POS`, `AUFTRAG_POS`, `PROJEKT`, `WEITERBERECHNUNG`, `KATALOG`, `MANUELL` |
| `quelle_id` | UUID | | ID des Quellobjekts |
| `menge_original` | DECIMAL(12,3) | | Ursprüngliche Menge (aus Quelle) |
| `menge_abgerechnet` | DECIMAL(12,3) | | Bereits in früheren Rechnungen abgerechnet |
| `menge_aktuell` | DECIMAL(12,3) | ✓ | In dieser Rechnung abzurechnen |
| `einzelpreis_original` | DECIMAL(12,4) | | Ursprünglicher Preis |
| `abweichung_begruendung` | TEXT | | Pflicht wenn Preis/Menge abweicht |

**Positionsart-ENUM (erweitert):**

```sql
CREATE TYPE ar_positionsart AS ENUM (
  'TITEL',           -- NEU: Sammeltitel ohne Preis
  'LEISTUNG',
  'MATERIAL',
  'GERAETEMIETE',
  'CONTAINER',
  'PAUSCHALE',       -- NEU: Pauschalposition
  'STUNDEN',         -- NEU: Stundenabrechnung
  'SU_LEISTUNG',     -- NEU: Subunternehmer-Leistung
  'NACHLASS',        -- NEU: Rabatt/Nachlass-Position
  'SONSTIG'
);
```

---

### 2.4 AR_KORREKTUR_HISTORIE (NEU)

Tracking aller Korrekturen einer Rechnung.

| Attribut | Typ | Pflicht | Beschreibung |
|----------|-----|---------|--------------|
| `historie_id` | UUID | ✓ | |
| `original_ar_id` | UUID | ✓ | FK → AUSGANGSRECHNUNG (Original) |
| `korrektur_ar_id` | UUID | ✓ | FK → AUSGANGSRECHNUNG (Korrektur) |
| `korrektur_typ` | ENUM | ✓ | `STORNO`, `TEILKORREKTUR`, `PREISKORREKTUR`, `MENGENKORREKTUR` |
| `differenz_netto` | DECIMAL(12,2) | ✓ | Betragsdifferenz |
| `begruendung` | TEXT | ✓ | |
| `erstellt_von` | UUID | ✓ | |
| `erstellt_am` | TIMESTAMP | ✓ | |
| `freigegeben_von` | UUID | | |
| `freigegeben_am` | TIMESTAMP | | |

---

## 3. Workflow-Erweiterungen

### 3.1 WF-M11-01a: Rechnung aus Angebot erstellen (NEU)

```
WF-M11-01a: Rechnung aus Angebot erstellen
├─ Trigger: Benutzeraktion "Rechnung erstellen" im Angebot-Kontext
├─ Vorbedingungen:
│   - Angebot.status = 'ANGENOMMEN'
│   - Auftrag existiert und ist aktiv
├─ Schritte:
│   1. Angebot auswählen
│   2. Abzurechnende Positionen wählen:
│      a. Alle übernehmen
│      b. Nur bestimmte Titel
│      c. Einzelne Positionen selektieren
│   3. Für jede Position prüfen:
│      - Bereits abgerechnete Menge
│      - Noch offene Menge
│   4. Mengen anpassen (wenn Teilrechnung)
│   5. Preise prüfen/anpassen (mit Begründungspflicht bei Abweichung)
│   6. Titel-Struktur übernehmen oder anpassen
│   7. Zusätzliche Positionen manuell ergänzen
│   8. Weiterberechnungs-Positionen aus Projekt hinzufügen
│   9. Summenprüfung
│  10. Speichern als ENTWURF
├─ Ausgabe: AR im Status ENTWURF
└─ Audit-Log: 'AR erstellt aus Angebot [Nr]'
```

### 3.2 WF-M11-01b: Rechnung manuell erstellen (NEU)

```
WF-M11-01b: Rechnung manuell erstellen
├─ Trigger: Benutzeraktion "Neue Rechnung" ohne Angebotsbezug
├─ Schritte:
│   1. Projekt auswählen (Pflicht)
│   2. Kunde wird aus Projekt übernommen (änderbar)
│   3. Rechnungsart wählen
│   4. Struktur wählen:
│      a. "Aus Projekt-Leistungen" → Weiterberechnungs-Pool
│      b. "Aus Katalog" → Positionskatalog
│      c. "Freie Eingabe" → Manuelle Positionen
│   5. Titel erstellen oder aus Vorlagen
│   6. Positionen hinzufügen
│   7. Summenprüfung
│   8. Speichern als ENTWURF
├─ Ausgabe: AR im Status ENTWURF
└─ Audit-Log: 'AR manuell erstellt für Projekt [Nr]'
```

### 3.3 WF-M11-02: Rechnung bearbeiten (Editierbarkeit bis Festschreibung)

```
WF-M11-02: Rechnung bearbeiten
├─ Vorbedingung: AR.ist_festgeschrieben = FALSE
├─ Erlaubte Status für Bearbeitung:
│   - ENTWURF (alle Felder)
│   - ERSTELLT (alle Felder)
│   - IN_PRUEFUNG (nur mit Berechtigung)
│   - KORREKTUR (alle Felder)
├─ Editierbare Elemente:
│   - Titel hinzufügen/entfernen/umordnen
│   - Positionen hinzufügen/entfernen/ändern
│   - Mengen und Preise anpassen
│   - Texte (Einleitung, Schluss, Positionstexte)
│   - Rabatte, Skonto
│   - Leistungszeitraum
│   - Fälligkeitsdatum
├─ Nicht editierbar (nach Erstellung):
│   - Rechnungsnummer
│   - Verknüpftes Projekt
│   - Verknüpfter Kunde (nur mit Eskalation)
├─ Output: Aktualisierte Rechnung
└─ Audit-Log: Jede Änderung wird protokolliert
```

### 3.4 WF-M11-03: Rechnung festschreiben (NEU)

```
WF-M11-03: Rechnung festschreiben
├─ Trigger: Benutzeraktion "Festschreiben" nach Freigabe
├─ Vorbedingungen:
│   - AR.status IN ('FREIGEGEBEN')
│   - Alle Pflichtfelder gefüllt
│   - Mindestens eine Position vorhanden
│   - Summen korrekt berechnet
├─ Schritte:
│   1. Validierung aller Daten
│   2. PDF generieren und ablegen
│   3. Setzen:
│      - ist_festgeschrieben = TRUE
│      - festgeschrieben_am = NOW()
│      - festgeschrieben_von = current_user
│      - status = 'FESTGESCHRIEBEN'
│   4. Positionen als "abgerechnet" markieren in Quell-Objekten
│   5. Audit-Log erstellen
├─ WICHTIG: Nach Festschreibung KEINE Änderung mehr möglich!
├─ Output: Unveränderbare Rechnung
└─ Audit-Log: 'AR [Nr] festgeschrieben'
```

### 3.5 WF-M11-04: Korrekturrechnung erstellen (NEU)

```
WF-M11-04: Korrekturrechnung erstellen
├─ Trigger: Benutzeraktion "Korrektur erstellen" bei festgeschriebener AR
├─ Vorbedingungen:
│   - Original-AR.ist_festgeschrieben = TRUE
│   - Original-AR.status NOT IN ('STORNIERT', 'GUTGESCHRIEBEN')
├─ Korrekturtypen:
│   a. VOLLSTORNO:
│      - Erstellt Storno-Rechnung mit negativen Beträgen
│      - Original-AR.status → 'STORNIERT'
│      - Alle Mengen werden wieder "offen"
│   
│   b. TEILKORREKTUR:
│      - Korrektur einzelner Positionen
│      - Nur Differenzbeträge in Korrektur-AR
│      
│   c. GUTSCHRIFT:
│      - Teilgutschrift ohne vollständige Stornierung
│
├─ Schritte:
│   1. Korrekturtyp wählen
│   2. Pflichtfeld: Korrektur-Begründung (min. 50 Zeichen)
│   3. Bei Teilkorrektur: Betroffene Positionen wählen
│   4. Neue Beträge/Mengen eingeben
│   5. Differenz wird automatisch berechnet
│   6. Neue Rechnungsnummer generieren (Format: R-JJ-NNNNN-K1)
│   7. Verknüpfung herstellen:
│      - Korrektur-AR.korrektur_zu_ar_id = Original-AR.ar_id
│      - Original-AR.korrigiert_durch_ar_id = Korrektur-AR.ar_id
│   8. Freigabe-Workflow durchlaufen (wie normale AR)
│   9. Bei Freigabe: Original-Status aktualisieren
│
├─ Prüfungen:
│   - Korrektur-Betrag darf nicht größer als Original sein
│   - Bei Vollstorno: Summe muss exakt negativ sein
│
├─ Output: Korrekturrechnung im Status ENTWURF
└─ Audit-Log: 'Korrektur zu AR [Original-Nr] erstellt'
```

### 3.6 WF-M11-05: Positionen aus Projekt importieren (NEU)

```
WF-M11-05: Positionen aus Projekt importieren
├─ Trigger: Benutzeraktion im Rechnungseditor
├─ Importquellen:
│   1. Weiterberechnungs-Pool:
│      - Zeitbuchungen (ZEITBUCHUNG)
│      - Materialverbräuche (MATERIAL_VERBRAUCH)
│      - Geräte-Mietvorgänge (MIETVORGANG)
│      - Container (CONTAINER_NUTZUNG)
│      - SU-Leistungen (SU_VERGABE mit weiterberechnung_status = 'OFFEN')
│   
│   2. Aus Angebot/Auftrag:
│      - Noch nicht abgerechnete Positionen
│      - Mit Original-Mengen und Preisen
│   
│   3. Aus Positionskatalog:
│      - Standard-LV-Positionen
│      - Mit Richtpreisen
│
├─ Schritte:
│   1. Quelle wählen
│   2. Verfügbare Positionen anzeigen (bereits abgerechnete ausgegraut)
│   3. Multi-Select der gewünschten Positionen
│   4. Preis-Transformation anzeigen:
│      - EK-Preis (aus Quelle)
│      - VK-Preis (mit Aufschlag)
│      - Aufschlag-Satz (konfigurierbar)
│   5. Bestätigen → Positionen werden als AR_POSITION angelegt
│   6. quelle_typ und quelle_id werden gesetzt
│
├─ Output: Neue Positionen in der Rechnung
└─ Audit-Log: 'X Positionen aus [Quelle] importiert'
```

---

## 4. Geschäftsregeln

### 4.1 Editierbarkeits-Regeln

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ REGEL: EDITIERBARKEIT BIS FESTSCHREIBUNG                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. ENTWURF:          Alle Felder editierbar durch Ersteller + BL + PL      │
│ 2. ERSTELLT:         Alle Felder editierbar durch Ersteller + PL           │
│ 3. IN_PRUEFUNG:      Nur Prüfer kann "Zurück zur Bearbeitung" setzen       │
│ 4. KORREKTUR:        Alle Felder editierbar durch Ersteller + PL           │
│ 5. FREIGEGEBEN:      Nur Festschreibung oder Zurück möglich                │
│ 6. FESTGESCHRIEBEN:  KEINE Änderung mehr – nur Korrekturrechnung           │
│ 7. VERSENDET+:       KEINE Änderung – nur Zahlungseingang/Mahnung          │
│                                                                             │
│ AUSNAHME: GF kann bei FREIGEGEBEN noch Änderungen anfordern               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Korrektur-Regeln

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ REGEL: KORREKTUR FESTGESCHRIEBENER RECHNUNGEN                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. Nur festgeschriebene Rechnungen können korrigiert werden                │
│                                                                             │
│ 2. Korrektur-Berechtigung: FIB, GF                                         │
│                                                                             │
│ 3. Pflichtangaben bei Korrektur:                                           │
│    - Begründung (min. 50 Zeichen)                                          │
│    - Korrekturtyp                                                          │
│                                                                             │
│ 4. Korrektur durchläuft denselben Freigabe-Workflow                        │
│                                                                             │
│ 5. Bei Storno:                                                             │
│    - Wenn bereits bezahlt → Erstattung buchen                              │
│    - Offene Posten aktualisieren                                           │
│                                                                             │
│ 6. Eine Rechnung kann mehrfach korrigiert werden (K1, K2, K3...)           │
│                                                                             │
│ 7. Korrektur-Kette muss nachvollziehbar bleiben                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Positions-Import-Regeln

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ REGEL: IMPORT VON POSITIONEN                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. Aus Angebot:                                                            │
│    - Nur von angenommenen Angeboten                                        │
│    - Titel-Struktur wird übernommen                                        │
│    - Preise können überschrieben werden (mit Begründung)                   │
│                                                                             │
│ 2. Aus Projekt (Weiterberechnung):                                         │
│    - EK→VK Transformation mit konfigurierbarem Aufschlag                   │
│    - Zeiteinträge: Stundensatz_VK aus MITARBEITER oder Standard            │
│    - Material: Aufschlag % aus ARTIKELGRUPPE oder Standard                 │
│    - Geräte: Tagessatz_VK aus GERAET                                       │
│    - Container: Aufschlag % aus Einstellung                                │
│    - SU: VK-Betrag aus SU_VERGABE.weiterberechnung_betrag                  │
│                                                                             │
│ 3. Bereits abgerechnete Positionen:                                        │
│    - Werden mit verbleibender Menge angezeigt                              │
│    - Können nicht doppelt importiert werden                                │
│                                                                             │
│ 4. Manuell erstellte Positionen:                                           │
│    - Keine Quelle-Referenz                                                 │
│    - Volle Freiheit bei Preisgestaltung                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. SQL-Erweiterungen

### 5.1 Tabellen-Erweiterung AUSGANGSRECHNUNG

```sql
-- Erweiterung der bestehenden Tabelle
ALTER TABLE ausgangsrechnung 
ADD COLUMN IF NOT EXISTS angebot_id UUID REFERENCES angebot(angebot_id),
ADD COLUMN IF NOT EXISTS erstellungsart VARCHAR(20) DEFAULT 'MANUELL'
  CHECK (erstellungsart IN ('AUS_ANGEBOT', 'AUS_AUFTRAG', 'AUS_PROJEKT', 'MANUELL')),
ADD COLUMN IF NOT EXISTS ist_festgeschrieben BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS festgeschrieben_am TIMESTAMP,
ADD COLUMN IF NOT EXISTS festgeschrieben_von UUID REFERENCES mitarbeiter(mitarbeiter_id),
ADD COLUMN IF NOT EXISTS korrektur_zu_ar_id UUID REFERENCES ausgangsrechnung(ar_id),
ADD COLUMN IF NOT EXISTS korrigiert_durch_ar_id UUID REFERENCES ausgangsrechnung(ar_id),
ADD COLUMN IF NOT EXISTS korrektur_grund TEXT,
ADD COLUMN IF NOT EXISTS ursprungs_rechnungs_nr VARCHAR(30),
ADD COLUMN IF NOT EXISTS einleitung TEXT,
ADD COLUMN IF NOT EXISTS schlusstext TEXT,
ADD COLUMN IF NOT EXISTS rabatt_prozent DECIMAL(5,2),
ADD COLUMN IF NOT EXISTS rabatt_betrag DECIMAL(10,2),
ADD COLUMN IF NOT EXISTS skonto_prozent DECIMAL(5,2),
ADD COLUMN IF NOT EXISTS skonto_tage INT,
ADD COLUMN IF NOT EXISTS skonto_betrag DECIMAL(12,2);

-- Neue Status-Werte
ALTER TYPE ar_status ADD VALUE IF NOT EXISTS 'FESTGESCHRIEBEN' AFTER 'FREIGEGEBEN';
ALTER TYPE ar_status ADD VALUE IF NOT EXISTS 'GUTGESCHRIEBEN' AFTER 'STORNIERT';

-- Neue Rechnungsart-Werte
ALTER TYPE ar_rechnungsart ADD VALUE IF NOT EXISTS 'KORREKTUR' AFTER 'STORNO';
ALTER TYPE ar_rechnungsart ADD VALUE IF NOT EXISTS 'PROFORMA' AFTER 'GUTSCHRIFT';
```

### 5.2 Neue Tabelle AR_TITEL

```sql
CREATE TABLE ar_titel (
  ar_titel_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ar_id UUID NOT NULL REFERENCES ausgangsrechnung(ar_id) ON DELETE CASCADE,
  titel_nr VARCHAR(10) NOT NULL,
  bezeichnung VARCHAR(500) NOT NULL,
  sortierung INT NOT NULL,
  summe_netto DECIMAL(12,2),
  ist_optional BOOLEAN NOT NULL DEFAULT FALSE,
  quelle_typ VARCHAR(20) CHECK (quelle_typ IN ('ANGEBOT_TITEL', 'PROJEKT', 'KATALOG', 'MANUELL')),
  quelle_id UUID,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE (ar_id, titel_nr)
);

CREATE INDEX idx_ar_titel_ar ON ar_titel(ar_id);
```

### 5.3 Erweiterung AR_POSITION

```sql
-- Erweiterung der bestehenden Tabelle
ALTER TABLE ar_position
ADD COLUMN IF NOT EXISTS ar_titel_id UUID REFERENCES ar_titel(ar_titel_id),
ADD COLUMN IF NOT EXISTS parent_pos_id UUID REFERENCES ar_position(ar_pos_id),
ADD COLUMN IF NOT EXISTS hierarchie_ebene INT DEFAULT 0,
ADD COLUMN IF NOT EXISTS ist_titel BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS ist_optional BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS ist_alternativ BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS gewerk_id UUID REFERENCES gewerk_katalog(gewerk_id),
ADD COLUMN IF NOT EXISTS katalog_position_id UUID REFERENCES position_katalog(katalog_pos_id),
ADD COLUMN IF NOT EXISTS quelle_typ VARCHAR(30) 
  CHECK (quelle_typ IN ('ANGEBOT_POS', 'AUFTRAG_POS', 'PROJEKT', 'WEITERBERECHNUNG', 'KATALOG', 'MANUELL')),
ADD COLUMN IF NOT EXISTS quelle_id UUID,
ADD COLUMN IF NOT EXISTS menge_original DECIMAL(12,3),
ADD COLUMN IF NOT EXISTS menge_abgerechnet DECIMAL(12,3) DEFAULT 0,
ADD COLUMN IF NOT EXISTS menge_aktuell DECIMAL(12,3),
ADD COLUMN IF NOT EXISTS einzelpreis_original DECIMAL(12,4),
ADD COLUMN IF NOT EXISTS abweichung_begruendung TEXT;

-- Neue Positionsarten
ALTER TYPE ar_positionsart ADD VALUE IF NOT EXISTS 'TITEL';
ALTER TYPE ar_positionsart ADD VALUE IF NOT EXISTS 'PAUSCHALE';
ALTER TYPE ar_positionsart ADD VALUE IF NOT EXISTS 'STUNDEN';
ALTER TYPE ar_positionsart ADD VALUE IF NOT EXISTS 'SU_LEISTUNG';
ALTER TYPE ar_positionsart ADD VALUE IF NOT EXISTS 'NACHLASS';

CREATE INDEX idx_ar_position_titel ON ar_position(ar_titel_id);
CREATE INDEX idx_ar_position_parent ON ar_position(parent_pos_id);
```

### 5.4 Neue Tabelle AR_KORREKTUR_HISTORIE

```sql
CREATE TABLE ar_korrektur_historie (
  historie_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_ar_id UUID NOT NULL REFERENCES ausgangsrechnung(ar_id),
  korrektur_ar_id UUID NOT NULL REFERENCES ausgangsrechnung(ar_id),
  korrektur_typ VARCHAR(20) NOT NULL 
    CHECK (korrektur_typ IN ('STORNO', 'TEILKORREKTUR', 'PREISKORREKTUR', 'MENGENKORREKTUR')),
  differenz_netto DECIMAL(12,2) NOT NULL,
  begruendung TEXT NOT NULL,
  erstellt_von UUID NOT NULL REFERENCES mitarbeiter(mitarbeiter_id),
  erstellt_am TIMESTAMP NOT NULL DEFAULT NOW(),
  freigegeben_von UUID REFERENCES mitarbeiter(mitarbeiter_id),
  freigegeben_am TIMESTAMP
);

CREATE INDEX idx_ar_korrektur_original ON ar_korrektur_historie(original_ar_id);
CREATE INDEX idx_ar_korrektur_korrektur ON ar_korrektur_historie(korrektur_ar_id);
```

### 5.5 View für Rechnungs-Übersicht

```sql
CREATE OR REPLACE VIEW v_ausgangsrechnung_uebersicht AS
SELECT 
  ar.ar_id,
  ar.rechnungs_nr,
  ar.rechnungsdatum,
  ar.rechnungsart,
  ar.erstellungsart,
  ar.status,
  ar.ist_festgeschrieben,
  ar.brutto_summe,
  ar.zahlbetrag,
  ar.offener_betrag,
  ar.faelligkeitsdatum,
  CASE WHEN ar.faelligkeitsdatum < CURRENT_DATE AND ar.status NOT IN ('BEZAHLT', 'STORNIERT', 'GUTGESCHRIEBEN') 
       THEN TRUE ELSE FALSE END AS ist_ueberfaellig,
  p.projekt_nr,
  p.kurzbezeichnung AS projekt_name,
  k.display_name AS kunde_name,
  a.angebot_nr AS basis_angebot,
  korr.rechnungs_nr AS korrektur_zu,
  (SELECT COUNT(*) FROM ar_korrektur_historie h WHERE h.original_ar_id = ar.ar_id) AS anzahl_korrekturen
FROM ausgangsrechnung ar
JOIN schaden_projekt p ON ar.projekt_id = p.projekt_id
JOIN kunde k ON ar.kunde_id = k.kunde_id
LEFT JOIN angebot a ON ar.angebot_id = a.angebot_id
LEFT JOIN ausgangsrechnung korr ON ar.korrektur_zu_ar_id = korr.ar_id;
```

---

## 6. UI-Anforderungen

### 6.1 Rechnungseditor (Neu)

- **Titel-Struktur**: Tree-View links mit Drag & Drop (analog Angebots-Editor)
- **Positions-Editor**: Rechts mit Details zur ausgewählten Position
- **Import-Wizard**: Modal für Positions-Import aus verschiedenen Quellen
- **Summenbereich**: Live-Berechnung von Netto, MwSt, Brutto, Rabatt, Skonto
- **Quell-Anzeige**: Badge zeigt Herkunft jeder Position (Angebot, Projekt, Manuell)
- **Korrektur-Modus**: Spezieller Editor für Korrekturrechnungen mit Differenz-Anzeige

### 6.2 Korrektur-Dialog

- **Original-Rechnung**: Anzeige links
- **Korrektur**: Editierbare Felder rechts
- **Differenz-Berechnung**: Automatisch in der Mitte
- **Begründungspflicht**: Prominent platziert

### 6.3 Status-Workflow-Anzeige

- **Horizontale Timeline**: Zeigt alle Status-Übergänge
- **Festschreibungs-Warnung**: Deutliche Warnung vor Festschreibung
- **Korrektur-Kette**: Visualisierung bei korrigierten Rechnungen

---

## 7. Berechtigungsmatrix (Ergänzung)

| Aktion | BL | PL | SM | FIB | GF |
|--------|:--:|:--:|:--:|:---:|:--:|
| AR erstellen (Entwurf) | ✓ | ✓ | – | ✓ | ✓ |
| AR bearbeiten (Entwurf) | ✓ | ✓ | – | ✓ | ✓ |
| AR zur Prüfung einreichen | ✓ | ✓ | – | ✓ | ✓ |
| AR freigeben (<5k€) | – | ✓ | – | ✓ | ✓ |
| AR freigeben (≥5k€) | – | – | – | ✓ | ✓ |
| AR festschreiben | – | – | – | ✓ | ✓ |
| AR versenden | – | – | – | ✓ | ✓ |
| **Korrektur erstellen** | – | – | – | ✓ | ✓ |
| **Korrektur freigeben** | – | – | – | – | ✓ |
| Storno erstellen | – | – | – | – | 🔒 |

---

## 8. Automationen (Ergänzung)

### AUTO-AR-FESTSCHREIBUNG

| Attribut | Wert |
|----------|------|
| **ID** | AUTO-AR-01 |
| **Trigger** | AR.status wird 'FREIGEGEBEN' |
| **Bedingung** | AR.ist_festgeschrieben = FALSE |
| **Aktion** | Aufgabe an FIB: "AR [Nr] zur Festschreibung bereit" |
| **Timeout** | 3 Tage |
| **Eskalation** | Nach Timeout: Aufgabe an GF |

### AUTO-AR-KORREKTUR-ALERT

| Attribut | Wert |
|----------|------|
| **ID** | AUTO-AR-02 |
| **Trigger** | Korrekturrechnung erstellt |
| **Aktion** | 1. E-Mail an GF<br>2. Dashboard-Alert<br>3. Audit-Log |

---

## 9. Migration

### 9.1 Bestehende Daten

```sql
-- Bestehende Rechnungen als "festgeschrieben" markieren wenn versendet
UPDATE ausgangsrechnung 
SET ist_festgeschrieben = TRUE,
    festgeschrieben_am = versendet_am
WHERE status IN ('VERSENDET', 'TEILZAHLUNG', 'BEZAHLT', 'UEBERFAELLIG', 'GEMAHNT', 'STORNIERT');

-- Erstellungsart setzen
UPDATE ausgangsrechnung 
SET erstellungsart = CASE 
  WHEN auftrag_id IS NOT NULL THEN 'AUS_AUFTRAG'
  ELSE 'MANUELL'
END
WHERE erstellungsart IS NULL;
```

---

## 10. Testfälle

| Test-ID | Beschreibung | Erwartetes Ergebnis |
|---------|--------------|---------------------|
| TC-AR-01 | AR aus Angebot erstellen | Titel-Struktur wird übernommen |
| TC-AR-02 | AR manuell erstellen | Freie Positionserfassung möglich |
| TC-AR-03 | Position aus Projekt importieren | EK→VK Transformation korrekt |
| TC-AR-04 | AR im Entwurf bearbeiten | Alle Felder editierbar |
| TC-AR-05 | AR festschreiben | Keine Änderung mehr möglich |
| TC-AR-06 | Korrekturrechnung erstellen | Original-AR wird als storniert markiert |
| TC-AR-07 | Teilkorrektur erstellen | Nur Differenz in neuer AR |
| TC-AR-08 | Bereits abgerechnete Position erneut importieren | Wird verhindert mit Warnung |
| TC-AR-09 | Preisabweichung ohne Begründung | Speichern wird verhindert |
| TC-AR-10 | Korrekturkette anzeigen | Alle Korrekturen sichtbar |

---

*Patch-Version: 2.0*  
*Autor: ERP-Entwicklungsteam*  
*Review erforderlich durch: Projektleitung*
