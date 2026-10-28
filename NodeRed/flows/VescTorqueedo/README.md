# VESC / Torqeedo – Node-RED-Flow

[VescTorqueedo.json](VescTorqueedo.json) ist der vollständige Import für die
VESC-Steuerung mit SOG-PID-Tempomat (0–4 kn), Hard-STOP und Rückmeldung der
aktiven Steuerquelle Poti/WiFi.

Der Flow entspricht unverändert dem veröffentlichten Stand
[Tronic250Vesc, Commit 44b955e](https://github.com/AndreasMulzer/Tronic250Vesc/tree/44b955e0317c07226e3c87c796da0ad2cee7f71e).

Voraussetzungen:

- Das dazugehörige [LispBM-Script auf CAN-ID 29](https://github.com/AndreasMulzer/Tronic250Vesc/blob/44b955e0317c07226e3c87c796da0ad2cee7f71e/LispBM/tronic250r.lisp).
- `node-red-dashboard`, `@signalk/node-red-embedded` und die Kontext-Stores `memory` und `file`.
- Die im Gesamtflow vorhandenen Dashboard-Gruppen und der externe Lüfter-Link.

Den bestehenden VESC-Tab ersetzen. Installation, Bedienung und Signal-K-Pfade
stehen in der [zugehörigen Anleitung](https://github.com/AndreasMulzer/Tronic250Vesc/blob/44b955e0317c07226e3c87c796da0ad2cee7f71e/docs/installation.md).
