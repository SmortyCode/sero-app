# SERO — Mikro-Interaktions- und Erlebnis-Audit
*Interaction-Design-Review auf Basis von `/Users/smorty/sero-app/web/sero.js`, `sero.css`, `index.html` (read-only, Stand 02.08.2026)*

---

## 1. Kurzfazit

Die App ist handwerklich weiter, als die meisten nativen Apps im Store: richtungsbewusste Tab-Übergänge, Karten-Stagger, Odometer-Zahlen, Gyro-Holo mit kontextbezogener iOS-Permission, Undo statt Dialoge, Reduced-Motion sauber respektiert. Das ist Linear/Things-Niveau in Vanilla JS.

Zwei Dinge stehen dem Gänsehaut-Anspruch im Weg:

1. **Der Scan-Moment ist fragmentiert.** Anticipation (Warten auf die Analyse) ist ein 14-px-Spinner in einer Listenzeile, der Reveal (`showScanResult`) feuert irgendwann per Polling — und die Hauptzahl der ganzen App, der Marktwert, **erscheint ohne jede Animation**, weil der Odometer dort nie anrollt (Bug, siehe 3.2).
2. **Haptik existiert auf dem Zielgerät nicht.** Alle drei `navigator.vibrate`-Aufrufe sind auf iOS Safari/PWA No-Ops — sie wirken heute nur auf Android. Der Capacitor-Umzug ist die Chance, das mit einem Wrapper in einem Schritt zu lösen (siehe 4).

---

## 2. Wo die App SCHON magisch ist (nicht anfassen)

| Detail | Beleg | Warum es trägt |
|---|---|---|
| App-Recede hinter Sheets: `#viewApp.recede` skaliert auf .945, rundet Ecken, dimmt | `sero.css:69–75` | Echtes iOS-Modal-Gefühl, macht jedes Sheet räumlich |
| Richtungsbewusster Tab-Wechsel (`pageInR`/`pageInL`, Richtung wird VOR dem Verstecken gemerkt) | `sero.js:1392–1402`, `sero.css:242–254` | Räumliche Kontinuität — selten selbst in nativen Apps |
| Grid-Stagger mit `--i`-Kappung bei 10 (40 ms Delay je Kachel) | `sero.js:2053`, `sero.css:536–537` | Lebendig, ohne bei 200 Stücken zu nerven |
| Swipe auf Kachel mit Aufdecken der Aktion, „armed“-Zustand + `vibrate(8)` bei 80 px | `sero.js:2101–2149` | Things-Kaliber: man SIEHT, was passieren wird |
| Holo-Tilt + Gyro, Permission erst beim Tipp aufs Foto, Reduced-Motion-Ausnahme | `sero.js:3353–3400`, `sero.css:683–685` | Der eine „Wow“-Effekt, der zur Sache (Holo-Karte) gehört statt Deko zu sein |
| Odometer mit Fallback bei Formwechsel/Reduced-Motion | `sero.js:507–525`, `sero.css:1566–1568` | Apple-Wallet-Look, technisch sauber |
| Undo-Toast statt Nachfrage-Dialog (`removeItemWithUndo`, Favoriten-Swipe) | `sero.js:114–124, 2139–2145` | Vertrauens-Muster von Things/Mail |
| Scan-Orb: atmender Glow + `:active scale(.88) rotate(-4deg)` | `sero.css:212–222` | Der Hero-Knopf fühlt sich physisch an |
| Content-Maske unter der Glas-Tab-Bar | `sero.css:229–230` | Das Detail, das „Liquid Glass“ glaubwürdig macht |
| Offline-Foto-Rettung mit Aktions-Toast | `sero.js:2370–2379` | Fehlerfall als gestalteter Moment |
| iOS-Titel-Verdichtung mit Wert-Pille (`wireMiniVal`) | `sero.js:1445 ff.` | Systemgefühl |

Ehrliche Einordnung: An Übergängen und Zuständen muss nichts „gerettet“ werden. Es geht um Verdichtung an zwei Stellen (Scan, Listing-live) und um Haptik, die real ankommt.

---

## 3. Der Scan-Moment: Ist-Zustand seziert

**Ablauf heute:** „Analysieren“ → `closeSheet()` → `switchTab("tabScan")` → Toast „1 Foto — Wird analysiert“ (`sero.js:2413–2415`) → Warten als Spinner-Zeile in „Zuletzt gescannt“ (`sero.js:2560`) mit Server-`status_text` → Polling alle 2,2 s (`sero.js:1933–1934`) bzw. SSE → `showScanResult` (`sero.js:303–339`): Bottom-Card `rise .5s`, Foto `resPop .7s`, Wert, Quelle, zwei Buttons.

### 3.1 Was fehlt: die Bühne
Die Wartezeit — der emotional spannendste Teil („was ist sie wert?“) — findet in einer 47-px-Listenzeile statt. Kein Zeitgefühl, kein Fortschritt, obwohl `avg_scan_s` aus echten Messungen vorliegt (`sero.js:2576`). Der Reveal kann den Nutzer zudem in jedem Tab „überfallen“, weil er am Polling hängt, nicht an einer Inszenierung.

### 3.2 Der konkrete Fehler: der Odometer rollt im Scan-Ergebnis nie
`sero.js:330` ruft `countUp(el, "scanres_" + item.id, val)`. In `countUp` (`sero.js:509`) gilt `from = state.anim?.[key] ?? to` — der Schlüssel ist pro Stück neu, also `from === to`, damit `txt === old`, `rollable = false` → `el.textContent = txt`. **Die wichtigste Zahl der App ploppt statisch hin.** Fix: vor dem Aufruf `state.anim["scanres_" + item.id] = 0;` setzen (oder `countUp` einen optionalen `from`-Parameter geben). Kostet eine Zeile, verändert den Kernmoment.

### 3.3 Choreografie-Vorschlag: drei Akte

**Akt 1 — Anticipation (Analyse-Bühne im Scan-Tab).** Solange `state.watchNew` auf ein Stück mit `status === "analyzing"` zeigt, ersetzt eine Bühne den `scan-hero` (`index.html:101–109`, gerendert in `renderScan`, `sero.js:2546`):
- Das geparkte Foto (URL liegt aus `openStagedSheet` vor) mittig, ca. 180 px, mit dem vorhandenen `res-photo`-Glow.
- **Scan-Strahl**: `@keyframes scanbeam` — horizontaler Gradient-Balken, `top 0→100%`, `1600ms cubic-bezier(.45,0,.55,1) infinite alternate`.
- `status_text`-Stufen mit 250-ms-Crossfade statt hartem Austausch.
- **Fortschrittsring aus echten Daten**: füllt in `avg_scan_s` Sekunden auf 90 % (`transition: stroke-dashoffset` mit `cubic-bezier(.2,.6,.2,1)`), hält dort, springt beim Ergebnis auf 100 %. Darunter: „dauert meist rund {avg} Sekunden“ — die ehrliche Zahl, die die App ohnehin schon misst.

**Akt 2 — Reveal (`showScanResult`, gestaffelt statt gleichzeitig).** Heute animiert nur das Foto (`resPop`), alles andere steht sofort da. Vorschlag (alles über CSS-`animation-delay` + `backwards`, eine Karte, ein DOM-Aufbau):

| Zeit | Element | Animation | Haptik |
|---|---|---|---|
| 0 ms | Karte | `rise .5s var(--spring)` (bestehend) | — |
| 120 ms | Foto | `resPop .7s var(--spring)` (bestehend, nur Delay ergänzen) | leicht (10 ms) |
| 300 ms | Badge „Erkannt“ | Scale .6→1, `.35s var(--spring)` | — |
| 400 ms | Name + Sub | TranslateY 8 px + Fade, `.3s ease-out` | — |
| **550 ms** | **Marktwert** | Odometer rollt von 0 (Fix aus 3.2), `.65s cubic-bezier(.35,1.05,.35,1)` (bestehende `.odr`-Kurve), dazu ein weicher Radial-Glow-Puls hinter `.res-val` (`.6s ease-out`) | mittel (`[12,50,20]`), per `setTimeout(550)` statt wie heute sofort bei Karten-Einblendung (`sero.js:338`) |
| 900 ms | Quellen-Pille + Buttons | Fade `.25s` | — |

Gesamt ~1,15 s — schnell genug für Stapel-Scanner, langsam genug für Gänsehaut. `prefers-reduced-motion` deckt der bestehende `countUp`-Fallback plus ein `@media`-Block über die Delays ab.

**Akt 3 — Belohnung/Schleife.** „Weiter scannen“ existiert und ist richtig. Ergänzung nur beim allerersten Scan überhaupt: eine Zeile unter den Buttons — „Ein Tipp auf Verkaufen macht daraus ein fertiges eBay-Listing.“ (localStorage-Flag, analog `sero_tour`). Das verbindet den Aha-Moment mit der Positionierung, ohne den Flow zu unterbrechen.

**Sound: Nein (Standard).** Begründung: Zielszenario ist der Karten-Flipper, der 30 Stücke am Tisch oder auf der Börse durchscannt — Ton nervt dort ab Scan 3. iOS-PWA-Audio braucht zudem Gesten-Kontext und bricht gern (AudioContext-Resume). Die Belohnung soll über Haptik laufen, die mit Capacitor real wird. Einzige spätere Ausnahme, opt-in: ein dezenter Ton bei der „Verkauft“-Push-Nachricht — das ist der Moment, in dem Geld real wird, und er passiert außerhalb der Nutzungssituation.

---

## 4. Haptik-Inventur — und die unbequeme Wahrheit

Bestand: genau drei Stellen — Meilenstein `[10,60,18]` (`sero.js:263`), Scan-Ergebnis `12` (`sero.js:338`), Swipe-„armed“ `8` (`sero.js:2119`).

**`navigator.vibrate` existiert auf iOS Safari/PWA nicht** — auf Svens Zielplattform iPhone passiert heute bei allen drei Stellen: nichts. Sie wirken nur auf Android. Konsequenz für den Capacitor-Umzug:

```
function haptic(kind = "light") {
  const H = window.Capacitor?.Plugins?.Haptics;
  if (H) { kind === "success" ? H.notification({type:"SUCCESS"}) : H.impact({style: kind.toUpperCase()}); return; }
  if (navigator.vibrate) navigator.vibrate({light:10, medium:[12,50,20], success:[10,60,18]}[kind] || 10);
}
```

Einmal einführen, die drei bestehenden Aufrufe umstellen — dann fühlt sich die App-Store-Version am Tag eins nativ an. Fehlende Stellen (nach Wichtigkeit): `celebrate()` „Live auf eBay“ (**der Geld-Moment hat heute null Haptik**, `sero.js:382–401`), Pull-to-Refresh beim Überschreiten der 85-px-Schwelle (`sero.js:422`), Long-Press-Menü beim Auslösen (`sero.js:2104`), Favoriten-Erfolg (`sero.js:2137`), Wert-Reveal in Akt 2.

---

## 5. Zehn Mikro-Interaktionen, sortiert nach Wirkung

1. **Scan-Reveal-Choreografie inkl. Odometer-Fix.** `showScanResult` (`sero.js:303–339`): `state.anim["scanres_"+item.id] = 0` vor `countUp`; CSS-Delays auf `.res-badge`, `.res-name`, `.res-sub`, `.res-val`, `.res-src`, `.party-actions` (neue Keyframes `revealUp` = TranslateY 8 px + Fade); Vibrate von Zeile 338 in ein `setTimeout(…, 550)` verschieben. Größter Hebel der ganzen Liste.
2. **Analyse-Bühne im Scan-Tab.** `renderScan` (`sero.js:2546`): wenn das `state.watchNew`-Stück analysiert, Bühne statt `scan-hero` rendern; Keyframes `scanbeam`; Fortschrittsring aus `time_saved.avg_scan_s`; `status_text`-Crossfade (Klasse togglen statt `textContent` hart setzen).
3. **`haptic()`-Wrapper mit Capacitor-Bridge** (siehe 4). Betroffen: `sero.js:263, 338, 2119` plus fünf neue Stellen. Eine Funktion, App-weiter Effekt.
4. **`celebrate()` zum zweiten Gänsehaut-Moment machen.** `sero.js:382–401`: Preis per `countUp` von 0 rollen (Seed wie in Punkt 1, Schlüssel `party_<draft_id>`); `party-ring` als SVG-Kreis mit `stroke-dashoffset`-Draw (Kurve `cubic-bezier(.22,.9,.16,1)` wie `sv2-draw`, `sero.css:1390`); `haptic("success")`. Optional 10–12 Partikel in Navy/Gold (CSS-only, `transform`-Keyframes) — mehr nicht, „Clean + Wow“ heißt hier: ein Effekt, nicht Konfettiregen.
5. **Sheet-Snap-back federn + Backdrop mitziehen.** Der Griff folgt 1:1 (`sero.js:3735–3755`), aber unterhalb 90 px teleportiert das Sheet zurück, weil `.sheet` (`sero.css:1025–1033`) keine `transition` hat. Fix: beim Loslassen Klasse `.snapback { transition: transform .35s var(--spring) }` setzen und nach `transitionend` entfernen. Dazu im `pointermove` Backdrop-Opacity und `recede`-Skalierung proportional zu `dy` interpolieren — dann fühlt sich die Geste physisch statt geskriptet an.
6. **Pull-to-Refresh mit Arm-Zustand.** `attachPTR` (`sero.js:404–433`): Monogramm mit `rotate(${dist*1.8}deg)` mitdrehen; bei `dist > 85` Klasse `.armed` (Scale 1.15, `.2s var(--spring)`) + `haptic("light")` einmalig; beim Erfolg nicht hart auf `opacity 0`, sondern `.2s`-Fade. Drei Zeilen JS, ein CSS-Block.
7. **Tab-Retap = Scroll-to-top, kein Animations-Replay.** `switchTab` (`sero.js:1392–1408`) spielt auch beim Tipp auf den aktiven Tab `pageInR` neu ab. Early-Return wenn `id` schon sichtbar, stattdessen `page.querySelector(".page-scroll").scrollTo({top:0, behavior:"smooth"})` — iOS-Standardverhalten, das Vielnutzer erwarten.
8. **Long-Press-Buildup auf Kacheln.** `sero.js:2102–2105`: bei `pointerdown` Klasse `.pressing` (Scale .97 über 450 ms `ease-out` — die Kachel „duckt sich“ während der Timer läuft), beim Auslösen `haptic("light")`; bei `pointermove`/`up` Klasse entfernen. Macht die versteckte Geste entdeckbar.
9. **Favoriten-Stern auf der Kachel feiern.** Nach erfolgreichem Rechts-Swipe (`sero.js:2134–2147`) erscheint das `gfav`-Icon erst nach `loadCollection` irgendwann. Sofort optimistisch auf der Kachel einblenden mit `ringPop`-artigem Keyframe (Scale 0→1.15→1, `.5s var(--spring)`) — die Belohnung gehört ans Objekt, nicht nur in den Toast.
10. **`res-grip` ehrlich machen.** Die Ergebnis-Karte zeigt einen Griff (`sero.js:316`, `sero.css:1206`), der nichts tut — schließen geht nur per Backdrop. Entweder die Griff-Logik des Sheets (`sero.js:3735–3755`) wiederverwenden oder den Griff entfernen. Affordances, die lügen, sind der Unterschied zwischen „fast Apple“ und Apple. (Gleiche Klasse Kleinigkeit: `closeDetail` blendet nach 260 ms aus, `slideout` dauert 280 ms — `sero.js:2969` vs. `sero.css:626` — die letzten Frames werden abgeschnitten; Timeout auf 290 ms.)

---

## 6. Onboarding: die ersten 60 Sekunden

**Ist-Ablauf:** Splash (Puls, ~350 ms) → Login mit Telegram-Code → Tour nach 850 ms (`sero.js:1363`): drei Text-Karten (`TOUR`, `sero.js:1267–1303`) → letzter Button „Erste Karte scannen“ → `switchTab("tabScan")` → Nutzer muss **noch einmal** „Scannen“ tippen → Kamera → „Scan prüfen“-Sheet → „Analysieren“ → Warten (Spinner-Zeile, unbekannte Dauer) → Ergebnis.

**Wo die App heute verliert:**
1. **Der doppelte Tipp zur Kamera.** Der Tour-Endbutton verspricht „Erste Karte scannen“, landet aber nur auf dem Scan-Tab (`sero.js:1295`). Fix: im Click-Handler synchron `$("cameraInput").click()` auslösen — der Button-Tap ist der Gesten-Kontext, den iOS für den Kamera-Dialog verlangt; das heutige `setTimeout(300)` verschenkt ihn. Ein Schritt weniger zum Aha.
2. **Die Wartezeit ohne Erwartungsmanagement.** Beim allerersten Scan gibt es noch keine `avg_scan_s`-Daten (Statistiken erscheinen erst ab 3 Scans, `sero.js:2575`) — genau dann ist die Unsicherheit am größten. Die Analyse-Bühne (Punkt 5.2) braucht für Scan 1–2 einen ehrlichen Festwert („dauert meist unter einer Minute“).
3. **Drei abstrakte Tour-Karten vor dem ersten Erlebnis.** „Wert verfolgen“ und „Verkaufen“ erklären Dinge, die der Nutzer noch nicht gesehen hat. Vorschlag: Tour auf **eine** Karte kürzen (Scannen), die anderen beiden Botschaften kontextuell nachliefern — „Verkaufen“ als Einmal-Hinweis auf der ersten Ergebnis-Karte (siehe Akt 3), „Wert verfolgen“ als Einmal-Hinweis beim ersten Sammlungs-Besuch mit ≥1 Stück. Erlebnis schlägt Erklärung.
4. Der Telegram-Code-Login liegt vor all dem — außerhalb dieses Audits, aber er ist die teuerste Hürde der ersten 60 Sekunden und sollte vor dem Store-Launch gegen Sign in with Apple antreten.

**Schnellster Weg zum ersten Aha in Zahlen:** Heute: Splash → Tour (3 Karten, ≥4 Tipps) → Tab → Scannen → Kamera ≈ 6–7 Interaktionen vor dem Foto. Mit Punkt 1+3: Splash → 1 Tour-Karte → Kamera = 2 Tipps. Der Marktwert der ersten Karte — das eigentliche Verkaufsargument — rückt damit unter eine Minute ab App-Start.

---

## 7. Priorisierung (Aufwand → Wirkung)

| # | Maßnahme | Aufwand | Wirkung |
|---|---|---|---|
| 1 | Odometer-Seed-Fix im Scan-Ergebnis | 1 Zeile | Kernmoment repariert |
| 2 | Tour-Endbutton öffnet Kamera direkt | ~3 Zeilen | Onboarding-Conversion |
| 3 | Reveal-Staffelung (CSS-Delays + Haptik-Timing) | ~30 Zeilen CSS, 5 JS | Gänsehaut |
| 4 | `haptic()`-Wrapper (Capacitor-ready) | ~15 Zeilen | Native Anmutung ab Store-Tag 1 |
| 5 | Analyse-Bühne mit Fortschrittsring | ~80 Zeilen | Anticipation, wahrgenommene Geschwindigkeit |
| 6 | `celebrate()`-Upgrade | ~40 Zeilen | Der Geld-Moment trägt die Positionierung |
| 7–10 | Sheet-Snapback, PTR-Arm, Tab-Retap, Long-Press, Stern-Pop, res-grip | je < 20 Zeilen | Summe macht „poliert“ |

Alles oben nutzt ausschließlich vorhandene Bausteine (`--spring`, `.odr`-Kurve, `resPop`, `rise`, `sv2-draw`-Easing) — kein neues Framework, keine neue Designsprache, konsistent mit „Mix: Clean + Wow“.