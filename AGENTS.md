# SERO PWA — Regeln für KI-Assistenten

**Antworte auf Deutsch.** Dies ist nur das Frontend. Backend, Tests und die
vollständigen Regeln liegen in `~/ebay-bot` — dort `AGENTS.md` und
`docs/IMPLEMENTATION_STATUS.md` lesen, bevor du hier etwas änderst.

## Was hier liegt

`web/sero.js` (die komplette App, kein Framework), `web/sero.css`,
`web/sero-dark.css`, `web/index.html`. Ausgeliefert wird das vom Backend
unter `http://localhost:3000/app/` — **Speichern ist sofort live**, kein Build.

## Die zwei Fallen

1. **Den passenden Versions-Pin hochzählen.** `index.html` führt DREI
   unabhängige Pins: `sero.css?v=`, `sero-dark.css?v=` und `sero.js?v=`.
   CSS-Änderung → CSS-Pin, JS-Änderung → JS-Pin. Zählst du den falschen hoch,
   liefert der Cache dem Handy weiter die alte Datei. Der Smoke-Test prüft
   den Dark-Pin nicht — daran denken.
2. **Kein eigener Server auf Port 3000.** Dort läuft der Dauerdienst, an dem
   Svens Handy hängt. Zum Neuladen des Backends:
   `launchctl kickstart -k gui/501/com.listo.web`.

## Texte

Duzen. Keine Ausrufezeichen, keine Emojis. Die App sagt nie „Wir".
„Stück", „listen", „Marktwert", „tippen".

Das englische Wörterbuch heißt `STR_EN`; **der deutsche Text IST der
Schlüssel**, nachgeschlagen über `L()`. Fehlt ein Schlüssel, fällt die Ausgabe
still auf Deutsch zurück — kein Fehler, kein roter Test. Wer einen deutschen
Text ändert, muss den Schlüssel dort mitändern, nicht nur neue Texte
eintragen. Zeichengenau kopieren (Auslassungspunkte, Anführungszeichen).

**Preise nie erfinden.** Ist die Analyse fertig (`status === "ready"`) und es
gibt keinen belegten Marktwert, heißt es „Wert unbekannt" — keine Schätzung.
Solange die Analyse läuft, ist „Wert wird noch ermittelt" richtig und bleibt.

## Nicht vorschlagen

Drag-and-Drop-Sortierung (war gebaut, auf dem Handy unbrauchbar) und ein
Framework-Umbau ohne konkrete Begründung.

Prüfen: `node --check web/sero.js`, danach `sh ~/ebay-bot/tests/smoke.sh`.
