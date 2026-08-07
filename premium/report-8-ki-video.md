# SERO — 4 produktionsreife Video-Konzepte
*Regie-Dokument. Stand: 02.08.2026. Alle Prompts englisch (so performen die Modelle), alle Overlays deutsch nach Wording-Regeln (Stück, listen, Marktwert, tippen, keine Ausrufezeichen, SERO in 3. Person).*

---

## 0. Was die App schon mitbringt — und was KI niemals übernimmt

Ehrliche Bestandsaufnahme: Die Hälfte des Materials liefert die App selbst, und zwar besser, als KI es je könnte. Diese Elemente sind **fertig gebaut** und werden als Screen-Capture gedreht, nie generiert:

| App-Element | Code-Anker | Warum es im Video trägt |
|---|---|---|
| Scan-Orb mit atmendem Glow | `.tab-cam` + `@keyframes orbbreathe` (sero.css:204–218), Tap-Feedback `scale(.88) rotate(-4deg)` | Der eine Knopf, um den sich alles dreht — 3,2s-Atmung ist bereits eine Loop-Animation |
| „Erkannt"-Reveal-Sheet | `showScanResult()` (sero.js:303), `.party.scanres`, Foto-Pop `@keyframes resPop` (sero.css:1230) | Der Wow-Moment der App ist der Wow-Moment des Spots |
| Odometer-Zahlen | `countUp()` (sero.js:507), Apple-Wallet-Rollen | Geld, das sich sichtbar bewegt — die Kernbotschaft in einer Animation |
| Gyro-Holo | `.holo-wrap::before` Conic-Gradient in Rosé/Gold/Mint/Eisblau, `mix-blend-mode: color-dodge` (sero.css:673–682), Gyro-Treiber sero.js:3378 | Ein echter Karten-Schimmer auf dem Screen — kein anderes Tool der Kategorie hat das |
| Kerzen-Chart | `candleChart()` (sero.js:621), „TradingView-Style" | Verwandelt Sammeln visuell in Trading — exakt die Positionierung |
| Liquid-Glass-Tab-Bar | `.tabbar` mit `blur(26px)` (sero.css:176) | Der Apple-Look, der Premium signalisiert |
| Marken-Splash | `#splash` + `splashPulse` (sero.css:91) | Kalter App-Start ist ein gratis Branding-Beat |

**Produktions-Notizen für die Captures (aus dem Code, vor dem Dreh prüfen):**

1. **Testmodus-Badge ausblenden** — `#dryBadge` (index.html:60) darf in keinem Take sichtbar sein. Capture-Account im Normalmodus.
2. **Privacy-Blur aus** — `.hideable.veiled` legt `blur(10px)` über Werte (sero.css:287). Vor dem Dreh deaktivieren.
3. **Odometer rollt nur bei gleicher Stellenzahl** (`rollable`-Check, sero.js:512). Capture-Werte planen: z. B. 1.243,50 € → 1.687,20 € rollt, 987,00 € → 1.240,00 € springt hart. Und: Im Scan-Ergebnis-Sheet rollt `res-val` beim ersten Anzeigen *nicht* (sero.js:509 setzt `from = to` bei neuem Key) — die echte Roll-Animation lebt auf Home (`.h-value`, countUp „dashVal", sero.js:1697). Also: fürs Rollen den Dashboard-Wert nach einem Scan filmen, nicht dem Sheet hinterherjagen.
4. **Gyro-Holo braucht auf iOS einen Permission-Tap** auf die Karte (sero.js:3392–3397). Vor dem Take einmal tippen, dann läuft der Schimmer mit der Neigung.
5. **Tipp auf den Chart togglet Linie ↔ Kerzen** (sero.js:1712) — ein filmbarer Interaktionsbeat.
6. Screen-Recordings nativ 9:16 vom iPhone; für Composites (Screen ins gefilmte Gerät tracken) in ProRes via QuickTime-Mac-Capture.
7. **App-Store-Preview-Regel:** Apple verlangt überwiegend echtes App-Material. Jeder Spot bekommt daher zwei Schnittfassungen: Reel-Cut (cinematisch) und Preview-Cut (gleiche Struktur, UI-Anteil > 80 %).

---

## Konzept 1 — „Der Reveal" (15 s, 9:16)

**Einsatz:** App-Store-Preview (Preview-Cut) + Reels/TikTok (Reel-Cut).
**Dramaturgie:** Eine Karte. Ein Foto. Der Wert materialisiert sich — erst als Licht (KI), dann als echte Zahl (Capture). Der Trick ist der Match-Cut von generiertem Licht auf echte UI.

**Licht-Satz des Konzepts (wortgleich in jedem KI-Prompt):**
`single warm practical lamp from the upper left, cool soft window fill from the right, faint haze in the air`

### Shotliste

| Zeit | Shot | Quelle |
|---|---|---|
| 0,0–2,5 s | Hand legt geslabbte Karte auf dunkle Eiche, zieht sich zurück | **Kling 2.5 Pro**, I2V |
| 2,5–4,5 s | Over-Shoulder: iPhone hebt sich über die Karte, Kamerabild live | **Real** (Screen in Post ersetzt) |
| 4,5–7,5 s | Screen-Capture: Scan → „Erkannt"-Sheet poppt, Wert steht | **Real** (Capture, Punch-in in Post) |
| 7,5–11,0 s | Licht steigt aus der Kartenoberfläche, verdichtet sich zu einem Glühen | **Hailuo 02** |
| 11,0–15,0 s | Match-Cut: Glühen → Home-Dashboard, Odometer rollt auf neuen Gesamtwert; Endcard Navy + SR-Monogramm | **Real Capture + After Effects** |

### Prompts

**Shot 1 — Kling 2.5 Pro** (Image-to-Video; Kling, weil es Hände und Objektgewicht am glaubwürdigsten rendert und das First Frame am treuesten hält). First Frame: echtes Foto einer echten Karte aus Svens Bestand auf dem echten Tisch — die Karten-Art bleibt damit korrekt.

> Static macro shot on a 100mm macro lens, f/2.8, shallow depth of field. A collector's hand in a dark navy sweater gently places a single graded trading card in a clear acrylic slab onto a dark oak table, then withdraws out of frame. The card settles with soft, believable weight, a faint reflection sliding across the slab. single warm practical lamp from the upper left, cool soft window fill from the right, faint haze in the air. Photorealistic, filmic, Kodak Vision3 500T look, fine grain, deep true blacks. The camera does not move. No text, no numbers, no UI elements, no extra fingers, no warping of the card face.

**Shot 4 — Hailuo 02** (Hailuo, weil die Regie-Kommandos in Klammern die präziseste Kamerafahrt liefern und das Modell Partikelphysik sauber hält).

> [Push in, slow] Extreme macro gliding over the surface of a holographic trading card lying on dark oak. Hundreds of fine particles of golden-blue light rise slowly from the foil surface like embers in reverse, drifting upward and condensing into one soft glowing orb just above the card. single warm practical lamp from the upper left, cool soft window fill from the right, faint haze in the air. 100mm macro, f/2.8, focus locked on the foil texture. Photorealistic with restrained, physical lighting, filmic grain, deep navy shadows. No text, no numbers, no UI elements, no lens flares, no fantasy symbols.

### Real drehen

- Shot 2: echtes iPhone in echter Hand über der echten Karte, ProRes; Display in Post per Corner-Pin-Tracking durch den Clean-Capture ersetzt (Displays flackern sonst, und nur so ist die UI pixelscharf).
- Shot 3: Capture des kompletten echten Ablaufs — Tap auf `.tab-cam`, Foto, `.party.scanres` mit „Erkannt"-Badge und `resPop` des Fotos.
- Shot 5: Capture des Home-Tabs direkt nach dem Scan, damit `.h-value` sichtbar rollt (Stellenzahl beachten, Notiz 3).

### Schnitt

Der Match-Cut ist die Naht: Das Hailuo-Glühen wandert in den letzten 10 Frames leicht nach oben aus dem Bild — Schnitt auf das Dashboard, in dem exakt in diesem Moment das Odometer losrollt. Das Licht „wird" die Zahl. Ein gemeinsames Grade (Navy-Schatten, warme Haut) plus 35-mm-Grain über alle Quellen.

### Sound

Referenz: **Apples „iPhone 15 Pro — Titanium"-Spot** — taktile Mikro-Whooshes, trockener Sub-Tap beim Antippen, sonst fast Stille. Beim Reveal ein einzelner weicher Glas-Ton (kein Kassen-Kling — das wäre unter dem Niveau der App). Odometer bekommt ein leises mechanisches Rattern wie eine Rolex-Datumsscheibe.

### Text-Overlays (Reel-Cut)

- 2,5 s: „Fotografieren."
- 5,5 s: „Erkannt."
- 9,0 s: „Marktwert aus echten eBay-Verkäufen"
- 13,0 s: „Ein Tipp. Gelistet."
- Endcard: SR-Monogramm + „SERO macht deine Karten zu Geld."

*(Preview-Cut: gleiche Struktur, Shots 1 und 4 raus, dafür längere UI-Strecken — Sammlung-Grid, Kerzen-Chart, Verkauf-Tab.)*

---

## Konzept 2 — „Aus der Schuhschachtel" (30 s, 9:16)

**Einsatz:** Reels/TikTok/YouTube-Shorts, Launch-Story für die Zielgruppe 16–45, die genau diese Schachtel im Schrank hat.
**Dramaturgie:** Dachboden → Erinnerung (2003, Schulhof) → Gegenwart: dieselbe Geste, die Karte ins Licht zu halten — nur dass diesmal SERO antwortet. Endet mit der stärksten Zeile aus den Marketing-Bausteinen.

**Licht-Satz des Konzepts:** `one hard shaft of late-afternoon sunlight from a small window, dust motes drifting, everything else falling into soft warm darkness`

### Shotliste

| Zeit | Shot | Quelle |
|---|---|---|
| 0–3,5 s | Dachboden, Lichtschacht, Hand zieht Schuhschachtel vom Regal | **Sora 2** |
| 3,5–6,5 s | Macro: Deckel öffnet sich, Karten im Staublicht | **Real** |
| 6,5–10,5 s | Flashback 2003: zwei Jungs tauschen Karten auf dem Schulhof, MiniDV-Look | **Runway Gen-4** |
| 10,5–14 s | Gegenwart: erwachsene Hand fächert dieselben Karten, hebt eine ins Licht | **Real** |
| 14–19 s | Phone-Scan über der Schulter, „Erkannt"-Sheet | **Real** (Composite) |
| 19–24 s | Montage: drei schnelle Scans, Dashboard-Odometer rollt, Tipp auf den Chart → Kerzen | **Real** (Captures) |
| 24–27 s | Fast leere Schachtel im Abendlicht, Phone liegt daneben, Verkauf-Tab „Aktiv" | **Real** |
| 27–30 s | Endcard | **Post** |

### Prompts

**Shot 1 — Sora 2** (Sora, weil es komplexe Räume mit physikalisch korrektem Staub und mehreren Bildelementen am stabilsten baut — für den einen Establishing-Shot, den man real teuer ausleuchten müsste).

> Handheld documentary feel with a gentle sway, 35mm lens, vertical frame. A dim attic; one hard shaft of late-afternoon sunlight from a small window, dust motes drifting, everything else falling into soft warm darkness. A man in his early thirties reaches up to a high wooden shelf and pulls down an old cardboard shoebox; a thin layer of dust slides off the lid as it tilts. Photorealistic, filmic, Kodak Portra palette, fine grain, muted warm browns. No text, no logos, no readable labels, no faces in sharp focus.

**Shot 3 — Runway Gen-4** (Runway, weil die References-Funktion denselben Jungen über mehrere Takes konsistent hält und Gen-4 Film-Stock-Ästhetik — hier MiniDV — am kontrollierbarsten emuliert; Kinder-Casting fürs echte Drehen wäre der teuerste Posten des Films).

> A memory, shot as if on a consumer MiniDV camcorder from 2003: two ten-year-old boys crouch on schoolyard asphalt trading collectible cards, backpacks beside them, baggy early-2000s clothes. One boy holds a card up against the overcast sky, grinning at how it catches the light. 4:3 image floating inside the vertical frame, slight interlacing, blown-out highlights, washed-out colors, handheld wobble, tape noise. No text, no readable logos, no modern objects, no smartphones.

### Real drehen

- Shots 2, 4, 7 sind mit Svens echtem Bestand in einer Stunde gedreht — echte Schachtel, echte Karten, ein LED-Spot als „Fenster", Staub aus der Sprühflasche. Diese drei Shots real zu machen ist billiger UND glaubwürdiger als jede Generierung.
- Shot 5/6: Captures wie in Konzept 1; zusätzlich der Chart-Toggle-Tap (sero.js:1712) als sichtbare Interaktion und der Verkauf-Tab mit Segment „Aktiv" (index.html:126).
- Die Geste „Karte ins Licht halten" wird in Shot 3 (KI, Kind) und Shot 4 (real, Erwachsener) identisch kadriert — das ist der emotionale Match-Cut des Films.

### Schnitt

Flashback als 4:3-Fenster im 9:16-Frame, Gegenwart vollformatig — der Formatwechsel erzählt den Zeitsprung ohne ein Wort. Die Montage 19–24 s schneidet auf den Takt des Sounds (siehe unten), jeder Scan ein Beat.

### Sound

Referenz: **Ólafur Arnalds, Filzklavier-Ästhetik** („saman") mit Tape-Hiss unter dem Dachboden und Flashback; ab dem ersten Scan (14 s) kippt es in einen kleinen, trockenen Rhythmus aus echten Kartengeräuschen (Riffeln, Ablegen). Letzter Beat bei 28 s: der echte eBay-Verkaufs-Push-Ton. Bewusste Entscheidung: Der Verkauf gehört laut Marketing-Doku nicht in die Ablauf-Tour, aber er IST die Positionierung („macht sie zu Geld") — als letzter Story-Beat dramatisiert er die Leitplanke, ohne dass die App etwas verspricht, was sie nicht hält.

### Text-Overlays

- 5 s: „Du weißt, was da drin liegt."
- 12 s: „SERO weiß, was es wert ist."
- 21 s: „Echte Verkäufe statt Wunschpreise." *(Zeile 4 aus den Ein-Zeilern, marketing-bausteine.md §5)*
- Endcard: „Der Stapel schrumpft, nicht dein Wochenende." + Wordmark

---

## Konzept 3 — „Holo" (8 s Loop, 4:5-safe in 9:16)

**Einsatz:** Instagram-Grid-Anker, Profil-Header, Loop im Reel-Hintergrund. Pure Ästhetik, null Erklärung.
**Idee:** Die Foliage einer Holo-Karte als Landschaft — ein Makro-Überflug, bei dem ein Regenbogen-Lichtband über die Prägung wandert. Die Farben sind exakt die vier Töne des App-Shaders (`.holo-wrap::before`, sero.css:675–677): Rosé, Gold, Mint, Eisblau. Das Video ist damit der App-Effekt in physisch — wer danach die App öffnet und die Karte neigt, erkennt das Video wieder.

### Shotliste

| Zeit | Shot | Quelle |
|---|---|---|
| 0–8 s | Ein einziger nahtloser Makro-Flug, Lichtband vollendet genau einen Durchlauf | **Luma Ray2** (Loop-Modus) |
| Variante B | Realer Gegenschuss: Phone in der Hand, Gyro-Holo läuft synchron auf dem Screen | **Real** (On-Device) |

### Prompt

**Luma Ray2** (Luma, weil es als einziges Tool nativen Seamless-Loop mit identischem Start-/Endframe beherrscht — genau die eine Fähigkeit, an der dieses Konzept hängt; als First/Last Frame dasselbe echte Makro-Foto einer Holo-Karte aus dem Bestand setzen).

> Seamless loop. Extreme macro flyover across the surface of a holographic trading card, the embossed foil pattern filling the frame like an aerial landscape of iridescent dunes. A single band of rainbow interference light — rose, gold, mint, ice-blue in that order — sweeps slowly from left to right across the ridges, as if the card is tilting under one soft light source. Probe macro lens look, 100mm equivalent, f/4, razor-thin focus plane, specular highlights blooming gently. Deep navy vignette at the frame edges. Photorealistic, no particles, no sparkles, no fantasy elements, no text, no fingers. First and last frame identical; the light band completes exactly one full sweep.

### Real drehen

- Variante B ist Pflicht als zweiter Grid-Post: echtes iPhone in der Hand, Detail-Ansicht offen, Gyro-Permission getappt (Notiz 4), dann langsames Neigen — das `color-dodge`-Schimmern auf dem Screen wird direkt abgefilmt (nicht gecaptured — hier will man das echte Display mit Moiré-freiem 1/50-Shutter, weil die Hand-Bewegung zum Effekt gehört).
- Budget-Variante des KI-Shots: echte Holo-Karte auf einem Drehteller, LED-Leiste schwenkt — ein Nachmittag, ein Probe-Objektiv-Verleih. Ehrlich gesagt schlägt die reale Makro-Aufnahme die KI hier möglicherweise; Ray2 ist der Weg, wenn der „unmögliche" Flug in 2 mm Höhe über die Prägung gewünscht ist.

### Schnitt

Keiner — es ist ein Shot. Export als 8s-Loop, 4:5-Crop-Safe (alles Wichtige mittig), das SR-Monogramm klein und statisch unten rechts ab Sekunde 0.

### Sound

Silent-first designen (Grid-Autoplay ist stumm). Für die Reel-Verwendung: ein einzelner Ambient-Shimmer, Referenz **„Blade Runner 2049"-Ambient-Bett (Zimmer/Wallfisch)**, darunter feiner Vinyl-Crackle im Loop — der Crackle loopt hörbar nahtlos mit dem Bild.

### Text-Overlays

Keine. Das ist der Punkt.

---

## Konzept 4 (eigenes) — „Der Stapel" (20 s, 9:16)

**Einsatz:** TikTok/Reels-Performance-Spot; die Anti-These zu jedem „Sammlung zeigen"-Content der Konkurrenz. Collectr-Content zeigt Werte, die wachsen — SERO zeigt einen Stapel, der *verschwindet*.
**Dramaturgie:** Direkt aus der „ehrlichen Rechnung" (marketing-bausteine.md §3): 100 Karten, ~100 Minuten. Ein Timelapse, in dem der physische Stapel schrumpft, während der Portfolio-Wert im Bild-in-Bild rollt. Zeit wird zu Geld, sichtbar in einem einzigen Frame.

**Licht-Satz des Konzepts:** `cold blue-grey rim light from the right tracing every edge, one warm desk lamp glowing far behind, deep shadows`

### Shotliste

| Zeit | Shot | Quelle |
|---|---|---|
| 0–3 s | Low-Angle-Makro: der Stapel ragt wie ein Monolith auf, Kamera kippt nach oben | **Hailuo 02** |
| 3–15 s | Locked-off Timelapse: Hände scannen Karte um Karte, Stapel schrumpft; oben rechts PiP: Dashboard-Odometer klettert synchron | **Real** (Timelapse + Capture) |
| 15–17 s | Letzte Karte abgelegt, leerer Tisch, Phone liegt flach, Gesamtwert auf dem Screen | **Real** |
| 17–20 s | Endcard | **Post** |

### Prompt

**Shot 1 — Hailuo 02** (Hailuo, weil die Klammer-Regiekommandos die einzige zuverlässige Methode sind, eine exakte Tilt-Fahrt zu bekommen, und das Modell statische Objekte mit dramatischer Kamera nicht „mitanimiert" — der Stapel muss stehen wie ein Gebäude).

> [Tilt up, slow, ending in a slight push in] Extreme low-angle macro from tabletop height: a tall, slightly uneven stack of hundreds of trading cards towers like a monolith in a dark room. cold blue-grey rim light from the right tracing every edge, one warm desk lamp glowing far behind, deep shadows. 24mm lens at close focus, cinematic contrast, photorealistic, fine film grain. The stack stands perfectly still; only the camera moves. No text, no numbers, no hands, no faces, no card artwork in sharp focus.

### Real drehen

- Der Timelapse ist der Film: echte 100 Karten aus dem Bestand, ein Stativ, 45 Minuten Realzeit auf 12 Sekunden gerafft. Das Phone liegt sichtbar im Bild, die App läuft echt — jede Karte wird wirklich gescannt.
- Parallel läuft ein Screen-Recording des Home-Tabs mit; in Post als PiP oben rechts eingesetzt, Odometer-Rolls (`.h-value`) auf die Timelapse-Beats synchronisiert. Werte so planen, dass die Stellenzahl konstant bleibt (Notiz 3), sonst rollt nichts.
- Shot 3 real: derselbe Tisch, dieselbe Lampe wie im Hailuo-Shot — der Licht-Satz oben ist auch die Anweisung an den echten Set-Aufbau, damit KI-Opener und Real-Footage als ein Raum lesen.

### Schnitt

Harter Schnitt vom KI-Monolithen (3 s) in den Timelapse — gleiche Achse, gleiches Licht. Der Timelapse beschleunigt: erst 2 Karten/Sekunde, am Ende 6. Der Zähler-Overlay („100 … 63 … 27 … 0") läuft mit echten Zwischenständen, nicht linear — Ehrlichkeit auch im Rhythmus.

### Sound

Perkussion komplett aus echten Kartengeräuschen gebaut — Riffeln, Snaps, Ablegen auf Holz — zu einem Beat geschichtet, darunter ein Metronom-Tick. Referenz: **Teenage-Engineering-Produktfilme** (trocken, taktil, kein Musikbett von der Stange). Bei Sekunde 15 bricht alles ab: Stille, ein Raumton, das leise Odometer-Rattern.

### Text-Overlays

- 1 s: „100 Karten."
- 8 s: „~100 Minuten." *(Tilde ist Pflicht laut Wording-Notiz: Zeitwerte nie ohne Näherungszeichen)*
- 16 s: „0 Felder von Hand." *(Stat-Trio Variante 1, marketing-bausteine.md §2)*
- Endcard: „Der Stapel schrumpft, nicht dein Wochenende." + SR-Monogramm

---

## Die 5 Prompt-Techniken, damit KI-Video nicht nach KI-Slop aussieht

1. **Real-Plate-Anker statt Text-to-Video.** Jeder KI-Shot mit Objektbezug startet als Image-to-Video von einem echten Foto (Svens Karten, der echte Tisch, die echte Schachtel). Das Modell setzt nur Bewegung fort, statt Karten-Artwork zu erfinden — erfundene Karten erkennt die Zielgruppe (Sammler) in einem Frame. Kling und Luma halten das First Frame am treuesten; bei Luma zusätzlich Last Frame setzen.

2. **Eine Kamerabewegung pro Shot, physikalisch beschrieben.** „[Push in, slow]" plus Brennweite plus Blende — nie „epic cinematic 8k drone dolly zoom". Keyword-Salat erzeugt die typische schwimmende Alles-bewegt-sich-Optik, die jeder sofort als KI liest. Ein Shot, eine Bewegung, ein benanntes Objektiv; alles andere steht still (und das Stillstehen explizit in den Prompt schreiben: „The stack stands perfectly still; only the camera moves").

3. **Ein Licht-Satz pro Konzept, wortgleich in jedem Prompt.** Menschen erkennen KI-Montagen nicht an einzelnen Bildern, sondern daran, dass das Licht zwischen Shots springt. Deshalb hat oben jedes Konzept einen definierten Licht-Satz, der unverändert in jeden Prompt kopiert wird — und derselbe Satz ist die Aufbau-Anweisung fürs Real-Set, damit KI- und Real-Shots als ein Raum lesen.

4. **Null Text, null Zahlen, null UI in KI-Shots.** Jeder Prompt endet auf „no text, no numbers, no UI elements". KI-Typografie ist der schnellste Slop-Verräter — und ein halluzinierter Marktwert wäre bei dieser App zusätzlich inhaltlich fatal, weil echte Zahlen aus echten eBay-Verkäufen das Produktversprechen SIND. Alle Zahlen kommen aus Screen-Captures, alle Overlays aus der Post.

5. **10 Sekunden generieren, 2–4 verwenden — und alles unter ein Grade ziehen.** Modelle degradieren zum Ende der Generierung (Morphing, Drift); die besten Sekunden liegen nach dem Einschwingen. Deshalb ist oben kein KI-Shot länger als 4 s geschnitten. Im Finish ein gemeinsames Color-Grade (Navy-Schatten passend zu #102e5a) plus eine 35-mm-Grain-Ebene über *alle* Quellen — KI, Screen-Capture, Real. Die gemeinsame Körnung ist das, was die Mischung als ein Werk verkauft.

---

## Produktionsreihenfolge (Empfehlung)

1. **Konzept 3** zuerst — ein Shot, kleinstes Risiko, sofortiger Grid-Anker, und die Variante B testet nebenbei den Gyro-Holo-Dreh für alle anderen Konzepte.
2. **Konzept 1** als Nächstes — liefert Preview-Cut für den App-Store-Launch (Pflichtmaterial) und den ersten Reel.
3. **Konzept 4** — billigster Drehtag (Stativ + 100 Karten), höchstes TikTok-Potenzial.
4. **Konzept 2** zuletzt — teuerster Spot (zwei KI-Sequenzen mit Continuity-Anspruch), lohnt sich, sobald die ersten drei Reichweite belegt haben.

Referenzierte Dateien: `/Users/smorty/sero-app/web/sero.css`, `/Users/smorty/sero-app/web/sero.js`, `/Users/smorty/sero-app/web/index.html`, `/Users/smorty/sero-app/legal/marketing-bausteine.md`