# eBay Sold-Daten 2026 — belastbare Lage, Alternativen, Risiken

**Kernaussage vorab:** Sven irrt. Es gibt 2026 **keine normale eBay-API für verkaufte Artikel**. Der einzige offizielle Weg ist zugangsbeschränkt — und **Svens Antrag wurde bereits abgelehnt**. Das steht dokumentiert im eigenen Code.

---

## 0. Bereits im Projekt dokumentiert (nicht Vermutung, sondern Fakt)

`/Users/smorty/ebay-bot/web/ebay_insights.py:1-13` — Modul-Docstring:

> „STAND 03.08.2026: eBay hat Svens Antrag ABGELEHNT. Wortlaut des Developer Support: „Access to this API is highly limited and generally reserved for eBay's approved partners only." Der OAuth-Scope `buy.marketplace.insights` ist der App nicht gewährt — schon die Token-Anfrage scheitert mit `invalid_scope`, nicht erst der Aufruf."

Damit ist Frage (2) für Svens konkreten Fall bereits empirisch beantwortet. Alles Folgende ordnet das ein und prüft, was sonst bleibt.

---

## 1. Finding API / `findCompletedItems` — endgültig tot

| Ereignis | Datum | Beleg |
|---|---|---|
| `findCompletedItems` deprecated + Zugang gesperrt | **15.10.2020** | [Collectors Universe: „eBay killing API Access for Sales History on 10/15"](https://forums.collectors.com/discussion/1045021/ebay-killing-api-access-for-sales-history-on-10-15), [ebaysdk-python #334](https://github.com/timotheus/ebaysdk-python/issues/334) |
| Finding API + Shopping API deprecated | **04.01.2024** | [eBay Community Alert](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062) |
| Finding API + Shopping API **decommissioned** | **05.02.2025** | dito — Originalzitat: „The traditional Finding API and Shopping API are now deprecated as of 2024/01/04. They will be decommissioned on 2025/02/05." |

**Bewertung:** Die Endpunkte existieren seit über einem Jahr nicht mehr. Jede Anleitung im Netz, die `findCompletedItems` empfiehlt, ist veraltet. eBay verweist als Nachfolger auf die **Browse API** — die aber **keine verkauften Artikel** liefert (siehe §3).

---

## 2. Marketplace Insights API — offiziell, aber faktisch geschlossen

**Status:** `v1_beta` (seit Jahren Beta), **Limited Release**. Eigene eBay-Formulierung: „available only to select developers approved by business units"; die Doku sagt aktuell, die API sei „restricted and not open to new users at this time".

**Datenumfang (was sie liefern *würde*):**
- Endpunkt: `GET /buy/marketplace_insights/v1_beta/item_sales/search`
- Suche nach Keyword, GTIN, Kategorie, Produkt
- **Nur 90 Tage rückwirkend** — das ist die harte Grenze
- ca. 8 Marktplätze (u. a. EBAY_DE)
- Felder u. a. `lastSoldPrice`, `lastSoldDate`, `itemId`, `condition`, Titel, Kategorie

**Antragsverfahren & Erfolgsaussicht:** Antrag über eBay Developer Support / Developer Portal mit Business-Case; historisch zusätzlich ein „Application Growth Check". Community-Berichte durchweg negativ: „We were denied access. They said they no longer provide access besides for major partner or something along those lines." ([eBay Community: Marketplace Insights API access](https://community.ebay.com/t5/eBay-APIs-Talk-to-your-fellow/Marketplace-Insights-API-access/td-p/34838736)). Kosten sind nicht öffentlich, weil kein offenes Programm existiert.

**Für Sven:** bereits abgelehnt (§0). Ein zweiter Antrag ist nur sinnvoll, wenn sich die Ausgangslage ändert (EPN-Publisher, echtes Volumen, Vertrag). Als Grundlage für einen Launch **nicht einplanbar**.

> ⚠️ Ich konnte `developer.ebay.com` nicht direkt abrufen (durchgehende Timeouts/403 gegen automatisierte Requests). Die Angaben zu Feldern und Marktplatzliste stammen aus Suchergebnis-Snippets der eBay-Doku und einem Doku-Spiegel, **nicht** aus einem von mir selbst gelesenen Originaldokument — als *unbestätigt im Detail* zu behandeln. Der Limited-Release-Status und das 90-Tage-Fenster sind dagegen mehrfach unabhängig belegt.

---

## 3. Alternativen im offiziellen eBay-Programm — alle liefern AKTIVE Angebote

| API | Liefert | Zugang | Sold-Daten? |
|---|---|---|---|
| **Browse API** | aktive Angebote, Suche, Item-Details | offen (Sven nutzt sie: `/Users/smorty/ebay-bot/bot/ebay/browse.py`) | **Nein.** eBay-Community: „If you are looking for sold items by other sellers, currently not possible through the Browse API… it became 'Restricted' data." |
| **Feed API** | `feed_scope=NEWLY_LISTED` (täglich) bzw. `ALL_ACTIVE` (wöchentlich) | **Limited Release**, erfordert **EPN-Publisher-Konto + EDP-Konto + Vertrag + „rigorous business case review"** | **Nein** — schon die Scope-Namen sagen *active* |
| **eBay Partner Network (EPN)** | Affiliate-Programm + Data Feeds für aktive Listings | Bewerbung, Prüfung | **Nein** |
| **Terapeak / Product Research** | 3 Jahre aggregierte Verkaufsstatistiken | **nur Seller-Hub-UI**, Store-Abo nötig | Ja, aber **es gibt keine Terapeak-API** — eBay-Antwort: „At this time there is not a Terapeak API" |
| **Sell APIs (Fulfillment/Analytics)** | **eigene** Verkäufe | OAuth mit Seller-Scope | Ja — aber nur Svens eigene Orders, kein Marktbild. (Aktuell fehlt der Scope: 403.) |

**Fazit §3:** Innerhalb des offiziellen Programms gibt es für fremde Sold-Daten **exakt einen** Weg — Marketplace Insights. Sonst nichts. Der „Nachfolger" Browse API ersetzt `findCompletedItems` inhaltlich nicht.

Quellen: [Buy APIs Requirements](https://developer.ebay.com/api-docs/buy/static/buy-requirements.html), [Feed API Overview](https://developer.ebay.com/api-docs/buy/feed/static/overview.html), [EPN Data Feeds](https://partnernetwork.ebay.com/solutions/scale-listings-on-your-site-with-data-feeds), [eBay Community: Question about API for sold items](https://community.ebay.com/t5/RESTful-Buy-APIs-Browse/Question-about-API-for-sold-items/td-p/35485158/), [eBay Community: Terapeak API Documentation](https://community.ebay.com/t5/Seller-Tools/Terapeak-API-Documentation/td-p/32138281)

---

## 4. Kommerzielle Drittanbieter

| Anbieter | Modell | Preis | Rechtslage / Zuverlässigkeit |
|---|---|---|---|
| **130point** | Web-Tool + Apps, **keine dokumentierte öffentliche API** | kostenlos | Kein Vertrag, keine SLA, keine Lizenz. Siehe §5. |
| **SoldComps** (`sold-comps.com`) | JSON-API, „gleiches 90-Tage-Fenster, 8 Marktplätze, kein Approval", bis 240 Sold-Preise/Call | Free Tier + bezahlte Stufen | **Achtung: Eigenwerbung.** Die „Alternatives"-Seite ist eine Vendor-Landingpage. Woher die Daten kommen, wird nicht offengelegt — wenn es Scraping ist, erbt Sven dasselbe Risiko, nur mit Zwischenhändler. **Unbestätigt.** |
| **Apify-Actors** (mehrere „eBay Sold Listings" Scraper) | Pay-per-Result | variabel | Explizit Scraping. Rechtlich identisches Problem, kommerziell verpackt. Qualität/Verfügbarkeit je Actor schwankend. |
| **WorthPoint** | Consumer-Abo, ~1 Mrd. Datensätze | 28,99–46,99 $/Monat (jährlich 259,99–449,99 $) | **Keine öffentliche Entwickler-API auffindbar.** Für Programmatik ungeeignet; Scraping des Portals wäre erneut ToS-Bruch. |
| **PriceCharting** | offizielle API, Sven ist Kunde | **59 $/Jahr (Legendary)**, **1 Call/Sekunde** | **Siehe §6 — hier liegt das größte akute Problem.** |

Quellen: [SoldComps Alternatives](https://sold-comps.com/alternatives), [Apify eBay Sold Listings](https://apify.com/caffein.dev/ebay-sold-listings), [WorthPoint Preise (Review)](https://www.underpriced.app/blog/worthpoint-review-2026), [PriceCharting API-Doku](https://www.pricecharting.com/api-documentation)

---

## 5. 130point konkret — die heutige Primärquelle

**Was der Code tut** (`/Users/smorty/ebay-bot/web/sold.py`):
- `POST https://back.130point.com/sales/` mit `data={"query":…, "type":"2", "subcat":"-1"}` (Zeile 50-53) — ein **undokumentierter interner Backend-Endpunkt**, keine öffentliche API
- **gefälschter Browser-User-Agent** (Zeile 28-29: Chrome/126 auf macOS)
- HTML-Antwort wird per Regex geparst (`ROW = re.compile(r'<tr id="dRow".*?</tr>')`, Zeile 31)
- Globale Drossel `MIN_GAP = 8.0` Sekunden **für den gesamten Server** (Zeile 37), 10-Minuten-Cooldown nach HTTP 429 (Zeile 64-67)

**Recherche-Ergebnis zu (5):**
- **Offizielle API: nein.** Keine Entwicklerdokumentation, kein API-Portal, keine Registrierung auffindbar.
- **Dokumentierte Nutzungsbedingungen: keine gefunden.** Weder `130point.com/` noch `130point.com/about` waren automatisiert abrufbar — beide antworten mit **HTTP 403 auf nicht-Browser-Requests**. Das ist selbst ein Befund: die Seite wehrt automatisierten Zugriff aktiv ab. Genau deshalb muss `sold.py` den User-Agent fälschen.
- **Rate Limits: undokumentiert, aber real** — der eigene Code beweist 429-Antworten (Zeile 64).
- **Eigenbeschreibung** ([130point/about](https://130point.com/about)): aggregiert Sold- und Live-Listings von eBay, Fanatics Collect, Goldin, MySlabs, Pristine, Heritage; Kombination aus eBay-APIs und „smart scraping" der Listing-Beschreibungen (u. a. um akzeptierte Best-Offer-Preise zu rekonstruieren, die eBay bewusst verbirgt).

**Der letzte Punkt ist zentral:** 130point ist selbst ein Aggregator, der teils scrapt. Sven scrapt also einen Scraper — zwei ungesicherte Glieder in einer Kette, an der die Kernfunktion seines Produkts hängt.

---

## 6. 🔴 Akuter Befund: PriceCharting-Lizenz wird heute verletzt

PriceCharting ist laut `/Users/smorty/ebay-bot/web/pricecharting.py:1` „Svens Hauptpreisquelle (Legendary Sub)". Die [PriceCharting Terms of Service](https://www.pricecharting.com/page/terms-of-service) sagen:

> „Price Data cannot be used in any software, application, or system that is accessible to third parties, including customers, clients, or the general public, without express written permission."

Nutzung ist auf **„Internal Business Purposes"** beschränkt — definiert als Nutzung durch den Abonnenten und seine Mitarbeiter/Auftragnehmer **innerhalb** der Organisation, ausdrücklich **nicht zur externen Anzeige oder Weiterverbreitung**.

**SERO ist per Definition genau das Verbotene:** eine App für zehntausende fremde Nutzer, die PriceCharting-Preise anzeigt. Das ist kein Graubereich — das ist der wörtlich ausgeschlossene Fall. Ohne schriftliche Sondererlaubnis von PriceCharting ist der Launch ein Vertragsbruch ab Nutzer 1.

Dazu die Kapazität: **1 Call/Sekunde**, bei Überschreitung „your calls will be blocked and your account permissions revoked if it persists". Für ein 59-$/Jahr-Abo ist das konsistent — es ist ein Einzelnutzer-Produkt, kein Backend für eine Consumer-App.

**Handlungsbedarf:** Sven sollte PriceCharting **vor dem Launch** anschreiben und eine kommerzielle Redistributions-Lizenz erfragen. Das ist ein normaler Geschäftsvorgang und vermutlich der billigste Fix im ganzen Bericht.

---

## 7. 🔴 eBay User Agreement — verschärft zum 20.02.2026

Aktueller Wortlaut ([eBay User Agreement](https://www.ebay.de/help/policies/member-behaviour-policies/user-agreement?id=4259), Update wirksam **20.02.2026**):

> „use any robot, spider, scraper, data mining tools, data gathering and extraction tools, or other automated means **(including, without limitation buy-for-me agents, LLM-driven bots, or any end-to-end flow that attempts to place orders without human review)** to access our Services for any purpose, except with the prior express permission of eBay"

Belege: [Value Added Resource, Feb 2026](https://www.valueaddedresource.net/ebay-bans-ai-agents-updates-arbitration-user-agreement-feb-2026/), [The Register](https://www.theregister.com/2026/01/22/ebay_updates_legalese_to_ban/), [PYMNTS](https://www.pymnts.com/artificial-intelligence-2/2026/ebay-blocks-use-of-ai-buy-for-me-agents/)

**Zwei Treffer für SERO:**
1. Automatisiertes Abgreifen von eBay-Daten ohne Erlaubnis — auch mittelbar über einen Aggregator.
2. Der eingeklammerte Zusatz zielt direkt auf **LLM-gesteuerte Bots** und **End-to-End-Flows ohne menschliche Prüfung**. SERO ist eine KI-App, die auf einen Tipp ein fertiges Listing erzeugt. Der Einstellvorgang läuft über die legitime Sell-API und ist damit anders zu bewerten als „buy-for-me" — aber Sven sollte sich klarmachen, dass eBay 2026 genau diese Produktgattung im Blick hat.

**Das schärfste Risiko ist nicht die App, sondern Svens Existenz:** Er ist gewerblicher eBay-Händler („seromunich"). Ein Verstoß gegen das User Agreement gefährdet nicht bloß ein Feature — eBay kann Verkäuferkonten sperren. Die Sold-Daten-Frage hängt damit an seinem Haupteinkommen.

---

## 8. Rechtslage DE/EU beim Scraping — nüchtern

- **§ 87b UrhG (Datenbankherstellerrecht):** Unwesentliche Teile einer öffentlichen Datenbank dürfen entnommen werden; **wiederholte und systematische** Entnahme unwesentlicher Teile mit kumulativer Wirkung ist aber untersagt. Ein Dauerbetrieb mit zehntausenden Nutzern ist per Definition wiederholt und systematisch. ([§ 87b UrhG](https://www.gesetze-im-internet.de/urhg/__87b.html), [WBS Legal: Ist Screen Scraping legal?](https://www.wbs.legal/urheberrecht/ist-screen-scraping-legal-15081/))
- **AGB-Verstoß:** Einseitige Website-Hinweise reichen nach OLG-Frankfurt-Linie nicht immer; ein **akzeptiertes** Vertragsverhältnis (Svens eBay-Händlerkonto, sein PriceCharting-Abo) ist dagegen bindend. Genau hier liegt Sven falsch — er hat beide Verträge unterschrieben.
- **UWG:** Umgehung technischer Schutzmaßnahmen (UA-Spoofing gegen einen 403-Filter) kann als wettbewerbswidrige Behinderung gewertet werden.
- **Hinweis:** keine Rechtsberatung. Vor Launch mit zehntausenden Nutzern gehört das zu einem IT-Rechtsanwalt — die Kosten sind marginal gegen das Risiko.

---

## 9. Technische Skalierungs-Rechnung — der Weg trägt nicht

Aus `sold.py` direkt ableitbar:

| Größe | Wert | Quelle |
|---|---|---|
| Globaler Mindestabstand 130point | 8,0 s (**serverweit**, ein `asyncio.Lock`) | `sold.py:35-37` |
| **Theoretisches Maximum** | **10.800 Anfragen/Tag** | 86.400 / 8 |
| Cache-TTL | 12 h | `sold.py:22` |
| PriceCharting-Limit | 1/s = 86.400/Tag | ToS |
| Cooldown nach 429 | 600 s Totalausfall der Sold-Quelle | `sold.py:66` |

Bei zehntausenden Nutzern mit je mehreren Scans pro Tag ist die 130point-Quelle **um ein bis zwei Größenordnungen zu klein**. Der 12-h-Cache hilft nur bei Karten, die viele Nutzer gleichzeitig scannen — im Long-Tail der Sammlerware (genau Svens Markt) greift er kaum. Praktisch heißt das: die Warteschlange läuft voll, Scans fallen auf die KI-Schätzung zurück, und der Nutzer bekommt eine geratene Zahl, die aussieht wie ein Marktwert.

**Genau das ist die Wurzel des One-Piece-Fehlers vom 03.08.** — nicht nur der PriceCharting-Fehlmatch (Sammelkarte „Nami [Manga] OP01-016" statt Buch) und die grobe Rasterung (nur 9/9.5/10, deshalb 9.0 und 9.4 im selben Feld), sondern die Tatsache, dass bei fehlendem Beleg still eine Schätzung ausgegeben wird. Bei 10.000 Nutzern wird aus einem Einzelfall ein systematischer Fehler mit Haftungspotenzial: Nutzer treffen auf Basis dieser Zahlen Kauf- und Verkaufsentscheidungen.

---

## 10. Empfehlung

### Kann 130point Primärquelle für ein Produkt mit zehntausenden Nutzern sein?

**Nein. Eindeutig nicht.** Kein Vertrag, keine SLA, keine dokumentierten Bedingungen, aktive Abwehr automatisierter Zugriffe (403), belegte 429-Sperren, Regex-Parsing auf fremdes HTML, harte Obergrenze ~10.800 Requests/Tag, und ein Rechtsrisiko, das auf Svens Händlerkonto durchschlägt. Als Prototyp-Krücke war das vertretbar. Als Fundament eines Produkts ist es fahrlässig.

### Empfohlene Zielarchitektur

**1. Kurzfristig, vor jedem Launch (Pflicht):**
- **PriceCharting kommerzielle Lizenz anfragen.** Billigster Fix, größter Effekt. Ohne sie ist der Launch heute schon Vertragsbruch.
- **eBay Browse API zur Primärquelle machen** (`/Users/smorty/ebay-bot/bot/ebay/browse.py`). Sie ist offiziell, lizenziert, skaliert und liefert **aktive Angebote** — also Angebotspreise. Für ein Listing-Tool ist die Frage „zu welchem Preis stelle ich ein?" mit Angebotspreisen + Konkurrenzdichte oft besser beantwortet als mit 3 alten Sold-Punkten.
- **Ehrliche UI-Kennzeichnung.** Wenn kein Beleg existiert, muss die App „keine belastbaren Vergleichsdaten" sagen — nicht eine KI-Schätzung als Marktwert ausgeben. Das ist gleichzeitig der Fix für den 603-€-Fehler und die beste Haftungsabsicherung.

**2. Mittelfristig — der eigentlich tragfähige Weg:**
- **Kategoriespezifische lizenzierte Quellen** statt eBay-Sold als Universallösung: **Cardmarket** (EU-Markt, Partnerprogramm — laut Projektnotizen bereits als „nur Partner" identifiziert), **TCGplayer** (US), für Bücher/Manga ein anderer Weg als für Karten. Der One-Piece-Fehler zeigt: eine Quelle für alle Warengruppen ist die eigentliche Schwachstelle.
- **Eigener Datenbestand als Burggraben.** Sven hat etwas, das kein Aggregator hat: Nutzer, die über SERO **einstellen**. Über die Sell-APIs (Fulfillment-Scope nachziehen — bekanntes offenes Todo) kann SERO für jedes über die App erzeugte Listing den **tatsächlichen Ausgang** erfassen: verkauft ja/nein, zu welchem Preis, nach wie vielen Tagen. Das ist first-party, vertraglich sauber, für Svens Nische genauer als jede fremde API — und es wird mit jedem Nutzer besser. **Das ist die einzige Antwort, die bei zehntausenden Nutzern trägt.**

**3. Optional:** EPN-Publisher werden und Marketplace Insights erneut beantragen. Mit echtem Volumen und Partner-Status steigt die Chance. Aber: **nicht einplanen** — die Ablehnung liegt vor, und ein Produkt darf nicht an einer Genehmigung hängen, die eBay nach eigener Aussage „generally reserved for approved partners only" vergibt.

### Schonungslos

Das Produkt steht auf drei Beinen, von denen zwei rechtlich nicht tragen: PriceCharting (ausdrücklich verbotene Nutzungsart) und 130point (undokumentiertes Scraping eines Scrapers mit UA-Spoofing). Das dritte Bein — die eBay Browse API — ist sauber, wird aber nur als Beiwerk genutzt. Sven sollte den Launch nicht daran ausrichten, doch noch eine Sold-API zu bekommen, sondern daran, ein Produkt zu bauen, das ohne fremde Sold-Daten funktioniert und sich seine eigenen aufbaut.

---

### Relevante Dateien
- `/Users/smorty/ebay-bot/web/ebay_insights.py` — dokumentierte eBay-Ablehnung (Z. 1-13), Feature-Flag `SERO_EBAY_INSIGHTS`
- `/Users/smorty/ebay-bot/web/sold.py` — 130point-Anbindung: Endpunkt Z. 50-53, UA-Spoofing Z. 28-29, Drossel Z. 35-37, 429-Cooldown Z. 64-67, Regex-Parser Z. 31
- `/Users/smorty/ebay-bot/web/pricecharting.py` — Legendary-Sub, Grade-Mapping Z. 1-7 (nur 9/9.5/10 → Ursache der 9.0-vs-9.4-Kollision)
- `/Users/smorty/ebay-bot/bot/ebay/browse.py` — die offizielle, lizenzierte Quelle (aktive Angebote)

*Hinweis: strikt read-only gearbeitet — keine Datei geändert, kein Server gestartet, keine Requests gegen localhost.*