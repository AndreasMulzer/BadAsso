# VESC / Torqeedo – Node-RED-Flow

[VescTorqueedo.json](VescTorqueedo.json) steuert den Tronic 250R mit
VESC-Firmware 6.02 über den VESC Express und **CAN-ID 29**. Der Flow bietet
manuelle Vorwärts-/Rückwärtsfahrt, einen SOG-PID-Tempomaten für **0–4 kn**,
aktiven Hard-STOP und die Rückmeldung der tatsächlich aktiven Steuerquelle.

## Dashboard und Bedienung

| Bedienelement | Funktion |
| --- | --- |
| **ein/aus** | Gibt Fahrbefehle aus Node-RED frei. Ausschalten beendet den Tempomaten und fordert Hard-STOP an. |
| **Fahrt (%)** | Manueller Drehzahlsollwert von −100 bis +100 %. Positive Werte fahren vorwärts, negative rückwärts. Eine Betätigung beendet den Tempomaten; Nullstellung fordert Hard-STOP an. |
| **STOP (HARD)** | Beendet den Tempomaten und fordert aktive Motorbremsung an, auch bei ausgeschalteter Node-RED-Freigabe. |
| **Tempomat EIN/AUS** | Schaltet die automatische Geschwindigkeitsregelung ein oder aus. Ausschalten fordert Hard-STOP an. |
| **Geschwindigkeit** | Sollgeschwindigkeit von 0 bis 4 kn in Schritten von 0,1 kn. |
| **Reglerparameter** | Kp, Ki, Kd und SOG-Mittelwert-Fenster bearbeiten und mit **Übernehmen & speichern** gemeinsam anwenden. **Verwerfen** lädt die aktuell verwendeten Werte. |
| **Tempomat-Status** | Zeigt Sollgeschwindigkeit, SOG, Stellwert, P-/I-/D-Anteile und Statusgrund. |
| **Motor / Steuerquelle** | Zeigt den vom VESC angewendeten Modus, die gemessene Drehrichtung und die aktive Quelle Poti/WiFi. |
| **Rpm / Temp FET** | Zeigt Propellerdrehzahl und Temperatur der Leistungselektronik. |

Die Prozentwerte sind **Drehzahlsollwerte**, keine gemessene Motorleistung.
±100 % entsprechen im LispBM-Script ±75.000 ERPM. Die Node-RED-Freigabe
betrifft Handregler und Tempomat; das lokale Potentiometer bleibt eine
eigenständige Steuerquelle. Rekuperations-, Brake- und Handbrake-Bedienelemente
sind entfernt.

## PID-Tempomat

Der Tempomat verwendet `navigation.speedOverGround` aus `vessels.self`.
Signal K liefert m/s; der Regler rechnet für Bedienung und Regelung in kn um.
Es arbeitet ein durchgehender PID-Regler mit einem Ausgang von **0 bis 100 %**:

```text
Fehler = Sollgeschwindigkeit − gefilterte SOG
P = Kp × Fehler
I = I + Ki × Fehler × dt                 [mit Anti-Windup]
D = −Kd × gefilterte SOG-Änderung pro Sekunde
Drehzahlsollwert = begrenze(P + I + D, 0 %, 100 %)
```

| Parameter | Wert |
| --- | ---: |
| Regeltakt | 200 ms |
| Kp | 100 Prozentpunkte/kn |
| Ki | 5 Prozentpunkte/(kn·s) |
| Kd | 10 Prozentpunkte·s/kn |
| SOG-Mittelwert | letzte 10 gültige GPS-Messwerte, einstellbar |
| D-Filter | 0,5 s, nur für die SOG-Änderungsrate im D-Anteil |

Beim Einschalten starten I und D bei null; der P-Anteil wirkt sofort.
Mit diesen Werten führen beispielsweise **1 kn positive Abweichung sofort zu
100 %**. Während der Fahrt verändern I und D den Gesamtstellwert. Es gibt
keine zusätzliche Vollgasstufe und keine Stellwertrampe. Der D-Anteil nutzt
neue SOG-Messungen mit Quellzeitstempel; Sollwertänderungen erzeugen keinen
D-Sprung. Anti-Windup verhindert weiteres Aufintegrieren in Richtung eines
bereits gesättigten Ausgangs.

Zum Starten Motor freigeben, frische SOG und VESC-Rückmeldungen abwarten,
Sollgeschwindigkeit wählen und den Tempomaten einschalten. Rückwärtsfahrt
bleibt manuell. Bei **0 kn Sollwert** bleibt der Tempomat eingeschaltet und
fordert Hard-STOP an; ein positiver Sollwert lässt ihn wieder regeln. Ein
vom PID berechneter Ausgang von 0 % bei positivem Sollwert ist dagegen ein
normaler Drehzahlsollwert, kein gesonderter Hard-STOP-Befehl.

### SOG glätten

Der PID verwendet einen **gleitenden arithmetischen Mittelwert über die letzten
10 gültigen SOG-Messwerte**. Der bisherige SOG-Tiefpass ist ersetzt. Jeder neue
Quellzeitstempel zählt einmal; Regelticks, doppelte und ältere Messungen füllen
das Fenster nicht. Bei vollem Fenster fällt der älteste Wert heraus. Beim
Anlaufen wird nur durch die Anzahl der vorhandenen Werte geteilt; es werden
keine Nullen ergänzt. Auch das Einschalten des PID verwendet diesen Mittelwert.

Im Dashboard unter **Motor → Reglerparameter** lässt sich die Fenstergröße über
**SOG-Mittelwert [Werte]** auf **1–100** einstellen und speichern. Bei kleinerem
Fenster bleiben die neuesten Werte, ein größeres Fenster füllt sich mit neuen
Messungen. Ein Fensterwechsel behält das Integral und setzt nur die D-Steigung
zurück, damit kein künstlicher D-Stoß entsteht.

Zehn Werte entsprechen bei 1 Hz ungefähr zehn Sekunden und bei 5 Hz ungefähr
zwei Sekunden. Das verzögert auch echte Geschwindigkeitsänderungen; Fenster
und PID-Verstärkungen sind deshalb gemeinsam auf dem Boot abzustimmen. Die
bisherigen Verstärkungen werden beim Update nicht verändert.

Ungültige SOG oder Quell-/Empfangslücken über 3 s löschen die Historie.
Die Frischeprüfung gilt für die neueste Messung, nicht für jeden einzelnen
Wert im Fenster. `navigation.speedOverGround` bleibt die rohe Sensorquelle.
Statusanzeige und `controller.cruise.sogFiltered` zeigen den Mittelwert;
die rohe SOG und die Anzahl der Fensterwerte werden getrennt angezeigt.
Der vorhandene 0,5-s-Filter glättet nur noch den D-Anteil, nicht die SOG.

### Reglerparameter anpassen

Im Dashboard unter **Motor → Reglerparameter** stehen **Kp, Ki, Kd und die SOG-Fenstergröße** als
editierbare Zahlenfelder. Die aktiven Werte werden zusätzlich unter den
Feldern angezeigt. Änderungen wirken erst nach **Übernehmen & speichern**;
**Verwerfen** setzt den Eingabeentwurf zurück. Rückmeldungen überschreiben
keine ungespeicherten Eingaben.

Die Startwerte sind **Kp=100, Ki=5 und Kd=10**. Einstellbar sind Kp/Kd von
0 bis 10.000 und Ki von 0 bis 1.000; die Einheiten stehen an den Feldern.
Für Kp/Ki/Kd sind Nullwerte zulässig; die SOG-Fenstergröße muss ganzzahlig zwischen 1 und 100 liegen. Die Grenzen sind Eingabegrenzen, keine am Boot
abgestimmten Empfehlungen.

Nach erfolgreichem Speichern werden die Werte im Datei-Kontext
`vescCruiseConfig.pid` beziehungsweise `vescCruiseConfig.sogWindowSize` abgelegt und nach einem Neustart oder Deploy erneut
geladen. Bei laufendem PID gelten Änderungen sofort; Integral und
bei reinen Kp-/Ki-/Kd-Änderungen auch der D-Filterzustand bleiben erhalten. Speichern schaltet den Tempomaten
nicht ein und hebt keine Potiübernahme auf.

Die separate D-Filterzeitkonstante bleibt im Block `CFG` des Function-Knotens
**Tempomat + Handbetrieb (PID, kn)** einstellbar.

## Hard-STOP und Poti-/WiFi-Übernahme

Der Hard-STOP ist im zugehörigen LispBM-Script umgesetzt: Der Motorloop
fordert mit `set-brake-rel 1.0` **50-mal pro Sekunde** den maximal zulässigen
relativen Bremsstrom an. Die Strom-, Batterie- und Temperaturgrenzen der
Firmware bleiben wirksam. Ein neuer ausdrücklicher Fahrbefehl löst den
Hard-STOP wieder.

Das Potentiometer übernimmt nach einer Bewegung von **mehr als 100 mV über
fünf Motorzyklen, also 100 ms**. Das gilt auch während Hard-STOP. Bei einer
Potiübernahme beendet Node-RED den Tempomaten ohne Gegenbefehl. Periodische
PID-Folgebefehle können die Kontrolle danach nicht zurückholen; eine erneute
WiFi-Übernahme benötigt eine ausdrückliche Bedienung im Dashboard.

Die angezeigte Quelle stammt aus der **VESC-Rückmeldung nach Anwendung des
Motorbefehls**, nicht aus dem zuletzt gesendeten Node-RED-Befehl. Beim Bremsen
können daher gleichzeitig „Harter STOP aktiv“, „Motor dreht vorwärts“ und
„Steuerquelle: Steuerung (WiFi)“ angezeigt werden. Die gemessene Drehrichtung
bleibt vom angewendeten Steuerungsmodus getrennt.

## Signal-K-Statusmeldungen

Alle folgenden Pfade beginnen mit **`propulsion.torqeedo.`**:

| Pfad | Bedeutung / Werte |
| --- | --- |
| `controller.activeSource` | Tatsächliche Steuerquelle: `poti`, `wifi`, `unknown`. |
| `controller.activeMode` | Angewendeter Modus: `forward`, `reverse`, `stop`, `hardStop`, `unknown`. `stop` bedeutet normalen Null-Drehzahlsollwert. |
| `controller.hardStopActive` | Rückgemeldeter Hard-STOP-Modus; `null` bei fehlender Rückmeldung. |
| `controller.controlStatus` | Lesbarer Statustext mit Modus, Motorbewegung und Quelle. |
| `controller.feedbackAvailable` | Steuerungsrückmeldung vorhanden und höchstens 1,5 s alt. |
| `controller.appliedPercent` | Vom VESC rückgemeldeter Drehzahlsollwert als Verhältnis −1 bis +1. |
| `controller.command` / `controller.commandPercent` | Zuletzt gesendete Anforderung und Sollwert; keine Bestätigung der Ausführung. Der numerische Sollwert ist ein Verhältnis −1 bis +1. |
| `drive.state` | Gemessener Zustand: `forward`, `reverse`, `stopped`, `fault`, `unknown`; Stillstandsbereich ±100 ERPM. |
| `drive.available` / `controller.available` | Motortelemetrie vorhanden und höchstens 2 s alt. |
| `state` | Standard-Signal-K-Zustand: `started`, `stopped`, `unusable`, bei unbekanntem Zustand `null`. |
| `transmission.gear` | Richtung aus angewendetem Befehl: `Forward`, `Reverse`, `Neutral`, bei Fehler `Fault`; ohne Rückmeldung `null`. |
| `controller.cruise.enabled` | Tempomat eingeschaltet. |
| `controller.cruise.targetSpeed` / `controller.cruise.sog` | Sollgeschwindigkeit und aktuelle SOG in m/s. |
| `controller.cruise.sogFiltered` | Vom PID verwendeter SOG-Mittelwert in m/s; bei alten Daten `null`. |
| `controller.cruise.sogWindowSize` / `controller.cruise.sogSampleCount` | Eingestellte Fenstergröße und Anzahl aktuell gültiger Messwerte im Fenster. |
| `controller.cruise.output` / `p` / `i` / `d` | Ausgang und PID-Anteile als Verhältnis; die letzten drei vollständigen Pfade sind `controller.cruise.p`, `.i` und `.d`. |
| `controller.cruise.reason` | Status- oder Abschaltgrund des Tempomaten. |

Zusätzlich werden Drehzahlen, Spannung, Ströme, Leistung, Temperaturen und
VESC-Fehlercodes veröffentlicht. Die Umrechnung verwendet **7 Polpaare und
14:1 Getriebeuntersetzung**: `drive.revolutions` ist die Propellerdrehzahl in Hz,
Temperaturen werden in K und Leistungen in W übertragen. Änderungen werden
direkt veröffentlicht; ein vollständiger **Heartbeat jede Sekunde** erneuert
auch die Zeitstempel gleichbleibender Statuswerte. Veraltete Zustände werden
als unbekannt gemeldet, statt daraus Stillstand abzuleiten.

## Datenfehler und Neustart

Ein aktiver Tempomat wird mit Hard-STOP-Anforderung beendet bei:

- fehlender, ungültiger oder mehr als **3 s** alter SOG;
- fehlender/veralteter Motortelemetrie, spätestens nach **2,5 s**, oder VESC-Fehlern;
- fehlender oder mehr als **1,5 s** alter Steuerungsrückmeldung;
- einer Regeltaktlücke von mehr als **1,5 s**.

Neue Daten schalten den Tempomaten nicht automatisch wieder ein.
Nach Start/Deploy fordert Node-RED einmalig Hard-STOP an und der Tempomat
bleibt aus. Sollgeschwindigkeit, PID-Parameter, SOG-Fenstergröße und Motorfreigabe werden
persistent gespeichert; der aktive Tempomat und sein Integrator nicht.

Das LispBM-Script hält bei WiFi-Verbindungsabbruch den zuletzt angewendeten
Befehl. Ein aus Node-RED gesendeter Stopp erreicht den VESC nur bei bestehender
Verbindung. Ein allgemeiner zusätzlicher Kommunikations-Watchdog ist nicht
implementiert.

## Weitere erhaltene Funktionen

Die FET-Lüftersteuerung prüft alle **10 s** die Reglertemperatur. Oberhalb
von **45 °C** sendet sie für Kanal **16** den Sollwert `Temperatur × 1,5`,
sonst 0; `fadeMs` ist 0. Die Nachricht läuft über den vorhandenen externen
Lüfter-Link. Auch die Test-Injects für STOP und 30 % Fahrt verwenden die
Motorfreigabe beziehungsweise die gemeinsame Steuerungslogik.

## Voraussetzungen und Installation

- Das passende [LispBM-Script auf CAN-ID 29](https://github.com/AndreasMulzer/Tronic250Vesc/blob/158e0ef7148bc52e55932e95a71c35036aa9448c/LispBM/tronic250r.lisp) auf dem Tronic, nicht auf dem VESC Express.
- Eine in Signal K eingebettete Node-RED-Instanz mit `node-red-dashboard` und `@signalk/node-red-embedded`.
- Die Kontext-Stores `memory` und `file`, zum Beispiel:

```js
contextStorage: {
    default: "memory",
    memory: { module: "memory" },
    file: { module: "localfilesystem" }
}
```

Das TCP-Ziel ist im Flow mit **192.168.0.195:65102** vorbelegt. Bei mehreren
GPS-Quellen im SOG-Subscribe-Knoten eine passende Quelle wählen. Die bestehenden
Dashboard-Gruppen und der externe Lüfter-Link `d884558431ecd46d` müssen im
Gesamtflow vorhanden beziehungsweise zugeordnet sein.

Zur Installation Poti auf Neutral stellen, den bisherigen VESC-Tab deaktivieren,
das passende LispBM-Script installieren und den bisherigen VESC-Tab durch
[VescTorqueedo.json](VescTorqueedo.json) ersetzen. Nach Deploy Rückmeldungen
abwarten und die Node-RED-Motorfreigabe bei Bedarf einschalten.

Der Flow entspricht unverändert dem Stand
[Tronic250Vesc, Commit 158e0ef](https://github.com/AndreasMulzer/Tronic250Vesc/tree/158e0ef7148bc52e55932e95a71c35036aa9448c).
Weitere Details und Quellen stehen in der
[zugehörigen Installationsanleitung](https://github.com/AndreasMulzer/Tronic250Vesc/blob/158e0ef7148bc52e55932e95a71c35036aa9448c/docs/installation.md).
Die lokalen Softwareprüfungen belegen keine Installation, Bremskraft oder
Regelgüte auf dem Boot.
