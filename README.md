# Qu Timetracker

Zeiterfassung über Stream Deck und Bitfocus Companion mit automatischer Übertragung der Zeiteinträge an Hakuna.

## 1. Übersicht

Der Qu Timetracker ermöglicht es, Zeiteinträge über ein Stream Deck zu erfassen.

Der Benutzer wählt in Bitfocus Companion:

1. Projekt
2. Tätigkeit / Task
3. Dauer
4. Senden

Die verfügbaren Projekte und Tasks werden aus Hakuna geladen und über Node-RED für Companion bereitgestellt.

Die eigentliche Zeiterfassung erfolgt anschliessend über die Hakuna API.

### Architektur

```text
┌─────────────────────┐
│     Stream Deck     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Bitfocus Companion  │
│      Port 8000      │
└──────────┬──────────┘
           │ HTTP
           ▼
┌─────────────────────┐
│      Node-RED       │
│      Port 1880      │
└──────────┬──────────┘
           │ REST API
           ▼
┌─────────────────────┐
│       Hakuna        │
│      REST API       │
└─────────────────────┘
```

---

# 2. Komponenten

## Node-RED

Node-RED übernimmt die komplette Logik zwischen Companion und Hakuna.

Aufgaben:

- Projekte von Hakuna laden
- Tasks von Hakuna laden
- Projekt- und Task-IDs speichern
- Daten für Companion bereitstellen
- Auswahl von Projekt und Task entgegennehmen
- aktuelle Zeiteinträge von Hakuna abrufen
- nächsten Zeitblock berechnen
- neuen Zeiteintrag an Hakuna senden
- Erfolg oder Fehler an Companion zurückgeben

Webinterface:

```text
http://timetracker.local:1880
```

Node-RED-Konfiguration auf dem Server:

```text
/home/admin/.node-red/
```

Wichtige Dateien:

```text
flows.json
package.json
package-lock.json
```

---

## Bitfocus Companion

Companion stellt die Benutzeroberfläche auf dem Stream Deck bereit.

Webinterface:

```text
http://timetracker.local:8000
```

Aufgaben:

- Projektbuttons anzeigen
- Taskbuttons anzeigen
- Zeit auswählen
- ausgewählte Werte in Custom Variables speichern
- Zeiteintrag an Node-RED senden
- Rückmeldung des Zeitrapports anzeigen

Die Companion-Konfiguration wird als `.companionconfig` exportiert und im Repository unter folgendem Verzeichnis gespeichert:

```text
companion/
```

---

## Hakuna

Hakuna ist das führende System für:

- Projekte
- Tasks
- Benutzer
- Zeiteinträge

API-Basis:

```text
https://app.hakuna.ch/api/v1/
```

Die Authentifizierung erfolgt über:

```text
X-Auth-Token
```

Der API-Token darf NICHT im Git-Repository gespeichert werden.

---

# 3. Repository-Struktur

```text
Qu_Timetracker/
│
├── README.md
├── .gitignore
│
├── node-red/
│   ├── flows.json
│   ├── package.json
│   └── package-lock.json
│
├── companion/
│   └── *.companionconfig
│
└── docs/
```

Repository:

```text
GitHub → Joel-3012/Qu_Timetracker
```

---

# 4. Datenfluss

## Zeiteintrag

Companion sendet beispielsweise:

```json
{
  "Project": "Bärensaal Worb Phase 2",
  "task": "Installation",
  "Time": "75"
}
```

Node-RED verarbeitet diese Daten.

Ablauf:

```text
POST /zeitrapport
        │
        ▼
JSON parsen
        │
        ▼
Eingabe sichern
        │
        ▼
Zeitraum heute bauen
        │
        ▼
Hakuna Zeiteinträge abrufen
        │
        ▼
Zeitblock & Mapping
        │
        ▼
POST an Hakuna
        │
        ▼
Rückmeldung an Companion
```

---

# 5. Berechnung des Zeitblocks

Node-RED fragt zuerst die heutigen Zeiteinträge bei Hakuna ab.

Dabei wird folgender Zeitraum verwendet:

```text
start_date = heute
end_date   = heute
```

Aus den vorhandenen Einträgen wird die letzte Endzeit bestimmt.

Beispiel:

```text
Vorhandener letzter Eintrag:

07:00 – 08:15

Companion:
Time = 30

Neuer Eintrag:

08:15 – 08:45
```

Wenn noch kein Zeiteintrag vorhanden ist, wird aktuell verwendet:

```text
Fallback Startzeit = 07:00
```

---

# 6. Hakuna Mapping

Hakuna arbeitet intern mit IDs.

Companion arbeitet hingegen mit Namen.

Beispiel:

```text
Companion:

Project = Chalet Kudu
Task    = Installation
```

Node-RED übersetzt dies beispielsweise in:

```text
Chalet Kudu   → Project ID 12
Installation  → Task ID 14
```

Dafür werden zwei globale Node-RED-Maps verwendet:

```javascript
global.get("projektMap")
global.get("activityMap")
```

Diese werden durch den Flow

```text
Projekt- & Task-Mapping speichern
```

erzeugt.

---

# 7. Projekte und Tasks aktualisieren

Node-RED ruft die Projekte über folgende Hakuna API ab:

```text
GET /api/v1/projects
```

Nur aktive Projekte werden übernommen.

Archivierte Projekte werden ignoriert.

Beim Laden werden gleichzeitig die verfügbaren Tasks verarbeitet.

Der Flow läuft:

```text
beim Start von Node-RED
```

und zusätzlich:

```text
täglich um 12:00 Uhr
```

Die Aktualisierung kann im Node-RED-Editor über den Inject-Node

```text
Lade Projekte
```

auch manuell ausgelöst werden.

---

# 8. Node-RED API für Companion

## Projektliste

```text
GET /projects-list
```

Antwort:

```json
[
  "Projekt A",
  "Projekt B",
  "Projekt C"
]
```

---

## Taskliste

```text
GET /tasks
```

Antwort:

```json
[
  "Installation",
  "Programmierung",
  "Service"
]
```

---

## Einzelner Projektbutton

```text
GET /companion/button/projects/:index
```

Beispiel:

```text
GET /companion/button/projects/1
```

liefert das erste Projekt alphabetisch.

```text
GET /companion/button/projects/10
```

liefert das zehnte Projekt.

---

## Einzelner Taskbutton

```text
GET /companion/button/tasks/:index
```

Beispiel:

```text
GET /companion/button/tasks/13
```

liefert Task Nummer 13.

Die Sortierung erfolgt alphabetisch.

---

# 9. Companion Auswahl

## Projekt auswählen

```text
POST /companion/select-project
```

Unterstützt:

```json
{
  "index": 1
}
```

oder:

```json
{
  "name": "Chalet Kudu"
}
```

---

## Task auswählen

```text
POST /companion/select-task
```

Unterstützt:

```json
{
  "index": 1
}
```

oder:

```json
{
  "name": "Installation"
}
```

---

## Aktuelle Auswahl

```text
GET /companion/selection
```

Beispiel:

```json
{
  "project": "Chalet Kudu",
  "task": "Installation"
}
```

Die Auswahl wird innerhalb des Node-RED-Flows unter

```text
companion_selection
```

gespeichert.

---

# 10. Zeitrapport senden

Direkter Endpoint:

```text
POST /zeitrapport
```

Payload:

```json
{
  "Project": "Chalet Kudu",
  "task": "Installation",
  "Time": "15"
}
```

`Time` wird in Minuten angegeben.

---

## Companion Commit Endpoint

Zusätzlich existiert:

```text
POST /companion/commit
```

Dieser verwendet das zuvor ausgewählte Projekt und den Task.

Companion muss dadurch beim Senden nur noch die Dauer übergeben.

Beispiel:

```json
{
  "minutes": 15
}
```

Node-RED erstellt daraus intern:

```json
{
  "Project": "Chalet Kudu",
  "task": "Installation",
  "Time": "15"
}
```

und sendet dies intern an:

```text
http://127.0.0.1:1880/zeitrapport
```

---

# 11. Rückmeldung an Companion

Nach erfolgreichem Hakuna-POST antwortet Node-RED:

```text
✔️ OK
```

Bei einem Fehler:

```text
⚠️ Fehler
```

Diese Rückmeldung kann in Companion verwendet werden, um beispielsweise den Send-Button visuell als erfolgreich oder fehlerhaft darzustellen.

---

# 12. Companion Custom Variables

Im aktuellen Aufbau werden unter anderem folgende Custom Variables verwendet:

```text
$(custom:Project_FB)
$(custom:Task_FB)
$(custom:Time_FB)
```

Zusätzlich werden Variablen für die dynamischen Projekt- und Taskbuttons verwendet.

Beispiel:

```text
$(custom:Task_1)
$(custom:Task_2)
...
$(custom:Task_15)
```

Die Anzahl kann bei zusätzlichen Tasks erweitert werden.

Die entsprechenden Companion-Trigger müssen ebenfalls für die zusätzlichen Indizes vorhanden sein.

---

# 13. API Token

Der Hakuna API Token wird NICHT direkt im Flow gespeichert.

Im Node-RED-Tab `Timetracker` existiert die Credential-Umgebungsvariable:

```text
API_Token
```

Verwendung innerhalb der HTTP-Nodes:

```text
$(API_Token)
```

Header:

```text
X-Auth-Token: $(API_Token)
```

## API Token ändern

Node-RED öffnen:

```text
http://timetracker.local:1880
```

Dann:

```text
Timetracker Tab
→ Tab bearbeiten
→ Umgebungsvariablen
→ API_Token
```

Neuen Token eintragen und deployen.

Der Token darf niemals in GitHub committed werden.

---

# 14. Node-RED Backup

Produktive Node-RED-Dateien:

```text
/home/admin/.node-red/
```

Repository:

```text
/home/admin/timetracker/node-red/
```

Aktuellen Stand kopieren:

```bash
cp ~/.node-red/flows.json ~/timetracker/node-red/
cp ~/.node-red/package.json ~/timetracker/node-red/
cp ~/.node-red/package-lock.json ~/timetracker/node-red/
```

Danach:

```bash
cd ~/timetracker

git add .
git commit -m "Update Node-RED configuration"
git push
```

---

# 15. Companion Backup

Companion speichert seine laufende Datenbank unter:

```text
/home/companion/.config/companion-nodejs/v4.0/db.sqlite
```

Für das GitHub-Projekt wird jedoch bevorzugt der offizielle Companion-Konfigurationsexport verwendet.

Dieser wird über das Companion-Webinterface erstellt:

```text
http://timetracker.local:8000
```

Die exportierte `.companionconfig` Datei wird anschliessend unter

```text
/home/admin/timetracker/companion/
```

gespeichert.

Danach:

```bash
cd ~/timetracker

git add companion/
git commit -m "Update Companion configuration backup"
git push
```

---

# 16. Wiederherstellung

## Node-RED

Repository klonen:

```bash
git clone <REPOSITORY>
```

Die Dateien aus

```text
node-red/
```

in das Node-RED User Directory übernehmen.

Danach müssen lokale Credentials wie der Hakuna API Token wieder eingerichtet werden.

---

## Companion

Companion installieren und starten.

Danach:

```text
Companion Webinterface
→ Import / Restore
→ .companionconfig auswählen
```

Nach dem Import kontrollieren:

- Connections
- Custom Variables
- Triggers
- Pages
- Buttons
- Feedbacks
- Node-RED URLs

---

# 17. Wichtige Ports

| Dienst | Port |
|---|---:|
| Node-RED | 1880 |
| Bitfocus Companion | 8000 |

Lokale Adressen:

```text
Node-RED:
http://timetracker.local:1880

Companion:
http://timetracker.local:8000
```

---

# 18. Fehlerdiagnose

## Projekt oder Task fehlt

Zuerst prüfen:

```text
http://timetracker.local:1880/projects-list
```

und:

```text
http://timetracker.local:1880/tasks
```

Danach beispielsweise:

```text
http://timetracker.local:1880/companion/button/tasks/13
```

Wenn Node-RED den Task korrekt zurückgibt, liegt das Problem wahrscheinlich auf der Companion-Seite.

Prüfen:

- Custom Variable vorhanden?
- Trigger vorhanden?
- korrekter Index?
- korrekte Node-RED URL?
- Trigger Action vorhanden?

---

## Hakuna-Eintrag schlägt fehl

Node-RED Debug prüfen.

Insbesondere:

```text
Empfange Companion
Zeitblock & Mapping
Commit IN
Commit OUT
```

Kontrollieren:

- Project vorhanden
- Task vorhanden
- Time gültige Zahl
- Mapping vorhanden
- API Token gültig
- Hakuna API erreichbar

---

# 19. Sicherheit

Folgende Daten gehören NICHT ins Repository:

- Hakuna API Token
- Passwörter
- GitHub Personal Access Tokens
- Node-RED Credential-Dateien
- sonstige Secrets

Insbesondere nicht committen:

```text
flows_cred.json
```

Die `.gitignore` muss sensible Dateien ausschliessen.

Vor einem Push kann zusätzlich geprüft werden:

```bash
grep -RniE 'API[_-]?Token|X-Auth-Token|Authorization|Bearer|password|secret|token' node-red/
```

Ein Treffer wie

```text
$(API_Token)
```

ist in Ordnung, da dies nur die Referenz auf die Umgebungsvariable ist.

---

# 20. Git Workflow

Aktuellen Status prüfen:

```bash
cd ~/timetracker
git status
```

Änderungen hinzufügen:

```bash
git add .
```

Commit:

```bash
git commit -m "Beschreibung der Änderung"
```

Push:

```bash
git push
```

Historie anzeigen:

```bash
git log --oneline --decorate -10
```

---

# 21. Aktueller Projektstatus

Funktionsfähig:

- Hakuna API Authentifizierung
- Projekte aus Hakuna laden
- Tasks aus Hakuna laden
- aktive Projekte filtern
- Projekt-Mapping
- Task-Mapping
- Companion Projektbuttons
- Companion Taskbuttons
- Projekt auswählen
- Task auswählen
- Dauer auswählen
- Zeitrapport senden
- letzten Zeiteintrag des Tages berücksichtigen
- neuen Zeitblock berechnen
- Hakuna Zeiteintrag erstellen
- OK/Fehler-Rückmeldung an Companion
- Node-RED Backup über Git
- Companion Konfigurationsbackup über Git

## Noch zu verbessern

- Companion Trigger für zusätzliche Projekte/Tasks kontrollieren
- Backup-Prozess automatisieren
- alte Companion-Backups verwalten
- Restore-Prozess testen
- Fehlerausgaben von Hakuna detaillierter an Companion weitergeben
- optional Health-Check für Node-RED/Hakuna ergänzen
