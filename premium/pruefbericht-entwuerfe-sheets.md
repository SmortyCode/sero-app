# Sheet-/Overlay-Zustand in SERO — warum die App grau und verkleinert hängen bleibt

> **Wichtiger Hinweis vorab:** `/Users/smorty/sero-app/web/sero.js` **und** `index.html` wurden **während meiner Analyse um 13:24 Uhr verändert** (vorher 12:49). Offensichtlich hat parallel ein anderer Agent genau an dieser Stelle gepatcht. Ich habe **nichts geschrieben** (strikt read-only). Alle Zeilennummern unten beziehen sich auf den **aktuellen** Stand (13:24, MD5 `caf99935cb39456678c040e971cceb89`). Wo sich der Befund durch den Patch verändert hat, steht das dabei.

---

## 1. Der Mechanismus (das ist die Antwort auf die zentrale Frage)

`recede` wird an **genau einer** Stelle gesetzt und an **zwei** Stellen entfernt:

| Datei:Zeile | Aktion |
|---|---|
| `sero.js:3764` | `$("viewApp").classList.add("recede")` — in `openSheet()` |
| `sero.js:3793` | `remove("recede")` — in `closeSheet()`, **vor** dem Frühausstieg |
| `sero.js:3812` | `remove("recede")` — im **neuen** Wächter `pruefeSchleier()` (seit 13:24) |

`sero.css:81-84`:
```css
#viewApp.recede { transform: scale(.945) translateY(10px);
  border-radius: 24px; overflow: hidden; filter: brightness(.82); }
```
→ **verkleinert (0,945) + abgedunkelt (brightness .82)** = exakt Svens Screenshot. Ein `.party`-Overlay (`sero.css:1173-1179`) dunkelt zwar auch ab, **verkleinert aber nicht** — der Screenshot zeigt also eindeutig `recede`, nicht `.party`.

**Der Klemmpunkt ist nicht `closeSheet`, sondern die Asymmetrie:**
`closeSheet()` (`sero.js:3784-3805`) entfernt `recede` in **jedem** Fall — die Zeile steht bei `3793` **vor** dem `if (sh.hidden) return;` bei `3794`. Ein Frühausstieg in `closeSheet` kann `recede` also **nicht** stehenlassen.

Das Problem ist umgekehrt: **Das Sheet kann unsichtbar werden, ohne dass `closeSheet` läuft.** Dann bleibt `recede` gesetzt, das Backdrop (`z-index:20`) ist ebenfalls `hidden`, also ist die App weiter bedienbar — nur klein und dunkel. Genau Svens Beschreibung.

Der Hebel dafür ist `sero.css:1104`:
```css
.sheet.closing { transform: translateY(105%) !important; transition: transform .26s ease-in; }
```
Das `!important` schiebt das Sheet **komplett aus dem Bild**, egal was `openSheet` danach tut. Bis 13:24 hat `openSheet` die Klasse `closing` **nicht** entfernt und der 260-ms-Timer aus `closeSheet` hat sein Opfer **nicht** geprüft. Ergebnis: **jedes `openSheet()` innerhalb von 260 ms nach einem `closeSheet()` erzeugte den Hängezustand** — Sheet weg, `recede` bleibt, für immer.

Seit 13:24 ist das an zwei Stellen entschärft:
* `sero.js:3761` — `openSheet` räumt `closing` von Sheet **und** Backdrop weg
* `sero.js:3801` — der Timer bricht ab, wenn `closing` inzwischen fehlt
* `sero.js:3810-3815` — Wächter `pruefeSchleier()` alle 1200 ms + bei `visibilitychange`

---

## 2. Gefundene Pfade, nach Wahrscheinlichkeit sortiert

### Pfad 1 — Paywall-Umleitung nach „Analysieren" (bester Erklärer, HÖCHSTE Wahrscheinlichkeit)

**Code:** `sero.js:2477` → `sero.js:309-313`
```js
// 2477 (im Save-Handler von openStagedSheet):
if (!handleScanError(e)) $("sheetErr").textContent = e.message;
// 310:
if (e && e.status === 402) { closeSheet(); openPaywall(); return true; }
```
`openPaywall()` (`sero.js:268-307`) ruft **synchron, in derselben Mikrosekunde** `openSheet()` auf — der ungünstigst mögliche Abstand zum `closeSheet()` davor.

**Reproduktion (alte Version):**
1. Sammlung-Tab öffnen
2. Kamera-Knopf **im Suchfeld der Sammlung** tippen (`sero.js:74-78`, `camBtn` mit Klasse `search-cam`, wird an `#colSearchBox` gehängt — **das ist Svens „über Sammlung hochladen"**)
3. Foto aufnehmen/wählen → `stageUpload` (`2410`/`2414`) → Lade-Sheet (`2421`) → „Scan prüfen" (`2450`)
4. **„Analysieren"** tippen, während das Scan-Kontingent erschöpft ist → Server antwortet 402
5. **Ergebnis alt:** Paywall wird geöffnet, ist aber durch `.sheet.closing{…!important}` aus dem Bild geschoben; 260 ms später räumt der alte Timer Sheet **und** Backdrop endgültig weg. **Zurück bleibt nur `recede`.** App klein + dunkel, kein Sheet, kein Backdrop, dauerhaft.

**Warum das am besten passt:** Es liegt exakt im Upload-Ablauf, es ist der einzige Pfad mit `closeSheet()` und `openSheet()` im selben Tick (Abstand 0 ms → 100 % Trefferquote statt Zufall), und es tritt **nur beim Hochladen** auf — nirgends sonst. Deckt sich mit „jedes Mal wenn ich über Sammlung was hochlade".
**Status:** durch den 13:24-Patch (`3761`/`3801`) behoben — die Paywall erscheint jetzt korrekt.

---

### Pfad 2 — „Alle Fotos zeigen dasselbe Stück" im Stapel-Scan (deterministisch, zweithöchste)

**Code:** `sero.js:2531`
```js
$("batchSingle").onclick = () => { closeSheet(); setTimeout(openAddSheet, 250); };
```
250 ms vs. 260 ms Aufräum-Timer → das neue Sheet wurde **10 ms nach seinem Erscheinen** vom alten Timer wieder versteckt. Kein Zufall, sondern jedes Mal.

**Reproduktion (alte Version):** Sammlung → Kamera-Knopf → in der Galerie **mehrere** Fotos wählen (`2403-2406`) → Sheet „Stapel-Scan" → **„Alle Fotos zeigen dasselbe Stück"** → Sheet blitzt kurz auf und ist weg, App bleibt grau/verkleinert.
**Status:** durch `3761`/`3801` behoben.

---

### Pfad 3 — Sheet sichtbar „offen", aber weggezogen: hängengebliebene Griff-Geste (bleibt offen)

**Code:** `sero.js:3817-3838` (Griff-Geste) im Zusammenspiel mit `openSheet` (`3750-3783`)

`openSheet` setzt `sh.style.transform` und `sh.style.transition` **nicht** zurück — nur `closeSheet` tut das (`3796`), und auch nur, wenn das Sheet nicht schon `hidden` ist. Bleibt nach einem Zieh-Versuch ein Inline-`transform: translateY(200px); transition:none` stehen (iOS liefert bei Rand-/Systemgesten nicht immer `pointerup`/`pointercancel`, `sy` bleibt dann auf `3821` gesetzt), gilt:
* Sheet ist `hidden === false`, sitzt aber teilweise/ganz unterhalb des Bildschirms
* der neue Wächter `pruefeSchleier` (`3811`) prüft nur `!$("sheet").hidden` → hält `recede` **aktiv**

**Reproduktion:** Scan-prüfen-Sheet am Griff nach unten ziehen, den Finger dabei über den unteren Bildschirmrand hinausziehen und loslassen (iOS-Home-Indikator-Bereich) → Sheet unten weg, App bleibt klein/dunkel, und der Wächter räumt nicht auf.
**Status: nicht behoben.** Das ist der einzige mir bekannte Pfad, der auch nach dem 13:24-Patch dauerhaft hängen bleibt.

---

### Pfad 4 — „Weiteres Foto"/„Rückseite fotografieren" mit 250-ms-Kamera (Restrisiko)

**Code:** `sero.js:2486`
```js
smb.onclick = () => { state.stageKeep = true; state.stageResume = true;
                      closeSheet(); setTimeout(() => $("cameraInput").click(), 250); };
```
Gleiche 250/260-Konstellation wie Pfad 2, aber der `click()` öffnet nur den Kamera-Dialog; das nächste `openSheet` (`2421`) kommt erst mit der Nutzeraktion, also weit nach 260 ms. In der alten Version nur dann kritisch, wenn das `change`-Event sofort feuerte (Desktop/Automatisierung). Praktisch selten.
**Status:** durch `3761`/`3801` entschärft.

---

### Pfad 5 — `pruefeAblage()` bzw. `loadCollection()` öffnen ein Sheet aus dem Hintergrund

**Code:** `sero.js:1939-1945` (`pruefeAblage`) und `sero.js:1949-1954` (`loadCollection`)
```js
1943: if ((r.photos || []).length && $("sheet").hidden) openStagedSheet(r.photos);
1952: if ((r.photos || []).length) openStagedSheet(r.photos);   // KEINE hidden-Prüfung
```
Zeile `1952` (Erst-Check nach `state._stageChecked`) öffnet das Staged-Sheet **ohne** die Sichtbarkeitsprüfung, die `1943` hat. Läuft das zufällig im 260-ms-Schließfenster, entstand in der alten Version derselbe Hängezustand. Ausgelöst wird `pruefeAblage` bei **jeder** Rückkehr in die App (`sero.js:1922-1933`, `visibilitychange`) — also nach jedem Kamera-Aufruf auf iOS.

Nebenbefund (gehört eher zu Svens Beschwerde 2): Dieser Pfad legt **alte, verwaiste Ablage-Fotos** wieder aufs Sheet. Wenn der Nutzer dann „Analysieren" tippt, entsteht ein weiteres Stück/Entwurf aus Fotos, die er längst vergessen hat.
**Status:** Race durch `3761`/`3801` entschärft; die fehlende `hidden`-Prüfung bei `1952` ist weiterhin inkonsistent zu `1943`.

---

### Pfad 6 — `.party`-Overlays parallel zu einem Sheet (kein Hänger, aber Verwirrung)

`showScanResult` (`sero.js:316-362`), `showScanFailed` (`366-402`), `celebrate` (`405-425`), `checkMilestone` (`240-264`), `showTour` (`1329-1363`) hängen alle ein `.party`-Div an `document.body`. `.party` (`sero.css:1173-1179`) hat `z-index:70`, `rgba(8,16,32,.55)` + `blur(10px)` — legt sich also **über** Sheet (z 21) und Backdrop (z 20).

Diese Overlays fassen `recede` **nie** an. Sie können den Hängezustand also nicht erzeugen. Sie können ihn aber **maskieren** und den neuen Wächter blockieren: `pruefeSchleier` (`3811`) zählt `document.querySelector(".party")` als „etwas ist offen" und entfernt `recede` dann nicht.

`showScanResult` wird aus `loadCollection` heraus getriggert (`sero.js:1985-1990`, über `state.watchNew`, Poll alle 2200 ms bei `1994`) — also **mitten in den Upload-Ablauf hinein**, unabhängig davon, welches Sheet gerade offen ist. Der `#resNext`-Knopf (`360`) ruft `switchTab("tabScan")` + Kamera, ohne ein evtl. offenes Sheet zu schließen.

---

### Pfad 7 — `switchTab` schließt nie ein Sheet (Struktur-Befund)

`switchTab` (`sero.js:1452-1468`) fasst weder Sheet noch `recede` an. Das ist meist unkritisch, weil das Backdrop (`sero.css:1090`, `inset:0`, `z-index:20`) alle Tab-Klicks abfängt und über `sheetBackdrop.onclick = closeSheet` (`3782`) sauber schließt. **Sobald das Sheet aber unsichtbar-aber-`recede` ist, ist auch das Backdrop weg** — der Nutzer kann normal navigieren und die App bleibt trotzdem klein und dunkel. Genau das erklärt, warum Sven weiterklicken kann und der Zustand trotzdem „hängt".

---

## 3. Was „springt das zurück" wörtlich meint

Das ist eine **zweite, unabhängige** Sache und **nicht** vom 13:24-Patch berührt: Der Erfolgspfad wirft den Nutzer aus der Sammlung heraus in den Scanner-Tab.

* `sero.js:2472-2475` (nach „Analysieren"): `closeSheet(); switchTab("tabScan"); loadCollection();`
* `sero.js:2518-2523` (Stapel-Scan): dito
* `sero.js:2549-2550` (`openAddSheet`): dito

Wer in der Sammlung startet (Kamera-Knopf `sero.js:74-78` im Suchfeld), landet nach dem Upload also **immer** im Scanner-Tab. „Springt zurück" + „wird gräulich" ist damit die Kombination aus erzwungenem Tab-Wechsel (`2473`) und stehengebliebenem `recede`.

---

## 4. Bewertung

**Bester Erklärer für Svens Beobachtung: Pfad 1 (402/Paywall).** Als einziger liegt er im Upload-Ablauf, hat 0 ms Abstand zwischen `closeSheet` und `openSheet` (also 100 % Reproduzierbarkeit statt Timing-Glück) und tritt ausschließlich beim Hochladen auf. Pfad 2 ist ebenso deterministisch, betrifft aber nur den Stapel-Scan mit anschließendem „Alle Fotos zeigen dasselbe Stück".

**Was nach dem 13:24-Patch noch offen ist:**
1. **Pfad 3** (hängengebliebenes Inline-`transform` am Sheet) — einziger verbleibender Dauer-Hänger. `openSheet` sollte `sh.style.transform`/`transition` zurücksetzen und `sy` invalidieren; `pruefeSchleier` (`3811`) sollte nicht nur `hidden` prüfen, sondern auch, ob das Sheet tatsächlich im Viewport liegt.
2. **Wächter-Latenz:** bis zu 1200 ms sichtbarer Grau-Zustand (`sero.js:3814`) — Symptombehandlung, nicht Ursache.
3. **Inkonsistenz `1952` vs. `1943`** (fehlende `hidden`-Prüfung).
4. **Cache-Buster nicht mitgezogen:** `index.html:182` lädt weiterhin `sero.js?v=93`, obwohl `sero.js` um 13:24 neu geschrieben wurde (`index.html:23` ebenso `sero.css?v=52`). Es gibt **keinen** Service Worker im Projekt (geprüft: kein `sw.js`, kein `serviceWorker`-Aufruf), also greift nur der HTTP-Cache — Svens installierte PWA kann trotzdem weiter die alte Datei ausliefern. **Der Fix wirkt bei ihm erst nach hartem Neuladen bzw. nach Erhöhen der `?v=`-Nummer.** (Nur Beobachtung — ich habe nichts geändert.)

**Randnotiz zu Beschwerde 2 (nicht mein Auftrag, aber im gelesenen Code sichtbar):** In `sero.js:2463-2468` wird bei einem Fehler von `items-from-stage` mit `e.offline` die Ablage bewusst **behalten** (`state.stageKeep = true`) und der Fehler durchgereicht. Erreicht der POST den Server, geht aber nur die Antwort verloren, ist das Stück serverseitig bereits angelegt — der nächste Tipp auf „Analysieren" legt ein **zweites** an. Zusammen mit Pfad 5 (alte Ablage-Fotos kommen von selbst wieder aufs Sheet) ist das eine plausible Frontend-Quelle für die Entwurfs-Vermehrung.