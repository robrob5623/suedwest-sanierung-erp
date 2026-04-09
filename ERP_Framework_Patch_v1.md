# ERP-Framework Extension (Meeting-Patch)

## Rollen, App, Disposition, Arbeitsvorbereitung, Qualität, Newsletter, Logistik, Mietservice, Container, Einkauf, Schadenmanager, HR

---

# 0. Leitprinzipien (erweitert)

Die bestehenden Leitprinzipien bleiben unverändert und werden erweitert:

1. **Database-First / SCHADEN_PROJEKT als Kernobjekt** bleibt unverändert.

2. **No-Silent-Failure** gilt nun explizit auch für:
   - Kalender-Änderungen
   - Vorplanung → Terminierung
   - Geräte-Blockungen
   - MBL-Finalisierung
   - Qualitätsabnahmen
   - Projekt-Sperren
   - Inventarbewegungen (Ein-/Ausbuchung, Rückgabe, Reklamation)
   - Geräte-Lifecycle (`FREI`/`VORGEPLANT`/`ZUR_REINIGUNG`/`GESPERRT`)
   - Liefertermine (verschiebt sich hinter Baustart → automatische Benachrichtigung)
   - MBL/Kommissionierung (Teil-kommissioniert, nicht realisierbar → Pflichtbegründung)
   - Container-Bestellung (Entwurf/To-do an Disponent, schriftlicher Nachweis, Blacklistprüfung)
   - On-Hold-Projekte (wöchentliche Statusaktualisierung + Vorgesetzten-Sicht)

3. **Mobile-First** bleibt, wird um neue Rollen (Azubi/Helfer/Facharbeiter) + neue App-Screens erweitert.

---

# 1. Rollenmodell (erweitert)

## 1.1 Neue/konkretisierte Rollen

| Rolle | Beschreibung |
|-------|--------------|
| `AZUBI` | Auszubildender mit Lernfunktionen |
| `BAUHELFER` | Reduzierte Rechte gegenüber Facharbeiter |
| `FACHARBEITER` | Basis-Handwerkerrolle (ehem. HW) |
| `VORARBEITER` | VL bleibt, wird funktional erweitert |
| `BAULEITER` | BL mit Disposition & Qualitätslogik |
| `PROJEKTLEITER` | PL mit Preis-/DB-Sicht |
| `SCHADENMANAGER` | SM mit nahezu PL-Rechten |
| `LOGISTIK` | Granular getrennt von Einkauf |
| `EINKAUF` | Granular getrennt von Logistik |
| `HR` | Erweitert um Azubi-/Lern-Funktionen |
| `GESCHAEFTSLEITUNG` | GL mit Vollzugriff |

**Rollen-Mapping zum bestehenden Blueprint:**

- `HW (Handwerker)` → `FACHARBEITER` (Basis) + `BAUHELFER` (reduziert)
- `VL (Vorarbeiter)` → bleibt, wird funktional erweitert
- `EK/LOG` → wird in `EINKAUF` und `LOGISTIK` granular getrennt (Rechte & To-Dos)
- `HR` → bleibt, wird um Azubi-/Lern-Funktionen erweitert

---

## 1.2 Preis-Sichtbarkeit (neues, hartes Recht)

| Rolle | Preis-Sicht |
|-------|-------------|
| `BAULEITER` | **KEINE** Preise in Angebot/Auftrag/Rechnung (nur Positionen ohne Werte) |
| `VORARBEITER` | **KEINE** Preise |
| `FACHARBEITER` | **KEINE** Preise |
| `BAUHELFER` | **KEINE** Preise |
| `AZUBI` | **KEINE** Preise |
| `PROJEKTLEITER` | Volle Preis-/DB-Sicht |
| `GESCHAEFTSLEITUNG` | Volle Preis-/DB-Sicht |
| `CONTROLLING` | Volle Preis-/DB-Sicht |

---

## 1.3 Bewertungs-/Disziplin-Sichtbarkeit

### Bewertungssystem

- Bauleiter kann bewerten, **nur mit Begründung (Pflichtfeld)**
- Handwerker sehen **nur eigenen Punktestand/Dashboard**, aber **nicht** die interne Bewertungs-/Notizspalte des Bauleiters

### Disziplinarische Maßnahmen

| Aktion | VORARBEITER | BAULEITER | HR/GL |
|--------|-------------|-----------|-------|
| Ermahnung erfassen | Eingeschränkt (weiterleiten) | ✓ (erfassen/auslösen) | ✓ (final) |
| Abmahnung | — | ✓ (erfassen) | ✓ (final) |
| Unfallprüfung | — | ✓ | ✓ |

---

# 2. Modul-Erweiterungen (Blueprint-konform)

## 2.1 M04 – Mobile App (Erweiterung um Rollen & neue Screens)

Das bestehende M04 (Zeiten, Material, Fotos, Status, Formulare, Offline) wird erweitert.

### Neue App-Funktionspakete nach Rolle

#### AZUBI

| Funktion | Beschreibung |
|----------|--------------|
| Einsatzkalender | Aus Teamkalender, inkl. Fahrzeug/Projekt |
| Berufsschule/Prüfungstermine | Kalendersicht |
| Digitales Berichtsheft | Inkl. Spracheingabe |
| Lernportal | Lehrvideos, Wissensmodule |
| Kommunikation | Chat mit Ausbilder |
| KI-Companion | Trainings-Wissensbasis (Name später festlegen) |
| Urlaub/Krankmeldung | Beantragen & hochladen |

#### BAUHELFER

| Funktion | Beschreibung |
|----------|--------------|
| Teamkalender | + eigener Kalender |
| Abwesenheiten | Krankmeldung/Urlaub |
| Ansprechpartner | Telefonliste |
| Erste-Hilfe | Unfallmeldungen |
| Wissensdatenbank | Baustellen-Risiken/Schadstoff-Wissen |
| PSA-Bestellkatalog | Kleidung, Handschuhe, Kniepads etc. |
| Umfragen/Abstimmungen | Nur Führungskräfte starten |

#### FACHARBEITER

- Alles wie `BAUHELFER`
- Zusätzlich: **Material-/Maschinenanfragen** stellen

#### VORARBEITER

- Alles wie `FACHARBEITER`
- Zusätzlich:
  - Tagesbericht
  - Probearbeits-/Probezeit-Feedback
  - Ermahnungen erfassen (weiter an Bauleiter/HR)
  - Abgespeckte Bewertungs-Zuarbeit (v.a. Probezeit)

#### BAULEITER

- App/Web: Qualitäts- und Abnahme-Workflows, Disposition, Plan-Override (siehe M07/M16/M18)

---

## 2.2 M05 – Aufgaben & Projektleiter-Dashboard (Erweiterungen)

Das PL-Dashboard existiert bereits (KPIs, Ampeln, Kanban, Kalender, Portfolio, Eskalation).

### Neu: Projekt-Sperrlogik (Hard Gate)

Bei überfälligen To-Dos/fehlender Aktivität:

| Stufe | Aktion |
|-------|--------|
| Stufe 1 | Erinnerung an PL |
| Stufe 2 | Warnung „Sperre droht" |
| Stufe 3 | **Projekt gesperrt**, Entsperrung nur durch Vorgesetzten (GL/Operative Leitung) |

→ Alles mit Audit-Trail + Eskalationsaufgabe (No-Silent-Failure)

### Neu: Vertretung/Übergabe

Bei Umzuweisung eines Projekts erhält neuer PL automatisch einen **Kurzbericht** aus Audit-Log/Projektstatus (interner Bericht, nicht Kundenbericht).

### Neu: Teamkalender-Miniansicht

Im PL-Dashboard: Wo sind Kolonnen unterwegs.

### Neu: Datei-Upload Drag&Drop

Auf Dashboard-Ebene mit Projektzuweisung (zusätzlich zum Projekt-DMS). DMS bleibt Quelle der Wahrheit.

---

## 2.3 M06 – Angebot & Angebotsprüfung (Erweiterung)

Das Angebotsmodul existiert; es wird erweitert um UI- und Policy-Logik.

### Angebotsstruktur & Bedienlogik

Zwei Hauptaktionen:

1. **„Titel erstellen"** – Vordefinierte Titelvorlagen mit vordefinierten Positionen
2. **„Angebotsposition erstellen"** – Einzelpositionen

**Weitere Logik:**

- Titel können als **Optional/Alternativ** markiert werden → im PDF ausgewiesen, Summenlogik sauber (Alternative nicht in Gesamtsumme oder separat)
- **Positionskatalog öffnen**: Multi-Select, Reihenfolge der Auswahl wird übernommen, Bulk-Übernahme in Angebot

### Angebotsfreigabe / Auftragsbestätigung / Reklamation

**Statuspfad:**

```
Angebot versendet → Dokumentenerkennung „Freigabe" (falls möglich)
    ├─→ FREIGABE → Auftragsbestätigung erzeugen (bearbeitbar)
    └─→ REKLAMATION → Nachverhandlung/Nachkalkulation/Anpassung
```

### Deckungsbeitrags-Policy

- Wenn Projekt-DB unter Schwelle: PL darf **einmalig** freigeben, aber nur mit **Pflichtnotiz (Begründung)**
- Wiederholung/Pattern → automatische Meldung an Vorgesetzten + Audit
- Zeit-/Kostenmanipulation: PL/BL dürfen Zeit nicht „frei" korrigieren; Korrekturen nur über Vorgesetzte (und auditierbar)

---

## 2.4 M07 – Ausführung (Erweiterung)

Ausführungs-/Einsatzplanung/Abnahme/Nacharbeit existiert bereits.

### Neu: Raum-Status & Qualitätslabels

Räume/Teilbereiche erhalten Label-Status:

| Status | Beschreibung |
|--------|--------------|
| `ABGENOMMEN` | Raum vollständig abgenommen |
| `GEPRUEFT` | Qualitätsprüfung durchgeführt |
| `AUSBESSERUNG_NOTWENDIG` | Nacharbeit erforderlich |
| `ABSPRACHE_NOTWENDIG` | Klärungsbedarf mit Kunde/PL |
| `KRITISCH` | Sofortiges Handeln erforderlich |

Bauleiter (und Vorarbeiter „light") kann Räume labeln; Ziel: PL sieht beim Wiedereinstieg sofort Risiken/offene Punkte.

### Neu: Trigger-Benachrichtigungen

| Trigger | Aktion |
|---------|--------|
| **50% Aufgaben erledigt** | BL erhält To-Do „Qualitätskontrolle/Teilabnahme prüfen" |
| **100% Aufgaben erledigt** | To-Do „Abnahme fällig" + Pflichtnotiz bei Ablehnung |
| **Planzeit erreicht, <50% erledigt** | Dringende BL-Benachrichtigung „Kontrollpflicht" |
| **Ungenehmigte Überstunden** | BL-Benachrichtigung zur Vermeidung |

### Zeiterfassung-Sicht für Bauleiter

- BL sieht Ein-/Ausloggen & Zeitbuchungen
- **Kann nicht ändern**, außer per definierter Anfrage/Workflow (mit Audit)

---

## 2.5 M09 – Material/Lager (Erweiterung)

Material existiert (Bestand, Entnahme, Bestellung).

### Neu: MBL – Materialbestellliste (Sub-Modul)

**Automatische MBL-Erstellung nach Auftragsbestätigung:**

- Materialbedarf wird aus Auftrag/Positionen abgeleitet (inkl. Hinweise wie „Farbe immer Eimer" statt ml)
- PL kann Mengen anpassen (z.B. Verschnitt 15–20%)
- Zusätzliche Spalte: **Werkzeug/Equipment-Vorplanung** (nicht Angebot, sondern Arbeitsvorbereitung/MBL)

**Finalisieren-Hard Gate:**

Klick „Finalisieren" → Bestätigungsdialog → danach:
- MBL wird im Projekt festgeschrieben
- To-Dos an **Logistik (Kommissionierung)** und **Einkauf (Beschaffung)** werden erzeugt
- Audit-Log Event

**Logistik-Kommissionierung:**

- Liste abarbeiten (Checkbox je Position)
- Beim Abhaken: Bestände werden abgezogen
- Nicht verfügbar → Notizpflicht (Begründung) → Meldung an PL/BL

**Einkauf:**

- Offene Positionen = „zu beschaffen"
- Optional: Lieferantenanfrage direkt verknüpfen

**Mindestbestand-Automationen** (später aktivieren):

- Unterschreitung (z.B. Farbe < 5 Eimer) → automatische Anfrage/Bestellvorschlag

---

## 2.6 M13 – Formularbaukasten (Erweiterung)

Formularbaukasten existiert (Protokolle, Checklisten, Unterschrift).

### Neue Standardformulare

| Formular | Felder |
|----------|--------|
| **Unfallmeldung** | Inkl. Zeuge/Prüffelder („Unfallheft prüfen") |
| **Ermahnung/Abmahnung** | Vorfallbericht |
| **Fahrzeug-Checklisten** | Digital abhaken, Fehlteile melden → Fuhrparkbeauftragter |
| **Lieferschein-/Geräteaufstellung** | + Zählerstände (Vorarbeiter/Facharbeiter) |

---

## 2.7 M15 – DMS & Kommunikation/Timeline (Erweiterung)

DMS/Timeline/Events existiert, inkl. Audit-Log-Verlinkung.

### Kommunikations-Sichtbarkeit

| Kanal | Sichtbarkeit |
|-------|--------------|
| Handwerker ↔ Projektleiter | Sichtbar für Handwerker (projektbezogen) |
| Bauleiter ↔ Projektleiter | **NICHT** sichtbar für Handwerker (eigener Kanal/Thread) |

„Dringend" Label → sofortige PL-Benachrichtigung („Abstimmung notwendig")

### Kalender-Änderungen als Timeline-Event

Jede Terminänderung (Vorplanung/Terminiert/Verschoben) erzeugt:
- Timeline-Event
- Änderungsbegründung (Pflicht bei Änderungen)
- Audit-Log-Eintrag

---

# 3. Neues Modul-Cluster: Kalender, Disposition, Geräte, Vorplanung (M16 – neu)

## 3.1 Teamkalender & Disposition (Bauleiter)

### Funktionen

| Funktion | Beschreibung |
|----------|--------------|
| Ansicht | Tag/Woche/Monat |
| Drag&Drop | Projekte verschieben |
| Vorplanung | Keine Push-Benachrichtigung an Mitarbeiter |
| Terminiert | Benachrichtigung/Einladung |

### Benachrichtigungsregel Mitarbeiter

- Erinnerung für nächste Woche wird automatisiert **Freitag der Vorwoche** gesendet (nicht 6 Wochen vorher)
- Mitarbeiter dürfen Vorplanung im Kalender sehen (Transparenz), aber werden erst kurz vorher „aktiv" erinnert

### Personal-Disposition

- Monatsübersicht + kleiner Forecast: Wann welche Mitarbeiter verfügbar/verplant
- Urlaubs-/Abwesenheitskalender integriert

### Wetter für Baustellen

Bauleiter-Requirement: Wetterintegration für Baustellenplanung

---

## 3.2 Terminbestätigung (Human-in-the-Loop)

Wenn Terminierung/Disposition abgeschlossen:

1. System erzeugt „**Kunden-Benachrichtigung vorbereiten**"
2. Erst nach Klick **„Bestätigen & senden"** wird E-Mail versandt

→ Vermeidung „aus Versehen terminiert"

---

## 3.3 Subunternehmer-Disposition (optional)

| Schritt | Aktion |
|---------|--------|
| 1 | PL stellt Kapazitätsanfrage an SU |
| 2 | SU antwortet (sichtbar für BL) |
| 3 | BL kann Termin verschieben/planen |
| 4 | Bei Terminänderung: **Notizpflicht** + automatische SU-Benachrichtigung |

---

# 4. Geräte- und Blocklogik (Erweiterung M09/M16)

## 4.1 Geräte vorplanen (PL) + Override (BL)

- PL kann Geräte (z.B. Trockner) **vorplanen/blocken**
- Blockdauer: **max. 2 Wochen**, danach automatische Freigabe + Hinweis
- BL kann Override setzen (Dringlichkeit/Projektabschluss):
  - PL erhält sofort Task „Umplanung erforderlich"
  - Audit-Log Pflicht

## 4.2 Engpass-Alerts

Gerät fehlt / Kapazität überschritten:
- Nachricht an PL + BL (ggf. „externer Dienstleister" Trigger)

---

# 5. Arbeitsvorbereitung nach Räumen (neue Detail-Logik)

Nutzt M05/M07/M13/M15.

## 5.1 Struktur

- **Titel = Raum** (aus Auftrag ableitbar)
- **Positionen darunter = Aufgaben** (automatisch aus Auftragspositionen / Katalog)

## 5.2 Task-Interaktion (App)

### Pro Aufgabe:

| Aktion | Beschreibung |
|--------|--------------|
| Play/Stop | Zeiterfassung |
| Erledigt | Wandert nach unten, bleibt sichtbar |
| Blockiert (Doppelklick) | **Notizpflicht** → Meldung an BL |

### Pro Raum:

- Chat/Kommentarlog mit **@Mention** (PL/BL)
- Fotos/Videos hinzufügen

### Stundenpositionen:

1. System erkennt Stundenposition → generiert Berichtsvorlage
2. Nach Stop: Bericht (Dauer, beteiligte MA, Unterschrift MA + Kunde)
3. Nach Unterschrift: Aufgabe gesperrt; Änderungen nur via BL-Workflow + Audit

## 5.3 Erinnerungslogik „Stundenposition beachten"

Wenn Stundenposition am Tag geplant ist: Reminder alle 2 Stunden (09/11/13/15 Uhr) solange nicht „abgehakt".

## 5.4 Rechnungstrigger

| Fortschritt | Aktion |
|-------------|--------|
| 50% erledigt | Hinweis an PL „Teilrechnung möglich" |
| 100% erledigt | Hinweis „Schlussrechnung erforderlich" |

---

# 6. Weekly Customer Update / Newsletter (M19 – neu)

## 6.1 Ablauf

1. System generiert wöchentlichen Projektbericht (intern) + Kundenupdate (extern)
2. PL bekommt freitags: Zusammenfassung (Status, Risiken, Bilder optional, Stunden optional)
3. **Hard Gate: Freigabe erforderlich**; PL kann Inhalte an/abwählen
4. Versand/Empfänger/Status werden im Versand-Log dokumentiert (DMS/Compliance)

---

# 7. Erweiterte Berechtigungs-Regeln

## 7.1 Must-Have (pro Rolle)

| Rolle | Berechtigungen |
|-------|----------------|
| **Alle Ebenen** | Zugriff auf Personalakte-Dashboard (eigene Daten), Zeiterfassung, Abwesenheiten |
| **Azubi/Helfer/Facharbeiter/Vorarbeiter** | Teamkalender lesen; eigene Abwesenheit verwalten; Wissensbasis; Unfall/Erste Hilfe |
| **Vorarbeiter** | Teamzeiten prüfen (nicht final ändern), Tagesberichte, Ermahnungen erfassen/weiterleiten |
| **Bauleiter** | Teamkalender bearbeiten, Disposition, Qualitäts-/Abnahmelogik, Zeit-Monitoring (read), Geräte-Override (mit Benachrichtigung) |
| **Projektleiter** | Angebote/Aufträge/Rechnung/DB, Subunternehmerkartei, Materialbedarf/MBL Freigabe, Newsletter-Freigabe, Eskalations-/Ampelsystem |

## 7.2 Must-Not (harte Verbote)

- BL/VL/HW/Azubi/Helfer/Facharbeiter sehen **keine Preise**
- PL/BL dürfen Zeit nicht „frei" korrigieren; Korrekturen nur über Vorgesetzte + Audit
- Handwerker sehen **nicht** den BL↔PL-Kommunikationskanal sowie BL-Bewertungsnotizen

---

# 8. M09+ – Inventar & Lager (Erweiterung)

## 8.1 Barcode/QR-basiertes Inventar (Pflicht)

Jedes Material/Tool/Gerät erhält **Barcode/QR**.

### Buchungsarten

| Art | Beschreibung |
|-----|--------------|
| `EINBUCHUNG` | Wareneingang, Rückgabe, Umbuchung |
| `AUSBUCHUNG` | Projektzuweisung oder Mitarbeiterzuweisung |
| `RUECKGABE` | Rücknahme ins Lager |
| `REKLAMATION` | beschädigt/fehlt/Fehllieferung |

### Pflichtfelder pro Buchung

- **wer** (Mitarbeiter)
- **wann** (Timestamp)
- **wohin** (Projekt-ID / Lagerort / Fahrzeug / Mitarbeiter)
- **warum** (optional, aber Pflicht bei Abweichungen)

## 8.2 Kommissionierung: Tages- und Vorlauf-To-dos

Dashboard „Kommissionierung" für Logistik:
- „Heute fällig"
- „Morgen fällig"
- „Nächste 3 Tage"

MBL muss für Logistik nach **Fälligkeit/Projektstart** sortierbar sein.

**Double-Control:** Vorarbeiter/Facharbeiter bestätigen digital beim Einladen (Checkliste), um Aussage-gegen-Aussage zu vermeiden.

## 8.3 Inventur & Abnormalitätsberichte (Pflicht, monatlich)

### Monatlicher Inventur-Report an Niederlassungsleiter/GL:

- Bestandsveränderung je Kategorie
- Top-Abweichungen vs. erwartetem Verbrauch
- „Schwund-/Diebstahlindikatoren"

### Anomalie-Erkennung (Ampel)

| Indikator | Beschreibung |
|-----------|--------------|
| Mehrverbrauch-Pattern | Wiederholt bei gleicher PL/Team-Kombination |
| MBL-Abweichung | Geplant vs. tatsächlich ausgebucht über Schwelle |
| Reklamationsquote | Häufige Reklamationen je Lieferant / Material |
| Zubehörverbrauch | Auffällig je Gerät/Team |

---

# 9. M20 – Mietservice & Geräte-Lifecycle (neu)

Für Trockner/Heizgeräte/Klima.

## 9.1 Geräte-Statusmodell (Pflicht)

| Status | Beschreibung |
|--------|--------------|
| `FREI` | Verfügbar |
| `VORGEPLANT` | Für Projekt blockiert |
| `IM_EINSATZ` | Projekt oder Mietkunde |
| `ZUR_REINIGUNG` | Nach Rückgabe |
| `IN_WARTUNG` | DGUV/Service |
| `GESPERRT` | Defekt/fehlende Prüfung |

## 9.2 Entnahme-/Aufstell-/Abhol-Prozess

### Entnahme (Scan)

- Gerät wird Projekt/Mietvorgang zugewiesen
- Technische Daten werden automatisch im Entnahmeschein mitgeführt

### Aufstellung (Scan + Zählerstand + Unterschrift)

- Pflicht: Unterschrift Kunde oder **Notizpflicht**, warum nicht (z.B. Leerstand)
- Optional: GPS-Koordinate speichern (für Map/Routen)

### Zwischenmessungen & Messprotokoll

Messprotokoll kann in den Gerätereport integriert werden (variable Anzahl Messpunkte/Termine)

### Abholung (Scan + Unterschrift)

- Abschlussbericht erzeugen
- Status springt auf `ZUR_REINIGUNG`
- Automatisches To-do an Verantwortlichen „Reinigung durchführen"

### Technikdaten (Pflichtfelder)

| Feld | Beschreibung |
|------|--------------|
| m² Leistung | Trocknung / Trockenhaltung |
| Luftvolumen | m³/h |
| Leistungsaufnahme | kW |
| Bedienungsanleitung | Dokument |
| Wartungs-/DGUV-Historie | Prüfnachweise |

## 9.3 Wartung/DGUV & Intervalle (Pflicht)

Pro Gerät oder Gerätegruppe definierbare Intervalle:

| Wartungsart | Beschreibung |
|-------------|--------------|
| DGUV-Prüfung | Reminder vor Ablauf |
| Funktionsprüfung | Vor Heizperiode (Heizgeräte) |
| Filterprüfung/Filterwechsel | Als Wartungs-Checklist-Punkt |

Jede Wartung erzeugt:
- Event im Geräte-Log
- Dokument/Prüfnachweis-Upload
- Nächster Fälligkeitstermin

## 9.4 Konfliktauflösung: Mietservice vs. Projektvorplanung

Regel: Mietservice kann Geräte-Blockungen **übersteuern**, wenn:
- Dringlichkeits-Workflow ausgelöst wurde
- Projektleiter bekommt automatisch Task „Umplanung erforderlich"
- Audit-Pflicht

## 9.5 Mietservice-Lead-Intake (Website-Schnittstelle)

Website-Kontaktanfragen erzeugen:
- Ticket im Mietservice-Modul
- Verfügbarkeitscheck (inkl. vorgeplante Geräte)
- Dringlichkeitsalarm bei Großanfrage (z.B. „20 Geräte")

## 9.6 Geräte-Map & Ampel (Pflicht)

Karte mit Geräte-Standorten (Projekt/Mietkunde)

| Ampel | Bedeutung |
|-------|-----------|
| 🟢 Grün | Laufzeit ok |
| 🟡 Gelb | Laufzeit läuft in X Tagen aus |
| 🔴 Rot | Laufzeit überschritten / Abholung fällig |

---

# 10. M21 – Container & Entsorgung (neu)

## 10.1 Container-Preislogik für Angebote (Projekt)

- Im Angebot/Auftrag wird standardmäßig der **teuerste Region-Preis** (inkl. Marge) verwendet als Sicherheitsnetz
- Disponent (Container/Entsorgung) optimiert später auf günstigeren Anbieter

## 10.2 Datenfluss (Entwurf → Disponent → schriftlicher Nachweis)

### Projektleiter erstellt Container-Anfrage-Entwurf (Formular):

| Feld | Pflicht |
|------|---------|
| Standort | ✓ |
| Rechnungsadresse | ✓ |
| Öffentlicher Raum/Privat | ✓ |
| Zufahrt/Platz (Breite/Abstände) | ✓ |
| Containerart/Menge | ✓ |
| Foto/Abstellort | Optional, empfohlen |
| Notizfeld | Optional |

### Entwurf triggert:

- To-do an **Container-Disponent**
- Vorschlagliste Containerdienste (Region)

### Disponent:

- prüft Blacklist
- setzt „beauftragt" / „bestätigt"
- speichert schriftlichen Nachweis (Mail/PDF) im Projekt

## 10.3 Blacklist (Pflicht)

Blacklist im Miet-/Container-Kontext:
- Problemkunden / Zahlungsausfälle / Betrugsfälle
- Blockiert automatische Freigabe → zwingt GL-Entscheid

---

# 11. Modul Einkauf (Erweiterung)

## 11.1 Projektzugriff & MBL-Handling

Einkauf sieht **komplette MBL**, mit Markierungen:
- „muss extern beschafft werden"
- „führt zu Mindestbestand-Unterschreitung"

### Einkauf kann setzen:

| Status | Beschreibung |
|--------|--------------|
| `BESTELLT_AM` | Bestelldatum |
| `LIEFERDATUM_ZUGESAGT` | Vom Lieferanten bestätigt |
| `LIEFERDATUM_VERSCHOBEN` | Neue Zusage |
| `NICHT_VERFUEGBAR` | Pflichtbegründung erforderlich |

Einkauf kann im Projekt **nichts löschen**.

## 11.2 Strategischer Einkauf & Basisprodukte

- Basisprodukte sind im System hinterlegt (Lieferant, Preis, Verpackungseinheit, Mindestbestand)
- Wiederkehrender Bedarf (Regel parametrierbar) → Kandidat „strategischer Einkauf"

## 11.3 Preisfortschreibung in Kalkulation/Nachkalkulation

- Preisänderungen wirken **nur für neue Projekte** (bestehende bleiben mit altem Preis gebunden)
- Optionaler Alert an Operative Leitung bei „kritischen Preisänderungen"

## 11.4 Lieferanten-KPIs & Jahresgespräche

### KPIs:

| KPI | Beschreibung |
|-----|--------------|
| Umsatz pro Lieferant | Gesamtvolumen |
| Umsatz pro Produkt pro Lieferant | Produktspezifisch |
| Reklamationsquote | Anteil fehlerhafte Lieferungen |
| Liefertermintreue | Pünktlichkeitsquote |

### Automatische Wiedervorlage:

- Jahresgespräch (Pflicht)
- Optional: Quartalsgespräche

## 11.5 Ideen-/Vorschlagskasten (operativ → Einkauf)

- Alle operativen Rollen dürfen Ideen einreichen (Produkt/Tool/Verbrauchsmaterial)
- **Hard Gate:** Bestellungen außerhalb Projektbedarf/Basisbestand erfordern Pflichtbegründung + Budgetreferenz

## 11.6 Liefertermin-Automation (No-Silent-Failure)

Wenn Lieferdatum > geplanter Baustart/Einbau-Datum:
- Automatische Nachricht an Bauleiter/Disposition + Projektleiter
- Optional „Umplanung erforderlich"-To-do

---

# 12. Modul Logistik (Erweiterung)

## 12.1 Projektakten-Zugriff (Pflicht)

Zugriff auf Projektakte für:
- MBL
- Geräte-/Mietservice-Vorgänge
- Liefer-/Kommissionierstatus
- Relevante Dokumente (Lieferscheine, Reklamationen)

## 12.2 Fuhrparkmanagement (Pflicht)

### Wartungsintervalle:

| Intervall | Beschreibung |
|-----------|--------------|
| Straßenverkehrstauglichkeit | Regelmäßig |
| Öl/Kühlwasser/AdBlue | Nach Kilometer-/Zeitregel |
| Reifenwechsel | Saisonal |
| Service | Nach Plan |
| HU/TÜV | Gesetzlich |

### Weitere Funktionen:

- Fahrzeug-Checklisten (App)
- Wenn Fahrzeug in Werkstatt: Status "nicht verfügbar" + automatische Info an Disposition/Bauleiter

## 12.3 Reklamationen & Liefermonitor (Pflicht)

### Reklamationsworkflow:

beschädigt/fehlt → Foto/Notiz → Lieferant zugeordnet

### Liefermonitor-Status:

| Status | Beschreibung |
|--------|--------------|
| `ANGEFRAGT` | Anfrage gestellt |
| `BESTELLT` | Bestellung ausgelöst |
| `ZUGESAGT` | Termin bestätigt |
| `VERSPAETET` | Verzögerung gemeldet |
| `EINGETROFFEN` | Ware erhalten |

## 12.4 Schnittstelle Logistik ↔ Einkauf (Pflicht)

- Logistik kann Bedarfe an Einkauf senden (inkl. Begründung)
- Mindestbestand-Unterschreitung erzeugt automatisch Einkaufsticket

---

# 13. Schadenmanager (Erweiterung)

## 13.1 Superdashboard über alle Projektleiter (Pflicht)

Aggregiert:
- Kritische Projekte
- Überfällige/eskalierende To-dos
- „Stillstand" (keine Veränderung X Tage) mit Ausnahme „On Hold"
- Auslastung je Projektleiter (# Projekte, Status, Kritikalität)

## 13.2 Eskalationslogik

System eskaliert automatisch stufenweise.

**Ein Tag bevor** die Eskalation an Vorgesetzten geht:
- Schadenmanager erhält kritisches To-do („Intervention möglich")

Ziel: Probleme auf Ebene SM/PL lösen, bevor GL belastet wird.

## 13.3 On-Hold-Projekte (Pflicht)

- Projektleiter kann Status `ON_HOLD` setzen **nur mit Begründung**
- Wöchentliche Pflichtaktualisierung (kurzer Status + Next Step)
- Vorgesetzte sehen On-Hold-Liste inkl. Begründungen

## 13.4 LO/LB Workflowboard (Leckortung/Leckbehebung)

Schadenmanager braucht eigenes Board:
- LO beauftragen
- LB beauftragen
- Nachfassen 2./3. Termin
- Terminzusage / Durchführung / Ergebnisdokumente

Karten-/Map-Ansicht für Dienstleister (geografisch)

## 13.5 Routen-/Terminlogik (optional)

- Slot-basierte Terminierung mit Pufferregel (distanzbasiert)
- Option: „Route gestartet" Event an Schadenmanager (nicht zwingend an Kunden)
- Fokus: Transparenz & Pufferdisziplin

## 13.6 Rechte

Schadenmanager hat **nahezu PL-Rechte**, da er perspektivisch kleinere Projekte selbst führt (inkl. Angebot/Rechnung).

---

# 14. HR (Erweiterung)

## 14.1 Kernfunktionen (Pflicht)

| Funktion | Beschreibung |
|----------|--------------|
| Abwesenheiten | Urlaub/Krankheit inkl. Dokumentupload |
| Personalakten | Bewertungen, Disziplinarverfahren (mit GL-Freigabe) |
| Onboarding/Preboarding/Offboarding | Neu zu modellieren |

### Pflichtwiedervorlagen:

- Erste-Hilfe-Kurse
- Arbeitsmedizinische Untersuchungen
- Unterweisungen/Sicherheitsunterweisungen

Unfallmeldungen laufen als Pflicht-CC an HR.

## 14.2 Schnittstellen

| Schnittstelle | Beschreibung |
|---------------|--------------|
| HR ↔ Recruiting | Recruiting als eigene Domäne, HR erhält Übergaben |
| HR ↔ Finance/Controlling | Mitarbeiter-KPIs, Krankheitstage, Abwesenheitstrends |
| HR ↔ Ausbildung | Schule/Kammer, Berichtsheft, Lernmodule |

## 14.3 Kommunikationskanal

Jeder Mitarbeiter kann HR über App kontaktieren (Ticket/Chat), auditierbar.

---

# 15. Neue Kern-Entitäten (Datenmodell-Erweiterung)

## 15.1 Kalender & Disposition

| Entität | Beschreibung |
|---------|--------------|
| `KALENDER_EVENT` | team/individuell, status: `VORPLANUNG`\|`TERMINIERT`\|`VERSCHOBEN`, notizpflicht bei Änderung |
| `DISPOSITION_PLAN` | Monatsforecast, Ressourcen: Mitarbeiter/Fahrzeug/SU |

## 15.2 Qualität & Bewertung

| Entität | Beschreibung |
|---------|--------------|
| `QUALITAETS_CHECK` | raum, status, notiz, attachments, entschieden_von |
| `BEWERTUNG` | Mitarbeiterbewertung |
| `BEWERTUNG_BEGRUENDUNG` | Pflichtfeld zur Bewertung |
| `DISZIPLIN_EVENT` | Ermahnung/Abmahnung, Verknüpfung HR |
| `UNFALLMELDUNG` | Zeuge, Prüfung, Maßnahmen |

## 15.3 Material & Logistik

| Entität | Beschreibung |
|---------|--------------|
| `MBL` | Materialbestellliste (Kopf) |
| `MBL_POSITION` | Einzelpositionen der MBL |
| `KOMMISSIONIER_STATUS` | Status je Position |
| `INVENTORY_ITEM` | barcode, typ, lagerort, zustand |
| `INVENTORY_TX` | ein/aus/rueckgabe/reklamation, projekt, user, timestamp, reason |
| `DELIVERY` | lieferant, status, zugesagt, verschoben, dokumente |

## 15.4 Geräte & Mietservice

| Entität | Beschreibung |
|---------|--------------|
| `DEVICE` | Gerätestammdaten |
| `DEVICE_EVENT` | Statusänderungen, Einsätze |
| `DEVICE_MAINTENANCE` | Wartungen, DGUV |
| `RENTAL_CASE` | mietkunde/projekt, signaturen, reports, messprotokolle |

## 15.5 Container & Entsorgung

| Entität | Beschreibung |
|---------|--------------|
| `CONTAINER_REQUEST_DRAFT` | Anfrage-Entwurf vom PL |
| `CONTAINER_ORDER` | Beauftragte Bestellung |
| `BLACKLIST_ENTRY` | Gesperrte Kunden/Partner |

## 15.6 Einkauf & Lieferanten

| Entität | Beschreibung |
|---------|--------------|
| `SUPPLIER_KPI` | umsatz, quote, liefertermintreue, reklamationen |

## 15.7 Newsletter & Kommunikation

| Entität | Beschreibung |
|---------|--------------|
| `NEWSLETTER_DRAFT` | Inhalte toggelbar |
| `VERSAND_LOG` | Empfänger/Status |

## 15.8 Projektstatus

| Entität | Beschreibung |
|---------|--------------|
| `ON_HOLD_STATUS` | begruendung, weekly_update, approver_view |

---

# 16. Automationen & Alerts (No-Silent-Failure)

| Trigger | Aktion |
|---------|--------|
| Kalenderänderung ohne Notiz | Block/Fehler |
| Vorplanung > X Wochen ohne Terminierung | Erinnerung BL |
| Geräteblock > 2 Wochen | Auto-Freigabe + Benachrichtigung |
| MBL finalisiert | To-Dos an Logistik/Einkauf + Audit |
| 50/100% Tasks erledigt | Abnahme-/Rechnungs-Trigger |
| Planzeit erreicht & Fortschritt < Schwelle | BL Alarm |
| PL-Inaktivität/To-Do-Überfälligkeit | Sperrkaskade + Eskalation |
| Lieferdatum > Baustart | Benachrichtigung BL/Disposition/PL |

---

**Ende des ERP-Framework Extension Patch**

*Dokumentversion: 1.0 (Patch)*

*Basiert auf: ERP-Framework Blueprint v1.0*

*Für: SüdWest Sanierung GmbH*
