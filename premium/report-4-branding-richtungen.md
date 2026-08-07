# SERO — Vier Branding-Richtungen

**Ausgangslage, ehrlich benannt:** Das Produkt ist bereits auf Premium-Niveau. Liquid-Glass-Tab-Bar mit atmendem Scan-Orb (`sero.css:176–218`), Gyro-Holo mit color-dodge-Regenbogen (`sero.css:673–682`), Odometer im Apple-Wallet-Look (`sero.css:502`), Kerzen-Chart, kondensierender Large-Title, ein sauber gedoppelter Dark-Mode („Studio Black", `sero-dark.css`). Die Branding-Frage ist deshalb nicht „Wie retten wir die App", sondern: **Welche Haltung trägt die Marke nach außen** — App-Icon, Instagram, App-Store-Seite, Ton. Drei der vier Richtungen unten lassen das Produkt weitgehend in Ruhe.

**Ein Befund vorweg, der alles prägt:** Die Marke hat sich in den Texten längst entschieden, sie weiß es nur noch nicht. „Die ehrliche Rechnung", „Echte Verkäufe statt Wunschpreise", die Fußnote, die die eigene Rechnung angreifbar macht (`marketing-bausteine.md`, Abschnitte 3–5), Kerzen-Charts, `font-variant-numeric: tabular-nums` an praktisch jeder Zahl — das ist die Stimme eines **nüchternen, ehrlichen Marktplatzes**, nicht die einer Sammel-Kuschel-App. Richtung D baut genau darauf.

---

## Richtung A — „Studio" (Ultra-Clean-Minimal)

**Kernidee in einem Satz:** SERO ist das Werkzeug, das so selbstverständlich aussieht wie eine Apple-System-App — die Karte ist der Star, die Marke tritt einen Schritt zurück.

**Farbpalette:**
- Navy `#102e5a` bleibt Anker und wird **einzige** Markenfarbe
- `#ffffff` / `#fafbfc` (bestehendes `--bg-elevated`) hell, `#000000` / `#141519` dunkel — exakt der Ist-Zustand
- Grün `#0da05a` / Rot `#e5484d` (bestehende `--chart-up`/`--chart-dn`) bleiben streng funktional
- **Gestrichen wird:** der Verlauf im Primary-Button (`#1d4d95→#102e5a`, `sero.css:126`) → flaches Navy; der Radial-Verlauf im Scan-Orb (`sero.css:208`) → flaches Navy mit hartem weißem Ring

**Typo:** SF/System pur, nichts dazuholen. Auf der Web-/Store-Seite: `Inter Tight` (Google) als SF-Ersatz, damit Web und App identisch wirken. Wordmark bleibt das bestehende Asset (`assets/wordmark-navy.png`).

**Bildsprache:** Freigestellte Karte auf reinem Weiß, harter, kurzer Schatten, sonst nichts. Eine Karte pro Bild. Der Marktwert als einzige zweite Information, in Navy.

**App-Icon:** Weißes Feld, SR-Monogramm in Navy, zentriert, viel Luft — keine Kante, kein Verlauf, kein Schatten. (Dunkle Icon-Variante: Navy-Feld, weißes Monogramm.)

**Instagram:** Galerie wie ein Apple-Produktkatalog — weiße Kacheln, eine Karte, eine Zahl. Captions einzeilig. Kein Grain, keine Sticker. Wiedererkennung entsteht durch radikale Gleichförmigkeit.

**Ton:** Noch knapper als jetzt. „Ein Foto. Ein Marktwert. Ein Listing." — die bestehenden Ein-Zeiler aus `marketing-bausteine.md` Abschnitt 5 passen unverändert.

**Bleibt vs. ändert sich:** ~95 % des bestehenden Designs bleiben — diese Richtung ist im Grunde die Radikalisierung dessen, was `sero.css` schon ist. Es fallen nur die letzten „Wow"-Reste: Hero-Verläufe (`sero.css:399–408`) würden zu flachem Navy, der Glanz-Lauf über die Karten (`gshine`, `sero.css:594–599`) entfiele. **Genau das ist der Haken:** Svens gewählter Stil ist „Mix: Clean + Wow" — A streicht das Wow.

**Risiko/Aufwand:** Aufwand minimal (ein Nachmittag CSS, neue Icon-Exports). Risiko: Unsichtbarkeit. In einem Feed voller Holo-Glitzer-Karten-Content ist „noch cleaner" keine Position, und gegen Collectr (das ebenfalls clean-modern auftritt) differenziert es null.

---

## Richtung B — „Foil" (Retro-Tech / Y2K)

**Kernidee in einem Satz:** SERO fühlt sich an wie das Ziehen einer Holo aus dem 99er-Booster — die Marke trägt die Folie, mit der die Zielgruppe aufgewachsen ist.

**Farbpalette:** Navy `#102e5a` bleibt, wird aber zum **Nachtgrund**, auf dem die Folie leuchtet. Das Geniale: Die Holo-Palette steht bereits im Code — die Conic-Gradient-Stops des Gyro-Holo (`sero.css:675–677`) als Hex:
- Holo-Pink `#ff5a82`, Holo-Gold `#ffdc78`, Holo-Mint `#78ffbe`, Holo-Blau `#6eb4ff`
- Chrom: `#e8edf4` / `#aab6c6` (angelehnt ans bestehende Silber-Siegel `#b9c2cf→#8b96a6`, `sero.css:571`)
- Tiefe: `#0a1d3c` (bestehender Hero-Gradient-Endpunkt, `sero.css:404`)

**Typo:** `Space Grotesk` (Google) als Display für Social/Store — techy, aber lesbar; für Akzente sparsam `Space Mono`. In der App bleibt SF. Finger weg von `Orbitron`/`VT323` als Fließtext — das kippt in Kostüm.

**Bildsprache:** Makro-Aufnahmen echter Holo-Folie als Textur, VHS-Grain, Binder-Seiten, Kellerfund-Ästhetik. Chrom-Schriftzüge auf dunklem Navy. Vorher/Nachher: „Schuhkarton → Kurs".

**App-Icon:** Dunkles Navy-Feld, SR-Monogramm als Chrom-Folie mit schräg einfallendem Holo-Schimmer (der Vierfarb-Verlauf von oben, in einem schmalen diagonalen Band gefangen) — wie eine Karte, die man ins Licht kippt.

**Instagram:** Die stärkste organische Richtung für 16–45. Holo-Textur-Reels, Nostalgie-Hooks („Du hast sie 1999 gezogen. SERO sagt dir, was sie heute wert ist."), Scan-Screenrecordings mit dem Gyro-Holo der App als natürlichem Content. Der Holo-Effekt der App IST hier der Werbespot.

**Ton:** Nostalgisch, aber trocken — die Wording-Regeln (keine Ausrufezeichen) schützen vor dem Y2K-Kreisch-Kitsch von selbst. „Damals getauscht. Heute gelistet."

**Bleibt vs. ändert sich:** In der App bliebe fast alles — der Gyro-Holo (`holo-wrap`) und der Glanz-Lauf (`gshine`) würden vom Easter Egg zum Marken-Signature befördert, evtl. bekäme der Scan-Orb einen dezenten Holo-Ring statt des blauen Glows (`sero.css:213–218`). Außenauftritt (Icon, Social, Store-Screenshots) würde komplett neu.

**Risiko/Aufwand:** Aufwand mittel (Asset-Produktion: Folien-Makros, Chrom-Type, Templates). Risiko: **Austauschbarkeit im Genre** — Holo-Glitzer ist die Standard-Ästhetik von Karten-TikTok; man reiht sich ein statt herauszustechen. Und es verkauft die falsche Hälfte der Positionierung: Nostalgie sagt „behalten und streicheln", SERO sagt „zu Geld machen".

---

## Richtung C — „The Vault" (Luxus / Auktionshaus)

**Kernidee in einem Satz:** Jedes Stück wird behandelt wie ein Lot bei Sotheby's — SERO ist das Auktionshaus, das in deine Hosentasche passt.

**Farbpalette:**
- Navy vertieft sich zu Mitternacht: `#0a1d3c` (existiert bereits als Hero-Endpunkt) und `#081527` als Bühne
- Champagner-Gold `#c9a962`, dunkel `#a8842f` — Vorsicht: der dunkle Ton ist exakt das bestehende PSA-Gold-Siegel (`gseal-in.gold`, `sero.css:570`); das Gold-Siegel müsste dann exklusiv bleiben oder die Marke übernimmt es bewusst
- Papier `#f6f2e9`, Tinte `#10131a`

**Typo:** `Fraunces` (Google, mit optischen Display-Schnitten) für Überschriften und Preise auf Social/Store — Serifen mit Charakter, nicht museal. UI-Zahlen bleiben SF tabular.

**Bildsprache:** Einzelne Karte auf tiefdunklem Grund, gerichtetes Licht wie in einer Vitrine, Gold-Linien als Rahmen. Slabs wie Preziosen fotografiert. Keine Stapel, keine Binder — nur Einzelstücke.

**App-Icon:** Feld in `#081527`, SR-Monogramm goldgeprägt (subtiler Emboss), hauchdünne goldene Keyline als Rahmen — wie das Schild an einer Tresortür.

**Instagram:** Schwarze Kacheln, goldene Serifen-Zahlen, „Lot"-Inszenierung einzelner Grails, wöchentliches „Stück der Woche". Sehr fotogen, sehr langsam.

**Ton:** Gemessen, knapp. „Jedes Stück ein Lot." — **aber hier knirscht es:** Die verbindlichen Regeln sagen Duzen, und die bestehende Marketing-DNA ist die „ehrliche Rechnung" mit Fußnote — Anti-Aura, Pro-Transparenz. Ein Auktionshaus, das duzt und seine Kalkulation offenlegt, ist ein interessanter Hybrid, aber die Fallhöhe ist real.

**Bleibt vs. ändert sich:** Am meisten von allen vieren. Studio White (`html:not(.force-dark) body { background:#fff }`, `sero.css:61`) widerspricht der Vault-Bühne — konsequent wäre Dark-first, also eine Umkehrung der Theme-Logik. Gold als neuer Akzent kollidiert mit der semantischen Grader-Farblogik (`gseal-in`) und dem Favoriten-Stern `#f5a623` (`sero.css:560`). Die Zielgruppe „Flipper 16–45" fühlt sich von Auktionshaus-Gestik eher ausgeschlossen als eingeladen.

**Risiko/Aufwand:** Aufwand hoch (Theme-Umbau, komplette Asset-Neuproduktion, Ton-Neuabstimmung). Risiko hoch: Luxus-Codes signalisieren „für die anderen", und ein Solo-Founder kann das nötige durchgängige Produktions-Niveau (Fotografie!) schwer halten. Wert hätte diese Richtung später als **Sub-Brand für ein Premium-Tier** (etwa ein „Vault"-Bereich für Stücke ab 500 €) — nicht als Hauptmarke.

---

## Richtung D — „Der Kurszettel" (Out-of-the-box: die ehrliche Börse für Karten)

**Kernidee in einem Satz:** SERO behandelt Karten als das, was die Zielgruppe längst in ihnen sieht — Assets mit Kurs — und wird zur Trade-Republic-Ernsthaftigkeit mit Sammlerherz: nüchtern, transparent, handelbar.

**Warum das keine erfundene Richtung ist:** Sie liegt bereits im Code und in den Texten. Kerzen-Chart (`sero.css:1158–1178`), Odometer, `tabular-nums` überall, der Apple-Stocks-Hero (`sero.css:427–439`), die Login-Zeile „Deine Karten. Dein Marktplatz." (`index.html:31`), „Echte Verkäufe statt Wunschpreise" und „Die ehrliche Rechnung" (`marketing-bausteine.md`). D macht aus diesen Indizien die These.

**Farbpalette:** Disziplinierteste der vier — **keine neue Farbe, sondern eine neue Textur:**
- Navy `#102e5a` bleibt unangetastet der Marken-Anker („die Bank")
- Grün `#0da05a` / Rot `#e5484d` (hell) bzw. `#30d158` / `#ff453a` (dunkel, `sero-dark.css`) werden vom Chart-Detail zum sichtbaren Marken-Vokabular — streng semantisch (nur Kursbewegung, nie Deko)
- Graphit `#8b93a1` (bestehendes Dark-`--label-2`) für Sekundäres
- Papierweiß / Studio Black als Bühnen — beides existiert

**Typo:** Der eigentliche Marken-Hebel: `IBM Plex Mono` (Google) für Kurse, Labels, Zert-Nummern und Daten auf Social und Store-Screens — die Ästhetik des Börsentickers und des Kassenbons. Headlines bleiben SF/`Inter Tight`. In der App reicht `font-family`-Umstellung auf Mono an genau drei Stellen: Wert-Ziffern (`--h-value`, `v-main`), Odometer, Chart-Legende — der Rest bleibt.

**Bildsprache:** Das freigestellte Karten-Foto (Pflicht-Regel, bleibt Hauptbild) neben seinem Kurs: links Karte, rechts Kerzen-Sparkline und Mono-Zahlen. Screenshots echter verkaufter Angebote als wiederkehrendes Beweis-Format. Layout-Raster wie ein Kurszettel im Wirtschaftsteil.

**App-Icon:** Navy-Feld, weißes SR-Monogramm, dessen auslaufender kalligrafischer Endstrich in einen feinen, ansteigenden Kurs-Tick übergeht — auf 60 px liest es sich als Monogramm, auf 512 px erzählt es die Positionierung. (Fallback, falls der Strich nicht trägt: Monogramm pur, darunter eine 3-Kerzen-Mikroreihe in Grün/Weiß.)

**Instagram:** Serielle Formate statt Einzelposts — der Vorteil für einen Solo-Founder: einmal Template, täglich Inhalt. „Marktbericht": Karte des Tages mit echtem Verkaufskurs. „Top-Mover der Woche": drei Karten, drei Deltas in Grün/Rot. „Wunschpreis vs. Kurs": Aktiv-Angebot gegen tatsächliche Verkäufe. Alles direkt aus der App exportierbar — die Kerzen-Charts existieren ja schon.

**Ton:** Exakt der Ton, der in `marketing-bausteine.md` bereits geschrieben steht — trocken, zahlengetragen, angreifbar-ehrlich. Neue Zeilen in derselben Stimme: „Kurs statt Bauchgefühl." · „Dein Binder hat einen Kurs. SERO zeigt ihn dir." · „Gefühlt wertvoll ist keine Zahl." Alle Wording-Regeln (Stück, listen, Marktwert, tippen, Duzen, keine Ausrufezeichen, SERO in 3. Person) passen ohne Verbiegung — sie klingen in dieser Richtung sogar am natürlichsten.

**Bleibt vs. ändert sich:** In der App bleibt fast alles — inklusive Gyro-Holo und Glanz-Lauf, die als Produkt-Momente das Sammlerherz halten (der Kurszettel ist die Haltung, die Holo ist die Ware). Es ändern sich: Ziffern-Typo an den drei genannten Stellen, App-Icon, sämtliche Außen-Assets, Store-Screenshots im Kurszettel-Raster.

**Risiko/Aufwand:** Aufwand niedrig bis mittel (ein Font, ein Icon, drei Social-Templates). Risiko: Finance-Kälte — wenn nur noch Zahlen sprechen, verliert man die Sammler, die (noch) nicht verkaufen wollen. Gegenmittel ist eingebaut: Foto-first-Regel, Holo-Tilt, Sammler-Fortschritt bleiben unangetastet; die Kälte-Grenze ist markiert.

---

## Empfehlung: Richtung D — „Der Kurszettel"

**Für Svens Situation ist D die klare Wahl**, aus vier Gründen:

1. **Sie ist die Positionierung, wörtlich genommen.** „Collectr sagt dir, was deine Karten wert sind. SERO macht sie zu Geld." — A sagt dazu nichts, B verkauft Nostalgie (behalten), C verkauft Aura (bewundern). Nur D verkauft den Kurs, also das Verkaufen. Und gegen Collectr, das visuell die Sammel-Seite besetzt, ist „die Börse für deinen Binder" die einzige Richtung mit echter Kontrastkante.

2. **Sie ist zu ~70 % schon gebaut.** Kerzen-Chart, Odometer, tabular-nums, Stocks-Hero, Studio Black, „Deine Karten. Dein Marktplatz.", die komplette „ehrliche Rechnung" — Sven müsste für A, B oder C gegen sein eigenes Material arbeiten; für D muss er es nur zuspitzen. Für einen Solo-Founder ist das der Unterschied zwischen einem Wochenende und einem Quartal.

3. **Sie skaliert Content ohne Produktionsapparat.** B braucht laufend hochwertige Folien-Makros und Trend-Timing, C braucht Vitrinen-Fotografie. D braucht Templates plus die Daten, die die App ohnehin erzeugt — jeder echte eBay-Verkauf ist ein Post. Das ist die einzige Richtung, deren Instagram-Frequenz ein Einzelner neben dem eBay-Store durchhält.

4. **Sie passt in die deutsche Szene.** Die Zielgruppe 16–45 ist die Trade-Republic-Generation — Kurs-Ästhetik ist ihr vertraut und signalisiert Seriosität, die im deutschsprachigen Karten-Markt (viel Wildwuchs, viele Wunschpreise) rar und damit wertvoll ist. „Ehrlicher Makler" ist in Deutschland eine stärkere Vertrauensposition als „glänzendes Spielzeug".

**Konkrete erste Schritte:** IBM Plex Mono an `h-value`, Odometer und `v-main` testen; App-Icon-Entwurf „Monogramm mit Kurs-Tick"; drei Social-Templates (Marktbericht / Top-Mover / Wunschpreis vs. Kurs) im Kurszettel-Raster; Store-Screenshots mit Mono-Kursen über den bestehenden Screens. Nichts davon berührt die produktive App-Logik.

**Was von B trotzdem mitkommt:** Der Gyro-Holo bleibt der Signature-Moment im Produkt und in jedem Screenrecording — er ist der Beweis, dass hinter dem Kurszettel ein Sammler steht. C wird nicht verworfen, sondern geparkt: als spätere Premium-Tier-Ästhetik („Vault" für Stücke ab einem Schwellenwert) hat das Auktionshaus-Vokabular einen legitimen Platz — als Hauptmarke nicht.