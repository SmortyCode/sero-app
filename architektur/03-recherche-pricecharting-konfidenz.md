# SERO — PriceCharting-API & Match-Konfidenz

**Hinweis zur Quellenlage:** `pricecharting.com` liefert dem Fetch-Tool HTTP 403. Die Doku-Inhalte unten stammen aus einem Text-Proxy-Abruf derselben Seite (`https://r.jina.ai/https://www.pricecharting.com/api-documentation`) und wurden gegen die **wortgleiche Schwester-Doku** von SportsCardsPro (gleicher Betreiber, kartenfokussiert) gegengeprüft — beide Quellen stimmen in allen Feldbedeutungen überein. Am Code wurde nichts geändert, kein Server gestartet, kein POST abgesetzt.

---

## Teil A — PriceCharting-API im Detail

### A1 Endpunkte, Auth, Limits, Kosten

| Punkt | Fakt | Quelle |
|---|---|---|
| Basis-URL | `https://www.pricecharting.com` | api-documentation |
| Auth | 40-Zeichen-Token als Query-Parameter `t=`; statisch pro Abo; zu finden auf der Subscription-Seite unter „API/Download" | api-documentation |
| `GET /api/product` | **genau ein** Produkt; Parameter `id=`, `upc=` **oder** `q=` (Volltext) | api-documentation |
| `GET /api/products` | **bis zu 20** Treffer; nur `q=`; liefert reduzierte Felder (`id`, `product-name`, `console-name`) | api-documentation |
| Marketplace | `/api/offers`, `/api/offer-details`, `/api/offer-publish`, `/api/offer-ship`, `/api/offer-feedback`, `/api/offer-end`, `/api/offer-refund` | api-documentation |
| Antwortformat | JSON mit `"status": "success"` / `"error"`; Datumsangaben `YYYY-MM-DD`; **Preise als Integer in US-Cent** (17,32 $ = `1732`) | api-documentation |
| Rate Limit | *„The API is limited to 1 call every second. Any more than that and your calls will be blocked and your account permissions revoked if it persists."* | api-documentation |
| CSV-Limit | *„CSV calls are limited to one every 10 minutes."* — Vollabzug einmal je 24 h, Spalten = API-Keys | api-documentation |
| Preis | Free 0 $ · Collector Sub 6 $/Monat · **Legendary Sub 49 $/Monat** — API und CSV-Vollabzug ausschließlich in Legendary | pricecharting-pro |

**Skalierungs-Konsequenz für „zehntausende Nutzer":** 1 Anfrage/Sekunde ist ein hartes Dach von 86.400 Abfragen/Tag — für das *gesamte* Konto, nicht pro Nutzer. Der Katalog-Cache in `/Users/smorty/ebay-bot/web/catalog.py` ist damit keine Optimierung, sondern Voraussetzung. Ab echter Nutzerzahl ist der **CSV-Vollabzug (24 h, Legendary)** der eigentlich vorgesehene Weg: einmal täglich alle Produkte + Preise lokal spiegeln, dann Matching komplett offline — das umgeht Rate Limit *und* Latenz *und* macht das Ranking (siehe Teil B) beliebig teuer rechenbar.

### A2 Die Preisfelder — was sie WIRKLICH bedeuten

Ein Feld heißt bei Videospielen etwas völlig anderes als bei Karten und nochmals anderes bei Comics. Das ist der zentrale Fallstrick:

| JSON-Key | Videospiele | **Sammelkarten** | **Comics/Manga** |
|---|---|---|---|
| `loose-price` | Nur Modul/Disc | **Ungraded** | Ungraded |
| `cib-price` | Complete in Box | **Graded 7 / 7.5** | Graded 4.0 / 4.5 |
| `new-price` | Originalverpackt/versiegelt | **Graded 8 / 8.5** | Graded 6.0 / 6.5 |
| `graded-price` | WATA/VGA-graded | **Graded 9** | Graded 8.0 / 8.5 |
| `box-only-price` | Nur Originalkarton | **Graded 9.5** | Graded 9.2 |
| `condition-17-price` | – | **CGC 10** | Graded 9.4 |
| `manual-only-price` | Nur Originalanleitung | **PSA 10** | Graded 9.8 |
| `bgs-10-price` | – | **BGS 10** | Graded 10.0 |
| `condition-18-price` | – | **SGC 10** | – |

Nicht-Preisfelder: `id`, `product-name`, `console-name`, `genre`, `release-date` (`YYYY-MM-DD`), `upc`, `asin`, `epid` (eBay-Produkt-ID!), `sales-volume` (verkaufte Stück/Jahr), `gamestop-price`, sowie `retail-loose-buy/sell`, `retail-cib-buy/sell`, `retail-new-buy/sell`.

**Damit bestätigt: es gibt kein Feld für 9.0 vs. 9.4 bei Karten.** Die Karten-Leiter ist 7/7.5 → 8/8.5 → 9 → 9.5 → 10. Eine Beckett 9.0 und eine Beckett 9.4 fallen beide auf `graded-price`. Die lineare Interpolation in `/Users/smorty/ebay-bot/web/pricecharting.py:49-73` ist deshalb sachlich die richtige Antwort auf ein echtes Datenlücken-Problem.

### A3 Drei belegbare Fehler im aktuellen Code

**(1) Falsche Feldbedeutung im Docstring** — `/Users/smorty/ebay-bot/web/pricecharting.py:6` behauptet `condition-17/18 = CGC 10 / CGC 9.5`. Laut Doku ist `condition-18-price` = **SGC 10**, nicht CGC 9.5. Ein CGC-9.5-Feld existiert überhaupt nicht (eine CGC 9.5 fällt auf `box-only-price`, das generisch „Graded 9.5" ist). Der Code selbst nutzt `condition-18` nirgends, es ist also nur die Doku — aber sie führt beim nächsten Umbau in die Irre.

**(2) Die Grade-Leiter ist für Comics/Manga schlicht falsch** — das ist der *zweite*, bisher unentdeckte Teil von Svens Manga-Bug. `_grade_fields()` in `/Users/smorty/ebay-bot/web/pricecharting.py:21-46` kennt nur die Karten- und die Videospiel-Leiter. Selbst wenn das Matching repariert wird und PriceCharting korrekt „One Piece Vol. 103 [Paperback]" liefert, würde eine Beckett/CGC 9.0 über die Kartenleiter auf `graded-price` gemappt — bei Comics bedeutet dieses Feld aber **8.0/8.5**. Für Comics wäre korrekt: 9.0 liegt zwischen `graded-price` (8.5) und `box-only-price` (9.2). Es braucht eine dritte Leiter, gewählt anhand der Kandidaten-Domäne.

**(3) Der Cache-Schlüssel verhindert Korrekturen** — `/Users/smorty/ebay-bot/web/pricecharting.py:81` cached unter `sha1(query)`. Ein einmal falsch gematchtes Produkt bleibt damit 12 h an der Anfrage kleben und ist nicht gezielt invalidierbar. Besser: zweistufig cachen — `query → pc_id` (kurze TTL, das ist die unsichere Zuordnung) und `pc_id → Produktdaten` (lange TTL, das ist der stabile Fakt).

### A4 Gibt es einen Match-Score? Nein — und das ist die eigentliche Ursache

**Es existiert kein Relevanz-, Score- oder Confidence-Feld in der API.** Die Doku sagt zur Suche nur: *„Search for a product based on its title and/or console. If multiple products match the search, only the best matches are returned."* Kein numerischer Wert, keine Trefferzahl, keine Alternativen.

`/api/product?q=` gibt also **einen** Treffer zurück, ohne jedes Signal, ob er passt — und `/Users/smorty/ebay-bot/web/pricecharting.py:87-92` behandelt jedes Ergebnis mit einer `id` als gültig. Genau hier entstand der 603-€-Fehler.

**Der strukturelle Fix: `/api/products` statt `/api/product`.** Bis zu 20 Kandidaten mit `id`, `product-name` und `console-name` — genug, um selbst zu ranken, und der einzige Weg, Mehrdeutigkeit überhaupt zu *sehen*.

### A5 Welche Felder machen Svens Fehl-Match maschinell erkennbar?

Ich habe die PriceCharting-Suche für Svens Fall nachgestellt (`search-products?q=one+piece+manga+103`). Ergebnis:

- Die Trefferliste **mischt Domänen**: One-Piece-TCG-Karten (`OP13-118` etc.) *und* das Buch.
- **Das richtige Produkt existiert**: „One Piece Vol. 103 [Paperback] (2022)".
- Die Domäne steckt in **`console-name`**: das Buch liegt unter **„Comic Books One Piece"**, die Karten unter TCG-Set-Namen wie „One Piece Romance Dawn", „One Piece Paramount War", „One Piece Carrying on His Will". Bestätigt durch die Kategorie-Seiten selbst: `https://www.pricecharting.com/console/comic-books-one-piece`, `.../console/comic-books-witchblade-manga`.

**Also: `console-name` ist das entscheidende Feld.** Es trägt bei PriceCharting die Domäne — „Comic Books …" für Bücher/Manga, echte Konsolennamen für Spiele, Set-Namen für TCG. Der Fehl-Match Buch→Karte wäre mit einer einzigen Prüfung auf das Präfix `Comic Books` verhindert worden. („console" ist hier ein historischer Name aus der Videospiel-Herkunft der Seite, kein Konsolenfeld.)

Weitere brauchbare Diskriminatoren: `release-date` (Jahr, Band 103 = 2022), `upc` (bei Büchern ISBN-nah, bei Einzelkarten nie vorhanden), `genre`, `epid` (direkter Abgleich gegen eBay möglich). *Unbestätigt:* ob `genre` bei Karten/Comics befüllt ist und ob die Antwort ein undokumentiertes Bild-Feld enthält — die dokumentierte Key-Tabelle enthält **kein** Bild.

*Unbestätigt:* ob `/api/product?q=` exakt dasselbe Ranking benutzt wie die Website-Suche. Für die Fehleranalyse spielt das keine Rolle — beide liefern gemischte Domänen.

---

## Teil B — Etablierte Verfahren zur Match-Konfidenz

**String-Ähnlichkeit.** Levenshtein (Editierdistanz, gut bei Tippfehlern), Jaro-Winkler (gewichtet gemeinsame Präfixe, Standard bei Namen), und Token-Verfahren. Für Produkttitel ist `token_set_ratio` der praktische Standard: er vergleicht Wortmengen und gibt 100, wenn eine Menge Teilmenge der anderen ist — robust gegen Zusätze wie „[Paperback] (2022)". Achtung, genau diese Toleranz ist auch die Gefahr: „One Piece" ⊂ „One Piece Vol. 103" ergibt 100. Token-Ähnlichkeit darf deshalb **nie allein** entscheiden, sondern nur zusammen mit harten Identifikatoren.

**Feld-gewichtete Scores.** Der Lehrbuchstandard ist das **Fellegi-Sunter-Modell** der probabilistischen Record Linkage: pro Vergleichsfeld werden `m` (Übereinstimmungswahrscheinlichkeit bei echtem Match) und `u` (bei Nicht-Match) geschätzt; das Feldgewicht ist `log2(m/u)` — „Bits an Evidenz". Die Gewichte werden **addiert** (Annahme: Felder unabhängig) und mit einem Prior verrechnet; zwei Schwellen trennen Match / unklar / Nicht-Match. Referenzimplementierung: Splink.

**Perceptual Image Hashing.** pHash/dHash erzeugen 64-Bit-Fingerabdrücke, verglichen per Hamming-Distanz. Praxisempfehlung: konservativ bei ≤ 5 starten, je nach Datensatz bis 10 öffnen. pHash und dHash sind die einzigen gängigen Verfahren mit normalverteilten Distanzen, also mit vorhersagbarem Schwellverhalten — für ein kalibriertes System der richtige Griff.

**Embedding-Vergleiche.** CLIP-Bild-Embeddings plus Kosinus-Ähnlichkeit gegen Katalogbilder; robuster gegen Beleuchtung/Winkel/Slab-Reflexionen als pHash, aber teurer und ohne allgemeingültige Schwelle — die muss man am eigenen Datensatz bestimmen.

**Abstention / „lieber keine Antwort".** Das ist formal **selective prediction** mit reject option. Die klassische Grundlage ist **Chows Regel (1970)**: bei Kosten für eine Falschantwort gegenüber Kosten für eine Nicht-Antwort ist es optimal, nur oberhalb einer Konfidenzschwelle zu antworten. Modern als Risk-Coverage-Tradeoff formuliert (Geifman/El-Yaniv 2017). Damit die Schwelle interpretierbar ist, muss der Score **kalibriert** sein — Platt-Scaling (Sigmoid) oder isotonische Regression auf einem Holdout-Set, in scikit-learn `CalibratedClassifierCV`. Bei wenigen Labeln ist Platt vorzuziehen, isotonisch überanpasst schnell.

---

## Teil C — Konkretes Konfidenz-Modell für SERO

### C0 Architektur-Vorbedingung

1. `/api/products?q=` statt `/api/product?q=` — 20 Kandidaten holen (`/Users/smorty/ebay-bot/web/pricecharting.py:87`).
2. Jeden Kandidaten scoren, Top-1 nehmen, Top-2 als Ambiguitätsmaß behalten.
3. Für den Gewinner einmal `/api/product?id=<id>` für die vollen Preisfelder. Kosten: 2 Calls statt 1 — bei 1 Call/s durch den Katalog-Cache tragbar.

### C1 Stufe 1 — Domänen-Gate (hart, keine Punkte)

Domäne des Kandidaten aus `console-name`:

```
console_name.startswith("Comic Books")        → comic
console_name in KONSOLEN_LISTE               → game        # Playstation, Nintendo 64, Xbox, Game Boy, ...
sonst                                        → card
```

Domäne des Items aus der bereits vorhandenen Kategorisierung (`guess_category`, `/Users/smorty/ebay-bot/web/app_api.py:173-180`) plus Vision-Ausgabe. Regel:

> **`domain(kandidat) != domain(item)` → Konfidenz 0, Kandidat verworfen, keine weitere Rechnung.**

Das allein hätte Svens 603 € verhindert. Und dieselbe Domäne wählt anschließend die richtige Grade-Leiter (Karte / Comic / Videospiel) in `_grade_fields()` — siehe A3(2).

Sofort-Match: `upc` identisch oder `epid` identisch → Konfidenz 1.0, Rest überspringen.

### C2 Stufe 2 — Additive Evidenz in Bits (Fellegi-Sunter-Stil)

```
S = w0 + Σ w_i
P = 1 / (1 + 2^(-S))          # Sigmoid über log2-Odds
```

| # | Merkmal | Bedingung | Gewicht (Bits) |
|---|---|---|---|
| — | **Prior** `w0` | immer | **−3.0** |
| 1 | **Hart-ID** (Kartennr. `OP01-016`, Band `Vol. 103`, Set-Code) | im `product-name` enthalten | **+6.0** |
| | | Item hat eine, fehlt im Namen | **−6.0** |
| | | Item hat keine | 0 |
| 2 | **Namensähnlichkeit** `token_set_ratio(item.name, product-name)` | ≥ 95 | +3.0 |
| | | 85–94 | +1.5 |
| | | 70–84 | 0 |
| | | 50–69 | −2.0 |
| | | < 50 | −4.0 |
| 3 | **Set/Serie** `token_set_ratio(item.set_name, console-name)` | ≥ 90 | +3.0 |
| | | 70–89 | +1.0 |
| | | 40–69 | 0 |
| | | < 40 | −2.5 |
| 4 | **Jahr** `abs(release-date.year − item.year)` | ≤ 1 | +1.5 |
| | | 2–3 | 0 |
| | | > 3 | −2.0 |
| 5 | **Bild** Hamming(pHash Foto, pHash Katalogbild), 64 Bit | ≤ 8 | +6.0 |
| | | 9–12 | +2.0 |
| | | 13–18 | 0 |
| | | > 18 | −3.0 |
| 6 | **Preis-Plausibilität** `r = max(pc/markt, markt/pc)` gegen 130point/eBay-Median | ≤ 1.5 | +2.0 |
| | | 1.5–3 | 0 |
| | | 3–6 | −3.0 |
| | | > 6 | −6.0 |
| 7 | **Sprache** (Item JP/DE ↔ `console-name` enthält „Japanese"/„German") | stimmig | +1.0 |
| | | widersprüchlich | −2.0 |

Fehlende Daten ⇒ Gewicht 0 (kein Beweis ist kein Gegenbeweis). Merkmal 5 muss gegen **TCGdex/Scryfall/YGOPRODeck-Bilder** aus `/Users/smorty/ebay-bot/web/prices.py` laufen, nicht gegen PriceCharting — die API dokumentiert **kein** Bildfeld (A5).

**Nachrechnung Svens Fall (ohne Domänen-Gate, um die Robustheit zu zeigen):**
Item „One Piece Manga Band 103", Kandidat „Nami [Manga] OP01-016" / console „One Piece Romance Dawn".
`w0 −3.0` · Hart-ID „103" fehlt im Namen `−6.0` · Name-Ratio ≈ 40 `−4.0` · Set-Ratio niedrig `−2.5` · Jahr Set 2022 vs. Buch 2022, unauffällig `0` · kein Bildvergleich `0` · Preis 603 € vs. Marktmedian ~8 € → r ≈ 75 `−6.0`
**S = −21.5 → P ≈ 0.0000003.** Sicher verworfen, mit oder ohne Gate.

### C3 Stufe 3 — Schwellen und Abstention

| P | Verhalten |
|---|---|
| **≥ 0.90** | Wert wird benutzt, Anzeige „PriceCharting-Verkäufe" |
| **0.65 – 0.90** | nur als Rückfall, wenn keine bessere Quelle da ist; Anzeige „Zuordnung unsicher" (heutiges `pricecharting_weak`) |
| **< 0.65** | verworfen — **keine Zahl ist besser als eine falsche** |

**Zusätzliche Ambiguitäts-Regel:** `P₁ − P₂ < 0.15` (Top-1 vs. Top-2 aus `/api/products`) → verwerfen, auch bei hohem P₁. Zwei fast gleich gute Kandidaten heißt: die Suche kann es nicht entscheiden, und dann darf SERO es auch nicht.

**Woher kommt die 0.90?** Aus Chows Regel, nicht aus dem Bauch. Ist eine falsche Preisangabe `k`-mal so teuer wie eine fehlende, lautet die optimale Schwelle `τ = k / (k + 1)`. Für ein Produkt, das automatisch eBay-Listings erzeugt, ist ein falscher Preis leicht das Neunfache eines fehlenden wert (Fehlverkauf, Reklamation, Vertrauensverlust) → `k = 9` → `τ = 0.9`. Sven kann `k` bewusst wählen und die Schwelle fällt heraus — das ist der Punkt, an dem eine Geschäftsentscheidung eine Zahl wird.

### C4 Kalibrierung

Die Bit-Gewichte oben sind **begründete Startwerte, keine gemessenen**. Der saubere Weg:

1. 200–500 Paare (Item, PC-Kandidat) aus der bestehenden Historie labeln — richtig/falsch.
2. Logistische Regression auf den sieben Merkmalen → die Koeffizienten *sind* dann die empirischen log-Odds-Gewichte, geteilt durch `ln 2` für Bits.
3. Platt-Scaling auf einem separaten Holdout (bei dieser Datenmenge nicht isotonisch — überanpasst).
4. Risk-Coverage-Kurve zeichnen und `τ` daran ablesen statt zu raten.

Bis dahin fährt das Modell mit den Startwerten und dem harten Domänen-Gate — das Gate trägt den Fehlerfall auch ohne jede Kalibrierung.

### C5 Was das gegenüber heute besser macht

Die bestehende Prüfung in `/Users/smorty/ebay-bot/web/catalog.py:149-174` (`pc_trusted`: alle Zahlen-Tokens der Anfrage müssen im PC-Namen vorkommen) ist ein guter Reflex, aber **binär und einseitig** — sie kennt keine Abstufung, ignoriert die Domäne, hat keine Bilder und kein Ambiguitätsmaß. Die Faktor-4-/Faktor-6-Wächter in `catalog.py:212-240` und `/Users/smorty/ebay-bot/web/app_api.py:833-845` sind nachgelagerte Notbremsen, die erst greifen, wenn ein unabhängiger Marktwert existiert. Im Modell oben wird aus derselben Information Merkmal 6 — abgestuft statt Alles-oder-Nichts, und sie wirkt *vor* der Preisauswahl statt danach.

---

## Quellen

- [PriceCharting API Documentation](https://www.pricecharting.com/api-documentation) (abgerufen über Text-Proxy, s. o.)
- [PriceCharting Premium — Tarife](https://www.pricecharting.com/pricecharting-pro)
- [SportsCardsPro API Documentation](https://www.sportscardspro.com/api-documentation) — kartenspezifische Bestätigung aller Feldbedeutungen
- [PriceCharting: Comic Books One Piece](https://www.pricecharting.com/console/comic-books-one-piece) — Beleg für `console-name`-Präfix
- [PriceCharting: How We Get Our Prices](https://www.pricecharting.com/page/methodology)
- [Splink — The Fellegi-Sunter Model](https://moj-analytical-services.github.io/splink/topic_guides/theory/fellegi_sunter.html)
- [RapidFuzz — fuzz.token_set_ratio](https://rapidfuzz.github.io/RapidFuzz/Usage/fuzz.html)
- [Hamming distributions of popular perceptual hashing techniques (DFRWS/ScienceDirect)](https://www.sciencedirect.com/science/article/pii/S2666281723000100)
- [pHash — How To](https://phash.org/docs/howto.html)
- [Selective Abstention in Machine Learning (Chow's rule, Risk-Coverage)](https://www.emergentmind.com/topics/selective-abstention)
- [scikit-learn Probability Calibration / Platt vs. Isotonic](https://machinelearningmastery.com/calibrated-classification-model-in-scikit-learn/)
- [Roboflow — CLIP Embeddings](https://inference.roboflow.com/foundation/clip/)