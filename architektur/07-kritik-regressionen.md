# Skeptiker-Report: Regressionsrisiken der vorgeschlagenen Preis-Pipeline

Alles unten ist am echten Code nachgelesen (Datei:Zeile) oder read-only aus `data.db` (`file:…?mode=ro`) gemessen. **Nichts geändert, kein Server gestartet, kein Request gegen localhost.**

**Urteil vorweg:** Die Diagnose der Vorberichte ist im Kern richtig — der `card_key`-Leck ist real und wächst *heute noch*. Aber der Entwurf enthält **fünf Vorschläge, die produktive Funktionen brechen würden**, und **vier Aussagen, die am Code nicht stimmen**. Der gefährlichste Punkt ist nicht der Katalog, sondern die Kette `est_value = None` → Listing-Preis fällt weg → derselbe 307,90-€-Fehler, den Sven am 02.08. schon einmal gefixt hat.

---

## A — WÜRDE FUNKTIONIERENDES BRECHEN (nach Risiko sortiert)

### A1 🔴 `card_key_of` → `None` löst eine `IntegrityError` aus und *verschlimmert* den Müll

**Vorschlag** (Pipeline-Entwurf §5.4): `card_key_of` soll `None` statt `"solo:"+uuid` liefern, „der Aufrufer muss das als UNBEKANNT_IDENTITAET behandeln".

**Am Code:** Kein Aufrufer tut das. `/Users/smorty/ebay-bot/web/app_api.py:791` und `:1001` rufen unmittelbar `catalog.upsert_card(store, card_ref)` mit dem Rückgabewert.

Die Tabellen reagieren unterschiedlich (im Scratchpad gegen eine Wegwerf-DB verifiziert, nicht gegen `data.db`):
- `cards.card_key TEXT PRIMARY KEY` (`catalog.py:32`) — **SQLite erlaubt NULL in einem TEXT-PK**. `upsert_card` würde also bei *jedem* Lauf eine neue NULL-Zeile einfügen, weil `ON CONFLICT` bei NULL nie greift. Das ist schlimmer als heute.
- `card_prices.card_key TEXT NOT NULL` (`catalog.py:36`) — hier fliegt `IntegrityError: NOT NULL constraint failed`.

Die Exception landet im `except Exception` bei `app_api.py:802` bzw. `:1011` → `row = None`, geloggt, Scan läuft weiter. **Der Fehler wäre unsichtbar**, während `cards` weiter zumüllt.

**Sicherere Alternative:** `card_key_of` behält seinen Vertrag (immer ein String). Der Deterministik-Fix gehört **eine Ebene höher**: `app_api.py:790` / `:1000` von `setdefault` auf explizite Zuweisung umstellen — das ist der eigentliche Bug (siehe A2-Beleg). Wenn dann *immer noch* kein Kern übrig ist, `"solo:" + sha1(item_id)` statt `uuid4()` — deterministisch, kollisionsfrei, kein Schema-Risiko, und beim nächsten Refresh trifft der Cache endlich.

**Beleg, dass `setdefault` die Wurzel ist** (aus der Produktions-DB, alle sechs `solo:`-Items):
```
{'single': False, 'game': 'other', 'name': None, ...}  | One Piece #103 Manga Japanisch Beckett B…
{'single': False, 'game': 'other', 'name': None, ...}  | Grand Theft Auto Vice City PS2 USA WATA …
```
Schlüssel `"name"` ist **vorhanden** mit Wert `None` → `setdefault` ist ein No-Op → Zweig 1 in `catalog.py:49`.

---

### A2 🔴 „KI-Schätzung ist niemals ein Wert" bricht den Listing-Preis — im Bot *und* in der App

**Vorschlag** (Pipeline-Entwurf §0 Leitprinzip 4 + §9): `est_value` nur noch bei `price_state == "belegt"`, `price_source = "estimate"` verschwindet als Wertquelle, `app_api.py:852-855`, `:982-989`, `:842-844` streichen.

**Was daran hängt:**

```
app_api.py:2249-2252   est = item_value(item)
                       if est:
                           preset["market_value"] = f"{est:.2f}"
                           preset["market_source"] = item.get("price_label") or "Marktwert"
```
`market_value` fließt über `app_run_pipeline(..., market=market)` (`:2306-2310`) in `bot/main.py:215` → `apply_price_rule`. Dessen Rangfolge (`bot/main.py:222-235`):

```
Nutzerpreis > Vorlage > BELEGTER Marktwert > eBay-Comps > KI-Schätzung > None
```

`est_value = None` ⇒ `market_value` fehlt ⇒ die Pipeline recherchiert blind neu. **Das ist exakt der Fehler vom 02.08.**, dokumentiert in `STATUS.md:394`:

> „FEHLER 2 „307,90 € statt 653,02 €": … bei Listing OHNE price_mode macht die Pipeline eine BLIND-Recherche (0 aktive Angebote) → Rückfall auf Claudes Rahmen 250–400 € → Median 325 × 0,95 → 307,90. Der belegte Marktwert (est_value 653,02) wurde NIE gelesen."

Zusätzlich brechen die Preis-Modi: `app_api.py:2258` und `:2260` sind beide mit `and est` gegated — „Marktwert" und „Marktwert −10 %" liefern dann **stillschweigend keinen `tpl_price`**.

Und weiter, alles auf `est_value` gebaut:
| Funktion | Ort | Verhalten bei `None` |
|---|---|---|
| Preisalarm | `app_api.py:583-596` | `if … value is None: return` → **Alarm feuert nie** |
| Preisverlauf | `app_api.py:598-601` | `if value is None: return` → Lücke in der Kurve, `delta7` (`:1702`) tot |
| Sammlungswert | `app_api.py:1677`, `:2928` | `(i["est_value"] or 0)` → Wert **sinkt sichtbar**, ohne Erklärung |
| PWA-Anzeige | `sero.js:2167`, `:3270` | zeigt `—`; Sortierung `valdesc` (`:2052`) behandelt null als 0 |

**Sicherere Alternative:** `est_value` behalten, `price_state`/`price_reason` **additiv** danebenlegen. Die UI entscheidet, was sie zeigt — der Server behält seinen Vertrag. Konkret: bei `price_state != "belegt"` bleibt `est_value` gesetzt, `price_source` wird `"estimate"`, und `item_public` bekommt zusätzlich `value_verdict`. Für das Listing gilt weiterhin Svens Rangfolge; nur der *Anzeigetext* wird ehrlich. Svens Regel „lieber keine Zahl als eine falsche" heißt nicht „lieber ein Listing ohne Preis" — ein Listing ohne Preis ist auch ein Schaden.

---

### A3 🔴 Der neue System-Prompt trifft den Telegram-Bot mit — und er ist zu eng

**Vorschlag** (Schema-Bericht Teil B): `SYSTEM_PROMPT` in `bot/claude_client.py:29-95` ersetzen.

**Am Code:** `bot/main.py:526` ruft dieselbe `analyzer.analyze()`. Der Telegram-Bot ist Svens produktiver Listing-Weg für **alle** Warengruppen. Der heutige Prompt sagt in Zeile 29-31 ausdrücklich:

> „Weltklasse-Produkterkennungs- und eBay-Listing-Experte für **ALLE Produktkategorien** — Elektronik, Fashion, Haushalt, Spielzeug, Möbel, Werkzeug — mit besonderer Spezialtiefe bei Sammlerware"

Der Ersatzprompt öffnet mit „Du bist das Erkennungsmodul von SERO. Du siehst Fotos eines **Sammlerstücks**" und listet `item_class` ohne Elektronik/Fashion/Möbel/Werkzeug (nur `other`). Die Titel-Formel verliert die Kategorie-Muster für Elektronik/Sneaker (`claude_client.py:50-53`). Ein iPhone über Telegram bekäme danach `item_class: "other"` — und im Entwurf ist `other` = **ABBRUCH, kein Preis** (Pipeline §1, S2).

Weitere konkrete Regressionen im Ersatzprompt:
- **`condition_description` verliert die „nur bei Gebraucht"-Regel** (heute `:75`). Der Wert geht an die eBay-Inventory-API.
- **USK-Regel überlebt** (7.5), gut — aber `tests/test_usk.py` (11 Tests) prüft den *heutigen* Wortlaut. Vor jedem Prompt-Tausch: `pytest tests/test_usk.py tests/test_json_parsing.py tests/test_single_value_aspects.py` laufen lassen.

**Sicherere Alternative:** Prompt **nicht ersetzen, sondern ergänzen.** Die Sammler-Blöcke (Warenart, Slab-Regeln, Konfidenz) als zusätzliche Abschnitte anhängen, die Allgemeinware-Teile Wort für Wort stehen lassen. Und `item_class` um `electronics`, `fashion`, `home`, `tool` erweitern — sonst ist `other` kein Randfall, sondern Svens halbes Sortiment.

---

### A4 🟠 „E3 (Cardmarket) schlägt E2 (echte Verkäufe)" kippt einen ausdrücklichen Sven-Entscheid

**Vorschlag** (Pipeline §6.2): bei `raw` soll der Katalogpreis über den 130point-Verkäufen stehen.

**Gegenbeleg, `STATUS.md:336` (30.07.):**
> „Preis-Priorität **gedreht**: Ø letzte 3 eBay-Verkäufe (exakte Karte+Grade) SCHLÄGT PriceCharting (dünne Daten bei frischen JP-Slabs: Glurak CGC 10 = 51,61 € PC vs. **138,99 € echte Verkäufe**)"

Und das Produktversprechen auf dem Login-Schirm, `STATUS.md:432`:
> „Marktwert aus **echten eBay-Verkäufen**"

Dieselbe Zeile steht in der PWA (`/Users/smorty/sero-app/web/sero.js:509`): *„Nicht was verlangt wird — was wirklich bezahlt wurde."*

Der Entwurf begründet die Umkehr mit „ein Cardmarket-Trend ist ein Aggregat über hunderte Transaktionen" — das ist argumentativ vertretbar, aber es ist **eine Produktentscheidung, keine Bugfix-Folge**, und sie widerspricht dem, was auf dem Anmeldebildschirm steht.

**Sicherere Alternative:** Reihenfolge unangetastet lassen. Wenn E3 stabiler ist, gehört das in Wächter T2 (Quellen-Widerspruch) — Cardmarket als *Referenz* gegen einen Sold-Ausreißer, nicht als Gewinner.

---

### A5 🟠 `fits` als Filter in `research_price` bricht die Marktanzeige und den Telegram-Weg

**Vorschlag** (Pipeline §6.2): `research_price` bekommt `titel_filter`, der Katalog reicht `sold.fits` durch.

**Am Code:** `research_price` (`/Users/smorty/ebay-bot/bot/ebay/browse.py:19-76`) hat **einen** Rückgabewert, der drei Aufgaben erfüllt:
1. `eu_probe` im Katalog (`app_api.py:801`, `:1010`) — soll gefiltert werden ✅
2. `item["market"]` = die angezeigten Vergleichsangebote inkl. `samples` (`app_api.py:826-827`, `:969-970`)
3. `bot/main.py:571` — der Telegram-Preisvorschlag für **Allgemeinware**

`fits` (`sold.py:190-193`) verlangt, dass **jedes** Ziffern-Token der Query im Titel steht. Bei „Bosch GSR 18V-55 Professional Akkuschrauber 2x2.0Ah" sind das `18v-55`, `2x2.0ah` — kaum ein eBay-Titel enthält beide. Ergebnis: `count` fällt unter die Schwelle, `median` verschwindet, und der Telegram-Bot verliert seinen Preisanker für Nicht-Sammlerware.

**Sicherere Alternative:** Filter **nur** in der `eu_probe`-Lambda anwenden, nicht im Aufruf für `item["market"]`. Also zwei getrennte Aufrufe statt eines Parameters — kostet einen Browse-Call mehr, hält aber die Anzeige und den Bot heil. Und `fits` vorher gegen Nicht-Karten testen: `tests/test_sold_filter.py` (8 Tests) deckt heute nur Karten ab.

---

### A6 🟠 Der Monotonie-Sweeper schreibt in eine **global geteilte** Tabelle — im Autolauf, ohne Aufsicht

**Vorschlag** (Pipeline §8.2b): isotone Regression über `card_prices` im 12-h-Lauf `periodic_refresh` (`app_api.py:2384`), Abweichler mit `force=True` neu holen.

**Risiken am Code:**
- `card_prices` hat **keinen Mandanten** (`catalog.py:36-39`). Ein Fehler im Sweeper schlägt auf jeden Nutzer durch.
- `force=True` umgeht die TTL (`catalog.py:116`) → jede betroffene Zeile feuert erneut gegen 130point mit **8 s globaler Drossel** (`sold.py:37`). Bei 247 Zeilen heute wären das im Worst Case 33 Minuten Vollauslastung der einzigen Verkaufsquelle — währenddessen laufen alle Live-Scans in den 429-Cooldown (`sold.py:66`, 600 s).
- `periodic_refresh` hat **kein** Abbruchkriterium und **kein** Trockenlauf-Flag.
- Nach dem 02.08. (Datenverlust durch einen Agenten) ist ein automatischer Massenschreiber auf geteilte Daten die falsche Reihenfolge.

**Sicherere Alternative:** Sweeper als **separates Skript mit `--dry-run` als Default**, das erst nur loggt. Zusätzlich ein Deckel (`max 50 force-Refreshes pro Runde`). Erst wenn Sven das Log gesehen hat, scharf schalten.

---

## B — AM CODE NICHT UMSETZBAR / FALSCHE ANNAHMEN

### B1 ❌ „holder aus `detect_card` (vorhanden)" — ist im Hauptpfad **nicht** vorhanden

Der Pipeline-Entwurf schreibt in S2: *„(a) `holder` aus `detect_card` (vorhanden)"*.

**Tatsächliche Aufrufer von `detect_card`** (vollständige Suche):
```
web/cardscan.py:292   det = await detect_card(client, src_path)   ← innerhalb slab_recut()
```
Das ist der **einzige** Aufruf. `crop_photos` (`cardscan.py:266-284`) ruft nur `_cutout` (rembg), kein Vision. Und `slab_recut` läuft nur, wenn `listing["graded_info"]` bereits gesetzt ist **und** `photos_raw` existiert **und** der Cutout nicht Slab-Format hat (`app_api.py:908`, `:922`).

**Konsequenz:** S2 ist kein Umschalten, sondern **ein zusätzlicher Sonnet-5-Vision-Call pro Scan** — für jedes Stück, auch für die 90 %, die keine Slabs sind. Das ist der teuerste Einzelposten des ganzen Entwurfs (Rechnung in C3) und wird nirgends als Kosten ausgewiesen.

**Sicherere Alternative:** `holder` und `item_class` aus dem **bestehenden** `analyzer.analyze`-Call ziehen (das Modell sieht die Fotos ohnehin). Kein zweiter Call, keine zusätzlichen Bild-Tokens.

### B2 ❌ „`graded_info=null` erzwingen wenn `holder != slab`" — in `parse_listing_json` unmöglich

`parse_listing_json` (`bot/claude_client.py:115-132`) bekommt **nur den Antworttext**. Es hat keinen Zugriff auf Fotos, keinen auf `detect_card`. Es kann nur `data["holder"]` gegen `data["graded_info"]` prüfen — also die Selbstkonsistenz derselben Modellantwort. Das ist ein sinnvoller Check, aber es ist **keine** unabhängige Validierung, wie der Entwurf suggeriert („es fehlt die Code-Prüfung").

### B3 ❌ pHash / Token-Ähnlichkeit brauchen neue Abhängigkeiten — und `requirements.txt` ist schon jetzt unvollständig

Im venv geprüft:
```
cv2 5.0.0 · QRCodeDetector ✓ · barcode ✓ · rembg ✓ · numpy ✓
pyzbar FEHLT · rapidfuzz FEHLT · imagehash FEHLT
```
`requirements.txt` enthält **acht** Pakete: telegram, anthropic, httpx, Pillow, dotenv, pytest, pytest-asyncio, pillow-heif. **`opencv-python`, `rembg` und `numpy` stehen nicht drin**, obwohl `cardscan.py:159-176` sie importiert. Auf einem frischen Server fällt der Freisteller **stumm** aus, weil `cardscan.py:254` die `ImportError` abfängt.

Der QR-/Barcode-Kanal (P1) ist tatsächlich abhängigkeitsfrei — das stimmt. Das Bit-Scoring mit pHash (§5.3, Merkmal 5) ist es **nicht**.

**Sicherere Alternative:** Erst `requirements.txt` in Ordnung bringen (mit Versionspins), dann neue Pakete. Bildvergleich in P1 weglassen — sein Bit-Gewicht ist ohnehin 0, solange kein Katalogbild vorliegt.

### B4 ⚠️ Die DB-Zahlen der Vorberichte sind bereits veraltet — das Leck wächst *jetzt*

| Messung (heute, read-only) | Vorbericht | Jetzt |
|---|---|---|
| `card_prices` gesamt | 230 | **247** |
| davon `solo:` | 223 (97 %) | **239 (96,8 %)** |
| davon `pricecharting_weak` | 87 | **90** |
| `cards` gesamt | 282 | **302** |
| `cards` ohne `name` | 262 | **281** |
| `collection_items` | 15 | **13** (ein Konto) |

Zwei identische GTA-Vice-City-Items (`e34807d4`, `c9208739`) tragen zwei verschiedene `solo:`-Schlüssel und denselben Wert 97,52 € — der Beleg dafür, dass der geteilte Katalog für diese Warengruppe null Wirkung hat.

### B5 ⚠️ Der `_zwischenstufe`-Vorschlag dreht einen Fix von *gestern* zurück

Der Entwurf (§6.3, P0-4) will Zwischennoten „als Spanne ausweisen statt zu interpolieren, und **abrunden** statt zu interpolieren". `STATUS.md:542` (03.08., also heute):

> „FIX: neue Funktion `_zwischenstufe()` ordnet zwischen zwei bekannten Stützstellen linear ein (9.0=100 → 9.2=140 → 9.4=180 → 9.5=200 USD). Nur wenn BEIDE Nachbarwerte vorliegen und plausibel sortiert sind — sonst bleibt es beim Feldwert, **lieber grob und belegt als fein und geraten**. Test prüft die Monotonie über acht Stufen."

Der Code (`pricecharting.py:69-70`) macht bereits genau das Konservative, was der Entwurf fordert. **Kein Handlungsbedarf** — außer der Kennzeichnung im Label, und die ist additiv.

### B6 ℹ️ Der `condition-18`-Doku-Fehler ist real, aber folgenlos

`pricecharting.py:6` behauptet `condition-17/18 = BGS 10 / CGC 10 / CGC 9.5`. `_grade_fields` (`:21-46`) nutzt `condition-18-price` **nirgends**. Reiner Kommentarfehler, kein Bug, keine Regressionsgefahr — aber auch keine Dringlichkeit.

### B7 ✅ Bestätigt: Wächter P3 ist tatsächlich tot

`catalog.py:212` greift nur bei `n_avg < 2`. `fetch_sold` liefert erst ab `len(fresh) >= 2` bzw. `len(sales) >= 2` überhaupt ein Ergebnis (`sold.py:281`, `:288`) — `n_avg` ist also nie < 2. Der Wächter kann nicht feuern. Das stimmt und ist ein echter, risikoarmer Fix.

---

## C — WAS DIE PIPELINE REAL KOSTET

### C1 Der harte Deckel ist **nicht** 130point, sondern der Freisteller

Alle Vorberichte nennen 10.800 Scans/Tag (130point, `MIN_GAP=8.0`). Der bindende Engpass liegt zwei Größenordnungen früher:

| Engpass | Ort | Deckel/Tag |
|---|---|---|
| **rembg-Semaphore(1)**, ~18 s/Foto × 2 Fotos = 36 s **exklusiv, serverweit** | `cardscan.py:263`, `:274-282` | **≈ 2.400** |
| 2 Scan-Worker × gemessen ~75 s/Scan | `app_api.py:2700-2701`, `SCAN_TIMEOUT=900` `:2448` | **≈ 2.300** |
| 130point-Drossel 8 s | `sold.py:37` | 10.800 |
| PriceCharting 1 Call/s | ToS | 86.400 |

**Gemessene `scan_seconds` aus der Produktions-DB:** 38,4 / 55,5 / 57,4 / 95,8 / **138,8** s.

Die 138,8-Sekunden-Zeile ist bemerkenswert: `SCAN_TIMEOUT` ist 900 s, aber vier gleichzeitige Uploads laufen bereits heute in die Freisteller-Warteschlange (`STATUS.md`-Kommentar bei `app_api.py:2444-2447` dokumentiert genau das). **Zehntausende Nutzer sind mit dieser Architektur um Faktor 10–100 zu viel** — und zwar unabhängig von jeder Preisquelle.

### C2 Claude-Kosten pro Scan, heute

Konfigurierte Modelle: `claude-sonnet-4-6` (`bot/config.py:113`), `claude-haiku-4-5-20251001` (`web/prices.py:24`), `claude-sonnet-5` (`web/cardscan.py:34`).

| Call | Modell | Input | Output | Kosten |
|---|---|---|---|---|
| `analyzer.analyze` (2 Fotos) | Sonnet 4.6 ($3/$15 per MTok) | ~2 × 2.460 Bild + ~1.900 Prompt ≈ 6.900 | ~1.200 | ≈ **$0,039** |
| `identify_card` | Haiku 4.5 ($1/$5) | ~600 | ~300 | ≈ **$0,002** |
| `slab_recut` → `detect_card` (nur Slabs, 2 Fotos parallel) | Sonnet 5 ($2/$10 Intro bis 31.08.26) | 2 × ~1.960 | 2 × ~600 | ≈ **$0,020** |

**Heute: ≈ $0,041 pro Raw-Scan, ≈ $0,061 pro Slab-Scan.** Bei 10.000 Scans/Tag: **410–610 $/Tag**, also 150.000–220.000 $/Jahr — vor jeder Prompt-Verlängerung.

**Nebenbefund:** `CLAUDE_MAX_EDGE = 2048` (`claude_client.py:27`) mit Kommentar „Opus 4.7+ liest hochauflösend". Das konfigurierte Modell ist aber `claude-sonnet-4-6`, das bei **1568 px** deckelt — die Extrapixel werden serverseitig weggerechnet. Kostet nichts extra, aber Encoding-Zeit und Bandbreite. Umgekehrt: `cardscan._b64` skaliert auf **1400 px** (`:38`) und schickt das an `claude-sonnet-5`, das 2576 px könnte. **Der Label-Crop-Vorschlag (P1) läuft heute also gegen ein bereits heruntergerechnetes Bild** — der Gewinn wäre größer als der Entwurf annimmt, aber nur, wenn `max_edge` in `_b64` mitgezogen wird.

### C3 Was der Entwurf **zusätzlich** kostet

| Neuer Posten | Zusatz/Scan |
|---|---|
| S2-Klassifizierung als eigener Vision-Call (B1) | **+$0,020** |
| S3 LLM-Ablesung auf Label-Crop | +$0,010 (nur Slabs) |
| Längerer System-Prompt (~1.900 → ~4.400 Tokens) | +$0,0075 · **mit Caching +$0,0008** |
| `/api/products` + `/api/product` statt 1 Call | 2× PriceCharting-Kontingent |
| 2 Browse-Calls statt 1 (A5-Alternative) | eBay-Kontingent |

**≈ +$0,03/Scan ⇒ ≈ +300 $/Tag bei 10.000 Scans.** Die Pipeline wird also **~70 % teurer**, um Fehl-Matches zu verhindern — was vertretbar sein kann, aber es steht in keinem der Vorberichte.

### C4 Prompt-Caching: rechnet sich, aber weniger als erhofft

`_call` (`claude_client.py:253-259`) übergibt `system=SYSTEM_PROMPT` als reinen String. Der Wechsel auf die Blockform mit `cache_control` ist korrekt vorgeschlagen. Zahlen:

- Minimale cachebare Präfixlänge bei **Sonnet 4.6: 1024 Tokens**. Der heutige Prompt (~5.500 Zeichen Deutsch ≈ 1.700–2.000 Tokens) liegt knapp darüber; der neue (~4.400) deutlich. ✅ Die Behauptung des Schema-Berichts stimmt.
- Cache-Write 1,25×, Read 0,1×. **Break-even bei 5-Minuten-TTL: zwei Requests.** Bei 2 Workern × 75 s liegen Scans ~37 s auseinander → TTL hält. ✅
- **Aber:** Renderreihenfolge ist `tools → system → messages`. Die Bilder stehen in `messages`, also **nach** dem Breakpoint. Gecacht werden nur ~1.900 von ~6.900 Input-Tokens → Ersparnis ≈ **$0,005/Scan**, nicht die oft zitierten 90 %.
- **Voraussetzung:** Der Prompt muss **byte-identisch** bleiben. Kein Datum, keine Item-ID, kein `f"…{version}…"` hineininterpolieren. Kontrolle über `resp.usage.cache_read_input_tokens` — das Log bei `:261-262` müsste dafür erweitert werden.

---

## D — WIDERSPRÜCHE ZU SVENS VERBINDLICHEN REGELN

| Regel | Beleg | Verstoß im Entwurf |
|---|---|---|
| „Marktwert aus **echten eBay-Verkäufen**" (Login-Versprechen) | `STATUS.md:432`, `sero.js:509` | §6.2 stellt Cardmarket über Sold (A4) |
| Sold **schlägt** PriceCharting — ausdrücklicher Entscheid 30.07. | `STATUS.md:336` | dito |
| „lieber grob und belegt als fein und geraten" | `STATUS.md:542` | §6.3 will die Interpolation zurückdrehen (B5) |
| **Eigenes Foto ist überall das Hauptbild** | `STATUS.md:163`, `:167` | S1.5 `label_crop`: Wenn der Crop in `item["photos"]` landet (wie `crop_photos` bei `app_api.py:895`), wird er über `main_image_index`/`reorder_photos` (`bot/main.py:269`) zum eBay-Hauptbild. **Muss ein separates Feld sein.** |
| Keine Drag-Sortierung, „Lehre" vom 02.08. | `STATUS.md:437` | nicht berührt ✅ |
| Leitbegriff **Stück**, **listen**, **Marktwert**, **tippen** | `STATUS.md:386` | Der neue Prompt (7.4) setzt das korrekt um ✅ |
| „tagesaktuelle Marktpreise", bewusst **nicht** „Echtzeit" | `STATUS.md:218` | nicht berührt ✅ |
| Prüf-Agenten nie mit Schreibrechten (02.08.) | Memory | A6: Sweeper schreibt automatisch in geteilte Daten |

---

## E — WAS KOMPLETT FEHLT

1. **Es gibt keinen einzigen Test für `catalog.py`.** `grep -rn "catalog\|refresh_price\|card_key" tests/*.py` → **0 Treffer**. 118 Tests im Repo, davon 8 für `sold.fits`, 10 für Pricing, 11 für USK — und **null** für das Modul, das den globalen, über alle Nutzer geteilten Preis schreibt. Das ist die größte Regressionslücke im Projekt, und keiner der fünf Berichte erwähnt sie.
2. **Kein Migrationsplan für die 239 `solo:`-Zeilen und 281 namenlosen `cards`.** Der Entwurf nennt es als „P3-15, getrennter Lauf" — ohne Backup-Schritt, ohne Trockenlauf, ohne Rollback. `backup.sh` existiert im Repo; es wird nirgends referenziert.
3. **Kein Feature-Flag.** `ebay_insights.py:28` zeigt das gute Muster (`SERO_EBAY_INSIGHTS=1`). Ein Umbau dieser Größe braucht `SERO_PIPELINE_V2=1` mit Rückfall auf den alten Pfad — sonst ist jeder Fehler ein Produktionsausfall.
4. **Die beiden Dokumente widersprechen sich beim Feldort.** Pipeline §0.1 legt `item_class`, `holder`, `price_state`, `price_reason`, `price_evidence`, `price_low/high`, `identity_conf` **top-level** an. Der Schema-Bericht Regel 3 legt sie in `analysis`. Top-level bedeutet: **jedes dieser Felder muss in `PREIS_FELDER`/`ANALYSE_FELDER` eingetragen werden** (`app_api.py:725-733`), sonst verwirft `col_save_analyse` (`:744-746`) sie beim nächsten Lauf **stillschweigend**. Der Schema-Bericht nennt nur `valuation` und `schema_version`. `label_crop_path` und `bild_qualitaet` aus S1 fehlen in beiden Listen.
5. **PWA-Nutzlast nicht bedacht.** `LS.set` (`sero.js:655`) schluckt Quota-Fehler still (`catch { /* voll */ }`). iOS-Safari deckelt localStorage bei ~5 MB. `valuation.considered` + `entity_match.features` + `plausibility.checks` sind pro Stück leicht 2–4 KB. Bei 1.000 Stücken ist der Sammlungs-Cache tot — **ohne Fehlermeldung**, die App wird nur wieder langsam. `_DETAIL_ONLY` (`app_api.py:566`) muss `valuation` mitnehmen.
6. **Kein Wort zur Mandantentrennung.** `tests/test_multitenant.py` (7 Tests) sichert sie heute. `card_prices` ist bewusst global — ein `price_reason` mit Nutzer-spezifischem Text darf dort **nicht** hinein.
7. **Kein Rollback-Pfad für `pricecharting_weak`.** 90 Zeilen stehen heute darauf. Wenn `weak` abgeschafft wird (P0-3), was passiert mit den bestehenden Zeilen? Sie behalten `source="pricecharting_weak"` — und `sero.js:513` hat dafür einen Anzeigetext. Weder Migration noch Abwärtskompatibilität sind beschrieben.

---

## F — DIE VIER VORSCHLÄGE, DIE ICH OHNE VORBEHALT UNTERSCHREIBE

Risikoarm, sofort belegbar, kein geteilter Zustand berührt:

1. **`setdefault` → explizite Zuweisung** in `app_api.py:790` und `:1000`. Zwei Zeilen. Behebt 96,8 % der Katalog-Zeilen. Vorher: `pytest` komplett, nachher: eine Zeile in `cards` prüfen.
2. **Domänen-Gate über `console-name`** in `pricecharting.py` — verwerfen statt bepreisen. Direkter Fix des 603-€-Falls, berührt keinen anderen Pfad.
3. **Grader-Kanon zentralisieren** (`beckett` → nicht blind `BGS`) und `grade_bucket` normalisieren. `"BECKETT 9.0"` / `"BGS 9"` / `"BECKETT 9"` sind heute drei Buckets.
4. **Die Regex in `claude_client.py:158-159` entschärfen.** `\b[A-Za-z]{2,5}(?=\s*\d{1,2}(?:\.\d)?\b)` ersetzt das erste 2–5-Buchstaben-Wort vor einer Zahl — bei „One Piece Band 9 Beckett 9.0" trifft es **`Band`**. Das ist ein aktiver Bug, kein Konzeptproblem, und er trifft genau die wertvollen Stücke (die Funktion läuft nur bei gesetztem `graded_info`).

**Reihenfolge, die ich empfehle:** 1 → Tests für `catalog.py` schreiben → 4 → 2 → 3. Erst danach über neue Module (`web/slab.py`, `web/match.py`) reden.

---

### Gelesene Dateien (nur gelesen, nichts geändert)
`/Users/smorty/ebay-bot/web/catalog.py` · `/Users/smorty/ebay-bot/web/app_api.py` · `/Users/smorty/ebay-bot/web/cardscan.py` · `/Users/smorty/ebay-bot/web/pricecharting.py` · `/Users/smorty/ebay-bot/web/sold.py` · `/Users/smorty/ebay-bot/web/prices.py` · `/Users/smorty/ebay-bot/web/ebay_insights.py` · `/Users/smorty/ebay-bot/bot/claude_client.py` · `/Users/smorty/ebay-bot/bot/main.py` · `/Users/smorty/ebay-bot/bot/config.py` · `/Users/smorty/ebay-bot/bot/ebay/browse.py` · `/Users/smorty/ebay-bot/bot/ebay/inventory.py` · `/Users/smorty/ebay-bot/requirements.txt` · `/Users/smorty/ebay-bot/STATUS.md` · `/Users/smorty/sero-app/web/sero.js` · `/Users/smorty/ebay-bot/data.db` (read-only)

### Quellen für die Preis- und Token-Angaben
Modell-Preise und Vision-Token-Regeln aus der geladenen `claude-api`-Skill (Modelltabelle Stand 2026-06-24: Sonnet 4.6 $3/$15, Sonnet 5 $3/$15 mit Intro $2/$10 bis 2026-08-31, Haiku 4.5 $1/$5; Cache-Minimum Sonnet 4.6 = 1024 Tokens, Write 1,25× / Read 0,1×; Bild-Deckel Sonnet 4.6 = 1568 px, Sonnet 5 = 2576 px). Die Token-Schätzungen pro Bild sind gerechnet, nicht gemessen — **vor einer Budget-Entscheidung mit `client.messages.count_tokens()` gegen echte Scan-Fotos nachrechnen.**