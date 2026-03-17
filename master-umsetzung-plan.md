# OpenMic — Master-Umsetzungsplan: Von 0 bis Markt

> Open-source Meeting-Capture für macOS. Kein Bot. Kein Lock-in. Markdown. REST API. Agent-first.

---

## Die eine Entscheidung, die alles andere bestimmt

Bevor irgendwas passiert:

**"Baue ich das selbst, oder suche ich jemanden, der es baut?"**

| Pfad | Was es braucht | Timeline bis MVP |
|------|---------------|-----------------|
| Selbst bauen | Starke Swift/macOS-Kenntnisse, 3-4 Monate Vollzeit | ~4 Monate |
| Co-Founder finden | 1-2 Wochen Suche, dann 3-4 Monate gemeinsam | ~5 Monate |
| Contractor beauftragen | $10-20k Budget, klare Spec (haben wir) | ~3 Monate |

---

## Die These

In 2026 bestimmt sich der Wert von Meeting-Notes dadurch, ob AI-Agents sie lesen können. Granola hat sich selbst disqualifiziert (verschlüsselte DB, MCP-only). Das Fenster ist jetzt offen — aber nur für Wochen, nicht Monate.

**Absorbing States (worauf alles zurückfällt):**
1. Openness schlägt Lock-in — immer
2. Agent-readable by default ist das eigentliche Produkt
3. Das Dateiformat (`~/OpenMic/*.md` + YAML-Frontmatter) ist wichtiger als jedes Feature

**Die größte Illusion:** Dass local-only Transcription + local LLM gut genug ist für den Free-Tier. Whisper-base auf einem 8GB MacBook Air + 7B-LLM-Summaries fühlen sich an wie B-minus. Plan B: Cloud-Transcription für die ersten 1.000 User subventionieren, local als Cost-Saving nachziehen.

---

## Phase 0: Entscheidung & Setup (Woche 1)

### Ziel: Commitment + Infrastruktur

| # | Aktion | Output | Deadline |
|---|--------|--------|----------|
| 0.1 | Entscheidung treffen: selbst / Co-Founder / Contractor | Klares Commitment | Tag 1 |
| 0.2 | Falls Co-Founder: Post in Swift-Communities (r/swift, Swift Forums, Indie Dev Monday, iOS Dev Weekly) | Bewerbungen | Tag 1-3 |
| 0.3 | Apple Developer Account ($99/yr) einrichten | Developer ID für Signierung | Tag 1 |
| 0.4 | Xcode-Projekt anlegen: SwiftUI macOS App, SPM | Kompilierbares Projekt im Repo | Tag 1 |
| 0.5 | CI/CD: GitHub Actions für Build + Notarization | Automatische `.dmg`-Erstellung | Tag 2-3 |
| 0.6 | Trademark-Check "OpenMic" | Go/No-Go für den Namen | Tag 2 |

**Bottleneck:** Die Entscheidung selbst. Alles andere ist Prokrastination.

---

## Phase 1: Audio Capture + Transcription — MVP (Wochen 2-6)

### Ziel: App nimmt Meeting-Audio auf und erstellt ein Transkript

```
[Woche 2-3]  Menu-Bar-Shell + Audio Capture
[Woche 4-5]  whisper.cpp Integration
[Woche 6]    Transcript-Output als .txt + erste Tests
```

#### 1.1 Menu-Bar App Shell (Woche 2)
- SwiftUI `MenuBarExtra` mit Mic-Icon
- Start/Stop-Button
- Status-Anzeige: "Capturing: Zoom"
- Kein Dock-Icon (menu bar only)

**Input:** Xcode-Projekt
**Output:** Laufende App mit UI-Shell, noch ohne Audio
**Bottleneck:** Keiner — das ist der einfache Teil

#### 1.2 Audio Capture (Woche 2-3)
- `ScreenCaptureKit` (`SCStream`) für Audio-only Capture
- Per-App Audio-Isolation: Zoom, Meet, Teams, Slack per Bundle-ID erkennen
- Mic-Input separat für Speaker-Diarization (Mic = "Du", System = "Andere")
- Audio-Buffer im Memory, temp `.wav`-Chunks für Verarbeitung
- Permissions: Screen Recording + Microphone

**Input:** App-Shell + ScreenCaptureKit-Docs
**Output:** Audio-Bytes fließen von Meeting-Apps in die App
**Bottleneck:** ScreenCaptureKit ist schlecht dokumentiert für Audio-only. Apple's Entitlements + Sandboxing = 2-3 Tage Frust einplanen.

**Key Decision:** ScreenCaptureKit (macOS 13+, einfacher, braucht Screen Recording Permission) vs. Core Audio Tap (macOS 14+, gezielter, neuer). **Empfehlung: ScreenCaptureKit** — breitere Kompatibilität, mehr Beispiele online.

#### 1.3 Transcription Engine (Woche 4-5)
- `whisper.cpp` via Swift C-Interop einbinden
- Model-Download-Manager: `large-v3-turbo` (Default) + `base` (Fallback für schwache Macs)
- 30-Sekunden-Chunks mit 5s Overlap
- Partial Transcripts in die UI streamen
- Speaker-Labels: Mic-Kanal = "Du", System-Kanal = "Andere"

**Input:** Audio-Buffer aus 1.2
**Output:** Timestamped Transcript-Segmente
**Bottleneck:** Memory-Management. `large-v3` braucht ~3GB RAM. Auf 8GB-Macs wird's eng. Braucht intelligentes Model-Switching.

#### 1.4 Transcript Output (Woche 6)
- Raw Transcript als `.txt` speichern in `~/OpenMic/transcripts/`
- Format: `[HH:MM:SS] Speaker: Text`
- Erste End-to-End-Tests mit echten Meetings

**Input:** Transcript-Segmente aus 1.3
**Output:** Lesbare `.txt`-Dateien auf Disk
**Bottleneck:** Keiner — Datei schreiben ist trivial

### Phase 1 Exit-Kriterium
> Die App läuft im Menu Bar, captured Audio von Zoom, transkribiert es lokal, und speichert ein lesbares Transkript als Textdatei. Kein Markdown, keine AI-Notes, keine API. Aber es funktioniert.

---

## Phase 2: Note Generation + Markdown (Wochen 7-10)

### Ziel: Aus Transkripten werden strukturierte Markdown-Notes

```
[Woche 7-8]  LLM-Integration (lokal + Cloud)
[Woche 9]    Markdown Writer + Frontmatter
[Woche 10]   Note Browser UI + Templates
```

#### 2.1 LLM Summarization (Woche 7-8)
- **Lokal:** `llama.cpp` oder `MLX` — Phi-3 / Mistral-7B / Llama-3-8B
- **Cloud (opt-in):** User gibt eigenen API-Key ein (OpenAI, Anthropic, Deepgram)
- System-Prompt erzeugt strukturierten Output:
  - Titel (auto-generiert)
  - Summary (2-3 Sätze)
  - Key Decisions
  - Action Items (mit Assignees)
  - Diskussions-Themen
- `NoteGenerator`-Protocol: abstrahiert Local vs. Cloud
- API-Keys im macOS Keychain, nie in Config-Dateien

**Input:** Raw Transcript
**Output:** Strukturiertes Note-Objekt
**Bottleneck:** Qualität lokaler LLMs. Hier entscheidet sich, ob der Free-Tier benutzbar ist. **Fallback-Plan:** Cloud-Transcription für Early Adopters subventionieren.

#### 2.2 Markdown Writer (Woche 9)
- YAML-Frontmatter: title, date, duration, participants, app, tags, template, openmic_version
- Body: Summary → Decisions → Action Items → Discussion → Raw Transcript (in `<details>`)
- Dateiname: `YYYY-MM-DD-slugified-title.md`
- Output-Directory: `~/OpenMic/` (konfigurierbar)
- File-Watcher für externe Edits

**Das Dateiformat ist das wichtigste Deliverable des gesamten Projekts.** Es muss so gut sein, dass es zum Standard wird. Jeder Agent, jedes Tool soll diese `.md`-Dateien nativ verstehen.

```yaml
---
title: "Sprint Planning - Q1 Week 12"
date: 2024-03-15T10:00:00-05:00
duration: 45m
participants: [You, Sarah Chen, Mike Johnson]
app: zoom
tags: [sprint, planning]
template: default
openmic_version: "1.0.0"
---
```

**Input:** Note-Objekt aus 2.1
**Output:** `.md`-Datei auf Disk
**Bottleneck:** Keiner technisch — aber das Schema muss stimmen. Hier 2-3 Tage in Format-Design investieren.

#### 2.3 Note Browser + Templates (Woche 10)
- `NoteBrowserView`: Liste aller Notes, suchbar, filterbar nach Datum/Tag/App
- `NoteDetailView`: Markdown-Renderer + Inline-Editor
- `LiveCaptureView`: Echtzeit-Transkript + Note-Preview während Capture
- 3 Built-in Templates: Default, Standup, 1:1
- Änderungen im Editor speichern direkt in die `.md`-Datei

**Input:** Markdown-Dateien auf Disk
**Output:** Browsable, editierbare UI
**Bottleneck:** SwiftUI Markdown-Rendering. Nativer Support ist limitiert — evtl. `swift-markdown` oder `Ink` als Parser.

### Phase 2 Exit-Kriterium
> Nach dem Meeting hat man eine saubere `.md`-Datei mit Frontmatter, Summary, Decisions, Action Items. Man kann sie in der App browsen, editieren, oder direkt im Finder/Editor öffnen. Sie ist von jedem AI-Agent lesbar.

---

## Phase 3: REST API (Wochen 11-12)

### Ziel: Programmatischer Zugang zu allem

#### 3.1 Embedded HTTP Server (Woche 11)
- **Swifter** (lightweight) auf `localhost:7890`
- Optional Bearer-Token-Auth
- Endpoints:

```
GET    /api/v1/notes                    # Alle Notes (paginiert, filterbar)
GET    /api/v1/notes/:id                # Einzelne Note (Content + Metadata)
POST   /api/v1/notes                    # Note manuell erstellen
PUT    /api/v1/notes/:id                # Note updaten
DELETE /api/v1/notes/:id                # Note löschen
GET    /api/v1/notes/search?q=keyword   # Volltextsuche
GET    /api/v1/transcripts/:id          # Raw Transcript
POST   /api/v1/capture/start            # Capture starten
POST   /api/v1/capture/stop             # Capture stoppen
GET    /api/v1/capture/status           # Status
GET    /api/v1/health                   # Health Check
```

**Input:** Markdown-Dateien auf Disk + laufende App
**Output:** JSON-API auf localhost
**Bottleneck:** Keiner — ~200 Zeilen Code, File-Reads als HTTP wrappen.

#### 3.2 API Docs + OpenAPI Spec (Woche 12)
- OpenAPI/Swagger-Spec
- Docs-Page auf `localhost:7890/docs`
- Beispiel-Requests in README

**Input:** Fertige Endpoints
**Output:** Dokumentierte API
**Bottleneck:** Keiner

### Phase 3 Exit-Kriterium
> Jedes Tool — Claude Code, Cursor, custom Scripts — kann per HTTP auf alle Meeting-Notes zugreifen, suchen, und die Capture steuern. Das ist der Killer-Differentiator zu Granola.

---

## Phase 4: Polish + Ship (Wochen 13-15)

### Ziel: Produkt, das man Fremden geben kann

#### 4.1 Preferences UI (Woche 13)
- General: Output-Directory, Launch at Login, Menu-Bar-Verhalten
- Audio: Input/Output-Device, App-Erkennung
- Transcription: Model-Auswahl, Sprache, Cloud-Fallback
- AI/Notes: LLM-Auswahl, API-Keys, Default-Template
- API: Enable/Disable, Port, Auth-Token
- Privacy: Daten-Retention, Auto-Delete nach N Tagen

#### 4.2 Onboarding (Woche 13)
- First-Run: Permissions erklären + anfordern
- Model-Download mit Fortschrittsanzeige
- Output-Directory wählen
- Optional: API-Keys eingeben

#### 4.3 Packaging + Distribution (Woche 14)
- Code-Signing mit Apple Developer ID
- Notarization via `notarytool`
- `.dmg` erstellen mit GitHub Actions
- GitHub Release mit Changelog
- Homebrew Cask (nice-to-have, nicht kritisch)

#### 4.4 Landing Page (Woche 14-15)
- 1-Pager: Hero + Value Props + Download-Button + GitHub-Link
- Keine Registrierung, kein Account nötig
- Gehostet auf GitHub Pages oder Vercel
- SEO für "Granola alternative", "open source meeting notes", "macOS meeting capture"

#### 4.5 Beta-Test (Woche 15)
- 10-20 Beta-Tester aus der ursprünglichen Twitter-Diskussion
- Feedback-Kanal: GitHub Issues + Discord
- Kritische Bugs fixen

**Bottleneck:** Notarization ist bürokratisch. Einen vollen Tag einplanen. Beta-Feedback kann Scope-Creep verursachen — streng priorisieren.

### Phase 4 Exit-Kriterium
> Ein Fremder kann die App downloaden, installieren, ein Meeting capturen, und brauchbare Markdown-Notes bekommen. Ohne Hilfe. Ohne Docs lesen. Es funktioniert einfach.

---

## Phase 5: Launch (Woche 16)

### Ziel: 1.000 User in den ersten 30 Tagen

#### Launch-Sequenz (ein Tag, in dieser Reihenfolge):

| Zeit | Aktion | Kanal |
|------|--------|-------|
| 08:00 | Product Hunt Launch | producthunt.com |
| 09:00 | Hacker News "Show HN" Post | news.ycombinator.com |
| 10:00 | Twitter/X Thread (Antwort auf den Original-Granola-Tweet) | twitter.com |
| 10:30 | Reddit Posts | r/macapps, r/productivity, r/selfhosted |
| 11:00 | Persönliche DMs an Leute aus der Granola-Diskussion | twitter.com |
| 14:00 | LinkedIn Post | linkedin.com |

#### Launch-Assets (vorher vorbereiten):
- [ ] 60-Sekunden Demo-Video (Screen Recording: Meeting → Capture → Notes → API-Call)
- [ ] 5 Screenshots: Menu Bar, Note Browser, Markdown-Output, API-Response, Preferences
- [ ] Twitter-Thread (8-10 Tweets): Problem → Lösung → Demo → Download
- [ ] HN-Kommentar: technische Details, warum ScreenCaptureKit, warum whisper.cpp
- [ ] Product Hunt Tagline: "Your meetings, as markdown files. Open source. Agent-ready."

#### Launch-Narrative:
> "Granola ging closed. Wir gehen open. Gleiche UX — deine Daten gehören dir. Markdown-Files, REST API, lokal-first. Für eine Welt, in der Agents deine Meeting-Notes lesen."

**Bottleneck:** Timing. Die Twitter-Diskussion ist jetzt heiß. Jede Woche, die vergeht, kühlt die Aufmerksamkeit ab. Wenn möglich: MVP (Phase 1) früh shippen als "alpha", vollständiges Produkt nachlegen.

### Phase 5 Exit-Kriterium
> 1.000+ Downloads, 500+ aktive User, 100+ GitHub Stars, aktive Community in Issues/Discord.

---

## Phase 6: Monetarisierung (Wochen 17-22)

### Ziel: Erste zahlende Kunden

**Wichtig:** Erst monetarisieren, wenn das kostenlose Produkt bewiesen ist. Nicht vorher.

#### 6.1 Tier-Struktur

| Tier | Preis | Was |
|------|-------|-----|
| **Free** | $0 forever | Alles lokal: Capture, Transcription, LLM-Notes, Markdown, API. Keine Limits. |
| **Pro** | $12/mo · $99/yr | Managed Cloud-Transcription (Deepgram), Managed LLM-Summarization (Claude/GPT-4), Advanced Speaker-Diarization, Meeting-Analytics, Custom Vocabulary, Email/Slack-Delivery |
| **Team** | $8/user/mo (min 5) | Shared Note Library, Team Search, SSO/SAML, Admin Controls, Shared Templates, Audit Log |
| **Enterprise** | $25/user/mo | Self-Hosted, On-Prem Transcription, Custom LLM, Dedicated Support, SLAs |

#### 6.2 Monetarisierungs-Prinzipien (nicht verhandelbar)
1. **API nie hinter Paywall** — Free User bekommen die volle REST API
2. **Daten-Export nie einschränken** — Markdown-Files gehören dem User
3. **Lokale Features nie einschränken** — Alles auf deinem Mac ist kostenlos
4. **Conversion durch Quality-Gap** — Cloud ist besser als lokal. User upgraden, weil sie wollen, nicht weil wir den Free-Tier kastriert haben
5. **Open Source = Growth Engine** — MIT-Lizenz bedeutet: Entwickler bauen Integrationen, schreiben darüber, empfehlen es

#### 6.3 Implementierung

| # | Aktion | Woche |
|---|--------|-------|
| 6.3.1 | Stripe/LemonSqueezy Integration | 17-18 |
| 6.3.2 | Cloud-Transcription Backend (Deepgram-Proxy) | 18-19 |
| 6.3.3 | Cloud-LLM Backend (Anthropic/OpenAI-Proxy) | 19-20 |
| 6.3.4 | Pro-Feature-Gating in der App | 20 |
| 6.3.5 | 30-Tage Free Trial für Pro | 20 |
| 6.3.6 | Usage-Tracking + Billing (Meetings/Monat) | 21 |
| 6.3.7 | Account-System (minimal: Email + API-Key) | 21-22 |

#### 6.4 Unit Economics

| Kostenposition | Pro Monat/User |
|----------------|---------------|
| Cloud-Transcription (~20 Meetings, 30min) | ~$3.00 |
| Cloud-LLM Summarization | ~$0.80 |
| Infrastruktur (API, Storage, Delivery) | ~$0.50 |
| **Gesamt COGS** | **~$4.30** |
| **Revenue bei $12/mo** | **$12.00** |
| **Bruttomarge** | **~64%** |

**Risiko:** Wenn Power-User 40+ Meetings/Monat haben, sinkt die Marge auf ~40%. **Lösung:** Fair-Use-Limit bei 50 Meetings/Monat, danach $0.30/Meeting.

#### 6.5 Revenue-Projektion (konservativ)

| Monat | Free User | Pro Subs | Pro MRR | Team Subs | Team MRR | **Gesamt MRR** |
|-------|-----------|----------|---------|-----------|----------|---------------|
| 6 | 5.000 | 250 | $3.000 | 0 | $0 | **$3.000** |
| 12 | 25.000 | 1.750 | $21.000 | 200 | $1.600 | **$22.600** |
| 24 | 100.000 | 8.000 | $96.000 | 2.000 | $16.000 | **$112.000** |

### Phase 6 Exit-Kriterium
> 50+ zahlende Pro-Subscriber, $600+ MRR, positive Unit Economics, kein User-Churn wegen Qualitätsproblemen.

---

## Phase 7: Moat bauen (Wochen 23+)

### Ziel: Uneinholbar werden — nicht durch Lock-in, sondern durch Standard-Setzung

#### 7.1 Das Dateiformat als Standard etablieren
- **OpenMic Note Format Spec** als eigenes Dokument veröffentlichen
- Andere Tools ermutigen, das gleiche Frontmatter-Schema zu nutzen
- "OpenMic-compatible" Badge für Tools, die das Format lesen/schreiben
- **Das ist der echte Moat.** Features sind temporär. Standards sind für immer.

#### 7.2 Ecosystem
- Template Marketplace (Community-built, 70/30 Revenue Share)
- Integration Plugins: Notion Export, Linear Ticket Creation, CRM Sync, Obsidian Sync
- MCP-Server (auf der REST API aufgebaut — nicht als Ersatz)

#### 7.3 Calendar Integration (v1.1)
- EventKit: Kalender-Events matchen → Meeting-Titel, Teilnehmer, Agenda auto-befüllen
- Pre-Meeting Note Stubs erstellen

#### 7.4 Weitere Revenue-Kanäle
- Affiliate-Partnerships mit Deepgram, AssemblyAI (für BYOK-User)
- Enterprise Self-Hosted ($25/user/mo)
- Consulting für Custom Deployments

---

## Zusammenfassung: Kritischer Pfad

```
Woche 1     ████ Phase 0: Entscheidung + Setup
Woche 2-6   ████████████████████ Phase 1: Audio + Transcription (MVP)
Woche 7-10  ████████████████ Phase 2: Notes + Markdown
Woche 11-12 ████████ Phase 3: REST API
Woche 13-15 ████████████ Phase 4: Polish + Ship
Woche 16    ████ Phase 5: LAUNCH
Woche 17-22 ████████████████████████ Phase 6: Monetarisierung
Woche 23+   ████████████████████████████ Phase 7: Moat + Ecosystem
```

**Total bis Launch: ~4 Monate**
**Total bis erste Revenue: ~5.5 Monate**

---

## Die 5 Dinge, die dieses Projekt killen können

| # | Risiko | Wahrscheinlichkeit | Mitigation |
|---|--------|-------------------|------------|
| 1 | Kein Swift-Developer verfügbar | Hoch | Sofort entscheiden: selbst lernen, Co-Founder, oder Contractor. Nicht warten. |
| 2 | ScreenCaptureKit funktioniert nicht zuverlässig | Mittel | Prototype in Woche 2. Wenn es nicht geht, auf Virtual Audio Device (BlackHole) als Fallback. |
| 3 | Lokale Transcription ist zu schlecht | Hoch | Cloud-Transcription für Early Adopters subventionieren. Qualität > Purismus. |
| 4 | Jemand anders baut es schneller | Hoch | MVP so früh wie möglich shippen (Phase 1 als Alpha). Community aufbauen bevor das Produkt perfekt ist. |
| 5 | Apple ändert ScreenCaptureKit-APIs | Niedrig | Abstraction Layer in `AudioCaptureManager`. Nicht direkt an eine API koppeln. |

---

## Nächster Schritt: Jetzt, in den nächsten 30 Minuten

**Öffne Xcode. Erstelle ein neues SwiftUI macOS Projekt. Füge ein Menu-Bar-Icon hinzu. Lass `ScreenCaptureKit` Audio-Bytes in die Console loggen.**

Alles andere ist Planung. Die Planung ist fertig. Jetzt wird gebaut.
