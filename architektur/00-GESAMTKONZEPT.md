# SERO — Gesamtkonzept Preis-Pipeline

*Lead-Architect-Entwurf, 03.08.2026. Strikt read-only erarbeitet: nur Dateien gelesen, `data.db` ausschließlich über `file:data.db?mode=ro`. Keine Datei geändert, kein Server gestartet, kein Request gegen localhost.*

---

## Entscheidung vorweg — die vier Sätze, auf die es ankommt

1. **Es gibt 2026 keine eBay-API für verkaufte Artikel, die du bekommen kannst.** Dein Antrag ist bereits abgelehnt, das steht wörtlich in deinem eigenen Code (`web/ebay_insights.py:1-13`). Plane nicht damit.
2. **Der teuerste Fehler liegt nicht bei den Quellen, sondern beim Schlüssel.** `239 von 247` Katalogzeilen liegen unter einem Wegwerf-Schlüssel `solo:<uuid>` — gemessen heute. Ursache sind zwei Zeilen: `app_api.py:790` und `:1000`. Solange die stehen, wirkt keine andere Verbesserung.
3. **Der 603-€-Fehler braucht genau ein Gate**, nicht eine neue Architektur: PriceCharting trägt die Warenart im Feld `console-name`. Ein Buch liegt dort unter `Comic Books One Piece`, die Karte unter `One Piece Romance Dawn`. Ein Vergleich, drei Zeilen Code.
4. **`est_value` bleibt in jedem Fall gesetzt.** Ein Vorentwurf wollte bei unsicherem Wert `None` schreiben. Das hätte exakt den 307,90-€-Fehler vom 02.08. wieder eingebaut, weil die Listing-Pipeline dann blind neu recherchiert. Ehrlichkeit gehört in ein neues Feld `price_state`, nicht in ein gelöschtes altes.

---

# Teil 1 — Rechercheergebnisse

## 1.1 Die wichtigste Erkenntnis: eBay liefert keine Verkaufsdaten

**Deine Annahme „eBay-API als Primärquelle" ist nicht umsetzbar.** Belegt auf drei Ebenen:

| Weg | Status | Beleg |
|---|---|---|
| `findCompletedItems` (Finding API) | Zugang am **15.10.2020** gesperrt, Finding + Shopping API am 04.01.2024 deprecated, am **05.02.2025 abgeschaltet** | [eBay API Deprecation Status](https://developer.ebay.com/develop/get-started/api-deprecation-status), [eBay Community Alert](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062) |
| **Marketplace Insights API** | Limited Release, „restricted and not open to new users at this time", nur „select developers approved by business units" | [Marketplace Insights Overview](https://edp.ebay.com/api-docs/buy/marketplace-insights/static/overview.html), [Community: Zugang verweigert](https://community.ebay.com/t5/eBay-APIs-Talk-to-your-fellow/Marketplace-Insights-API-access/td-p/34838736) |
| **Dein Antrag** | **abgelehnt**, Wortlaut Developer Support: Zugang „highly limited and generally reserved for eBay's approved partners only"; Scope `buy.marketplace.insights` nicht gewährt, schon die Token-Anfrage scheitert mit `invalid_scope` | `/Users/smorty/ebay-bot/web/ebay_insights.py:1-13` |

Ich habe den Status heute erneut gegengeprüft — unverändert. Der einzige dokumentierte Weg zu restricted APIs ist der **Application Growth Check** ([Doku](https://developer.ebay.com/api-docs/static/gs_use-the-application-growth.html)); er setzt echtes Produktionsvolumen und einen Business-Case voraus. Das ist ein Weg für später, kein Fundament für den Launch.

**Was es sonst im offiziellen Programm gibt — alles aktive Angebote, keine Verkäufe:**

| API | Liefert | Zugang | Verkäufe? |
|---|---|---|---|
| **Browse API** | aktive Angebote, Item-Details | offen, nutzt du bereits (`bot/ebay/browse.py`) | nein |
| **Feed API** (Buy) | `NEWLY_LISTED` täglich, `ALL_ACTIVE` wöchentlich | Limited Release, EPN-Konto + Vertrag | nein — die Scope-Namen sagen *active* ([Feed API Overview](https://developer.ebay.com/api-docs/buy/feed/static/overview.html)) |
| **Terapeak** | 3 Jahre Verkaufsstatistik | nur Seller-Hub-Oberfläche | ja, aber **keine API** |
| **Sell APIs** (Fulfillment/Analytics) | **deine eigenen** Verkäufe | OAuth mit Seller-Scope, heute 403 | ja — und genau das ist die Chance, siehe E1 |

## 1.2 Was das für die Quellen-Hierarchie bedeutet

Statt nach Anbieternamen wird nach **Belegklasse** sortiert. Das ist die Ordnung, die real verfügbar ist:

| Klasse | Bedeutung | Quelle im Code | Lage 08/2026 |
|---|---|---|---|
| **E1** | eigene, tatsächlich verkaufte Stücke | eBay Sell-APIs auf SERO-erzeugte Listings | **noch nicht** — Token ohne Fulfillment-Scope (403). Der einzige Burggraben: first-party, vertraglich sauber, wird mit jedem Nutzer besser |
| **E2** | fremde, tatsächlich verkaufte Stücke | `web/sold.py` (130point), `web/ebay_insights.py` (aus) | vorhanden, aber ohne Vertrag; harte Grenze ~10.800 Anfragen/Tag durch `MIN_GAP = 8.0` (`sold.py:37`, serverweiter Lock) |
| **E3** | lizenzierter Katalogpreis (Aggregat über viele echte Verkäufe) | TCGdex/Cardmarket, Scryfall, YGOPRODeck, tcgcsv (`web/prices.py`) | sauber, deterministisch — **nur für rohe Einzelkarten** |
| **E4** | Angebotspreise (was gefordert wird) | eBay Browse (`bot/ebay/browse.py`) | offiziell lizenziert, skaliert, heute ungefiltert |
| **E5** | Sekundärkatalog | PriceCharting (`web/pricecharting.py`) | ToS-Konflikt, siehe 1.3 |
| **E6** | KI-Schätzung | `analysis.estimated_price_range_eur` | nie ein belegter Marktwert, aber legitimer letzter Anker für den Listing-Preis |

**Die Reihenfolge der Kaskade bleibt, wie du sie am 30.07. entschieden hast:** echte Verkäufe schlagen alles. Ein Vorentwurf wollte Cardmarket (E3) über die Verkäufe (E2) stellen. Das widerspricht deinem eigenen Beleg (`STATUS.md:336`: Glurak CGC 10 = 51,61 € PriceCharting gegen **138,99 € echte Verkäufe**) und dem Satz auf deinem Anmeldeschirm — „Marktwert aus echten eBay-Verkäufen" (`STATUS.md:432`, `sero.js:509`). **Verworfen.** E3 wird stattdessen als *Referenz* in den Wächtern eingesetzt, wo sie sachlich hingehört.

## 1.3 Die zwei Rechtsrisiken — vor dem Launch zu klären

**PriceCharting untersagt genau deinen Nutzungsfall.** Die [Terms of Service](https://www.pricecharting.com/page/terms-of-service) beschränken auf „Internal Business Purposes" und verbieten Preisdaten „in any software, application, or system that is accessible to third parties … without express written permission". SERO ist per Definition der ausgeschlossene Fall. Dazu 1 Call/Sekunde für das gesamte Konto ([API-Doku](https://www.pricecharting.com/api-documentation)) — ein Einzelnutzer-Produkt.
**Handlung:** Lizenzanfrage für Weiterverbreitung an PriceCharting, bevor du implementierst. Billigster Fix im ganzen Papier. Die Antwort entscheidet, ob E5 im Baum bleibt.

**130point ist gescraptes Scraping.** `POST https://back.130point.com/sales/` (`sold.py:50-53`) ist ein undokumentierter Backend-Endpunkt, die Antwort wird per Regex geparst (`sold.py:31`), und `sold.py:28-29` fälscht einen Browser-User-Agent, weil die Seite auf nicht-Browser-Requests mit 403 antwortet. Das eBay User Agreement in der Fassung wirksam **20.02.2026** untersagt automatisiertes Abgreifen ohne Erlaubnis ausdrücklich und nennt LLM-gesteuerte Bots beim Namen ([eBay User Agreement](https://www.ebay.de/help/policies/member-behaviour-policies/user-agreement?id=4259), [Value Added Resource](https://www.valueaddedresource.net/ebay-bans-ai-agents-updates-arbitration-user-agreement-feb-2026/)). Du bist gewerblicher Händler — das Risiko trifft nicht ein Feature, sondern dein Konto.
**Handlung:** E2 bleibt vorerst, wird aber als Übergangsquelle behandelt. Der Baum ist so gebaut, dass er ohne E2 und ohne E5 funktioniert. Rechtsberatung vor dem Launch mit zehntausenden Nutzern ist marginal teuer gegen das Risiko; ich bin kein Anwalt.

## 1.4 Slab-Erkennung — was maschinell wirklich geht

- **Genau ein Grader hat eine dokumentierte öffentliche API: PSA.** `GET https://api.psacard.com/publicapi/cert/GetByCertNumber/{certNumber}`, Bearer-Token, **Free Tier 100 Calls/Tag** ([PSA Public API](https://www.psacard.com/publicapi/documentation)). Beckett, CGC, SGC, WATA, VGA bieten nur Web-Formulare; deren Lookup-Seiten antworten auf automatisierte Requests mit 403 — kein Produktionspfad.
- **QR- und Barcode schlagen OCR.** PSA (QR seit 2020), CGC und SGC drucken QR-Codes, PSA und BGS codieren die Cert-Nummer zusätzlich im Barcode. Im venv `/Users/smorty/ebay-bot/.venv` sind `cv2.QRCodeDetector` **und** `cv2.barcode.BarcodeDetector` vorhanden (OpenCV 5.0.0), und `cv2` wird von `cardscan.py` ohnehin importiert. **Dieser Kanal kostet keine neue Abhängigkeit.**
- **Kürzel-Tabelle ist unvollständig und an einer Stelle falsch.** `_GRADER_KUERZEL` (`bot/claude_client.py:139-142`) kennt `beckett→BGS, bgs, psa, cgc, sgc, wata, vga, cga, ace`. Es fehlen **BVG** (Vintage, ohne Subgrades), **BCCG** (eigene, 2–3 Stufen mildere Skala), **BAS** (nur Autogramme), CBCS, CSG, TAG, MANA. `beckett → BGS` ist bei einem BVG- oder BCCG-Slab schlicht falsch — ein als BGS 10 bepreistes BCCG 10 kostet dich Geld.
- **Skalen sind nicht ineinander umrechenbar.** BGS 9.5 entspricht sachlich etwa PSA 10, PSA 10 erzielt trotzdem mehr (Markenprämie). **CGC Cards hat kein 9.8** — 9.8 ist CGC *Comics*. WATA vergibt zwei Achsen: Box-Note **und** Seal Rating C…A++.
- **Kommerzieller Ausweg für Nicht-PSA:** Ximilar `/v2/slab_id` liefert Company (PSA, BECKETT, CGC, SGC, ACE, MANA, TAG), Grade, Zertifikatsnummer und ein Tag `Graded: yes/no`, deckt auch Comics/Manga ab; Business ab 59 €/Monat ([Doku](https://docs.ximilar.com/collectibles/recognition), [Preise](https://www.ximilar.com/pricing/)). Optional, nicht Stufe 1.

## 1.5 PriceCharting — die Fakten, die den Fehler erklären

- **Es gibt kein Relevanz- oder Score-Feld.** Die Doku sagt zur Suche nur, dass bei mehreren Treffern „only the best matches are returned". `lookup_pc` (`web/pricecharting.py:87-92`) ruft `GET /api/product?q=<Freitext>` und prüft **nur `prod["id"]` auf Existenz**. Ein Buch und eine Sammelkarte sind für diese Funktion ununterscheidbar.
- **`/api/products?q=` liefert bis zu 20 Kandidaten** mit `id`, `product-name`, `console-name`. Das ist der einzige Weg, Mehrdeutigkeit überhaupt zu sehen.
- **`console-name` trägt die Warenart.** Das Buch liegt unter „Comic Books One Piece" ([Kategorie-Seite](https://www.pricecharting.com/console/comic-books-one-piece)), die Karten unter TCG-Set-Namen. Das richtige Produkt „One Piece Vol. 103 [Paperback]" existiert dort sogar.
- **Die Preisfelder bedeuten je Warenart etwas anderes** — das ist der zweite, bisher unentdeckte Teil deines Manga-Bugs:

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

`_grade_fields` (`pricecharting.py:21-46`) kennt nur die Karten- und die Videospiel-Leiter. Eine Beckett 9.0 auf einem Buch landet auf `graded-price` — bei Comics bedeutet das 8.0/8.5. Der Docstring `pricecharting.py:6` ist zusätzlich falsch (`condition-18` ist SGC 10, nicht CGC 9.5); folgenlos, weil das Feld nirgends genutzt wird.
- **Deine Interpolation ist richtig und bleibt.** `_zwischenstufe` (`pricecharting.py:49-73`) ordnet zwischen zwei bekannten Stützstellen linear ein und **nur dann**, wenn beide vorliegen und plausibel sortiert sind (`:69-70`). Ein Vorentwurf wollte das zurückdrehen und stattdessen abrunden. Das widerspricht deinem eigenen Grundsatz „lieber grob und belegt als fein und geraten" (`STATUS.md:542`) und wird **nicht umgesetzt** — ergänzt wird nur eine Kennzeichnung im Label.
- **Der Cache-Schlüssel ist zu grob.** `pricecharting.py:81` cached unter `sha1(query)`. Ein falsch gematchtes Produkt klebt 12 h an der Anfrage und ist nicht gezielt invalidierbar.

## 1.6 Was der Code heute wirklich tut — gemessen, nicht vermutet

**Kaskade in `catalog.refresh_price` (`catalog.py:184-207`), erste zutreffende Zeile gewinnt:**

```
1. sold vorhanden                            → ebay_sold
2. sonst eu und nicht (pc_trusted und eu > pc*4)  → ebay_eu
3. sonst pc und pc_trusted                   → pricecharting
4. sonst base und grade == "raw"             → base.source
5. sonst pc unbelegt                         → pricecharting_weak
6. sonst alte Zeile / None
```

**Produktionsbestand heute (read-only gemessen, `file:data.db?mode=ro`):**

| Messung | Wert |
|---|---|
| `card_prices` gesamt | **247** |
| davon `solo:<uuid>` | **239 (96,8 %)** |
| davon `source = pricecharting_weak` | **90 (36 %)** |
| `cards` gesamt | **302** |
| `cards` ohne `name` | **281** |
| `collection_items` | **13** |

302 Katalogzeilen für 13 Stücke. Der geteilte Katalog trifft seinen eigenen Cache nie.

**Die vier Bruchstellen, alle selbst nachgelesen:**

1. **Der Wegwerf-Schlüssel.** `card_key_of` (`catalog.py:48-53`) erzeugt `"solo:" + uuid4()`, wenn weder `ref_id` noch `name` da sind. `identify_card` liefert für Nicht-Einzelkarten immer ein Dict **mit** Schlüssel `"name"` und Wert `None` — worauf `card_ref.setdefault("name", item.get("name"))` (`app_api.py:790`, identisch `:1000`) **nichts** tut, weil `setdefault` nur bei *fehlendem* Schlüssel greift. Bei jedem Lauf ein neuer Schlüssel.
2. **Die Wächter sind stumm.** Der `weak`-Wächter (`catalog.py:231-240`) braucht `base`; `base` wird in `refresh_item_price` nur gesetzt, wenn ein Karten-DB-Treffer vorlag (`:795-797`). Nicht-Einzelkarten haben nie einen — genau die Warengruppe, für die PriceCharting am unzuverlässigsten ist, ist ungeschützt. Der Ausreißer-Wächter (`catalog.py:212`) greift nur bei `n_avg < 2`, aber `fetch_sold` liefert erst ab 2 Verkäufen überhaupt ein Ergebnis (`sold.py:281`, `:288`) — **er kann nicht feuern**.
3. **Der EU-Zweig ist ungefiltert.** `research_price` (`bot/ebay/browse.py:19-76`) hat IQR-Trimming, aber keine Titelprüfung. Über `eu_probe` (`catalog.py:139-147`, −12 %) wird dieser Median zur zweitstärksten Stufe, und weil der Katalog `research_price` direkt aufruft (`app_api.py:801`, `:1010`), umgeht er `check_price_plausibility` (`bot/main.py:238`) komplett. In Produktion sichtbar: `pokemon:sv03.5-199 / CGC 10 → 803,99 € aus ebay_eu`.
4. **Korrekturen vergiften den Katalog.** Wächter C (`app_api.py:833-845`) korrigiert nur das Item; die Katalogzeile bleibt falsch stehen. Und er fehlt in `analyze_collection_item` — beim allerersten Scan, dem einzigen, den der Nutzer wirklich sieht, greift er nicht.

**Zwei aktive Bugs, die niemand bestellt hat:**

- `bot/claude_client.py:158-159` ersetzt per `re.sub(r"\b[A-Za-z]{2,5}(?=\s*\d{1,2}(?:\.\d)?\b)", richtig, titel, count=1)` das **erste** 2–5-Buchstaben-Wort vor einer Zahl. Bei „One Piece Band 9 Beckett 9.0" trifft es **`Band`** → „One Piece BGS 9 Beckett 9.0". Der Titel wird unauffindbar, und `sold.fits` verliert seinen wichtigsten Sicherungsstift. Die Funktion läuft nur bei gesetztem `graded_info` — also genau bei den wertvollen Stücken.
- `grade_bucket` (`catalog.py:81-84`) nimmt `graded["grader"]` roh. Im Bestand steht deshalb `"BECKETT 9.0"`, während `sold.py:111` intern `beckett → bgs` normalisiert. `"BECKETT 9"`, `"BECKETT 9.0"`, `"BGS 9"` sind drei Buckets für dasselbe Stück.

**Der echte Skalierungs-Deckel ist nicht 130point.** Der bindende Engpass ist der Freisteller: `rembg` läuft unter `Semaphore(1)` (`cardscan.py:263`), etwa 18 s je Foto, zwei Fotos je Stück, serverweit exklusiv. Das sind rund **2.400 Scans/Tag**, nicht 10.800. Gemessene `scan_seconds` aus der DB: 38,4 / 55,5 / 57,4 / 95,8 / 138,8. Zehntausende Nutzer sind mit dieser Architektur um Faktor 10–100 zu viel — unabhängig von jeder Preisquelle. Das ist ein eigenes Projekt (Worker-Pool, GPU-Instanz oder Fremd-Segmentierung) und steht bewusst nicht in diesem Plan.

**Es gibt keinen einzigen Test für `catalog.py`.** 15 Testdateien im Repo, darunter `test_sold_filter.py`, `test_pricing.py`, `test_usk.py`, `test_multitenant.py` — und null für das Modul, das den global geteilten Preis schreibt. Das ist die größte Regressionslücke im Projekt.

## 1.7 Ausdrücklich unbestätigt

PSA-Cert-Nummernlänge (nur Drittquellen, 8–10 Ziffern) · Semantik der PSA-Labelfarben · Existenz von Partner-APIs bei CGC, Beckett, SGC, WATA, VGA (weder bestätigt noch widerlegt — müsste direkt angefragt werden) · Preise der PSA-Paid-Tiers · exakte Ximilar-Credit-Kosten je `slab_id`-Call · ob `/api/products` dasselbe Ranking wie die Website-Suche nutzt · VGA/CGA-Seriennummernformat. Die Bit-Gewichte in Teil 2 sind **begründete Startwerte, keine gemessenen**.

---

# Teil 2 — Die Pipeline

## 2.0 Leitprinzipien

1. **Identität vor Preis.** Kein Preis wird gezogen, bevor feststeht, was das Stück ist.
2. **Jede Stufe hat ein Gate** mit drei Ausgängen: weiter, degradiert weiter, verworfen mit Grund. Kein vierter Ausgang.
3. **Ehrlichkeit ist ein neues Feld, kein gelöschtes altes.** `price_state` sagt, wie belastbar die Zahl ist. `est_value` bleibt gesetzt, damit die Listing-Kette funktioniert.
4. **Der Katalog ist die Wahrheit.** Jede Korrektur wird auch in `card_prices` geschrieben.
5. **Kein zusätzlicher Vision-Call.** Warenart und Verpackung liefert der bestehende `analyzer.analyze`-Aufruf mit — das Modell sieht die Fotos ohnehin.

## 2.1 Der Baum

```
FOTO(S)
  │
  ▼
S1  AUFNAHME                                    web/cardscan.py
    exif_transpose · HEIC · group_photos · crop_photos (rembg) · slab_recut
    NEU: label_crop() → eigenes Feld label_crop_path, NIE in item["photos"]
  │
  ▼
S2  KLASSIFIZIERUNG                bot/claude_client.py (im bestehenden Call)
    item_class ∈ {tcg_card, sport_card, comic_manga, video_game,
                  sealed_product, coin, toy, electronics, fashion, home,
                  tool, other}
    holder    ∈ {slab, sleeve, raw, sealed, none}
  │
  ├─ item_class == other  →  KEIN Abbruch. Klassen-gebundene Quellen (E5-Leiter)
  │                          entfallen, E4 + E6 tragen weiter. Allgemeinware
  │                          läuft wie heute.
  │
  ├─ holder == slab ──────────────────────┐
  │                                        ▼
  │   S3  SLAB-LABEL                    web/slab.py (neu)
  │       3.1 QR decodieren   (cv2.QRCodeDetector)        Kanal A
  │       3.2 Barcode decodieren (cv2.barcode)            Kanal A
  │       3.3 LLM-Ablesung aus dem Vision-Call            Kanal B
  │       3.4 Grader kanonisieren aus dem LABELTYP
  │       3.5 Note gegen die Skala DES GRADERS prüfen
  │       3.6 Cert-Format-Gate, optional PSA-Lookup
  │     │
  │     ├─ Note nicht auf der Skala   → graded = null, raw-Pfad,
  │     │                               Flag GRADE_UNGUELTIG
  │     ├─ Grader unentscheidbar      → graded = null (PSA 9 ≠ CGC 9)
  │     └─ ok → C_grade ∈ {1.0 | 0.85 | 0.6}
  │
  ├─ holder ∈ {sleeve, raw, sealed} → C_grade = 1.0, graded = null
  │
  ▼
S4  ENTITY MATCHING                         web/match.py (neu) + prices.py
    4.1 identify_card, jetzt MIT item_class als Vorgabe
    4.2 Kandidaten: Karten-DB (nur tcg/sport) · PC /api/products (20 Stück)
    4.3 DOMÄNEN-GATE (hart): domain(kandidat) == item_class ?
    4.4 Bit-Scoring → P je Kandidat
    4.5 Ambiguität: P1 − P2 ≥ 0.15
    4.6 card_key deterministisch — NIE uuid4
  │
  ├─ kein Kandidat übersteht das Gate → E5 entfällt, Kaskade läuft ohne PC
  ├─ P1 < 0.65                        → E5 entfällt
  ├─ P1 − P2 < 0.15                   → E5 entfällt, Grund MEHRDEUTIG
  └─ sonst identity_conf = P1
  │
  ▼
S5  BEWERTUNG                                        web/catalog.py
    Katalog-Read (card_key × grade, TTL nach Belegklasse)
    parallel: E2 sold · E5 pc (nur wenn S4 bestanden) · E4 eu_probe (gefiltert)
    Kaskade unverändert in der REIHENFOLGE, geändert in den GATES
  │
  ▼
S6  PLAUSIBILITÄTS-TOR                    catalog.py + app_api.py (beide Pfade)
    T1 Anker-Test      T2 Quellen-Widerspruch   T3 Streuung
    T4 Grade-Monotonie T5 Konfidenz-Schwelle
  │
  ├─ C_gesamt ≥ 0.90 und alle Tests bestanden → price_state = "belegt"
  ├─ 0.65 ≤ C_gesamt < 0.90 oder ein Test knapp → price_state = "spanne"
  └─ sonst                                      → price_state = "unbekannt"

  In ALLEN drei Fällen bleibt est_value gesetzt.
  Nur die ANZEIGE und der price_basis im Listing ändern sich.
```

## 2.2 Stufe S1 — Aufnahme

| | |
|---|---|
| **Eingabe** | `item["photos"]`, optional `photos_raw` |
| **Verarbeitung** | vorhanden: `exif_transpose` (`cardscan.py:38-45`), HEIC (`:19-25`), `group_photos` (`:341-378`), `crop_photos` (`:266-284`), `slab_recut` mit Seitenverhältnis- und Zerr-Schutz (`:287-338`, Prüfungen `:313`, `:320-323`). **Neu:** `label_crop()` schneidet aus dem entzerrten Slab den oberen Streifen; `glanz_score()` misst über eine HSV-Maske (hoch V, niedrig S) den Reflexionsanteil |
| **Ausgabe** | `photos[]`, `photos_raw[]`, **neu** `label_crop_path` (eigenes Feld), `bild_qualitaet = {glanz, kanten_ok}` |
| **Konfidenz** | `C_bild = 1.0` bei `glanz < 0.10` und erfolgreichem `slab_recut`; `0.8` bei 0.10–0.25; `0.6` darüber |
| **Fallback** | Freisteller scheitert → Original behalten (heutiges Verhalten, richtig). `slab_recut` unplausibel → verwerfen. Glanz zu hoch → Pipeline läuft weiter mit `C_bild = 0.6` |
| **Zieldatei** | `/Users/smorty/ebay-bot/web/cardscan.py`, Aufruf in `app_api.py:922-937` |

**Harte Regel:** `label_crop_path` darf **niemals** in `item["photos"]` landen. `crop_photos` schreibt dorthin (`app_api.py:895`), und über `main_image_index` / `reorder_photos` (`bot/main.py:269`) würde der Label-Ausschnitt zum eBay-Hauptbild. Dein Grundsatz „eigenes Foto ist überall das Hauptbild" (`STATUS.md:163`) bleibt unangetastet.

**Nebenbefund:** `requirements.txt` listet acht Pakete — `opencv-python`, `rembg` und `numpy` fehlen, obwohl `cardscan.py:159-176` sie importiert. Auf einem frischen Server fällt der Freisteller **stumm** aus, weil `cardscan.py:254` die `ImportError` abfängt. Das gehört vor jede weitere Arbeit in Ordnung gebracht.

## 2.3 Stufe S2 — Klassifizierung

| | |
|---|---|
| **Eingabe** | Fotos (im bestehenden `analyzer.analyze`-Call), Verkäufertext |
| **Verarbeitung** | Das Modell liefert `classification.item_class` und `classification.holder` mit. **Kein zweiter Vision-Call.** `detect_card` (`cardscan.py:101`) bleibt, wo es ist — innerhalb `slab_recut`, dem einzigen Aufrufer (`cardscan.py:292`) |
| **Ausgabe** | `analysis.classification.{item_class, holder, language, confidence, evidence}` |
| **Konfidenz** | `C_klasse = 1.0` bei `confidence.item_class == "hoch"`; `0.75` bei „mittel"; `0.5` bei „niedrig" |
| **Fallback** | `item_class = "other"` ist **kein Abbruch**. Es entfällt nur die klassen-gebundene PriceCharting-Leiter; E4 und E6 tragen weiter. Dein Telegram-Weg für Elektronik, Fashion, Werkzeug läuft unverändert |
| **Zieldatei** | `bot/claude_client.py` (Prompt + `parse_listing_json:127-130`), Validierung gegen die Enum-Liste |

**Selbstkonsistenz-Prüfung in `parse_listing_json`:** Wenn `holder != "slab"`, wird `graded_info` auf `None` gesetzt. Das ist eine Prüfung derselben Modellantwort gegen sich selbst, keine unabhängige Validierung — ehrlich benannt, aber wirksam gegen die häufigste Verwechslung (Toploader als Slab).

## 2.4 Stufe S3 — Slab: Grader, Note, Zertifikatsnummer

**Zwei-Kanal-Prinzip:** Kanal A ist QR/Barcode auf dem Label-Crop, Kanal B die LLM-Ablesung. Stimmen sie überein, ist die Cert-Nummer sicher und es braucht keinen PSA-Call. Weichen sie ab, entscheidet der PSA-Lookup (nur bei PSA möglich).

**Grader-Kanon** (`web/slab.py`, eine Wahrheit für drei Nutzer — `claude_client.py:139-142`, `catalog.py:81-84`, `sold.py:104-111`):

```
PSA · BGS · BVG · BCCG · BAS · CGC · CBCS · SGC · WATA · VGA · CGA · TAG · ACE · MANA · CSG
```

`beckett` wird **nicht mehr blind auf BGS** gemappt, sondern aus dem Labeltyp entschieden: vier Subgrades sichtbar → BGS; Vintage ohne Subgrades → BVG; „Collectors Club" → BCCG; nur Autogramm-Bestätigung → BAS. Nicht entscheidbar → `grader = null`, `grader_raw = "BECKETT"`.

**Skalen-Validierung** — das eigentliche Gate:

| Grader | Gültig | Besonderheit |
|---|---|---|
| PSA | 1–10, 1.5 | Cert 8–10 Ziffern (Heuristik) |
| BGS | 1–10 in 0.5-Schritten, real auch 9.2/9.4 | vier Subgrades, Labelfarbe silber/gold/schwarz ist Datenfeld |
| BVG | wie BGS | ohne Subgrades |
| BCCG | eigene, mildere Skala | **nie** gegen BGS-Preise mappen |
| CGC Cards | 1.5–9.5 Halbstufen, Gem Mint 10, Pristine 10 | **kein 9.8** |
| CGC Comics | 0.5–10.0 | 9.8 ist Standard-Topnote |
| SGC | 1–10 | moderne Certs 7 Ziffern, alte 11 mit Bindestrich |
| WATA | Box-Note + Seal Rating C…A++ | zwei Achsen |

**Die Regel, die den 9.8-Fall abfängt:** `grader == CGC` und `item_class == tcg_card` und `grade == 9.8` ist auf dieser Skala unmöglich. Entweder ist die Klasse falsch (es ist ein Comic) oder die Note. Beides heißt: `graded = null`, raw-Pfad, Flag. Niemals eine unmögliche Note bepreisen.

| | |
|---|---|
| **Ausgabe** | `analysis.slab = {grader, grader_raw, scale, grade, grade_numeric, subgrades, seal_rating, cert_number, cert_channel, cert_verified, positive_evidence, confidence}` |
| **Konfidenz** | `1.0` Kanal A und B einig oder Lookup verifiziert · `0.85` nur B, Format plausibel, Note auf Skala · `0.6` nur B, Cert unplausibel · `0.0` Note nicht auf der Skala |
| **Fallback** | Note ungültig oder Grader unentscheidbar → `graded = null`, raw-Pfad, kein Abbruch. Das Stück hat einen Wert, nur keinen belegbaren Grade-Aufschlag |
| **Zieldatei** | `/Users/smorty/ebay-bot/web/slab.py` (neu): `decode_codes()`, `GRADER_KANON`, `SKALEN`, `CERT_FORMAT`, `normalize_grade()`, `scale_ok()`, `grade_rank()` |

**PSA-Budget-Regel:** 100 Calls/Tag sind für zehntausende Nutzer nichts. Deshalb (a) Cert-Nummern **permanent** cachen — eine Cert-Nummer ist unveränderlich, der perfekteste Cache-Schlüssel im System, eigene Tabelle `certs(cert_number PK, grader, subject, grade, year, brand, payload, ts)`; (b) Lookup nur auslösen, wenn Kanal A und B widersprechen **oder** der erwartete Wert über 200 € liegt; (c) vor dem Skalieren mit PSA über ein Kontingent sprechen.

## 2.5 Stufe S4 — Entity Matching

**Domänen-Gate (hart, keine Punkte):**

```python
def domain_of_pc(console_name: str) -> str:
    c = (console_name or "").lower()
    if c.startswith("comic books"):        return "comic_manga"
    if c in KONSOLEN:                      return "video_game"     # Playstation …, Nintendo …, Xbox …, Game Boy …
    return "card"                          # TCG- und Sport-Set-Namen
```

Regel: `domain(kandidat)` unvereinbar mit `item_class` → Kandidat verworfen, keine weitere Rechnung. Das allein verhindert die 603 €.
Sofort-Match ohne Scoring: `upc` oder `epid` identisch → `P = 1.0`.

**Bit-Scoring** (Fellegi-Sunter-Stil, [Splink-Doku](https://moj-analytical-services.github.io/splink/topic_guides/theory/fellegi_sunter.html)):

```
S = w0 + Σ w_i          P = 1 / (1 + 2^(-S))
```

| Merkmal | Bedingung | Bits |
|---|---|---|
| Prior `w0` | immer | **−3.0** |
| Hart-ID (Kartennr. `OP01-016`, `Vol. 103`, Set-Code) | im Kandidatennamen enthalten | **+6.0** |
| | Item hat eine, fehlt im Namen | **−6.0** |
| | Item hat keine | 0 |
| Namensähnlichkeit (Token-Set, 0–100) | ≥95 / 85–94 / 70–84 / 50–69 / <50 | +3.0 / +1.5 / 0 / −2.0 / −4.0 |
| Set-Serie gegen `console-name` | ≥90 / 70–89 / 40–69 / <40 | +3.0 / +1.0 / 0 / −2.5 |
| Jahr-Differenz | ≤1 / 2–3 / >3 | +1.5 / 0 / −2.0 |
| Preis-Plausibilität `r = max(a/b, b/a)` gegen die stärkste andere Quelle | ≤1.5 / 1.5–3 / 3–6 / >6 | +2.0 / 0 / −3.0 / −6.0 |
| Sprache stimmig / widersprüchlich | | +1.0 / −2.0 |

Fehlende Daten ergeben Gewicht 0 — kein Beweis ist kein Gegenbeweis.

**Bildvergleich (pHash) ist bewusst nicht in Stufe 1.** `imagehash` und `rapidfuzz` sind nicht installiert, und solange kein Katalogbild vorliegt, ist das Gewicht ohnehin 0. Die Token-Ähnlichkeit läuft mit der Standardbibliothek:

```python
def token_set_ratio(a: str, b: str) -> int:
    """Ersatz für rapidfuzz — keine neue Abhängigkeit."""
    import difflib
    ta, tb = set(_norm(a).split()), set(_norm(b).split())
    if not ta or not tb:
        return 0
    gemeinsam = ta & tb
    rest_a, rest_b = " ".join(sorted(ta - gemeinsam)), " ".join(sorted(tb - gemeinsam))
    basis = " ".join(sorted(gemeinsam))
    return max(
        int(100 * difflib.SequenceMatcher(None, basis, (basis + " " + rest_a).strip()).ratio()),
        int(100 * difflib.SequenceMatcher(None, basis, (basis + " " + rest_b).strip()).ratio()),
        int(100 * difflib.SequenceMatcher(None, (basis + " " + rest_a).strip(),
                                                (basis + " " + rest_b).strip()).ratio()),
    )
```

**Nachrechnung deines Falls**, bewusst ohne Domänen-Gate, um zu zeigen, dass auch das Scoring allein trägt:
Item „One Piece Manga Band 103" gegen Kandidat „Nami [Manga] OP01-016" / console „One Piece Romance Dawn":
`w0 −3.0` · Hart-ID „103" fehlt `−6.0` · Name-Ratio ca. 40 `−4.0` · Set-Ratio niedrig `−2.5` · Jahr 0 · Preis 603 € gegen Marktmedian ca. 8 € → r ≈ 75 `−6.0`
**S = −21.5 → P ≈ 3·10⁻⁷.** Sicher verworfen, mit oder ohne Gate.

**Schwellen:** `P ≥ 0.90` verwenden · `0.65 ≤ P < 0.90` nur als Rückfall mit Kennzeichnung · `P < 0.65` verwerfen · zusätzlich `P1 − P2 < 0.15` → verwerfen, auch bei hohem P1. Zwei fast gleich gute Kandidaten heißt: die Suche kann es nicht entscheiden, dann darf SERO es auch nicht.

**Woher 0.90 kommt:** Chows Regel für selektive Vorhersage. Ist ein falscher Preis `k`-mal so teuer wie ein fehlender, ist die optimale Schwelle `τ = k/(k+1)`. Für ein Produkt, das automatisch Listings erzeugt, ist `k = 9` realistisch (Fehlverkauf, Reklamation, Vertrauensverlust) → `τ = 0.90`. Du wählst `k`, die Schwelle fällt heraus.

**Der Schlüssel — und was ausdrücklich NICHT gemacht wird:**

Ein Vorentwurf wollte `card_key_of` bei fehlender Identität `None` zurückgeben lassen. Das habe ich geprüft: `cards.card_key` ist `TEXT PRIMARY KEY` (`catalog.py:32`) und SQLite **erlaubt NULL in einem TEXT-PK** — `ON CONFLICT` greift nie, jeder Lauf fügt eine neue NULL-Zeile ein. `card_prices.card_key` ist `TEXT NOT NULL` (`catalog.py:38`) und wirft `IntegrityError`, der im `except Exception` bei `app_api.py:802` bzw. `:1011` verschwindet. Das wäre schlimmer als heute und unsichtbar. **Verworfen.**

Der Fix gehört eine Ebene höher und ist zwei Zeilen:

```python
# app_api.py:789-791 und :999-1001
card_ref = dict(item.get("card") or card_info or {})
card_ref["name"] = card_ref.get("name") or item.get("name")     # statt setdefault
card_ref["item_class"] = (item.get("analysis") or {}).get("classification", {}).get("item_class")
card_ref["language"]   = (item.get("card") or {}).get("language")
```

```python
# catalog.py:48-60 — Vertrag bleibt: IMMER ein String
def card_key_of(card: dict) -> str:
    if card.get("ref_id"):
        return f"{card.get('game') or card.get('item_class') or 'x'}:{card['ref_id']}"
    kern = _norm("|".join(str(card.get(k) or "") for k in
                 ("item_class", "game", "name", "number", "set_name", "language", "edition")))
    if kern.strip():
        return "h:" + hashlib.sha1(kern.encode()).hexdigest()[:16]
    # Letzter Rückfall: DETERMINISTISCH statt uuid4 — beim nächsten Refresh
    # trifft der Cache, statt eine neue Zeile zu erzeugen.
    return "solo:" + hashlib.sha1(str(card.get("item_id") or "").encode()).hexdigest()[:12]
```

`language` und `edition` gehören in den Hash. Ohne sie teilen sich Deutsch und Englisch, 1st Edition und Unlimited eine Preiszeile — bei mehrfachem Preisunterschied.

## 2.6 Stufe S5 — Bewertung

**Die Kaskade behält ihre Reihenfolge.** Geändert werden die Gates davor und die Behandlung von `weak`:

```
1. sold (E2)                                          → ebay_sold
2. sonst eu (E4, GEFILTERT) und nicht (pc_trusted und eu > pc*4)  → ebay_eu
3. sonst pc (E5) und S4 bestanden (P ≥ 0.90)          → pricecharting
4. sonst base (E3/E6) — jetzt AUCH bei graded         → base.source
5. sonst pc mit 0.65 ≤ P < 0.90                       → pricecharting_weak
6. sonst alte Zeile / None
```

**Vier Änderungen, jede einzeln begründet:**

- **Stufe 3 bekommt ein Gate.** Nur ein Kandidat, der S4 bestanden hat, wird überhaupt bepreist. `pc_trusted` (`catalog.py:149-174`, alle Ziffern-Tokens müssen im PC-Namen stehen) bleibt als zweite, unabhängige Prüfung erhalten — es ist ein guter Reflex und kostet nichts.
- **Stufe 4 gilt jetzt auch für gegradete Stücke.** Heute steht dort `and grade == "raw"` (`catalog.py:196`), weshalb jeder Slab direkt auf `pricecharting_weak` durchfällt — den unsichersten Zweig. Neu: als Referenz dient die **raw-Zeile desselben `card_key`** aus dem Katalog, die `get_price` (`catalog.py:87`) ohnehin schon lesen kann. Kein neuer externer Call, keine neue Tabelle.
- **Stufe 5 setzt keinen belegten Marktwert mehr.** Der Wert wird eingetragen, aber mit `price_state = "unbekannt"` und `price_reason = "UNBEKANNT_ZUORDNUNG"`. Die App zeigt ihn nicht als Marktwert. Er bleibt für den Listing-Preis verfügbar, damit die Kette nicht bricht.
- **Stufe 2 wird gefiltert** — aber nur dort.

**Der EU-Filter, ohne Kollateralschaden:** `research_price` hat einen Rückgabewert für drei Aufgaben — `eu_probe` im Katalog, `item["market"]` als angezeigte Vergleichsangebote (`app_api.py:826-827`, `:969-970`) und den Telegram-Preisvorschlag für Allgemeinware (`bot/main.py:571`). `sold.fits` verlangt, dass **jedes** Ziffern-Token im Titel steht; bei „Bosch GSR 18V-55 Professional 2x2.0Ah" findet das kaum ein Angebot. Ein globaler Filter würde die Marktanzeige und den Bot beschädigen.

**Lösung:** Der Filter wird als Callable **nur in die `eu_probe`-Lambda injiziert**, nicht in den Aufruf für `item["market"]`:

```python
# bot/ebay/browse.py:19
async def research_price(client, query, limit=20, titel_filter=None):
    ...   # Filter läuft VOR dem IQR-Trimm auf item.get("title")

# app_api.py:801 / :1010
eu_probe=lambda q: research_price(ebay, q, titel_filter=_fits_fuer(item))
```

`bot/` importiert dabei nicht auf `web/` — sonst entsteht ein Zyklus. Zusätzlich gehört `check_price_plausibility` (`bot/main.py:238`) in dieselbe Lambda; der EU-Zweig umgeht diesen Wächter heute komplett.

**Die dritte Grade-Leiter** kommt in `_grade_fields` (`pricecharting.py:21-46`), gewählt nach `item_class`:

```python
if item_class == "comic_manga":
    if val >= 10:  return ["bgs-10-price", "manual-only-price"]
    if val >= 9.8: return ["manual-only-price", "condition-17-price"]
    if val >= 9.4: return ["condition-17-price", "box-only-price"]
    if val >= 9.2: return ["box-only-price", "graded-price"]
    if val >= 8.0: return ["graded-price", "new-price"]
    if val >= 6.0: return ["new-price", "cib-price"]
    return ["cib-price", "loose-price"]
```

**TTL nach Belegklasse** (heute `catalog.py:115` nach Quellenname): E1/E2 24 h · E3 12 h · E4 6 h · E5 6 h · `unbekannt` 1 h. Die Selbstheilung schwacher Werte ist ein gutes Prinzip und bleibt.

**Cache zweistufig** in `pricecharting.py:81`: `query → pc_id` mit 1 h TTL (die unsichere Zuordnung), `pc_id → Produktdaten` mit 12 h (der stabile Fakt). So ist ein Fehl-Match gezielt invalidierbar.

## 2.7 Stufe S6 — Plausibilitäts-Tor

**Der Anker — der strukturelle Fix, ohne neue Infrastruktur:**

```python
# in catalog.refresh_price, vor der Kaskade
anker = base
if anker is None and grade != "raw":
    r = get_price(store, card_key, "raw")               # existiert bereits
    if r and r.get("value_eur"):
        anker = {"value": r["value_eur"], "source": r["source"], "label": r["source_label"]}
if anker is None:
    est = (listing or {}).get("estimated_price_range_eur")
    if est:
        anker = {"band": (est["low"], est["high"])}     # E6, nur als Band
```

Damit hat jeder Wächter immer eine Referenz — auch beim Manga, auch beim Slab.

**Die fünf Tests:**

| # | Test | Bedingung | Reaktion |
|---|---|---|---|
| T1 | **Anker** | Wert in `[anker/4, anker*4]`, beim Band `[low*0.4, high*2.5]` | verletzt bei E4/E5 → `unbekannt`, `UNBEKANNT_WIDERSPRUCH`. Verletzt bei E1/E2 mit n≥3 → Wert bleibt, `price_state = "spanne"` (echte Verkäufe dürfen den Anker schlagen) |
| T2 | **Quellen-Widerspruch** | zwei unabhängige Quellen, Verhältnis `max/min` | ≤2 → Konfidenz +0.05 · 2–4 → `spanne` mit `[min, max]` · >4 → `unbekannt` |
| T3 | **Streuung** | bei E2: `max(sales)/min(sales)` | >6 bei `n_avg < 4` → `spanne`. **Ersetzt den toten Wächter** `catalog.py:212-224`, dessen Bedingung `n_avg < 2` nie eintritt |
| T4 | **Grade-Monotonie** | siehe 2.8 | schwächere Belegklasse verliert und wird zu `spanne` degradiert — **kein Wert wird umgebogen** |
| T5 | **Konfidenz** | `C_gesamt = C_bild × C_klasse × identity_conf × C_grade × C_quelle` | ≥0.90 `belegt` · 0.65–0.90 `spanne` · <0.65 `unbekannt` |

`C_quelle`: E1 `1.0` · E2 mit n≥3 frisch `0.95`, n=2 `0.85`, `stale` `0.75` · E3 `0.95` · E4 mit ≥5 gefilterten Treffern `0.8`, 3–4 `0.7` · E5 mit P≥0.90 `0.85`, mit P≥0.65 `0.5`.

**Korrekturen schreiben zurück in den Katalog.** Heute korrigiert Wächter C nur das Item (`app_api.py:833-845`); die Zeile bleibt vergiftet, deshalb stehen 90 von 247 Zeilen auf `pricecharting_weak`. Neu: dieselbe Korrektur geht in `card_prices`, und `valuation.catalog.poisoned_cleared = true` hält es fest. Außerdem gehört Wächter C **auch in `analyze_collection_item`** — beim ersten Scan, dem einzigen, den der Nutzer sieht, fehlt er heute.

## 2.8 Grade-Monotonie als Katalog-Invariante

Für jeden `card_key` und jeden **kanonischen Grader** gilt:

```
grade_rank(g, n1) < grade_rank(g, n2)  ⟹  value(key, g, n1) ≤ value(key, g, n2) · 1.05
```

**Streng nur innerhalb eines Graders.** Über Grader hinweg ist Monotonie sachlich falsch (BGS 9.5 ≈ PSA 10, PSA 10 erzielt trotzdem mehr). **Raw ist nicht Teil der Kette** — ein PSA 3 kann weniger wert sein als eine rohe Near-Mint-Karte. Stattdessen ein Anker-Test: `grade ≥ 8` ⟹ `value ≥ value(raw) · 0.8`, sonst Flag ohne automatischen Eingriff.

Durchsetzung an zwei Stellen:
- **Beim Schreiben** (`catalog.py` vor dem `INSERT` bei `:242`): Geschwisterzeilen lesen, Verletzung prüfen. Wer verliert, entscheidet die **Belegklasse**: E1/E2 schlägt E3/E4/E5 (die schwächere Zeile wird `spanne`); gleiche Klasse → **beide** werden `spanne`. Eine belegte Zeile wird nie still zurechtgebogen, das wäre Datenfälschung.
- **Als Sweeper** — und zwar als **separates Skript mit `--dry-run` als Default**, nicht im 12-h-Autolauf. `card_prices` hat keinen Mandanten (`catalog.py:36-39`); ein Fehler schlägt auf jeden Nutzer durch. `force=True` umgeht die TTL und feuert gegen 130point mit 8 s globaler Drossel — bei 247 Zeilen wären das im Schlechtfall 33 Minuten Vollauslastung der einzigen Verkaufsquelle, währenddessen laufen alle Live-Scans in den 600-s-Cooldown (`sold.py:66`). Deckel: höchstens 50 Force-Refreshes je Lauf. Nach dem 02.08. ist ein automatischer Massenschreiber auf geteilte Daten die falsche Reihenfolge.

## 2.9 `price_state` und `price_reason` — und warum `est_value` bleibt

| `price_state` | `est_value` | Anzeige in der App | Listing-Preis |
|---|---|---|---|
| `belegt` | gesetzt | Marktwert mit Quellenlabel | `market_value` gesetzt, `market_source` = Quellenlabel |
| `spanne` | gesetzt (Mitte) | „6 – 12 €" mit Hinweis, **keine Punktzahl** | `market_value` gesetzt, `market_source` = „Spanne, Zuordnung grob" |
| `unbekannt` | gesetzt (bester Rückfall) | „Wert unbekannt — <Grund>", manuelle Eingabe angeboten | `market_value` gesetzt, `market_source` = „KI-Schätzung, nicht belegt" |

**Warum `est_value` nie `None` wird:** `item_value` (`app_api.py:470-475`) speist `preset["market_value"]` (`:2249-2252`), das über `app_run_pipeline(..., market=market)` (`:2306-2310`) in `apply_price_rule` (`bot/main.py:213-235`) landet. Dessen Rangfolge lautet: Nutzerpreis > Vorlage > Marktwert > eBay-Comps > KI-Schätzung. Fehlt `market_value`, recherchiert die Pipeline blind neu — genau der Fehler vom 02.08. (`STATUS.md:394`: 307,90 € statt 653,02 €). Zusätzlich sind die Preis-Modi „Marktwert" und „Marktwert −10 %" mit `and est` gegated (`app_api.py:2258`, `:2260`) und lieferten stillschweigend nichts. Und `check_alert` (`:583-596`), `snapshot_price` (`:598-601`), der Sammlungswert (`:1677`, `:2928`) sowie die PWA-Sortierung hängen alle daran.

„Lieber keine Zahl als eine falsche" heißt: **die App nennt es nicht Marktwert**. Es heißt nicht: das Listing bekommt keinen Preis. Ein Listing ohne Preis ist auch ein Schaden.

**`price_reason` als geschlossenes Enum:**

| Code | Text für den Nutzer | Nächster Versuch |
|---|---|---|
| `UNBEKANNT_KLASSE` | „Nicht sicher erkannt, was das ist." | bei neuem Foto |
| `UNBEKANNT_IDENTITAET` | „Stück nicht eindeutig zugeordnet — bitte Set und Nummer prüfen." | manuelle Korrektur über `/cardsearch` (`app_api.py:3505`) |
| `UNBEKANNT_MEHRDEUTIG` | „Mehrere passende Produkte gefunden, keines eindeutig." | manuelle Auswahl aus den Top 3 |
| `UNBEKANNT_ZUORDNUNG` | „Preisquelle passt nicht sicher zum Stück." | 12-h-Autolauf |
| `UNBEKANNT_GRADE_UNGUELTIG` | „Die abgelesene Note passt nicht zur Skala dieses Anbieters." | neues Label-Foto |
| `UNBEKANNT_KEINE_BELEGE` | „Keine belastbaren Vergleichsdaten." | 12-h-Autolauf |
| `UNBEKANNT_WIDERSPRUCH` | „Die Quellen widersprechen sich zu stark." | 12-h-Autolauf |
| `UNBEKANNT_MONOTONIE` | „Preis passt nicht zur Bewertungsstufe." | Sweeper |
| `UNBEKANNT_DROSSEL` | „Preisabfrage ausgelastet — gleich erneut." | nach Cooldown (600 s, `sold.py:66`) |

Ein „unbekannt" ist **kein Fehler**: `status` bleibt `ready`, das Listing entsteht, der Nutzer kann den Preis selbst eintragen.

---

# Teil 3 — Das JSON-Schema

## 3.1 Vier Regeln

1. **Additiv, nie ersetzend.** Kein vorhandenes Feld wird umbenannt, verschoben oder umgedeutet.
2. **Legacy-Felder werden abgeleitet.** Ab v1 berechnet der Schreiber zuerst die neuen Blöcke und projiziert daraus `est_value`, `price_source`, `price_label`, `graded`.
3. **Der Scan-Teil reist im vorhandenen `analysis`-Blob mit.** `analysis` steht bereits in `ANALYSE_FELDER` (`app_api.py:731-733`) — für `classification`, `slab`, `identity`, `queries`, `scan` ist damit **keine** Persistenz-Änderung nötig.
4. **Kein DDL.** `collection_items.data` ist JSON, `card_prices.detail` ebenfalls (`catalog.py:36-40`). Kein `ALTER TABLE`, kein Bulk-Update, kein Schreibzugriff auf Bestandszeilen.

**Namenskollisionen geprüft:** `grading` ist belegt (KI-Grading, `app_api.py:3492`), `sold` doppelt belegt (Belege gegen `sold_ts`, dokumentiert `:539-542`), `match` belegt in API-Antworten (`:3527ff`) — deshalb `entity_match`. Frei und übernommen: `valuation`, `classification`, `slab`, `identity`, `plausibility`, `schema_version`.

## 3.2 Kommentiertes Beispiel — dein Fehlerfall, richtig behandelt

```jsonc
{
  // ── 1) Versions-Marker (neu, v1) ────────────────────────────────
  "schema_version": 1,          // fehlt = v0 (Altdatensatz)

  // ── 2) Bestand, unverändert ─────────────────────────────────────
  "status": "ready",
  "name": "One Piece Band 103 Manga Deutsch Carlsen BGS 9.0",
  "category": "Sonstiges",
  "condition": "USED_EXCELLENT",
  "quantity": 1,
  "photos": ["/…/abc_slab.png", "/…/def.jpg"],
  "photos_raw": ["/…/abc.jpg", "/…/def.jpg"],

  "card_info": {"single": false, "game": "onepiece", "name": null},
  "card": null,
  "card_key": "comic_manga:one-piece:v103:de",   // kein solo:<uuid> mehr

  "est_value": 8.50,                 // ABGELEITET, immer gesetzt
  "est_low": 6.00,
  "est_high": 12.00,
  "price_source": "ebay_sold",       // ABGELEITET über die Mapping-Tabelle 3.4
  "price_label": "Ø letzte 3 eBay-Verkäufe",
  "price_detail": {"pc_usd": null, "pc_field": null, "pc_product": null},
  "price_updated": 1754236800.0,
  "market": {"count": 7, "median": 11.0, "estimated": false, "samples": []},
  "sold": {"avg3": 8.50, "n_avg": 3, "sales": []},

  "graded": {"grader": "BGS", "grade": "9.0", "cert_number": "0018472913"},
                                     // ABGELEITET aus analysis.slab,
                                     // Kürzel normalisiert (nicht "BECKETT")

  // ── 3) analysis: Bestand unverändert, neue Blöcke ergänzt ───────
  "analysis": {
    "title": "One Piece Band 103 Manga Deutsch Carlsen BGS 9.0",
    "description_html": "<p>…</p>",
    "category_query": "One Piece Manga",
    "condition": "USED_EXCELLENT",
    "aspects": {"Marke": ["Carlsen"], "Sprache": ["Deutsch"]},
    "search_query_for_pricing": "One Piece Vol 103 Manga BGS 9",
    "estimated_price_range_eur": {"low": 6.0, "high": 12.0},
    "graded_info": {"grader": "BGS", "grade": "9.0", "cert_number": "0018472913"},
    "format": "FIXED_PRICE", "quantity": 1, "main_image_index": 0,
    "estimated_weight_grams": 400, "uncertain": false, "question": null,

    "scan": {"v": 1, "pipeline": "vision@2026.08.1",
             "model": "claude-sonnet-4-6", "ts": 1754236780.4},

    "classification": {                       // HARTES ROUTING-FELD
      "item_class": "comic_manga",
      "holder": "slab",
      "language": "de",
      "confidence": {"item_class": "hoch", "holder": "hoch", "language": "hoch"},
      "evidence": ["Label nennt Verlag und Bandnummer, nicht Set und Kartennummer",
                   "Buchrücken im Case sichtbar"]
    },

    "slab": {                                 // null wenn holder != "slab"
      "grader": "BGS", "grader_raw": "BECKETT", "scale": "bgs_10",
      "grade": "9.0", "grade_numeric": 9.0,
      "subgrades": null, "seal_rating": null, "label_color": null,
      "cert_number": "0018472913", "cert_channel": "ocr",
      "cert_verified": false, "positive_evidence": true,
      "confidence": {"grader": "hoch", "grade": "hoch",
                     "cert_number": "niedrig", "subgrades": null}
    },

    "identity": {
      "title_native": "ONE PIECE 103", "franchise": "One Piece",
      "number": "103", "number_kind": "volume",
      "set_name": null, "set_code": null, "set_total": null,
      "publisher": "Carlsen", "release_year": 2022,
      "edition": null, "variant": null, "isbn": null, "upc": null,
      "confidence": {"number": "hoch", "release_year": "mittel",
                     "publisher": "mittel", "set_code": null}
    },

    "queries": {                              // getrennt statt EINER
      "sold": "One Piece Vol 103 Manga BGS 9",
      "catalog": "One Piece Volume 103",
      "ebay": "One Piece Band 103 Manga Beckett 9"
    },

    "photos_meta": {"label_index": 0, "front_index": 0, "back_index": 1},
    "label_crop_path": "/…/abc_label.png"     // NIE in photos[]
  },

  // ── 4) valuation: neu, Top-Level, vom Preis-Pfad geschrieben ────
  "valuation": {
    "v": 1, "pipeline": "value@2026.08.1", "ts": 1754236812.9,

    "value_eur": 8.50, "value_low_eur": 6.00, "value_high_eur": 12.00,
    "source": "aggregator_sold",       // neue Taxonomie, Tabelle 3.4
    "legacy_source": "ebay_sold",      // was in price_source projiziert wird
    "label": "Ø letzte 3 eBay-Verkäufe",
    "confidence": 0.82,
    "price_state": "belegt",           // belegt | spanne | unbekannt
    "price_reason": null,

    "grade_bucket": "BGS 9",
    "grade_exact": false,              // Quelle rastert gröber als die Note
    "grade_note": "Quelle kennt nur 9 / 9.5 / 10 — zwischen den Stufen eingeordnet",

    "currency": "EUR",
    "fx": {"pair": "USD/EUR", "rate": 0.9123, "as_of": "2026-08-03"},

    "evidence": {
      "sample_size": 3, "sample_total": 6, "window_days": 90,
      "oldest_days": 41, "newest_days": 6, "stale": false,
      "items": [{"title": "One Piece Vol. 103 Manga BGS 9", "price_eur": 9.00,
                 "date": "2026-07-28", "url": "https://…", "fits": true}]
    },

    "considered": [                    // auch die verworfenen Quellen
      {"source": "aggregator_sold", "value_eur": 8.50, "sample_size": 3, "used": true},
      {"source": "pricecharting", "value_eur": 603.12, "used": false,
       "rejected": "domain_mismatch"},
      {"source": "ebay_active_de", "value_eur": 9.68, "sample_size": 7,
       "used": false, "rejected": "weaker_source"}
    ],

    "entity_match": {
      "status": "rejected", "provider": "pricecharting",
      "candidate": {"id": "6710612", "name": "Nami [Manga] OP01-016",
                    "console_name": "One Piece Romance Dawn", "domain": "card"},
      "runner_up": null, "score_bits": -21.5, "probability": 0.0000003,
      "margin": null,
      "features": {
        "domain_gate": {"expected": "comic_manga", "got": "card", "ok": false},
        "hard_id":    {"value": "103", "found": false, "bits": -6.0},
        "name_ratio": {"value": 40, "bits": -4.0},
        "set_ratio":  {"value": 12, "bits": -2.5},
        "year_delta": {"value": 0, "bits": 0.0},
        "price_ratio":{"value": 70.9, "bits": -6.0}
      },
      "decision_reason": "domain_mismatch"
    },

    "plausibility": {
      "ok": true,
      "checks": [
        {"id": "domain_gate", "result": "fail", "action": "candidate_rejected"},
        {"id": "anker",       "result": "pass", "ratio": 1.07},
        {"id": "monotonie",   "result": "skip", "detail": "keine Nachbarnote"}
      ],
      "corrections": [{"from": 603.12, "to": 8.50, "by": "domain_gate",
                       "was_source": "pricecharting"}]
    },

    "catalog": {"card_key": "comic_manga:one-piece:v103:de", "grade": "BGS 9",
                "shared": true, "poisoned_cleared": true}
  }
}
```

## 3.3 Feldtabelle — nur die Neuerungen

| Feld | Typ | Pflicht | Bedeutung | seit |
|---|---|---|---|---|
| `schema_version` | int | ja (v1) | fehlt = v0. Der einzige Schalter für Leser | v1 |
| `valuation` | dict/null | nein | Bewertung mit Quellen-Attribution | v1 |
| `analysis.scan.{v,pipeline,model,ts}` | – | ja | welcher Prompt und Code hat das erzeugt | v1 |
| `analysis.classification.item_class` | enum | **ja** | `tcg_card`, `sport_card`, `comic_manga`, `video_game`, `sealed_product`, `coin`, `toy`, `electronics`, `fashion`, `home`, `tool`, `other` | v1 |
| `analysis.classification.holder` | enum | ja | `slab`, `sleeve`, `raw`, `sealed`, `none` | v1 |
| `analysis.classification.language` | str/null | nein | ISO-639-1 | v1 |
| `analysis.classification.confidence.*` | enum/null | ja | `hoch`, `mittel`, `niedrig` | v1 |
| `analysis.slab` | dict/null | ja | null wenn `holder != "slab"` | v1 |
| `analysis.slab.grader` | enum/null | ja | nur aus der Whitelist | v1 |
| `analysis.slab.grader_raw` | str/null | ja | wörtlich vom Label | v1 |
| `analysis.slab.scale` | enum | ja | entscheidet die Preisleiter | v1 |
| `analysis.slab.subgrades` | dict/null | nein | alle vier oder keiner | v1 |
| `analysis.slab.cert_channel` | enum/null | ja | `qr`, `barcode`, `ocr`, null | v1 |
| `analysis.slab.cert_verified` | bool | ja | true nur nach echtem Lookup | v1 |
| `analysis.slab.positive_evidence` | bool | ja | Grader und Note und (Cert oder Code) | v1 |
| `analysis.identity.number_kind` | enum/null | ja | `volume`, `issue`, `card_number` — verhindert Band-gegen-Kartennummer | v1 |
| `analysis.queries.{sold,catalog,ebay}` | str | ja | getrennte Suchanfragen je Zielsystem | v1 |
| `analysis.label_crop_path` | str/null | nein | eigener Pfad, **nie** in `photos[]` | v1 |
| `valuation.price_state` | enum | ja | `belegt`, `spanne`, `unbekannt` | v1 |
| `valuation.price_reason` | enum/null | ja | Enum aus 2.9 | v1 |
| `valuation.confidence` | float 0–1 | ja | `C_gesamt` | v1 |
| `valuation.evidence.*` | – | ja | Stichprobengröße, Alter, Belege mit URL | v1 |
| `valuation.considered[]` | list | ja | jede geprüfte Quelle mit `used` und `rejected` | v1 |
| `valuation.entity_match.*` | – | ja | Kandidat, Bits je Merkmal, Wahrscheinlichkeit, Grund | v1 |
| `valuation.plausibility.*` | – | ja | Testergebnisse und angewandte Korrekturen | v1 |
| `valuation.catalog.poisoned_cleared` | bool | nein | Korrektur ging auch in `card_prices` | v1 |

## 3.4 Quellen-Taxonomie und Projektion

| `valuation.source` | Herkunft | → `price_source` (Legacy) |
|---|---|---|
| `ebay_completed` | Marketplace Insights (`web/ebay_insights.py`, aus) | `ebay_sold` |
| `aggregator_sold` | 130point (`web/sold.py`) | `ebay_sold` |
| `ebay_active_de` | Browse-Median × 0,88 (`catalog.py:139-147`) | `ebay_eu` |
| `ebay_active` | Browse-Median | `ebay` |
| `pricecharting` | Match bestätigt (P ≥ 0.90) | `pricecharting` |
| `pricecharting_unmatched` | 0.65 ≤ P < 0.90 | `pricecharting_weak` |
| `tcg_db_cardmarket` / `_scryfall` / `_ygoprodeck` / `_tcgplayer` | `web/prices.py`, `web/tcgcsv.py` | `cardmarket` / `scryfall` / `ygoprodeck` / `tcgplayer` |
| `catalog_estimate` | geteilte Katalogzeile | Quelle der Zeile |
| `seller_listing` | eBay-Import | `listing` |
| `ai_estimate` | `estimated_price_range_eur` | `estimate` |

`ebay_completed` und `aggregator_sold` bilden bewusst beide auf `ebay_sold` ab — so greift `SOURCE_INFO` (`/Users/smorty/sero-app/web/sero.js:509`) unverändert, und der Unterschied bleibt trotzdem in `valuation.source` erhalten.

```python
def projiziere_legacy(item, analysis, valuation):
    """Die EINZIGE Stelle, an der Legacy-Felder ab v1 entstehen."""
    if valuation and valuation.get("value_eur") is not None:
        item["est_value"]    = valuation["value_eur"]      # IMMER gesetzt
        item["price_source"] = valuation["legacy_source"]
        item["price_label"]  = valuation["label"]
    s = (analysis or {}).get("slab")
    item["graded"] = ({"grader": s["grader"], "grade": s["grade"],
                       "cert_number": s["cert_number"]}
                      if s and s.get("positive_evidence") else None)
    item["schema_version"] = 1
```

## 3.5 Migrationsstrategie

**1. Lesen alter Datensätze — ohne Migration.** Kein Backfill, kein Bulk-Update, kein Schreibzugriff auf Bestandszeilen:

```python
version   = item.get("schema_version", 0)
valuation = item.get("valuation")                                  # None bei v0
klasse    = (item.get("analysis") or {}).get("classification")     # None bei v0
```

Jeder heutige Leser — `item_public` (`app_api.py:514-560`), die PWA, `snapshot_price`, der Export — arbeitet weiter auf den Legacy-Feldern und braucht **keine** Änderung.

**2. Lazy Upgrade.** v0-Zeilen bekommen die Blöcke erst beim nächsten Scan oder Refresh. Bestand wird nie angefasst. Das ist gleichzeitig Migration und Absicherung gegen den 02.08.

**3. Drei Versionsebenen.** Datensatz (`schema_version`), Block (`analysis.scan.v`, `valuation.v`), Lauf (`pipeline: "name@2026.08.1"`). Ohne die Lauf-Kennung ist nach einem Prompt-Wechsel nicht rekonstruierbar, welche Werte noch aus der alten Logik stammen.

**4. Vorwärtskompatibel.** Blöcke werden gemerged, nicht ersetzt: `{**alt, **neu}`. Unbekannte Schlüssel innerhalb eines Blocks werden nie entfernt.

**5. Rollback.** `schema_version` auf 0 setzen oder die Blöcke ignorieren. Die Legacy-Felder sind vollständig und selbsttragend. Es gibt keinen Zustand, in dem ein Rollback Daten verliert.

**6. Die eine Persistenzfalle — vorher beheben.** `col_save_analyse` (`app_api.py:736-753`) und `refresh_item_price` (`:862-868`) kopieren **ausschließlich** die Felder aus `PREIS_FELDER` / `ANALYSE_FELDER` (`:725-733`). Ein neues Top-Level-Feld, das dort nicht steht, wird beim nächsten Lauf **stillschweigend verworfen** — kein Fehler, kein Log. Genau zwei Ergänzungen sind nötig:

```python
PREIS_FELDER = ("card", "card_key", "card_info", "est_value", "price_source",
                "price_label", "price_detail", "market", "sold", "price_updated",
                "valuation", "schema_version")           # ← die zwei Neuen
```

`analysis` steht bereits in `ANALYSE_FELDER` — deshalb brauchen `classification`, `slab`, `identity`, `queries`, `scan` und `label_crop_path` keine Änderung. Das ist der ganze Grund für Regel 3.

**7. Katalog.** `card_prices.detail` ist bereits JSON — der `valuation`-Block wandert unverändert hinein, kein DDL.

**8. PWA-Nutzlast — nicht übersehen.** `LS.set` (`sero.js:655`) schluckt Quota-Fehler still, iOS-Safari deckelt localStorage bei etwa 5 MB. `considered` plus `entity_match.features` plus `plausibility` sind je Stück leicht 2–4 KB. Bei 1.000 Stücken ist der Sammlungs-Cache tot, ohne Fehlermeldung. Deshalb: **`valuation` gehört in `_DETAIL_ONLY`** (`app_api.py:566`) — die Listenantwort bekommt nur `price_state`, `price_reason` und `confidence` als drei flache Schlüssel:

```python
**({"value_state": v.get("price_state"), "value_reason": v.get("price_reason"),
    "value_confidence": v.get("confidence")} if (v := item.get("valuation")) else {})
```

**9. Mandantentrennung.** `card_prices` ist bewusst global (`tests/test_multitenant.py` sichert die Item-Trennung). In `valuation`, das dorthin geschrieben wird, darf **kein** nutzerspezifischer Text stehen.

---

# Teil 4 — Der System-Prompt

Dieser Prompt **ersetzt** `SYSTEM_PROMPT` in `/Users/smorty/ebay-bot/bot/claude_client.py:29-95` — aber er **erweitert** ihn inhaltlich, statt ihn auszutauschen. Die Allgemeinware-Teile (Produkterkennung, Titel-Formel mit Kategorie-Mustern, Beschreibungs-Template, USK-Regel im Wortlaut, `uncertain`) bleiben erhalten, weil `bot/main.py:526` denselben Analyzer für deinen Telegram-Weg nutzt — dort läuft dein ganzes Sortiment, nicht nur Sammlerware. Alle bisherigen Ausgabefelder bleiben; `parse_listing_json` (`:127-130`) läuft unverändert.

```text
Du bist ein Weltklasse-Produkterkennungs- und eBay-Listing-Experte für ALLE
Produktkategorien — Elektronik, Fashion, Haushalt, Spielzeug, Möbel, Werkzeug —
mit besonderer Spezialtiefe bei Sammlerware (Trading Cards, graded Slabs,
Plüschfiguren, Videospiele, Manga und Comics).

Deine Ausgabe bestimmt einen Marktwert, den ein Mensch für eine Kauf- oder
Verkaufsentscheidung benutzt. Eine falsche Zahl ist teurer als eine fehlende.
Deshalb gilt über allem:

  GRUNDGESETZ FÜR IDENTITÄT UND GRADING
  Was du am Bild nicht ablesen oder den Angaben des Verkäufers nicht entnehmen
  kannst, ist null. Kein Raten, kein Ergänzen aus Wahrscheinlichkeit, kein
  Schluss von einem Feld auf ein anderes. null ist immer eine korrekte Antwort.
  Ein erfundener Wert ist immer ein Fehler.
  Das gilt für: item_class, holder, alle Felder in slab und identity,
  Set-Codes, Nummern, Sprache, Auflage und Variante.
  Für den Listing-TEXT (Titel, Beschreibung) darfst du weiterhin die
  wahrscheinlichste Produktvariante wählen und die Annahme in "assumptions"
  vermerken — dort ist eine begründete Annahme besser als ein generisches
  Listing.

═══════════════════════════════════════════════════════════════════════
1  PRODUKTERKENNUNG
═══════════════════════════════════════════════════════════════════════
- Identifiziere Marke und exaktes Modell aus allen visuellen Hinweisen: Logos,
  Schriftzüge, Kamera-Anordnung, Anschlüsse, Tasten, Form, Material, Farbe,
  Größenverhältnisse (eine Hand im Bild ist eine Größenreferenz).
- Nutze dein Produktwissen aktiv: bei Smartphones verraten Kamera-Layout und
  Gehäusedetails die Generation, bei Sneakern Silhouette und Details, bei
  Konsolen Anschlüsse und Gehäuse.
- Sind mehrere Modelle plausibel: wähle für den Listing-Text das
  wahrscheinlichste und vermerke die Annahme knapp in "assumptions". Schreibe
  niemals ein generisches Listing ("Smartphone schwarz"), wenn das Modell
  erkennbar ist.
- Angaben des Verkäufers haben IMMER Vorrang vor der Bilderkennung — bei
  Preis, Menge, Zustand, Modell und Format.

═══════════════════════════════════════════════════════════════════════
2  KONFIDENZ — bei jeder strukturierten Aussage
═══════════════════════════════════════════════════════════════════════
Jedes Konfidenz-Feld nimmt genau einen dieser drei Werte:
  "hoch"    Direkt und eindeutig im Bild lesbar oder vom Verkäufer genannt.
  "mittel"  Erkennbar, aber teilweise verdeckt, gespiegelt, unscharf oder aus
            zwei Hinweisen zusammengesetzt.
  "niedrig" Vermutet. Der Wert könnte falsch sein.
Ist ein Wert null, ist auch seine Konfidenz null.
Nutze "niedrig" großzügig. Ein ehrliches "niedrig" ist wertvoll, ein
geschöntes "hoch" ist Sabotage.

═══════════════════════════════════════════════════════════════════════
3  WARENART — item_class
═══════════════════════════════════════════════════════════════════════
Bestimme zuerst, was für ein Stück das ist. Danach richtet sich, in welchem
Markt der Wert gesucht wird. Eine falsche Warenart führt garantiert zu einem
falschen Preis, egal wie gut der Rest stimmt.

  "tcg_card"        Einzelne Sammelkarte eines Kartenspiels
                    (Pokémon, Magic, Yu-Gi-Oh, One Piece, Lorcana, Digimon)
  "sport_card"      Einzelne Sportkarte (Panini, Topps, Upper Deck)
  "comic_manga"     Heft, Manga-Band, Comic, Graphic Novel, Buch
  "video_game"      Videospiel, Konsole, Zubehör
  "sealed_product"  Ungeöffnete Verpackung: Display, Booster-Box, ETB, Tin
  "coin"            Münze, Medaille, Banknote
  "toy"             Figur, Plüsch, Modell, Statue
  "electronics"     Handy, Rechner, Audio, Foto, Haushaltselektronik
  "fashion"         Kleidung, Schuhe, Taschen, Uhren, Schmuck
  "home"            Möbel, Haushalt, Deko, Garten
  "tool"            Werkzeug, Maschinen, Kfz-Teile
  "other"           Alles andere

UNTERSCHEIDUNGSHILFEN
- Ein Label mit Titel, Heft- oder Bandnummer, Erscheinungsdatum und Verlag
  gehört zu "comic_manga". Ein Label mit Jahr, Set, Kartenname und
  Kartennummer gehört zu "tcg_card" oder "sport_card".
- "One Piece" ist sowohl ein Kartenspiel als auch eine Manga-Reihe. Ein
  gedrucktes Buch mit Rücken und Seiten ist NIE eine Karte, auch wenn die
  Reihe ein Kartenspiel hat. Achte auf Buchrücken, ISBN, Verlagslogo,
  Seitenschnitt.
- Eine Nummer ist nicht gleich Nummer: "Band 103" ist eine Bandnummer
  (number_kind "volume"), "OP01-016" eine Kartennummer ("card_number"),
  "#4" bei einem Comic eine Heftnummer ("issue"). Trage die Art immer ein.

═══════════════════════════════════════════════════════════════════════
4  VERPACKUNGSART — holder
═══════════════════════════════════════════════════════════════════════
  "slab"    In einem verschweißten Bewertungsgehäuse mit BEDRUCKTEM,
            undurchsichtigem Label.
  "sleeve"  In Hülle, Toploader oder Magnetholder — durchsichtig, kein
            bedrucktes Label.
  "raw"     Ohne Schutz.
  "sealed"  Originalversiegelte Verpackung.
  "none"    Warenart, bei der die Frage nicht sinnvoll ist.

Das entscheidende Merkmal für "slab" ist das opake, bedruckte Label am Rand
des Gehäuses. Ein Toploader ist über die ganze Fläche durchsichtig und sagt
NICHTS über den Zustand aus. Er ist niemals ein Grading.

═══════════════════════════════════════════════════════════════════════
5  GRADING — die strengsten Regeln dieses Prompts
═══════════════════════════════════════════════════════════════════════

5.1  Fülle den Block "slab" ausschließlich, wenn holder == "slab". Sonst ist
     "slab" null. Eine lose Karte, eine Karte in einer Hülle und eine
     versiegelte Box sind NIEMALS gegradet.

5.2  POSITIVE EVIDENZ. Setze positive_evidence nur bei allen dreien:
     (a) ein Grader-Name oder -Logo ist erkennbar, UND
     (b) eine Note ist lesbar, UND
     (c) eine Zertifikatsnummer ist lesbar ODER ein QR- oder Barcode ist
         sichtbar.
     Fehlt eines, ist positive_evidence false. Fülle die lesbaren Felder
     trotzdem, lasse den Rest null.

5.3  ERFUNDENE ZERTIFIKATSNUMMERN SIND DER SCHWERSTE FEHLER.
     Die Nummer wird später maschinell nachgeschlagen. Eine erfundene Nummer
     liefert entweder einen Fehlschlag oder, schlimmer, den Datensatz eines
     fremden Stücks.
     - Gib eine Nummer NUR aus, wenn du jede Ziffer im Bild siehst.
     - Ergänze nie fehlende Stellen, auch wenn du die übliche Länge kennst.
     - Nimm nie eine Nummer aus einem anderen Foto, aus deinem Vorwissen oder
       aus einem Beispiel in diesen Anweisungen.
     - Ist ein Teil verdeckt, gespiegelt oder unscharf: cert_number = null,
       cert_channel = null.
     - Einen QR- oder Barcode, den du nicht sicher lesen kannst, vermerkst du
       in "assumptions" und setzt beide Felder null.
     - cert_verified ist IMMER false. Nur ein echter Lookup ändert das.

5.4  KORREKTE GRADER-KÜRZEL — nur diese Werte sind erlaubt
     PSA   Professional Sports Authenticator
     BGS   Beckett Grading Services, moderne Karten, mit Subgrades
     BVG   Beckett Vintage Grading, ohne Subgrades
     BCCG  Beckett Collectors Club Grading, eigene, deutlich mildere Skala
     BAS   Beckett Authentication Services, nur Autogramme
     CGC   Certified Guaranty Company
     CBCS  Certified Bookstore Comic Services
     SGC   Sportscard Guaranty
     WATA  Wata Games
     VGA   Video Game Authority
     CGA   Collectible Grading Authority
     TAG · ACE · MANA · CSG

     "Beckett" ist der FIRMENNAME, kein Kürzel. Ein Beckett-Slab kann BGS,
     BVG, BCCG oder BAS sein. Entscheide am Label:
       vier Subgrades (Centering/Corners/Edges/Surface) sichtbar → BGS
       Vintage-Karte ohne Subgrades                              → BVG
       Aufschrift "Collectors Club" oder "BCCG"                   → BCCG
       nur Autogramm-Bestätigung ohne Kartennote                  → BAS
     Nicht entscheidbar: grader = null, grader_raw = "BECKETT",
     confidence.grader = null. Rate NICHT auf BGS.
     Schreibe in grader_raw immer wörtlich, was auf dem Label steht.

5.5  SKALA — scale
     psa_10 | bgs_10 | bvg_10 | bccg_10 | cgc_cards_10 | cgc_comics_10 |
     sgc_10 | wata_10 | vga_100 | unknown
     CGC bewertet Karten und Comics nach unterschiedlichen Skalen. Eine 9.8
     gibt es bei CGC-COMICS, nicht bei CGC-Karten. Siehst du bei einer Karte
     eine 9.8, prüfe, ob es in Wahrheit ein Comic ist.
     Unsicher: scale = "unknown".

5.6  NOTE. Die Gesamtnote ist die typografisch größte, hervorgehobene Zahl.
     Übernimm sie wörtlich als Text ("9.0" bleibt "9.0") und zusätzlich als
     Zahl in grade_numeric. Verwechsle sie nie mit einer Subgrade, einer
     Jahreszahl, einer Kartennummer oder einem Teil der Zertifikatsnummer.

5.7  SUBGRADES — alle vier oder keiner. Kannst du nicht alle vier sicher
     lesen, ist subgrades null. Gib niemals eine Teilmenge aus. BVG-Slabs
     haben keine Subgrades.

5.8  WATA vergibt ZWEI Werte: eine Box-Note und ein Seal Rating (C bis A++).
     Lies beide. Fehlt eines, setze es null. Übernimm nie das Seal Rating
     als Note.

═══════════════════════════════════════════════════════════════════════
6  IDENTITÄT, SPRACHE, SET, NUMMER
═══════════════════════════════════════════════════════════════════════
6.1  SPRACHE aus gedrucktem Text auf dem Stück selbst: Kartenname, Regeltext,
     Verlagsangabe, Labelaufschrift. Kein Text lesbar → language = null.
     Schließe NIE von Marke, Reihe oder Vorwissen auf die Sprache.
     Japanische Ausgaben haben eigene Set-Codes, die nicht auf englische Sets
     abbilden. Rechne nie um.
6.2  SET UND SET-CODE exakt so übernehmen, wie sie auf dem Stück oder Label
     stehen. Nicht übersetzen, nicht kürzen, nicht vervollständigen. Nicht
     lesbar → null. Leite den Set-Namen niemals aus dem Kartennamen ab —
     dieselbe Karte erscheint in mehreren Sets zu sehr verschiedenen Preisen.
6.3  NUMMER. Bei "199/165": number = "199", set_total = "165", number_kind =
     "card_number". Führende Nullen entfernen. Bei Manga und Comics die Band-
     bzw. Heftnummer mit number_kind "volume" bzw. "issue".
     Die Nummer ist der wichtigste Einzelwert für die Preissuche. Verdeckt,
     angeschnitten oder unscharf: number = null. Eine falsche Nummer ist
     schlimmer als keine.
6.4  AUFLAGE UND VARIANTE ("1st Edition", "Unlimited", "Holo", "Reverse
     Holo", "Promo", "Full Art") nur eintragen, wenn das Zeichen sichtbar
     ist. Das Fehlen eines Stempels ist kein Beweis für "Unlimited".

═══════════════════════════════════════════════════════════════════════
7  DER LISTING-ENTWURF
═══════════════════════════════════════════════════════════════════════
7.1  TITEL-FORMEL (immer exakt diese Reihenfolge, deutsch, Ziel 60-70
     Zeichen, max 80):
     Marke → Produkt/Modell → Variante/Setnummer → wichtigstes Merkmal →
     Größe/Farbe/Sprache → Grading oder Zustandskeyword.
     Natürliche Käufersprache. Keine Wörter in Großbuchstaben, keine Füller
     wie TOP, RAR, L@@K, WOW, keine Sonderzeichen-Deko, kein Keyword-Stuffing.
     Kategorie-Muster:
       Trading Card: "[TCG] [Kartenname] [Setnr.] [Set] [Sprache] [Grade|NM]"
                     z. B. "Pokémon Glurak ex 199/165 151 Deutsch PSA 10"
       Manga/Comic:  "[Reihe] Band [Nr.] [Art] [Sprache] [Verlag] [Grade]"
                     z. B. "One Piece Band 103 Manga Deutsch Carlsen BGS 9.0"
       Elektronik:   "[Marke] [Modell] [Speicher/Variante] [Farbe] [Zustand]"
       Fashion:      "[Marke] [Modell] [Colorway] [Größe]"
       Plüsch:       "[Marke/Serie] [Figur] [Größe cm] [Merkmal]"
     Bei einem Slab: das korrekte Kürzel aus 5.4 plus Note ans ENDE, exakt in
     der Form "BGS 9.0". Ist grader null, schreibe KEIN Kürzel in den Titel.

7.2  BESCHREIBUNG (immer exakt diese HTML-Struktur, nichts anderes):
     <p>[2-3 Sätze: was ist es, was macht es besonders — ehrlich, keine
     Superlative]</p>
     <p><b>Details</b></p>
     <ul><li>[je ein Fakt pro Zeile: Marke, Modell, Set/Nummer, Größe, Farbe,
     Sprache]</li></ul>
     <p><b>Zustand</b></p>
     <p>[ehrliche Zustandsbeschreibung anhand der Fotos; bei Neuware "Neu und
     ungeöffnet"]</p>
     <p><b>Versand</b></p>
     <p>Schneller Versand, sicher und sorgfältig verpackt.</p>
     Nur beschreiben, was auf den Fotos erkennbar oder vom Verkäufer angegeben
     ist. Keine erfundenen Details (Speichergröße, Jahr, OVP, Auflage).
     Unbekanntes weglassen, nicht umschreiben.

7.3  SPRACHE UND STIL für title, subtitle, description_html,
     condition_description und assumptions:
     - Keine Ausrufezeichen.
     - Keine Emojis, keine Symbolzeichen als Schmuck.
     - Kein "Wir". Sachlich formulieren: statt "Wir versenden schnell" →
       "Schneller Versand".
     - Sag "Stück", nicht "Artikel", "Item" oder "Produkt", wenn das
       Sammlungsobjekt gemeint ist.
     - Sag "listen" für das Einstellen bei eBay.
     - Sag "Marktwert" für den ermittelten Preis, nicht "Schätzwert" oder
       "Preisvorschlag".
     - Sag "tippen" für eine Bedienhandlung, nicht "klicken".
     - Keine Superlative, keine Werbesprache, keine Dringlichkeit.

7.4  MERKMALE (aspects). Nur belegbare Merkmale.
     Bei Videospielen, DVDs und Blu-rays: NUR wenn ein deutsches USK-Logo auf
     dem Cover zu sehen ist (Farbe: weiß/grün=0, gelb=6, blau=12, rot=16,
     schwarz=18), das Merkmal "USK-Einstufung": ["USK ab X Jahren"] setzen —
     das verhindert eBays 18-Sperre bei niedriger eingestuften Spielen. OHNE
     sichtbares deutsches USK-Logo (US-Importe mit ESRB, japanische mit CERO)
     das Merkmal KOMPLETT WEGLASSEN — kein Feld "USK", keine Ersatzangabe,
     auch nicht "nicht vorhanden" oder "keine". Niemals ESRB, PEGI oder CERO
     in ein Alters-Merkmal schreiben: eBay.de liest das als
     Erwachsenen-Kennzeichen und sperrt das Listing.

7.5  SUCHANFRAGEN — drei getrennte Queries für drei verschiedene Systeme:
     queries.sold    Suche in VERKAUFSTITELN. Kurz, mit Nummer und, falls
                     vorhanden, Grader und Note. Keine Füllwörter. Bei Manga
                     und Comics gehört die Bandnummer zwingend hinein.
     queries.catalog Suche in einem PRODUKTKATALOG. Offizieller Produktname
                     ohne Zustands- und Gradingangaben.
     queries.ebay    Suche in AKTIVEN Angeboten, deutsch, wie ein Käufer
                     suchen würde.
     search_query_for_pricing setzt du identisch zu queries.sold.

7.6  MENGE, FORMAT, PREIS. quantity > 1 nur, wenn der Verkäufer ausdrücklich
     mehrere identische Stücke oder Sets in EINEM Listing anbietet; Titel und
     Beschreibung beschreiben dann EIN Exemplar, der Preis gilt pro Exemplar.
     format ist FIXED_PRICE, außer der Verkäufer verlangt ausdrücklich eine
     Auktion. user_price und best_offer nur bei ausdrücklicher Nennung.

7.7  UNSICHERHEIT. uncertain = true und eine question NUR, wenn du das Stück
     überhaupt nicht zuordnen kannst oder eine Information fehlt, ohne die das
     Listing irreführend wäre (etwa echt gegen Replika bei Luxusware).
     Varianten-Unsicherheiten gehören in die Konfidenz-Felder und in
     "assumptions", nicht in eine blockierende Rückfrage.

═══════════════════════════════════════════════════════════════════════
8  AUSGABEFORMAT UND SCHEMA
═══════════════════════════════════════════════════════════════════════
Antworte mit genau einem JSON-Objekt. Nichts davor, nichts danach. Keine
Markdown-Fences. Keine Kommentare, weder // noch /* */ noch #. Keine
nachgestellten Kommata. Fehlende Werte als JSON-null, nicht als "null", "-",
"unbekannt" oder "".

{
  "title": "",
  "subtitle": null,
  "description_html": "",
  "category_query": "",
  "condition": "NEW | USED_EXCELLENT | USED_GOOD | USED_ACCEPTABLE",
  "condition_description": null,
  "aspects": {"Marke": [""], "Modell": [""]},
  "search_query_for_pricing": "",
  "estimated_price_range_eur": {"low": 0.0, "high": 0.0},
  "user_price": null,
  "best_offer": null,
  "format": "FIXED_PRICE",
  "auction_days": 7,
  "quantity": 1,
  "main_image_index": 0,
  "graded_info": null,
  "assumptions": null,
  "estimated_weight_grams": 0,
  "uncertain": false,
  "question": null,

  "classification": {
    "item_class": "",
    "holder": "",
    "language": null,
    "confidence": {"item_class": "", "holder": "", "language": null},
    "evidence": []
  },

  "slab": null,

  "identity": {
    "title_native": null, "franchise": null, "series": null,
    "number": null, "number_kind": null,
    "set_name": null, "set_code": null, "set_total": null,
    "publisher": null, "release_year": null,
    "edition": null, "variant": null, "isbn": null, "upc": null,
    "confidence": {"number": null, "set_code": null, "release_year": null,
                   "publisher": null, "edition": null, "variant": null}
  },

  "queries": {"sold": "", "catalog": "", "ebay": ""},
  "photos_meta": {"label_index": null, "front_index": 0, "back_index": null}
}

Wenn holder == "slab", hat "slab" diese Form:
{
  "grader": null, "grader_raw": null, "scale": "unknown",
  "grade": null, "grade_numeric": null,
  "subgrades": null, "seal_rating": null, "label_color": null,
  "cert_number": null, "cert_channel": null,
  "cert_verified": false, "positive_evidence": false,
  "confidence": {"grader": null, "grade": null, "cert_number": null,
                 "subgrades": null}
}

"graded_info" füllst du ausschließlich aus "slab" und nur dann, wenn
positive_evidence true ist:
  {"grader": <slab.grader>, "grade": <slab.grade>,
   "cert_number": <slab.cert_number>}
Sonst ist graded_info null.

estimated_price_range_eur ist Pflicht und bleibt deine ehrliche Einschätzung
aus Produktwissen: wofür verkauft sich GENAU dieses Stück in diesem Zustand
realistisch auf eBay. Alltagsware ehrlich niedrig ansetzen (eine Flasche
Apfelschorle ist 1-2 Euro, kein Sammlerstück), bei Sammlerware den echten
Marktwert. Diese Spanne ist ein Plausibilitäts-Anker gegen falsche
Vergleichstreffer.

═══════════════════════════════════════════════════════════════════════
9  LETZTE PRÜFUNG VOR DER ANTWORT
═══════════════════════════════════════════════════════════════════════
  1. Gültiges JSON, ohne Kommentare, ohne Text davor oder danach?
  2. Steht in item_class die Warenart, die das Stück TATSÄCHLICH ist — nicht
     die, die die Reihe nahelegt?
  3. Ist jede Zertifikatsnummer im Bild vollständig lesbar? Wenn nicht: null.
  4. Ist jedes Grader-Kürzel aus der Liste in 5.4? Steht nirgends "Beckett"
     als Kürzel?
  5. Sind Subgrades entweder vollständig oder null?
  6. Hat jede nicht-null-Aussage eine Konfidenz und jede null-Aussage
     Konfidenz null?
  7. Enthält ein Endtext ein Ausrufezeichen, ein Emoji oder ein "Wir"?
  8. Steht irgendwo in classification, slab oder identity ein Wert, den du
     nicht am Bild oder in den Angaben des Verkäufers belegen kannst? Dann
     ersetze ihn durch null.
```

## 4.1 Was der neue Prompt behebt

| Problem heute | Behebung |
|---|---|
| „Wähle das WAHRSCHEINLICHSTE" gilt für alles (`:40-41`) | Das Grundgesetz trennt: für Identität und Grading gilt null, für den Listing-Text bleibt die begründete Annahme erlaubt. Der Telegram-Weg verliert nichts. |
| Keine Warenart | `classification.item_class` mit expliziten Unterscheidungshilfen und deinem One-Piece-Fall als benanntem Beispiel. Darauf setzt das Domänen-Gate auf. |
| Ein `//`-Kommentar mitten im JSON-Beispiel (`:76`) | Schema in Abschnitt 8 ist reines, gültiges JSON. Alle Erklärungen stehen in den nummerierten Regeln davor. |
| Grading zu dünn (`:85`) | `scale`, `subgrades` (alle oder keiner), `seal_rating`, `cert_channel`, `grader_raw`, `positive_evidence`, Konfidenz je Feld. |
| Kürzel unvollständig, `beckett→BGS` falsch | Vollständige Whitelist plus Entscheidungsbaum. Bei Unentscheidbarkeit `grader: null`, nicht BGS. |
| Titel-Regex zerstört Titel (`:158-159`) | Der Prompt schreibt das korrekte Kürzel selbst in der Form „BGS 9.0" ans Titelende. Damit greift `re.search(rf"\b{richtig}\b", …)` bei `:155` und die gefährliche `re.sub`-Zeile wird nicht mehr erreicht. Eine Entschärfung ohne Code-Änderung — der eigentliche Regex-Fix bleibt trotzdem nötig. |
| Eine Query für drei Systeme | `queries.sold`, `.catalog`, `.ebay`. `search_query_for_pricing` bleibt als Alias, damit `app_api.py:799`, `:821`, `:1008` unverändert laufen. |
| Wording-Regeln fehlen | Abschnitt 7.3 nennt sie einzeln, Prüfpunkt 7 kontrolliert sie. |

## 4.2 Zwei Hinweise zur Einbindung

**Prompt-Caching aktivieren.** `_call` (`claude_client.py:253-259`) übergibt `system=SYSTEM_PROMPT` als reinen String. Der neue Prompt ist länger; bei zehntausenden Scans zahlt jeder Aufruf den vollen Eingabepreis. Blockform genügt:

```python
system=[{"type": "text", "text": SYSTEM_PROMPT,
         "cache_control": {"type": "ephemeral"}}],
```

Wichtig, damit die Erwartung stimmt: gecacht wird nur der System-Block. Die Bilder stehen in `messages`, also **nach** dem Breakpoint — von rund 6.900 Input-Tokens je Scan sind etwa 1.900–4.400 cachebar. Ersparnis in der Größenordnung 0,005 $ je Scan, nicht die oft zitierten 90 %. Der Prompt muss dafür **byte-identisch** bleiben: kein Datum, keine Item-ID, kein `f"…{version}…"` hineininterpolieren. Kontrolle über `resp.usage.cache_read_input_tokens`; das Log bei `:261-262` müsste das mitschreiben.

**`parse_listing_json` bleibt lauffähig.** Die Pflichtfelder (`:127-130`) liefert das neue Schema alle. Sinnvolle Ergänzungen dort, in dieser Reihenfolge: (a) `classification.item_class` gegen die Enum-Liste prüfen, sonst auf `"other"` setzen; (b) `slab.grader` gegen die Whitelist prüfen, sonst `None`; (c) `graded_info = None` erzwingen, wenn `holder != "slab"`; (d) die `re.sub`-Zeile `:158` entschärfen.

**Nebenbefund:** `CLAUDE_MAX_EDGE = 2048` (`claude_client.py:27`) mit dem Kommentar „Opus 4.7+ liest hochauflösend" — das konfigurierte Modell ist aber `claude-sonnet-4-6` (`bot/config.py:113`), das bei 1568 px deckelt. Die Extrapixel werden serverseitig weggerechnet. Kostet nichts, aber Encoding-Zeit und Bandbreite.

---

# Teil 5 — Umsetzungsplan

Sortiert nach Hebel geteilt durch Risiko. Aufwand: S = unter einem halben Tag, M = ein bis zwei Tage, L = mehr.

## Stufe 0 — Netz spannen (vor allem anderen)

| | |
|---|---|
| **Was** | (a) Tests für `catalog.py` schreiben: `card_key_of` deterministisch, `grade_bucket`-Normalisierung, Kaskaden-Reihenfolge, jeder Wächter einzeln, TTL-Staffelung. Gegen eine Wegwerf-DB im Scratchpad, **nie** gegen `data.db`. (b) `requirements.txt` um `opencv-python`, `rembg`, `numpy` ergänzen, mit Versionspins. (c) Feature-Flag `SERO_PIPELINE_V2` nach dem Vorbild von `SERO_EBAY_INSIGHTS` (`ebay_insights.py:28`). (d) `backup.sh` einmal laufen lassen und den Ablageort notieren. |
| **Aufwand** | M |
| **Kaputt-Risiko** | Praktisch keins — nur neue Dateien. Die `requirements.txt`-Ergänzung könnte auf einem Server eine andere OpenCV-Version ziehen; deshalb pinnen auf die Version im venv (cv2 5.0.0, rembg 2.0.76, numpy 2.4.6). |
| **Prüfung** | `pytest` komplett grün, vorher und nachher gleiche Anzahl. Neue Tests decken die Kaskade ab. |

**Ohne diese Stufe ist alles Folgende unkontrolliert.** 15 Testdateien, keine für das Modul, das den global geteilten Preis schreibt.

## Stufe 1 — Der Schlüssel (größter Hebel im ganzen Plan)

| | |
|---|---|
| **Was** | `app_api.py:790` und `:1000` von `setdefault` auf explizite Zuweisung. `card_key_of` (`catalog.py:48-60`) deterministisch, `uuid4` raus, `item_class`/`language`/`edition` in den Hash. Vertrag bleibt: immer ein String, nie `None`. |
| **Aufwand** | S (rund fünf Zeilen) |
| **Kaputt-Risiko** | Bestehende `solo:`- und `h:`-Zeilen werden nicht mehr getroffen — die 239 Altzeilen bleiben als Karteileichen liegen und werden neu aufgebaut. Das ist gewollt, kostet aber je Stück einmalig eine frische Abfrage gegen 130point und PriceCharting. Bei 13 Stücken irrelevant. Kollisionsrisiko des Hashs: `sha1[:16]` bei realistischen Volumina vernachlässigbar. |
| **Prüfung** | Ein Stück zweimal refreshen und in `card_prices` zählen: vorher zwei neue Zeilen, nachher eine. `SELECT COUNT(*) FROM cards` darf beim zweiten Lauf nicht steigen. Die beiden GTA-Vice-City-Stücke (`e34807d4`, `c9208739`) müssen danach denselben Schlüssel tragen. |

## Stufe 2 — Domänen-Gate für PriceCharting (direkter Fix der 603 €)

| | |
|---|---|
| **Was** | `lookup_pc` (`pricecharting.py:76-121`) auf `/api/products?q=` umstellen, bis zu 20 Kandidaten holen, `domain_of_pc(console-name)` gegen `item_class` prüfen, Kandidaten ohne passende Domäne verwerfen. Für den Gewinner einmal `/api/product?id=`. Cache zweistufig. |
| **Aufwand** | M |
| **Kaputt-Risiko** | Zwei PriceCharting-Calls statt einem — das Limit von 1/s ist ein Konto-Limit, bei aktivem Katalog-Cache tragbar, bei einem Massen-Refresh nicht. Deshalb erst nach Stufe 1. Wenn `item_class` fehlt (v0-Datensätze), muss das Gate **durchlassen**, nicht sperren — sonst verlieren Altstücke ihren Preis. |
| **Prüfung** | Neuen Test: Item `comic_manga` gegen Kandidat mit `console-name = "One Piece Romance Dawn"` → verworfen. Item `comic_manga` gegen `"Comic Books One Piece"` → akzeptiert. Danach das echte Manga-Stück refreshen und prüfen, dass der Wert nicht mehr 603 € ist. |

## Stufe 3 — Grader-Kanon und Skalen zentral

| | |
|---|---|
| **Was** | `web/slab.py` neu: `GRADER_KANON`, `SKALEN`, `CERT_FORMAT`, `normalize_grade`, `scale_ok`, `grade_rank`. `_GRADER_KUERZEL` (`claude_client.py:139-142`) importiert von dort. `grade_bucket` (`catalog.py:81-84`) normalisiert. Die Regex `claude_client.py:158-159` entschärfen: nur ersetzen, wenn das getroffene Wort selbst ein bekanntes Grader-Kürzel oder eine plausible Verschreibung davon ist. |
| **Aufwand** | S bis M |
| **Kaputt-Risiko** | **Die Bucket-Umbenennung ist der heikle Teil.** `"BECKETT 9.0"` wird zu `"BGS 9"` — die alte Zeile wird verwaist, der neue Bucket ist leer und muss einmal neu geholt werden. Bei 247 Zeilen unkritisch, bei 100.000 nicht. Deshalb **jetzt** machen, nicht später. Der Regex-Fix könnte einen Titel unkorrigiert lassen, den er heute korrigiert — das ist besser als „Band" zu zerstören. |
| **Prüfung** | Test: `_grader_im_titel_richtigstellen({"title": "One Piece Band 9 Beckett 9.0", "graded_info": {"grader": "beckett"}})` darf `Band` **nicht** anfassen. Test: `grade_bucket({"grader": "Beckett", "grade": "9.0"})` und `{"grader": "BGS", "grade": "9"}` ergeben denselben String. |

## Stufe 4 — Anker und Wächter scharf schalten

| | |
|---|---|
| **Was** | Anker-Regel aus 2.7 in `refresh_price`. Stufe 4 der Kaskade (`catalog.py:196`) auch für gegradete Stücke. Ausreißer-Wächter (`:212`) von `n_avg < 2` auf `n_avg < 4` plus Streuungsmaß. Wächter C (`app_api.py:833-845`) zusätzlich in `analyze_collection_item`. Jede Korrektur schreibt in die Katalogzeile zurück. |
| **Aufwand** | M |
| **Kaputt-Risiko** | **Hier ändern sich bestehende Preise.** Der Ausreißer-Wächter feuert erstmals überhaupt und kann Werte korrigieren, die heute stehen bleiben. Der Rückschreib-Pfad ist ein neuer Schreibzugriff auf die geteilte Tabelle. |
| **Prüfung** | Erst nur loggen, nicht korrigieren: eine Runde mit `SERO_PIPELINE_V2=0` und einer Log-Zeile je Fall, die der Wächter treffen *würde*. Log ansehen. Dann scharf schalten. |

## Stufe 5 — `price_state` und der ehrliche Anzeigezustand

| | |
|---|---|
| **Was** | `valuation`-Block schreiben, `PREIS_FELDER` um `"valuation"` und `"schema_version"` ergänzen, `projiziere_legacy` einbauen, `_DETAIL_ONLY` (`app_api.py:566`) um `valuation` erweitern, drei flache Schlüssel in `item_public`. In der PWA: bei `spanne` eine Spanne statt einer Zahl, bei `unbekannt` „Wert unbekannt — <Grund>". `market_source` im Listing-Preset (`:2252`) trägt den Zustand mit. |
| **Aufwand** | M bis L (PWA-Anteil) |
| **Kaputt-Risiko** | Wenn `PREIS_FELDER` vergessen wird, verschwindet `valuation` beim nächsten Lauf **stillschweigend**. Wenn `valuation` in die Listenantwort gerät, läuft der localStorage-Cache der PWA voll und die App wird ohne Fehlermeldung langsam. `est_value` darf unter keinen Umständen `None` werden. |
| **Prüfung** | Nach einem Refresh prüfen, dass `valuation` im gespeicherten JSON steht. Größe der Listenantwort vor und nach der Änderung messen — Differenz muss nahe null sein. Ein Stück mit `price_state = "unbekannt"` in den Listing-Dialog geben und kontrollieren, dass ein Preis vorgeschlagen wird und `price_basis` die Herkunft nennt. |

## Stufe 6 — Entity Matching mit Konfidenz

| | |
|---|---|
| **Was** | `web/match.py`: `token_set_ratio` mit `difflib`, Bit-Scoring, Ambiguitätsregel, `identity_conf`. Bildvergleich bleibt draußen. |
| **Aufwand** | M |
| **Kaputt-Risiko** | Zu strenge Schwellen lassen legitime Treffer fallen und drücken mehr Stücke in `spanne`. Die Gewichte sind Startwerte, keine gemessenen. |
| **Prüfung** | Erst im Schattenbetrieb: `P` berechnen und loggen, aber die Entscheidung noch nicht darauf stützen. 50 Stücke aus der Historie ansehen, dann die Schwelle festziehen. Später: 200–500 gelabelte Paare, logistische Regression, Koeffizienten durch `ln 2` für Bits, Platt-Scaling auf einem Holdout (nicht isotonisch, das überanpasst bei der Datenmenge). |

## Stufe 7 — EU-Zweig filtern und dritte Grade-Leiter

| | |
|---|---|
| **Was** | `research_price(…, titel_filter=None)` (`browse.py:19`), Filter **nur** in der `eu_probe`-Lambda; `check_price_plausibility` in dieselbe Lambda. Comics-Leiter in `_grade_fields`. Docstring `pricecharting.py:6` korrigieren. |
| **Aufwand** | S bis M |
| **Kaputt-Risiko** | Wenn der Filter versehentlich auch für `item["market"]` gilt, verliert die Marktanzeige ihre Beispiele und der Telegram-Bot seinen Preisanker für Allgemeinware — `sold.fits` verlangt jedes Ziffern-Token im Titel, was bei „Bosch GSR 18V-55 2x2.0Ah" fast nie zutrifft. Der `eu_probe`-Median wird seltener zustande kommen; die Kaskade fällt dann öfter auf PriceCharting oder `unbekannt`. |
| **Prüfung** | Test, dass `item["market"]["count"]` bei einem Werkzeug-Stück unverändert bleibt. `tests/test_sold_filter.py` um Nicht-Karten-Fälle erweitern. |

## Stufe 8 — QR/Barcode, Label-Crop, PSA

| | |
|---|---|
| **Was** | `decode_codes()` mit `cv2` auf dem entzerrten Slab, `label_crop()` als eigenes Feld, PSA Public API mit permanentem Cert-Cache in einer neuen Tabelle `certs`. |
| **Aufwand** | M |
| **Kaputt-Risiko** | `label_crop_path` darf nie in `photos[]` — sonst wird der Label-Ausschnitt zum eBay-Hauptbild. PSA-Free-Tier ist bei 100 Calls/Tag schnell erschöpft; ohne Cache und Auslöse-Regel steht der Kanal nach einer Stunde. |
| **Prüfung** | Test, dass `photos` nach dem Crop-Lauf dieselbe Länge und dieselben Einträge hat. Zähler für PSA-Calls je Tag ins Log. |

## Bewusst NICHT jetzt

| Vorschlag | Warum nicht |
|---|---|
| **`card_key_of` gibt `None` zurück** | `cards.card_key` ist ein TEXT-PK und SQLite erlaubt dort NULL — jeder Lauf schriebe eine neue NULL-Zeile. `card_prices.card_key` ist NOT NULL und wirft `IntegrityError`, der im `except Exception` (`app_api.py:802`, `:1011`) verschwindet. Der Fehler wäre unsichtbar und schlimmer als heute. |
| **`est_value = None` bei unsicherem Wert** | Bricht `apply_price_rule` (`bot/main.py:227`), die Preis-Modi (`app_api.py:2258`, `:2260`), den Preisalarm (`:583`), den Preisverlauf (`:598`), den Sammlungswert (`:1677`, `:2928`) und die PWA-Sortierung. Es ist exakt der Fehler vom 02.08. Ehrlichkeit kommt über `price_state`. |
| **Cardmarket über echte Verkäufe stellen** | Widerspricht deinem Entscheid vom 30.07. (`STATUS.md:336`) und dem Satz auf dem Anmeldeschirm „Marktwert aus echten eBay-Verkäufen" (`STATUS.md:432`, `sero.js:509`). E3 wird Referenz in den Wächtern, nicht Gewinner. |
| **Prompt komplett durch einen Sammler-Prompt ersetzen** | `bot/main.py:526` nutzt denselben Analyzer für dein ganzes Sortiment. Elektronik, Fashion, Möbel, Werkzeug würden auf `other` fallen. Der Prompt in Teil 4 erweitert, statt zu ersetzen. |
| **Interpolation zwischen Grade-Stufen zurückdrehen** | `_zwischenstufe` (`pricecharting.py:49-73`) tut bereits das Konservative: nur zwischen zwei vorhandenen, plausibel sortierten Stützstellen. Dein Grundsatz „lieber grob und belegt als fein und geraten" (`STATUS.md:542`) ist erfüllt. Ergänzt wird nur `grade_exact: false` im Label. |
| **Eigener Vision-Call für die Klassifizierung** | `detect_card` läuft heute nur innerhalb `slab_recut` (`cardscan.py:292`), also selten. Ein eigener Call je Scan kostet rund 0,02 $ zusätzlich, bei 10.000 Scans 200 $ am Tag — für eine Information, die der bestehende Call mitliefern kann. |
| **Monotonie-Sweeper im 12-h-Autolauf** | `card_prices` ist global ohne Mandant. `force=True` umgeht die TTL und würde 247 Zeilen mit 8 s Abstand gegen 130point feuern: rund 33 Minuten Vollauslastung, alle Live-Scans im 600-s-Cooldown. Separates Skript, `--dry-run` als Default, Deckel bei 50 Zeilen je Lauf. |
| **pHash und rapidfuzz im Scoring** | Nicht installiert, `requirements.txt` ist schon jetzt unvollständig. Das Bit-Gewicht ist ohnehin 0, solange kein Katalogbild vorliegt. `difflib` aus der Standardbibliothek reicht für Stufe 1. |
| **Migration der 239 `solo:`-Zeilen** | Getrennter, sorgfältig geprüfter Lauf nach dem Backup, mit Trockenlauf und Rollback — und erst, wenn Stufe 1 bis 4 stehen. Nach dem 02.08. ist das kein Nebenbei-Schritt. |
| **Marketplace Insights erneut beantragen** | Erst mit echtem Produktionsvolumen über den Application Growth Check. Die Ablehnung liegt vor. Ein Produkt darf nicht an einer Genehmigung hängen, die eBay „generally reserved for approved partners only" vergibt. |
| **Scan-Durchsatz (rembg-Semaphore)** | Der echte Deckel bei rund 2.400 Scans/Tag ist ein eigenes Projekt (Worker-Pool, GPU, oder Fremd-Segmentierung), nicht Teil der Preis-Pipeline. Aber es ist der Engpass, der zuerst reißt, wenn die Nutzerzahlen kommen. |

## Was parallel dazu ohne Code passieren muss

1. **PriceCharting anschreiben** und eine Lizenz für Weiterverbreitung erfragen. Ohne sie ist der Launch ab Nutzer 1 ein Vertragsbruch. Die Antwort entscheidet, ob E5 im Baum bleibt.
2. **Fulfillment-Scope nachziehen** (`/verbinden`), damit E1 überhaupt entstehen kann. Das ist die einzige Datenquelle, die bei zehntausenden Nutzern trägt und niemandem gehört außer dir: für jedes über SERO erzeugte Listing der tatsächliche Ausgang — verkauft ja/nein, zu welchem Preis, nach wie vielen Tagen.
3. **IT-Rechtsanwalt** zu 130point und dem eBay User Agreement, bevor die Nutzerzahlen kommen. Die Kosten sind marginal gegen ein gesperrtes Händlerkonto.

---

## Schonungslos zum Schluss

Das Gerüst ist besser, als der Fehlerfall vermuten lässt. `catalog.py` als Konzept, `sold.fits` als strengster Relevanzfilter im System, die TCGdex-Nummernpflicht (`prices.py:154-161`) als Vorbild für alle anderen Quellen, die Schreib-Isolation mit Frisch-Nachlesen, der Papierkorb — das ist gute Arbeit, und nichts davon muss weg.

Kaputt sind drei Dinge, und alle drei sind klein: eine `setdefault`-Zeile, die zweimal vorkommt und 97 % des Katalogs entwertet; eine fehlende Prüfung eines Feldes, das PriceCharting frei Haus mitliefert; und ein Regex, der „Band" für ein Grader-Kürzel hält.

Was strukturell wackelt, ist nicht der Code, sondern das Fundament: zwei deiner drei Preisbeine tragen rechtlich nicht — PriceCharting untersagt genau deine Nutzungsart, 130point ist gescraptes Scraping mit gefälschtem User-Agent gegen einen 403-Filter. Das dritte Bein, die Browse API, ist sauber und wird als Beiwerk behandelt. Der Baum oben ist deshalb bewusst so gebaut, dass er funktioniert, wenn E5 und E2 wegfallen. Er endet dann öfter bei „Wert unbekannt". Das ist kein Mangel des Entwurfs, sondern die ehrliche Abbildung der Datenlage — und genau das, was du verlangt hast.

---

### Dateien im Bestand (nur gelesen, nichts geändert)

`/Users/smorty/ebay-bot/web/catalog.py` · `/Users/smorty/ebay-bot/web/app_api.py` · `/Users/smorty/ebay-bot/web/pricecharting.py` · `/Users/smorty/ebay-bot/web/sold.py` · `/Users/smorty/ebay-bot/web/prices.py` · `/Users/smorty/ebay-bot/web/cardscan.py` · `/Users/smorty/ebay-bot/web/ebay_insights.py` · `/Users/smorty/ebay-bot/bot/claude_client.py` · `/Users/smorty/ebay-bot/bot/main.py` · `/Users/smorty/ebay-bot/bot/config.py` · `/Users/smorty/ebay-bot/bot/ebay/browse.py` · `/Users/smorty/ebay-bot/requirements.txt` · `/Users/smorty/ebay-bot/tests/` · `/Users/smorty/ebay-bot/STATUS.md` · `/Users/smorty/sero-app/web/sero.js` · `/Users/smorty/ebay-bot/data.db` (read-only)

### Neu anzulegen

`/Users/smorty/ebay-bot/web/slab.py` (Grader-Kanon, Skalen, `grade_rank`, QR/Barcode) · `/Users/smorty/ebay-bot/web/match.py` (Domänen-Gate, Bit-Scoring, Ambiguität) · `/Users/smorty/ebay-bot/tests/test_catalog.py` · Sweeper-Skript mit `--dry-run`

### Quellen

- [eBay API Deprecation Status](https://developer.ebay.com/develop/get-started/api-deprecation-status) · [eBay Community: Finding und Shopping API abgeschaltet 2025](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062)
- [Marketplace Insights Overview](https://edp.ebay.com/api-docs/buy/marketplace-insights/static/overview.html) · [Marketplace Insights Methods](https://developer.ebay.com/api-docs/buy/marketplace_insights/resources/methods) · [eBay Community: Zugang verweigert](https://community.ebay.com/t5/eBay-APIs-Talk-to-your-fellow/Marketplace-Insights-API-access/td-p/34838736) · [Application Growth Check](https://developer.ebay.com/api-docs/static/gs_use-the-application-growth.html)
- [Buy Feed API Overview](https://developer.ebay.com/api-docs/buy/feed/static/overview.html) · [Buy APIs Requirements](https://developer.ebay.com/api-docs/buy/static/buy-requirements.html)
- [eBay User Agreement](https://www.ebay.de/help/policies/member-behaviour-policies/user-agreement?id=4259) · [Value Added Resource, Februar 2026](https://www.valueaddedresource.net/ebay-bans-ai-agents-updates-arbitration-user-agreement-feb-2026/) · [The Register](https://www.theregister.com/2026/01/22/ebay_updates_legalese_to_ban/)
- [PriceCharting API-Doku](https://www.pricecharting.com/api-documentation) · [PriceCharting Terms of Service](https://www.pricecharting.com/page/terms-of-service) · [PriceCharting Premium](https://www.pricecharting.com/pricecharting-pro) · [Kategorie Comic Books One Piece](https://www.pricecharting.com/console/comic-books-one-piece) · [SportsCardsPro API-Doku](https://www.sportscardspro.com/api-documentation)
- [PSA Public API](https://www.psacard.com/publicapi/documentation) · [PSA Swagger](https://api.psacard.com/publicapi/swagger/ui/index)
- [CGC Cards Grading Scale](https://www.cgccards.com/card-grading/grading-scale/) · [Beckett Card Lookup](https://www.beckett.com/grading/card-lookup) · [allvintagecards: Beckett-Guide](https://allvintagecards.com/beckett-card-grading-guide/) · [cardgrading.app: Beckett-System](https://cardgrading.app/beckett-grading-system) · [WATA Cert Verification](https://blog.watagames.com/2023/10/17/cert-verification/) · [packz.io: SGC Cert Lookup](https://www.packz.io/blog/sgc-cert-lookup-guide)
- [Ximilar Collectibles Recognition](https://docs.ximilar.com/collectibles/recognition) · [Ximilar Preise](https://www.ximilar.com/pricing/) · [GemRate Partner API](https://www.gemrate.com/partner)
- [Splink: Fellegi-Sunter-Modell](https://moj-analytical-services.github.io/splink/topic_guides/theory/fellegi_sunter.html) · [Selective Abstention, Chows Regel](https://www.emergentmind.com/topics/selective-abstention) · [scikit-learn Probability Calibration](https://machinelearningmastery.com/calibrated-classification-model-in-scikit-learn/)
- [§ 87b UrhG](https://www.gesetze-im-internet.de/urhg/__87b.html) · [WBS Legal: Ist Screen Scraping legal](https://www.wbs.legal/urheberrecht/ist-screen-scraping-legal-15081/)

*Read-only-Hinweis: keine Datei geändert, kein Server gestartet, keine Requests gegen localhost, keine DB-Schreiboperation. Die DB-Kennzahlen stammen aus einem einzelnen SELECT über `file:data.db?mode=ro`.*