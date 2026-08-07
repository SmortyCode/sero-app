# Skeptiker-Gegenprüfung — alles am Code belegt (Stand app_api.py 13:31, sero.js/css/index.html 13:28)

**Vorbemerkung zu den Zeilennummern.** Die drei Analysen liegen auf drei verschiedenen Dateiständen:
- Die **Kopplungs-Spezifikation** ist ~54 Zeilen versetzt (sie nennt den Wiederverwendungs-Guard bei 2116-2119, `discard` bei 1483, `delete_collection_item` bei 2058 — tatsächlich: `app_api.py:2170-2174`, `:1516`, `:2102`). Sie beschreibt den Stand **vor** dem 13:31-Patch.
- Die **Sheet-Analyse** nennt MD5 `caf99935…` für sero.js; aktuell ist `c6ba8ebfbd1cdc5e591630eed39a1864`. Die Datei wurde nach ihrer Analyse nochmals geändert; alle Angaben ab Zeile 3750 sind um +10 versetzt.
- Nur der **Entwurfs-Lebenszyklus** ist backendseitig zeilengenau.

Wer nach diesen Papieren patcht, patcht an falschen Stellen. **Vor jeder Umsetzung neu verankern.**

---

## (1) Behauptungen, die NICHT stimmen

### F1 — KRITISCH: „Pfad 1 (402 → Paywall) ist der beste Erklärer für Svens Grau-Hänger." — Für Sven **unmöglich**.
`claim_scans` (`app_api.py:2486-2500`) startet mit `if scan_freigeschaltet(account): return None`. `scan_freigeschaltet` (`:2483-2486`) ist wahr bei `is_admin` **oder** `plan in ("starter","reseller","shop")` **oder** `is_premium`. Aus der DB (read-only):

```
accounts: 3 | sven.manuel@aol.com | telegram_id 5694742134 | plan "shop"
kv:       premium_3 = {"on": true}
.env:     ALLOWED_USER_ID=5694742134   →  is_admin(account) == True  (app_api.py:227-228)
```

Sven erfüllt **alle drei** Bedingungen. `/collection/items-from-stage` (`:1924-1932`) kann für ihn nie 402 liefern; `handleScanError` (`sero.js:309-313`) feuert nie; `openPaywall` wird nie aus dem Upload-Ablauf gerufen. Der als „HÖCHSTE Wahrscheinlichkeit, 100 % Trefferquote" verkaufte Pfad ist bei Sven **tot**.

**Konsequenz:** Die Aussage „durch den 13:24/13:28-Patch behoben" ist für Svens Beschwerde 1 **unbelegt**. Wer den Patch als Lösung abhakt, hat die Ursache nicht angefasst. Es bleiben Pfad 2 (nur Stapel-Scan), Pfad 3 und zwei bisher **nicht** genannte Pfade (siehe Ü1/Ü2 unten).

### F2 — KRITISCH: „`published` und `ended` werden NIE gelöscht" (Spez §2 „archiviert = ended, bleibt für immer erhalten", §6 „Eiserne Regel").
Falsch. `Store.stale_draft_ids` (`bot/drafts.py:297-300`):
```sql
SELECT id FROM drafts WHERE status NOT IN ('published') AND updated_at < ?
```
`NOT IN ('published')` — also **auch `ended`, `ready`, `error`, `uncertain`**. `cleanup_stale_drafts` (`bot/main.py:489-508`) löscht diese nach **7 Tagen** hart (`store.delete_draft`, `:497`), läuft beim Bot-Start (`bot/main.py:1821`) und danach täglich (`:1823`). Der Kommentar bei `drafts.py:296` („Unfertige Drafts") beschreibt nicht, was der Code tut.

Aktuell noch harmlos (`SELECT count(*) … updated_at < now-7d` ⇒ **0**), weil der Sales-Sync die `ended`-Zeilen anfasst. Aber: Svens 6 `ready`-Entwürfe (u. a. die zwei GTA-III-USK-18-Leichen) sterben, sobald sie 7 Tage unberührt sind — und `item["draft_id"]` bleibt dann als toter Zeiger stehen. Damit fällt auch die Spez-Zusage „archivieren = Soft-Delete, echtes Löschen erst nach 30 Tagen" (§6 Stufe 2) in sich zusammen: der Bot löscht nach 7.

### F3 — „Identische Bild-Bytes werden stumm verworfen, daraus entsteht kein zweites Stück" (Spez §5).
Falsch. Der SHA1-Dedupe in `stage_add` (`app_api.py:1866-1877`) vergleicht **nur gegen Dateien, die gerade in derselben Ablage liegen**. `items_from_stage` leert die Ablage (`:1957`, `shutil.rmtree(d)`), und `STAGE_GAP` räumt sie zusätzlich (`:1858-1863`). Ein zweiter Upload derselben Bytes nach einem abgeschlossenen Scan wird **nicht** erkannt. Genau das belegt die Forensik der Lebenszyklus-Analyse (dreimal `2c54d8f3…`).

### F4 — „Stück löschen rührt den Entwurf nicht an" (B3 / Lebenszyklus b1).
Seit 13:31 nicht mehr wahr: `app_api.py:2122-2131` löscht den Entwurf mit, sofern sein Status nicht in `("published","ended","dry_run_done")` liegt. Dieser Code ist **ungeprüft** und selbst der derzeit größte Risikoposten (siehe K1).

### F5 — „`pruefeSchleier` prüft nur `!$("sheet").hidden`" (Sheet-Analyse Pfad 3/6).
Falsch. `sero.js:3821`:
```js
const offen = !$("sheet").hidden || !$("detail").hidden || !!document.querySelector(".party");
```
Er prüft drei Dinge. Das macht ihn **stumpfer**, nicht schärfer — jedes hängende `#detail` und jedes liegengebliebene `.party` blockiert das Entfernen von `recede` dauerhaft (siehe Ü2/Ü3).

### F6 — „Cache-Buster nicht mitgezogen, `sero.js?v=93`/`sero.css?v=52`".
Überholt. `index.html:23` = `sero.css?v=53`, `index.html:182` = `sero.js?v=94`. Beide wurden erhöht.

### F7 — „`aehnliches_stueck()` ist rein textbasiert (`card_key` oder identischer Name)".
Ungenau. `app_api.py:400-434` bildet einen Fingerabdruck aus **den ersten 4 Wörtern + der Menge aller Zahlen** (`:409-415`), ignoriert `card_key`, wenn er mit `solo:` beginnt (`:419-420`), und **überspringt verkaufte Stücke** (`:427`). Für Videospiele wie „Grand Theft Auto III" liefert der Katalog fast sicher `solo:` ⇒ es zählt nur der Wort-/Zahlen-Fingerabdruck.

### F8 — Spez §5 baut auf Funktionen, die es so nicht gibt.
`catalog.grade_bucket` (`web/catalog.py:81`) und `catalog._norm` (`:44`) existieren, aber:
- Regel 1 („`card_key` identisch") greift bei `solo:`-Schlüsseln **nie** — also genau im GTA-Fall nicht.
- Regel 2 („Ähnlichkeit ≥ 0,9") existiert **nirgends** im Code; es gibt keine Ähnlichkeitsfunktion.
- Die bereits vorhandene, für genau diesen Zweck geschriebene `aehnliches_stueck()` wird in der Spez gar nicht als Basis genommen. Man würde eine zweite, konkurrierende Dubletten-Logik einführen.

### F9 — Spez §4 nennt das Ereignis „Stück wiederhergestellt".
Es gibt **keinen Restore-Endpunkt**. `_trash` (`app_api.py:2111-2118`) wird nur beschrieben, in `_purge_trash` (`:2641-2651`) nach 30 Tagen gelöscht und in der DSGVO-Löschung (`:2812-2820`) gelesen. Das „Rückgängig" in der App ist rein clientseitig: ein 6-Sekunden-`setTimeout`, der den DELETE-Request gar nicht erst absendet (`sero.js:114-124`).

### F10 — B1: „Der `ended`-Eintrag hängt für immer ohne `item_id` im Verkauf-Tab."
Zweifach falsch: er wird nach 7 Tagen gelöscht (F2), und der Verkauf-Tab zeigt ohnehin nur `published`/`ended`/`ready|dry_run_done|error|uncertain` (`app_api.py:3007-3011`) **und nur mit `listing`** (`:3004`). Entwürfe in `new`/`downloading`/`analyzing` sieht Sven **nie**. Die Zahl „29 Entwürfe" ist nicht das, was in seiner Oberfläche steht.

### F11 — Kleinkram
- „Genau **drei** Stellen rufen `store.create_draft()`" — es sind **zwei**: `app_api.py:2180` und `bot/main.py:1032` (grep über das ganze Projekt).
- „`_list_locks` deckt nur denselben Item-Key ab" (Spez §2.3) — stimmt (`:2149`), aber der Schlüssel enthält bereits `account['id']` (`:2149`), die Formulierung „pro Item" ist also enger als der Code.

---

## (2) Vorgeschlagene Fixes, die etwas Funktionierendes brechen — nach Risiko

### K1 — HÖCHSTES RISIKO, bereits im laufenden Code: `NoneType`-Absturz in der Listing-Pipeline
Der 13:31-Patch löscht Entwürfe auch im Status `downloading` und `analyzing` (`app_api.py:2124-2131` — die Ausnahmeliste ist nur `published/ended/dry_run_done`). Gleichzeitig läuft für genau diesen Entwurf `prepare_and_run` im Hintergrund:

```
app_api.py:2235   async def prepare_and_run():
app_api.py:2236       draft = store.get_draft(draft_id)      ← KEINE None-Prüfung
   …Render-Schleife, mehrere Sekunden bis Minuten (BiRefNet)…
app_api.py:2253       draft["original_photos"] = originals   ← TypeError, wenn gelöscht
```

**Ablauf:** „Auf eBay listen" tippen → sofort das Stück löschen (genau Svens Aufräum-Verhalten heute Mittag) → der Entwurf ist im Status `downloading` → wird gelöscht → `prepare_and_run` stirbt bei `:2253`. `_spawn` (`:184-187`) fängt nichts ab, die Exception landet nur im Task. `_list_locks` (`:2140-2156`) schützt nicht gegen Löschen.

Zum Vergleich: für Sammlungsstücke ist genau dieses Muster längst gelöst — `col_save_analyse` (`:736-753`) liest frisch und gibt `None` zurück, wenn das Stück während der Analyse verschwand. Der Entwurfs-Pfad hat diesen Schutz nicht. **Das gehört repariert, bevor irgendetwas anderes an der Kopplung passiert.**

Nebenbefunde am selben Patch:
- `store.get_draft(did)` statt `own_draft(did, account)` (`:2124`) — keine Besitzprüfung.
- Kein `cleanup_photos(draft)` — `tmp/<draft_id>/` bleibt liegen, bis der Bot aufräumt.
- Asymmetrie: das Stück wandert 30 Tage in `_trash` (`:2111-2118`), der Entwurf ist **sofort und endgültig** weg.

### K2 — „`published` → Löschen verweigern" (Spez §4) bricht die Lösch-UI still
`removeItemWithUndo` (`sero.js:114-124`) entfernt das Stück **optimistisch** aus `state.items`, rendert neu und schickt den DELETE erst nach 6 s — mit `.catch(() => {})`. Ein Server-Fehler wird **verschluckt**; `loadCollection()` holt das Stück still zurück. Für Sven sähe das so aus: Stück verschwindet, kommt 6 Sekunden später kommentarlos wieder, keine Meldung. Der Fix muss zwingend zusammen mit einer Fehlerbehandlung im Frontend kommen — sonst ist er schlimmer als der Ist-Zustand.

### K3 — Bestätigungs-Rückfragen im `/list`-Endpunkt brechen zwei Aufrufer
Spez §3 will bei `ended` + `sold_ts` eine Rückfrage und ein neues Antwortfeld `action`. Beide Aufrufer kennen das nicht:
- `listNow` (`sero.js:2373-2381`) wertet **jede** 2xx-Antwort als Erfolg und toastet fest `"Entwurf erstellt — liegt im Verkauf-Tab"`.
- Der Auto-List-Wächter (`sero.js:1892-1897`) ruft `listNow` bei jedem SSE-Event für `status==="ready" && !draft_id` — er würde eine „Bestätigung nötig"-Antwort als Erfolg verbuchen und die ID aus `sellWatch` löschen (`:1894`), d. h. **der Nutzer bekommt die Rückfrage nie zu sehen und das Stück wird nie gelistet.**

### K4 — `draft["item_id"]` + `item["draft_ids"]`: die Schreib-Stelle ist die falsche
`_list_collection_item` speichert bei `:2185-2186` den **kompletten** Item-Blob, der bei Funktionseintritt gelesen wurde (`:2160`), per `col_save` — **nicht** über das schonende `col_save_analyse` (`:736-753`). Läuft parallel eine Analyse, ein `refresh_item_price` oder eine Nutzer-Eingabe, wird deren Ergebnis überschrieben. Jede zusätzliche Schreiboperation an dieser Stelle (Historie-Liste!) vergrößert dieses Fenster. **Erst die Schreib-Disziplin herstellen, dann Felder ergänzen.**

### K5 — `Store.update_draft` setzt den Status auf `"new"` zurück
`bot/drafts.py:272-279`:
```python
"UPDATE drafts SET data = ?, status = ?, … ", (json.dumps(clean), clean.get("status", "new"), …)
```
Jeder neue „Entwurf aktualisieren"-Pfad (Spez §3, Zeile `ready/dry_run_done`), der ein Teil-Dict übergibt, setzt den Entwurf auf `new` — er **verschwindet damit aus dem Verkauf-Tab** (`app_api.py:3007-3011` kennt `new` nicht) und wird zum unsichtbaren Zombie. Alle 23 bestehenden Aufrufer machen Read-Modify-Write mit vollem Dict; das ist eine ungeschriebene Regel, die man leicht bricht.

### K6 — Telegram-Bot: die Invariante ist im Web nicht durchsetzbar
`uid_for` (`app_api.py:223-226`) liefert bei verknüpfter Telegram-ID **die Telegram-ID**. Alle 29 Entwürfe haben `chat_id 5694742134` — App und Bot teilen sich denselben Namensraum. Konkret:
- `bot/main.py:1187` `store.latest_draft_for_chat(chat_id)` (`drafts.py:286-291`) holt **den zuletzt erstellten Entwurf des Chats, egal aus welcher Quelle**. Schreibt Sven dem Bot innerhalb von 60 s nach einem App-Listing irgendeinen Text — oder steht der App-Entwurf auf `uncertain` — hängt der Bot den Text als `caption` an **Svens App-Entwurf** und startet `run_pipeline` neu (`bot/main.py:1190-1195`). Das überschreibt das Listing des Sammlungsstücks.
- `bot/main.py:1388` (`discard`-Callback) und `:1046` löschen Entwürfe ohne jede Rücksicht auf `item["draft_id"]` — **zwei weitere Verwaisungs-Quellen**, die keine der drei Analysen nennt (sie nennen nur den Web-`discard` und `stale_draft_ids`).

Jede Kopplungs-Regel, die nur in `app_api.py` steht, ist damit **umgehbar**. Wer `draft["item_id"]` einführt, muss den Bot mindestens dazu bringen, verlinkte Entwürfe zu ignorieren — sonst sieht Sven im Bot plötzlich seine Sammlungs-Entwürfe und kann sie dort kaputtmachen.

### K7 — Klasse-B-Vorschlag („verwaisten `published`-Entwurf einem Stück zuordnen") ändert rückwirkend Svens Sammlungswert
`portfolio_series` (`app_api.py:628-634`) und `portfolio_ohlc` (`:653-663`) schließen **jedes Stück aus, dessen `draft_id` zu einem `published`-Entwurf gehört**:
```python
qty = {i["id"]: … for i in col_all(account_id) if not (i.get("draft_id") in live_draft_ids)}
```
Ein Aufräum-Lauf, der ein bestehendes Stück an den verwaisten `published`-Entwurf `74925bead28e` („Vice City") knüpft, entfernt dieses Stück **rückwirkend aus dem Wert-Chart und der Kopfzahl**. Sven sieht seinen Sammlungswert fallen, ohne dass er etwas verkauft hat. Das ist genau die Art Kollateralschaden, vor der er gewarnt hat.

### K8 — Soft-Delete (`archived_at`, 30 Tage) ist ohne Bot-Änderung nicht haltbar
Siehe F2: `stale_draft_ids` löscht alles Nicht-`published` nach 7 Tagen hart. Ein „archivierter" Entwurf ist nach 7, nicht nach 30 Tagen weg. Um die Zusage zu halten, müsste man `bot/drafts.py:292-306` ändern — also **den Bot-Betrieb anfassen**, der heute funktioniert.

### K9 — `/sales` kollabiert doppelte Zuordnungen still
`app_api.py:2996`: `item_by_draft = {i.get("draft_id"): i for i in col_all(…) if i.get("draft_id")}` — ein **Dict**. Zwei Stücke mit derselben `draft_id` ⇒ eines verschwindet lautlos aus der Anzeige. Eine automatische Adoptions-Routine (Spez §6 C1/C2) kann genau diesen Zustand herstellen, und man sieht es nicht. Vor jeder Massen-Adoption braucht es eine Eindeutigkeits-Prüfung.

### K10 — Dubletten-Erkennung „nach der Analyse bei `card_key`" (Spez §5) greift an drei Stellen daneben
`adopt_draft` (`:2671-2714`) und `import_listings` (`:2718 ff.`) legen Stücke mit `"status": "ready"` an, **ohne je durch die Analyse zu laufen** — kein `card_key`, kein Hook. Genau diese beiden Wege haben Svens 1-Euro-Geisterstücke erzeugt. Und `rescue_stuck_scans` (`:2607-2636`) setzt hängende Items alle 5 Minuten zurück auf `analyzing` und stößt sie neu an — ein Dubletten-Banner, das an der Analyse hängt, würde bei jedem Rettungslauf erneut feuern und (bei Auto-List-Aussetzung) das Stück dauerhaft blockieren.

---

## (3) Was übersehen wurde

### Ü1 — DATENVERLUST, heute live: der Fehlerpfad löscht die geparkten Fotos
`closeSheet` (`sero.js:3794-3801`):
```js
if (state.stageOpen) {
  state.stageOpen = false;
  if (state.stageKeep) state.stageKeep = false;
  else post("/api/app/collection/stage/clear?" + devQ()).catch(() => {});
}
```
`handleScanError` (`:309-313`) ruft bei 402 `closeSheet(); openPaywall();`. Im Save-Handler von `openStagedSheet` (`:2456-2480`) wird `state.stageKeep = true` **erst nach** der erfolgreichen Antwort gesetzt (`:2469`). Bei 402 ist `stageKeep` also `false` ⇒ **die Fotos des Nutzers werden serverseitig gelöscht**, während ihm die Paywall gezeigt wird. Serverseitig sind sie zu diesem Zeitpunkt noch da: `claim_scans` (`:1932`) bricht **vor** `shutil.rmtree(d)` (`:1957`) ab.

Sven trifft das nicht (F1), **jeden zahlenden Neukunden am Limit aber sehr wohl.** Keine der drei Analysen nennt das — die Sheet-Analyse behandelt denselben Pfad ausschließlich als Kosmetik-Problem.

### Ü2 — Der Grau-Hänger, den der Patch NICHT abdeckt: `#detail` hat exakt dieselbe `closing`-Race
Genau der Fehler, der für `.sheet` gerade behoben wurde, steckt unverändert im Detail-Overlay:

```js
sero.js:3033  d.classList.add("closing");
sero.js:3034  setTimeout(() => { d.hidden = true; d.classList.remove("closing"); }, 260);
```
```js
sero.js:3042  async function openItemDetail(itemId, seg = "overview") {
sero.js:3044    $("detail").hidden = false;      ← entfernt "closing" NICHT
```
`sero.css:695`: `.detail.closing { animation: slideout … forwards; }` — `forwards` friert das Overlay im Endzustand ein. Der Timer aus `closeDetail` prüft **nichts** (anders als der reparierte Sheet-Timer bei `:3810-3811`).

Folge: Wird innerhalb von 260 ms nach `closeDetail()` ein `openItemDetail()` ausgelöst, ist `#detail.hidden === false`, aber unsichtbar — und `pruefeSchleier` (`:3821`) zählt `!$("detail").hidden` als „etwas ist offen" und **entfernt `recede` nie mehr**. Erreichbar u. a. über den Verkauf-Tab (`:2739-2747`, Klick auf eine Zeile öffnet ein Detail) und über `showScanResult`→`#resOpen` (`:359`, 300-ms-Timer). Das ist ein **dauerhafter Grau-Hänger nach dem Patch** und wird von keiner Analyse genannt.

### Ü3 — `.party`-Overlays haben keinen Auto-Dismiss und blockieren den Wächter dauerhaft
`showScanResult` (`:316-362`) und `showScanFailed` (`:366-402`) hängen ein `.party` an `document.body`, das **nur** per `#resOpen`/`#resNext`/`#resRetry` oder Klick exakt auf den Hintergrund (`:361`, `e.target === el`) verschwindet. Kein Timeout, keine Sperre gegen zwei gleichzeitige Overlays — `loadCollection` (`:1985-1991`) kann `showScanResult` erneut auslösen, sobald `state.watchNew` wieder gesetzt ist. Ein liegengebliebenes `.party` macht `document.querySelector(".party")` in `pruefeSchleier` (`:3821`) dauerhaft wahr ⇒ `recede` wird **nie** entfernt. Die Sheet-Analyse erkennt die Blockade (Pfad 6), zieht aber nicht den Schluss, dass das ein eigenständiger Dauer-Hänger ist.

### Ü4 — Pfad 3 ist korrekt, aber unvollständig beschrieben
`openSheet` (`:3760-3792`) setzt weder `sh.style.transform` noch `sh.style.transition` zurück. `closeSheet` setzt nur `transform` zurück (`:3805`) — und **erst nach** `if (sh.hidden) return;` (`:3804`), also nicht bei einem schon versteckten Sheet. Ein hängengebliebenes `transition:"none"` aus der Griff-Geste (`:3838`) überschreibt zusätzlich die `.closing`-Transition (`sero.css:1114`), und `sy` (`:3831`) bleibt gesetzt, wenn `pointerup`/`pointercancel` ausbleiben — ein späterer `pointermove` ohne `pointerdown` zieht das Sheet dann erneut weg.

### Ü5 — Zwei weitere Verwaisungs-Wege über den Bot
`bot/main.py:1388` (`discard`-Callback) und `bot/main.py:1046` (fehlgeschlagener Foto-Download) rufen `store.delete_draft` ohne jeden Bezug zu `collection_items`. Da Bot und App dieselbe `chat_id` teilen (K6), kann Sven per Telegram einen App-Entwurf verwerfen ⇒ `item["draft_id"]` zeigt ins Leere ⇒ Knopf heißt für immer „Listing-Entwurf öffnen" (`sero.js:2570`, `:2584`). Die Analysen nennen nur den Web-`discard` (`app_api.py:1516-1523`) und `stale_draft_ids`.

### Ü6 — Verwaiste Nebentabellen beim Löschen
`delete_collection_item` (`:2132-2135`) löscht nur aus `collection_items`. `price_history` und `price_alerts` behalten ihre Zeilen. **Kein Schaden an den Zahlen** — `portfolio_series`/`portfolio_ohlc` filtern über `qty` (`:641`, `:663`) und die Alarm-Anzeige über `a["item_id"] in by_id` (`:2932`). Reiner Speicher-Müll, aber wer den Papierkorb umbaut, sollte es wissen.

---

## (4) Sicherste Reihenfolge

**Stufe 0 — Blutungen stoppen (heute, klein, ohne Datenmodell-Änderung)**
1. `prepare_and_run` (`app_api.py:2236`): `if not draft: return` — behebt K1, den einzigen echten Absturz. Zusätzlich in `:2253-2259` frisch lesen statt am alten Objekt weiterschreiben.
2. `delete_collection_item` (`:2124`): `own_draft(...)` statt `store.get_draft(...)`, und Entwürfe in `downloading`/`analyzing` **nicht** löschen, sondern nur markieren — sonst reißt man laufende Pipelines ab.
3. `handleScanError` (`sero.js:310`): `state.stageKeep = true` **vor** `closeSheet()` — behebt Ü1 (Datenverlust).

**Stufe 1 — Beschwerde 1 wirklich schließen (Frontend, kein Backend-Risiko)**
4. `openItemDetail` (`sero.js:3044`): `closing` entfernen + Timer in `closeDetail` (`:3034`) prüfen lassen — spiegelbildlich zum Sheet-Fix. Behebt Ü2.
5. `openSheet`: `sh.style.transform = ""`, `sh.style.transition = ""`, Griff-`sy` invalidieren. Behebt Ü4/Pfad 3.
6. `.party`-Overlays: vor dem Anhängen bestehende `.party` entfernen; `pruefeSchleier` zusätzlich prüfen, ob das offene Element wirklich im Viewport liegt. Behebt Ü3.
7. Erst **danach** entscheiden, ob `switchTab("tabScan")` nach dem Upload (`sero.js:2472-2474`, `:2518-2523`, `:2549-2550`) bleibt — das ist das „springt zurück", eine reine Produktentscheidung.

**Stufe 2 — Diagnose statt Vermutung**
8. Einen **nur lesenden** Zähler/Log bauen: verwaiste lebende Entwürfe, Zeiger ins Leere, Stücke mit doppelter `draft_id`. Läuft mit, ändert nichts. Ohne diese Grundlinie ist später nicht beweisbar, dass ein Fix gewirkt hat.

**Stufe 3 — Kopplung, additiv und rückwärtskompatibel**
9. `draft["item_id"]` **nur zusätzlich** schreiben (`:2185`), nichts darauf verlassen. Vorher K4 lösen (schonendes Schreiben à la `col_save_analyse`).
10. Referenz aufräumen an den Stellen, wo heute Zeiger ins Leere entstehen: `discard` (`:1516-1523`), `end` (`:1556-1568`), Sales-Sync (`:2540-2545`). Rein additiv, bricht nichts.
11. `stale_draft_ids` (`bot/drafts.py:292-306`) so ändern, dass verlinkte und `ended`-Entwürfe überleben — **erst jetzt**, weil es den Bot-Betrieb berührt, und mit vorherigem Trockenlauf.

**Stufe 4 — Dubletten, ausschließlich als Vorschlag**
12. Die **vorhandene** `aehnliches_stueck()` (`:400-434`) auch beim Auto-Listen abfragen (`sero.js:1894`) und dort nur **aussetzen + Banner**, nie automatisch zusammenführen. Keine zweite Vergleichslogik einführen (F8). Der Hinweis inkl. `draft_id` wird bereits geliefert (`:2046-2050`) und im Frontend angezeigt (`sero.js:3142-3145`) — er muss nur an der richtigen Stelle abgefragt werden.
13. `adopt_draft` (`:2671`): den Automatismus im Verkauf-Tab (`sero.js:2739-2747`) durch eine Rückfrage ersetzen und den Auktions-Startpreis nicht mehr als `est_value` übernehmen (`:2701`, `:2713`). Das ist der Verstärker, der die Sammlung mit 1-Euro-Stücken geflutet hat.

**Stufe 5 — Altlasten**
14. Aufräumen der 7 verwaisten Entwürfe: Trockenlauf, Ausgabe an Sven, Ausführung nur nach ausdrücklicher Bestätigung — und **ohne** die Klasse-B-Verknüpfung von `published`-Entwürfen (K7), solange nicht geklärt ist, dass Sven den Sprung im Sammlungswert versteht und will.

**Nicht anfassen, bevor Stufe 0-2 stehen:** `published`-Entwürfe, `sync_sales_status`, `publish-drafts` (Bulk), `import_listings`. Dort hängt echtes Geld — Sven hat aktuell zwei parallele Live-Listings für „Vice City" (`147480067874` via `74925bead28e` ohne Stück, `147480090061` via `9f8f51762eca`); das ist ein eBay-Problem, kein Code-Problem, und sollte von Hand geklärt werden, bevor eine Automatik daran rührt.