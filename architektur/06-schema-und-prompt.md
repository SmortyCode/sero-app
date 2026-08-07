# SERO — Datenmodell und Vision-System-Prompt

*Read-only erarbeitet, 03.08.2026. Keine Datei geändert, kein Server gestartet, kein Request gegen localhost. Alle Zeilenangaben selbst im Code nachgelesen.*

---

# TEIL A — Das JSON-Datenmodell

## A0. Die vier Leitregeln, aus denen alles folgt

**Regel 1 — Additiv, nie ersetzend.** Kein einziges heute vorhandenes Feld wird umbenannt, verschoben oder in seiner Bedeutung geändert. Die neuen Blöcke sind zusätzliche Schlüssel. Ein alter Datensatz ohne sie funktioniert unverändert; ein neuer Datensatz funktioniert in einer alten App-Version unverändert.

**Regel 2 — Legacy-Felder werden abgeleitet, nicht mehr primär gesetzt.** Ab Version 1 berechnet der Schreiber zuerst die neuen Blöcke und *projiziert* daraus `est_value`, `price_source`, `price_label`, `graded` usw. Damit ist die Abwärtskompatibilität kein Anhängsel, sondern eine Funktion mit einer Zeile Aufruf — und jederzeit prüfbar.

**Regel 3 — Der Scan-Teil reist im vorhandenen `analysis`-Blob mit.** Das ist der wichtigste Kniff für die Abwärtskompatibilität. `analysis` ist bereits in `ANALYSE_FELDER` (`/Users/smorty/ebay-bot/web/app_api.py:731-733`) und wird komplett persistiert. Alles, was die Vision liefert, gehört also **in** `analysis` — dann braucht die Persistenz **keine** Änderung. Nur der Bewertungs-Teil ist ein neuer Top-Level-Schlüssel, weil ihn der Preis-Pfad schreibt und der Analyse-Pfad nicht überschreiben darf.

**Regel 4 — Kein DDL.** `collection_items.data` ist ein JSON-Blob, `card_prices.detail` ebenfalls JSON (`/Users/smorty/ebay-bot/web/catalog.py:36-40`). Das gesamte neue Modell kommt ohne `ALTER TABLE`, ohne Migrations-Skript und ohne einen einzigen Schreibzugriff auf Bestandszeilen aus. Nach dem 02.08. ist das keine Eleganz, sondern eine Sicherheitsanforderung.

### Namenskollisionen — geprüft

| Kandidat | Kollision? | Entscheidung |
|---|---|---|
| `grading` | **JA** — belegt durch die KI-Grading-Schätzung (`app_api.py:3492`) | verworfen |
| `sold` | **JA** — doppelt belegt (Belege vs. `sold_ts`), im Code dokumentiert (`app_api.py:539-542`) | nicht anfassen |
| `match` | belegt in API-Antworten (`app_api.py:3527,3541,3553,3563`) | → `entity_match` |
| `valuation`, `classification`, `slab`, `identity`, `plausibility`, `schema_version` | frei (0 Treffer) | übernommen |

---

## A1. Vollständiges kommentiertes JSON-Beispiel

Gezeigt am realen Fehlerfall (One-Piece-Manga Band 103, Beckett 9.0) — **so, wie das neue Modell ihn richtig behandelt**. `//`-Kommentare dienen nur der Erklärung und stehen nie in den echten Daten.

```jsonc
{
  // ══════════════════════════════════════════════════════════════
  // 1) VERSIONS-MARKER  — neu, v1
  // ══════════════════════════════════════════════════════════════
  "schema_version": 1,          // fehlt => v0 (Altdatensatz). Einziger Skalar,
                                // den ein Leser prüfen muss.

  // ══════════════════════════════════════════════════════════════
  // 2) BESTAND — unverändert, Bedeutung identisch (v0)
  // ══════════════════════════════════════════════════════════════
  "status": "ready",
  "status_text": null,
  "error": null,
  "name": "One Piece Band 103 Manga Deutsch Beckett 9.0",
  "category": "Sonstiges",
  "condition": "USED_EXCELLENT",
  "quantity": 1,
  "notes": null,
  "tags": [],
  "favorite": false,
  "wishlist": false,
  "purchase_price": null,
  "photos": ["/…/abc_slab.png", "/…/def.jpg"],
  "photos_raw": ["/…/abc.jpg", "/…/def.jpg"],
  "remote_photos": [],

  "card_info": {"single": false, "game": "onepiece", "name": null,
                "number": null, "set_total": null, "set_hint": null},
  "card": null,
  "card_key": "comic:one-piece:v103:de",   // ← kein "solo:<uuid>" mehr (siehe A5)

  "est_value": 8.50,                 // ABGELEITET aus valuation.value_eur
  "est_low": 6.00,
  "est_high": 12.00,
  "price_source": "ebay_sold",       // ABGELEITET (Mapping-Tabelle A3)
  "price_label": "Ø letzte 3 eBay-Verkäufe",
  "price_detail": {"pc_usd": null, "pc_field": null, "pc_product": null},
  "price_updated": 1754236800.0,
  "market": {"count": 7, "min": 6.5, "max": 24.0, "median": 11.0,
             "estimated": false, "samples": [/* … */]},
  "sold": {"avg3": 8.50, "n_avg": 3, "sales": [/* … */]},
  "sold_ts": null,

  "graded": {"grader": "BGS", "grade": "9.0", "cert_number": "0018472913"},
                                     // ABGELEITET aus analysis.slab, jetzt mit
                                     // normalisiertem Kürzel statt "BECKETT"
  "grading": null,                   // KI-Grading (card-grader) — unberührt
  "graded_market": null,
  "draft_id": null,
  "scan_seconds": 41.2,

  // ══════════════════════════════════════════════════════════════
  // 3) analysis — Bestandsfelder unverändert, neue Blöcke ergänzt
  //    Reist ohne Persistenz-Änderung mit (Regel 3).
  // ══════════════════════════════════════════════════════════════
  "analysis": {
    // ---- Bestand (v0), unverändert -------------------------------
    "title": "One Piece Band 103 Manga Deutsch Beckett BGS 9.0",
    "subtitle": null,
    "description_html": "<p>…</p>",
    "category_query": "One Piece Manga",
    "condition": "USED_EXCELLENT",
    "condition_description": "Im Beckett-Case, Note 9.0.",
    "aspects": {"Marke": ["Carlsen"], "Sprache": ["Deutsch"]},
    "search_query_for_pricing": "One Piece Band 103 Manga BGS 9",
    "estimated_price_range_eur": {"low": 6.0, "high": 12.0},
    "user_price": null,
    "best_offer": null,
    "format": "FIXED_PRICE",
    "auction_days": 7,
    "quantity": 1,
    "main_image_index": 0,
    "graded_info": {"grader": "BGS", "grade": "9.0",
                    "cert_number": "0018472913"},
    "assumptions": "Cert-Nummer nur per OCR gelesen, nicht verifiziert.",
    "estimated_weight_grams": 400,
    "uncertain": false,
    "question": null,

    // ---- NEU (v1) ------------------------------------------------
    "scan": {                        // Herkunft dieses Vision-Laufs
      "v": 1,
      "pipeline": "vision@2026.08.1",
      "model": "claude-sonnet-4-6",
      "ts": 1754236780.4
    },

    "classification": {              // ★ HARTES ROUTING-FELD
      "item_class": "comic_manga",   // tcg_card | sport_card | comic_manga |
                                     // video_game | sealed_product | coin |
                                     // toy | other
      "holder": "slab",              // slab | sleeve | raw | sealed | none
      "language": "de",              // ISO-639-1 oder null
      "confidence": {"item_class": "hoch", "holder": "hoch",
                     "language": "hoch"},
      "evidence": [
        "Label nennt Verlag und Bandnummer, nicht Set und Kartennummer",
        "Buchruecken im Case sichtbar"
      ]
    },

    "slab": {                        // null wenn holder != "slab"
      "grader": "BGS",               // NUR aus der Kuerzel-Whitelist
      "grader_raw": "BECKETT",       // was woertlich auf dem Label stand
      "scale": "bgs_10",             // psa_10 | bgs_10 | bvg_10 | bccg_10 |
                                     // cgc_cards_10 | cgc_comics_10 | sgc_10 |
                                     // wata_10 | vga_100 | unknown
      "grade": "9.0",
      "grade_numeric": 9.0,
      "subgrades": null,             // {centering,corners,edges,surface} o. null
      "seal_rating": null,           // nur WATA (C … A++)
      "label_color": null,
      "cert_number": "0018472913",
      "cert_channel": "ocr",         // qr | barcode | ocr | null
      "cert_verified": false,        // true NUR nach echtem Lookup
      "cert_verified_at": null,
      "positive_evidence": true,     // Grader UND Note UND (Cert ODER Code)
      "confidence": {"grader": "hoch", "grade": "hoch",
                     "cert_number": "niedrig", "subgrades": null}
    },

    "identity": {                    // was das Stueck IST, strukturiert
      "title_native": "ONE PIECE 103",
      "franchise": "One Piece",
      "series": "One Piece",
      "number": "103",
      "number_kind": "volume",       // volume | issue | card_number | null
      "set_name": null,
      "set_code": null,
      "set_total": null,
      "publisher": "Carlsen",
      "release_year": 2022,
      "edition": null,               // "1st Edition" | "Unlimited" | null
      "variant": null,               // "Holo" | "Reverse Holo" | …
      "isbn": null,
      "upc": null,
      "confidence": {"number": "hoch", "release_year": "mittel",
                     "publisher": "mittel", "set_code": null}
    },

    "queries": {                     // GETRENNTE Queries statt EINER
      "sold": "One Piece Vol 103 Manga BGS 9",
      "catalog": "One Piece Volume 103",
      "ebay": "One Piece Band 103 Manga Beckett 9"
    },

    "photos_meta": {"label_index": 0, "front_index": 0, "back_index": 1}
  },

  // ══════════════════════════════════════════════════════════════
  // 4) valuation — NEU (v1), Top-Level, vom PREIS-Pfad geschrieben
  // ══════════════════════════════════════════════════════════════
  "valuation": {
    "v": 1,
    "pipeline": "value@2026.08.1",
    "ts": 1754236812.9,

    "value_eur": 8.50,
    "value_low_eur": 6.00,
    "value_high_eur": 12.00,
    "source": "aggregator_sold",     // NEUE Taxonomie (A3)
    "legacy_source": "ebay_sold",    // was in price_source projiziert wird
    "label": "Ø letzte 3 eBay-Verkäufe",
    "confidence": 0.82,
    "verdict": "belegt",             // belegt | unsicher | kein_wert

    "grade_bucket": "BGS 9.0",
    "grade_exact": false,            // Quelle kennt 9.0 nicht exakt
    "grade_note": "Quelle rastert nur 9 / 9.5 / 10 — Wert als Spanne lesen",

    "currency": "EUR",
    "fx": {"pair": "USD/EUR", "rate": 0.9123,
           "as_of": "2026-08-03", "source": "ezb"},

    "evidence": {                    // Belege der GEWÄHLTEN Quelle
      "sample_size": 3,
      "sample_total": 6,
      "window_days": 90,
      "oldest_days": 41, "newest_days": 6, "median_age_days": 22,
      "stale": false,
      "items": [
        {"title": "One Piece Vol. 103 Manga BGS 9", "price": 9.00,
         "currency": "EUR", "price_eur": 9.00, "date": "2026-07-28",
         "url": "https://…", "fits": true}
      ]
    },

    "considered": [                  // ALLE Quellen, auch die verworfenen
      {"source": "aggregator_sold", "value_eur": 8.50,
       "sample_size": 3, "used": true},
      {"source": "pricecharting",  "value_eur": 603.12, "sample_size": null,
       "used": false, "rejected": "domain_mismatch"},
      {"source": "ebay_active_de", "value_eur": 9.68,
       "sample_size": 7, "used": false, "rejected": "weaker_source"},
      {"source": "ai_estimate",    "value_eur": 9.00,
       "sample_size": null, "used": false, "rejected": "weaker_source"}
    ],

    "entity_match": {                // die Zuordnung, die den Preis trägt
      "status": "rejected",          // matched | ambiguous | rejected | none
      "provider": "pricecharting",
      "candidate": {"id": "6710612", "name": "Nami [Manga] OP01-016",
                    "console_name": "One Piece Romance Dawn",
                    "domain": "card"},
      "runner_up": null,
      "score_bits": -21.5,
      "probability": 0.0000003,
      "margin": null,                // P1 − P2, null bei nur einem Kandidaten
      "features": {
        "domain_gate": {"expected": "comic_manga", "got": "card",
                        "ok": false, "bits": null},
        "hard_id":     {"value": "103", "found": false, "bits": -6.0},
        "name_ratio":  {"value": 40,  "bits": -4.0},
        "set_ratio":   {"value": 12,  "bits": -2.5},
        "year_delta":  {"value": 0,   "bits": 0.0},
        "image_hamming": {"value": null, "bits": 0.0},
        "price_ratio": {"value": 70.9, "bits": -6.0},
        "language":    {"value": "de vs. n/a", "bits": 0.0}
      },
      "decision_reason": "domain_mismatch"
    },

    "plausibility": {
      "ok": true,
      "checks": [
        {"id": "domain_gate",       "result": "fail", "action": "candidate_rejected",
         "detail": "comic_manga vs. card"},
        {"id": "pc_hard_tokens",    "result": "fail", "action": "candidate_rejected",
         "detail": "103 fehlt im PC-Produktnamen"},
        {"id": "outlier_vs_reference", "result": "pass", "ratio": 1.07},
        {"id": "grade_monotonic",   "result": "skip",
         "detail": "keine Nachbarnote im Katalog"}
      ],
      "corrections": [
        {"from": 603.12, "to": 8.50, "by": "domain_gate",
         "was_source": "pricecharting"}
      ]
    },

    "catalog": {
      "card_key": "comic:one-piece:v103:de",
      "grade": "BGS 9.0",
      "row_updated_at": 1754236812.9,
      "shared": true,                // Wert kam/kommt aus der geteilten Zeile
      "poisoned_cleared": true       // Fehlwert wurde AUCH im Katalog korrigiert
    }
  }
}
```

---

## A2. Feldtabellen

### A2.1 Bestand — Top-Level (alle bleiben, alle unverändert)

| Feld | Typ | Pflicht | Bedeutung | seit |
|---|---|---|---|---|
| `status` | str | ja | `analyzing` \| `ready` \| `error` | v0 |
| `status_text` | str\|null | nein | Live-Fortschritt während des Scans | v0 |
| `error` | str\|null | nein | sichtbarer Fehlertext | v0 |
| `name` | str | ja | Titel des Stücks; `NUR_WENN_LEER` | v0 |
| `category` | str | ja | aus `guess_category` (`app_api.py:173-180`) | v0 |
| `condition` | str\|null | nein | eBay-Zustandscode | v0 |
| `quantity` | int | ja | Default 1 | v0 |
| `notes` | str\|null | nein | Nutzertext | v0 |
| `tags` | list[str] | nein | max. 12 × 30 Zeichen | v0 |
| `favorite` / `wishlist` | bool | nein | Nutzerflags | v0 |
| `purchase_price` | str\|null | nein | über `parse_price` | v0 |
| `photos` / `photos_raw` / `remote_photos` | list[str] | ja/nein | Pfade bzw. eBay-URLs | v0 |
| `analysis` | dict | ja | komplettes Vision-JSON — **Träger der neuen Scan-Blöcke** | v0 |
| `card_info` | dict\|null | nein | `identify_card`-Ausgabe | v0 |
| `card` | dict\|null | nein | Karten-DB-Datensatz | v0 |
| `card_key` | str | ja | Katalog-Schlüssel | v0 |
| `est_value` | float\|null | nein | **die große Zahl** — ab v1 abgeleitet | v0 |
| `est_low` / `est_high` | float\|null | nein | KI-Spanne | v0 |
| `price_source` | str\|null | nein | Legacy-Enum (A3) — ab v1 abgeleitet | v0 |
| `price_label` | str\|null | nein | Anzeigetext — ab v1 abgeleitet | v0 |
| `price_detail` | dict\|null | nein | Quellen-Rohwerte, gemerged | v0 |
| `price_updated` | float | ja | Unix-Zeit | v0 |
| `market` | dict\|null | nein | Browse-Zweitmeinung | v0 |
| `sold` | dict\|null | nein | Verkaufsbelege (**nicht** „verkauft") | v0 |
| `sold_ts` | float\|null | nein | „ist verkauft" — Namenskollision, dokumentiert | v0 |
| `graded` | dict\|null | nein | `{grader,grade,cert_number}` — ab v1 abgeleitet | v0 |
| `grading` | dict\|null | nein | KI-Grading-Schätzung (card-grader) | v0 |
| `graded_market` | dict\|null | nein | PSA-9/10-Marktabfrage | v0 |
| `draft_id` | str\|null | nein | Verknüpfung zum eBay-Entwurf | v0 |
| `scan_seconds` | float | nein | echte Scan-Dauer | v0 |
| **`schema_version`** | **int** | **ja (v1)** | **fehlt ⇒ v0. Der einzige Schalter für Leser.** | **v1** |
| **`valuation`** | **dict\|null** | **nein** | **Bewertung mit Quellen-Attribution (A2.4)** | **v1** |

### A2.2 Neu in `analysis` — `classification`, `slab`, `identity`

| Feld | Typ | Pflicht | Bedeutung | seit |
|---|---|---|---|---|
| `analysis.scan.v` | int | ja | Block-Version | v1 |
| `analysis.scan.pipeline` | str | ja | `vision@JJJJ.MM.n` — welcher Prompt/Code hat das erzeugt | v1 |
| `analysis.scan.model` | str | ja | Modell-ID des Vision-Laufs | v1 |
| `analysis.scan.ts` | float | ja | Unix-Zeit des Laufs | v1 |
| `classification.item_class` | enum | **ja** | **Hartes Routing-Feld.** `tcg_card`\|`sport_card`\|`comic_manga`\|`video_game`\|`sealed_product`\|`coin`\|`toy`\|`other` | v1 |
| `classification.holder` | enum | ja | `slab`\|`sleeve`\|`raw`\|`sealed`\|`none` | v1 |
| `classification.language` | str\|null | nein | ISO-639-1; null wenn nicht ablesbar | v1 |
| `classification.confidence.*` | enum\|null | ja | `hoch`\|`mittel`\|`niedrig` je Aussage | v1 |
| `classification.evidence` | list[str] | nein | 1–3 kurze Begründungen aus dem Bild | v1 |
| `slab` | dict\|null | ja | null wenn `holder != "slab"` | v1 |
| `slab.grader` | enum\|null | ja | nur aus Whitelist: PSA, BGS, BVG, BCCG, BAS, CGC, CBCS, SGC, WATA, VGA, CGA, TAG, ACE, MANA, CSG | v1 |
| `slab.grader_raw` | str\|null | ja | woertlich vom Label — für Nachvollziehbarkeit | v1 |
| `slab.scale` | enum | ja | welche Notenskala; entscheidet die Preisleiter | v1 |
| `slab.grade` / `grade_numeric` | str\|null / float\|null | ja | Note als Text und als Zahl | v1 |
| `slab.subgrades` | dict\|null | nein | alle vier oder keiner (§B, Regel 9) | v1 |
| `slab.seal_rating` | str\|null | nein | nur WATA (C … A++) | v1 |
| `slab.cert_number` | str\|null | ja | null statt Rateraten | v1 |
| `slab.cert_channel` | enum\|null | ja | `qr`\|`barcode`\|`ocr`\|null | v1 |
| `slab.cert_verified` | bool | ja | **true nur nach echtem Lookup** (heute immer false) | v1 |
| `slab.positive_evidence` | bool | ja | Grader ∧ Note ∧ (Cert ∨ Code) | v1 |
| `identity.*` | s. Beispiel | nein | strukturierte Identität statt Freitext | v1 |
| `identity.number_kind` | enum\|null | ja | `volume`\|`issue`\|`card_number`\|null — verhindert Band↔Kartennummer-Verwechslung | v1 |
| `queries.{sold,catalog,ebay}` | str | ja | getrennte Suchanfragen je Quellentyp | v1 |
| `photos_meta.{label,front,back}_index` | int\|null | nein | welches Foto zeigt was | v1 |

### A2.3 `valuation` — Kopf

| Feld | Typ | Pflicht | Bedeutung | seit |
|---|---|---|---|---|
| `v` / `pipeline` / `ts` | int/str/float | ja | Block-Version, Pipeline-Kennung, Zeitstempel | v1 |
| `value_eur` | float\|null | ja | der Marktwert; null = kein belastbarer Wert | v1 |
| `value_low_eur` / `value_high_eur` | float\|null | nein | Spanne (Pflicht wenn `grade_exact=false`) | v1 |
| `source` | enum | ja | neue Quellen-Taxonomie (A3) | v1 |
| `legacy_source` | enum | ja | Projektionsziel für `price_source` | v1 |
| `label` | str | ja | Anzeigetext | v1 |
| `confidence` | float 0–1 | ja | kalibrierte Wahrscheinlichkeit der Zuordnung | v1 |
| `verdict` | enum | ja | `belegt`\|`unsicher`\|`kein_wert` — steuert die UI | v1 |
| `grade_bucket` | str | ja | z. B. `BGS 9.0` oder `raw` | v1 |
| `grade_exact` | bool | ja | false ⇒ Quelle rastert gröber als die Note | v1 |
| `grade_note` | str\|null | nein | Klartext dazu | v1 |
| `currency` | str | ja | immer `EUR` für `value_eur` | v1 |
| `fx` | dict\|null | ja bei Fremdwährung | `{pair, rate, as_of, source}` | v1 |

### A2.4 `valuation.evidence` / `considered` / `entity_match` / `plausibility`

| Feld | Typ | Pflicht | Bedeutung | seit |
|---|---|---|---|---|
| `evidence.sample_size` | int\|null | ja | **Stichprobengröße** der gewählten Quelle | v1 |
| `evidence.sample_total` | int\|null | nein | wie viele Treffer es insgesamt gab | v1 |
| `evidence.window_days` | int\|null | nein | Betrachtungsfenster (90 bei Sold) | v1 |
| `evidence.oldest_days` / `newest_days` / `median_age_days` | int\|null | ja | **Alter der Daten** | v1 |
| `evidence.stale` | bool | ja | alle Belege älter als das Fenster | v1 |
| `evidence.items[]` | list[dict] | ja | **Belege**: `{title, price, currency, price_eur, date, url, fits}` | v1 |
| `considered[]` | list[dict] | ja | **jede geprüfte Quelle** mit `used` und `rejected`-Grund | v1 |
| `entity_match.status` | enum | ja | `matched`\|`ambiguous`\|`rejected`\|`none` | v1 |
| `entity_match.candidate` | dict\|null | ja | inkl. `domain` — das Feld, an dem der Manga-Bug scheiterte | v1 |
| `entity_match.runner_up` | dict\|null | nein | Zweitbester — Grundlage für `margin` | v1 |
| `entity_match.score_bits` | float\|null | ja | additive Evidenz in Bits | v1 |
| `entity_match.probability` | float\|null | ja | `1/(1+2^-S)` | v1 |
| `entity_match.margin` | float\|null | nein | P₁−P₂; < 0.15 ⇒ verwerfen | v1 |
| `entity_match.features` | dict | ja | je Merkmal Wert **und** Bit-Beitrag — nachrechenbar | v1 |
| `entity_match.decision_reason` | str\|null | ja | maschinenlesbarer Grund | v1 |
| `plausibility.ok` | bool | ja | Gesamtergebnis | v1 |
| `plausibility.checks[]` | list[dict] | ja | `{id, result: pass\|fail\|skip, action, detail}` | v1 |
| `plausibility.corrections[]` | list[dict] | nein | jede angewandte Korrektur mit Vorher/Nachher | v1 |
| `catalog.poisoned_cleared` | bool | nein | Korrektur wurde auch in `card_prices` geschrieben | v1 |

---

## A3. Quellen-Taxonomie und Projektion auf `price_source`

Die vorgegebene Aufzählung (`ebay_completed` / `aggregator_sold` / `pricecharting` / `catalog_estimate` / `ai_estimate`) deckt zwei im Code aktive Quellen **nicht** ab: den eBay-Browse-Median (aktive Angebote, heute `ebay_eu` und `ebay`) und die Karten-Datenbanken (`cardmarket`, `scryfall`, `ygoprodeck`, `tcgplayer`). Sie wird deshalb erweitert — die vorgegebenen Werte bleiben unverändert:

| `valuation.source` (neu) | Herkunft im Code | → `price_source` (Legacy) | Belege? |
|---|---|---|---|
| `ebay_completed` | eBay Marketplace Insights (`web/ebay_insights.py`, per Default aus) | `ebay_sold` | ja, echte Verkäufe |
| `aggregator_sold` | 130point (`web/sold.py`) | `ebay_sold` | ja, echte Verkäufe |
| `ebay_active_de` | Browse-Median × 0,88 (`catalog.py:139-147`) | `ebay_eu` | nein, Angebotspreise |
| `ebay_active` | Browse-Median (`bot/ebay/browse.py`) | `ebay` | nein, Angebotspreise |
| `pricecharting` | `web/pricecharting.py`, Match bestätigt | `pricecharting` | aggregiert, keine Einzelbelege |
| `pricecharting_unmatched` | Match nicht bestätigt | `pricecharting_weak` | — **darf ab v1 kein Preis mehr sein** |
| `tcg_db_cardmarket` | TCGdex (`web/prices.py`) | `cardmarket` | Index |
| `tcg_db_scryfall` | Scryfall | `scryfall` | Index |
| `tcg_db_ygoprodeck` | YGOPRODeck | `ygoprodeck` | Index |
| `tcg_db_tcgplayer` | tcgcsv (`web/tcgcsv.py`) | `tcgplayer` | Index |
| `catalog_estimate` | geteilte Katalogzeile eines anderen Scans | Quelle der Zeile | ererbt |
| `seller_listing` | eBay-Import | `listing` | — |
| `ai_estimate` | `estimated_price_range_eur` | `estimate` | **keine** |

**Wichtig:** `ebay_completed` und `aggregator_sold` bilden bewusst beide auf `ebay_sold` ab. Damit greift `SOURCE_INFO` in `/Users/smorty/sero-app/web/sero.js:509` unverändert, und der Unterschied zwischen offizieller API und Aggregator bleibt trotzdem in `valuation.source` erhalten — genau die Information, die Sven braucht, sollte eBay den Insights-Zugang doch noch erteilen.

**Projektionsfunktion** (Pseudocode, die einzige Stelle, an der Legacy-Felder ab v1 entstehen):

```python
def projiziere_legacy(item, analysis, valuation):
    if valuation and valuation["verdict"] != "kein_wert":
        item["est_value"]    = valuation["value_eur"]
        item["price_source"] = valuation["legacy_source"]
        item["price_label"]  = valuation["label"]
    else:
        item["est_value"] = None          # ehrlich: keine Zahl
        item["price_source"] = None
        item["price_label"] = None
    s = analysis.get("slab")
    item["graded"] = ({"grader": s["grader"], "grade": s["grade"],
                       "cert_number": s["cert_number"]}
                      if s and s.get("positive_evidence") else None)
    item["schema_version"] = 1
```

---

## A4. Migrationsstrategie

### 1. Lesen alter Datensätze — ohne Migration

Es gibt **keine** Migration. Kein Backfill, kein Bulk-Update, kein Schreibzugriff auf Bestandszeilen. Regel für jeden Leser:

```python
version   = item.get("schema_version", 0)
valuation = item.get("valuation")           # None bei v0 — legitim
scan      = (item.get("analysis") or {}).get("classification")   # None bei v0
```

Jeder heutige Leser — `item_public` (`app_api.py:514-560`), die PWA, `snapshot_price`, der Export — arbeitet weiter ausschließlich auf den Legacy-Feldern und braucht **keine einzige Änderung**. Die neuen Blöcke sind für ihn unsichtbar.

### 2. Schreiben im neuen Format

Der Schreiber berechnet **zuerst** die neuen Blöcke, ruft dann `projiziere_legacy()` und schreibt beides gemeinsam. Solange beides geschrieben wird, ist ein Rückfall auf v0-Verhalten jederzeit möglich: Blöcke ignorieren, Legacy-Felder sind vollständig. Es geht dann nur die *Erklärung* verloren, nie der Wert.

### 3. Versionierung — drei Ebenen

| Ebene | Feld | Zweck |
|---|---|---|
| Datensatz | `schema_version: 1` | Der eine Schalter. Fehlt ⇒ v0. |
| Block | `analysis.scan.v`, `valuation.v` | Blöcke entwickeln sich unabhängig weiter |
| Lauf | `analysis.scan.pipeline`, `valuation.pipeline` | `name@JJJJ.MM.n` — welcher Prompt/Code hat diesen Wert erzeugt. Ohne dieses Feld ist nach einem Prompt-Wechsel nicht mehr rekonstruierbar, welche Datensätze noch aus der alten Logik stammen. |

### 4. Lazy Upgrade

v0-Zeilen erhalten die neuen Blöcke **erst beim nächsten Scan oder Refresh** dieses Stücks. Bestand wird nie angefasst. Das ist gleichzeitig die Migration und die Absicherung gegen den Vorfall vom 02.08.

### 5. Vorwärtskompatibilität

Beim Schreiben werden Blöcke **gemerged, nicht ersetzt**: `{**alt, **neu}` mit expliziter Löschliste. So kastriert ein älterer Server einen neueren Datensatz nicht. Unbekannte Schlüssel innerhalb eines Blocks werden nie entfernt.

### 6. 🔴 Die eine Code-Falle, die vorher behoben sein muss

`/Users/smorty/ebay-bot/web/app_api.py:725-733`:

```python
PREIS_FELDER = ("card", "card_key", "card_info", "est_value", "price_source",
                "price_label", "price_detail", "market", "sold", "price_updated")

ANALYSE_FELDER = PREIS_FELDER + ("status", "status_text", "error", "analysis",
                                 "photos", "photos_raw", "graded",
                                 "est_low", "est_high", "grading")
```

`col_save_analyse` (`:740-746`) und `refresh_item_price` (`:862-866`) kopieren **ausschließlich** die hier genannten Felder in den frisch gelesenen Datensatz. Ein neues Top-Level-Feld, das nicht in der Liste steht, wird beim nächsten Lauf **stillschweigend verworfen** — kein Fehler, kein Log, es ist einfach weg.

Konkret nötig sind genau zwei Ergänzungen:

```python
PREIS_FELDER = (..., "price_updated", "valuation", "schema_version")
```

`analysis` steht bereits in `ANALYSE_FELDER` — deshalb brauchen `classification`, `slab`, `identity`, `queries` und `scan` **keine** Änderung. Das ist der ganze Grund für Regel 3.

### 7. Katalog (`card_prices`)

`detail` ist bereits JSON (`catalog.py:36-40`). Der `valuation`-Block wandert unverändert dort hinein — **kein DDL**. Zusätzlich: wenn ein Wächter am Item korrigiert, muss dieselbe Korrektur in die Katalogzeile geschrieben und `catalog.poisoned_cleared: true` gesetzt werden. Ohne das bleibt der Befund aus der Bestandsaufnahme bestehen: 87 von 230 Zeilen stehen auf `pricecharting_weak`, die 603-€-Zeile 70-fach.

### 8. Anzeige (optional, später)

Will die PWA die Konfidenz zeigen, kommen in `item_public` drei Schlüssel dazu — **nur wenn vorhanden**, damit ältere App-Versionen unverändert laufen:

```python
**({"value_confidence": v["confidence"],
    "value_verdict": v["verdict"],
    "value_evidence": v["evidence"]} if (v := item.get("valuation")) else {})
```

### 9. Rollback

`schema_version` auf 0 setzen bzw. die Blöcke ignorieren. Die Legacy-Felder sind vollständig und selbsttragend. Es gibt keinen Zustand, in dem ein Rollback Daten verliert.

---

## A5. Warum `card_key` im Beispiel `comic:one-piece:v103:de` heißt

Die Bestandsaufnahme hat gemessen: 223 von 230 `card_prices`-Zeilen liegen unter `solo:<uuid>`, weil `card_key_of` (`catalog.py:48-60`) bei fehlendem `name` **und** fehlender `ref_id` einen Wegwerfschlüssel erzeugt — und `setdefault("name", …)` in `app_api.py:790` bzw. `:1000` ein vorhandenes `None` nicht überschreiben kann.

Das neue Modell liefert die fehlende Identität: `classification.item_class` + `identity.*` + `classification.language` ergeben einen deterministischen Schlüssel auch für Nicht-Karten:

```
{item_class}:{normalisierter Franchise/Set}:{number_kind-Präfix}{number}:{language}[:{edition}][:{variant}]
```

Das ist eine **Code-Änderung außerhalb dieses Auftrags** — aber ohne sie bleibt der Katalog wirkungslos, und das Datenmodell wäre reine Dekoration. Deshalb hier explizit benannt.

---

# TEIL B — Der Vision-System-Prompt

## B1. Was am heutigen Prompt konkret schlecht ist

Alle Zitate wörtlich aus `/Users/smorty/ebay-bot/bot/claude_client.py`.

### B1.1 Der Prompt fordert ausdrücklich zum Raten auf

> **Zeile 40-41:** „Sind mehrere Modelle plausibel: Wähle das WAHRSCHEINLICHSTE und vermerke die Annahme knapp im Feld ‚assumptions'."

Das ist für ein Handy-Listing sinnvoll und für eine Marktwert-Ermittlung fatal. Der Prompt erteilt der Vision eine ausdrückliche Lizenz zu raten, und der geratene Wert wandert ungeprüft in `search_query_for_pricing` und von dort an 130point, PriceCharting und Browse gleichzeitig. Es gibt keine Stelle im System, die einen geratenen von einem abgelesenen Wert unterscheiden kann.

### B1.2 Es gibt kein Warenart-Feld — die Wurzel des 603-€-Falls

Der gesamte Prompt kennt keine Produktklasse. `category_query` ist Freitext für den eBay-Kategorievorschlag, `guess_category` (`app_api.py:173-180`) ist eine Keyword-Liste. Ein Manga und eine Sammelkarte sind für die nachgelagerte Preiskette ununterscheidbar. `lookup_pc` (`pricecharting.py:87-92`) prüft nur, ob `prod["id"]` existiert — mehr Information hat sie gar nicht.

### B1.3 Das Schema-Beispiel ist selbst kein gültiges JSON

> **Zeile 66:** „Antworte AUSSCHLIESSLICH mit validem JSON. Keine Markdown-Fences, kein Vortext, kein Nachtext."

> **Zeile 76:** `"aspects": {"Marke": ["..."], "Modell": ["..."]},  // Bei Videospielen/DVDs/Blu-rays: NUR wenn …`

Das Schema, an dem sich das Modell orientieren soll, enthält einen `//`-Kommentar — in JSON ungültig. Der Prompt verlangt striktes JSON und führt im selben Atemzug vor, wie man es bricht. Dass `parse_listing_json` (`:115-132`) und `cardscan._json_of` (Prosa-Toleranz, Klammer-Suche) defensiv nachputzen müssen, ist die Folge.

### B1.4 Der Unsicherheits-Mechanismus ist im Scanner totes Holz

> **Zeile 88:** `"uncertain": false,`

Und in `app_api.py:906`:

```python
listing.pop("uncertain", None)
```

Der Scan-Pfad wirft das Feld weg, bevor es irgendjemand liest. Die einzige eingebaute Bremse gegen eine falsche Erkennung ist im Sammlungs-Scan wirkungslos.

### B1.5 Das Grading-Feld ist zu dünn für den Zweck

> **Zeile 85:** `"graded_info": "NUR wenn die Karte/das Spiel sichtbar in einem Grading-Slab (Plastikgehäuse mit Bewertungslabel) steckt, sonst null: {\"grader\": \"PSA\", \"grade\": \"9.5\", \"cert_number\": \"12345678\"} — Werte exakt vom Slab-Label ablesen, nicht lesbare Felder null. Eine LOSE Karte ist NIEMALS graded."`

Die Absicht ist richtig. Was fehlt: keine Skala (BGS 9.5 ≠ CGC 9.5 ≠ BCCG 9.5), keine Subgrades, keine Formatvorgabe für die Cert-Nummer, kein Konfidenz-Feld, keine Pflicht zu positiver Evidenz. Der Satz „Eine LOSE Karte ist NIEMALS graded" ist eine Instruktion — im Code gibt es keine Validierung, die sie erzwingt.

### B1.6 Die Kürzel-Tabelle ist unvollständig und an einer Stelle falsch

`Zeile 139-142`:

```python
_GRADER_KUERZEL = {
    "beckett": "BGS", "bgs": "BGS", "psa": "PSA", "cgc": "CGC", "sgc": "SGC",
    "wata": "WATA", "vga": "VGA", "cga": "CGA", "ace": "ACE",
}
```

Es fehlen `bvg`, `bccg`, `bas`, `cbcs`, `tag`, `csg`, `mana`. Und `beckett → BGS` ist bei einem Vintage-Slab (BVG) oder einem Collectors-Club-Slab (BCCG) schlicht falsch — BCCG-Noten liegen typischerweise zwei bis drei Stufen über BGS. Ein als BGS 10 bepreistes BCCG 10 ist genau die Fehlerklasse, die Sven Geld kostet.

### B1.7 🔴 Die Titel-Korrektur kann korrekte Titel zerstören

`Zeile 158-159`:

```python
neu = _re.sub(r"\b[A-Za-z]{2,5}(?=\s*\d{1,2}(?:\.\d)?\b)",
              richtig, titel, count=1)
```

Der Ausdruck ersetzt das **erste** Wort aus 2–5 Buchstaben, das vor einer ein- oder zweistelligen Zahl steht — unabhängig davon, ob es ein Grader-Kürzel ist. Bei einem Titel wie `"One Piece Band 9 Beckett 9.0"` trifft er `Band` und macht daraus `"One Piece BGS 9 Beckett 9.0"`. Der Titel wird unauffindbar, und `sold.fits` (`sold.py:175-210`) verliert seinen wichtigsten Sicherungsstift, weil die Bandnummer jetzt hinter einem Grader-Kürzel steht. Die Funktion greift nur, wenn `graded_info` gesetzt ist — also genau bei den wertvollen Stücken.

### B1.8 Eine Query für drei Quellen mit drei Syntaxen

> **Zeile 77:** `"search_query_for_pricing": "präzise Query für Preisvergleich auf eBay (Marke + Modell + Variante)"`

Derselbe Freitext geht an 130point (Volltext über Verkaufstitel), an PriceCharting (Produktkatalog-Suche) und an die eBay Browse API (`app_api.py:799, 821`). Drei Systeme, drei Trefferlogiken, eine Zeichenkette. Kein Feld hält fest, *welches Produkt* am Ende bepreist wurde.

### B1.9 Svens Wording-Regeln stehen nicht drin

Der Prompt verbietet Füllwörter („TOP", „RAR", „L@@K", „WOW", Zeile 49), sagt aber nichts zu Ausrufezeichen, Emojis, „Wir"-Formulierungen oder der Begriffswahl (Stück, listen, Marktwert, tippen). Bei zehntausenden Nutzern entstehen daraus zehntausende Texte in fremder Stimme.

---

## B2. Der neue System-Prompt (deutsch, direkt einsetzbar)

Ersetzt `SYSTEM_PROMPT` in `bot/claude_client.py:29-95`. Alle bisherigen Ausgabefelder bleiben erhalten — `parse_listing_json` (`:127-130`) und alle Leser laufen unverändert weiter.

```text
Du bist das Erkennungsmodul von SERO. Du siehst Fotos eines Sammlerstücks und
lieferst zwei Dinge: eine exakte, belegbare Identifikation und daraus einen
fertigen eBay-Listing-Entwurf. Beides ausschliesslich als JSON.

Deine Ausgabe bestimmt einen Marktwert, den ein Mensch fuer eine Kauf- oder
Verkaufsentscheidung benutzt. Eine falsche Zahl ist teurer als eine fehlende.
Deshalb gilt vor allem anderen:

  GRUNDGESETZ — Was du nicht am Bild ablesen oder den Angaben des Verkaeufers
  entnehmen kannst, ist "null". Niemals raten, niemals aus Wahrscheinlichkeit
  ergaenzen, niemals aus einem Feld auf ein anderes schliessen. "null" ist
  immer eine korrekte Antwort. Eine erfundene Antwort ist immer ein Fehler.

═══════════════════════════════════════════════════════════════════════════
1  AUSGABEFORMAT
═══════════════════════════════════════════════════════════════════════════
Antworte mit genau einem JSON-Objekt. Nichts davor, nichts danach.
Keine Markdown-Fences. Keine Kommentare — weder // noch /* */ noch #.
Keine nachgestellten Kommata. Alle Zeichenketten in doppelten Anfuehrungs-
zeichen. Fehlende Werte als JSON-null, nicht als "null", "-", "unbekannt"
oder "".

Das vollstaendige Schema steht in Abschnitt 8. Die Regeln davor sagen, wie
die Felder zu fuellen sind. Bei Widerspruch gewinnen die Regeln.

═══════════════════════════════════════════════════════════════════════════
2  KONFIDENZ — bei jeder Aussage
═══════════════════════════════════════════════════════════════════════════
Jedes Konfidenz-Feld nimmt genau einen dieser drei Werte:

  "hoch"    Direkt und eindeutig im Bild lesbar oder vom Verkaeufer genannt.
            Du wuerdest darauf wetten.
  "mittel"  Erkennbar, aber teilweise verdeckt, gespiegelt, unscharf oder aus
            zwei Hinweisen zusammengesetzt.
  "niedrig" Vermutet. Der Wert koennte falsch sein.

Wenn ein Wert null ist, ist auch seine Konfidenz null.
Nutze "niedrig" grosszuegig. Ein ehrliches "niedrig" ist wertvoll; ein
geschoentes "hoch" ist Sabotage. Setze nie "hoch", nur weil eine Angabe
plausibel klingt.

═══════════════════════════════════════════════════════════════════════════
3  WARENART — item_class (das wichtigste Feld ueberhaupt)
═══════════════════════════════════════════════════════════════════════════
Bestimme zuerst, WAS fuer ein Stueck das ist. Danach richtet sich, in welchem
Markt der Wert gesucht wird. Eine falsche Warenart fuehrt garantiert zu einem
falschen Preis, egal wie gut der Rest stimmt.

  "tcg_card"        Einzelne Sammelkarte eines Kartenspiels
                    (Pokemon, Magic, Yu-Gi-Oh, One Piece, Lorcana, Digimon …)
  "sport_card"      Einzelne Sportkarte (Panini, Topps, Upper Deck …)
  "comic_manga"     Heft, Manga-Band, Comic, Graphic Novel, Buch
  "video_game"      Videospiel, Konsole, Zubehoer
  "sealed_product"  Ungeoeffnete Verpackung: Display, Booster-Box, ETB,
                    Blister, Tin
  "coin"            Muenze, Medaille, Banknote
  "toy"             Figur, Pluesch, Modell, Statue
  "other"           Alles andere

UNTERSCHEIDUNGSHILFEN (die Labelaufschrift verraet die Warenart):
- Ein Label mit Titel + Heft-/Bandnummer + Erscheinungsdatum + Verlag gehoert
  zu "comic_manga". Ein Label mit Jahr + Set + Kartenname + Kartennummer
  gehoert zu "tcg_card" oder "sport_card".
- "One Piece" ist sowohl ein Kartenspiel als auch eine Manga-Reihe. Ein
  gedrucktes Buch mit Ruecken und Seiten ist NIE eine Karte, auch wenn die
  Reihe ein Kartenspiel hat. Achte auf Buchruecken, ISBN, Verlagslogo,
  Seitenschnitt.
- Eine Nummer ist nicht gleich Nummer: "Band 103" ist eine Bandnummer
  (number_kind "volume"), "OP01-016" eine Kartennummer (number_kind
  "card_number"), "#4" bei einem Comic eine Heftnummer ("issue"). Trage die
  Art immer mit ein.

═══════════════════════════════════════════════════════════════════════════
4  VERPACKUNGSART — holder
═══════════════════════════════════════════════════════════════════════════
  "slab"    In einem verschweissten Bewertungsgehaeuse mit BEDRUCKTEM,
            undurchsichtigem Label.
  "sleeve"  In Huelle, Toploader oder Magnetholder — durchsichtig, kein
            bedrucktes Label.
  "raw"     Ohne Schutz.
  "sealed"  Originalversiegelte Verpackung (Display, Booster-Box, Blister).
  "none"    Warenart, bei der die Frage nicht sinnvoll ist.

Das entscheidende Merkmal fuer "slab" ist das opake, bedruckte Label am Rand
des Gehaeuses. Ein Toploader ist ueber die ganze Flaeche durchsichtig.
Ein Toploader oder eine Huelle sagen NICHTS ueber den Zustand aus und sind
niemals ein Grading.

═══════════════════════════════════════════════════════════════════════════
5  GRADING — die strengsten Regeln des ganzen Prompts
═══════════════════════════════════════════════════════════════════════════

5.1  WANN "slab" gefuellt wird
Fuelle den Block "slab" ausschliesslich, wenn holder == "slab". In jedem
anderen Fall ist "slab" null. Eine lose Karte, eine Karte in einer Huelle und
eine versiegelte Box sind NIEMALS gegradet.

5.2  POSITIVE EVIDENZ — setze "positive_evidence" nur bei ALLEN dreien:
  (a) Ein Grader-Name oder -Logo ist erkennbar, UND
  (b) eine Note ist lesbar, UND
  (c) eine Zertifikatsnummer ist lesbar ODER ein QR-/Barcode ist sichtbar.
Fehlt eines davon, ist positive_evidence false. Fuelle die lesbaren Felder
trotzdem, lasse den Rest null.

5.3  ERFUNDENE ZERTIFIKATSNUMMERN SIND DER SCHWERSTE FEHLER
Eine Zertifikatsnummer wird spaeter maschinell nachgeschlagen. Eine erfundene
Nummer erzeugt entweder einen Fehlschlag oder — schlimmer — den Datensatz
eines fremden Stuecks.
- Gib eine Nummer NUR aus, wenn du jede Ziffer im Bild siehst.
- Ergaenze nie fehlende Stellen, auch wenn du die uebliche Laenge kennst.
- Nimm nie eine Nummer von einem anderen Foto, aus deinem Vorwissen oder aus
  einem Beispiel in diesen Anweisungen.
- Ist ein Teil verdeckt, gespiegelt oder unscharf: cert_number = null,
  cert_channel = null.
- Steht die Nummer in einem QR- oder Barcode, den du nicht sicher lesen
  kannst, setze cert_channel = null und cert_number = null. Vermerke die
  sichtbaren Codes in "assumptions".
- cert_verified ist IMMER false. Nur ein echter Lookup darf das aendern; du
  fuehrst keinen durch.

5.4  KORREKTE GRADER-KUERZEL (nur diese Werte sind erlaubt)
  PSA   Professional Sports Authenticator
  BGS   Beckett Grading Services — moderne Karten, mit Subgrades
  BVG   Beckett Vintage Grading — Vintage, OHNE Subgrades
  BCCG  Beckett Collectors Club Grading — eigene, deutlich mildere Skala
  BAS   Beckett Authentication Services — nur Autogramme
  CGC   Certified Guaranty Company
  CBCS  Certified Bookstore Comic Services
  SGC   Sportscard Guaranty
  WATA  Wata Games
  VGA   Video Game Authority
  CGA   Collectible Grading Authority
  TAG / ACE / MANA / CSG

  "Beckett" ist der FIRMENNAME, kein Kuerzel. Ein Beckett-Slab kann BGS, BVG,
  BCCG oder BAS sein. Entscheide anhand des Labels:
    Vier Subgrades (Centering/Corners/Edges/Surface) sichtbar → BGS
    Vintage-Karte ohne Subgrades                             → BVG
    Aufschrift "Collectors Club" / "BCCG"                     → BCCG
    Nur Autogramm-Bestaetigung ohne Kartennote                → BAS
  Ist es nicht entscheidbar: grader = null, grader_raw = "BECKETT",
  confidence.grader = null. Rate NICHT auf BGS.

  Schreibe in "grader_raw" immer woertlich, was auf dem Label steht.

5.5  SKALA — scale
  psa_10 | bgs_10 | bvg_10 | bccg_10 | cgc_cards_10 | cgc_comics_10 |
  sgc_10 | wata_10 | vga_100 | unknown
  CGC bewertet Karten und Comics nach unterschiedlichen Skalen. Eine 9.8
  gibt es bei CGC-COMICS, nicht bei CGC-Karten. Siehst du bei einer Karte
  eine 9.8, pruefe, ob es in Wahrheit ein Comic ist.
  Bist du unsicher: scale = "unknown". Rate nicht.

5.6  NOTE
  Die Gesamtnote ist die typografisch groesste, hervorgehobene Zahl.
  Uebernimm sie woertlich als Text ("9.0" bleibt "9.0", nicht "9") und
  zusaetzlich als Zahl in grade_numeric.
  Verwechsle sie nie mit einer Subgrade, einer Jahreszahl, einer Kartennummer
  oder einem Teil der Zertifikatsnummer.

5.7  SUBGRADES — alle vier oder keiner
  Subgrades erscheinen als Vierergruppe (Centering, Corners, Edges, Surface).
  Kannst du nicht alle vier sicher lesen, ist subgrades = null. Gib niemals
  eine Teilmenge aus. BVG-Slabs haben keine Subgrades.

5.8  WATA
  WATA vergibt ZWEI Werte: eine Box-Note (Zahl) und ein Seal Rating
  (C bis A++). Lies beide. Fehlt eines, setze es null — uebernimm nie das
  Seal Rating als Note.

═══════════════════════════════════════════════════════════════════════════
6  IDENTITAET, SPRACHE, SET, NUMMER
═══════════════════════════════════════════════════════════════════════════

6.1  SPRACHE
  Bestimme sie an gedrucktem Text auf dem Stueck selbst: Kartenname,
  Regeltext, Verlagsangabe, Labelaufschrift.
  Kein Text lesbar → language = null. Schliesse NIE von der Marke, der Reihe
  oder deinem Vorwissen auf die Sprache.
  Japanische Ausgaben haben eigene Set-Codes, die nicht auf englische Sets
  abbilden. Rechne nie um. Uebernimm den Code, wie er dasteht.

6.2  SET UND SET-CODE
  Uebernimm Set-Namen und Set-Code exakt so, wie sie auf dem Stueck oder dem
  Label stehen. Uebersetze nicht, kuerze nicht, vervollstaendige nicht.
  Nicht lesbar → null. Leite den Set-Namen niemals aus dem Kartennamen ab —
  dieselbe Karte erscheint in mehreren Sets zu sehr verschiedenen Preisen.

6.3  NUMMER
  Bei "199/165": number = "199", set_total = "165", number_kind =
  "card_number". Fuehrende Nullen entfernen.
  Bei Manga/Comics: die Band- bzw. Heftnummer, number_kind "volume" bzw.
  "issue".
  Die Nummer ist der wichtigste Einzelwert fuer die Preissuche. Ist sie
  verdeckt, angeschnitten oder unscharf: number = null und
  confidence.number = null. Eine falsche Nummer ist schlimmer als keine.

6.4  AUFLAGE UND VARIANTE
  "1st Edition", "Unlimited", "Holo", "Reverse Holo", "Promo", "Full Art"
  usw. nur eintragen, wenn das entsprechende Zeichen sichtbar ist. Sonst
  null. Das Fehlen eines Stempels ist kein Beweis fuer "Unlimited".

═══════════════════════════════════════════════════════════════════════════
7  DER LISTING-ENTWURF
═══════════════════════════════════════════════════════════════════════════

7.1  ANGABEN DES VERKAEUFERS
  Text oder Bildunterschrift des Verkaeufers haben IMMER Vorrang vor deiner
  Bilderkennung — bei Preis, Menge, Zustand, Modell und Format.

7.2  TITEL-FORMEL (exakt diese Reihenfolge, deutsch, 60-70 Zeichen, max 80)
  Marke/Reihe → Produkt/Modell → Variante oder Nummer → wichtigstes Merkmal
  → Groesse/Farbe/Sprache → Grading oder Zustandskeyword.
  Bei einem Slab: das korrekte Kuerzel aus 5.4 plus Note ans Ende, exakt in
  der Form "BGS 9.0". Ist grader null, schreibe KEIN Kuerzel in den Titel.
  Natuerliche Kaeufersprache. Keine Woerter in Grossbuchstaben, keine
  Fuellwoerter wie TOP, RAR, L@@K, WOW, keine Sonderzeichen-Deko, kein
  Keyword-Stuffing.
  Beispiele:
    tcg_card:    "Pokemon Glurak ex 199/165 151 Deutsch PSA 10"
    comic_manga: "One Piece Band 103 Manga Deutsch Carlsen BGS 9.0"
    video_game:  "Zelda Ocarina of Time N64 PAL Modul Deutsch"

7.3  BESCHREIBUNG (exakt diese HTML-Struktur, nichts anderes)
  <p>[2-3 Saetze: was ist es, was macht es besonders — sachlich]</p>
  <p><b>Details</b></p>
  <ul><li>[je ein Fakt: Marke, Modell, Set, Nummer, Sprache, Grading …]</li></ul>
  <p><b>Zustand</b></p>
  <p>[ehrliche Beschreibung anhand der Fotos; bei Neuware "Neu und ungeoeffnet"]</p>
  <p><b>Versand</b></p>
  <p>Schneller Versand, sicher und sorgfaeltig verpackt.</p>

  Nur beschreiben, was auf den Fotos erkennbar oder vom Verkaeufer angegeben
  ist. Keine erfundenen Details (Speichergroesse, Jahr, OVP, Auflage).
  Unbekanntes weglassen, nicht umschreiben.

7.4  SPRACHE UND STIL ALLER ENDTEXTE
  Betrifft title, subtitle, description_html, condition_description und
  assumptions:
  - Keine Ausrufezeichen.
  - Keine Emojis, keine Symbolzeichen als Schmuck.
  - Kein "Wir". Formuliere sachlich in der dritten Person oder unpersoenlich.
    Statt "Wir versenden schnell" → "Schneller Versand".
  - Sag "Stueck", nicht "Artikel", "Item" oder "Produkt", wenn das
    Sammlungsobjekt gemeint ist.
  - Sag "listen" fuer das Einstellen bei eBay.
  - Sag "Marktwert" fuer den ermittelten Preis, nicht "Schaetzwert",
    "Bewertung" oder "Preisvorschlag".
  - Sag "tippen" fuer eine Bedienhandlung, nicht "klicken".
  - Keine Superlative, keine Werbesprache, keine Dringlichkeit.

7.5  MERKMALE (aspects)
  Nur belegbare Merkmale.
  USK: Bei Videospielen, DVDs und Blu-rays das Merkmal
  "USK-Einstufung": ["USK ab X Jahren"] NUR setzen, wenn ein deutsches
  USK-Logo auf dem Cover sichtbar ist (weiss/gruen=0, gelb=6, blau=12,
  rot=16, schwarz=18). Ohne sichtbares deutsches USK-Logo (US-Importe mit
  ESRB, japanische mit CERO) das Merkmal KOMPLETT WEGLASSEN — kein Feld,
  keine Ersatzangabe, auch nicht "nicht vorhanden" oder "keine". Niemals
  ESRB, PEGI oder CERO in ein Altersmerkmal schreiben: eBay.de liest das als
  Erwachsenen-Kennzeichen und sperrt das Listing.

7.6  SUCHANFRAGEN — drei getrennte Queries
  Die Preisermittlung fragt drei verschiedene Systeme. Formuliere fuer jedes:
  - queries.sold:    fuer die Suche in VERKAUFSTITELN. Kurz, mit Nummer und
                     — falls vorhanden — Grader und Note. Keine Fuellwoerter.
                     Bei Manga/Comics gehoert die Bandnummer zwingend hinein.
  - queries.catalog: fuer eine PRODUKTKATALOG-Suche. Offizieller Produktname
                     ohne Zustands- und Gradingangaben.
  - queries.ebay:    fuer die Suche in AKTIVEN Angeboten. Wie ein Kaeufer
                     suchen wuerde, deutsch.
  search_query_for_pricing setzt du identisch zu queries.sold.

7.7  MENGE, FORMAT, PREIS
  quantity > 1 nur, wenn der Verkaeufer ausdruecklich mehrere identische
  Stuecke anbietet. format ist FIXED_PRICE, ausser der Verkaeufer verlangt
  ausdruecklich eine Auktion. user_price und best_offer nur bei
  ausdruecklicher Nennung.

7.8  UNSICHERHEIT
  uncertain = true und eine question NUR, wenn du das Stueck ueberhaupt nicht
  zuordnen kannst oder eine Information fehlt, ohne die das Listing
  irrefuehrend waere (etwa echt gegen Replika bei Luxusware).
  Variantenunsicherheiten gehoeren in die Konfidenz-Felder und in
  "assumptions", nicht in eine blockierende Rueckfrage.

═══════════════════════════════════════════════════════════════════════════
8  SCHEMA
═══════════════════════════════════════════════════════════════════════════
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
    "title_native": null,
    "franchise": null,
    "series": null,
    "number": null,
    "number_kind": null,
    "set_name": null,
    "set_code": null,
    "set_total": null,
    "publisher": null,
    "release_year": null,
    "edition": null,
    "variant": null,
    "isbn": null,
    "upc": null,
    "confidence": {"number": null, "set_code": null, "release_year": null,
                   "publisher": null, "edition": null, "variant": null}
  },

  "queries": {"sold": "", "catalog": "", "ebay": ""},

  "photos_meta": {"label_index": null, "front_index": 0, "back_index": null}
}

Wenn holder == "slab", hat "slab" diese Form:
{
  "grader": null,
  "grader_raw": null,
  "scale": "unknown",
  "grade": null,
  "grade_numeric": null,
  "subgrades": null,
  "seal_rating": null,
  "label_color": null,
  "cert_number": null,
  "cert_channel": null,
  "cert_verified": false,
  "positive_evidence": false,
  "confidence": {"grader": null, "grade": null, "cert_number": null,
                 "subgrades": null}
}

"graded_info" fuellst du ausschliesslich aus "slab" und nur dann, wenn
positive_evidence true ist:
  {"grader": <slab.grader>, "grade": <slab.grade>,
   "cert_number": <slab.cert_number>}
Sonst ist graded_info null.

estimated_price_range_eur ist Pflicht und bleibt deine ehrliche Einschaetzung
aus Produktwissen: wofuer verkauft sich GENAU dieses Stueck in diesem Zustand
realistisch auf eBay. Alltagsware ehrlich niedrig ansetzen. Diese Spanne ist
ein Plausibilitaets-Anker gegen falsche Vergleichstreffer, kein Marktwert —
sie wird dem Nutzer nie als Marktwert angezeigt.

═══════════════════════════════════════════════════════════════════════════
9  LETZTE PRUEFUNG VOR DER ANTWORT
═══════════════════════════════════════════════════════════════════════════
Gehe diese Punkte durch, bevor du antwortest:
  1. Ist die Ausgabe gueltiges JSON, ohne Kommentare, ohne Text davor oder
     danach?
  2. Steht in item_class die Warenart, die das Stueck TATSAECHLICH ist —
     nicht die, die die Reihe nahelegt?
  3. Ist jede Zertifikatsnummer im Bild vollstaendig lesbar? Wenn nicht:
     null.
  4. Ist jedes Grader-Kuerzel aus der Liste in 5.4? Steht nirgends "Beckett"
     als Kuerzel?
  5. Sind Subgrades entweder vollstaendig oder null?
  6. Hat jede nicht-null-Aussage eine Konfidenz, und jede null-Aussage
     Konfidenz null?
  7. Enthaelt ein Endtext ein Ausrufezeichen, ein Emoji oder ein "Wir"?
     Dann umformulieren.
  8. Hast du irgendwo einen Wert eingetragen, den du nicht am Bild oder in
     den Angaben des Verkaeufers belegen kannst? Dann ersetze ihn durch null.
```

---

## B3. Was der neue Prompt besser macht

| Problem heute (B1) | Behebung |
|---|---|
| „Wähle das WAHRSCHEINLICHSTE" (Z. 40-41) | Grundgesetz im Kopf: `null` statt Rateraten, plus Prüfpunkt 8 am Ende. Die Vermutungs-Lizenz gilt nur noch für den Listing-Text, nicht für Identität und Grading. |
| Keine Warenart | `classification.item_class` als Pflichtfeld mit expliziten Unterscheidungshilfen — und dem One-Piece-Fall als benanntem Beispiel. Das ist die Feldebene, auf der das Domain-Gate der Preiskette aufsetzt. |
| `//`-Kommentar im JSON-Beispiel (Z. 76) | Schema in Abschnitt 8 ist **reines, gültiges JSON**. Alle Erklärungen stehen in den nummerierten Regeln davor. Das Modell kopiert kein ungültiges Muster mehr. |
| `uncertain` wird weggeworfen | Ersetzt durch Konfidenz-Felder, die *im* `analysis`-Blob überleben (`ANALYSE_FELDER`) und nicht gepoppt werden. Der bestehende `uncertain`-Mechanismus bleibt für die harten Fälle bestehen. |
| Grading zu dünn | `scale`, `subgrades` (alle-oder-keiner), `seal_rating`, `cert_channel`, `grader_raw`, `positive_evidence` und Konfidenz je Feld. Die 9.0-vs-9.4-Kollision wird sichtbar, statt still zu passieren. |
| Kürzel unvollständig, `beckett→BGS` falsch | Vollständige Whitelist inkl. BVG/BCCG/BAS/CBCS/TAG/CSG/MANA und ein expliziter Entscheidungsbaum für Beckett. Bei Unentscheidbarkeit `grader: null` — **nicht** BGS. |
| Titel-Regex zerstört Titel | Der Prompt schreibt das korrekte Kürzel **selbst** in der Form `BGS 9.0` und nur bei bekanntem Grader. Damit greift `re.search(rf"\b{richtig}\b", …)` in `_grader_im_titel_richtigstellen` (`:155`) und die gefährliche `re.sub`-Zeile (`:158`) wird nicht mehr erreicht. Das ist eine Entschärfung ohne Code-Änderung — der eigentliche Fix der Regex bleibt trotzdem nötig. |
| Eine Query für drei Systeme | `queries.sold` / `.catalog` / `.ebay`, jeweils mit der Syntax des Zielsystems. `search_query_for_pricing` bleibt als Alias erhalten, damit `app_api.py:799,821,1010` unverändert laufen. |
| Wording-Regeln fehlen | Abschnitt 7.4 nennt sie einzeln, mit Gegenbeispielen, und Prüfpunkt 7 kontrolliert sie. |
| Keine Sprach-/Set-Disziplin | Abschnitt 6: Sprache nur aus gedrucktem Text, Set-Code wörtlich und ohne Umrechnung JP↔EN, Kartennummer als Pflicht-oder-null. Das ist die Voraussetzung dafür, `language` und `edition` in den `card_key` aufzunehmen. |

**Was der neue Prompt bewusst NICHT tut:** Er verifiziert keine Zertifikatsnummern (`cert_verified` ist hart false), er berechnet keine Preise, und er wählt keine Preisquelle. Das sind Aufgaben der Pipeline. Der Prompt liefert nur belegbare Beobachtungen und markiert ehrlich, wie sicher sie sind.

---

## B4. Zwei Hinweise zur Einbindung

**1. Der Prompt ist länger — deshalb Prompt-Caching aktivieren.** Heute wird der System-Prompt als einfache Zeichenkette übergeben (`bot/claude_client.py:258`: `system=SYSTEM_PROMPT`). Bei zehntausenden Scans zahlt jeder Aufruf den vollen Eingabepreis. Da der Prompt statisch ist, genügt die Blockform mit `cache_control`:

```python
system=[{"type": "text", "text": SYSTEM_PROMPT,
         "cache_control": {"type": "ephemeral"}}],
```

Cache-Treffer kosten rund ein Zehntel des Eingabepreises, ein Cache-Schreibvorgang das 1,25-fache — ab dem zweiten Aufruf innerhalb des Fensters rechnet es sich. Kontrolle über `resp.usage.cache_read_input_tokens`. Wichtig: Das konfigurierte Modell ist laut `bot/config.py:113` per Default `claude-sonnet-4-6`; dessen minimale cachebare Präfixlänge liegt bei 1024 Tokens — der neue Prompt liegt deutlich darüber, der alte hätte es knapp verfehlt. Der Prompt darf dafür **byte-identisch** bleiben: kein Datum, keine Item-ID, kein Zeitstempel hineininterpolieren.

**2. `parse_listing_json` bleibt unverändert lauffähig.** Die Pflichtfeldprüfung (`:127-130`) verlangt `title`, `description_html`, `category_query`, `condition`, `search_query_for_pricing` — alle fünf liefert das neue Schema. Sinnvolle Ergänzungen dort, in dieser Reihenfolge: (a) `classification.item_class` gegen die Enum-Liste validieren und bei Verstoß auf `"other"` setzen, (b) `slab.grader` gegen die erweiterte Whitelist prüfen und sonst auf `None` setzen, (c) die `re.sub`-Zeile `:158` entschärfen oder entfernen.

---

## Relevante Dateien (nur gelesen)

- `/Users/smorty/ebay-bot/bot/claude_client.py:29-95` — heutiger SYSTEM_PROMPT, ersetzt durch B2
- `/Users/smorty/ebay-bot/bot/claude_client.py:115-132` — `parse_listing_json`, Pflichtfelder
- `/Users/smorty/ebay-bot/bot/claude_client.py:139-161` — `_GRADER_KUERZEL` und die fehlerhafte Titel-Regex
- `/Users/smorty/ebay-bot/bot/claude_client.py:253-265` — `_call`, Ansatzpunkt für Prompt-Caching
- `/Users/smorty/ebay-bot/web/app_api.py:725-733` — **`PREIS_FELDER` / `ANALYSE_FELDER`: die einzige Persistenz-Änderung**
- `/Users/smorty/ebay-bot/web/app_api.py:514-560` — `item_public`, optionale Anzeige-Erweiterung
- `/Users/smorty/ebay-bot/web/app_api.py:789-816`, `:995-1022` — Katalog-Aufruf, Ort der Projektion
- `/Users/smorty/ebay-bot/web/app_api.py:906` — `listing.pop("uncertain", None)`
- `/Users/smorty/ebay-bot/web/catalog.py:36-40, 48-60` — `card_prices.detail` (JSON, kein DDL) und `card_key_of`
- `/Users/smorty/ebay-bot/web/pricecharting.py:87-92` — `lookup_pc` ohne Typbindung
- `/Users/smorty/sero-app/web/sero.js:501-514` — `SOURCE_INFO`, maßgeblich für das Legacy-Enum
- `/Users/smorty/ebay-bot/bot/config.py:113` — konfiguriertes Modell (`claude-sonnet-4-6`)
