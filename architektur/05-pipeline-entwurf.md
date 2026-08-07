# SERO — Preis-Pipeline als strikter Entscheidungsbaum

*Entwurf 03.08.2026. Strikt read-only gearbeitet: nur Dateien gelesen und `import`-Prüfungen im venv gefahren. Keine Datei geändert, kein Server gestartet, kein Request gegen localhost, keine DB-Schreiboperation.*

---

## 0. Leitprinzipien

1. **Identität vor Preis.** Kein Preis wird gezogen, bevor feststeht *was* das Stück ist. Der 603-€-Fehler ist kein Preisfehler, sondern ein Identitätsfehler, der zu spät auffiel.
2. **Jede Stufe hat ein Gate.** Ein Gate hat genau drei Ausgänge: *weiter*, *degradiert weiter (Spanne statt Zahl)*, *Abbruch mit Grund*. Es gibt keinen vierten Ausgang „irgendwas raten".
3. **Belegklasse ist ein eigenes Feld, nicht nur ein Label.** Die App muss maschinell wissen, ob eine Zahl auf Verkäufen, auf Angeboten oder auf einem Fremdkatalog beruht.
4. **KI-Schätzung ist niemals ein Wert.** Sie ist ein *Plausibilitätsband*. Heute wird sie an drei Stellen als `est_value` geschrieben (`/Users/smorty/ebay-bot/web/app_api.py:852-855`, `:982-989`, `:842-844`). Das widerspricht Svens Regel direkt.
5. **Der Katalog ist die Wahrheit, nicht das Item.** Korrekturen, die nur am Item landen, vergiften die Katalogzeile für alle anderen (heute: `catalog.py` schreibt 603 €, `app_api.py:833-845` korrigiert nur das Item).

### 0.1 Neue Felder am Item (Vertrag für alle Stufen)

Alle müssen in `PREIS_FELDER` bzw. `ANALYSE_FELDER` eingetragen werden (`/Users/smorty/ebay-bot/web/app_api.py:725-734`) — sonst verwirft `col_save_analyse` sie stillschweigend.

| Feld | Werte | Gesetzt in Stufe |
|---|---|---|
| `item_class` | `tcg_card` \| `sport_card` \| `comic_manga` \| `video_game` \| `sealed` \| `other` | S2 |
| `holder` | `slab` \| `sleeve` \| `raw` \| `sealed` \| `unknown` | S2 |
| `graded` | `{grader, grade, cert_number, cert_verified, scale_ok, quelle}` | S3 |
| `identity_conf` | 0.0–1.0 | S4 |
| `price_state` | `belegt` \| `spanne` \| `unbekannt` | S6 |
| `price_reason` | Enum, siehe §9 | S6 |
| `price_evidence` | `E1`…`E5` (Belegklasse, §7) | S5 |
| `price_low` / `price_high` | Spanne, wenn `price_state != belegt` | S5/S6 |

`est_value` bleibt, bedeutet aber ab sofort: **nur gesetzt, wenn `price_state == "belegt"`.** Sonst `None`.

---

## 1. Gesamtbaum (ASCII)

```
FOTO(S)
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ S1  AUFNAHME & VORVERARBEITUNG                    web/cardscan.py   │
│  1.1 EXIF-Aufrichten, HEIC → RGB                                    │
│  1.2 Gruppierung Vorder-/Rückseite (group_photos)                   │
│  1.3 Freisteller (rembg/BiRefNet) → _cut.png                        │
│  1.4 Slab-Entzerrung (slab_recut, Perspektiv-Warp + Plausibilität)  │
│  1.5 NEU: Label-Band-Crop (oberer Streifen des Slabs, 4-6× Pixel)   │
│  1.6 NEU: Glanz-Score (HSV: hell+entsättigt) je Bildregion          │
└─────────────────────────────────────────────────────────────────────┘
  │  Glanz-Score Label > Schwelle UND nur 1 Foto?
  ├──── ja ──► Zweitfoto anfordern (Capture-Guidance) ─┐
  │                                                     │ kein Zweitfoto?
  ▼                                                     ▼  → S3 mit C_bild=0.6
┌─────────────────────────────────────────────────────────────────────┐
│ S2  KLASSIFIZIERUNG                     cardscan.py + claude_client │
│  holder ∈ {slab, sleeve, raw, sealed, unknown}                      │
│  item_class ∈ {tcg_card, sport_card, comic_manga, video_game,       │
│                sealed, other}                                        │
└─────────────────────────────────────────────────────────────────────┘
  │
  ├─ holder == unknown ODER item_class == other ──► ABBRUCH  UNBEKANNT_KLASSE
  │
  ├─ holder == slab ──────────────────────────────────────┐
  │                                                        ▼
  │   ┌──────────────────────────────────────────────────────────────┐
  │   │ S3  SLAB-LABEL LESEN                    NEU: web/slab.py     │
  │   │  3.1 QR-Code decodieren   (cv2.QRCodeDetector)  ── Kanal A   │
  │   │  3.2 Barcode decodieren   (cv2.barcode)         ── Kanal A   │
  │   │  3.3 LLM liest Label-Crop (grader/grade/cert)   ── Kanal B   │
  │   │  3.4 Grader kanonisieren (BGS/BVG/BCCG/CGC/…)                │
  │   │  3.5 Note gegen Skala DES GRADERS validieren                 │
  │   │  3.6 Cert-Format-Gate + optional PSA-Lookup                  │
  │   └──────────────────────────────────────────────────────────────┘
  │      │
  │      ├─ Note nicht auf der Skala des Graders ──► graded = null,
  │      │                                           holder → raw-Pfad,
  │      │                                           Flag GRADE_UNGUELTIG
  │      ├─ Grader unbekannt ODER Note fehlt ─────► graded = null (raw-Pfad)
  │      └─ ok ─► C_grade ∈ {1.0 (A∧B einig / Cert verifiziert),
  │                          0.85 (nur B, Format ok),
  │                          0.6 (nur B, Format unplausibel)}
  │                                                        │
  ├─ holder ∈ {sleeve, raw} ──► C_grade = 1.0, graded=null │
  ├─ holder == sealed ────────► C_grade = 1.0, graded=null │
  │                                                        │
  ▼◄───────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────┐
│ S4  ENTITY MATCHING                     NEU: web/match.py + prices  │
│  4.1 identify_card (Haiku) — jetzt MIT item_class als Vorgabe       │
│  4.2 Kandidaten je Klasse holen (Karten-DB / PC /api/products)      │
│  4.3 DOMÄNEN-GATE (hart): Kandidat.domain == item_class?            │
│  4.4 Bit-Scoring (Fellegi-Sunter) → P je Kandidat                   │
│  4.5 Ambiguitäts-Regel P₁ − P₂ ≥ 0.15                               │
│  4.6 identity_key deterministisch bilden (NIE solo:uuid)            │
└─────────────────────────────────────────────────────────────────────┘
  │
  ├─ kein Kandidat überlebt Domänen-Gate ──► ABBRUCH  UNBEKANNT_DOMAENE
  ├─ P₁ < 0.65 ────────────────────────────► ABBRUCH  UNBEKANNT_IDENTITAET
  ├─ P₁ − P₂ < 0.15 ───────────────────────► ABBRUCH  UNBEKANNT_MEHRDEUTIG
  └─ sonst ─► identity_conf = P₁
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ S5  BEWERTUNG — Quellen-Hierarchie nach BELEGKLASSE  web/catalog.py │
│  Katalog-Read (card_key × grade, TTL nach Belegklasse)              │
│  parallel: E1 eigene Verkäufe · E2 130point · E3 Karten-DB          │
│            E4 Browse(gefiltert) · E5 PriceCharting(domänengeprüft)  │
└─────────────────────────────────────────────────────────────────────┘
  │
  ├─ keine Quelle liefert ──► ABBRUCH  UNBEKANNT_KEINE_BELEGE
  └─ Gewinner nach §7 ─► value, evidence, C_quelle
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ S6  PLAUSIBILITÄTS-TOR              catalog.py + app_api.py         │
│  6.1 Referenz-Anker: raw-Zeile desselben card_key (IMMER vorhanden) │
│  6.2 Quellen-Widerspruch (Faktor-Test gegen 2. Quelle)              │
│  6.3 KI-Band-Test (estimated_price_range_eur)                       │
│  6.4 GRADE-MONOTONIE über den Katalog (§8)                          │
│  6.5 C_gesamt = C_klasse × identity_conf × C_grade × C_quelle       │
└─────────────────────────────────────────────────────────────────────┘
  │
  ├─ C_gesamt ≥ 0.90 und alle Tests bestanden ─► price_state = belegt
  │                                               est_value = Zahl
  ├─ 0.65 ≤ C_gesamt < 0.90 ODER ein Test knapp ► price_state = spanne
  │                                               est_value = None
  │                                               price_low/high gesetzt
  └─ sonst ─────────────────────────────────────► price_state = unbekannt
                                                  est_value = None
                                                  price_reason = <Grund>
```

---

## 2. Stufe S1 — Aufnahme & Vorverarbeitung

**Was heute schon da ist:** EXIF-Aufrichten und HEIC (`cardscan.py:38-45`, `:19-25`), Gruppierung Vorder-/Rückseite (`cardscan.py:341-378`), Freisteller mit Rechteck-Vorlage und Rotations-statt-Warp-Regel (`cardscan.py:152-256`), Slab-Entzerrung mit Seitenverhältnis- und Zerr-Schutz (`cardscan.py:287-338`, Prüfungen bei `:313` und `:320-323`).

**Was fehlt:** ein zweiter, enger Crop nur auf das Label-Band, und irgendeine Behandlung von Reflexionen.

| | |
|---|---|
| **Eingabe** | `item["photos"]` (Pfade), optional `photos_raw` |
| **Verarbeitung** | 1.1 `ImageOps.exif_transpose` + HEIC (vorhanden) · 1.2 `group_photos` (vorhanden) · 1.3 `crop_photos`/`_cutout` (vorhanden) · 1.4 `slab_recut` (vorhanden) · **1.5 NEU `label_crop()`**: aus dem entzerrten Slab den oberen Streifen (Höhenanteil aus dem bekannten Seitenverhältnis) als eigenes Bild · **1.6 NEU `glanz_score()`**: HSV-Maske hoch-V/niedrig-S über dem Label-Crop, Anteil als Zahl 0–1 |
| **Ausgabe** | `photos[]`, `photos_raw[]`, **NEU** `label_crop_path`, `bild_qualitaet = {glanz, schaerfe, kanten_ok}` |
| **Konfidenz** | `C_bild = 1.0` wenn `glanz < 0.10` und `slab_recut` erfolgreich; `0.8` bei `glanz 0.10–0.25`; `0.6` darüber |
| **Bei Misserfolg** | Freisteller scheitert → Original behalten (Verhalten von heute, `cardscan.py:271` — richtig so). `slab_recut` unplausibel → verwerfen und Studio-Freisteller behalten (`cardscan.py:314`, `:322` — richtig so). Glanz zu hoch **und** nur ein Foto → Capture-Guidance in der PWA („Slab leicht kippen"), Pipeline läuft trotzdem weiter, aber `C_bild = 0.6`, was in S3/S6 durchschlägt |
| **Zieldatei** | `/Users/smorty/ebay-bot/web/cardscan.py` — neue Funktionen `label_crop()` und `glanz_score()` neben `slab_recut` (`:287`); Aufruf in `analyze_collection_item` (`/Users/smorty/ebay-bot/web/app_api.py:922-937`, direkt nach dem bestehenden Slab-Rettungsfall) |

**Zwei Befunde nebenbei:**
- `crop_photos` (`cardscan.py:274-282`) stellt nur die ersten zwei Fotos frei. Das ist als Kostenentscheidung richtig, heißt aber: ein drittes Foto vom Label (das bei Reflexionen das beste wäre) wird nie aufbereitet. Für die Mehrfachaufnahme-Idee muss diese Grenze auf „die ersten zwei **plus** das Foto mit dem besten Label-Glanz-Score" erweitert werden.
- `requirements.txt` listet weder `opencv-python` noch `rembg` noch `numpy`, obwohl `cardscan.py:159-176` alle drei importiert. Im venv sind sie da (cv2 5.0.0, rembg 2.0.76, numpy 2.4.6), auf einem frischen Server wäre der Freisteller stumm tot — er fängt die `ImportError` nämlich ab (`cardscan.py:254`) und liefert einfach das Original zurück.

---

## 3. Stufe S2 — Klassifizierung

Heute klassifiziert `DETECT_PROMPT` (`cardscan.py:67-86`) bereits `slab|sleeve|raw|other`, und `detect_card` (`:101-104`) verlangt `confidence == "high"` sonst `None`. Das ist die richtige Bauart und wird beibehalten. Was fehlt, ist die **zweite Achse**: die Warenklasse.

`guess_category` (`/Users/smorty/ebay-bot/web/app_api.py:173-180`) ist heute reine Stichwort-Suche und liefert Anzeigekategorien („Pokémon", „Games"), keine Routing-Klasse. Sie darf bleiben — aber `item_class` ist ein zweites, hartes Feld.

| | |
|---|---|
| **Eingabe** | Fotos, `analysis` (Claude-Listing), `analysis.aspects`, `notes`, Label-Text aus S1 |
| **Verarbeitung** | (a) `holder` aus `detect_card` (vorhanden). (b) **NEU** `item_class` aus drei Signalen: Slab-Labeltyp (Comic-Label = *Titel + Heftnummer + Datum + Verlag*; Karten-Label = *Jahr + Set + Name + Kartennummer*), `analysis.aspects`, `category_query`. (c) **Positive-Evidenz-Regel**: `holder = slab` nur wenn ein opakes bedrucktes Band mit Text erkannt wurde — Transparenz über die ganze Fläche ⇒ `sleeve`, nie `slab` |
| **Ausgabe** | `holder`, `item_class`, `C_klasse` |
| **Konfidenz** | `C_klasse = 1.0` bei `detect_card.confidence == "high"` **und** eindeutiger Warenklasse aus ≥2 der drei Signale; `0.75` bei nur einem Signal; `0.0` bei Widerspruch |
| **Bei Misserfolg** | `holder = unknown` oder `item_class = other` → **kein Preis**, `price_state = unbekannt`, `price_reason = UNBEKANNT_KLASSE`. Das Item bleibt voll bearbeitbar, Titel/Beschreibung entstehen weiter — nur die Zahl fehlt. Kein Fallback auf „irgendeine Preisquelle" |
| **Zieldatei** | `DETECT_PROMPT` in `/Users/smorty/ebay-bot/web/cardscan.py:67-86` um `item_class` erweitern; Validierung in `detect_card` (`:101-104`) mitziehen. `item_class` zusätzlich aus der Listing-Analyse ableiten in `/Users/smorty/ebay-bot/bot/claude_client.py` (neues Pflichtfeld im Schema bei `:69-90`, Validierung in `parse_listing_json` `:127-130`) |

> Wichtig: `holder = sleeve` darf **keinen** Preisaufschlag und **keinen** graded-Pfad auslösen. Ein Toploader sagt nichts über den Zustand. Das ist heute korrekt (`claude_client.py:85`: „Eine LOSE Karte ist NIEMALS graded"), aber nur als Prompt-Satz — es fehlt die Code-Prüfung. Die gehört in `parse_listing_json`: wenn `holder != "slab"`, dann `graded_info = null`, egal was das Modell geschrieben hat.

---

## 4. Stufe S3 — Slab: Grader, Note, Zertifikatsnummer

Das ist die Stufe, die es heute praktisch nicht gibt. `graded_info` kommt roh aus dem Vision-Call (`claude_client.py:85`), wird nirgends validiert, und `_GRADER_KUERZEL` (`claude_client.py:139-142`) korrigiert nur den **Titel**, nicht `graded["grader"]` (`:145-161`) — deshalb steht im Produktionsbestand `"BECKETT 9.0"` als Grade-Bucket, obwohl `sold.py:111` intern `beckett → bgs` normalisiert. Drei Schreibweisen, drei Buckets, dasselbe Objekt.

### 4.1 Zwei-Kanal-Prinzip

**Verifiziert:** im venv (`/Users/smorty/ebay-bot/.venv`) sind `cv2.QRCodeDetector`, `cv2.QRCodeDetectorAruco` und `cv2.barcode.BarcodeDetector` alle vorhanden (OpenCV 5.0.0). **Der QR-/Barcode-Kanal kostet keine neue Abhängigkeit** — `cv2` wird von `cardscan.py:170` ohnehin schon importiert. `pyzbar` ist nicht installiert und wird auch nicht gebraucht.

- **Kanal A** — QR/Barcode auf dem entzerrten Slab bzw. Label-Crop. PSA (QR seit 2020), CGC und SGC drucken QR-Codes, die auf die Verifikationsseite zeigen; die Cert-Nummer steckt in der URL. PSA und BGS codieren sie zusätzlich im Barcode. Keine Halluzination möglich.
- **Kanal B** — LLM liest den Label-Crop aus S1.5.

Stimmen A und B überein → `C_grade = 1.0`, kein PSA-Lookup nötig. Weichen sie ab → PSA-Lookup entscheidet (nur bei PSA möglich). Nur B vorhanden → Format-Gate entscheidet.

### 4.2 Grader-Kanon (`claude_client.py:139-142` erweitern)

Heute: `beckett→BGS, bgs, psa, cgc, sgc, wata, vga, cga, ace`. Es fehlen `BVG`, `BCCG`, `BAS`, `CBCS`, `CSG`, `TAG`, `MANA`. Und `beckett → BGS` ist bei einem Vintage- (BVG) oder Collectors-Club-Slab (BCCG) schlicht falsch: BCCG-Noten sind notorisch 2–3 Stufen milder, ein BCCG 10 darf niemals wie ein BGS 10 bepreist werden. Der Kanon muss aus dem **Labeltyp** entscheiden, nicht aus dem Firmennamen.

### 4.3 Skalen-Validierung (das eigentliche Gate)

| Grader | Gültige Noten | Sondermerkmale |
|---|---|---|
| PSA | 1–10, dazu 1.5, halbe Stufen bei manchen | Cert 8–10 Ziffern (*Heuristik, keine PSA-Primärquelle*) |
| BGS | 1–10 in 0.5-Schritten, dazu 9.4/9.2 real auf Labeln | 4 Subgrades C/E/CO/S; Label-Farbe Silber/Gold/Schwarz ist Datenfeld |
| BVG | wie BGS, **ohne** Subgrades | |
| BCCG | eigene, mildere Skala | **nie** gegen BGS-Preise mappen |
| CGC Cards | 1.5–9.5 in Halbstufen, dann Gem Mint 10, Pristine 10 | **kein 9.8** — ein „CGC 9.8" ist entweder ein Comic oder ein Lesefehler |
| CGC Comics | 0.5–10.0 | 9.8 ist hier die Standard-Topnote; Farbband = Status (blau Universal, gelb Signature, grün Qualified, lila Restored) |
| SGC | 1–10 | moderne Certs 7 Ziffern, alte 11 Zeichen **mit Bindestrich** |
| WATA | Box-Note 10-Punkte-Skala **plus** Seal Rating C…A++ | zwei Achsen — nur die Zahl zu lesen verliert die halbe Information |

**Die Gate-Regel, die den 9.8-Fall abfängt:** Wenn `grader == CGC` und `item_class == tcg_card` und `grade == 9.8` → *das ist auf dieser Skala unmöglich*. Entweder ist die Klasse falsch (es ist ein Comic) oder die Note. Beides heißt: `graded = null`, `GRADE_UNGUELTIG`, weiter auf dem raw-Pfad. **Niemals** eine unmögliche Note bepreisen.

| | |
|---|---|
| **Eingabe** | Label-Crop (S1.5), Gesamtbild, `item_class` (S2), `analysis.graded_info` |
| **Verarbeitung** | 3.1 QR decodieren · 3.2 Barcode decodieren · 3.3 LLM auf Label-Crop (grader, grade, cert, **alle 4 Subgrades gemeinsam oder gar keinen**, Labelfarbe) · 3.4 Grader kanonisieren aus Labeltyp · 3.5 Note gegen die Skala **des jeweiligen Graders** prüfen · 3.6 Cert-Format-Gate je Grader, bei PSA optional Lookup |
| **Ausgabe** | `graded = {grader, grade, subgrades, cert_number, cert_verified: bool, scale_ok: bool, quelle: "qr"\|"barcode"\|"llm"\|"qr+llm"}` |
| **Konfidenz** | `1.0` Cert per Lookup verifiziert **oder** Kanal A ∧ B einig · `0.85` nur Kanal B, Cert-Format plausibel, Note auf Skala · `0.6` nur Kanal B, Cert unplausibel/fehlend · `0.0` Note nicht auf der Skala |
| **Bei Misserfolg** | Note ungültig → `graded = null` + Flag, **raw-Pfad**, kein Abbruch (das Stück hat ja einen Wert, nur eben keinen belegbaren Grade-Aufschlag). Grader unlesbar, Note lesbar → `graded = null`, weil `PSA 9 ≠ CGC 9` und ein Preis ohne Grader-Zuordnung wertlos ist. Kein Fallback auf „irgendein Grader" |
| **Zieldatei** | **NEU `/Users/smorty/ebay-bot/web/slab.py`**: `decode_codes()` (cv2), `GRADER_KANON`, `CERT_FORMAT`, `SKALEN`, `validate_grade()`, `grade_rank()`. Der Kanon ersetzt `_GRADER_KUERZEL` in `/Users/smorty/ebay-bot/bot/claude_client.py:139-142`, das dann aus `web/slab.py` importiert — **eine** Wahrheit für Titel-Korrektur (`:145-161`), Bucket-Bildung (`catalog.py:81-84`) und Sold-Filter (`sold.py:104-111`, `:111`). Aufruf der Stufe in `/Users/smorty/ebay-bot/web/app_api.py:908-909`, wo `graded` heute ungeprüft übernommen wird |

### 4.4 PSA Public API — Budget-Regel

`GET https://api.psacard.com/publicapi/cert/GetByCertNumber/{cert}`, Bearer-Token, **Free Tier 100 Calls/Tag**. Das ist der einzige Grader mit dokumentierter öffentlicher API. Bei zehntausenden Nutzern reicht das nicht — deshalb:

- **Cert-Nummern permanent cachen.** Eine Cert-Nummer ist unveränderlich; das ist der perfekteste Cache-Schlüssel im ganzen System. Gehört als eigene Tabelle `certs(cert_number PK, grader, subject, grade, year, brand, payload, ts)` neben `cards`/`card_prices` in `/Users/smorty/ebay-bot/web/catalog.py:26-41`.
- **Lookup nur auslösen, wenn** Kanal A und B widersprechen **oder** der erwartete Wert eine Schwelle (z. B. 200 €) überschreitet.
- Für Nicht-PSA-Slabs existiert kein dokumentierter Lookup. Kommerziell deckt Ximilar `/v2/slab_id` BECKETT/CGC/SGC/TAG/ACE/MANA inkl. Cert-Nummer und `Graded: yes/no` ab (Business ab 59 €/Monat). Das Scrapen der CGC-/Beckett-Lookup-Seiten ist **kein** Produktionspfad — sie antworten mit 403 auf automatisierte Requests.

---

## 5. Stufe S4 — Entity Matching mit ausgewiesener Konfidenz

Hier entsteht der Fehler von heute. `lookup_pc` (`/Users/smorty/ebay-bot/web/pricecharting.py:87-92`) ruft `GET /api/product?q=<Freitext>` und prüft **nur `prod["id"]` auf Existenz**. Ein Buch und eine Sammelkarte sind für diese Funktion ununterscheidbar.

Zum Vergleich: `price_pokemon_tcgdex` (`/Users/smorty/ebay-bot/web/prices.py:154-161`) macht es vorbildlich — *„Ist die Kartennummer erkannt, MUSS sie passen, sonst gar kein Treffer"* — und `:176-178` verwirft Kandidaten mit widersprüchlicher Set-Größe. Genau diese Härte fehlt bei PriceCharting.

### 5.1 Architektur-Vorbedingung

`/api/products?q=` statt `/api/product?q=`: bis zu 20 Kandidaten mit `id`, `product-name`, `console-name`. Das ist der einzige Weg, Mehrdeutigkeit überhaupt zu **sehen**. Dann Gewinner ermitteln und für ihn einmal `/api/product?id=<id>` für die vollen Preisfelder. Kosten: 2 Calls statt 1 — bei 1 Call/s durch den Katalog-Cache tragbar.

### 5.2 Domänen-Gate (hart, keine Punkte)

`console-name` trägt bei PriceCharting die Domäne:

```
console_name.startswith("Comic Books")   → comic_manga
console_name in KONSOLEN_LISTE           → video_game
sonst                                    → tcg_card / sport_card
```

**Regel: `domain(kandidat) != item_class` → Kandidat verworfen, Konfidenz 0, keine weitere Rechnung.** Das allein hätte die 603 € verhindert — das richtige Produkt („One Piece Vol. 103 [Paperback]", console `Comic Books One Piece`) existiert bei PriceCharting sogar, die Suche mischt nur die Domänen.

Sofort-Match ohne Scoring: `upc` identisch oder `epid` identisch → `P = 1.0`.

### 5.3 Bit-Scoring

```
S = w0 + Σ w_i        P = 1 / (1 + 2^(-S))
```

| Merkmal | Bedingung | Bits |
|---|---|---|
| Prior `w0` | immer | −3.0 |
| Hart-ID (Kartennr. `OP01-016`, `Vol. 103`, Set-Code) | im Kandidatennamen | +6.0 |
| | Item hat eine, fehlt im Namen | −6.0 |
| Namensähnlichkeit (Token-Set) | ≥95 / 85–94 / 70–84 / 50–69 / <50 | +3.0 / +1.5 / 0 / −2.0 / −4.0 |
| Set/Serie ↔ `console-name` | ≥90 / 70–89 / 40–69 / <40 | +3.0 / +1.0 / 0 / −2.5 |
| Jahr-Differenz | ≤1 / 2–3 / >3 | +1.5 / 0 / −2.0 |
| Bild (pHash-Hamming gegen TCGdex/Scryfall-Bild) | ≤8 / 9–12 / 13–18 / >18 | +6.0 / +2.0 / 0 / −3.0 |
| Preis-Plausibilität `r = max(a/b, b/a)` | ≤1.5 / 1.5–3 / 3–6 / >6 | +2.0 / 0 / −3.0 / −6.0 |
| Sprache stimmig / widersprüchlich | | +1.0 / −2.0 |

Fehlende Daten ⇒ Gewicht 0. Der Bildvergleich läuft gegen die Katalogbilder aus `prices.py:220`/`:251`/`:285`, **nicht** gegen PriceCharting — die PC-Doku nennt kein Bildfeld.

**Nachrechnung Svens Fall**, bewusst *ohne* Domänen-Gate, um zu zeigen dass auch das Scoring allein trägt: Item „One Piece Manga Band 103" vs. Kandidat „Nami [Manga] OP01-016" / console „One Piece Romance Dawn" → `−3.0` (Prior) `−6.0` (Hart-ID 103 fehlt) `−4.0` (Name) `−2.5` (Set) `0` (Jahr) `0` (kein Bild) `−6.0` (603 € vs. Marktmedian ~8 €) = **S = −21.5 → P ≈ 3·10⁻⁷**. Sicher verworfen.

### 5.4 Der `card_key` — Wegwerfschlüssel abschaffen

`card_key_of` (`/Users/smorty/ebay-bot/web/catalog.py:48-53`) erzeugt `"solo:" + uuid4()` wenn weder `ref_id` noch `name` da sind. Und `identify_card` liefert für Nicht-Einzelkarten immer ein Dict **mit** Schlüssel `"name"` und Wert `None` (`prices.py:82-88`) — worauf `card_ref.setdefault("name", item.get("name"))` (`app_api.py:790`, identisch `:1000`) **nichts** tut, weil `setdefault` nur bei *fehlendem* Schlüssel greift, nicht bei `None`. Ergebnis im Produktionsbestand: 223 von 230 `card_prices`-Zeilen unter `solo:`, `cards` mit 282 Zeilen bei 15 Items.

Zwei Änderungen:

```python
# app_api.py:789-791 und :999-1001
card_ref = dict(item.get("card") or card_info or {})
card_ref["name"] = card_ref.get("name") or item.get("name")   # statt setdefault
card_ref["item_class"] = item.get("item_class")
card_ref["language"]   = (item.get("card") or {}).get("language") or item.get("language")
```

```python
# catalog.py:48-60 — deterministisch, immer
def card_key_of(card):
    if card.get("ref_id"):
        return f"{card.get('game') or card.get('item_class') or 'x'}:{card['ref_id']}"
    kern = _norm("|".join(str(card.get(k) or "") for k in
                 ("item_class", "game", "name", "number", "set_name", "language", "edition")))
    if not kern.strip():
        return None      # KEIN Schlüssel — der Aufrufer muss das als
                         # UNBEKANNT_IDENTITAET behandeln, nicht als „solo:"
    return "h:" + hashlib.sha1(kern.encode()).hexdigest()[:16]
```

`None` statt `solo:uuid` ist der entscheidende Unterschied: heute *sieht* das System nicht, dass es keine Identität hat — es erfindet eine. `language` und `edition` gehören in den Hash (heute nicht: `catalog.py:59`), sonst teilen sich Deutsch/Englisch und 1st Edition/Unlimited eine Preiszeile.

| | |
|---|---|
| **Eingabe** | `analysis`, `item_class`, `graded`, `notes` |
| **Verarbeitung** | 4.1 `identify_card` **mit `item_class` als Vorgabe im System-Prompt** · 4.2 Kandidaten je Klasse (Karten-DB für `tcg_card`, PC `/api/products` für alle) · 4.3 Domänen-Gate · 4.4 Bit-Scoring · 4.5 Ambiguitätsregel · 4.6 `card_key` deterministisch |
| **Ausgabe** | `card_info`, `card`, `card_key`, `identity_conf = P₁`, `pc_id` des Gewinners |
| **Konfidenz** | `identity_conf = P₁` |
| **Bei Misserfolg** | kein Kandidat übersteht das Domänen-Gate → `UNBEKANNT_DOMAENE`. `P₁ < 0.65` → `UNBEKANNT_IDENTITAET`. `P₁ − P₂ < 0.15` → `UNBEKANNT_MEHRDEUTIG` (zwei fast gleich gute Kandidaten heißt: die Suche kann es nicht entscheiden, dann darf SERO es auch nicht). `card_key is None` → `UNBEKANNT_IDENTITAET`. **In allen vier Fällen: keine Preisabfrage überhaupt.** Das spart nebenbei genau die 130point-Abfragen, die heute die 8-s-Drossel verstopfen |
| **Zieldatei** | **NEU `/Users/smorty/ebay-bot/web/match.py`**: `domain_of_pc()`, `score_kandidat()`, `waehle_kandidat()` · `lookup_pc_candidates()` in `/Users/smorty/ebay-bot/web/pricecharting.py:76-121` · Schlüssel in `/Users/smorty/ebay-bot/web/catalog.py:48-60` · `setdefault`-Fix in `/Users/smorty/ebay-bot/web/app_api.py:790` und `:1000` · `item_class`-Vorgabe im Prompt von `/Users/smorty/ebay-bot/web/prices.py:78-88` |

### 5.5 Cache-Schlüssel zweistufig

`pricecharting.py:81` cached unter `sha1(query)` — ein einmal falsch gematchtes Produkt klebt 12 h an der Anfrage und ist nicht gezielt invalidierbar. Besser: `query → pc_id` mit kurzer TTL (das ist die unsichere Zuordnung) und `pc_id → Produktdaten` mit langer TTL (das ist der stabile Fakt).

---

## 6. Stufe S5 — Bewertung

Erst hier wird Geld angefasst, und nur mit einer geprüften Identität in der Hand.

### 6.1 Belegklassen statt Quellennamen

Die heutige Kaskade (`catalog.py:184-207`) ordnet nach Quellennamen. Das ist der Grund, warum ein *ungefilterter* Browse-Median (`ebay_eu`) stärker ist als ein *domänengeprüfter* PriceCharting-Treffer. Richtig ist eine Ordnung nach **Belegklasse**:

| Klasse | Bedeutung | Quelle | Verfügbarkeit heute |
|---|---|---|---|
| **E1** | eigene, tatsächlich verkaufte Stücke | eBay Sell-APIs auf SERO-erzeugte Listings | **noch nicht** (Token ohne Fulfillment-Scope, 403). Der Burggraben — first-party, vertraglich sauber, wird mit jedem Nutzer besser |
| **E2** | fremde, tatsächlich verkaufte Stücke | `web/sold.py` (130point) · `web/ebay_insights.py` falls je freigeschaltet | ja, aber ohne Vertrag; harte Obergrenze ~10.800 Anfragen/Tag durch `MIN_GAP = 8.0` (`sold.py:37`, serverweiter Lock) |
| **E3** | lizenzierter Katalogpreis (Aggregat über viele echte Verkäufe) | TCGdex/Cardmarket, Scryfall, YGOPRODeck, tcgcsv (`web/prices.py`) | ja, sauber, deterministisch — **nur für rohe Einzelkarten** |
| **E4** | Angebotspreise (was gefordert wird, nicht was gezahlt wurde) | eBay Browse (`bot/ebay/browse.py`) | ja, offiziell lizenziert, skaliert |
| **E5** | Sekundärkatalog | PriceCharting (`web/pricecharting.py`) | ja — **aber ToS-Konflikt, siehe §6.5** |
| **E6** | KI-Schätzung | `analysis.estimated_price_range_eur` | **nie ein Wert.** Nur Plausibilitätsband |

### 6.2 Die Hierarchie, aufgeteilt nach Grade-Bucket

Der entscheidende Unterschied zu heute: **die Reihenfolge hängt vom Grade-Bucket ab**, weil E3 für Slabs schlicht nicht existiert (Cardmarket kennt keine PSA-10-Preise).

```
grade == "raw":
    E1 (n≥3)  >  E3 (Katalogpreis)  >  E2 (n≥2)  >  E4  >  E5  >  unbekannt
grade != "raw":
    E1 (n≥3)  >  E2 (n≥2)           >  E4        >  E5  >  unbekannt
```

**Warum E3 bei raw über E2 steht** — das ist eine echte Umkehrung gegenüber heute (`catalog.py:184-190`, wo `sold` immer gewinnt): Ein Cardmarket-Trend ist ein Aggregat über hunderte Transaktionen, vertraglich sauber bezogen, täglich aktualisiert. Drei 130point-Zeilen sind drei Datenpunkte aus einer gescrapten HTML-Tabelle. Für eine rohe Einzelkarte ist der Katalogpreis schlicht der bessere Schätzer. Für Slabs existiert er nicht — dort bleibt E2 vorn.

**E4 muss gefiltert werden.** `research_price` (`/Users/smorty/ebay-bot/bot/ebay/browse.py:19-76`) hat IQR-Trimming (`:52-61`), aber **keinerlei Titelprüfung** — es kappt Preise, nicht falsche Artikel. Über `eu_probe` (`catalog.py:139-147`, −12 %) wird dieser ungefilterte Median heute zur zweitstärksten Kaskadenstufe. In Produktion sichtbar als `pokemon:sv03.5-199 / CGC 10 → 803,99 € aus ebay_eu`. Fix: `research_price` bekommt einen optionalen Parameter `titel_filter`, und der Katalog reicht `sold.fits` durch:

```python
# bot/ebay/browse.py:19  →  research_price(client, query, limit=20, titel_filter=None)
#   Filter läuft VOR dem IQR-Trimm auf item.get("title")
# web/catalog.py:139-147  →  eu_probe(query, titel_filter=fits)
```
Ein injizierter Callable statt eines Imports — `bot/` darf nicht auf `web/` importieren, sonst entsteht ein Zyklus (heute importiert `web/app_api.py` → `bot/ebay/browse.py`).

Zusätzlich: der EU-Zweig umgeht heute `check_price_plausibility` komplett, weil `catalog.py:142` `research_price` **direkt** aufruft, während `app_api.py:824`/`:967` es durch den Wächter schickt. Der Wächter (`/Users/smorty/ebay-bot/bot/main.py:238-266`) gehört in die `eu_probe`-Lambda.

### 6.3 Grade-Auflösung — der 9.0-vs-9.4-Fall als Invariante

PriceCharting kennt bei Karten nur die Leiter 7/7.5 → 8/8.5 → 9 → 9.5 → 10. `_zwischenstufe` (`pricecharting.py:49-73`) interpoliert linear und braucht dafür **beide** Stützstellen; fehlt eine, bleibt es beim Feldwert — dann landen 9.0 und 9.4 wieder identisch.

**Regel:** Wenn zwei Grade-Buckets desselben `card_key` denselben `pc_field` benutzen und die Interpolation nicht griff, dürfen sie **nicht als getrennte Punktpreise** ausgegeben werden. Beide bekommen `price_state = spanne` mit `grade_aufloesung: "grob"`. Das ist kein Einzelfix, sondern eine Eigenschaft der Quelle, die durchgereicht werden muss: `detail["pc_field"]` steht bereits in der Antwort (`pricecharting.py:118`), es wird nur nie ausgewertet.

**Und die dritte Leiter fehlt.** `_grade_fields` (`pricecharting.py:21-46`) kennt die Karten- und die Videospiel-Leiter, sonst nichts. Bei Comics/Manga bedeutet dasselbe JSON-Feld etwas völlig anderes:

| JSON-Key | Videospiele | Sammelkarten | Comics/Manga |
|---|---|---|---|
| `loose-price` | Modul/Disc | Ungraded | Ungraded |
| `cib-price` | Complete in Box | Graded 7 / 7.5 | Graded 4.0 / 4.5 |
| `new-price` | versiegelt | Graded 8 / 8.5 | Graded 6.0 / 6.5 |
| `graded-price` | WATA/VGA-graded | Graded 9 | **Graded 8.0 / 8.5** |
| `box-only-price` | leerer Karton | Graded 9.5 | Graded 9.2 |
| `condition-17-price` | – | CGC 10 | Graded 9.4 |
| `manual-only-price` | Anleitung | PSA 10 | Graded 9.8 |
| `bgs-10-price` | – | BGS 10 | Graded 10.0 |
| `condition-18-price` | – | **SGC 10** | – |

Das ist der *zweite*, bisher unentdeckte Teil des Manga-Bugs: selbst mit repariertem Matching würde eine Beckett 9.0 auf einem Buch über die Kartenleiter auf `graded-price` gemappt — bei Comics bedeutet das Feld aber 8.0/8.5. Der Docstring in `pricecharting.py:6` ist zusätzlich falsch: `condition-18-price` ist SGC 10, nicht CGC 9.5; ein CGC-9.5-Feld existiert gar nicht.

### 6.4 Katalog-Read und TTL

TTL nach Belegklasse statt nach Quellenname (heute `catalog.py:115`):

| Belegklasse | TTL |
|---|---|
| E1 / E2 | 24 h |
| E3 | 12 h |
| E4 | 6 h |
| E5 | 6 h |
| `unbekannt` | 1 h (Selbstheilung — das Prinzip von heute ist richtig) |

| | |
|---|---|
| **Eingabe** | `card_key`, `grade_bucket`, geprüfte Query, `graded`, `usd_rate`, `item_class` |
| **Verarbeitung** | Katalog-Read → bei TTL-Ablauf parallel E1–E5 abfragen (heute `asyncio.gather` bei `catalog.py:127`, bleibt), Gewinner nach §6.2 |
| **Ausgabe** | `value_eur`, `price_evidence ∈ {E1..E5}`, `source_label`, `detail`, `C_quelle` |
| **Konfidenz** | `C_quelle`: E1 `1.0` · E2 mit n≥3 frischen `0.95`, n=2 `0.85`, `stale` `0.75` · E3 `0.95` · E4 mit ≥5 gefilterten Treffern `0.8`, 3–4 `0.7` · E5 domänengeprüft `0.75`, sonst **verworfen** (nicht `weak`) |
| **Bei Misserfolg** | Alle Quellen leer → `price_state = unbekannt`, `price_reason = UNBEKANNT_KEINE_BELEGE`. 130point im 429-Cooldown (`sold.py:43-44`, `:66`) → `UNBEKANNT_DROSSEL` mit automatischem Wiederholungstermin, **nicht** stiller Rückfall auf E5. Die alte Katalogzeile bleibt als Anzeige stehen (`catalog.py:204-205`, richtig), aber mit ihrem eigenen Alter markiert |
| **Zieldatei** | `/Users/smorty/ebay-bot/web/catalog.py:102-250` (`refresh_price`) — Kaskade `:184-207` ersetzen; `/Users/smorty/ebay-bot/web/pricecharting.py:21-46` dritte Leiter; `/Users/smorty/ebay-bot/bot/ebay/browse.py:19` Filter-Parameter |

### 6.5 🔴 Der Elefant: PriceCharting darf E5 gar nicht sein

Die PriceCharting Terms of Service beschränken die Nutzung auf „Internal Business Purposes" und verbieten Preisdaten ausdrücklich „in any software, application, or system that is accessible to third parties … without express written permission". SERO ist per Definition genau der ausgeschlossene Fall. Dazu 1 Call/Sekunde für das *gesamte* Konto bei 49–59 $/Jahr — ein Einzelnutzer-Produkt, kein Backend für zehntausende Nutzer.

**Für den Entwurf heißt das:** E5 ist im Baum als Stufe vorgesehen, muss aber vor dem Launch entweder (a) durch eine schriftliche Redistributions-Lizenz abgesichert oder (b) ersatzlos gestrichen werden. Der Baum funktioniert ohne E5 — er endet dann eben öfter bei „unbekannt", was Svens erklärter Wunsch ist. Die Lizenzanfrage ist der billigste Fix im ganzen Papier; sie sollte vor der Implementierung raus, damit die Antwort die Architektur entscheidet und nicht umgekehrt.

Analog: 130point (E2) ist ein undokumentierter Backend-Endpunkt (`POST https://back.130point.com/sales/`, `sold.py:50-53`), der per Regex geparst wird (`sold.py:31`) und für den `sold.py:28-29` einen Browser-User-Agent fälscht — die Seite antwortet auf nicht-Browser-Requests mit 403. Das eBay User Agreement (Fassung wirksam 20.02.2026) untersagt automatisiertes Abgreifen ohne Erlaubnis ausdrücklich, auch mittelbar. Sven ist gewerblicher eBay-Händler; das Risiko trifft nicht ein Feature, sondern sein Konto. **E2 ist eine Übergangsquelle, kein Fundament.** Deshalb ist E1 im Baum ganz oben, obwohl es heute noch 403 liefert — dort muss die Reise hin.

---

## 7. Stufe S6 — Plausibilitäts-Tor vor der Anzeige

Fünf Tests, alle müssen bestanden sein, bevor eine Zahl erscheint.

### 7.1 Referenz-Anker — der strukturelle Fix

Heute sind die Wächter beim Manga **strukturell stumm**: `base` wird in `refresh_item_price` nur gesetzt wenn `updated` (`app_api.py:795-797`), und `updated` wird nur bei einem Karten-DB-Treffer wahr (`:768-774`). Nicht-Einzelkarten haben nie einen Karten-DB-Treffer → `base = None` → `if ref and ref > 0` (`catalog.py:233`) ist falsch → der Wächter läuft ins Leere. Genau die Warengruppe, für die PriceCharting am unzuverlässigsten ist, ist ungeschützt. Zusätzlich wird `base` bei gegradeten Stücken nie als Wert genutzt (`catalog.py:196`: `... and grade == "raw"`), weshalb jeder Slab direkt auf `pricecharting_weak` durchfällt — den unsichersten Zweig.

**Der Fix nutzt vorhandene Struktur:** Die Referenz für einen gegradeten Wert ist die **raw-Zeile desselben `card_key`** aus dem Katalog.

```python
# in catalog.refresh_price, vor der Kaskade
anker = base
if anker is None and grade != "raw":
    r = get_price(store, card_key, "raw")          # existiert bereits, catalog.py:87
    if r and r.get("value_eur"):
        anker = {"value": r["value_eur"], "source": r["source"], "label": r["source_label"]}
if anker is None:
    est = listing.get("estimated_price_range_eur")  # E6, NUR als Band, nie als Wert
    if est: anker = {"band": (est["low"], est["high"])}
```

Damit hat jeder Wächter immer eine Referenz — ohne neue Tabelle, ohne neuen externen Call.

### 7.2 Die fünf Tests

| # | Test | Bedingung | Reaktion |
|---|---|---|---|
| T1 | **Anker-Test** | `value` innerhalb `[anker/4, anker*4]` (bzw. `[low*0.4, high*2.5]` beim KI-Band) | Verletzt und Belegklasse ≤ E4 → `unbekannt`, `UNBEKANNT_WIDERSPRUCH`. Verletzt bei E1/E2 mit n≥3 → Wert bleibt, aber `price_state = spanne` + Hinweis (echte Verkäufe dürfen den Anker schlagen) |
| T2 | **Quellen-Widerspruch** | zwei unabhängige Quellen ≥ E4 vorhanden, Verhältnis `max/min` | ≤2 → +0.05 Konfidenz · 2–4 → `spanne` mit `[min, max]` · >4 → `unbekannt`, `UNBEKANNT_WIDERSPRUCH` |
| T3 | **Streuungs-Test** | bei E2: `max(sales)/min(sales)` der Belege | >6 bei n<4 → `spanne` statt Punktpreis. (Der heutige Ausreißer-Wächter `catalog.py:212-224` greift nur bei `n_avg < 2`, aber `fetch_sold` liefert erst ab 2 Verkäufen (`sold.py:281`, `:288`) — **er ist faktisch tot** und muss auf `n_avg < 4` plus Streuungsmaß umgestellt werden) |
| T4 | **Grade-Monotonie** | §8 | Verletzung → die Zeile mit der schwächeren Belegklasse verliert |
| T5 | **Konfidenz-Schwelle** | `C_gesamt = C_bild × C_klasse × identity_conf × C_grade × C_quelle` | ≥0.90 `belegt` · 0.65–0.90 `spanne` · <0.65 `unbekannt` |

**Woher 0.90 kommt:** Chows Regel. Ist eine falsche Preisangabe `k`-mal so teuer wie eine fehlende, ist die optimale Schwelle `τ = k/(k+1)`. Für ein Produkt, das automatisch eBay-Listings erzeugt, ist ein falscher Preis leicht das Neunfache eines fehlenden wert (Fehlverkauf, Reklamation, Vertrauensverlust) → `k = 9` → `τ = 0.90`. Sven wählt `k`, die Schwelle fällt heraus. Das ist der Punkt, an dem eine Geschäftsentscheidung eine Zahl wird — kein Bauchgefühl.

### 7.3 Korrektur muss den Katalog treffen

Heute korrigiert Wächter C (`app_api.py:833-845`) nur das Item; die Katalogzeile bleibt auf dem falschen Wert. Der nächste Leser derselben `card_key`+`grade`-Zeile bekommt wieder 603 € — im Bestand stehen 87 von 230 Zeilen (38 %) auf `pricecharting_weak`. **Jede Korrektur schreibt zurück in `card_prices`**, mit `price_state` und `price_reason` als eigene Spalten. Und Wächter C fehlt in `analyze_collection_item` komplett — beim allerersten Scan, dem einzigen, den der Nutzer wirklich sieht, greift er nicht.

| | |
|---|---|
| **Eingabe** | Katalogzeile, Anker, Zweitquelle, KI-Band, Nachbar-Grade-Zeilen |
| **Verarbeitung** | T1–T5 in dieser Reihenfolge |
| **Ausgabe** | `price_state`, `price_reason`, `est_value` **oder** `price_low`/`price_high`, `price_evidence`, `price_label` |
| **Konfidenz** | `C_gesamt`, wird persistiert (Debuggbarkeit — heute lässt sich nicht rekonstruieren, *warum* eine Zahl entstand) |
| **Bei Misserfolg** | Kein Fallback. `price_state = unbekannt` + Grund. Die App zeigt „Wert unbekannt — <Grund>" und bietet die manuelle Eingabe an |
| **Zieldatei** | T1–T4 in `/Users/smorty/ebay-bot/web/catalog.py` (ersetzt `:212-224` und `:231-240`), T5 + Anzeige in `/Users/smorty/ebay-bot/web/app_api.py:805-855` **und** spiegelbildlich `:1014-1022` — die beiden Pfade müssen identisch sein, was sie heute nicht sind |

---

## 8. Grade-Monotonie als Katalog-Invariante

Nicht als Einzelfix im Preisabruf, sondern als Eigenschaft, die die Tabelle `card_prices` jederzeit erfüllt.

### 8.1 Die Invariante

Für jeden `card_key` und jeden **kanonischen Grader** `g` gilt über die Kette der Noten:

```
grade_rank(g, n₁) < grade_rank(g, n₂)  ⟹  value(card_key, g, n₁) ≤ value(card_key, g, n₂) · (1 + ε)
```

mit `ε = 0.05` als Messrauschen-Toleranz.

**Streng nur innerhalb eines Graders.** Über Grader hinweg ist Monotonie sachlich falsch: BGS 9.5 entspricht ungefähr PSA 10, PSA 10 erzielt trotzdem regelmäßig mehr (Markenprämie); BCCG-Noten liegen 2–3 Stufen milder. Zwischen Gradern gilt deshalb nur ein **Bandtest** über eine Äquivalenztabelle mit weiter Toleranz — Verletzung setzt ein Flag, ändert aber keinen Wert.

**Raw ist nicht Teil der Kette.** Ein PSA 3 kann weniger wert sein als eine rohe Near-Mint-Karte. Stattdessen ein separater Anker-Test: `grade ≥ 8` ⟹ `value ≥ value(raw) · 0.8`, sonst Flag `MONOTONIE_RAW` (kein automatischer Eingriff, aber die Zeile wird nicht als `belegt` ausgeliefert).

### 8.2 Durchsetzung an drei Stellen

**(a) Beim Schreiben** — in `refresh_price`, vor dem `INSERT` (`catalog.py:242-248`): die Geschwisterzeilen desselben `card_key` und Graders lesen, Verletzung prüfen. Wer verliert, entscheidet die **Belegklasse, nicht der Zeitpunkt**:

```
E1/E2-Zeile schlägt E3/E4/E5-Zeile:   die schwächere Zeile → price_state = spanne
gleiche Belegklasse:                   BEIDE → spanne, Grund UNBEKANNT_MONOTONIE
```
Eine belegte Zeile wird niemals still „zurechtgebogen" — das wäre Datenfälschung. Sie wird entweder bestätigt oder zur Spanne degradiert.

**(b) Als Sweeper** — im 12-h-Autolauf `periodic_refresh` (`/Users/smorty/ebay-bot/web/app_api.py:2384`): pro `card_key`×Grader die Werte nach `grade_rank` sortieren und eine **isotone Regression** (Pool-Adjacent-Violators) rechnen. Zeilen, die von ihrer isotonen Hülle um mehr als Faktor 1.5 abweichen, bekommen `price_state = spanne` und werden beim nächsten Lauf mit `force=True` neu geholt. Das ist der Mechanismus, der Altlasten selbsttätig ausräumt, statt sie zu konservieren.

**(c) Bei der Bucket-Bildung** — `grade_bucket` (`catalog.py:81-84`) nimmt `graded["grader"]` **roh**. Im Bestand existieren deshalb `"BECKETT 9.0"`, `"BECKETT 9"`, `"BGS 9"` als drei Buckets für dasselbe Objekt. Monotonie über fragmentierte Buckets ist sinnlos. Also:

```python
def grade_bucket(graded):
    if not graded or not graded.get("grade"): return "raw"
    g = slab.GRADER_KANON.get(str(graded.get("grader","")).strip().lower())
    if not g: return "raw"                       # unbekannter Grader ⇒ kein Grade-Bucket
    n = slab.normalize_grade(graded["grade"])    # "9" → 9.0, "9,5" → 9.5
    if n is None or not slab.scale_ok(g, n): return "raw"
    return f"{g} {n:g}"                          # "BGS 9", "BGS 9.4", "PSA 10"
```

Die Bucket-Bildung ist damit derselbe Code wie die Skalen-Validierung aus S3 — **ein** Kanon, drei Nutzer (`claude_client.py:139-142`, `catalog.py:81-84`, `sold.py:104-111`).

### 8.3 `grade_rank` — die gemeinsame Ordnung

```python
# web/slab.py
def grade_rank(grader: str, note: float) -> float:
    """Ordinale Position INNERHALB der Skala eines Graders. Vergleiche über
    Grader hinweg sind bewusst NICHT definiert."""
```
Für Grader mit zwei Achsen (WATA: Box-Note **und** Seal Rating C…A++) ist `grade_rank` ein Tupel. Nur die Zahl zu vergleichen verliert den wertbestimmenden Teil und erzeugt scheinbare Monotonie-Verletzungen, die keine sind.

---

## 9. Der Zustand „Wert unbekannt, Grund X"

`price_reason` als geschlossenes Enum. Jeder Grund hat einen Nutzertext und eine definierte Wiederaufnahme.

| Code | Nutzertext (DE) | Nächster Versuch |
|---|---|---|
| `UNBEKANNT_KLASSE` | „Wir konnten nicht sicher erkennen, was das ist." | bei neuem Foto |
| `UNBEKANNT_IDENTITAET` | „Karte nicht eindeutig zugeordnet — bitte Set/Nummer prüfen." | manuelle Korrektur (`/cardsearch`, `app_api.py:3505`) |
| `UNBEKANNT_MEHRDEUTIG` | „Mehrere passende Produkte gefunden, keines eindeutig." | manuelle Auswahl aus Top-3 |
| `UNBEKANNT_DOMAENE` | „Kein Preis für diese Warenart verfügbar." | nie automatisch |
| `UNBEKANNT_GRADE_UNGUELTIG` | „Die abgelesene Note passt nicht zur Skala dieses Anbieters." | neues Label-Foto |
| `UNBEKANNT_KEINE_BELEGE` | „Keine belastbaren Vergleichsdaten." | 12-h-Autolauf |
| `UNBEKANNT_WIDERSPRUCH` | „Die Quellen widersprechen sich zu stark." | 12-h-Autolauf |
| `UNBEKANNT_MONOTONIE` | „Preis passt nicht zur Bewertungsstufe." | Sweeper |
| `UNBEKANNT_DROSSEL` | „Preisabfrage ausgelastet — wir versuchen es gleich erneut." | nach Cooldown (`sold.py:66`: 600 s) |

**Drei Regeln für die Anzeige:**
1. Ein „unbekannt" ist **kein Fehler** und darf nicht wie einer aussehen. `status` bleibt `ready`, nur `est_value` ist `None`. Das Listing entsteht trotzdem — der Nutzer trägt den Preis ein.
2. Bei `price_state = spanne` wird nie eine Punktzahl angezeigt, auch nicht als „ca.". Eine Spanne ist eine Spanne.
3. `price_source = "estimate"` verschwindet als Wertquelle. Die KI-Schätzung lebt weiter in `est_low`/`est_high` (`app_api.py:971-976`) — als Plausibilitätsband für T1, nicht als Preis. Konkret zu streichen: `app_api.py:852-855`, `:982-989` und der `estimate`-Zweig in `:842-844`.

---

## 10. Umsetzungsreihenfolge

**P0 — verhindert falsche Preise, wenig Aufwand, keine neue Abhängigkeit**
1. `setdefault`-Fix `app_api.py:790` und `:1000` + deterministischer `card_key` (`catalog.py:48-60`). Ohne das ist jede weitere Verbesserung wirkungslos, weil der Katalog seinen eigenen Cache nie trifft.
2. Domänen-Gate: `item_class` ableiten, `console-name`-Präfix prüfen, Klassen-Mismatch → Kandidat verworfen. **Das ist der direkte Fix des 603-€-Falls.**
3. `pricecharting_weak` abschaffen: ein unbelegter Treffer wird kein Preis, sondern `unbekannt`. (`catalog.py:199-203`)
4. Grader-Kanon in `web/slab.py` zentralisieren, Skalen-Validierung, `grade_bucket` normalisieren.
5. KI-Schätzung nie als `est_value`.

**P1 — größter Genauigkeitsgewinn pro Aufwand**
6. `/api/products` + Bit-Scoring + Ambiguitätsregel.
7. Anker-Regel (raw-Zeile als Referenz für Graded) — macht die vier stummen Wächter erstmals wirksam.
8. `fits`-Filter für den Browse-Zweig + `check_price_plausibility` in `eu_probe`.
9. Dritte Grade-Leiter (Comics) in `_grade_fields`, Docstring `pricecharting.py:6` korrigieren.

**P2 — Slab-Genauigkeit**
10. QR-/Barcode-Kanal (cv2, bereits installiert) + Label-Band-Crop.
11. PSA Public API mit permanentem Cert-Cache.
12. Monotonie-Invariante + Sweeper in `periodic_refresh`.

**P3 — Skalierung und Recht**
13. PriceCharting-Redistributionslizenz anfragen **oder** E5 streichen.
14. Fulfillment-Scope nachziehen → E1 aufbauen. Das ist die einzige Quelle, die bei zehntausenden Nutzern trägt und niemandem gehört außer Sven.
15. Migration: die 223 `solo:`-Zeilen und 262 namenlosen `cards`-Zeilen aufräumen (getrennter, sorgfältig geprüfter Lauf — nach dem Vorfall vom 02.08. mit Backup und Trockenlauf).

---

## 11. Verifiziert vs. unbestätigt

**Selbst im Code verifiziert:** alle Datei-/Zeilenangaben oben; OpenCV 5.0.0 im venv `/Users/smorty/ebay-bot/.venv` mit `QRCodeDetector`, `QRCodeDetectorAruco` **und** `barcode.BarcodeDetector` (der QR-Kanal braucht keine neue Abhängigkeit); `rembg` 2.0.76, `numpy` 2.4.6 installiert, aber **nicht in `requirements.txt`** (dort stehen nur telegram, anthropic, httpx, Pillow, dotenv, pytest, pillow-heif) — auf einem frischen Server fiele der Freisteller stumm aus, weil `cardscan.py:254` die `ImportError` abfängt; `pyzbar`, `rapidfuzz`, `imagehash` fehlen (Token-Ähnlichkeit ginge notfalls mit `difflib` aus der Standardbibliothek, pHash bräuchte `imagehash` oder eine Eigenimplementierung auf `numpy`).

**Aus der vorgelagerten Recherche übernommen, nicht erneut geprüft:** PriceCharting-Feldbedeutungen und ToS-Wortlaut; PSA-API-Endpunkt und 100-Calls-Limit; eBay-Deprecation-Daten; Marketplace-Insights-Ablehnung (steht allerdings wörtlich im eigenen Code, `/Users/smorty/ebay-bot/web/ebay_insights.py:1-13`); eBay-User-Agreement-Fassung 20.02.2026.

**Unbestätigt und als solches zu behandeln:** PSA-Cert-Nummernlänge (nur Drittquellen, 8–10 Ziffern); Semantik der PSA-Labelfarben; Existenz von Partner-APIs bei CGC/Beckett/SGC/WATA/VGA (weder bestätigt noch widerlegt — müsste direkt angefragt werden); VGA/CGA-Seriennummernformat; ob `/api/products` bei PriceCharting dasselbe Ranking wie die Website-Suche benutzt; exakte Ximilar-Credit-Kosten pro `slab_id`-Call.

**Die Bit-Gewichte in §5.3 sind begründete Startwerte, keine gemessenen.** Der saubere Weg: 200–500 Paare (Item, Kandidat) aus der Historie labeln, logistische Regression auf den sieben Merkmalen — die Koeffizienten *sind* dann die empirischen log-Odds-Gewichte (geteilt durch `ln 2` für Bits) — Platt-Scaling auf einem Holdout (bei dieser Datenmenge nicht isotonisch, das überanpasst), dann `τ` an der Risk-Coverage-Kurve ablesen statt zu raten. Bis dahin trägt das harte Domänen-Gate den Fehlerfall auch ohne jede Kalibrierung.

---

## 12. Berührte Dateien im Überblick

| Datei | Stufe | Art des Eingriffs |
|---|---|---|
| `/Users/smorty/ebay-bot/web/cardscan.py` | S1, S2 | `label_crop()`, `glanz_score()` neu; `DETECT_PROMPT:67-86` um `item_class` erweitern |
| **`/Users/smorty/ebay-bot/web/slab.py`** | S3, S8 | **neu** — Kanon, Skalen, `grade_rank`, QR/Barcode (cv2) |
| **`/Users/smorty/ebay-bot/web/match.py`** | S4 | **neu** — Domänen-Gate, Bit-Scoring, Ambiguität |
| `/Users/smorty/ebay-bot/bot/claude_client.py:69-90,131,139-161` | S2, S3 | `item_class` als Pflichtfeld; Kanon aus `web/slab.py`; `graded_info=null` erzwingen wenn `holder != slab` |
| `/Users/smorty/ebay-bot/web/prices.py:78-88` | S4 | `item_class` als Vorgabe im `identify_card`-Prompt |
| `/Users/smorty/ebay-bot/web/pricecharting.py:6,21-46,76-121` | S4, S5 | `/api/products`; dritte Grade-Leiter (Comics); zweistufiger Cache; Docstring-Fehler |
| `/Users/smorty/ebay-bot/web/catalog.py:48-60,81-84,102-250` | S4–S6, S8 | Schlüssel; Bucket-Normalisierung; Kaskade nach Belegklasse; Anker; Monotonie |
| `/Users/smorty/ebay-bot/bot/ebay/browse.py:19` | S5 | Parameter `titel_filter` (injiziert, kein Import) |
| `/Users/smorty/ebay-bot/web/sold.py:104-111,175-210` | S3, S5 | Grader-Kanon zentral beziehen; `fits` als `web/relevanz.py` teilbar machen |
| `/Users/smorty/ebay-bot/web/app_api.py:725-734,790,795-801,833-855,1000-1022,2384` | alle | Feldliste; `setdefault`; Anker; beide Preispfade angleichen; Sweeper |
| `/Users/smorty/ebay-bot/bot/main.py:238-266` | S5 | `check_price_plausibility` auch im EU-Zweig anwenden |

---

## 13. Schonungslos

Der Baum oben ist umsetzbar und liegt zu etwa 70 % schon als Code vor — `catalog.py` als Konzept, `sold.fits` als strengster Relevanzfilter im System, die TCGdex-Nummernpflicht als Vorbild, die Schreib-Isolation, der Papierkorb. Das ist gute Arbeit, und nichts davon muss weg.

Aber zwei der drei Preisbeine tragen rechtlich nicht: PriceCharting untersagt genau die Nutzungsart, die SERO ist, und 130point ist gescraptes Scraping mit gefälschtem User-Agent gegen einen 403-Filter. Das dritte Bein — die Browse API — ist sauber und wird als Beiwerk behandelt.

Der Entscheidungsbaum ist deshalb bewusst so gebaut, dass er **auch dann funktioniert, wenn E5 und E2 wegfallen**. Er endet dann öfter bei „Wert unbekannt". Das ist kein Mangel des Entwurfs, sondern die ehrliche Abbildung der Datenlage — und exakt das, was Sven verlangt hat.

*Read-only-Hinweis: keine Datei geändert, kein Server gestartet, keine Requests gegen localhost, keine DB-Schreiboperation.*