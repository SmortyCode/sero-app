# Best-in-Class-Muster für SERO — destilliert und angedockt

Basis: `/Users/smorty/sero-app/web/sero.js` (v85), `/Users/smorty/sero-app/web/sero.css` (v43), `/Users/smorty/sero-app/web/index.html`, `/Users/smorty/sero-app/legal/marketing-bausteine.md`.

## Wo SERO schon Best-in-Class ist (ehrlich, keine Schmeichelei)

Diese Dinge sind auf dem Niveau der Vorbilder und brauchen keine Nacharbeit:

- **Ein Spring für alles**: `--spring: cubic-bezier(.32,.72,0,1)` zieht sich durch Buttons, Tabs, Seitenwechsel, Kacheln (`sero.css:69` ff.). Genau das ist das Things-3-Geheimnis — eine Bewegungssprache, nicht zehn.
- **Richtungsbewusste Seitenwechsel** (`switchTab`, `sero.js:1392–1408`, `page-enter-l/r`) — räumliche Kontinuität, die selbst viele native Apps nicht haben.
- **Undo statt Confirm** beim Löschen (`removeItemWithUndo`, `sero.js:114–125`) — Linear-Schule, korrekt umgesetzt (Server-Delete erst nach Ablauf des Undo-Fensters).
- **Ehrlicher Fehl-Scan** (`showScanFailed`, `sero.js:343`) statt Fake-Erfolg, Offline-Foto-Netz (`sero.js:2370–2388`), Boot-Sicherheitsnetz (`sero.js:1313`).
- **Messwerte statt Werbezahlen**: Scanner-Stats erst ab 3 echten Scans (`sero.js:2575`), Meilenstein-Drossel für Bestandsnutzer (`sero.js:245`).
- **Holo-Gyro mit `prefers-reduced-motion`-Respekt** (`sero.css:684`) und iOS-Permission-Gate (`sero.js:3392–3396`).

Die folgenden 12 Muster setzen dort an, wo die Vorbilder noch etwas haben, das SERO fehlt.

---

## 1. Flighty — Gestaffelte Enthüllung des Ergebnisses

**Was:** Flighty zeigt nach der Flugsuche erst die Karte, dann faden die Fakten gestaffelt ein (~80 ms versetzt), die wichtigste Zahl zuletzt. Die Daten „treffen ein", statt da zu sein.

**Warum:** Gestaffelte Ankunft erzählt „hier wurde gerade live gerechnet" — dieselben Daten auf einen Schlag wirken wie eine statische Seite. Der letzte Slot ist der Höhepunkt.

**Wie in SERO:** `showScanResult` (`sero.js:303–339`) injiziert die komplette `res-card` auf einmal; nur der Wert zählt hoch. Die Mechanik für Staffelung existiert schon im Grid: `animation-delay: calc(var(--i,0) * 40ms)` (`sero.css:537`). Übertragen: `res-photo` sofort, `res-badge` „Erkannt" +80 ms, `res-name`/`res-sub` +160 ms, `res-val`-Odometer +320 ms als Finale. Dazu die Haptik verschieben: `navigator.vibrate(12)` feuert aktuell beim Öffnen (`sero.js:338`) — sie gehört auf den Moment, in dem der Odometer landet. Das ist der Wow-Moment der App; er kostet vier `animation-delay`-Zeilen.

## 2. Flighty — Benannte Wartephasen (die Texte liegen schon fertig da)

**Was:** Flighty benennt jede Phase des Wartens („Checking gate…"). Der Spinner erklärt, was gerade passiert.

**Warum:** Benannte Phasen verkürzen gefühlte Wartezeit und bauen nebenbei Vertrauen in die Pipeline auf — „Echte Verkäufe statt Wunschpreise" als Ladetext ist Positionierung im Wartemoment.

**Wie in SERO:** Die Analyzing-Zeilen zeigen `status_text` (`sero.js:2560` im Scan-Verlauf, `sero.js:3070` im Detail), gepollt alle 2,2 s (`sero.js:1934`). Die fünf Ein-Zeiler in `marketing-bausteine.md` §5 sind exakt dafür geschrieben („Label wird gelesen, Note folgt gleich.", „Echte Verkäufe statt Wunschpreise."). Falls der Server keine echten Phasen liefert, clientseitig rotieren: nach 3 s Zeile 3, nach 6 s Zeile 4. Null neues Wording nötig — es liegt seit Wochen im Legal-Ordner.

## 3. Flighty — Paywall mit den eigenen Live-Daten

**Was:** Flightys Paywall zeigt nicht Features, sondern DEINEN bereits getrackten Flug — und gated dann das Behalten.

**Warum:** Eine generische Rechnung ist ein Argument; die eigene Rechnung ist ein Beweis. Wer 47 Stücke gescannt hat, liest „~13 Std für 100 Karten" als Werbung, aber „Du hast 6 Stunden gespart" als Kontoauszug.

**Wie in SERO:** `openPaywall` (`sero.js:268–294`) zeigt die statische 100-Karten-Rechnung, obwohl `state.dash.time_saved` mit echten `items`/`saved_s`/`by_hand_s` direkt daneben liegt (genutzt in `timeSavedCard`, `sero.js:141–145`). Die personalisierte Variante ist in `marketing-bausteine.md` §4 fertig formuliert, inklusive Singular- und Minuten-Regeln. Ab `items >= 3` die persönliche Rechnung oben einsetzen, die 100-Karten-Rechnung als Fußnote darunter. Höchster Conversion-Hebel im ganzen Report bei kleinstem Aufwand.

## 4. Robinhood — Der Chart antwortet dem Finger

**Was:** Robinhoods Signatur: Finger auf den Chart legen und ziehen — die große Zahl darüber und das Delta laufen live mit, beim Loslassen springt alles federnd auf „heute" zurück, mit Haptik-Tick pro Datenpunkt.

**Warum:** Direkte Manipulation verwandelt den Portfolio-Wert von einer Anzeige in einen Besitz. „Was war meine Sammlung vor zwei Wochen wert" wird eine Geste statt einer Frage — das ist der emotionale Kern von „Wertverlauf".

**Wie in SERO:** Alles liegt bereit: `.big-chart` mit `candleChart`/`sparkline` (`sero.js:1634–1637`), `.ov-value` mit Odometer (`countUp`, `sero.js:507`, gefeuert in `1698`), `ohlc`/`histPts` mit Datumsachse (`1590`, `1632`), `ohlcLegend` (`1656`) zeigt schon OHLC-Werte einer Kerze. Ein `touchmove`-Handler auf `.big-chart`: Index aus `clientX` ableiten, `.ov-value` und `.ov-delta` auf den Punktwert setzen, `ohlcLegend` auf die berührte Kerze, `touchend` stellt Heute wieder her. Der Veil-Modus (`sero_hide`, `1596`) muss den Scrub natürlich mit verschleiern.

## 5. Things 3 — Bewegung mit Zieladresse

**Was:** In Things fliegt ein abgehakter Task sichtbar dorthin, wo er hingehört (ins Logbuch). Elemente verlassen den Screen in Richtung ihres Ziels, nie einfach per Fade.

**Warum:** Das baut ein räumliches Modell: Der Nutzer weiß ohne Nachdenken, WO seine Karte jetzt ist. Gerade bei einer Sammlung ist „es liegt jetzt dort" der halbe Besitz-Effekt.

**Wie in SERO:** `showScanResult` schließt mit generischem `out`-Fade (`sero.js:331–333`), danach ist die Karte „irgendwo". Übertragen: Beim Schließen skaliert `res-photo` per Transform auf die Position des Sammlung-Tabs (Element `data-tab="tabCollection"`, `index.html:143`), und dessen `.tic` feuert den vorhandenen `ticpop` (`sero.css:202–203`). Bei „Weiter scannen" (`resNext`, `sero.js:336`) genügt die kurze Flugbahn als Bestätigung, dass das Stück gesichert ist, bevor die Kamera aufgeht. Gleiches Muster für `listNow`: Entwurf fliegt Richtung Verkauf-Tab.

## 6. Linear — Optimistic UI konsequent zu Ende

**Was:** In Linear reflektiert jede Aktion sofort im UI; der Server zieht nach, Fehler rollen sichtbar zurück. Unter 100 ms Reaktion, immer.

**Warum:** Das ist auf Mobile das gesamte „native Gefühl". Ein Stern, der erst nach dem Roundtrip umschaltet, fühlt sich nach Website an — egal wie schön der Rest ist.

**Wie in SERO:** Das Löschen ist schon optimistisch (Undo-Muster). Der Favoriten-Stern nicht: `detailFav` (`sero.js:3040–3042`) macht `post → refreshDetail(true)` und wartet auf den Server, ebenso hängt der Preisalarm-Zustand am Refresh. Übertragen: Stern sofort togglen (`icon("starfill")` + Farbe), `post` im Hintergrund, bei Fehler zurückdrehen plus Toast. Dasselbe Muster für Tags und Notizen im Detail. Eine Handvoll Stellen, systematisch durchgehen.

## 7. Linear/iOS — Swipe-to-List: die Positionierung als Geste

**Was:** Linears mobile Erfolgsformel ist „die eine Kernaktion liegt auf einer Wischgeste" (Swipe = Issue erledigt). iOS-Mail hat den Verkauf derselben Idee: Archivieren ohne Zielknopf.

**Warum:** SEROs Versprechen ist „Collectr sagt dir, was deine Karten wert sind. SERO macht sie zu Geld." Wenn jede Kachel in der Sammlung einen Wisch von einem eBay-Listing entfernt ist, wird die Positionierung körperlich erfahrbar — kein Konkurrent kann diese Geste überhaupt anbieten, denen fehlt die Aktion dahinter.

**Wie in SERO:** Die Aktion existiert als Funktion: `listNow(id)` (`sero.js:2313`), inklusive Auto-Listing-Pfad im Sell-Modus (`sellWatch`, `1832–1837`). Die Kacheln haben schon `touch-action: pan-y` (`sero.css:536`) — horizontale Wischwege sind frei. Swipe links auf `irow`/Kachel legt einen navy „Listen"-Balken frei, Auslösen ruft `listNow`; Swipe rechts → `removeItemWithUndo` (Undo-Netz vorhanden). Wichtig: nur in Listenansichten (`scanHistory`, Verkauf), im Grid reicht das lange Drücken über `openItemMenu` (`sero.js:2507`).

## 8. Airbnb — Shared-Element-Übergang: die Karte bleibt dieselbe Karte

**Was:** Airbnb öffnet ein Listing, indem das Foto der Kachel selbst zur Hero-Ansicht aufzieht (shared element). Kein Schnitt, ein Objekt.

**Warum:** Für Sammler ist die Karte ein Objekt, kein Datensatz. Wenn die Kachel aus dem Grid physisch zum Detail aufzieht, fühlt sich die Sammlung wie ein Album an — das ist der Unterschied zwischen Datenbank und Besitz.

**Wie in SERO:** Kachel-Tap ruft `openItemDetail` (`sero.js:2975`), das `#detail`-Overlay (`index.html:151–159`) erscheint ohne räumliche Verbindung; das erste Bild landet in `.d-photos` (`sero.js:3063`, bei Karten im `holo-wrap`, `3052`). iOS-18-Safari kann die View-Transitions-API: `document.startViewTransition` um den Detail-Aufbau legen, `view-transition-name` dynamisch auf die getippte Kachel-`img` und das Ziel-`img` setzen, Fallback bleibt das heutige Verhalten. Der Gyro-Holo-Effekt direkt nach dem Aufziehen ist dann der zweite Schlag desselben Moments.

## 9. Arc — Onboarding endet IN der Handlung, nicht vor ihr

**Was:** Arcs Onboarding hat zwei Tricks: eigene Sprache statt Feature-Prosa, und der letzte Schritt IST die erste echte Handlung (du baust deinen ersten Space, statt einen „Fertig"-Knopf zu drücken).

**Warum:** Jeder Tap zwischen Tour-Ende und erstem Scan kostet Aktivierung. Und die Tour ist der einzige Ort, an dem die App ihre Haltung zeigen darf, bevor Daten da sind.

**Wie in SERO:** Die Tour (`TOUR`, `sero.js:1267–1272`) endet mit „Erste Karte scannen", macht aber nur `switchTab("tabScan")` (`1295`) — der Nutzer steht vor einem weiteren Knopf. `resNext` zeigt schon den richtigen Griff: `switchTab` + `$("cameraInput").click()` (`336`). Genau das gehört ans Tour-Ende. Zweitens: Die Tour-Texte sind Feature-Prosa („Deine Sammlung bekommt einen Gesamtwert mit täglichem Verlauf …"); in `marketing-bausteine.md` §1 liegen drei fertige, schärfere Sets — Set A (Fotografieren / Bestätigen / Listen) ist laut Doku die nächste Entsprechung zum Kernablauf und landet auf dem App-Wording.

## 10. Collectr / CardLadder — den Beleg-Vorteil sichtbar machen, die Share-Schleife übernehmen

**Was:** Collectr gewinnt durch Portfolio-Emotion und Teilen (Screenshots vom Sammlungswert sind ihr Growth-Kanal), verliert aber Vertrauen, weil Werte aus aktiven Listings aufgebläht wirken. CardLadder gewinnt durch Sales-basierte Ehrlichkeit, endet aber vor dem Verkauf und ist US-zentriert.

**Warum:** SERO sitzt exakt zwischen beiden: Sales-basiert UND mit Verkaufsweg. Dieser Unterschied muss auf einen Blick erkennbar sein — und Collectrs Wachstumsschleife ist übernehmbar, ihre Schwäche nicht.

**Wie in SERO:** Zwei konkrete Züge. Erstens Beleg-Ästhetik: Im Detail steht der belegte Marktwert und die KI-Schätzung (`marketLine`, `sero.js:3090–3092`) in derselben Optik. Die belegte Variante („eBay: N Angebote · Median X" plus `sold_comps` direkt darunter, `3117–3133` — Svens Regel, zu Recht) verdient ein sichtbares Beleg-Signal (z. B. grüner Punkt am `v-label`), die Schätzung ein zurückgenommenes „~". Vertrauen ist das Produkt. Zweitens Share-Karte: Langes Drücken auf den `ov-head` (`1615`) rendert Wert + Delta + Top-Stück als Bild (Canvas, SR-Monogramm, Navy) für `navigator.share` — der Hero-Designer (`openHeroDesigner`, `1761`) liefert die Personalisierung schon, der Veil-Modus (`eyeBtn`) muss respektiert werden. Jeder geteilte Screenshot trägt das Monogramm in Sammler-Chats.

## 11. BeReal / Locket — das Ritual: Session-Bilanz und Homescreen-Widget

**Was:** BeReal baut Gewohnheit über einen abgeschlossenen Moment mit Bilanz; Locket legt den emotionalen Inhalt auf den Homescreen, sodass die App präsent ist, ohne geöffnet zu werden.

**Warum:** SEROs natürliche Nutzung ist der Stapel-Abend des Flippers, nicht der tägliche Check. Das Ritual heißt „Sitzung abschließen und die Bilanz sehen" — und zwischen den Sitzungen hält der Portfoliowert auf dem Homescreen die Bindung.

**Wie in SERO:** Erstens Session-Bilanz: Nach einem Stapel (`openBatchSheet`, `sero.js:2443`, oder N Scans binnen einer Sitzung) ein `party`-Abschluss im Stil von `checkMilestone` (`240–264`): „Heute: 12 Stücke erfasst, 340 € Marktwert" — die Zeile „Der Stapel schrumpft, nicht dein Wochenende." (`marketing-bausteine.md` §5, Nr. 5) ist wörtlich dafür geschrieben. Zweitens Widget: Beim geplanten Capacitor-Schritt ein WidgetKit-Widget mit Portfoliowert + Tagesdelta — `deltas.d7/d30` existieren serverseitig (`sero.js:1592`), es fehlt nur `d1`. Das ist Collectrs meistgewünschtes Feature; wer es zuerst im deutschen Markt hat, besitzt den Homescreen des Sammlers.

## 12. Notion — Kontrollen-Diät über dem Hero-Wert

**Was:** Notion zeigt pro Ansicht genau eine dominante Sache und versteckt Sekundär-Kontrollen hinter einem Menü. Nichts konkurriert mit dem Inhalt.

**Warum:** Der Portfoliowert ist SEROs emotionalster Pixel. Drei gleichgewichtige Icon-Buttons direkt darüber (Chart-Modus, Auge, Refresh — `sero.js:1618–1620`) teilen sich die Aufmerksamkeit mit der Zahl, und der Chart-Modus-Toggle ist ein Einstellungs-, kein Alltagsknopf.

**Wie in SERO:** `chartModeBtn` aus dem `ov-top` entfernen und den Wechsel Kerzen/Linie auf einen Tap auf den Chart selbst legen (`1711–1714` bleibt als Logik identisch) — Robinhood macht es genauso. Das Auge bleibt (Privatsphäre ist ein Alltagsgriff), Refresh bleibt (Pull-to-Refresh existiert zwar über `attachPTR`, `1364`, aber der Knopf ist der sichtbare Fallback). Ergebnis: zwei Buttons, und die Zahl steht allein im Raum.

---

## Priorisierung (Wirkung pro Aufwand)

1. **#3 Paywall personalisieren** — Texte und Daten liegen fertig da, direkter Umsatz-Hebel.
2. **#1 + #2 Scan-Dramaturgie** — der Kern-Loop, nur CSS-Delays und Text-Rotation.
3. **#9 Tour endet in der Kamera** — eine Zeile (`cameraInput.click()`), messbar an Erst-Scan-Quote.
4. **#6 Optimistic UI** (Stern/Alarm) — klein, hebt das Gesamtgefühl.
5. **#4 Chart-Scrub** und **#7 Swipe-to-List** — die beiden Signature-Interaktionen, die SERO von Collectr/CardLadder erzählbar unterscheiden.
6. **#8 View Transition**, **#10 Share-Karte**, **#11 Widget** — größere Brocken; #11 sinnvoll mit dem Capacitor-Schritt bündeln.

Ein Muster wurde bewusst NICHT aufgenommen: Airbnb-artige Lottie-Feuerwerke über die ganze App. SEROs Zurückhaltung (ein Spring, Studio White, keine Ausrufezeichen) ist die richtige Entscheidung — die einzige Stelle, die mehr Feier verträgt, ist `celebrate` (`sero.js:382–401`): „Live auf eBay" ist der Peak des gesamten Produktversprechens, zeigt den Preis aber statisch (`party-price`, `390`). Dort den vorhandenen `countUp`-Odometer plus die Meilenstein-Haptik (`vibrate([10, 60, 18])`, `263`) einsetzen — Peak-End-Regel, mit Bordmitteln.