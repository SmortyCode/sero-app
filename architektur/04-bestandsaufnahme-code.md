# SERO — Ist-Architektur der Preis-Pipeline (Bestandsaufnahme im echten Code, 03.08.2026)

Read-only-Analyse. Es wurde keine Datei geändert, kein Server gestartet, kein POST abgesetzt. Die DB wurde ausschließlich über `file:data.db?mode=ro&immutable=1` gelesen.

---

## 0. Kurzfassung

Die gewünschte Pipeline ist zu ca. 80 % gebaut: Vision-Analyse → Karten-Identifikation → Karten-DBs → Verkaufsbelege → globaler Katalog → Wächter → Listing existieren alle. Sie ist aber an **einer** Stelle strukturell gebrochen: der globale Katalog läuft praktisch leer, weil 223 von 230 `card_prices`-Zeilen unter einem Wegwerf-Schlüssel `solo:<uuid>` liegen. Ursache ist ein `setdefault`-Fehler in `app_api.py`, nicht ein Konzeptfehler in `catalog.py`. Alle Wächter, die das Fehl-Match des One-Piece-Manga hätten stoppen können, sind vorhanden — sie greifen nur genau in diesem Fall nicht, weil ihnen die Referenz (`base`) fehlt.

---

## 1. Datenfluss (Ist)

```
Foto(s) → /collection/scan | /items-from-stage | /scan-batch  (app_api.py:1736 / 1943 / 1996)
   └─ Item angelegt: {status:"analyzing", photos, notes, quantity}   ← nur 4 Felder!
   └─ enqueue_scan → _scan_worker (app_api.py:2450, 2 Worker, Round-Robin, SCAN_TIMEOUT)
         └─ analyze_collection_item (app_api.py:875)
              1. cardscan.crop_photos       (web/cardscan.py:266, rembg/BiRefNet, lokal)
              2. analyzer.analyze           (bot/claude_client.py:170, Vision → Listing-JSON)
              3. ggf. cardscan.slab_recut   (web/cardscan.py:287)
              4. identify_card              (web/prices.py:61, Haiku-Text-Call)
              5. lookup_card_price          (web/prices.py:300 → TCGdex/Scryfall/YGOPRODeck/tcgcsv)
              6. research_price             (bot/ebay/browse.py:19, AKTIVE Angebote)
                 + check_price_plausibility (bot/main.py:238)
              7. catalog.refresh_price      (web/catalog.py:102) ← der eigentliche Preis-Entscheid
                 ├─ fetch_sold              (web/sold.py:213 → ebay_insights → 130point)
                 ├─ lookup_pc               (web/pricecharting.py:76)
                 └─ eu_probe = research_price(ebay, q)   (Browse, −12 %)
```

`refresh_item_price` (app_api.py:755) ist derselbe Ablauf ohne Vision — für Re-Scan, Refresh-Knopf, `match_item` und den 12-h-Autolauf `periodic_refresh` (app_api.py:2384).

---

## 2. Frage 1 — Die HEUTIGE Preis-Kaskade als Entscheidungsbaum

### 2a. Innerer Baum: `catalog.refresh_price` (web/catalog.py:102–250) — hier wird der Wert bestimmt

```
START refresh_price(card_key, grade, query, graded, usd_rate, base, force, eu_probe)
│
├─ [CACHE] row = get_price(card_key, grade)                          catalog.py:110
│   TTL gestaffelt nach Quelle der ALTEN Zeile:                      catalog.py:115
│     source=="ebay_sold"                → 24 h
│     source in ("ebay_eu","pricecharting") →  6 h
│     alles andere (estimate/weak/None)  →   1 h   ("Selbstheilung")
│   wenn row frisch UND not force → RETURN row   (kein einziger externer Call)
│
├─ [QUERY-ANREICHERUNG] Grader+Note an die Query hängen,             catalog.py:118-122
│   falls der Grader-Name noch nicht drinsteht ("… BGS 9.5")
│
├─ [PARALLEL] asyncio.gather(fetch_sold, lookup_pc)                  catalog.py:127
│     sold = 130point-Ø der letzten ≤3 Verkäufe   (Exception → None)
│     pc   = PriceCharting-Feldwert                (Exception → None)
│
├─ [SEQUENTIELL] eu = eu_probe(query) → Browse-Median × 0,88         catalog.py:139-147
│     nur wenn: median vorhanden, NICHT estimated,
│     count ≥ 2 (graded) bzw. ≥ 3 (raw)
│
├─ [PC-VERTRAUENSPRÜFUNG] pc_trusted                                 catalog.py:149-174
│     qhard = alle Ziffern-Token der Query
│             minus Grade-Token, minus Set-Nenner ("199/165" → 165)
│     pc_trusted = qhard nicht leer UND JEDES qhard-Token
│                  steht im PC-Produktnamen/Konsolennamen
│                  (Plattform-Synonyme: ps2→"playstation 2" usw.)
│
└─ [GEWINNER-KASKADE]  (erste zutreffende Zeile gewinnt)             catalog.py:184-207
   ┌───────────────────────────────────────────────────────────────────────────┐
   │ 1. sold vorhanden?            → value = sold.avg3      source="ebay_sold"  │
   │      Label je nachdem "Ø letzte N eBay-Verkäufe" (+ "älter als 90 Tage")   │
   │ 2. sonst eu vorhanden UND NICHT (pc_trusted und eu > pc*4)                 │
   │                               → value = eu             source="ebay_eu"    │
   │ 3. sonst pc UND pc_trusted    → value = pc.value       source="pricecharting"│
   │ 4. sonst base UND grade=="raw"→ value = base.value     source=base.source  │
   │      ⚠ base wird bei GEGRADETEN Stücken NIE genutzt                        │
   │ 5. sonst pc (unbelegt)        → value = pc.value       source="pricecharting_weak"│
   │ 6. sonst alte row             → RETURN row (unverändert)                   │
   │ 7. sonst                      → RETURN None                                │
   └───────────────────────────────────────────────────────────────────────────┘
   ↓
   [WÄCHTER A] nur wenn source=="ebay_sold" UND n_avg < 2             catalog.py:212-224
   [WÄCHTER B] nur wenn source=="pricecharting_weak"                  catalog.py:231-240
   ↓
   INSERT/UPDATE card_prices (card_key, grade) + RETURN get_price(...)
```

**Konfliktregel in einem Satz:** echte eBay-Verkäufe (130point) schlagen den EU-Markt, der EU-Markt schlägt PriceCharting, PriceCharting schlägt die Karten-DB-Basis (nur bei `raw`), und ein unbelegter PriceCharting-Treffer ist der allerletzte Rückfall vor „gar nichts".

### 2b. Äußerer Baum: `refresh_item_price` (app_api.py:755–873)

```
1. card_info fehlt, analysis da?  → identify_card nachholen        :759-762
2. card_info.single == True?
   └─ lookup_card_price(TCGdex/Scryfall/YGOPRODeck/tcgcsv)         :764-774
      Treffer → card/est_value/price_source/price_label/price_detail, updated=True
      kein Treffer UND alte Quelle war "tcgplayer" → alles verwerfen (auf None) :775-782
3. catalog.upsert_card → card_key ; grade = grade_bucket(graded)   :789-793
   base = {est_value,…} NUR WENN updated (also nur bei Karten-DB-Treffer!) :795-797
   row = catalog.refresh_price(...)                                 :798-801
   row.value_eur da? → est_value/price_source/price_label überschreiben,
                       detail.sold → item["sold"], detail.pc / detail.extra
                       werden in price_detail gemerged, updated=True :805-816
4. research = research_price(Browse) + check_price_plausibility     :823-824
   → item["market"] (Zweitmeinung, immer)                           :826-827
5. WÄCHTER C: price_source=="pricecharting_weak" UND
   |Wert| > 4× oder < ¼ des Browse-Medians → Wert auf Markt-Median  :833-845
6. if not updated (weder Karten-DB noch Katalog lieferten etwas):   :846-855
     research nicht estimated → est_value=median, source="ebay"
     research estimated und est_value None → source="estimate" ("KI-Schätzung")
7. price_updated=now; frisch nachlesen; NUR PREIS_FELDER schreiben  :856-868
8. snapshot_price + check_alert                                     :871-872
```

`analyze_collection_item` (app_api.py:875) ist fast identisch, mit zwei Unterschieden:
* Reihenfolge ist Karten-DB → Browse/Schätzung → **danach** Katalog (:996–1022), d. h. `base` ist hier **immer gesetzt**, sobald irgendein `est_value` existiert (:1003–1005) — auch eine reine KI-Schätzung.
* Zusätzlich gesetzt: `est_low`/`est_high` aus `analysis.estimated_price_range_eur` (:971–976) und als letzter Rückfall der Mittelwert daraus (:986–989).
* **Wächter C fehlt hier** — der `pricecharting_weak`-Abgleich gegen den Browse-Median existiert nur in `refresh_item_price`.

---

## 3. Frage 2 — Vorhandene Plausibilitäts- und Schutzmechanismen

| # | Ort | Was er prüft | Wann er greift / wo er blind ist |
|---|---|---|---|
| P1 | `check_price_plausibility` (bot/main.py:238) | Browse-Median gegen Claudes `estimated_price_range_eur`; < 3 Comps **oder** Median außerhalb `low*0.4 … high*2.5` → ersetzt Comps durch die Schätzung (`estimated:true`) | app_api.py:824 und :967. **Nicht** auf die `eu_probe`-Abfrage im Katalog angewandt (app_api.py:801/1010 ruft `research_price` direkt) — der EU-Zweig umgeht diesen Wächter komplett |
| P2 | `pc_trusted` (catalog.py:149-174) | Jedes Ziffern-Token der Query muss im PC-Produktnamen stehen (Grade-Token und Set-Nenner ausgenommen, Plattform-Synonyme berücksichtigt) | Degradiert den Treffer zu `pricecharting_weak`, verwirft ihn aber nicht |
| P3 | Ausreißer-Wächter „sold" (catalog.py:212-224) | Nur bei **einem einzigen** Beleg (`n_avg < 2`): Wert > 6× oder < ⅙ der Referenz (pc_trusted-Wert oder alte Katalogzeile) → zurück auf die Referenz, `detail.outlier_sold` | Bei 2–3 Belegen greift er nie; `fetch_sold` liefert ohnehin erst ab 2 Verkäufen ein Ergebnis (sold.py:281/288) — der Wächter ist damit faktisch **tot** |
| P4 | Wächter `pricecharting_weak` (catalog.py:231-240) | Wert > 4× oder < ¼ von `base.value` → verwerfen, zurück auf `base`, `detail.verworfen_pc` | Braucht `base`. In `refresh_item_price` ist `base` nur bei Karten-DB-Treffer gesetzt (:795-797) → **beim Manga immer None → Wächter stumm** |
| P5 | Wächter C (app_api.py:833-845) | dasselbe gegen den **Browse-Median** statt gegen `base` | Nur in `refresh_item_price`, **nicht** in `analyze_collection_item` → beim allerersten Scan greift er nicht |
| P6 | `_zwischenstufe` (pricecharting.py:49-73) | Lineare Interpolation 9.0↔9.5↔10.0, nur wenn **beide** Stützstellen existieren und `c_oben > c_unten` | Löst die 9.0-vs-9.4-Gleichheit, aber nur oberhalb 9.0; unterhalb bleibt alles im selben Feld |
| P7 | `sold.fits` (sold.py:175-210) | Alle Ziffern-Token (außer Jahreszahl) müssen im Verkaufstitel stehen; Grader-Trennung (PSA≠CGC, Beckett→BGS normalisiert); ungegradet nie gegen Slab; ≥ ½ der weichen Wörter | Der schärfste Filter im System |
| P8 | 90-Tage-Regel (sold.py:266-301) | Ø aus max. 3 Verkäufen; ≥2 frische → normal, sonst ≥2 alte → `stale:true` | Bewusst weich für seltene Ware |
| P9 | Drossel/Cooldown (sold.py:35-67) | 8 s Mindestabstand global, nach 429 → 10 min Pause, 12-h-Cache auch für Misserfolge | Bei Cooldown fällt die Kette stumm auf PC/Browse zurück |
| P10 | IQR-Trimm (browse.py:52-61) | Ausreißer bei ≥5 Angeboten kappen | **Keinerlei Titelprüfung** — kappt nur Preise, nicht falsche Artikel |
| P11 | TCGdex-Nummernpflicht (prices.py:154-161, :177) | Ist die Kartennummer erkannt, MUSS sie passen, sonst gar kein Treffer; widersprüchliche Set-Größe → Kandidat raus | Vorbildlich streng — genau diese Härte fehlt bei PriceCharting |
| P12 | `_grader_im_titel_richtigstellen` (claude_client.py:145-161) | Falsches Grader-Kürzel im **Titel** ersetzen (Beckett→BGS) | Korrigiert **nur den Titel**, nicht `graded_info["grader"]` → siehe Katalog-Buckets |
| P13 | `crop_quad`/`slab_recut`-Plausibilität (cardscan.py:133, :313, :320) | Zuschnitt-Seitenverhältnis 1.1–2.0, Kantensymmetrie ≤ 8 %, Mindestgröße | Bildseitig, keine Preisrelevanz |
| P14 | Schreib-Isolation (app_api.py:725-753, :862-868) | `PREIS_FELDER` / `ANALYSE_FELDER` / `NUR_WENN_LEER`; vor dem Schreiben frisch nachlesen | Schützt Nutzerdaten vor dem langlaufenden Scan — muss jedes neue Feld bewusst eintragen |
| P15 | Papierkorb (app_api.py:2130ff) | Item-JSON + Fotos 30 Tage nach `_trash` | Reaktion auf den Datenverlust vom 02.08. |

---

## 4. Frage 3 — Wo genau Fehl-Matches entstehen

**F1 — `lookup_pc` hat keinerlei Typ-/Kategorie-Bindung (web/pricecharting.py:87-92).**
Der Aufruf ist `GET /api/product?q=<Freitext>`. Es wird nur `prod["id"]` auf Existenz geprüft. Es gibt keine Prüfung auf `console-name` gegen die erwartete Produktart. Ein Buch („One Piece Band 103") und eine Sammelkarte („Nami [Manga] OP01-016") sind für diese Funktion ununterscheidbar. Das ist die Primärursache des 603-€-Falls.

**F2 — `pc_trusted` degradiert, verwirft aber nicht (catalog.py:171-174 + :199-203).**
Die Prüfung erkennt den Fehl-Match korrekt (`pricecharting_weak`), der Wert wandert trotzdem als Kaskaden-Stufe 5 in die Datenbank. Ein als „unbelegt" erkannter Treffer sollte kein Preis sein.

**F3 — Wächter P4 ist beim Manga strukturell stumm (app_api.py:795-797).**
`base` wird in `refresh_item_price` nur gesetzt, wenn `updated` — und `updated` wird nur bei einem Karten-DB-Treffer wahr. Nicht-Einzelkarten (Manga, Sealed, Games) haben nie einen Karten-DB-Treffer → `base = None` → `if ref and ref > 0` (catalog.py:233) ist falsch → der Wächter läuft ins Leere. Genau die Warengruppe, für die PriceCharting am unzuverlässigsten ist, ist ungeschützt.

**F4 — `base` wird bei gegradeten Stücken nie als Wert genutzt (catalog.py:196).**
`elif base and … and grade == "raw"`. Bei jedem Slab fällt die Kaskade an dieser Stufe durch und landet direkt auf `pricecharting_weak` — dem unsichersten Zweig.

**F5 — `research_price` filtert Titel überhaupt nicht (bot/ebay/browse.py:19-76).**
Es gibt IQR-Trimming, aber keine `fits()`-Prüfung, keine Grader-Trennung, keine Nummernprüfung. Über `eu_probe` (catalog.py:139-147, `−12 %`) wird dieser ungefilterte Median zur Kaskaden-Stufe 2 — stärker als PriceCharting. Und weil der Katalog `research_price` direkt aufruft, läuft er ohne P1. In Produktion sichtbar: `pokemon:sv03.5-199 / CGC 10 → 803,99 € aus ebay_eu`.

**F6 — `_grade_fields` mappt Kartenfelder auf Nicht-Karten (pricecharting.py:21-46).**
Nur `WATA/VGA/CGA` bekommen den Spiele-Zweig. Ein Beckett-gegradetes **Buch** landet im Kartenzweig (`graded-price`/`box-only-price`) — bei PriceCharting bedeutet `box-only-price` je nach Produktart etwas völlig anderes. Zusätzlich kennt die Quelle nur 9 / 9.5 / 10, weshalb 9.0 und 9.4 ohne P6 identisch bleiben.

**F7 — `identify_card` klassifiziert Nicht-Karten korrekt als `single:false`, liefert dann aber `name:null` (prices.py:80-88).**
Damit endet für alle Sealed/Games/Bücher jede Identität — und der Katalog-Schlüssel bricht (siehe § 6).

**F8 — Der Wert kommt aus einer Query, die niemand gegen das Stück prüft.**
`listing["search_query_for_pricing"]` (claude_client.py:77) ist freier Claude-Text und wird ungeprüft an 130point, PriceCharting **und** Browse gereicht (app_api.py:799, :821). Es gibt kein Feld, das festhält, *welches Produkt* die Kaskade eigentlich bepreist hat — außer `price_detail.pc_product` im Nachhinein.

**F9 — Vergiftete Katalogzeile.** Wenn Wächter C (app_api.py:833-845) den weak-Wert am **Item** korrigiert, bleibt die **Katalogzeile** auf dem falschen Wert stehen. Der nächste Leser derselben `card_key`+`grade`-Zeile bekommt wieder 603 €. Im Produktionsbestand: 87 von 230 Zeilen (38 %) stehen auf `pricecharting_weak`.

---

## 5. Frage 4 — ALLE Felder eines Sammlungsstücks (`collection_items.data`)

Tabelle: `collection_items(id TEXT PK, account_id INTEGER, created_at REAL, updated_at REAL, data TEXT)` — alles Weitere steckt als JSON-Blob in `data`. `id` und `created_at` werden beim Lesen hineinkopiert (app_api.py:348-349) und beim Schreiben wieder entfernt (app_api.py:364).

### 5.1 Top-Level-Felder (Code + Produktionsbestand, 15 Items)

| Feld | Typ | Gesetzt in | Bemerkung |
|---|---|---|---|
| `status` | str | :1774, :1991, :2043, :1024, :1033 | `analyzing` \| `ready` \| `error` |
| `status_text` | str/None | :886, :900, :944, :963, :993, :2493 | Live-Fortschritt |
| `error` | str/None | :456, :467, :1035, :2477 | Sichtbarer Fehlertext |
| `name` | str | :906, :2109, Import :2797 | Titel; `NUR_WENN_LEER` |
| `category` | str | :938, :2111 | `guess_category` |
| `condition` | str/None | :907, :2113 | eBay-Zustand |
| `quantity` | int | :1774, :2119 | Default 1 |
| `notes` | str/None | :1774, :2124 | Nutzertext |
| `tags` | list[str] | :2128 | max. 12 × 30 Zeichen |
| `favorite` | bool | :2125 | Nutzer |
| `wishlist` | bool | :2126 | Nutzer |
| `purchase_price` | str | :2122 | über `parse_price` |
| `photos` | list[str] | :1773, :895, :937 | Pfade, ggf. `_cut.png`/`_slab.png` |
| `photos_raw` | list[str] | :894 | Originale vor dem Freisteller |
| `remote_photos` | list[str] | :2757, :2807 | eBay-URLs bei Import |
| `analysis` | dict | :941 | **komplettes Claude-Listing-JSON** |
| `card_info` | dict | :762, :949, :3576 | `identify_card`-Ausgabe |
| `card` | dict/None | :769, :778, :956 | Karten-DB-Datensatz |
| `card_key` | str | :791, :1002 | Katalog-Schlüssel |
| `est_value` | float/None | :770, :806, :841, :848, :957, :987 | **die große Zahl** |
| `est_low` | float/None | :973 | aus `analysis` |
| `est_high` | float/None | :974 | aus `analysis` |
| `price_source` | str/None | :771, :807, :842, :849, :958 | `cardmarket`\|`scryfall`\|`ygoprodeck`\|`tcgplayer`\|`ebay_sold`\|`ebay_eu`\|`pricecharting`\|`pricecharting_weak`\|`ebay`\|`estimate`\|`listing` |
| `price_label` | str/None | :772, :808, :843, :959 | Anzeigetext |
| `price_detail` | dict/None | :773, :813, :839, :960 | Quellen-Rohwerte, gemerged |
| `price_updated` | float | :856, :1026 | Unix-Zeit |
| `market` | dict/None | :826, :969 | Browse-Zweitmeinung |
| `sold` | dict/None | :811, :1020 | **Verkaufsbelege** (nicht „verkauft"!) |
| `sold_ts` | float | :2591 | „ist verkauft" — Namenskollision im Code dokumentiert (:539-542) |
| `graded` | dict/None | :909, Import :2755 | Slab-Label |
| `grading` | dict | :3492 | KI-Grading-Schätzung (card-grader) |
| `graded_market` | dict | :3459 | PSA-9/10-Marktabfrage |
| `draft_id` | str | :2757, Adopt | Verknüpfung zum eBay-Entwurf |
| `scan_seconds` | float | :2471 | echte Scan-Dauer |
| `id` | str | :348 | *nicht persistiert* (Spalte) |
| `created_at` | float | :349 | *nicht persistiert* (Spalte) |

### 5.2 Verschachtelte Strukturen

**`analysis`** (bot/claude_client.py:68-90): `title`, `subtitle`, `description_html`, `category_query`, `condition`, `condition_description`, `aspects`, `search_query_for_pricing`, `estimated_price_range_eur{low,high}`, `user_price`, `best_offer{enabled,min_price}`, `format`, `auction_days`, `quantity`, `main_image_index`, `graded_info{grader,grade,cert_number}`, `assumptions`, `estimated_weight_grams`, `uncertain`, `question`.

**`card_info`** (prices.py:80-88 + app_api.py:3576): `single`, `game`, `name`, `number`, `set_total`, `set_hint`, `manual`, `tcgcsv`.

**`card`** (prices.py:214-225, :247-252, :283-286, tcgcsv.py:266): `game`, `name`, `name_en`, `set_name`, `number`, `total`, `rarity`, `ref_id`, `image`, `language`, `illustrator`, `variants`, `hp`, `types`.

**`graded`**: `grader`, `grade`, `cert_number`.

**`market`** (app_api.py:827): `count`, `min`, `max`, `median`, `estimated`, `samples[{title,price,url,image}]`.

**`sold`** (sold.py:283-301): `avg3`, `n_avg`, `stale`, `sales[{price,currency,title,date,url,image,old,price_eur}]`.

**`price_detail`** (Union aus mehreren Quellen): `pc_usd`, `pc_field`, `pc_product`, `pc_console`, `pc_id`, `tcgplayer_usd`, `tcgplayer_eur`, `trend`, `avg1`, `avg7`, `avg30`, `low`, `updated`, `eur`, `eur_foil`, `cardmarket`, `verworfen_pc`.

**`grading`** (app_api.py:3492-3501): `psa_low`, `psa_high`, `psa_likely`, `confidence`, `summary`, `worth_grading`, `recommendation`, `defects`, `ts`.

**`graded_market`** (app_api.py:3459): `psa10{median,count,query,samples}`, `psa9{…}`, `ts`.

**Nur in `card_prices.detail`, nicht am Item** (catalog.py:176-198, :217, :237): `sold`, `pc`, `value_us`, `value_eu`, `extra`, `outlier_sold`, `verworfen_pc`.

**Nur in der API-Antwort, nicht persistiert** (app_api.py:510-561, :2064-2086): `sold_comps`, `history`, `dublette`, `alert`, `draft`, `draft_status`, `item_url`.

> **Abwärtskompatibilität:** Jedes neue Feld muss in `PREIS_FELDER` (:725) bzw. `ANALYSE_FELDER` (:731) eingetragen werden, sonst wird es von `col_save_analyse`/`refresh_item_price` beim nächsten Lauf **stillschweigend verworfen** — es überlebt nur, wenn es außerhalb dieser Pfade geschrieben wird.

---

## 6. Frage 5 — Der globale Katalog und wo `card_key` bricht

### 6.1 Konzept
`cards(card_key PK, game, name, set_name, number, total, rarity, language, image, ref_id, updated_at)` +
`card_prices(card_key, grade) PK → value_eur, source, source_label, detail, updated_at`.
Eine Karte existiert einmal, ihr Preis pro Grade-Stufe einmal; alle Nutzer lesen dieselbe Zeile, jede externe Quelle wird 1× pro Karte statt 1× pro Nutzer befragt (catalog.py:1-8).

### 6.2 Schlüsselbildung (`card_key_of`, catalog.py:48-60)
```
1. weder ref_id noch name  → "solo:" + uuid4().hex[:12]      ← Wegwerfschlüssel
2. ref_id vorhanden        → f"{game or 'x'}:{ref_id}"        ← z. B. "pokemon:sv03.5-199"
3. sonst                   → "h:" + sha1(norm(game|name|number|set_name))[:16]
```
`grade_bucket` (catalog.py:81-84): `"raw"` oder `f"{GRADER.upper()} {grade}"`.

### 6.3 Wo es bricht — mit Zahlen aus der Produktions-DB

**B1 — Der `solo:`-Schlüssel ist kein Randfall, er ist der Normalfall.**

| Messung (data.db, read-only) | Wert |
|---|---|
| `card_prices` gesamt | 230 |
| davon `solo:` | **223 (97 %)** |
| davon `h:` | 2 |
| `cards` gesamt | 282 (bei **15** Items!) |
| `cards` ohne `name` | **262** |
| identische Wegwerf-Zeilen | `WATA 9.8 / 97,52 €` ×75, `BECKETT 9.0 / 603,12 €` ×70, `raw / 98,63 €` ×61 |

**B2 — Die Ursache ist ein `setdefault` auf einem vorhandenen `None`** (app_api.py:789-791 und :999-1001):
```python
card_ref = dict(item.get("card") or card_info or {})
card_ref.setdefault("name", item.get("name"))
```
`identify_card` liefert für Nicht-Einzelkarten immer ein Dict **mit** Schlüssel `"name"` — Wert `None` (prices.py:82-88, Beleg im Bestand: `ci.single=False, ci.name=None` bei allen Manga-/Games-Items). `setdefault` setzt nur bei **fehlendem** Schlüssel, nicht bei `None`. Also bleibt `name = None`, `ref_id` fehlt → Zweig 1 → neuer `uuid4()`. Und weil das bei **jedem** Lauf passiert (Analyse, Refresh-Knopf, 12-h-Autolauf), entsteht jedes Mal ein neuer Schlüssel: der Katalog trifft nie seinen eigenen Cache, `cards` wächst monoton (282 Zeilen für 15 Items), jede Zeile ist beim nächsten Lauf Müll.

**Folgen im Betrieb:** kein Cache-Hit → jeder Refresh feuert erneut gegen 130point (8 s Drossel, 429-Cooldown) und PriceCharting; die 24-h-TTL für `ebay_sold` ist wirkungslos; `aehnliches_stueck` (app_api.py:419-422) schließt `solo:`-Schlüssel bewusst aus und kann Dubletten nur noch über den Namensfingerabdruck finden; die Grundannahme „zehntausende Nutzer teilen sich eine Preisabfrage" ist für Sealed/Games/Bücher heute nicht wirksam.

**B3 — Der `h:`-Schlüssel ist bei Nicht-Karten instabil.**
Er hasht `game|name|number|set_name`. Bei Nicht-Karten ist `name` letztlich Claudes Titel. Wird das Stück umbenannt (app_api.py:2109) oder liefert eine Re-Analyse einen minimal anderen Titel, ändert sich der Schlüssel — die alte Preiszeile ist verwaist. Fehlt `game`, wird `"x"` eingesetzt: gleicher Name + gleiche Nummer in zwei Spielen kollidieren auf einer Zeile.

**B4 — Der Schlüssel kennt keine Sprache, Auflage und Variante.**
`cards.language` existiert als Spalte, geht aber nicht in den Hash ein. Deutsch/Englisch, 1st Edition/Unlimited, Holo/Reverse Holo landen auf **einer** Preiszeile — obwohl sie sich im Preis um ein Vielfaches unterscheiden.

**B5 — Die Grade-Buckets fragmentieren, weil der Grader-Name nicht normalisiert wird.**
`grade_bucket` nimmt `graded["grader"]` roh. Im Bestand steht deshalb `"BECKETT 9.0"`, während `sold.py:111` intern `beckett → bgs` normalisiert und `claude_client.py:145` nur den **Titel** korrigiert. `"BECKETT 9"` / `"BGS 9"` / `"BECKETT 9.0"` sind drei verschiedene Buckets für dasselbe Objekt. Auch `"9"` vs. `"9.0"` trennt.

**B6 — Katalogzeilen werden vergiftet und nie repariert.**
Korrekturen der Wächter C (app_api.py:833) landen nur am Item. Die Zeile `solo:… / BECKETT 9.0 / 603,12 € / pricecharting_weak` steht unverändert in der DB — heute 70× dupliziert.

**B7 — Es gibt keinen Aufräum-/Migrationspfad.** Kein Index auf `card_prices.updated_at`, kein Löschen verwaister `solo:`-Zeilen, kein Fremdschlüssel von `collection_items` auf `cards`.

---

## 7. Externe Ausgangslage — jetzt belegt

* **Finding API / `findCompletedItems`:** `findCompletedItems` wurde am **15.10.2020** deprecated und im Zugang beschränkt; die gesamte Finding API ist seit **04.01.2024** deprecated und wurde zum **05.02.2025** abgeschaltet. Nachfolger für Suche ist die Browse API — **ohne** Sold-Daten. Quelle: [eBay API deprecation status](https://developer.ebay.com/develop/get-started/api-deprecation-status), [eBay Community: Finding & Shopping API decommissioned 2025](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062).
* **Marketplace Insights API:** Limited Release, nur für von eBay-Business-Units freigegebene Partner; laut Doku „restricted and not open to new users at this time", Zugang wird nicht auf Antrag erteilt. Quelle: [Marketplace Insights Overview](https://edp.ebay.com/api-docs/buy/marketplace-insights/static/overview.html), [Get Started on a Buying Application](https://developer.ebay.com/develop/get-started/get-started-on-a-buying-application).
* Das deckt sich mit dem, was im Code steht: `web/ebay_insights.py:1-13` dokumentiert die **Ablehnung vom 03.08.2026** im Wortlaut, `enabled` ist per Default aus (`SERO_EBAY_INSIGHTS=1` schaltet es wieder scharf, :28). Der Aufruf steht bereits an **erster** Stelle in `fetch_sold` (sold.py:225-228) — die Umschaltung wäre eine Umgebungsvariable, kein Umbau.
* **Unbestätigt:** ob `back.130point.com/sales/` (POST `query`/`type=2`/`subcat=-1`) eine offiziell dokumentierte Schnittstelle ist oder nur ein Backend-Endpunkt der Website. Der Code parst HTML-Zeilen per Regex (`sold.py:31`, `_parse_rows`) — das ist ein Scraping-Vertrag ohne Zusage, mit sichtbarer Bursts-Drossel (429 → 10 min Cooldown).

---

## 8. Fazit für die Architektur — was tragfähig ist und was nicht

**Tragfähig und behaltenswert:**
`catalog.py` als Konzept (card_key × grade, TTL-Staffelung nach Quellengüte, Selbstheilung schwacher Werte); `sold.fits` als strengster Relevanzfilter im System; die TCGdex-Nummernpflicht als Vorbild für alle anderen Quellen; die Schreib-Isolation `PREIS_FELDER`/`ANALYSE_FELDER` mit Frisch-Nachlesen; Papierkorb statt Hart-Löschen; die 130point-Drossel.

**Die vier Bruchstellen, an denen eine neue Architektur ansetzen muss — ohne Neubau:**
1. **Identität vor Preis.** `card_key_of` darf nie einen Zufallsschlüssel erzeugen; `setdefault` in app_api.py:790/1000 kann `None` nicht überschreiben. Ein deterministischer Schlüssel auch für Nicht-Karten (Warenart + normalisierter Titel + Sprache + Auflage) macht 223 von 230 Katalogzeilen erst nutzbar.
2. **Quellen müssen ihren Treffer belegen.** PriceCharting braucht dieselbe Härte wie TCGdex (Produktart-Abgleich Buch/Karte/Spiel, nicht nur Ziffern-Token), und ein unbelegter Treffer darf kein Preis werden, statt als `pricecharting_weak` in die DB zu wandern.
3. **Die Wächter brauchen immer eine Referenz.** `base` muss unabhängig davon existieren, ob ein Karten-DB-Treffer vorlag, und auch bei gegradeten Stücken zählen; Wächter C gehört zusätzlich in `analyze_collection_item`; Korrekturen müssen die Katalogzeile mitkorrigieren, nicht nur das Item.
4. **Der EU-Zweig ist heute ungefiltert.** `research_price` braucht eine Titelprüfung (mindestens `sold.fits`) und `check_price_plausibility`, bevor sein Median × 0,88 die zweitstärkste Stufe der Kaskade besetzt.

Sources:
- https://developer.ebay.com/develop/get-started/api-deprecation-status
- https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062
- https://edp.ebay.com/api-docs/buy/marketplace-insights/static/overview.html
- https://developer.ebay.com/develop/get-started/get-started-on-a-buying-application