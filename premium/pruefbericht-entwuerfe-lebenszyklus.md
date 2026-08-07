# Entwurfs-Lebenszyklus in SERO — vollständige Aufklärung

**Arbeitsweise:** strikt read-only. DB nur über `sqlite3 -readonly "file:/Users/smorty/ebay-bot/data.db?mode=ro"` mit SELECTs, Dateien nur gelesen/gehasht (`md5 -q`). Nichts geschrieben, kein Server gestartet, kein POST.

**⚠️ Warnung:** `/Users/smorty/ebay-bot/web/app_api.py` wurde **während meiner Analyse von einem anderen Prozess/Agenten verändert** (mtime 13:31, Datei wuchs um ~2,9 KB; im Lösch-Endpunkt steht jetzt neuer Code samt Kommentar „Svens vier GTA-III-Entwürfe kamen genau so zustande"). Alle Zeilennummern unten beziehen sich auf den Stand **13:31 Uhr**. Falls parallel weiter editiert wird, verschieben sie sich.

---

## (a) Tabelle: JEDE Stelle, die einen Entwurf erzeugt

Es gibt im gesamten Projekt genau **drei** Stellen, die `store.create_draft()` aufrufen (`bot/drafts.py:251` = einziger INSERT in `drafts`):

| # | Code-Stelle | Auslöser (Endpunkt / UI) | Wird am Stück verankert? | Was passiert mit dem vorherigen Entwurf des Stücks? |
|---|---|---|---|---|
| 1 | `web/app_api.py:2180` in `_list_collection_item` (ab :2158) | `POST /collection/item/{id}/list` (`app_api.py:2144`) — UI: Detail-Knopf „Auf eBay listen" (`sero.js:3404`), Long-Press-Menü „sell" (`sero.js:2587`), **Auto-List im Scanner-Verkaufsmodus** (`sero.js:1894` → `listNow()` `sero.js:2373`), Verkaufs-Vorlage-Sheet | **Ja**, sofort: `item["draft_id"] = draft_id` + `col_save(...)` in `app_api.py:2185-2186`, noch VOR dem Fotokopieren, abgesichert durch eine Pro-Item-Sperre (`_list_locks`, :2140-2156) | **Nur bedingt geschützt:** Guard :2170-2174 gibt den alten Entwurf zurück, wenn er existiert und Status ≠ `ended`. Ist er `ended` → **neuer Entwurf, `draft_id` wird überschrieben, der alte Entwurf wird weder beendet noch gelöscht → verwaist.** Ist er gelöscht (`discard`) → neuer Entwurf (unkritisch). |
| 2 | `bot/main.py:1032` in `process_photo_group` | Foto an den Telegram-Bot (@…). Nutzt dieselbe `drafts`-Tabelle und dieselbe `chat_id` wie die App (`uid_for`, `app_api.py:223-226`, alle 29 Entwürfe haben `chat_id 5694742134`) | **Nein** — es entsteht gar kein `collection_item` | Kein Bezug zu Stücken; **jeder so erzeugte Entwurf ist strukturell verwaist**. Das erklärt die 15 Juli-Entwürfe (Pokémon/GTA, alle `ended`). |
| 3 | — | (keine weitere) `app_run_pipeline` (:1023 ff.), `app_run_upload`, `app_run_update`, `draft_action` `regen`/`imgren`/`answer`, `POST /sales/publish-drafts` (:3170) legen **nie** einen neuen Entwurf an, sie mutieren nur den vorhandenen (`store.update_draft`). `preset_listing` ist reine Abkürzung (überspringt die Foto-Analyse, :1035-1039) — kein Entwurfs-Erzeuger. | | |

**Gegenrichtung (Entwurf → Stück), erzeugt keine Entwürfe, aber Stück-Dubletten:**

| Stelle | Auslöser | Wirkung |
|---|---|---|
| `POST /collection/adopt/{draft_id}` (`app_api.py:2671`) | UI: **jeder Klick auf eine Zeile im Verkauf-Tab, die kein Stück hat** (`sero.js:2743`) — passiert automatisch, ohne Rückfrage | Legt ein **neues** `collection_item` an mit `draft_id` = Entwurf, `est_value` = `draft["price"]`, Fotos = Kopien der **Render-Bilder** aus `tmp/<draft>/`. Dedupe nur über `draft_id` (:2676-2678) — ein inhaltsgleiches Stück ohne `draft_id` wird nicht erkannt. Bei Auktionen ist `price` = 1,00 € → Stück landet mit **Wert 1,00 €** in der Sammlung. |
| `POST /collection/import` (`app_api.py:2718`) | „Bestehende eBay-Listings übernehmen" | Wie adopt, aber nur für `published` Entwürfe; überspringt bereits verlinkte. |

---

## (b) Welche Pfade verwaisen lassen — und warum

1. **Stück löschen ließ den Entwurf stehen** — `delete_collection_item` (`app_api.py:2101`). Bis heute Mittag wurde nur das Item gelöscht (Papierkorb), der Entwurf blieb unangetastet. **Das ist die Hauptquelle der 7 verwaisten Entwürfe.** (Ein paralleler Agent hat um 13:31 in :2121-2128 eine Löschung ergänzt — nur für Entwürfe, die nicht `published`/`ended`/`dry_run_done` sind.)
2. **Neu-Listen nach `ended`** — Guard `app_api.py:2170-2174`: Bei einem beendeten Listing wird ein neuer Entwurf erzeugt und `item["draft_id"]` überschrieben. Der alte, beendete Entwurf verliert seinen Anker. So sind die meisten der 17 `ended`-Entwürfe unverlinkt.
3. **Telegram-Pfad** (`bot/main.py:1032`) — nie verankert, per Konstruktion verwaist.
4. **Umgekehrte Lücke: `discard`** (`app_api.py:1516-1521`) löscht den Entwurf, räumt aber `item["draft_id"]` **nicht** auf → Stück zeigt auf eine Leiche. Gleiches gilt für die Bot-Aufräumroutine (`bot/main.py:493-494`: unfertige Entwürfe > 7 Tage werden gelöscht) — `draft_id` am Stück bleibt stehen. Es gibt im ganzen Backend **keine einzige Stelle, die `item["draft_id"]` wieder entfernt** (nur ein Schreibzugriff insgesamt: `app_api.py:2185`).
5. **Live-Listing ohne Stück:** Entwurf `74925bead28e` ist `published` mit `listing_id 147480067874` und hat **kein** Sammlungsstück. Löschen des Stücks beendet das eBay-Angebot nicht — Sven hat aktuell **zwei parallele Live-Listings für „Vice City"** (147480067874 **und** 147480090061, letzteres via `9f8f51762eca`, verlinkt an Item `e34807d4bff5`). Das ist der teuerste Effekt der fehlenden Kopplung.
6. **Kein Dubletten-Schutz beim Anlegen von Stücken:** `POST /collection/items` (:1717), `/collection/items-from-stage` (:1924), `/collection/scan-batch` (:1967) legen jedes Mal ein frisches Item mit neuer UUID an. Die Funktion `aehnliches_stueck()` (`app_api.py:400-421`) — geschrieben genau für Svens Wunsch — wird **nur in der Detail-Ansicht als Hinweis** benutzt (`app_api.py:2046`, Feld `dublette`), nicht beim Upload und **nicht** vor `create_draft`. Sie ist außerdem rein textbasiert (`card_key` oder identischer Name) und greift bei „noch nicht analysiert" gar nicht.

**Zahlen (nur gelesen):** 29 Entwürfe (17 `ended`, 6 `published`, 6 `ready`), 11 Stücke, 23 Entwürfe ohne Stück, 6 verlinkt.

---

## (c) Wie die vier GTA-III-Entwürfe entstanden sind

**Forensik:** Ich habe die Foto-Bytes verglichen (`md5`) — `tmp/<draft>/00.png` ist die Kopie von `item["photos"][0]`, und `_trash/<item>/item.json` enthält den Item-Stand beim Löschen.

Die vier Entwürfe sind **nicht** aus dem Nichts entstanden — jeder hatte ein eigenes Sammlungsstück, alle vier Stücke wurden gelöscht:

| Entwurf | erstellt | Format/Preis | Quell-Stück (heute im Papierkorb) | Foto-Hash (00) |
|---|---|---|---|---|
| `0adcfd186025` | 12:39:53 | Auktion 1 €, 5 T | Stück gelöscht, Fotos nicht mehr auffindbar | `08d010f2…` (.jpg) |
| `eb4ad01ac069` | 12:55:06 | Auktion 1 €, 3 T, **USK-18-Fehler** | `04552e60a936` (gelöscht 13:07:05) | `2c54d8f3…` |
| `e924e9065744` | 13:08:06 | Sofortkauf | `28e73ecbef2d` (gelöscht 13:18:21) | `beeacdc5…` |
| `3d6b5d3f8671` | 13:08:17 | Auktion 1 €, 5 T, **USK-18-Fehler** | `1ae290953229` (gelöscht 13:18:27) | `2c54d8f3…` |

**Die wahrscheinlichste Entstehungsgeschichte:**

1. **Beschwerde 1 erzeugt die Stück-Dubletten.** Die Fotos von `eb4ad01ac069` und `3d6b5d3f8671` sind **byte-identisch** (`00`: `2c54d8f3…`, `01`: `56e68aef…`) — und exakt dieselben Bytes liegen in **drei** Papierkorb-Stücken: `04552e60a936`, `1ae290953229`, `3a2fc3d7fe9f`. Dieselben zwei Fotos wurden also **dreimal** hochgeladen. Genau das beschreibt Sven: Der Upload „springt zurück und wird gräulich" (hängengebliebene `recede`-Klasse), er hält es für fehlgeschlagen und lädt erneut hoch. Jeder Upload = ein neues Item (kein Dubletten-Check, siehe b6). Ein vierter Anlauf mit **neuen** Fotos (`beeacdc5…`) ergab Item `28e73ecbef2d`.
2. **Jedes Item bekam seinen eigenen Entwurf.** Bei jedem Stück hat Sven (bzw. der Auto-List-Watcher `sero.js:1894`, der bei `status==="ready" && !draft_id` **automatisch** listet) „Verkaufen" ausgelöst → `app_api.py:2180` legt pro Item einen Entwurf an. Der Guard :2170 sieht nur den Entwurf **desselben** Items — er kennt weder Titel noch Fotos anderer Stücke. Vier Items → vier Entwürfe.
3. **Zwei Entwürfe blieben hängen, statt live zu gehen.** `eb4ad01ac069` und `3d6b5d3f8671` scheiterten beim Publizieren am eBay-USK-18-Fehler (`draft["error_text"]`, im Log `logs/launchd-web.err`: `App-Upload-Fehler für Draft eb4ad01ac069 … InventoryError: 🔞 …`). Sie bleiben ewig auf `ready` im Verkauf-Tab stehen.
4. **Löschen kappte den Anker, ohne die Entwürfe mitzunehmen.** Um 13:06/13:07 und 13:18 hat Sven die Dubletten aus der Sammlung entfernt. `delete_collection_item` ließ die Entwürfe liegen → aus vier verlinkten wurden vier verwaiste Entwürfe.
5. **Der Verkauf-Tab hat die Dubletten dann noch vermehrt.** Verwaiste Entwürfe erscheinen als Zeilen ohne Stück; ein Klick darauf ruft **automatisch `adopt`** auf (`sero.js:2743`). Beweis: die Papierkorb-Stücke `4b5308a20b36`, `e782c1fcbf03`, `e62e5a956553`, `a811076534c6`, `08d99219e0c0` haben `est_value` exakt **1.0** (= Auktions-Startpreis des Entwurfs), `price_updated: null` und Fotos, die **byte-identisch mit `tmp/<draft>/render_00.jpg`** sind — das kann nur `adopt_draft` (`app_api.py:2671-2714`) erzeugt haben. Deshalb tauchen mehrere Items mit **derselben** `draft_id` auf (z. B. dreimal `eb4ad01ac069`): Original-Stück gelöscht → Entwurf verwaist → angeklickt → als neues Stück mit Wert 1,00 € zurück in die Sammlung → wieder gelöscht. Ein Kreislauf, der Sammlung **und** Entwurfsliste aufbläht.

**Kurzfassung für Sven:** Ein und dasselbe GTA III wurde wegen des Upload-Rücksprungs vier Mal als neues Stück angelegt; jedes Stück hat pflichtbewusst seinen eigenen Entwurf gebaut; zwei davon scheiterten an eBays USK-18-Regel; beim Aufräumen verschwanden die Stücke, die Entwürfe blieben — und der Verkauf-Tab hat sie beim Antippen als neue 1-Euro-Stücke zurück in die Sammlung geholt.

---

## Was für die gewünschte Kopplung fehlt (Befund, keine Änderung)

- Die Verbindung ist **einseitig und unbewacht**: nur `item["draft_id"]` (`app_api.py:2185`); der Entwurf kennt sein Stück nicht (kein `draft["item_id"]`), Rückwege laufen über lineare Suche `item_by_draft()` (`app_api.py:436`).
- Vor `create_draft` (`app_api.py:2180`) wird **nur der eigene** `draft_id` geprüft — nicht, ob für dasselbe Produkt (Titel/`card_key`/Foto-Hash) bereits ein Entwurf oder ein Stück existiert. `aehnliches_stueck()` (:400) liegt fertig da, wird aber nur als Anzeige-Hinweis genutzt (:2046).
- Kein Aufräumen in Gegenrichtung: `discard` (:1516) und die 7-Tage-Bereinigung (`bot/main.py:493`) lassen `item["draft_id"]` als tote Referenz stehen.
- `adopt` (:2671) hat keinen Inhalts-Dedupe und übernimmt den Auktionsstartpreis als Sammlungswert.