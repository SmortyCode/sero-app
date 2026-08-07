# SERO — Social-Strategie für einen Solo-Founder mit 30–60 Min/Tag

**Ausgangslage, ehrlich eingeordnet:** SERO hat etwas, das 95 % aller App-Launches nicht haben — einen visuellen Kernmoment, der in unter 5 Sekunden erzählt ist (Karte vor die Kamera → Marktwert steht da), und einen Gründer mit echtem Warenbestand und echten Verkäufen (eBay-Store seromunich). Dazu sind mehrere UI-Momente bereits kamerafertig gebaut: der Gyro-Holo-Effekt auf Karten (`sero.js:3354–3396`, `.holo-wrap` in `sero.css:653 ff.`), die Meilenstein-Feier bei 10/25/50/100 Stücken (`sero.js:239–251`), der Kerzen-Chart (`candleChart()`, `sero.js:621`) und das PSA-10-Delta „Grading könnte sich lohnen: ~+{X} bei PSA 10*" (`sero.js:3188`). Nichts davon muss für Social gebaut werden — es muss nur gefilmt werden. Das ist die ganze Strategie in einem Satz: **Bildschirm abfilmen, was die App ohnehin tut, und die echten Karten aus dem eigenen Lager als Darsteller nehmen.**

---

## 1. Plattform-Priorität

| Prio | Plattform | Rolle | Zeitanteil |
|---|---|---|---|
| 1 | **TikTok** | Reichweite und Discovery | Hauptkanal, 1 Video/Tag |
| 2 | **Instagram Reels** | Käuferschicht 25–45, DMs, Mail-Day-Kultur | Crosspost desselben Videos + Stories |
| 3 | **YouTube Shorts** | Suchintention („Pokémon Karte Wert herausfinden") | Crosspost, null Extra-Aufwand |
| 4 | **X** | Build-in-Public, Indie-/Founder-Szene, später US-Karten-Szene | 10 Min/Tag, nur Text |

**Begründung:**

- **Die deutsche Karten-Szene lebt in Kurzvideo und in geschlossenen Räumen.** Pull-Videos und Slab-Content laufen auf TikTok organisch auch ohne Follower-Basis — genau das braucht ein Account bei null. Instagram ist, wo die kaufkräftigen Sammler (Mail-Days, Vitrinen-Posts, Grading-Reveals) tatsächlich sitzen und wo DMs zu Gesprächen werden. Die geschlossenen Räume (Facebook-Gruppen, Discords, WhatsApp-Gruppen, WhatNot-Streams) sind keine Posting-Kanäle, sondern Community-Kanäle — dazu unten in Abschnitt 4.
- **Ein Video, drei Plattformen.** Bei 30–60 Min/Tag ist das die einzige Rechnung, die aufgeht: Das 9:16-Video wird einmal gedreht und nativ (ohne TikTok-Wasserzeichen) auf TikTok, Reels und Shorts gestellt. Shorts ist dabei der stille Gewinner: „Was ist meine Karte wert"-Suchanfragen landen bei YouTube/Google, und Shorts ranken dort dauerhaft — das ist der einzige Kanal mit Long-Tail.
- **X ist keine Reichweiten-Plattform für die deutsche Karten-Szene** — die ist dort kaum vorhanden. X hat zwei andere Jobs: (a) Build-in-Public dokumentieren (Indie-Hacker-Szene ist dort, und die teilt gern Founder-Stories mit echten Zahlen), (b) Brückenkopf für die spätere US-Phase, denn die englischsprachige Hobby-Szene (PSA, Card-Twitter) ist auf X groß. 10 Minuten am Tag, keine Videos.
- **Bewusst NICHT bespielen:** eigene Facebook-Page, LinkedIn, eigener Discord-Server (ein leerer Server ist ein Todessignal — beitreten ja, gründen nein), Threads/Bluesky. Streichen spart die Stunde, die die vier Kernkanäle brauchen.

---

## 2. Die fünf Content-Formate

### Format 1 — „Karte → Wert in 3 Sekunden" (der Scan-Moment) — Pflichtformat, Rückgrat des Kanals

Das geborene Kurzvideo, weil der komplette Story-Arc (Frage → Antwort) in der App selbst passiert. Setup: Handy 1 filmt von schräg oben Hand + Karte + Handy 2, auf dem die App läuft. Kein Schnittprogramm nötig, CapCut reicht.

**Beispiel-Skript (15–20 Sek):**
- **Hook (Sek. 0–1):** Karte wird ins Bild gehalten, Gesicht unsichtbar. Gesprochen oder als Text-Overlay: „Die lag 3 Jahre in meiner Schublade. Pass auf."
- **Ablauf (Sek. 1–10):** Tipp auf den Scan-Orb (die `.tab-cam`-Animation mit dem Scale/Rotate beim Antippen, `sero.css:219`, ist selbst schon ein befriedigender Mikro-Moment — draufhalten). Foto. Laden — hier trägt der App-eigene Ladetext „Echte Verkäufe statt Wunschpreise." (marketing-bausteine.md §5) das Video inhaltlich. Dann Ergebnis: „Erkannt"-Badge, Kartenname, Marktwert. Kurze Stille, damit die Zahl wirkt.
- **Payoff (Sek. 10–15):** Zoom auf den Kerzen-Chart oder den Gyro-Holo-Effekt (Handy kippen, Karte glänzt — `holo-wrap.tilting`). Text-Overlay: „Marktwert aus echten eBay-Verkäufen. Nicht aus Wunschpreisen."
- **CTA:** „Welche Karte soll ich als Nächstes scannen — schreib sie in die Kommentare." (Kommentar-CTA schlägt Link-CTA in den ersten Wochen; der App-Link steht in der Bio.)

**Serienlogik:** Abwechselnd billige Karte („Die hier ist 80 Cent wert — und das ist okay") und teurer Slab. Die Ehrlichkeit bei Cent-Karten ist das Differenzierungsmerkmal gegen die Hype-Accounts und zahlt direkt auf „echte Verkäufe" ein.

### Format 2 — „Wunschpreis vs. Marktwert" (Stitch/Duett-Format)

Reagiert auf existierende Pull-Videos und Kleinanzeigen-/eBay-Fundstücke — nutzt fremde Reichweite, kostet nichts.

**Beispiel-Skript (20–30 Sek):**
- **Hook (Sek. 0–1):** Stitch eines Pull-Videos, eingefroren auf der gezogenen Karte. Overlay: „Er glaubt, das sind 300 €."
- **Ablauf:** Sven scannt dieselbe Karte (oder legt sie manuell an), zeigt den Marktwert aus verkauften Angeboten, dann den Kerzen-Chart mit dem echten Verlauf. Eine Zeile Einordnung: „Aktive Angebote wollen 300. Verkauft wurde zuletzt für 180."
- **CTA:** „Schick mir dein teuerstes Pull, ich sag dir, was es wirklich wert ist."

Wichtig: nie höhnisch, immer nüchtern — die Szene feiert Entlarvung von Wunschpreisen, aber sie bestraft Arroganz. Der Ton der App („Echte Verkäufe statt Wunschpreise") ist genau richtig.

### Format 3 — „Slab des Tages" mit seromunich (Beweis-Format) — Pflichtformat

Svens unfairer Vorteil: Er behauptet nicht, dass die App verkauft — er zeigt es mit echtem Bestand und echten Verkäufen. Kein Konkurrent kann das kopieren, Collectr schon gar nicht.

**Beispiel-Skript (20–30 Sek):**
- **Hook (Sek. 0–1):** Slab in der Hand vor dem Lager/Kartenbestand: „Ich bin eBay-Händler. Das hier liste ich jetzt in einer Minute."
- **Ablauf:** Scan des Slabs — die App liest Label, Note und Zert-Nummer (der Ladetext „Label wird gelesen, Note folgt gleich." läuft mit). Marktwert erscheint. Dann der eine Tipp: Listing entsteht — Titel, Pflichtfelder, Preis, freigestelltes Foto. Schnitt auf das echte, live stehende eBay-Listing im seromunich-Store.
- **Payoff (Folge-Video Tage später):** „Verkauft."-Screenshot mit echtem Verkaufspreis, dann Einpacken des Stücks (Mail-Day-Ästhetik rückwärts — die Szene liebt Pack-Videos). Overlay: „Der Stapel schrumpft, nicht dein Wochenende." (marketing-bausteine.md §5, Zeile 5 — fertiger Claim, einfach benutzen.)
- **CTA:** „Alles, was ich hier liste, findest du unter seromunich auf eBay." — Doppel-Effekt: Das Format verkauft App UND Ware gleichzeitig.

**Königsvideo dieses Formats:** die „Ehrliche Rechnung" aus marketing-bausteine.md §3 als Challenge verfilmt — „100 Karten, 100 Minuten": Timer an, Stapel abarbeiten im Zeitraffer, Timer-Stand am Ende neben dem Entwürfe-Tab im Verkauf-Bereich (`salesSeg`, index.html:126). Das ist das Video, das gepinnt wird.

### Format 4 — Build-in-Public auf X — Pflichtformat

Kein Video, reiner Text mit Screenshots. Zielgruppe hier nicht Sammler, sondern Founder/Indie-Szene — die bringt Erstnutzer, Feedback und später Presse/Newsletter-Erwähnungen.

**Beispiel-Posts (Ton: nüchtern, Zahlen, keine Ausrufezeichen — der SERO-Ton funktioniert auf X ausgezeichnet):**
- Pinned Thread: „Ich verkaufe seit Jahren Sammelkarten auf eBay. Jedes Listing von Hand: ~8 Minuten. Ich habe mir eine App gebaut, die es in einer macht. Thread über den Bau, die Zahlen und alles, was schiefgeht."
- Wöchentlich, festes Gerüst: Scans diese Woche / neue Nutzer / gelistete Stücke über die App / ein Learning / ein Fail. Screenshots direkt aus der App (der „Testmodus"-Badge `dryBadge` auf Dev-Screenshots ist dabei kein Makel, sondern Authentizitäts-Signal).
- Feature-Momente als Mini-Clips: der Gyro-Holo-Effekt und die Meilenstein-Feier sind exakt die Sorte UI-Detail, die auf Build-in-Public-X überdurchschnittlich geteilt wird.
- **CTA:** keiner nötig — Bio-Link reicht. Konsistenz ist der CTA.

### Format 5 — „Grading-Orakel" (Spannungsbogen-Format)

Nutzt den Grading-Bereich der App (`sero.js:3170–3193`): KI-Einschätzung „PSA {X} ({low}–{high}) · Sicherheit", PSA-10/PSA-9-Median aus aktiven Angeboten und das Lohnt-sich-Urteil inkl. ~25 € Gebühr.

**Beispiel-Skript, Teil 1 (15 Sek):**
- **Hook (Sek. 0–1):** Raw-Karte im Toploader: „Grading kostet 25 €. Lohnt sich das hier überhaupt?"
- **Ablauf:** Scan → App zeigt „Grading könnte sich lohnen: ~+140 € bei PSA 10*" samt PSA-10/PSA-9-Medianen. Verdict-Overlay: „Die App sagt: einschicken." Bei einer zweiten Karte im selben Video das Gegenteil: „Grading lohnt bei dieser Karte eher nicht" — das Nein macht das Ja glaubwürdig.
- **Teil 2 (Wochen später, der eigentliche Payoff):** Grading-Reveal — das erfolgreichste Format der gesamten Karten-Szene — gegen die damalige KI-Einschätzung geschnitten. „Die App hat PSA 9 getippt. Kam zurück als …" Trifft die Einschätzung, ist es der stärkste Beweis-Content überhaupt; liegt sie daneben, ist es ehrlicher Content mit garantierter Kommentar-Debatte. Beides gewinnt.
- **CTA:** „Zeig mir deine Grading-Kandidaten in den Kommentaren."

---

## 3. Vier-Wochen-Launch-Kalender

**Tagesrhythmus (Mo–Fr, ~35 Min):** 5 Min vorbereitetes Video auf TikTok + Reels + Shorts stellen · 15–20 Min Kommentare und DMs beantworten (in Woche 1–4 wichtiger als neuer Content) · 10 Min X-Post. **Sonntag (60–90 Min): Batch-Dreh** — 5–6 Scan-Videos am Stück, gleiches Setup, nur Karten wechseln. Ohne Batching kippt der Plan, mit Batching ist er bequem.

**Woche 1 — Fundament und erste Scans**
- Handles sichern (einheitlich, z. B. sero.app o. ä.), Bio überall gleich: Positionierung + App-Link. Profilbild: SR-Monogramm auf Navy.
- Sonntag: 6 Videos Format 1 drehen (3 Slabs, 3 Bulk-Karten — Spannbreite zeigen).
- Mo–Fr: täglich 1 Scan-Video. Auf X der Pinned-Thread (Format 4) plus 3 kurze Posts.
- Abends passiv: den 3–5 größten deutschen Pokémon-/TCG-Facebook-Gruppen und 2 Discords beitreten — nur lesen, Ton lernen, noch nichts posten.

**Woche 2 — Beweis-Schiene und erste Reaktionen**
- Batch: 3× Format 1, 2× Format 3 („Slab des Tages" mit echtem Listing), 1× Format 2 (erster Stitch).
- Kommentar-Wünsche aus Woche 1 einlösen („Ihr wolltet wissen, was X wert ist") — das trainiert die Community aufs Mitmachen.
- X: erster Wochenreport mit echten Zahlen, egal wie klein. „12 Nutzer, 340 Scans" schlägt jede Ankündigung.
- Erste „Verkauft"-Folge zu einem Slab-des-Tages-Video, sobald ein Stück weggeht.

**Woche 3 — Grading-Bogen öffnen und Community-Formate**
- Batch: 2× Format 1, 1× Format 2, 2× Format 5 Teil 1 (Karten real zum Grading einschicken — der Reveal in Woche 6–8 ist damit vorproduziert), 1× Format 3.
- „100 Karten, 100 Minuten"-Video drehen (Aufwand ~2 Std., zählt als Sonntags-Batch) — noch nicht posten, es ist das Launch-Video für Woche 4.
- In den Facebook-Gruppen erstmals aktiv: ausschließlich hilfreiche Wert-Antworten als Person (siehe Abschnitt 4, Idee 1). Kein App-Link ungefragt.

**Woche 4 — Launch-Woche**
- Montag: „100 Karten, 100 Minuten" auf allen Kanälen, pinnen. X-Thread mit der ehrlichen Rechnung inkl. Fußnoten-Logik aus marketing-bausteine.md §3 — genau diese Transparenz („die Minute ist ehrlich hoch gegriffen") performt in der Founder-Szene.
- Di–Fr: tägliches Bestformat im Wechsel, dazu die 30-Tage-Challenge starten (Abschnitt 4, Idee 3).
- Freitag: Recap auf X — 4 Wochen, alle Zahlen, Learnings. Dieser Post ist erfahrungsgemäß der reichweitenstärkste des Monats.
- Danach: Erfolgsformate weiterfahren, Flops streichen. Nach 4 Wochen zeigen die Daten, welche 2 Formate tragen — dann Fokus.

---

## 4. Drei unkonventionelle Growth-Ideen ohne Werbebudget

**Idee 1 — Der Wertcheck-Mensch werden.** In jeder großen Karten-Facebook-Gruppe und in Foren wird täglich mehrfach gefragt: „Was ist die wert?" — und die Antworten sind meist Bauchgefühl. Sven beantwortet 3–5 solcher Posts pro Tag (10 Minuten) mit echten Verkaufsdaten: Karte, letzter Verkaufsmedian, Spanne, nüchtern formuliert. Nie mit App-Link — der steht im Profil. Nach vier Wochen ist er in diesen Gruppen „der mit den echten Zahlen", und die Frage „womit machst du das" kommt von allein — die einzige Form von Werbung, die in Gruppen mit Selbstpromo-Verbot funktioniert. Skaliert später ins eigene Format: „Schick mir deine Karte" als Dauerrubrik, jede Einsendung ist Content und Lead zugleich.

**Idee 2 — Das Scan-Duell.** Versus-Format mit eingebauter Beteiligung: Ein Sammler (später: andere Creator, WhatNot-Streamer) schätzt den Marktwert einer Karte, dann scannt SERO — Mensch gegen Marktdaten, Punktestand über die Serie. Das Format ist duettierbar (TikTok-Mechanik: andere schätzen mit, bevor sie weiterwischen) und der natürliche Türöffner zu kleinen deutschen Breakern: Sven bietet 5–10 von ihnen freie Pro-Zugänge an, sie nutzen die App live im Stream für Sold-Comps — der Streamer bekommt schnellere Preisantworten on air, SERO steht ohne einen Cent Budget dauerhaft im Bild. WhatNot-Streamer sind für so etwas ansprechbar, weil Preisfindung live ihr tägliches Problem ist.

**Idee 3 — Die Meilenstein-Challenge.** Die App feiert 10/25/50/100 Stücke bereits mit einem eigenen Celebration-Screen (`sero.js:239–251`) und rechnet in „Deine ehrliche Rechnung" (marketing-bausteine.md §4) die persönliche Zeitersparnis vor („Du hast 100 Stücke erfasst. Von Hand wären das 13 Stunden gewesen — mit SERO waren es 100 Minuten."). Beides sind geborene Screenshot-Momente. Daraus wird die 30-Tage-Challenge „100 Stücke in 30 Tagen": Teilnehmer posten ihre Meilenstein-Screens unter einem Hashtag, Sven repostet jeden Einzelnen in den Stories und kürt am Ende die größte gelistete Sammlung. Kostet nichts, erzeugt UGC von fremden Accounts (das wertvollste Signal für den Algorithmus) und treibt exakt die Kernmetrik der App: erfasste und gelistete Stücke. Der Sammler-Fortschritt mit Stufen und Set-Lücken (`progressCard`, `sero.js:2704 ff.`) liefert für spätere Runden weitere Challenge-Varianten („Schließ eine Set-Lücke").

---

## 5. Was NICHT tun — die fünf typischen Fehler

1. **Als Marke in fremde Communities platzen.** App-Link-Drops in Facebook-Gruppen, Reddit oder Discords führen zu Bann und — schlimmer — zu Szene-Gedächtnis. Die Karten-Szene ist klein und nachtragend. Reihenfolge ist nicht verhandelbar: erst Person mit hilfreichen Antworten (Idee 1), Tool nur auf Nachfrage.
2. **Features bewerben statt Karten zeigen.** „KI-Erkennung, 4 Grader, automatische Pflichtfelder" ist ein Screenshot, an dem jeder vorbeiwischt. Die Karte ist der Star, die App der Nebendarsteller, der schneller ist als erwartet. Jedes Video beginnt mit einer Karte und einer Frage — nie mit der App. Feature-Kommunikation gehört in die App selbst (dafür existiert das Stat-Trio aus marketing-bausteine.md §2 bereits).
3. **Launch ankündigen statt launchen.** „Bald verfügbar"-Teaser-Wochen funktionieren nur mit bestehender Audience — bei null Followern sind sie tote Posts. Ebenso Gewinnspiele am Anfang: Giveaways ziehen Giveaway-Accounts an, keine Sammler, und die ersten 1000 Follower bestimmen, wem der Algorithmus die Videos zeigt. Der Launch ist das erste Scan-Video, nicht ein Countdown.
4. **Auch nur einmal mit Hype-Preisen erwischen lassen.** Das gesamte Vertrauensfundament von SERO ist „Marktwert aus echten Verkäufen". Ein einziges Video, in dem ein aktiver Wunschpreis als Wert verkauft wird, und die Kommentare zerlegen die Kernbehauptung — unrettbar, weil es das eigene Versprechen bricht. Deshalb in jedem Video die Quelle mitsprechen („laut verkauften Angeboten") und Wertaussagen mit „~" führen, wie es die Wording-Notizen (marketing-bausteine.md, Zeile 135) ohnehin vorschreiben. Gleiches gilt für inszenierte Pulls — die Szene erkennt gestellte Reveals sofort.
5. **Den Kanal-Zoo aufmachen und Kommentare liegen lassen.** Sechs Plattformen halbherzig plus ein leerer eigener Discord sind bei 30–60 Min/Tag der sichere Tod. Vier Kanäle, ein Video, und die knappe Zeit in Antworten investieren: In den ersten Monaten ist jede beantwortete Wertfrage im Kommentarbereich wertvoller als ein zusätzliches Video — Antworten sind auf TikTok eigener Content (Video-Reply auf Kommentar ist das halbe Format 1) und das stärkste Ranking-Signal, das ein kleiner Account hat.

---

**Zum Schluss die ehrliche Einordnung:** Die App braucht für diese Strategie keine einzige neue Funktion. Scan-Orb-Animation, Gyro-Holo, Odometer-Werte, Kerzen-Chart, Meilenstein-Feier und das Grading-Delta sind fertiges Filmmaterial; die Marketing-Bausteine liefern die Claims wortfertig („Echte Verkäufe statt Wunschpreise", „Der Stapel schrumpft, nicht dein Wochenende", die ehrliche Rechnung). Der Engpass ist ausschließlich Dreh-Routine — und die löst der Sonntags-Batch.