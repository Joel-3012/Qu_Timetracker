# Qu Timetracker

Zeiterfassung über Stream Deck / Bitfocus Companion und Node-RED mit Hakuna.

## Architektur

Stream Deck
→ Bitfocus Companion
→ Node-RED
→ Hakuna API

## Komponenten

- Node-RED
- Bitfocus Companion
- Stream Deck
- Hakuna API

## Sicherheit

Der Hakuna API Token wird in Node-RED als Credential-Umgebungsvariable
`API_Token` gespeichert.

**API-Tokens und Credentials dürfen nicht in diesem Repository gespeichert werden.**

## Backup-Struktur

- `node-red/` – Node-RED Flow
- `companion/` – Bitfocus Companion Konfiguration
- `docs/` – Dokumentation
