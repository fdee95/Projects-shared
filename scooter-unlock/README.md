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

Die Statusantwort (`AT+BKINF`) ist eine kommagetrennte `$...$`-Zeile;
je nach Firmware-Revision steht das Lock-Flag am Anfang oder am Ende der
Felderliste, daher wertet der Parser beide Layouts aus.

### Licht (experimentell, Beta)

Im [myTIER-Forum](https://mytier-forum.de/community/topic/akkustand-via-bluetooth-auslesen/page/3/)
wird zusätzlich ein `AT+BKLED=<passwort>,...$`-Kommando aus dem
dekompilierten App-Code genannt, das die LED/das Licht ansprechen soll
([konkreter Post](https://mytier-forum.de/community/reply/3768/)). Weder
OpenTIER noch der offizielle App-Reverse-Engineering-Stand, auf dem dieses
Projekt basiert, hatten das bisher implementiert — die Quelle ist ein
einzelner Forumsbeitrag, nicht verifiziert.

Diese App sendet testweise `AT+BKLED=<passwort>,1$` (an) /
`AT+BKLED=<passwort>,0$` (aus), nach dem Muster von `BKSCT`. **Ob `1`/`0`
wirklich an/aus bedeuten (oder überhaupt etwas tun), ist nicht bestätigt** —
einmal am eigenen Scooter ausprobieren. Antwortet der Scooter mit
`+ACK:BKLED,0` / `+ACK:BKLED,1`, übernimmt die App das als bestätigten
Zustand; ohne Antwort bleibt es bei der optimistischen lokalen Anzeige.

Weitere AT+BK-Befehle (z. B. für Klingel, Geschwindigkeitsbegrenzung oder
Passwortänderung) konnten bei der Recherche nicht gefunden werden — falls
jemand mehr aus dem dekompilierten App-Code oder per nRF Connect
mitschneidet, gerne im Forum-Thread nachlesen/ergänzen.

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
