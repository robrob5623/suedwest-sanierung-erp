```markdown
# A0_context.md

---

# Projektüberblick

**Projektname:** Schadenradar

**Projekttyp:** Webbasiertes SaaS-Portal

**Auftraggeber:** SüdWest Sanierung

**Zielnutzer:** Multiplikatoren für Gebäudeschäden, die die SüdWest Sanierung abwickelt(Hausverwaltungen, Versicherungsagenturen, Makler, Regulierer, Sachverständige, Kommunen, Baubiologen)

**Kernzweck:**  
Zentrale, transparente Plattform für die digitale Schadenmeldung und -verfolgung. Multiplikatoren melden Schäden und verfolgen den gesamten Bearbeitungsstatus digital.

**Primärziele:**
- Schadenmeldungen entgegennehmen
- Projektstatus live anzeigen
- Dokumente zentral bereitstellen
- Kommunikation direkt am Schaden ermöglichen
- E-Mails, Telefonate und manuelle Nachfragen reduzieren

**Ablöseziel:** Perspektivische Ablösung von Trello-Boards und manuellen Dateiablagen.

---

# Zielgruppen

| Rolle | Beschreibung |
|-------|--------------|
| Multiplikator-Admin | Administrativer Zugang für Multiplikator-Organisationen |
| Multiplikator-User | Standardnutzer bei Multiplikatoren |
| Projektleiter SüdWest | Operative Steuerung der Schadenprojekte |
| Innendienst/Buchhaltung | Kaufmännische Bearbeitung |
| Sachverständige/Regulierer | Externe Gutachter (optional) |
| System-Admin | Technische Administration |

---

# Problemstellung heute

**Ist-Zustand:**
- Pro Multiplikator ein eigenes Trello-Board
- Schaden = Trello-Karte
- Phasen = Trello-Listen
- Dokumente extern (SharePoint/Ordner)
- Kommunikation per E-Mail/Telefon
- Doppelte Datenpflege zwischen ERP und Trello

**Identifizierte Probleme:**
- Fehlende Transparenz
- Medienbrüche
- Manuelle Ablage
- Hoher Koordinationsaufwand
- Kein zentrales Reporting

---

# Zielbild / Vision

Ein zentrales Schadenportal mit folgenden Eigenschaften:

| Funktion | Beschreibung |
|----------|--------------|
| Live-Kontrollzentrum | Karte mit Radar-Punkten |
| Status-Tracking | Nachverfolgung je Schaden |
| Digitale Schadenakte | Strukturierte Dokumentenablage |
| Dokumentenautomation | Automatische Ablage via OCR |
| Aktivitäten-Log | Timeline aller Aktivitäten |
| Kalenderansicht | Terminübersicht |
| KPI-Dashboard | Statistiken und Reporting |
| Finanzübersicht | Angebots- und Rechnungsstatus |
| ERP-Integration | Single Source of Truth |

**Mandantenfähigkeit:** Multiplikatoren sehen ausschließlich ihre eigenen Schäden.

---

# Kernfunktionen

## Schadenmanagement
- Schaden anlegen
- Stammdaten erfassen
- Status/Phase verfolgen
- Priorisierung

## Digitale Schadenakte
- Strukturierte Ordner
- Dokumenten-Upload
- Automatische Zuordnung via OCR
- Versionierung
- Audit Trail

## Kommunikation
- Kommentare direkt am Schaden
- Aktivitäten-Log
- Benachrichtigungen

## Transparenz
- Timeline aller Ereignisse
- Kalender
- KPI/Statistiken
- Angebots-/Rechnungsstatus

## OCR / Dokumentautomation
1. OCR-Durchführung bei Upload
2. Extraktion von Projektnummer/Adresse/Schadennummer
3. Automatische Zuordnung zum Schaden
4. Ablage im passenden Ordner
5. Log-Eintrag erstellen

Manuelle Zuordnung nur bei niedriger Trefferwahrscheinlichkeit.

## Standard-Schadenphasen (Workflow)
Hauptphasen:

🔴Leckortung
🟠Leckagebehebung
🟡Rückbau
🔵Trocknung
🟢Wiederherstellung

🔴🟠🟡🔵🟢 Einheitliche Portal-Statuslogik

Diese Status gelten für alle Phasen identisch:

Schadenradar-Status	Bedeutung für Multiplikator	ERP Substatus Mapping
TERMIN_OFFEN	Termin noch nicht geplant	ORTSTERMIN_OFFEN
TERMIN_GEPLANT	Termin beauftragt	ORTSTERMIN_BEAUFTRAGT
VOR_ORT_ERFOLGT	Einsatz durchgeführt	ORTSTERMIN_ERFOLGT
DOKUMENTATION	Bericht/Protokoll/Angebot wird erstellt	PROTOKOLLERSTELLUNG, ANGEBOTSERSTELLUNG
FREIGABE_AUSSTEHEND	wartet auf Prüfung/Freigabe	ANGEBOT_ERSTELLT, ANGEBOT_GEPRUEFT, ANGEBOT_KORREKTUR, ANGEBOT_VERSENDET
IN_DURCHFUEHRUNG	Arbeiten laufen	DURCHFUEHRUNG
ABNAHME	Abnahme/Prüfung	ABNAHME_OFFEN, ABNAHME_ERFOLGT
ABRECHNUNG	Rechnung in Bearbeitung	RECHNUNGSERSTELLUNG, RECHNUNG_ERSTELLT, RECHNUNG_GEPRUEFT, RECHNUNG_KORREKTUR, RECHNUNG_VERSENDET
ABGESCHLOSSEN	Phase fertig	PHASE_ABGESCHLOSSEN

---

# Tech-Stack

## Frontend
| Technologie | Zweck |
|-------------|-------|
| Next.js 14 | Framework |
| Shadcn/UI | Komponentenbibliothek |
| TailwindCSS | Styling |

## Backend
| Technologie | Zweck |
|-------------|-------|
| Supabase | PostgreSQL, Auth, Storage, Edge Functions |
| tRPC | API-Layer |

## Automation & OCR
| Technologie | Zweck |
|-------------|-------|
| n8n | Workflow-Automation |
| Azure Document Intelligence | OCR |

## Deployment
| Plattform | Zweck |
|-----------|-------|
| Vercel | Frontend-Hosting |
| Supabase Cloud | Backend-Hosting |

## ERP-Integration
- ERP ist führendes System (Source of Truth)
- Schadenradar zeigt zensierte ERP-Daten an

---

# Corporate Design

## Farbpalette
| Bezeichnung | Hex-Wert |
|-------------|----------|
| Primary | `#84bc47` |
| Gray | `#6f6f6f` |
| Background | `#f2f0ec` |
| Text | `#000000` |
| White | `#ffffff` |

## Typografie
- Schriftart: Inter

## Layout-Vorgaben
- Light Mode only
- Sidebar: 256px
- Cards: `rounded-lg`, `shadow-sm`
- Stil: Ruhige, professionelle Operations-UI

---

# Randbedingungen

| Anforderung | Beschreibung |
|-------------|--------------|
| Multi-Tenant | Mandantenfähige Architektur |
| Datenschutz | DSGVO-konform |
| Datenführung | ERP-first (keine doppelte Datenhaltung) |
| Skalierbarkeit | Ausgelegt für viele Multiplikatoren |
| Mobile Nutzung | Via PWA |

---

# Annahmen

- ERP-Schnittstelle ist verfügbar und liefert strukturierte Daten
- OCR erreicht eine ausreichende Trefferquote für automatische Zuordnung
- Multiplikatoren akzeptieren den Wechsel von Trello zum neuen Portal
- Dokumente enthalten extrahierbare Merkmale (Projektnummer, Adresse, Schadennummer)
```