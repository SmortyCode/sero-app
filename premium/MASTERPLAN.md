# SERO Masterplan — Vom fertigen Produkt zur sichtbaren Marke

**Creative-Director-Verdichtung aus 8 Spezialisten-Reports. Stand: 02.08.2026.**

Die eine Erkenntnis, die alle Reports teilen: **SERO ist zu ~70 % fertig gebaut — auch das Branding und das Marketing.** Kerzen-Chart, Odometer, Gyro-Holo, tabular-nums, die „ehrliche Rechnung" und die Wording-Regeln sind bereits die Marke. Der Plan erfindet nichts Neues; er repariert 5 sichtbare Defekte, spitzt zu, was da ist, und filmt es dann ab.

**Rollenlegende:** [C] = Claude macht es sofort (Code, SVG, Texte) · [S+P] = Sven mit fertigem Prompt in Bild-/Video-KI · [S🎥] = Sven filmt real (~1 Std) · [S] = Sven manuell (Accounts, Uploads)

---

## Aufgelöste Widersprüche (Entscheidungen mit Begründung)

| # | Konflikt | Entscheidung | Begründung |
|---|---|---|---|
| 1 | **Branding: Richtung A (Studio) vs. B (Foil) vs. C (Vault) vs. D (Kurszettel)** | **D „Der Kurszettel"**, B als Produkt-Moment integriert, C geparkt als späteres Premium-Tier, A verworfen | D ist die Positionierung wörtlich („macht sie zu Geld"), zu ~70 % schon im Code (Kerzen-Chart, Odometer, tabular-nums, „Echte Verkäufe statt Wunschpreise"), als einzige Richtung von einem Solo-Founder content-seitig durchhaltbar (Templates statt Foto-Produktion) und die einzige mit Kontrastkante gegen Collectr. A streicht das „Wow" (widerspricht Svens Stil „Clean + Wow"), B verkauft Behalten statt Verkaufen, C schließt die Zielgruppe 16–45 aus. Der Gyro-Holo bleibt trotzdem Signature — „der Kurszettel ist die Haltung, die Holo ist die Ware". |
| 2 | **Odometer im Scan-Ergebnis: reparieren (Interaktions-Audit) vs. umgehen (KI-Video-Report: „Dashboard filmen statt Sheet")** | **Reparieren.** Der 1-Zeilen-Fix (`state.anim["scanres_"+item.id] = 0` vor `countUp`, sero.js:330) kommt VOR jedem Dreh. | Der KI-Video-Workaround dokumentiert einen Bug, kein Design. Nach dem Fix rollt die wichtigste Zahl der App an ihrer wichtigsten Stelle — im Produkt UND in jedem Video. Die Dreh-Notiz „Dashboard filmen" entfällt damit. |
| 3 | **Icon: Logo-Report empfiehlt Weg A (geometrisches SR), Branding-Report will „Monogramm mit Kurs-Tick"** | **Synthese:** Weg A (monolineares, geteiltes SR) als neue Primärmarke, dessen auslaufender Endstrich als Kurs-Tick-Variante getestet (D-Idee). Weg C (Slab) als Sofort-SVG für Favicon/Mono-Stellen, weil in 1–2 Std baubar. Kalligrafisches SR wird Heritage-Marke (Login, Splash, Marketing groß). | Beide Reports wollen dasselbe: monolinear, klein lesbar, Tinted-tauglich. Der Kurs-Tick ist nur ein Terminal-Detail auf Weg A — kein Widerspruch, eine Verschmelzung. |
| 4 | **Sound: Interaktions-Audit sagt „kein Sound in der App", Video-Report baut aufwendige Sound-Designs** | Beides gilt — verschiedene Medien. **In der App: kein Sound** (Stapel-Scanner, iOS-PWA-Audio fragil), einzige spätere Ausnahme opt-in beim „Verkauft"-Push. **In Videos: ja** (Apple-Titanium-Referenz, Kartengeräusch-Perkussion). Belohnung in der App läuft über Haptik — die mit dem `haptic()`-Wrapper Capacitor-ready wird (alle 3 `navigator.vibrate` sind auf iOS heute No-Ops). |
| 5 | **Onboarding-Tour: 3 Karten (Ist) vs. 1 Karte (Interaktions-Audit) vs. Wording-Set A (Benchmarks)** | **1 Tour-Karte** mit Set-A-Wording (Fotografieren / Bestätigen / Listen), Endbutton öffnet die Kamera **synchron** (`$("cameraInput").click()` im Click-Handler — der Tap ist der iOS-Gesten-Kontext, das heutige `setTimeout(300)` verschenkt ihn). „Verkaufen"- und „Wert"-Botschaften wandern als Einmal-Hinweise in den Kontext (erste Ergebnis-Karte / erster Sammlungs-Besuch). Erlebnis schlägt Erklärung: 2 Tipps bis zur Kamera statt 6–7. |
| 6 | **Reihenfolge: Token-Konsolidierung (UI-Audit, strukturell) vs. Mikro-Interaktionen (Interaktions-Audit, sichtbar)** | Erst die **sichtbaren Defekte und Kernmomente** (Tage 1–3), dann die Token-Konsolidierung (strukturell, macht künftige Arbeit billiger, aber kein Nutzer sieht sie direkt). Ausnahme: die fehlenden CSS-Klassen sind Defekte, keine Politur — die kommen ganz nach vorn. |

---

# PHASE 1 — Premium-UI/UX (Woche 1–2)

## ⭐ Größter Hebel: Das Scan-Moment-Paket

**Warum zuerst:** Der Scan ist Kernprodukt, Wow-Moment UND das Filmmaterial für Phase 3. Jede Stunde hier zahlt dreifach.

**1.1 [C] [S] Odometer-Seed-Fix** — `state.anim["scanres_"+item.id] = 0;` vor `countUp(el, "scanres_"+item.id, val)` (sero.js:330). Eine Zeile, repariert den Kernmoment.

**1.2 [C] [S] Reveal-Choreografie in `showScanResult`** (sero.js:303–339) — gestaffelt statt gleichzeitig, nur CSS-`animation-delay` + `backwards`:

| Zeit | Element | Animation | Haptik |
|---|---|---|---|
| 0 ms | Karte | `rise .5s var(--spring)` (bestehend) | — |
| 120 ms | Foto | `resPop .7s var(--spring)` + Delay | leicht (10 ms) |
| 300 ms | Badge „Erkannt" | Scale .6→1, `.35s var(--spring)` | — |
| 400 ms | Name + Sub | TranslateY 8px + Fade, `.3s ease-out` | — |
| **550 ms** | **Marktwert** | Odometer rollt von 0, `.65s cubic-bezier(.35,1.05,.35,1)`, Radial-Glow-Puls `.6s` | mittel `[12,50,20]` — Vibrate von Z. 338 in `setTimeout(…,550)` verschieben |
| 900 ms | Quelle + Buttons | Fade `.25s` | — |

Gesamt ~1,15 s. `prefers-reduced-motion` über bestehenden countUp-Fallback + einen `@media`-Block.

**1.3 [C] [S] `--muted`-Fix** — sero.css:1219 `color: var(--muted)` (Variable existiert nicht!) → `var(--label-2)`. Betrifft den Untertitel im Scan-Ergebnis.

**1.4 [C] [S] `haptic()`-Wrapper, Capacitor-ready:**
```js
function haptic(kind = "light") {
  const H = window.Capacitor?.Plugins?.Haptics;
  if (H) { kind === "success" ? H.notification({type:"SUCCESS"}) : H.impact({style: kind.toUpperCase()}); return; }
  if (navigator.vibrate) navigator.vibrate({light:10, medium:[12,50,20], success:[10,60,18]}[kind] || 10);
}
```
Bestehende 3 Aufrufe (sero.js:263, 338, 2119) umstellen; neue Stellen: `celebrate()` (der Geld-Moment hat heute null Haptik!), PTR-85px-Schwelle, Long-Press, Favoriten-Erfolg, Wert-Reveal.

## Weitere Maßnahmen in Reihenfolge

**1.5 [C] [S] Die 5 fehlenden CSS-Klassen (echte Defekte — das JS rendert Unstyled-HTML):**
- `.gstat` + `.gmore` (Sammlung-Kacheln, sero.js:2061/2082): Status-Punkt-Styling (`.gstat.live` grün, `.draft` orange, `.sold` label-3, `.wish #e0518e`), `.gmore` absolut top:10px right:10px mit `rgba(16,24,40,.45)` + blur(4px), `.gfav` auf top:42px (Überlappung!)
- `.login-feats` / `.lf-ic` / `.login-link` (index.html:32–36/50): der erste Screen jedes Nutzers zeigt heute Browser-Bullet-Points
- Swipe-Reveal `.rev-fav`/`.rev-more`/`.armed` (sero.js:2113–2118): Minimal-Fix als Ring am Objekt (`box-shadow: 0 0 0 3px color-mix(in srgb, var(--tint) 45%, transparent)`, Favorit mit `#f5a623 55%`)

**1.6 [C] [S] Onboarding-Verdichtung** (siehe Widerspruch 5): 1 Tour-Karte, Kamera-Direktstart, Kontext-Hinweise per localStorage-Flag. Erster Marktwert < 1 Minute ab App-Start.

**1.7 [C] [S] Paywall personalisieren** (Benchmarks #3 — höchster Conversion-Hebel bei kleinstem Aufwand): `openPaywall` (sero.js:268–294) zeigt statische 100-Karten-Rechnung, obwohl `state.dash.time_saved` mit echten Werten danebenliegt. Ab `items >= 3` die persönliche Rechnung oben (Texte fertig in marketing-bausteine.md §4), 100-Karten-Rechnung als Fußnote. Flighty-Prinzip: die eigene Rechnung ist ein Kontoauszug, keine Werbung.

**1.8 [C] [S] Quick-Fix-Sammlung (zusammen < 1 Tag):**
- Toten Dark-Token-Block sero.css:1625–1669 löschen (sero-dark.css ist die Wahrheit); `#2c5aa5` → `#2b5ea8` (ein Dark-Blau statt drei)
- `.big-chart` Dark: `#4fd1b5` (Fremdkörper-Teal) → `var(--tint)`
- `.ov-delta` auf `color-mix` mit `--green`/`--red` (entfernt 8 Hardcodes in 3 Dateien)
- `.gname` `min-height: 0` löschen (überschreibt 2.6em — Grid-Reihen ungleich)
- `.gph-none` `4.5%/3.2%` → `10px`, `.gitem::after` 12→10px
- JS-Inline-Farben: `#8e8e93` (6×) → `.ric.neutral { background: var(--label-3) }`, `#34a853` (Google-Grün, 3×) → `var(--green)`
- index.html: doppelte `apple-mobile-web-app-capable`-Metas, 2 Touch-Icons, widersprüchliche `status-bar-style` — je eins behalten
- `#salesSeg { }` leer, `!important` am Scan-Orb raus

**1.9 [C] [M] Mikro-Interaktions-Politur:** Optimistic UI für Stern/Alarm (`detailFav` sero.js:3040 togglet sofort, Server im Hintergrund, Fehler rollt zurück); Tab-Retap = Scroll-to-top statt Animations-Replay; Sheet-Snapback (`.snapback { transition: transform .35s var(--spring) }`); PTR-Arm-Zustand (rotate, Scale 1.15 bei >85px + haptic); Long-Press `.pressing` (Scale .97/450ms); Favoriten-Stern-Pop am Objekt; `res-grip` ehrlich machen (Griff-Logik oder weg); `closeDetail`-Timeout 260→290ms.

**1.10 [C] [M] Analyse-Bühne im Scan-Tab** (ersetzt den 47px-Spinner beim wichtigsten Warten): geparktes Foto ~180px mittig, Scan-Strahl (`@keyframes scanbeam`, 1600ms alternate), Fortschrittsring aus echtem `avg_scan_s` (füllt auf 90 %, springt bei Ergebnis auf 100 %), `status_text`-Crossfade 250ms mit den 5 fertigen Ein-Zeilern aus marketing-bausteine.md §5 („Echte Verkäufe statt Wunschpreise." als Ladetext = Positionierung im Wartemoment). Scan 1–2 ohne Messdaten: „dauert meist unter einer Minute".

**1.11 [C] [M] `celebrate()`-Upgrade** — der Peak des Produktversprechens („Live auf eBay") zeigt den Preis heute statisch: `countUp` von 0 (Key `party_<draft_id>`), `party-ring` als SVG-Draw (`cubic-bezier(.22,.9,.16,1)`), `haptic("success")`, optional 10–12 Partikel Navy/Gold CSS-only. Ein Effekt, kein Konfettiregen.

**1.12 [C] [L] Token-Konsolidierung** (strukturell, nach den sichtbaren Fixes):
- **Typo:** 30 → 9 Größen: `--fs-hero:40 / -large:32 / -title:26 / -heading:21 / -body:17 / -sub:15 / -footnote:13 / -caption:11.5 / -micro:10`; Gewichte 500/600/700/800 (650→600, 750/760→700). Inputs nie <16px (iOS-Zoom).
- **Spacing:** `--sp-1..6: 4/8/12/16/20/24`, die 41 ungeraden Ausreißer runden
- **Radius:** `--r-thumb:10 / -control:12 / -row:14 / -card:20 / -bar:26 / -sheet:34`
- **Schatten:** 5 Token statt ~24 (`--shadow-card` existiert; + `-photo`, `-float`, `-modal`, `-micro`)

**1.13 [C] [L] Signature-Interaktionen (Ende Phase 1 / Puffer):** Robinhood-Chart-Scrub (`touchmove` auf `.big-chart`, `.ov-value`/`ohlcLegend` laufen mit, Veil-Modus respektieren) und Swipe-to-List (Swipe links auf Listenzeilen → `listNow(id)`, sero.js:2313 — die Positionierung als Geste, kein Konkurrent kann sie anbieten). View-Transitions (Airbnb-Shared-Element) und Share-Karte danach.

---

# PHASE 2 — Branding & Identität (Woche 2–3, parallel zu Phase 1 startbar)

## ⭐ Größter Hebel: Richtung D „Der Kurszettel" festzurren + IBM Plex Mono an 3 Stellen

**Warum zuerst:** Diese eine Entscheidung steuert Icon, Store-Assets, Social-Templates und Ton. Und der Test kostet fast nichts: **[C] [S]** `IBM Plex Mono` nur an Wert-Ziffern (`h-value`, `v-main`), Odometer und Chart-Legende — der Rest bleibt SF. Keine neue Farbe: Navy `#102e5a` bleibt Anker, Grün `#0da05a`/Rot `#e5484d` (dunkel `#30d158`/`#ff453a`) werden streng semantisches Marken-Vokabular, Graphit `#8b93a1` für Sekundäres. Gyro-Holo und Glanz-Lauf bleiben unangetastet (Sammlerherz-Grenze gegen Finance-Kälte).

## Weitere Maßnahmen in Reihenfolge

**2.1 [C] [S] Sofort-SVG: Weg C (Slab-Marke)** — `<rect rx>` + `<line>` + `<circle>` + S-Pfad, alles monolinear `stroke="currentColor"` (1–2 Std). Sofort einsetzen an den drei Stellen, wo das kalligrafische SR heute objektiv unlesbar ist: Favicon (aktuell skaliert das 1024er-PNG auf 16px — schlechtester Fall), `.cam-mono` (30px, sero.css:220), `.ptr-mono` (20px, Z. 1599). Nebeneffekt: Dark-Mode-Filter-Hack (`brightness(0) invert(1)`) entfällt, iOS-18-Tinted gratis.

**2.2 [S+P] [M] Neue Primärmarke: Weg A (geteiltes SR, monolinear)** — Formfindung per Midjourney, Gewinner von Claude als sauberes SVG nachgebaut (24×24-Raster, Strichstärke 2,5 E ≈ 107px/1024, G2-Übergänge, Terminals 90° mit 0,4-E-Radius). Kurs-Tick-Terminal als D-Variante mittesten. Prompt:

> Minimalist geometric monogram logo, the letters S and R sharing one continuous stroke, the right side of the S flows seamlessly into the stem of the R, monoline uniform stroke weight, subtly rounded stroke terminals, pure white mark centered on deep navy blue #102e5a background, flat vector app icon style, generous negative space, Swiss graphic design, no gradients, no shadows, no additional text --ar 1:1 --v 6 --style raw --no calligraphy, serif, ornament

Fallback-Test, falls das Store-Regal mehr Auffälligkeit braucht — Weg B (Karten-Silhouette 63:88, +7°/−6°, hintere Karte `#24508f`):

> Flat vector iOS app icon: a white trading card with rounded corners (aspect ratio 63:88) tilted 7 degrees clockwise, a bold geometric navy letter S centered on the card, a second card in lighter blue #24508f peeking out behind it rotated -6 degrees, one subtle diagonal light streak sweeping across the front card, background is a deep navy radial gradient from #24508f at top-left to #102e5a, crisp edges, minimal, premium fintech aesthetic, no text other than the single letter S --ar 1:1 --v 6 --style raw

**2.3 [C] [S] App-Icon-Produktion nach Spezifikation:** Master 1024², ohne Alpha, ohne vorgerundete Ecken; Hintergrund = der Scan-Orb der App: `radial-gradient(circle at 30% 25%, #24508f → #102e5a)`; Marke in zentralen 66 % (~676px), Cap-Height ~560–600px; iOS-18-Dark-Variante (Glyphe weiß/`#a9c4ea` auf transparent, kein Verlauf), Tinted-Variante (Graustufen, opak); Social-Avatar mit Marke auf 56 % (Kreis-Crop); eBay-Store seromunich = derselbe Avatar. Kalligrafisches SR bleibt bewusst auf Login + Splash (Heritage).

**2.4 [C] [S] App-Store-Metadaten (fertig, nur einsetzen):**
- Titel (30 Z.): `SERO — Karten scannen & listen` (Fallback: `SERO: Karten scannen & listen`)
- Untertitel (29 Z.): `Vom Foto zum fertigen Listing` (Alternative: `Scan, Marktwert, eBay-Listing`)
- Beschreibung, erste 3 Zeilen: „Fotografiere eine Karte — SERO erkennt Stück, Set, Sprache und Grading-Label. / Der Marktwert kommt aus echten eBay-Verkäufen, nicht aus Wunschpreisen. / Ein Tipp macht daraus ein fertiges Listing. Andere Apps zeigen dir Werte. SERO macht sie zu Geld."
- Screenshot-Captions: u. a. „Tausende Verkäufe. Ein Marktwert." am Kerzen-Chart

**2.5 [C] [S] Slogan-Zuordnung (final, nach Einsatzort):**
- Store-Untertitel: „Vom Foto zum fertigen Listing."
- Video-Endcards: „Ein Tipp bis eBay." / „Behalten, was bleibt. Listen, was geht."
- Instagram-Bio: „Deine Sammlung, verkaufsfertig." + „Auch dein Bulk hat Marktwert."
- Merch/Sticker: „Binder leer, Konto voll."
- Pull-Video-Endcard: „Erst der Pull, dann der Payout."
- Launch-Claim: „Für die Karten, die gehen dürfen."
- Ads über der ehrlichen Rechnung: „Aus acht Minuten wird eine." (freistehend: „Aus rund acht Minuten…")
- Anti-Positionierung als Presse-/Pitch-Baustein: Dachzeile **„SERO ist keine App zum Zuschauen."** + die 5 Punkte (kein Tracker, kein Katalogwert, kein Marktplatz, kein Autopilot, keine übersetzte US-App)
- US-Phase vorbereitet: „From photo to finished listing." / „Empty binder, full account." / „Keep what stays. List what goes."

**2.6 [C] [M] Drei Social-Templates im Kurszettel-Raster** (einmal Template, täglich Inhalt — der Solo-Founder-Hebel): „Marktbericht" (Karte des Tages + echter Verkaufskurs), „Top-Mover der Woche" (3 Karten, 3 Deltas Grün/Rot), „Wunschpreis vs. Kurs". Layout: links freigestelltes Kartenfoto (Pflicht-Regel), rechts Kerzen-Sparkline + Mono-Zahlen. Claude baut sie als HTML/SVG-Exportvorlagen.

---

# PHASE 3 — Virales Marketing (Woche 3–6, Kanal-Setup ab Woche 1)

## ⭐ Größter Hebel: Der Sonntags-Batch — Format 1 „Karte → Wert in 3 Sekunden"

**Warum zuerst:** Die ganze Strategie in einem Satz: Bildschirm abfilmen, was die App ohnehin tut, echte Karten aus dem eigenen Lager als Darsteller. **[S🎥]** 60–90 Min am Sonntag = 5–6 Videos = die ganze Woche versorgt. Ohne Batching kippt der Plan, mit Batching ist er bequem. Setup: Handy 1 filmt schräg von oben Hand + Karte + Handy 2 mit App; CapCut reicht. Skript: Hook 0–1s („Die lag 3 Jahre in meiner Schublade. Pass auf.") → Scan-Orb-Tap, Ladetext „Echte Verkäufe statt Wunschpreise." trägt inhaltlich → Ergebnis, kurze Stille → Payoff: Gyro-Holo oder Kerzen-Chart → CTA in die Kommentare („Welche Karte soll ich als Nächstes scannen"). Serienlogik: abwechselnd Cent-Karte („80 Cent — und das ist okay") und teurer Slab — die Ehrlichkeit ist das Differenzierungsmerkmal.

**Dreh-Checkliste vor jedem Capture:** `#dryBadge` (Testmodus) aus · Privacy-Blur (`.hideable.veiled`) aus · Gyro-Permission-Tap vorab · Werte mit konstanter Stellenzahl planen, damit der Odometer rollt (nach Fix 1.1 rollt auch das Ergebnis-Sheet) · nativ 9:16, für Composites ProRes via Mac-Capture.

## Weitere Maßnahmen in Reihenfolge

**3.1 [S] [S] Kanal-Setup:** Handles einheitlich sichern; Prio: **TikTok (1 Video/Tag) > Instagram Reels (Crosspost + Stories, DMs) > YouTube Shorts (Crosspost, Long-Tail-Suche „Was ist meine Karte wert") > X (nur Text, Build-in-Public, 10 Min/Tag)**. Ein Video, drei Plattformen, nativ ohne Wasserzeichen. Profilbild: neue Marke auf Navy. Bewusst NICHT: Facebook-Page, LinkedIn, eigener Discord (leerer Server = Todessignal), Threads/Bluesky.

**3.2 [S] [M] 4-Wochen-Kalender** (Tagesrhythmus Mo–Fr ~35 Min: 5 Min posten, 15–20 Min Kommentare/DMs — wichtiger als neuer Content, 10 Min X):
- **W1:** Fundament. 6× Format 1 (3 Slabs, 3 Bulk). X-Pinned-Thread („Ich verkaufe seit Jahren Sammelkarten auf eBay. Jedes Listing von Hand: ~8 Minuten…"). 3–5 FB-Gruppen + 2 Discords beitreten — **nur lesen**.
- **W2:** Beweis-Schiene. 2× Format 3 „Slab des Tages" mit echtem seromunich-Listing (der unfaire Vorteil — Collectr kann das nicht kopieren), erster Stitch (Format 2 „Wunschpreis vs. Marktwert" — nie höhnisch, immer nüchtern), erster X-Wochenreport mit echten Zahlen („12 Nutzer, 340 Scans" schlägt jede Ankündigung), erste „Verkauft"-Folge.
- **W3:** Grading-Bogen öffnen (Format 5 „Grading-Orakel": „Grading kostet 25 €. Lohnt sich das hier?" — App zeigt „~+140 € bei PSA 10*"; Karten real einschicken = Reveal in W6–8 vorproduziert; das Nein zur zweiten Karte macht das Ja glaubwürdig). Königsvideo **„100 Karten, 100 Minuten"** drehen (Timer an, Zeitraffer, Timer neben Entwürfe-Tab), noch nicht posten. Erste hilfreiche Wert-Antworten in Gruppen — als Person, ohne Link.
- **W4 — Launch:** Montag „100 Karten, 100 Minuten" überall + pinnen; X-Thread mit der ehrlichen Rechnung inkl. Fußnoten-Transparenz. 30-Tage-Challenge starten. Freitag: Recap auf X (erfahrungsgemäß reichweitenstärkster Post des Monats). Danach: Daten zeigen die 2 Formate, die tragen — Fokus.

**3.3 [S] [M] Growth ohne Budget:** (a) **Der Wertcheck-Mensch** — täglich 3–5 „Was ist die wert?"-Posts in Gruppen mit echten Verkaufsdaten beantworten, nie mit Link; nach 4 Wochen ist Sven „der mit den echten Zahlen". (b) **Scan-Duell** — Mensch schätzt vs. SERO scannt, duettierbar; Türöffner zu 5–10 deutschen WhatNot-Breakern via freie Pro-Zugänge (Preisfindung live ist deren tägliches Problem). (c) **Meilenstein-Challenge** — „100 Stücke in 30 Tagen", Meilenstein-Screens (App feiert 10/25/50/100 bereits) unter Hashtag, jeder Repost in Stories; UGC ist das wertvollste Algorithmus-Signal und treibt exakt die Kernmetrik.

**3.4 [S+P] [M→L] KI-Video-Produktion in dieser Reihenfolge:**
1. **Konzept 3 „Holo"** (8s-Loop, ein Shot, kleinstes Risiko, Grid-Anker; die vier Farben sind exakt der App-Shader: Rosé `#ff5a82`, Gold `#ffdc78`, Mint `#78ffbe`, Eisblau `#6eb4ff`). Luma Ray2 (einziges Tool mit nativem Seamless-Loop; First/Last Frame = dasselbe echte Makro-Foto):
   > Seamless loop. Extreme macro flyover across the surface of a holographic trading card, the embossed foil pattern filling the frame like an aerial landscape of iridescent dunes. A single band of rainbow interference light — rose, gold, mint, ice-blue in that order — sweeps slowly from left to right across the ridges, as if the card is tilting under one soft light source. Probe macro lens look, 100mm equivalent, f/4, razor-thin focus plane, specular highlights blooming gently. Deep navy vignette at the frame edges. Photorealistic, no particles, no sparkles, no fantasy elements, no text, no fingers. First and last frame identical; the light band completes exactly one full sweep.

   Variante B (Pflicht als zweiter Grid-Post): echtes iPhone, Gyro-Holo abgefilmt (1/50-Shutter). Ehrlich: die reale Makro-Aufnahme (Drehteller + LED) schlägt die KI hier möglicherweise.
2. **Konzept 1 „Der Reveal"** (15s) — liefert den App-Store-Preview-Cut (Apple verlangt >80 % echtes UI-Material → jeder Spot in zwei Schnittfassungen). Match-Cut: Hailuo-Lichtglühen → Dashboard-Odometer rollt los. Kling-2.5-Pro-Prompt (I2V vom echten Karten-Foto) und Hailuo-02-Prompt liegen fertig im KI-Video-Report; Licht-Satz wortgleich in jedem Prompt: `single warm practical lamp from the upper left, cool soft window fill from the right, faint haze in the air`. Overlays: „Fotografieren." / „Erkannt." / „Marktwert aus echten eBay-Verkäufen" / „Ein Tipp. Gelistet." / Endcard „SERO macht deine Karten zu Geld."
3. **Konzept 4 „Der Stapel"** (20s, billigster Drehtag: Stativ + 100 echte Karten, 45 Min real auf 12s gerafft, PiP-Odometer synchron; Overlays „100 Karten." / „~100 Minuten." / „0 Felder von Hand." / „Der Stapel schrumpft, nicht dein Wochenende."). Hailuo-Opener-Prompt im Report.
4. **Konzept 2 „Aus der Schuhschachtel"** (30s, teuerster Spot mit Sora-2- und Runway-Gen-4-Sequenzen) — erst wenn 1–3 Reichweite belegt haben.

   **Die 5 Anti-Slop-Regeln für alle KI-Shots:** Real-Plate-Anker statt Text-to-Video · eine Kamerabewegung pro Shot, physikalisch beschrieben · ein Licht-Satz pro Konzept, wortgleich · null Text/Zahlen/UI in KI-Shots (alle Zahlen aus Captures — halluzinierte Marktwerte wären fatal, echte Zahlen SIND das Produktversprechen) · 10s generieren, 2–4s verwenden, gemeinsames Grade (Navy-Schatten `#102e5a`) + 35mm-Grain über alle Quellen.

**3.5 Die 5 Nicht-Tun-Regeln (verbindlich):** Nie als Marke in fremde Communities platzen (erst Person, Tool nur auf Nachfrage) · Karten zeigen, nicht Features (jedes Video beginnt mit einer Karte und einer Frage, nie mit der App) · launchen statt ankündigen (kein Countdown, keine Giveaways am Anfang — die ersten 1000 Follower prägen den Algorithmus) · **nie mit Hype-Preisen erwischen lassen** (Quelle immer mitsprechen: „laut verkauften Angeboten", Werte mit „~") · keinen Kanal-Zoo (4 Kanäle, ein Video, Zeit in Antworten — Video-Reply auf Kommentar ist auf TikTok eigener Content).

---

# WOCHE 1 — Was Sven und ich diese Woche konkret tun

**Claude (sofort startbar, Reihenfolge = Umsetzungsreihenfolge):**
1. **Heute:** Das Defekt- und Kernmoment-Paket in einem Rutsch — Odometer-Seed-Fix (1 Zeile), die 5 fehlenden CSS-Klassen (`.gstat`/`.gmore`/`.gfav`, `.login-feats`/`.lf-ic`/`.login-link`, Swipe-`.armed`), `--muted`-Fix, Quick-Fix-Sammlung 1.8. Zusammen < 1 Tag, danach ist nichts mehr sichtbar kaputt.
2. **Tag 2:** Reveal-Choreografie (0/120/300/400/550/900 ms) + `haptic()`-Wrapper + Tour auf 1 Karte mit Kamera-Direktstart + Paywall-Personalisierung.
3. **Tag 3:** Slab-SVG (Weg C) bauen und an Favicon/`.cam-mono`/`.ptr-mono` einsetzen; IBM-Plex-Mono-Test an `h-value`, Odometer, `v-main` (ein Screenshot-Vergleich für Svens Go/No-Go zu Richtung D); Store-Metadaten-Texte final ablegen.
4. **Danach/Puffer:** Analyse-Bühne mit Fortschrittsring, `celebrate()`-Upgrade, Mikro-Politur 1.9.

**Sven (Gesamtaufwand diese Woche: ~3–4 Std, davon 1 Std Dreh):**
1. **Mo–Di (~20 Min):** Handles sichern (TikTok, Instagram, YouTube, X — einheitlich), Bios mit „Deine Sammlung, verkaufsfertig." + App-Link.
2. **Di–Mi (~30 Min):** Weg-A-Prompt (und optional Weg B) in Midjourney laufen lassen, 3–4 Kandidaten an Claude zurück — ich baue den Gewinner als produktionsreifes SVG nach.
3. **Mi–Fr (~15 Min/Tag):** Den 3–5 größten deutschen TCG-Facebook-Gruppen und 2 Discords beitreten. Nur lesen, Ton lernen. Noch nichts posten.
4. **Entscheidung (5 Min):** Mono-Ziffern-Screenshot ansehen → Go für Richtung D. (Meine Empfehlung: Go — die Begründung steht oben, Widerspruch 1.)
5. **Sonntag (60–90 Min): Der erste Batch-Dreh.** 6× Format 1 (3 Slabs, 3 Bulk-Karten) mit der frisch reparierten App — der rollende Odometer und die gestaffelte Reveal-Choreografie aus Tag 1–2 sind dann bereits im Bild. Checkliste: Testmodus-Badge aus, Blur aus, Gyro-Tap vorab. Das erste Video geht Montag live — **der Launch ist das erste Scan-Video, nicht ein Countdown.**

**Ergebnis Ende Woche 1:** App ohne sichtbare Defekte, Kernmoment mit Gänsehaut-Choreografie, Onboarding auf 2 Tipps verkürzt, personalisierte Paywall, kleine Marke klein lesbar, Branding-Entscheidung gefallen, 6 Videos im Kasten, Kanäle stehen. Woche 2 beginnt mit täglichem Posten und der Token-Konsolidierung im Hintergrund.