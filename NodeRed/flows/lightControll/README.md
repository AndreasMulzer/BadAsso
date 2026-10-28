# Node-RED Lichtsteuerung (BadAsso)

Stand: 28.10.2026 · Flow: `lightControll.json`

## Überblick

Der Flow steuert die Lichtausgänge des Schiffs über ESP32-HSS-/LSS-Ausgangsboards. Er enthält 7 Lichtgruppen, ein Node-RED-Dashboard, eine Sammelabschaltung und eine automatische bzw. manuelle RGB-Nachtumschaltung. Die ESP-Ansteuerung erfolgt über die bestehenden Link-Nodes mit Nachrichten des Typs `set_multi`.

## Lichtgruppen

| Gruppe (`msg.area`) | Anzeige | Funktionen |
| --- | --- | --- |
| `innen` | Innen | Toggle, Helligkeit, RGB-Farbe, Nachtmodus |
| `aussen` | Außen | Toggle, Helligkeit, RGB-Farbe, Nachtmodus |
| `position` | Positionslichter | Toggle, Helligkeit |
| `achtern` | Achterlicht | Toggle, Helligkeit |
| `top` | Toplicht | Toggle, Helligkeit |
| `mast` | Mast | Toggle, Helligkeit |
| `unterwasser` | UW | Toggle, Helligkeit |

Die Kanalbelegung, maximalen Helligkeiten, Auto-Helligkeiten, Ein-/Ausschaltverzögerungen und regulären Fade-Zeiten stehen in der persistenten Lichtkonfiguration.

## Bedienung und Befehle

Die sieben Lichtsteuerungsfunktionen verwenden `msg.area` und `msg.topic`:

| Topic | Bedeutung |
| --- | --- |
| `toggle` | Gruppe je nach gespeichertem Zustand ein-/ausschalten |
| `slider` | Helligkeit über `msg.payload` (0–100 %) |
| `color` / `colour` / `rgb` | RGB-Werte über `msg.payload.r/g/b` (0–100 nach Umrechnung im Flow) |
| `alloff` | Alle Gruppen regulär ausschalten (Dashboard „ALLES AUS“) |
| `off` | Gruppe ausschalten |
| `refresh` | Status-LED aus Context aktualisieren, ohne ESP-Schaltbefehl |
| `night_apply` | Nachtmaske auf `innen` und `aussen` anwenden |

ESP-Befehle haben dieses Format:

```json
{
  "type": "set_multi",
  "items": [
    { "channel": 23, "value": 0, "fadeMs": 60000 }
  ]
}
```

Die Kanalnummern beziehen sich auf die bestehende Ausgangsboard-Zuordnung. Reguläre Ein-/Ausschaltverzögerungen aus der Konfiguration bleiben wirksam; ausstehende lokale Timer können bei neuen Befehlen ersetzt werden.

## Nachtmodus (nur Innen und Außen)

**Automatik:** Tag → RGB; Nacht + Hafen `outside` → nur Rot; Nacht + Hafen `inside` → RGB. Ausgeschaltete Gruppen bleiben ausgeschaltet.

| Übergang | Fade-Zeit |
| --- | ---: |
| Tag ↔ Nacht | 60 s |
| Nachts: Hafen aus-/einlaufen | 10 s |
| Manueller Moduswechsel | 1 s |

Der Dashboard-Button schaltet zyklisch `AUTO → NACHT EIN → NACHT AUS → AUTO`. `NACHT EIN` erzwingt Rot, `NACHT AUS` erzwingt RGB. Die Umschaltung gibt `night_apply` an beide RGB-Gruppen aus; vorhandene G-/B-Sollwerte werden für die spätere Rückkehr zu RGB aufbewahrt. Ohne gültige Tag-/Nacht- bzw. Hafeninformation wird im Automatikbetrieb kein neuer Zustand geraten.

## Context-Speicher (wichtig)

| Pfad | Bereich / Store | Zweck |
| --- | --- | --- |
| `flow.lights.config` | Flow / `file` | Lichtgruppen, Kanal- und Helligkeitskonfiguration |
| `flow.lights.status.channels` | Flow / `file` | Zuletzt von Node-RED **kommandierte** Kanalzielwerte |
| `flow.nightMode.dateTime.updates[0].values[0].value` | Flow / `file` | SignalK-Wert `day` / `night` |
| `flow.nightMode.mode` | Flow / `file` | `auto`, `on` oder `off` |
| `flow.nightMode.active` | Flow / `file` | Aktive Rotmodus-Maske |
| `flow.nightMode.ChannelBackupPreNightMode` | Flow / `file` | Unmaskierte Farb-/Kanal-Sollwerte zur Wiederherstellung |
| `global.BoatState.atHabour` | Global / `memory` | `inside` / `outside` (Schreibweise **atHabour** beibehalten) |

Weitere Zustandsfelder der Automatik (z. B. `signalK`, `inHarbor`, `resolved`, `lastTransition`) liegen ebenfalls unter `flow.nightMode` im Store `file`. Nachtmodus-Daten werden **nicht** in `flow.lights.nightMode` verwaltet.

**Aktuelle Einschränkung:** Die Dashboard-LEDs zeigen die zuletzt kommandierten Werte, **keine** vom ESP32 bestätigte Rückmeldung. Nach einem Neustart oder Kommunikationsfehler kann der tatsächliche Ausgang davon abweichen. Die physische Statusrückmeldung ist eine spätere Erweiterung.

## Installation / Inbetriebnahme

1. `lightControll.json` in Node-RED importieren; erforderliche Dashboard-, UI-LED- und SignalK-Nodes müssen installiert sein.
2. Bestehende Link-Verbindungen zu den HSS-/LSS-Boards prüfen.
3. Falls `flow.lights.config` im Store `file` noch nicht existiert, den Inject **„EINMALIG: Licht-Konfiguration initialisieren“** manuell auslösen. Bei vorhandener Konfiguration wird nichts überschrieben.
4. Deploy durchführen und Dashboard **Licht** aufrufen. `ui_ui_control` triggert beim Verbinden eine LED-Aktualisierung. Ein zusätzlicher manueller Refresh ist vorhanden.
5. Sicherstellen, dass SignalK `environment.mode` die Werte `day`/`night` liefert und `global.BoatState.atHabour` im Memory-Store gepflegt wird.

Der Flow enthält keine regelmäßige Neuinitialisierung der Lichtkonfiguration. Bei Änderungen an der Channel-Konfiguration den persistenten Context berücksichtigen.

## Wartung / Hinweise

- **Kein hartes globales Setzen aller Ausgangskanäle:** „ALLES AUS“ benutzt die vorhandenen Gruppensteuerungen und deren Abschaltlogik.
- Geteilte Kanalnummern in der Konfiguration beachten (z. B. Kanal 0 für Mast/Top und Kanal 15 in beiden Unterwasser-Segmenten).
- Lange Fades erfordern entsprechende Unterstützung von `fadeMs` in der ESP-Firmware.
- Vor Änderungen den Flow und die Context-Daten sichern. Node-Positionen und Link-Verbindungen sind Teil des bestehenden Projekts.
