# ScooterLock

Browserbasierte Web-App zum Ver-/Entriegeln eines **privat gekauften
myTIER GO** E-Scooters (Okai-Hardware, verkauft von TIER Mobility GmbH)
per Bluetooth LE — ohne Abhängigkeit von der offiziellen myTIER-App, deren
Weiterentwicklung TIER eingestellt hat.

Läuft komplett im Browser über die [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) —
eine einzige `index.html`, kein Build-Schritt, keine App-Store-Installation
nötig.

## Warum

TIER hat die myTIER-App nicht mehr weiterentwickelt. Ohne funktionierende
App werden privat gekaufte myTIER-GO-Scooter faktisch unbrauchbar. Diese
Seite bildet die zum Ver-/Entriegeln nötigen BLE-Kommandos nach, damit
Besitzer ihr eigenes Gerät weiter nutzen können.

**Nicht gedacht für:** TIER-Sharing-/Mietscooter. Die Seite spricht nur das
lokale BLE-Schloss an; Miet-Scooter erfordern ohnehin eine gültige,
serverseitig ausgestellte Fahrtfreigabe über TIERs Backend, die hier nicht
nachgebildet wird.

## Protokoll

Reverse engineered aus der offiziellen App, abgeglichen mit dem
Open-Source-Projekt [OpenTIER](https://github.com/andreknieriem/opentier)
(MIT-Lizenz), das dieselbe Lücke schließt.

- **BLE Service UUID:** `00002C00-0000-1000-8000-00805f9b34fb`
- **Write Characteristic:** `00002C01-0000-1000-8000-00805f9b34fb`
- **Notify Characteristic:** `00002C10-0000-1000-8000-00805f9b34fb`
- **Kommandos** (ASCII, `\r\n`-terminiert, in 20-Byte-Chunks geschrieben):
  - Entriegeln: `AT+BKSCT=<passwort>,0$`
  - Verriegeln: `AT+BKSCT=<passwort>,1$`
  - Status: `AT+BKINF=<passwort>,0$`

Das Passwort ist dasselbe, das dem Scooter beim Kauf beilag (Aufkleber/Box)
und in der offiziellen App beim Erstpairing eingegeben wurde. Es wird bei
jedem Kommando im Klartext mitgeschickt — es gibt keine
Challenge-Response- oder Verschlüsselungsschicht.

**Statusantwort (`+ACK:BKINF,...$`)** — Feldreihenfolge bestätigt durch
Live-Mitschnitte im [myTIER-Forum](https://mytier-forum.de/community/topic/akkustand-via-bluetooth-auslesen/page/3/)
(z. B. `+ACK:BKINF,0,21.3,0.00,1634.4,9,57,1$`):

| Index | Bedeutung | Beispielwert |
|---|---|---|
| 0 | unklar (evtl. Licht-Flag, unbestätigt — s. u.) | `0` |
| 1 | Geschwindigkeit (km/h) | `21.3` |
| 2 | Trip-km seit Entsperren | `0.00` |
| 3 | Gesamtkilometer (Tacho) | `1634.4` |
| 4 | Zähler, +1 pro Poll während der Fahrt — Zweck unklar | `9` |
| 5 | Akkustand (%) | `57` |
| 6 | Sperrstatus: `0` = gesperrt, `1` = entsperrt | `1` |

Achtung, Falle: Die Polarität von Index 6 ist **umgekehrt** zum `BKSCT`-
Kommando/-ACK (dort ist `0` = entriegelt, `1` = verriegelt). Eine frühere
Version dieser App hat außerdem die falschen Indizes für Akku/Sperrstatus
gelesen (Annahme einer variablen Feldreihenfolge, die sich nicht bestätigt
hat) — das ist inzwischen anhand der obigen Tabelle korrigiert.

Nur Index 5 und 6 werden aktuell in der UI angezeigt; Geschwindigkeit,
Trip- und Gesamt-km liegen im Rohtext vor, werden aber nicht ausgewertet.

Diese App (`index.html`) beschränkt sich bewusst auf `BKSCT`/`BKINF` —
das ist der Stand, der bereits genutzt wird und zuverlässig funktioniert.

## Test-Labor (experimentell)

Unter [`test/`](test/index.html) liegt eine separate Konsole für alles,
was (noch) nicht bestätigt ist. Verbindung + abonnierte Kanäle wie hier,
aber mit frei editierbaren Rohbefehlen, wählbarem Ziel-Kanal und einem
Live-Log statt einer fertigen UI — zum Anklicken und Ausprobieren am
eigenen Scooter, nicht zum täglichen Ver-/Entriegeln.

**Licht:** Ein `AT+BKLED=<passwort>,...$`-Kommando wird in einem
[myTIER-Forum-Post](https://mytier-forum.de/community/reply/3768/) aus dem
dekompilierten App-Code erwähnt. Zusätzliche Bestätigung im selben Thread:
das dekompilierte `VehicleStatus`-Objekt der offiziellen App hat ein Feld
`isHeadLampTurnedOn` (boolean) — Lichtsteuerung ist also real, nur der
genaue Bluetooth-Befehl dafür bleibt unbestätigt. Weder OpenTIER noch der
Stand, auf dem dieses Projekt sonst basiert, hatten `BKLED` implementiert.
Das Labor hat "Licht an/aus"-Presets, die `AT+BKLED=<passwort>,1$` bzw.
`,0$` nach dem `BKSCT`-Muster senden — **ob `1`/`0` wirklich an/aus
bedeuten oder der Befehl überhaupt etwas tut, ist offen.**

**Unbekannte Kanäle:** Ein Nutzer hat den Scooter per ESP32 (Standard-AT-
BLE-Firmware) gescannt und unter dem Service `0x2C00` fünf Characteristics
gefunden: `0x2C01` (Write, von dieser App genutzt), `0x2C10` (Notify,
genutzt) — sowie **`0x2C02`, `0x2C03`, `0x2C04`**, die bisher niemand
angesprochen hat, aber laut Scan dieselbe Write+Notify-Fähigkeit haben.
Das Labor entdeckt beim Verbinden automatisch alle Characteristics des
Service und lässt jede davon als Sendeziel wählen — falls Licht (oder
etwas anderes) über einen dieser drei Kanäle statt über `0x2C01` läuft,
lässt sich das dort ausprobieren.

Weitere AT+BK-Befehle (z. B. für Klingel, Geschwindigkeitsbegrenzung oder
Passwortänderung) konnten bei der Recherche nicht gefunden werden — im
Labor lässt sich mit freiem Text trotzdem danach suchen.

## iPhone

**Safari unterstützt kein Web Bluetooth** (Apple-Entscheidung, gilt wegen
WebKit-Zwang für alle iOS-Browser inkl. Chrome/Edge auf iOS). Für iPhone/iPad
brauchst du einen der wenigen Browser, die ihren eigenen BLE-Stack mitbringen:

- [Bluefy – Web BLE Browser](https://apps.apple.com/app/bluefy-web-ble-browser/id1492822055) (kostenlos)
- WebBLE (Alternative im App Store)

Seite darin öffnen, verbinden, fertig — kein Xcode, kein Mac nötig.

## Android / Desktop

Chrome oder Edge unterstützen Web Bluetooth nativ, keine zusätzliche App nötig.

## Hosting

Web Bluetooth verlangt einen "secure context" — die Seite muss über **HTTPS**
(oder `localhost`) geladen werden, ein simples `python3 -m http.server` im
lokalen Netz reicht nicht. Diese Seite läuft über GitHub Pages dieses Repos
unter `https://<user>.github.io/Projects-shared/scooter-unlock/`.

## Nutzung

1. Seite öffnen (Chrome/Edge auf Android/Desktop, Bluefy auf iOS).
2. Scooter-Passwort eintragen (steht auf Aufkleber/Box).
3. "Mit Scooter verbinden" tippen und den eigenen Scooter aus dem
   Bluetooth-Auswahldialog wählen.
4. Ver-/Entriegeln über den großen Button.

Das Passwort wird lokal im Browser (`localStorage`) gespeichert, nicht
irgendwo hochgeladen.

## Lizenz

MIT. Protokoll-Kenntnis basiert auf der Vorarbeit des OpenTIER-Projekts.
