# Umsetzungsplan — Grau-Hänger und Entwurfs-Flut

## Anker-Stand (neu verankert, read-only geprüft)

Die drei Analysen liegen auf drei verschiedenen Dateiständen. Alle Zeilen unten sind **am aktuellen Stand nachgeprüft**:

| Datei | MD5 | Stand | Zeilen |
|---|---|---|---|
| `/Users/smorty/ebay-bot/web/app_api.py` | `8e48d781cd934d0c83efae1c7f8205ab` | 03.08. 13:31 | 3630 |
| `/Users/smorty/sero-app/web/sero.js` | `c6ba8ebfbd1cdc5e591630eed39a1864` | 03.08. 13:28 | 3950 |
| `/Users/smorty/sero-app/web/index.html` | — | 03.08. 13:28 | `sero.css?v=53` (Z. 23), `sero.js?v=94` (Z. 182) |

**Regel vor jedem Patch:** MD5 neu bilden. Weicht er ab, die Funktion per `grep -n "def <name>"` neu greifen statt der Zeilennummer zu vertrauen. Die Kopplungs-Spezifikation ist rund 54 Zeilen versetzt, die Sheet-Analyse rund 10 — wer nach den Papieren patcht, patcht daneben.

Die Skeptiker-Kritik hat in allen Punkten Vorrang, in denen sie den Analysen widerspricht. Insbesondere: **Pfad 1 (402 → Paywall) scheidet als Erklärung für Svens Grau-Hänger aus** — sein Konto hat Plan `shop`, Premium und Admin-Status, `claim_scans` (`app_api.py:2486-2500`) liefert für ihn nie 402. Der 13:24/13:28-Patch hat den Grau-Hänger also **nicht nachweislich behoben**; die tragenden Ursachen stehen unten in Stufe 1.

---

# 1. Fixes in sicherer Reihenfolge

## Stufe 0 — Blutungen stoppen (heute, klein, kein Datenmodell)

### 0.1 `prepare_and_run` stürzt ab, wenn der Entwurf während des Renderns verschwindet
**Datei:** `/Users/smorty/ebay-bot/web/app_api.py`, Funktion `prepare_and_run` in `_list_collection_item`, **Zeile 2235-2262**

```python
async def prepare_and_run():
    draft = store.get_draft(draft_id)
    if not draft:                       # Stück wurde inzwischen gelöscht
        log.info("Entwurf %s vor dem Rendern verschwunden — Lauf beendet", draft_id)
        return
    ...                                  # Render-Schleife (Sekunden bis Minuten)
    draft = store.get_draft(draft_id)   # frisch lesen statt am alten Objekt weiterschreiben
    if not draft:
        return
    draft["original_photos"] = originals   # heute Zeile 2253
```

**Begründung:** Zeile 2253 schreibt in ein `dict`, das seit dem 13:31-Patch mitten im Lauf gelöscht werden kann — das ist der einzige echte Absturz im System.

**Was das kaputtmachen kann:** Nichts an der Funktion; einzige Nebenwirkung ist, dass `item["draft_id"]` als toter Zeiger stehenbleibt, wenn der Lauf abbricht — das behebt 0.2 bzw. 3.3.

**Prüfung:** Ein Test-Stück listen und innerhalb von zwei Sekunden löschen. Danach `grep -i "TypeError\|Traceback" /Users/smorty/ebay-bot/logs/launchd-web.err | tail` — dort darf kein neuer Eintrag stehen. Vorher denselben Handgriff auf dem Ist-Stand ausführen und den Traceback dokumentieren, sonst ist der Fix nicht belegt.

---

### 0.2 Löschen eines Stücks reißt laufende Pipelines ab und prüft den Besitz nicht
**Datei:** `app_api.py`, Funktion `delete_collection_item`, **Zeile 2122-2131**

```python
did = item.get("draft_id")
if did:
    entwurf = own_draft(did, account)        # statt store.get_draft — Besitzprüfung
    status = (entwurf or {}).get("status")
    if entwurf and status in ("new", "ready", "error", "uncertain"):
        cleanup_photos(entwurf)              # tmp/<draft_id> mit entsorgen (Import Z. 78)
        store.delete_draft(did)
        log.info("Entwurf %s mit dem Stück %s entfernt", did, item_id)
    elif entwurf and status in ("downloading", "analyzing"):
        # Laufende Pipeline nicht abreißen — nur markieren, aufgeräumt wird später
        entwurf["item_deleted"] = time.time()
        store.update_draft(did, entwurf)
```

**Begründung:** Der 13:31-Patch löscht heute auch Entwürfe im Status `downloading`/`analyzing` und benutzt `store.get_draft` ohne Konto-Prüfung — beides schafft genau die Absturzlage aus 0.1.

**Was das kaputtmachen kann:** Ein Entwurf in `downloading` bleibt jetzt bestehen, statt zu verschwinden — er taucht aber nicht im Verkauf-Tab auf, weil dort nur `published`/`ended`/`ready`/`dry_run_done`/`error`/`uncertain` **mit** `listing` angezeigt werden (`app_api.py:3004-3011`). Zweitens: `own_draft` bindet an `uid_for(account)` — hat der Entwurf eine andere `chat_id` (Telegram-Verknüpfung geändert), wird er gar nicht mehr gelöscht. Das ist die sichere Richtung.

**Prüfung:** (a) Stück mit `ready`-Entwurf löschen → Entwurf verschwindet aus dem Verkauf-Tab, `ls /Users/smorty/ebay-bot/tmp/<draft_id>` ist leer. (b) Stück direkt nach dem Listen löschen → Entwurf existiert weiter mit `item_deleted`, kein Traceback im Log. (c) `sqlite3 -readonly` prüfen, dass die Zahl der `published`/`ended`-Entwürfe unverändert bleibt.

**Hinweis:** `_trash/<item_id>/item.json` enthält `draft_id` — beim Wiederherstellen von Hand ist die alte Zuordnung damit nachvollziehbar.

---

### 0.3 Datenverlust: der 402-Pfad löscht die geparkten Fotos
**Datei:** `/Users/smorty/sero-app/web/sero.js`, Funktion `handleScanError`, **Zeile 309-313**

```js
function handleScanError(e, fallback) {
  if (e && e.status === 402) {
    state.stageKeep = true;      // Fotos bleiben auf dem Server liegen
    closeSheet(); openPaywall(); return true;
  }
```

**Begründung:** `closeSheet` (`sero.js:3796-3800`) räumt die Ablage per `/collection/stage/clear`, wenn `stageKeep` falsch ist — bei 402 sieht der Nutzer die Paywall und seine Fotos sind währenddessen weg.

**Was das kaputtmachen kann:** Die Ablage bleibt liegen und `pruefeAblage`/`loadCollection` (`sero.js:1943`, `1952`) legt die Fotos beim nächsten Öffnen wieder aufs Sheet — gewollt, aber es muss sichtbar sein, dass es alte Fotos sind. Sonst entsteht daraus wieder ein Stück zu viel.

**Prüfung:** Betrifft Sven nicht (sein Konto kann kein 402 auslösen), sondern zahlende Neukunden am Limit. Testen mit einem Zweitkonto ohne Kontingent: Upload → Paywall → Sheet neu öffnen → die Fotos liegen noch da.

---

## Stufe 1 — Beschwerde 1 wirklich schließen (Frontend, kein Backend-Risiko)

### 1.1 Das Detail-Overlay hat exakt die Race, die beim Sheet gerade repariert wurde
**Datei:** `sero.js`, `closeDetail` **Zeile 3029-3037**, `openItemDetail` **Zeile 3040-3042**, `openDraftDetail` **Zeile 3057-3058**

```js
function closeDetail() {
  const d = $("detail");
  ...
  d.classList.add("closing");
  setTimeout(() => {
    if (!d.classList.contains("closing")) return;   // inzwischen neu geöffnet
    d.hidden = true; d.classList.remove("closing");
  }, 260);
```
und in **beiden** Öffnern (3042 und 3058), jeweils vor `hidden = false`:
```js
$("detail").classList.remove("closing");
$("detail").hidden = false;
```

**Begründung:** `.detail.closing` (`sero.css:695`) friert das Overlay per `forwards` außerhalb des Bildes ein; `#detail.hidden` ist dann `false`, und `pruefeSchleier` (`sero.js:3821`) wertet das als „etwas ist offen" und entfernt `recede` **nie mehr** — das ist der Dauer-Hänger, den der 13:28-Patch nicht abdeckt.

**Erreichbar über:** Klick auf eine Zeile im Verkauf-Tab (`sero.js:2739-2747`, öffnet Detail direkt nach einem `closeDetail`) und `showScanResult` → `#resOpen` (`sero.js:359`, 300-ms-Timer).

**Was das kaputtmachen kann:** Wenn `closeDetail` und `openItemDetail` auf dasselbe Stück laufen, bleibt das Overlay jetzt sichtbar statt zu verschwinden — das ist das gewünschte Verhalten, aber der Poll-Timer aus `state.detail` muss weiterhin über `clearTimeout(state.detail?.poll)` sterben (steht bereits in 3031).

**Prüfung:** Verkauf-Tab, Zeile antippen, Detail schließen und innerhalb einer Sekunde erneut eine Zeile antippen — das Overlay muss sichtbar sein. Danach schließen: die App darf nicht verkleinert/dunkel bleiben.

---

### 1.2 `openSheet` räumt die Inline-Stile der Griff-Geste nicht weg
**Datei:** `sero.js`, `openSheet` **Zeile 3760-3792** (Klassen-Reset heute 3771), Griff-Geste **Zeile 3827-3849**

```js
// in openSheet, direkt nach dem closing-Reset (3771):
_sh.classList.remove("closing"); _bd.classList.remove("closing");
_sh.style.transform = ""; _sh.style.transition = "";
state.sheetDrag = null;          // hängengebliebene Geste ungültig machen
```
und in der Geste `sy` durch `state.sheetDrag` ersetzen (heute Closure-Variable in Zeile 3831), damit `openSheet` sie zurücksetzen kann.

**Begründung:** Bleibt nach einer abgebrochenen Zieh-Geste ein `transform: translateY(...)` plus `transition:none` am Sheet stehen (iOS liefert bei Rand-Gesten nicht immer `pointerup`), ist das Sheet unsichtbar, `hidden === false`, und `recede` bleibt hängen; `closeSheet` setzt `transform` nur zurück, wenn das Sheet nicht schon versteckt ist (`sero.js:3804-3805`).

**Was das kaputtmachen kann:** `transition = ""` fällt auf die CSS-Transition zurück — die Schließ-Animation (`sero.css:1114`) läuft danach wieder wie vorgesehen. Risiko gering.

**Prüfung:** Scan-prüfen-Sheet am Griff nach unten ziehen, den Finger über den unteren Bildschirmrand hinausziehen, loslassen, dann ein beliebiges anderes Sheet öffnen — es muss sichtbar sein.

---

### 1.3 `.party`-Overlays stapeln sich und blockieren den Wächter dauerhaft
**Datei:** `sero.js`, fünf Erzeuger: **247** (`checkMilestone`), **326** (`showScanResult`), **369** (`showScanFailed`), **407** (`celebrate`), **1331** (`showTour`)

Jeweils direkt vor dem `document.body.appendChild(el)`:
```js
document.querySelectorAll(".party").forEach((n) => n.remove());
```

**Begründung:** Ein `.party` verschwindet nur per Knopf oder Klick exakt auf den Hintergrund; bleibt eines liegen, ist `document.querySelector(".party")` in `pruefeSchleier` (`sero.js:3821`) dauerhaft wahr und `recede` wird nie entfernt.

**Was das kaputtmachen kann:** Zwei bewusst gleichzeitige Overlays gibt es nicht — aber `showScanResult` kann aus `loadCollection` (`sero.js:1985-1991`, Poll alle 2200 ms) mehrfach feuern; danach ist immer nur die jüngste Karte zu sehen. Das ist die richtige Richtung, kostet aber eine Erfolgskarte, wenn zwei Stücke gleichzeitig fertig werden. Falls Sven beide sehen soll: die Karte in eine Warteschlange legen statt sie zu ersetzen — nicht in dieser Stufe.

**Prüfung:** Zwei Stücke gleichzeitig scannen lassen, danach eine Karte wegtippen: `document.querySelectorAll(".party").length` muss 0 sein, die App darf nicht dunkel bleiben.

---

### 1.4 `pruefeSchleier` schärfen — Sichtbarkeit statt `hidden`
**Datei:** `sero.js`, **Zeile 3820-3824**

```js
function sichtbar(el) {
  if (!el || el.hidden) return false;
  const r = el.getBoundingClientRect();
  return r.height > 0 && r.top < window.innerHeight - 8;
}
function pruefeSchleier() {
  if (Date.now() < (state.schleierAb || 0)) return;   // Animationsfenster aussparen
  const offen = sichtbar($("sheet")) || sichtbar($("detail")) || !!document.querySelector(".party");
  if (!offen) $("viewApp").classList.remove("recede");
}
```
plus in `openSheet`/`openItemDetail`/`openDraftDetail`: `state.schleierAb = Date.now() + 700;`

**Begründung:** Der heutige Wächter prüft nur `hidden` und ist damit stumpf — ein aus dem Bild geschobenes Sheet oder Detail hält `recede` für immer aktiv.

**Was das kaputtmachen kann:** Ohne die 700-ms-Sperre würde der Wächter mitten in der Öffnungs-Animation zuschlagen und der Schleier flackert. Die Sperre ist deshalb Pflicht, nicht Kosmetik.

**Prüfung:** Zehnmal hintereinander ein Sheet öffnen und schließen — kein sichtbares Aufblitzen der ungedimmten App, und nach dem letzten Schließen ist `recede` weg (Konsole: `$("viewApp").className`).

---

### 1.5 Cache-Buster erhöhen
**Datei:** `/Users/smorty/sero-app/web/index.html`, **Zeile 23** (`sero.css?v=53` → `v=54`), **Zeile 182** (`sero.js?v=94` → `v=95`)

**Begründung:** Ohne neue Nummer liefert Svens installierte PWA weiter die alte Datei aus, und die Fixes wirken bei ihm gar nicht.

**Was das kaputtmachen kann:** Nichts; einmaliger Neu-Download. Es gibt keinen Service Worker im Projekt, nur den HTTP-Cache.

**Prüfung:** In der App das Profil öffnen, in der Konsole `document.querySelector('script[src*=sero.js]').src` — die neue Nummer muss dastehen.

---

### 1.6 „Springt zurück" — der erzwungene Tab-Wechsel (Produktentscheidung, zuletzt)
**Datei:** `sero.js`, **Zeile 2473**, **2519**, **2549** — jeweils `closeSheet(); switchTab("tabScan"); loadCollection();`

```js
// beim Öffnen des Stage-Sheets merken, wo der Upload begann:
state.uploadTab = state.tab;
...
closeSheet();
if (state.uploadTab === "tabScan") switchTab("tabScan");
loadCollection();
```

**Begründung:** Wer den Kamera-Knopf im Suchfeld der Sammlung tippt (`sero.js:74-78`), landet heute zwangsweise im Scanner-Tab — das ist der wörtliche Teil von „springt zurück", unabhängig vom Grau-Hänger.

**Was das kaputtmachen kann:** `showScanResult` wird aus `loadCollection` über `state.watchNew` getriggert und ist auf den Scanner-Tab hin gebaut; die Erfolgskarte muss auch über der Sammlung erscheinen. Vor der Umsetzung prüfen, ob die Karte tabunabhängig an `document.body` hängt — sie tut es (`sero.js:326`, `appendChild` an `body`), also unkritisch.

**Prüfung:** Aus der Sammlung heraus hochladen → nach dem Analysieren steht die Sammlung noch da, die Erfolgskarte liegt darüber. Aus dem Scanner heraus hochladen → Verhalten unverändert.

---

## Stufe 2 — Diagnose statt Vermutung (nur lesend, ändert nichts)

### 2.1 Grundlinien-Zähler
**Neue Datei:** `/Users/smorty/ebay-bot/tools/kopplung_check.py` — öffnet die DB ausschließlich über `sqlite3.connect("file:/Users/smorty/ebay-bot/data.db?mode=ro", uri=True)` und gibt aus:

- lebende Entwürfe (Status ≠ `ended`) ohne Stück
- Stücke mit `draft_id`, zu der kein Entwurf existiert (Zeiger ins Leere)
- `draft_id`, die von mehr als einem Stück gehalten wird
- je Klasse: ID, Status, Titel, Alter

**Begründung:** Ohne Grundlinie ist später nicht beweisbar, dass ein Fix gewirkt hat — heute: 29 Entwürfe, 11 Stücke, 6 verlinkt.

**Was das kaputtmachen kann:** Nichts, solange `mode=ro` steht und kein Endpunkt daraus wird. Kein Cron in dieser Stufe.

**Prüfung:** Zweimal hintereinander laufen lassen, identische Ausgabe. `ls -l data.db` — mtime unverändert.

---

## Stufe 3 — Kopplung, additiv und rückwärtskompatibel

### 3.1 Zuerst die Schreib-Disziplin, dann neue Felder
**Datei:** `app_api.py`, `_list_collection_item`, **Zeile 2185-2186**

Heute wird der komplette Item-Blob, gelesen bei Zeile 2160, per `col_save` zurückgeschrieben. Läuft parallel eine Analyse oder ein Preis-Update, wird deren Ergebnis überschrieben. Das schonende Muster steht bereits im Haus: `col_save_analyse` (`app_api.py:736-753`) liest frisch und gibt `None` zurück, wenn das Stück verschwunden ist.

```python
def col_save_feld(item_id: str, account: dict, **felder) -> dict | None:
    """Nur einzelne Felder setzen — liest frisch, kein Blob-Überschreiben."""
    frisch = col_get(item_id, account)
    if not frisch:
        return None
    frisch.update(felder)
    col_save(item_id, account["id"], frisch)
    return frisch
```

**Begründung:** Jede zusätzliche Schreiboperation an dieser Stelle vergrößert das Überschreib-Fenster — die Historie-Liste aus Stufe 3.2 kommt erst danach.

**Was das kaputtmachen kann:** `col_get` prüft das Konto; gibt es `None` zurück, muss der Listen-Vorgang sauber abbrechen statt weiterzulaufen. Das ist neu und muss im Aufrufer behandelt werden.

**Prüfung:** Ein Stück listen, während eine Preis-Aktualisierung läuft — `est_value` und `price_updated` dürfen danach nicht auf alten Werten stehen.

---

### 3.2 Gegenrichtung nur zusätzlich schreiben
**Datei:** `app_api.py`, direkt nach `store.create_draft` (**Zeile 2180**)

```python
d0 = store.get_draft(draft_id)
d0["item_id"] = item_id
d0["account_id"] = account["id"]
store.update_draft(draft_id, d0)
```

**Begründung:** Erst mit `draft["item_id"]` ist ein verwaister Entwurf überhaupt erkennbar, ohne alle Stücke linear zu durchsuchen (`item_by_draft`, `app_api.py:436`).

**Was das kaputtmachen kann:** `Store.update_draft` (`bot/drafts.py:272-279`) setzt den Status auf `"new"`, wenn das übergebene Dict kein `status` enthält — deshalb **immer** das volle, frisch gelesene Dict übergeben, nie ein Teil-Dict. Ein Entwurf im Status `new` verschwindet aus dem Verkauf-Tab (`app_api.py:3007-3011`) und wird zum unsichtbaren Zombie.

**Prüfung:** Nach dem Listen `sqlite3 -readonly` → `status` des neuen Entwurfs ist `downloading`, nicht `new`. Das Feld `item_id` steht im JSON. Kein Leseweg darf sich auf `item_id` verlassen, solange die Altbestände es nicht haben.

---

### 3.3 Zeiger ins Leere aufräumen, wo sie entstehen
**Datei:** `app_api.py` — `discard` **Zeile 1516-1523**, `end` **Zeile 1556-1569**, `mark_item_sold` **Zeile 2539**

```python
def loese_kopplung(draft_id: str, account: dict, *, historie: bool = True) -> None:
    it = item_by_draft(account["id"], draft_id)
    if not it or it.get("draft_id") != draft_id:
        return
    felder = {"draft_id": None}
    if historie:
        felder["draft_ids"] = [*(it.get("draft_ids") or []), draft_id]
    col_save_feld(it["id"], account, **felder)
```
Aufruf in `discard` nach `store.delete_draft(draft_id)` (Zeile 1521) und in `end` nach dem Statuswechsel auf `ended` (Zeile 1564).

**Begründung:** Es gibt im ganzen Backend heute **keine einzige Stelle, die `item["draft_id"]` wieder entfernt** — deshalb heißt der Knopf am Stück nach einem verworfenen Entwurf für immer „Listing-Entwurf öffnen" (`sero.js:2570`, `2584`) und das Stück lässt sich nie wieder listen.

**Was das kaputtmachen kann:** Ein Stück, dessen `ended`-Listing verkauft wurde, verliert die Verbindung zum Verkaufsbeleg, wenn `draft_ids` nicht ausgewertet wird — deshalb die Historie-Liste mitschreiben und `mark_item_sold` (`app_api.py:2539`) zusätzlich über `draft_ids` suchen lassen, nicht nur über `draft_id`.

**Prüfung:** (a) Entwurf verwerfen → das Stück zeigt wieder „Auf eBay listen" und lässt sich listen. (b) Listing beenden, neu einstellen, im Sandkasten als verkauft markieren → `sold_ts` landet am Stück.

---

### 3.4 Den Wiederverwendungs-Guard ehrlich machen
**Datei:** `app_api.py`, **Zeile 2170-2174**

```python
if item.get("draft_id"):
    existing = own_draft(item["draft_id"], account)
    if not existing:
        item = col_save_feld(item_id, account, draft_id=None) or item   # toter Zeiger
    elif existing.get("status") == "published":
        return {"ok": True, "draft_id": item["draft_id"], "action": "published"}
    elif existing.get("status") == "error":
        loesche_fehler(existing)                       # app_api.py:423
        _spawn(app_run_pipeline(account, item["draft_id"]))
        return {"ok": True, "draft_id": item["draft_id"], "action": "restarted"}
    elif existing.get("status") != "ended":
        return {"ok": True, "draft_id": item["draft_id"], "action": "reused"}
```

**Begründung:** Heute ist ein toter Zeiger nicht von einem lebenden zu unterscheiden, `published` kann durch einen zweiten Tipp ein Parallel-Listing erzeugen, und ein Entwurf im Status `error` bleibt für immer liegen („ich tippe listen und es passiert nichts").

**Was das kaputtmachen kann — der kritische Punkt:** `listNow` (`sero.js:2373-2381`) wertet **jede** 2xx-Antwort als Erfolg und toastet fest „Entwurf erstellt — liegt im Verkauf-Tab". Das Feld `action` muss deshalb **zuerst** im Frontend ausgewertet werden, sonst meldet die App weiter falsche Erfolge:

```js
const r = await post(...);
toast({ created: "Entwurf erstellt — liegt im Verkauf-Tab",
        reused: "Entwurf liegt schon im Verkauf-Tab",
        updated: "Entwurf aktualisiert",
        restarted: "Neuer Versuch läuft",
        published: "Steht schon bei eBay — dort bearbeiten" }[r.action] || "Entwurf erstellt", "arrowup");
```

**Keine Rückfrage-Antworten (409/„Bestätigung nötig") in diesen Endpunkt einbauen**, solange der Auto-List-Wächter (`sero.js:1894`) läuft: Er verbucht jede 2xx als Erfolg und löscht die ID aus `state.sellWatch` — Sven bekäme die Rückfrage nie zu sehen und das Stück würde nie gelistet.

**Prüfung:** Zweimal „listen" am selben Stück in jedem Status durchspielen (`ready`, `error`, `published`, `ended`) und nach jedem Durchgang `SELECT count(*) FROM drafts` vergleichen — nur bei `ended` darf die Zahl steigen.

---

### 3.5 `stale_draft_ids` löscht mehr, als der Kommentar behauptet
**Datei:** `/Users/smorty/ebay-bot/bot/drafts.py`, **Zeile 293-306**

```sql
SELECT id FROM drafts WHERE status NOT IN ('published') AND updated_at < ?
```
Das trifft **auch `ended`, `ready`, `error`, `uncertain`** — nach 7 Tagen hart gelöscht durch `cleanup_stale_drafts` (`bot/main.py:489-508`, Start bei `:1821`, danach täglich). Zielzustand:

```sql
SELECT id FROM drafts
 WHERE status NOT IN ('published', 'ended')
   AND json_extract(data, '$.item_id') IS NULL
   AND updated_at < ?
```

**Begründung:** Solange der Bot alles Nicht-`published` nach 7 Tagen löscht, ist jede Zusage „archivieren statt löschen, 30 Tage Frist" hinfällig, und Svens sechs `ready`-Entwürfe sterben mitsamt ihrer Kopplung.

**Was das kaputtmachen kann:** Das berührt den **laufenden Bot-Betrieb** — deshalb als letzter Schritt dieser Stufe, mit vorherigem Trockenlauf: die aktuelle Abfrage read-only ausführen und die Trefferliste ansehen (heute: 0 Treffer, weil der Sales-Sync die `ended`-Zeilen anfasst). Nach der Änderung wachsen alte Entwürfe unbegrenzt — dafür braucht es eine spätere, ausdrücklich bestätigte Aufräumung, nicht einen stillen Cron.

**Prüfung:** `SELECT status, count(*) FROM drafts GROUP BY status` vor und nach dem nächsten Bot-Neustart — keine Zeile darf verschwinden.

---

## Stufe 4 — Dubletten: erkennen ja, zusammenführen nein

### 4.1 Die vorhandene Erkennung an der richtigen Stelle abfragen
**Datei:** `app_api.py` `aehnliches_stueck` **Zeile 400-434** (fertig, im Einsatz bei `:2046-2050` als Feld `dublette`), Frontend-Anzeige `sero.js:3142-3145`, Auto-Listen `sero.js:1894`

```js
// sero.js:1894 — vor dem automatischen Listen prüfen
if (it && it.status === "ready" && !it.draft_id) {
  state.sellWatch.delete(id);
  const d = await getJSON(`/api/app/collection/item/${id}`);
  if (d.dublette && d.dublette.draft_id) {
    toast("Dieses Stück liegt schon im Verkauf — Entwurf öffnen?", "info",
          { label: "Öffnen", fn: () => openItemDetail(d.dublette.id, "sell") });
  } else {
    listNow(id);
  }
}
```

**Begründung:** Genau hier entsteht die Entwurfs-Flut — der Wächter listet jedes fertige Stück ungefragt, und der Guard bei `:2170` kennt nur den eigenen `draft_id`, nicht das Zwillings-Stück.

**Keine zweite Vergleichslogik einführen.** Die Spezifikation schlägt `card_key`-Gleichheit plus „Ähnlichkeit ≥ 0,9" vor — `card_key` ist bei Videospielen fast immer `solo:…` und greift im GTA-Fall nie, und eine Ähnlichkeitsfunktion existiert im Code nicht. `aehnliches_stueck` arbeitet bereits mit den ersten vier Wörtern plus der Menge aller Zahlen (`app_api.py:409-415`), ignoriert `solo:`-Schlüssel (`:419-420`) und überspringt verkaufte Stücke (`:427`) — das ist die passende Grundlage.

**Was das kaputtmachen kann:** Bei einem echten zweiten Exemplar (Händler-Normalfall) blockiert der Hinweis das Auto-Listen. Deshalb: Der Toast muss eine zweite Aktion „Trotzdem listen" haben, sonst ist Sven ausgesperrt. Zweitens setzt `rescue_stuck_scans` (`app_api.py:2607-2636`) hängende Stücke alle 5 Minuten zurück auf `analyzing` — der Hinweis darf deshalb nicht an der Analyse hängen, sondern nur am Auto-List-Wächter, sonst feuert er bei jedem Rettungslauf erneut.

**Prüfung:** Dasselbe Spiel zweimal im Verkaufsmodus scannen → beim zweiten Mal entsteht **kein** zweiter Entwurf, sondern der Hinweis. „Trotzdem listen" tippen → Entwurf entsteht.

---

### 4.2 Den Verstärker abstellen: automatisches `adopt` im Verkauf-Tab
**Datei:** `sero.js` **Zeile 2738-2750**, Backend `adopt_draft` `app_api.py:2672-2715`

```js
b.onclick = async () => {
  if (b.dataset.item) return openItemDetail(b.dataset.item, "sell");
  openDraftDetail(b.dataset.draft);       // erst zeigen, nicht sofort übernehmen
};
```
und die Übernahme als ausdrücklicher Knopf im Entwurfs-Detail („Als Stück in die Sammlung übernehmen"). Zusätzlich im Backend den Auktions-Startpreis nicht mehr als Sammlungswert setzen (`app_api.py:2700-2703` und `"est_value": value` bei `:2711`):

```python
if (d.get("format") or "").upper() == "AUCTION":
    value = None            # 1,00 € Startpreis ist kein Marktwert
```

**Begründung:** Jeder Klick auf eine Zeile ohne Stück legt heute ungefragt ein neues Stück mit dem Auktions-Startpreis 1,00 € an — das ist der Kreislauf, der Sammlung und Entwurfsliste zugleich aufgebläht hat (Beleg: die Papierkorb-Stücke mit `est_value: 1.0` und Fotos, die byte-identisch mit `tmp/<draft>/render_00.jpg` sind).

**Was das kaputtmachen kann:** „Bestehende eBay-Listings übernehmen" (`import_listings`, `app_api.py:2719`) nutzt denselben Weg — dort ist die automatische Übernahme gewollt und muss unangetastet bleiben. Zweitens: `/sales` baut `item_by_draft` als **Dict** (`app_api.py:2996`) — zwei Stücke mit derselben `draft_id` lassen eines lautlos aus der Anzeige fallen. Vor jeder Massen-Übernahme die Eindeutigkeit prüfen (Zähler aus 2.1).

**Prüfung:** Verkauf-Tab, Zeile ohne Stück antippen → es entsteht **kein** neues Stück (`SELECT count(*) FROM collection_items` unverändert), sondern die Entwurfs-Ansicht öffnet sich.

---

# 2. Aufräum-Routine für die 7 verwaisten Entwürfe

**Grundsatz: `published` und `ended` werden nicht angefasst — weder gelesen-geändert noch verknüpft noch gelöscht.** Dort hängt echtes Geld.

**Datei:** `/Users/smorty/ebay-bot/tools/entwuerfe_aufraeumen.py`, zwei getrennte Läufe.

### Lauf 1 — Trockenlauf (read-only, DB per `mode=ro`)
Für jeden Entwurf mit `chat_id = uid_for(account)`:

| Klasse | Bedingung | Vorschlag |
|---|---|---|
| **A — gesund** | ein Stück zeigt darauf | nichts, nur `item_id` nachtragen (Lauf 2) |
| **X — tabu** | `status` in (`published`, `ended`, `dry_run_done`) | **nicht anfassen, nicht verknüpfen, nicht löschen** — nur zählen |
| **C1** | Status in (`new`,`downloading`,`analyzing`,`uncertain`,`error`,`ready`), kein Stück zeigt darauf, **genau ein** Stück passt per `aehnliches_stueck` und hat selbst keinen lebenden Entwurf | Verknüpfung vorschlagen |
| **C2** | mehrere unfertige Entwürfe passen auf dasselbe Stück (der GTA-III-Fall) | jüngsten (`updated_at` max) verknüpfen, die übrigen archivieren |
| **C3** | kein passendes Stück | archivieren |

Ausgabe pro Entwurf: ID, Status, Titel, Alter, Fotoanzahl, Klasse, Vorschlag, Ziel-Stück. Erwartung beim heutigen Stand: 4× Grand Theft Auto III → einer verknüpft, drei archiviert; die übrigen drei verwaisten je nach Titel verknüpft oder archiviert; `published`/`ended` unverändert.

### Lauf 2 — Ausführung, nur nach Svens ausdrücklichem „ja" zur Liste aus Lauf 1
- **Archivieren heißt Soft-Delete:** `draft["archived_at"] = time.time()`, Ausblendung im Verkauf-Tab (Filter `app_api.py:3005-3011`), Fotos bleiben liegen. Kein `delete_draft`.
- **Voraussetzung:** Fix 3.5 muss stehen, sonst löscht `stale_draft_ids` (`bot/drafts.py:300`) die archivierten Entwürfe nach 7 Tagen hart und die Zusage „umkehrbar" ist wertlos.
- **Eindeutigkeits-Riegel vor jedem Schreiben:** keine `draft_id` darf danach von zwei Stücken gehalten werden (sonst verschwindet eines lautlos aus `/sales`, `app_api.py:2996`).
- **Protokoll:** jede Änderung mit Vorher-Wert in eine Datei schreiben, damit sie von Hand rückgängig zu machen ist.

**Prüfung nach dem Lauf:** Der Zähler aus 2.1 meldet 0 verwaiste lebende Entwürfe; `SELECT status, count(*) FROM drafts WHERE status IN ('published','ended') GROUP BY status` ist bit-identisch zu vorher; die Kopfzahl „Sammlungswert" in der App ist unverändert.

---

# 3. Was bewusst nicht gemacht wird

1. **Verwaiste `published`-Entwürfe werden nicht an Stücke geknüpft.** `portfolio_series` (`app_api.py:628-634`) und `portfolio_ohlc` (`:653-663`) schließen jedes Stück aus der Wertrechnung aus, dessen `draft_id` zu einem `published`-Entwurf gehört. Eine solche Verknüpfung ließe Svens Sammlungswert fallen, ohne dass er etwas verkauft hat.

2. **Kein automatisches Zusammenführen von Dubletten.** Das zweite Exemplar ist beim Händler der Normalfall — ein Merge vernichtet `purchase_price`, `condition`, `graded` und die Fotos des zweiten Stücks und ist nicht umkehrbar. Es bleibt beim Hinweis mit Knopf. Nach dem 02.08. gilt: vorschlagen statt ausführen.

3. **Kein Löschen-Verbot für Stücke mit `published`-Entwurf im Backend, solange das Frontend Fehler verschluckt.** `removeItemWithUndo` (`sero.js:114-124`) entfernt das Stück optimistisch, schickt den DELETE erst nach 6 Sekunden und fängt jeden Fehler mit `.catch(() => {})` ab — das Stück würde kommentarlos wieder auftauchen. Wenn dieses Verbot kommt, dann nur zusammen mit einer sichtbaren Fehlermeldung.

4. **Keine Bestätigungs-Rückfragen im Listen-Endpunkt.** Beide Aufrufer (`listNow` `sero.js:2373`, Auto-List-Wächter `:1894`) verbuchen jede 2xx als Erfolg — die Rückfrage käme nie bei Sven an und das Stück würde nie gelistet. Erkennung gehört ins Frontend (4.1).

5. **`sync_sales_status`, `publish-drafts` (Bulk) und `import_listings` bleiben unangetastet**, bis Stufe 0-2 stehen. Dort hängt Geld, und die Wege sind heute funktionsfähig.

6. **Die zwei parallelen Live-Listings für „Vice City"** (`147480067874` über Entwurf `74925bead28e` ohne Stück, `147480090061` über `9f8f51762eca`) werden **von Hand** bei eBay geklärt, bevor irgendeine Automatik daran rührt. Das ist kein Code-Problem.

7. **Kein Aufräum-Cron.** Jede Bereinigung läuft als Trockenlauf mit anschließender ausdrücklicher Bestätigung. Der Wächter aus 2.1 meldet nur, er ändert nichts.

8. **Kein zweiter Dubletten-Vergleicher.** `aehnliches_stueck` (`app_api.py:400-434`) ist vorhanden und getestet im Einsatz; eine konkurrierende Logik über `card_key`-Gleichheit würde bei `solo:`-Schlüsseln — also genau im GTA-Fall — nie greifen.

9. **Der Telegram-Bot wird in dieser Runde nicht umgebaut.** Bot und App teilen sich `chat_id 5694742134` (`uid_for`, `app_api.py:223-226`), und `latest_draft_for_chat` (`bot/drafts.py:286-291`, benutzt in `bot/main.py:1187`) kann einen App-Entwurf überschreiben; `bot/main.py:1388` und `:1046` löschen Entwürfe ohne Rücksicht auf Stücke. Jede Kopplungsregel, die nur in `app_api.py` steht, ist darüber umgehbar — das gehört als eigener, vorbereiteter Schritt behandelt, nicht nebenbei.

---

# 4. Reihenfolge in einer Zeile

0.1 → 0.2 → 0.3 → 1.1 → 1.2 → 1.3 → 1.4 → 1.5 → (1.6 nach Svens Entscheidung) → 2.1 → 3.1 → 3.2 → 3.3 → 3.4 → 3.5 → 4.1 → 4.2 → Aufräum-Trockenlauf → Aufräum-Ausführung.

Nach Stufe 1 tippt Sven noch einmal denselben Ablauf durch, der ihn heute Mittag gestört hat. Bleibt die App dabei nicht mehr verkleinert und dunkel stehen, ist Beschwerde 1 belegt geschlossen — vorher nicht.