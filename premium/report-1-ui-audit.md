# SERO — Hartes Visual-Audit gegen Premium-Maßstab

Geprüft: `/Users/smorty/sero-app/web/sero.css` (komplett, 1729 Z.), `sero-dark.css` (84 Z.), `index.html`, aus `sero.js` die Render-Pfade `renderDashboard` (Z. 1552), `renderCollection` (Z. 1971), `renderDetail` (Z. 3029), `renderScanMode` (Z. 2238), `openSheet` (Z. 3685).

## Vorab: Was schon Premium-Niveau hat (ehrlich, kein Schönreden)

- **Chip-Token-System** (sero.css Z. 39–41): exakt zwei Chip-Größen als Custom Properties, im Kommentar dokumentiert. Das ist echte Disziplin, die die meisten Apps nicht haben.
- **Ziffern-Disziplin** (Z. 1570–1573): ein Sammel-Selektor erzwingt `tabular-nums` auf allen Preisstellen — nichts flattert. Apple-Kaliber-Detail.
- **Einheitliches Press-Feedback** (Z. 1575–1578): jedes berührbare Element quittiert den Touch, zentral geregelt.
- **Liquid-Glass-Tab-Bar** (Z. 176–222): Specular-Kante via `::before`, Inhalt scrollt sichtbar unters Glas, und die `mask-image` auf `.page-scroll` (Z. 229) blendet Inhalt weich aus, bevor er unter die Bar taucht. Das ist die Sorte Detail, die iOS-Systemapps auszeichnet.
- **Dark Mode ist designt, nicht invertiert** (Grundsatz-Urteil, Details unter Punkt 6): eigene Flächen-Treppe, eigener Schatten-Token, aufgehellter Tint, eigene Kerzenfarben.
- **`--shadow-card` wird 20+ mal wiederverwendet** — der Karten-Schatten ist konsistent.
- **Reduced Motion** doppelt sauber behandelt (Holo Z. 683, global Z. 1614, sv2 Z. 1535).

Das Fundament ist gut. Die Probleme liegen in drei Bereichen: (a) **fehlende CSS-Klassen, die das JS bereits rendert** — das sind echte sichtbare Defekte, (b) **Wildwuchs in Skalen** (Typo, Radius, Spacing), (c) **eine zweite Farbpalette, die im JS lebt**.

---

## 1. Typografie-Skala: 30 Schriftgrößen, 7 Gewichte

Vollständiges Inventar aus sero.css (`grep font-size`):

| px | Vorkommen | px | Vorkommen |
|---|---|---|---|
| 9 | 1 | 15 | 9 |
| 10 | 2 | 15.5 | 3 |
| 10.5 | 6 | 16 | 6 |
| 11 | 9 | 16.5 | 4 |
| 11.5 | 14 | 17 | 10 |
| 12 | 12 | 18 / 20 | je 1 |
| 12.5 | 16 | 21 / 22 | je 3 |
| 13 | 13 | 24, 26, 27, 28 | je 1 |
| 13.5 | 12 | 32, 34, 36 | je 1 |
| 14 | 12 | 40 | 2 |
| 14.5 | 4 | 42, 44 | je 1 |

**30 unterschiedliche Größen.** Dazu kommen Inline-Größen im JS (110 `style="…"`-Attribute in sero.js), z. B. `renderCollection` Z. 1996: `<div class="h-value" style="font-size:28px">` — überschreibt die 36px aus `.hero .h-value` (CSS Z. 417) per Inline-Style. Typo-Entscheidungen leben im JS.

Das Halbpixel-Raster (11.5/12.5/13.5/14.5/16.5) erzeugt Paare, die niemand unterscheiden kann: 13 vs. 13.5 (25 Vorkommen zusammen), 11 vs. 11.5 (23 zusammen). Gewichte: **500(11), 600(29), 650(5), 700(24), 750(13), 760(1), 800(14)** — die 760 in `.sv2-num b` (Z. 1410) ist ein einsamer Ausreißer neben 13× 750.

**Konsolidierte Skala als Tokens** (in `:root` neben die Chip-Tokens, Z. 39):

```css
--fs-hero: 40px;      /* ov-value 44→40, res-val 42→40, stval 40 */
--fs-large: 32px;     /* large-title 32, hero h-value 36→32 */
--fs-title: 26px;     /* v-main 27, pt-price 26, party-price 28, pwm .v 24 */
--fs-heading: 21px;   /* d-name 21, empty/party h2 22, sheet h3 20, scan-hero h2 21 */
--fs-body: 17px;      /* Grundgröße, Buttons, Inputs (nie <16px auf Inputs — iOS-Zoom) */
--fs-sub: 15px;       /* rlabel 15.5, rvalue 15, ov-label 15, gval 15, 14/14.5→15 */
--fs-footnote: 13px;  /* 12.5/13/13.5 → 13; Chip-M bleibt 13 (Token existiert) */
--fs-caption: 11.5px; /* 11/11.5/12 → 11.5; Chip-S bleibt 11 */
--fs-micro: 10px;     /* nur Uppercase-Labels: sv2-cap, tab, hp span 9→10 */
```

9 statt 30 Stufen, Gewichte auf **500 / 600 / 700 / 800** eindampfen (650→600, 750→700, 760→700). Die Hierarchie Übersichts-Wert (44) → Large-Title (32) → Kartenwert (27) bleibt vollständig erhalten.

---

## 2. Farb-Disziplin: 19 Blautöne, 8 Grüns, 9 Rots — und eine Schattenpalette im JS

**Navy/Blau (19 distinct):** `#102e5a` `#14386e` `#0a1d3c` `#1b4483` `#1a4585` `#1d4d95` `#23508f` `#24508f` `#2c59a8` `#2c5aa5` `#2b5ea8` `#3567b8` `#4b86d6` `#5b93e8` `#7aa8ee` `#86ace6` `#a6d2ff` + JS: `#3478f6`, `#0b1f3e`. Drei davon liegen wenige Hex-Punkte auseinander: `#2c5aa5` (sero.css Dark-Block Z. 1698–1705), `#2b5ea8` (sero-dark.css Z. 49), `#2c59a8` (Orb-Glow Z. 216). Und `#23508f` vs. `#24508f` sind praktisch identisch (Z. 1696 vs. 208).

**Wo die Palette konkret bricht:**

1. **`--muted` wird benutzt, existiert aber nicht.** sero.css Z. 1219: `.scanres .res-sub { color: var(--muted); }` — die Variable ist nirgends definiert. Die Zeile ist „invalid at computed-value time", der Untertitel im Scan-Ergebnis (dem Wow-Moment) erbt volle Label-Farbe statt gedämpft zu sein.
2. **Doppelte, widersprüchliche Dark-Tokens.** sero.css Z. 1625–1669 definiert `--bg-elevated:#17191f`, `--label:#f2f4f9`, `--tint:#86ace6`; sero-dark.css (lädt danach, index.html Z. 20) überschreibt mit `#141519`, `#ffffff`, `#7aa8ee`. Der sero.css-Block ist toter Code und eine Wartungsfalle — wer dort ändert, sieht keinen Effekt.
3. **Zwei Dark-Blau-Werte gleichzeitig live:** `.fchip.on`/`.range-pill.on` = `#2b5ea8` (sero-dark gewinnt), aber `.qbox .qsend` und `.icon-btn.badged` = `#2c5aa5` (nur in sero.css, wird nicht überschrieben, Z. 1701–1705).
4. **Zweite Palette im JS:** 10 Hex-Farben inline in sero.js — `#8e8e93` (6×), `#34a853` (3×, Google-Grün statt `--green #248a3d`), `#c9a961`, `#5a9aa8`, `#a355d6`, `#3478f6`, `#e0518e`, `#e0382e`, `#fc7227`, `#8a5cf6` — vor allem als `.ric`-Icon-Hintergründe in `renderDetail` (Z. 3123–3213). Dazu `CAT_COLORS` (sero.js Z. 584) mit 11 weiteren Hexes. Im Dark Mode bleiben diese Farben unangepasst.
5. **Delta-Pillen mit eigenem Grün/Rot:** `.ov-delta.up/.down` (Z. 275–276) nutzt `#12995f`/`#c0392b` auf `#e7f8f0`/`#fdeceb` — vier Hardcodes, obwohl das `color-mix`-Muster mit `--green`/`--red` direkt darunter bei `.tchip` (Z. 723–724) bereits existiert. `#c0392b` ist zudem off-brand gegenüber `--red #d70015`.

**Grau-Töne:** hell diszipliniert (iOS-Standard `rgba(60,60,67,…)` + `#0b0f19`), dunkel dagegen sechs nahe Flächenwerte: `#141519` (elevated), `#16171c` (sheet), `#17191f` (tot), `#1d1f26` (thumb), `#23252b` (seg), `#48484d` (seg, tot). Sheet vs. Elevated (`#16171c` vs. `#141519`) unterscheidet niemand — ein Wert reicht.

---

## 3. Spacing-Rhythmus: 2er-Raster mit systematischem Wildwuchs

Histogramm aller px-Werte in padding/margin/gap: **8px(47), 12px(34), 10px(30), 4px(27), 6px(26), 2px(25), 16px(23), 14px(22)** dominieren — ein 2er-Raster existiert faktisch. Aber: **5px(7), 7px(5), 9px(5), 11px(12), 13px(8), 15px(4)** = 41 ungerade Ausreißer. Belege:

- `.alert-box` Z. 343: `padding: 13px 15px`
- `.news-card` Z. 488: `padding: 11px 13px`
- `.mv-row` Z. 313: `padding: 9px 14px`
- `.pwm` Z. 1272: `padding: 13px 12px`
- `.scan-banner` Z. 1260: `padding: 11px 14px`
- `--chip-m-pad` Z. 41: `7px 13px` (immerhin tokenisiert)

Es gibt **keinerlei Spacing-Tokens** (nur Radius und Chips). Empfehlung: 4er-Basisraster `--sp-1..6: 4/8/12/16/20/24` und die ungeraden Werte auf den nächsten Rasterwert runden — bei 13↔12 und 11↔12 sieht das kein Auge, aber die Datei hört auf, bei jeder neuen Karte zu würfeln.

---

## 4. Schatten & Tiefe: ~24 distinct Definitionen, 5 Ebenen würden reichen

Vollständige Liste (dedupliziert):

| Ebene | Wert | Wo |
|---|---|---|
| **Karte (Token)** | `0 2px 6px …05, 0 18px 40px …10` | `--shadow-card`, 20+ Nutzungen — gut |
| Karte (Abweichler) | `0 1px 1px …03, 0 4px 16px …05` | `.ov-card` Z. 301 — einziger Ausreißer vom Token |
| **Glas schwebend** | `0 10px 34px rgba(0,0,0,.22) + inset` | `.tabbar` Z. 184, `#toast` Z. 1009 — konsistent |
| **Foto-Thumbnail** | 4 Varianten: `.gph` Z. 544, `.gph-none` Z. 549, `.d-photos img`/`.holo-wrap img` Z. 649/661, `.res-photo img` Z. 1227 | vier verschiedene Rezepte für dieselbe Sache: „Karte wirft Schatten" |
| **CTA** | `0 1px 2px + 0 8px 20px -6px + inset` | `.btn-primary` Z. 129 |
| Hero | `0 2px 4px + 0 18px 44px -12px + inset` | `.hero` Z. 406 |
| Scan-Orb | `0 0 0 4px var(--bg) + 0 10px 24px` **mit `!important`** | `.tab-cam` Z. 209 |
| **Modal** | `0 30px 80px …4` / `0 -12px 50px …18` | `.party-card` Z. 1116, `.sheet` Z. 1032 |
| Klein | `0 1px 3px` (Z. 323, 741), `0 1px 4px` (Z. 793, 1082), `0 2px 6px` (Switch Z. 778) | Mikro-Streuung |
| Fokus | `0 0 0 2.5px var(--tint)` | Inputs Z. 108, 1048 — konsistent |

Konsolidierung: `--shadow-card` (existiert), `--shadow-photo` (eine Definition statt vier), `--shadow-float` (Glas), `--shadow-modal`, `--shadow-micro`. Die zwei `!important` am Scan-Orb (Z. 208–209) entfernen — nichts konkurriert dort.

## 5. Radius-System: 28 Werte, Tokens existieren, werden aber ignoriert

`--radius-card: 20px` und `--radius-sheet: 34px` existieren (Z. 34–35) — aber `var(--radius-card)` wird **nur 2×** benutzt. Daneben: 3, 4, 6, 7, 8, 9, 10, 11(5×), 12(10×), 13, 14(13×), 16(12×), 18(3×), 20, 24, 26, 30, 100px, 999, 50 %, und `4.5% / 3.2%`. Die 11er und 13er direkt neben 12 und 14 sind reiner Wildwuchs (`.search` 11, `.qbox input` 11, `.btn-secondary` 13, `.sell-head img` 11).

Konkreter Bruch: **`.gph` (Foto, Z. 544) hat 10px, `.gph-none` (Platzhalter, Z. 548) `4.5%/3.2%`** — Foto und Platzhalter im selben Slot haben unterschiedliche Eckgeometrie. Und der Glanz-Lauf `.gitem::after` (Z. 595) hat 12px über einem 10px-Foto.

Ziel-Skala: `--r-thumb:10` · `--r-control:12` (Inputs, Suche, Seg) · `--r-row:14` (ilist, sale-row) · `--r-card:20` (alle Karten; 16/18 → 20) · `--r-bar:26` · `--r-sheet:34` + 999/50 %.

## 6. Dark Mode: designt — mit drei Rissen

Positiv und klar „designt": eigene Flächen-Treppe (#000 Bühne → #141519 Karten), eigener Schatten-Token, Tint aufgehellt auf `#7aa8ee`, Kerzen auf iOS-Dark-Werte (`#30d158`/`#ff453a`), Tab-Bar bekommt eigene Specular-Kante (sero-dark.css Z. 58–60). Die Logo-Invertierung (`brightness(0) invert(1)`) ist für monochrome Marken legitim.

Die Risse:
1. **Toter Konflikt-Block** in sero.css (Punkt 2.2) — größtes Risiko.
2. **`.big-chart` wird im Dark Mode türkis:** `#4fd1b5` (Z. 1719–1720). Ein Teal-Akzent, der nirgendwo sonst in der App existiert — hell ist die Linie Marken-Navy (`--tint`). Das ist die eine Stelle, die „invertiert statt designt" wirkt. → `color: var(--tint)`.
3. **Drittes Grün/Rot-Paar:** `.ov-delta` dark = `#3ddc97`/`#ff8a80` (sero-dark Z. 50–51) neben `--green #30d158`/`--red #ff453a`.

## 7. Cheap-Look-Killer: die 10 konkretesten Fixes (Wirkung pro Aufwand)

**1. Fehlende Grid-Klassen nachrüsten — das JS rendert Unstyled-HTML.** `renderCollection` erzeugt `.gstat` (Status-Wort „Live/Entwurf/Verkauft/Wunsch", sero.js Z. 2061) und `.gmore` (Regler-Icon, Z. 2082) — **beide existieren in keiner CSS-Datei.** Der Status erscheint als nacktes Fließtext-Wort in der Kachel, das Icon hängt unpositioniert im Fluss zwischen Foto und Textblock. Fix in sero.css hinter Z. 588:
```css
.gstat { display: block; font-size: var(--chip-s-fs); font-weight: 700;
  letter-spacing: .2px; margin-bottom: 2px; color: var(--label-2); }
.gstat::before { content: "● "; font-size: 8px; vertical-align: 1px; }
.gstat.live { color: var(--green); }  .gstat.draft { color: var(--orange); }
.gstat.sold { color: var(--label-3); }  .gstat.wish { color: #e0518e; }
.gmore { position: absolute; top: 10px; right: 10px; z-index: 2;
  width: 26px; height: 26px; border-radius: 50%; display: grid; place-items: center;
  color: #fff; background: rgba(16, 24, 40, .45); backdrop-filter: blur(4px); }
.gfav { top: 42px; }  /* Stern unter den Regler, sonst Überlappung (beide top:10px right:10px) */
```

**2. Login-Feature-Liste stylen — der erste Screen jedes neuen Nutzers.** `index.html` Z. 32–36 rendert `<ul class="login-feats">` mit `.lf-ic` und Z. 50 `.login-link` — **alle drei ohne CSS-Regel**: eine Browser-Standard-Bullet-Liste mitten im Login. Fix hinter `.login-sub` (Z. 96):
```css
.login-feats { list-style: none; margin: 0 auto 26px; width: fit-content;
  display: flex; flex-direction: column; gap: 10px; text-align: left;
  font-size: 15px; font-weight: 500; color: var(--label-2);
  animation: pageIn .5s .2s var(--spring) backwards; }
.login-feats li { display: flex; align-items: center; gap: 10px; }
.lf-ic { width: 28px; height: 28px; border-radius: 8px; flex: 0 0 auto;
  display: grid; place-items: center; background: var(--tint-soft); color: var(--tint); }
.login-link { color: var(--tint); font-weight: 600; }
```

**3. `--muted` reparieren (Einzeiler).** sero.css Z. 1219: `color: var(--muted)` → `color: var(--label-2)`. Betrifft den Untertitel im Scan-Ergebnis — dem wichtigsten Moment der App.

**4. Swipe-Reveal sichtbar machen.** `renderCollection` setzt `.rev-fav`/`.rev-more`/`.armed` (sero.js Z. 2113–2118, Kommentar: „Aktion hinter der Kachel aufdecken, damit man SIEHT was passiert") — **keine der drei Klassen hat CSS.** Der Nutzer wischt, die Kachel verschiebt sich, dahinter ist: nichts. Minimal-Fix ohne JS-Umbau (Feedback am Objekt statt dahinter):
```css
.gitem.armed { box-shadow: 0 0 0 3px color-mix(in srgb, var(--tint) 45%, transparent), var(--shadow-card); }
.gitem.rev-fav.armed { box-shadow: 0 0 0 3px color-mix(in srgb, #f5a623 55%, transparent), var(--shadow-card); }
```
Sauber wäre ein Wrapper-Element pro Kachel mit Icon dahinter — das ist aber ein JS-Eingriff, kein CSS-Fix.

**5. Dark-Blau vereinheitlichen + toten Block löschen.** sero.css Z. 1625–1669 (kompletter Dark-Token-Block) entfernen — sero-dark.css ist die Wahrheit. Danach Z. 1701/1704: `#2c5aa5` → `#2b5ea8` (bzw. beide auf einen Token `--tint-solid-dark` ziehen). Ergebnis: ein Dark-Blau statt drei.

**6. Dark-Chart auf Marke: ** Z. 1719–1720 `.big-chart { color: #4fd1b5 }` → `color: var(--tint)`. Das Portfolio-Chart ist das Herzstück des Home-Tabs; türkis ist dort Fremdkörper.

**7. Foto-/Platzhalter-Ecken angleichen.** Z. 548 `.gph-none { border-radius: 4.5% / 3.2% }` → `border-radius: 10px` (wie `.gph`), Z. 595 `.gitem::after { border-radius: 12px }` → `10px`. Drei Werte, ein Blick ruhiger.

**8. `.ov-delta` auf das Token-Muster ziehen.** Z. 275–276 (und Dark-Kopien Z. 1726–1727, sero-dark Z. 50–51):
```css
.ov-delta.up { background: color-mix(in srgb, var(--green) 12%, transparent); color: var(--green); }
.ov-delta.down { background: color-mix(in srgb, var(--red) 10%, transparent); color: var(--red); }
```
Entfernt 8 Hardcodes in drei Dateien; Dark Mode stimmt automatisch.

**9. `.gname`-Doppel-Deklaration auflösen.** Z. 577–582: `min-height: 2.6em` wird vier Zeilen später von `min-height: 0` überschrieben (offensichtliches Versehen — `.rc-n` Z. 374 behält sein 2.5em). Folge: Kacheln mit einzeiligem Namen haben die Wertzeile höher als der Nachbar in derselben Grid-Reihe. `min-height: 0;` löschen.

**10. JS-Inline-Grau/Grün auf Tokens.** In `renderDetail`: `#8e8e93` (6×) → einheitlicher `.ric.neutral { background: var(--label-3) }`, `#34a853` (3×) → `var(--green)`. Zwei Suchen-und-Ersetzen, und der Detail-Screen folgt im Dark Mode der Palette statt Google-Grün zu zeigen.

**Kleinkram mit Minutenaufwand:** leere Regel `#salesSeg { }` (Z. 1729) löschen; `index.html` hat doppelte `apple-mobile-web-app-capable`-Metas (Z. 6+14), zwei verschiedene `apple-touch-icon`s (Z. 5: apple-touch-icon.png, Z. 17: app-icon.png) und zwei widersprüchliche `status-bar-style`-Metas (Z. 8 `default`, Z. 15 `black-translucent`) — je eins behalten; `letter-spacing` mischt px (Haupt-CSS) und em (sv2-Block) — auf em normalisieren, wenn ohnehin die Typo-Tokens kommen.

## Fazit

Die App ist visuell weiter als 90 % der PWAs — Tab-Bar, Motion, Ziffern-Disziplin und der designte Dark Mode sind real Premium. Was sie zurückhält, ist kein Geschmacksproblem, sondern Buchhaltung: **fünf Klassen, die das JS rendert, haben schlicht kein CSS** (Fixes 1, 2, 4 — vermutlich bei der v23-Konsolidierung verloren gegangen), und die Skalen (30 Schriftgrößen, 19 Blautöne, 28 Radien) sind über 43 Versionen gewachsen statt entschieden. Die zehn Fixes oben sind zusammen unter einem Tag Arbeit; die Token-Konsolidierung (Punkte 1–5) ist der eine strukturelle Schritt, der jede künftige Version billiger und konsistenter macht.