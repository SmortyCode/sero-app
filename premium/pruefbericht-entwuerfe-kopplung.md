# Kopplung Stück ↔ Entwurf — Spezifikation

Alle Zeilenangaben: `/Users/smorty/ebay-bot/web/app_api.py`, `/Users/smorty/ebay-bot/bot/drafts.py`, `/Users/smorty/ebay-bot/web/catalog.py`, `/Users/smorty/sero-app/web/sero.js`. Nur gelesen, nichts geändert.

---

## 1. Wie die Kopplung HEUTE funktioniert

**Sie ist einseitig und ungesichert.** Es gibt genau einen Zeiger: `item["draft_id"]` im JSON-Blob des Stücks. Der Entwurf weiß nichts von seinem Stück. Der Rückweg ist ein linearer Scan über alle Stücke — `item_by_draft`, app_api.py:400-404, und noch einmal in `sync_sales_status` → `mark_item_sold`, app_api.py:2483-2490 (dort sogar über *alle* Konten). Kein Index, keine Eindeutigkeitsprüfung, kein Aufräumen bei Löschungen.

Die **einzige** Wiederverwendungs-Prüfung steht in `_list_collection_item`, app_api.py:2116-2119:

```python
if item.get("draft_id"):
    existing = own_draft(item["draft_id"], account)
    if existing and existing.get("status") not in ("ended",):
        return {"ok": True, "draft_id": item["draft_id"], "existing": True}
```

Danach: `store.create_draft(...)` (app_api.py:2125), `item["draft_id"] = draft_id`, `col_save` (2130-2131) — der alte Zeiger wird **überschrieben**, der alte Entwurf bleibt liegen.

### Wo sie bricht

| # | Bruchstelle | Datei:Zeile | Folge |
|---|---|---|---|
| B1 | `status == 'ended'` fällt durch die Prüfung → neuer Entwurf, alter wird verwaist | app_api.py:2118 | Der beendete Entwurf hat kein Stück mehr. `mark_item_sold` (2483-2490) findet nie ein Stück → **ein Verkauf wird nach dem Neu-Einstellen nie mehr am Stück vermerkt**. Der ended-Eintrag hängt für immer ohne `item_id` im Verkauf-Tab (2954). |
| B2 | „Entwurf verwerfen" löscht den Draft, setzt aber `item["draft_id"]` **nicht** zurück | app_api.py:1483-1487 (`discard`) | Das Stück zeigt ins Leere. Frontend prüft nur auf Vorhandensein: `if (i.draft_id) return openItemDetail(i.id, "sell")` (sero.js:2586, Label 2570) → **das Stück lässt sich über das Sheet nie wieder listen**, der Knopf heißt für immer „Listing-Entwurf öffnen". |
| B3 | Stück löschen rührt den Entwurf nicht an | app_api.py:2058-2082 (`delete_collection_item`) | Papierkorb für das Stück, **Entwurf bleibt** — die häufigste Verwaisungs-Quelle. Genau das Muster „4× Grand Theft Auto III": vier Scan-/Löschversuche desselben Spiels, vier Entwürfe. |
| B4 | Auto-Listen im Scan-Modus „Verkaufen" | sero.js:1894 (`listNow(id)` in `sellWatch`), Registrierung 2471/2520/2548 | **Jeder** fertige Scan erzeugt ungefragt einen Entwurf; eine Sammelaufnahme (`item_ids`, app_api.py:1989) gleich mehrere. Ohne Dubletten-Prüfung ist das der Mengen-Verstärker. |
| B5 | `own_draft` bindet an `uid_for(account)` = telegram_id **oder** `10**15 + account_id` | app_api.py:326-330, 90, 226 | Ändert sich die Telegram-Verknüpfung, gehören alle bestehenden Entwürfe plötzlich einer anderen `chat_id` → `own_draft` liefert None → jedes Listen legt neu an, die alten sind unsichtbar und verwaist. Entwürfe aus dem Telegram-Bot (bot/main.py:1032) haben ohnehin nie ein Stück. |
| B6 | `stale_draft_ids` löscht unfertige Entwürfe hart, ohne Stücke zu pflegen | drafts.py:292-306, Aufruf nur bot/main.py:493 | Erzeugt genau den Zustand aus B2 (Zeiger ins Leere), diesmal ohne Nutzeraktion. |
| B7 | Wiederverwendung ist **passiv** | app_api.py:2119 | Bei `error`, `uncertain` oder hängendem `downloading`/`analyzing` gibt der Server nur `existing: true` zurück und startet **nichts** neu; die mitgeschickte Verkaufs-Vorlage (Format/Laufzeit/Preisregel, 2131-2141) wird kommentarlos verworfen. Für Sven: „ich tippe Listen und es passiert nichts". |
| B8 | Produkt-Identität existiert, wird aber nicht genutzt | `card_key` gesetzt in app_api.py:756 und 948, erzeugt von `catalog.card_key_of` (catalog.py:48-60) | `card_key` dient ausschließlich dem Preis-Katalog. Es gibt **keinen** Vergleich Stück↔Stück und keinen Vergleich Entwurf↔Stück. |

Merksatz: Heute ist `draft_id` eine Notiz, keine Beziehung. Jede Löschung an einem der beiden Enden hinterlässt Müll.

---

## 2. Zielbild: die Invariante

> **Ein Stück hat zu jedem Zeitpunkt höchstens EINEN lebenden Entwurf. Jeder lebende Entwurf gehört zu genau EINEM Stück.**

- **lebend** = Status `new`, `downloading`, `analyzing`, `uncertain`, `ready`, `dry_run_done`, `error`, `published`
- **archiviert** = `ended` (bleibt für immer erhalten, zählt aber nicht gegen die Invariante)
- **verwaist** = lebender Entwurf ohne Stück → darf es künftig nicht mehr geben

### Nötige Datenmodell-Ergänzungen (Spezifikation, kein Patch)

1. **Gegenrichtung:** `draft["item_id"]` und `draft["account_id"]` werden beim Anlegen mitgeschrieben (Stelle: app_api.py:2125-2131). Damit ist ein verwaister Entwurf *erkennbar*, und `item_by_draft` (400) sowie `mark_item_sold` (2483) brauchen keinen Full Scan mehr.
2. **Historie am Stück:** `item["draft_ids"]` = Liste **aller** je erzeugten Entwürfe. `item["draft_id"]` bleibt der *aktuelle lebende*. Beim Neu-Einstellen nach `ended` geht damit nichts verloren.
3. **Eindeutigkeit beim Setzen:** bevor `item["draft_id"] = X` geschrieben wird, muss geprüft sein, dass kein anderes Stück desselben Kontos X hält. (Die Sperre `_list_locks`, app_api.py:2087-2100, deckt heute nur denselben Item-Key ab.)
4. **Aufräumpflicht:** jede Stelle, die einen Entwurf löscht oder beendet, pflegt die Referenz mit — drei Stellen heute: `discard` (1485), `end` (1528), `stale_draft_ids` (drafts.py:292).
5. **uid-Migration:** wird eine Telegram-ID verknüpft/gelöst, müssen die `chat_id`s der bestehenden Entwürfe einmalig umgeschrieben werden, sonst reißt jede Kopplung (B5).

---

## 3. Entscheidungstabelle A — Nutzer tippt „Auf eBay listen" an einem Stück

Aufrufer: sero.js:2373 (`listNow`), 2587 (Sheet), 3394 (Detail-Knopf) → `POST /collection/item/{id}/list`.

| Draft-Status des Stücks | Verhalten des Servers | Neuer Entwurf? | Knopf-Beschriftung / Rückmeldung |
|---|---|---|---|
| **kein Entwurf** | anlegen wie heute | ja | „Auf eBay listen" → „Entwurf wird vorbereitet …" |
| **Zeiger ins Leere** (Entwurf gelöscht / fremde chat_id) | `draft_id` sofort auf `None` normalisieren, dann behandeln wie „kein Entwurf". **Nie** `existing: true` melden | ja | wie Erstlistung, keine Fehlermeldung |
| **downloading / analyzing** | wiederverwenden, nichts neu starten; Fortschritt zurückgeben. Abweichende Vorlage erst nach Pipeline-Ende anwenden | nein | „Läuft bereits …" + Fortschrittszeile (`latest_stage`, app_api.py:332) |
| **uncertain** (Rückfrage offen) | wiederverwenden, offene Frage zurückgeben | nein | „Rückfrage offen — bitte beantworten" |
| **ready / dry_run_done** | wiederverwenden **und aktualisieren**: Vorlage (Format/Laufzeit/Preisregel, Body-Felder aus 2131-2141) und aktuellen Marktwert (`preset["market_value"]`, 2148-2152) in **denselben** Entwurf schreiben. Fotos/Analyse bleiben. Neu erzeugen nur, wenn sich die **Fotos des Stücks** seit Erstellung geändert haben | nein | „Entwurf aktualisiert" |
| **error** | wiederverwenden **und Pipeline neu starten** (`app_run_pipeline`, 987), vorher `loesche_fehler` (423) | nein | „Neuer Versuch läuft" |
| **published** | **niemals** neuer Entwurf. Auf `app_run_update` verweisen (Aktion `save`, 1511-1516) | nein | Knopf heißt „Listing bearbeiten" |
| **ended**, Stück nicht verkauft | neuen Entwurf anlegen, alten aber **nicht verwaisen**: alte ID nach `item["draft_ids"]`, `draft["item_id"]` am alten Entwurf **bleibt stehen** | ja | „Erneut einstellen" |
| **ended**, Stück `sold_ts` gesetzt | Nachfrage vor dem Anlegen: „Dieses Stück ist als verkauft markiert. Trotzdem erneut einstellen?" | nur nach Bestätigung | „Erneut einstellen (verkauft)" |

Antwort-Erweiterung für die Oberfläche: `{"ok": true, "draft_id": …, "draft_status": …, "action": "created" | "reused" | "updated" | "restarted"}`. Heute liefert der Server nur `existing: true` (2119), und `listNow` (sero.js:2380) meldet **in jedem Fall** „Entwurf erstellt" — das ist die Rückmeldung, die Sven glauben lässt, es sei ein neuer entstanden.

## 4. Entscheidungstabelle B — Ereignisse, die die Referenz pflegen müssen

| Ereignis | Stelle heute | Was zusätzlich passieren muss |
|---|---|---|
| Entwurf verworfen (`discard`) | app_api.py:1483-1487 | `item["draft_id"] = None` (über `draft["item_id"]`) — behebt B2 |
| Listing beendet (`end`) | app_api.py:1526-1533 | Entwurf bleibt als `ended` erhalten; `item["draft_id"] = None`, ID wandert nach `item["draft_ids"]` |
| Sales-Sync setzt `ended` | app_api.py:2528-2535 | dasselbe; `mark_item_sold` über `draft["item_id"]` statt Full Scan |
| Stück gelöscht (Papierkorb) | app_api.py:2058-2082 | `published` → Löschen **verweigern** mit Hinweis „erst Listing beenden". `ready/error/uncertain/analyzing` → Entwurf mit in den Papierkorb (ID im `item.json` vermerken, damit Wiederherstellen möglich bleibt). `ended` → stehen lassen, als `item_deleted` markieren — behebt B3 |
| Stück wiederhergestellt | – | `draft_id` nur zurücksetzen, wenn der Entwurf noch existiert |
| Alt-Bereinigung `stale_draft_ids` | drafts.py:292-306 | verlinkte Entwürfe überspringen; falls doch gelöscht, Stück mit aufräumen |
| Telegram verknüpft/gelöst | app_api.py:226 | `chat_id` der bestehenden Entwürfe migrieren — behebt B5 |
| Auto-Listen nach Scan | sero.js:1894 | nur auslösen, wenn keine Produkt-Dublette mit lebendem Entwurf existiert (siehe §5) |

---

## 5. Zweites Foto desselben Produkts (Svens Frage 2)

**Regel: erkennen JA, automatisch zusammenführen NEIN.**

### Begründung gegen automatisches Zusammenführen
Bei einem Händler ist das zweite Exemplar der **Normalfall**, nicht der Fehler: anderer Zustand, andere Vollständigkeit (OVP/Anleitung), andere Grading-Note, anderer Einkaufspreis, andere Fotos. Ein automatisches Merge würde `purchase_price`, `condition`, `graded` und die Fotos eines echten zweiten Exemplars stillschweigend vernichten — und über `quantity` würde ein Verkauf die falsche Menge abbuchen. Ein Merge ist zudem nicht umkehrbar; ein Hinweis kostet einen Fingertipp, ein falsches Merge kostet Daten. Nach dem 02.08. gilt: **Vorschlagen statt Ausführen.**

### Ausnahme, die stumm bleiben darf
Identische Bild-Bytes (SHA1 — die Stage-Ablage rechnet ihn bereits, app_api.py:1836-1842) sind kein zweites Exemplar, sondern derselbe Scan doppelt. Der wird wie bisher stumm verworfen; daraus entsteht gar kein zweites Stück und damit auch kein zweiter Entwurf.

### Erkennung — wann und womit
Zeitpunkt: **nach** der Analyse, an der Stelle, an der `card_key` gesetzt wird (app_api.py:948). Vorher ist die Identität noch nicht bekannt.

Vergleich in dieser Reihenfolge:
1. `card_key` identisch **und** `catalog.grade_bucket(item["graded"])` identisch → sicherer Treffer (catalog.py:48-60).
2. `card_key` beginnt mit `solo:` (keine Katalog-Identität, catalog.py:50-53) → Fallback über `catalog._norm(name)` + Kategorie, Ähnlichkeit ≥ 0,9 → Treffer *mit Unsicherheitsmarkierung*.
3. Kein Treffer → normal weiter.

### Verhalten bei Treffer

| Lage des gefundenen Vorgänger-Stücks | Verhalten |
|---|---|
| hat lebenden Entwurf (`ready`/`error`/`uncertain`) | Hinweis-Banner am neuen Stück: „Dieses Produkt hast du schon — dafür liegt ein Entwurf im Verkauf." Aktionen: **[Zum Entwurf]** · **[Als 2. Exemplar behalten]** · **[Zusammenführen]** |
| hat `published`-Entwurf | Banner: „Steht bereits bei eBay." Aktionen: [Listing ansehen] · [Als 2. Exemplar behalten]. Kein Merge-Angebot, solange das Listing aktiv ist |
| hat `ended`/keinen Entwurf | reiner Hinweis „2. Exemplar", keine Aktion erzwungen |
| Scan-Modus „Verkaufen" (Auto-Listen) | **Auto-Listen aussetzen**, Banner zeigen, auf Sven warten. Genau hier entsteht heute die Entwurfs-Flut (B4) |

„Zusammenführen" bedeutet: `quantity + 1`, Fotos ans vorhandene Stück anhängen, das neue Stück in den **Papierkorb** (nicht löschen, app_api.py:2064-2077) — damit umkehrbar.

**Nicht erlaubt:** das neue Stück an den *fremden* Entwurf hängen. Ein Entwurf gehört genau einem Stück, sonst kippt die Invariante und die Entwurfs-Fotos passen nicht mehr zum Stück.

---

## 6. Aufräum-Routine für die 7 Altlasten

**Eiserne Regel: `published` und `ended` werden NIE gelöscht und nie automatisch verändert.** Kein Cron, der ungefragt aufräumt. Zweistufig: Trockenlauf → Bestätigung → Ausführung, mit Protokolldatei analog `_stage_log` (app_api.py:1743-1752).

### Stufe 1 — Klassifizierung (rein lesend)
Für jeden Entwurf mit `chat_id == uid_for(account)`:

| Klasse | Bedingung | Vorschlag |
|---|---|---|
| **A — gesund** | irgendein Stück zeigt darauf | nichts tun, nur `draft["item_id"]` nachtragen (Rückrichtung herstellen) |
| **B — veröffentlicht/beendet, ohne Stück** | `status` in (`published`, `ended`) | **nie löschen.** Wenn genau ein Stück per `card_key`/Titel passt und selbst keinen lebenden Entwurf hat: Verknüpfung *vorschlagen*. Sonst stehen lassen und im Verkauf-Tab als „ohne Stück" markieren |
| **C1 — unfertig, adoptierbar** | `status` in (`new`,`downloading`,`analyzing`,`uncertain`,`error`,`ready`), kein Stück zeigt darauf, aber genau ein Stück passt (gleicher `card_key`, sonst normalisierter Titel) und hat keinen lebenden Entwurf | **adoptieren**: `item["draft_id"]` setzen + `draft["item_id"]` setzen |
| **C2 — unfertig, mehrfach** | mehrere unfertige Entwürfe passen auf dasselbe Stück (der GTA-III-Fall) | **jüngsten** (`updated_at` max) adoptieren, die übrigen archivieren |
| **C3 — unfertig, heimatlos** | kein passendes Stück | **archivieren, nicht löschen** |

### Stufe 2 — „Archivieren" heißt Soft-Delete
`draft["archived_at"] = time.time()`, Ausblendung aus dem Verkauf-Tab (Filter in app_api.py:2951-2955), Fotos bleiben zunächst liegen. Erst nach 30 Tagen echtes Löschen — dieselbe Frist wie der Item-Papierkorb (app_api.py:2593-2597). So ist jeder Fehlgriff umkehrbar.

### Stufe 3 — Trockenlauf-Ausgabe für Sven
Pro Entwurf: `draft_id`, Status, Titel, Alter, Foto-Anzahl, Klasse, Vorschlag, ggf. Ziel-Stück. Erwartetes Ergebnis bei der aktuellen Lage: 4× GTA III → einer wird adoptiert, drei archiviert; die restlichen drei verwaisten je nach Titel adoptiert oder archiviert; `published`/`ended` unangetastet.

### Danach
Ein Wächter-Job (nur **meldend**, nicht löschend) zählt einmal täglich verwaiste lebende Entwürfe und Zeiger ins Leere. Solange die Regeln aus §3/§4 greifen, muss diese Zahl 0 bleiben — das ist der Regressionstest im Betrieb.

---

## 7. Rückmeldung in der Oberfläche

1. **Statuszeile am Stück** — `item_public` liefert `draft_status` bereits (app_api.py:481-487, 521-523); die Sammlungskachel zeigt ihn nicht. Nötig: „Entwurf bereit" / „Aktiv auf eBay" / „Beendet" / „Fehler beim Listen".
2. **Selbstbeschriftender Knopf** statt der heutigen Zwei-Fälle-Logik (sero.js:2570): „Auf eBay listen" · „Entwurf öffnen" · „Entwurf aktualisieren" · „Erneut versuchen" · „Listing bearbeiten" · „Erneut einstellen".
3. **Ehrliche Toasts** — `listNow` sagt heute immer „Entwurf erstellt — liegt im Verkauf-Tab" (sero.js:2380). Muss dem `action`-Feld folgen: erstellt / geöffnet / aktualisiert / neu gestartet.
4. **Verkauf-Tab:** Einträge ohne Stück sichtbar markieren („ohne Stück") — `entry["item_id"]` ist dort bereits `None` (app_api.py:2954), es wird nur nicht angezeigt — mit Aktion „Stück zuordnen".
5. **Dubletten-Hinweis als Banner am Stück, nicht als Sheet.** Ein Sheet an dieser Stelle würde beim Auto-Listen direkt nach dem Scan aufpoppen und mit Beschwerde 1 kollidieren (`recede` auf `#viewApp` bleibt hängen). Das Banner ist zudem nicht wegtippbar-verloren.
6. **Beim Löschen eines Stücks:** „Dazu gibt es einen Entwurf — er wird mitgelöscht." bzw. bei aktivem Listing: „Das Stück steht bei eBay. Bitte erst das Listing beenden."

---

## 8. Abnahme-Kriterien

- Zweimal „Listen" am selben Stück → genau ein Entwurf, egal bei welchem Status.
- Entwurf verwerfen → Stück ist sofort wieder listbar (heute nicht, B2).
- Beenden → neu einstellen → verkauft: der Verkauf landet am Stück (heute nicht, B1).
- Stück löschen → kein lebender Entwurf bleibt übrig (heute nicht, B3).
- Zweites Foto desselben Spiels → **kein** zweiter Entwurf ohne ausdrückliche Bestätigung.
- Zähler „verwaiste lebende Entwürfe" = 0; `published`/`ended` unverändert in Zahl und Inhalt.