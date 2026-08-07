# SERO — Entscheidungsvorlage Preisdaten
**Stand 3. August 2026. Alle Aussagen beruhen auf der gegengeprüften Recherche; wo etwas ungeprüft ist, steht es dabei.**

---

## 1. Die Antwort in drei Sätzen

**Es gibt 2026 keine sofort buchbare Quelle für echte eBay.de-Verkaufspreise, deren Nutzungsbedingungen die Anzeige in einer Abo-App erlauben** — jeder einzelne Kandidat scheitert entweder am Zugang (Marketplace Insights abgelehnt, TCGplayer und Cardmarket nehmen keine Anträge an, CardTrader verlangt für Marktdaten laut eigener ToS eine Einzelfallgenehmigung), an der Weitergabeklausel (PriceCharting untersagt jede Software, die Dritten zugänglich ist; SerpApi räumt überhaupt keine Rechte an den Ergebnisdaten ein) oder an der Datenherkunft (Apify und 130point sind Scraper, das Risiko wird nur verschoben, nicht beseitigt).

Die beste erreichbare Näherung ist ein Dreiklang: **Ximilar** für die Erkennung, **PokemonPriceTracker Business** als einzige sofort buchbare Quelle mit echter Notenabstufung und Cardmarket-EUR, dazu die freien Kataloge für Magic und Yu-Gi-Oh — und darunter **SEROs eigener Verkaufspool** aus den Angeboten, die die App selbst einstellt und deren Abschluss der Nutzer bestätigt. Kosten bei 1.000 Nutzern rund 700 bis 900 Euro im Monat, also 0,70 bis 0,90 Euro je Nutzer.

Der Haken hat zwei Teile: **für gegradete Slabs außerhalb Pokémon, für Retro-Videospiele und für Manga existiert überhaupt keine lizenzierte Preisquelle** — dort bleibt nur die Anzeige aktueller eBay-Angebote als klar getrennter Block, ausdrücklich nicht als berechneter Marktwert. Und **ab etwa 2.000 Nutzern läuft jede antragsfreie Option aus**, dann ist mindestens ein Gespräch unvermeidbar.

---

## 2. Was sofort buchbar ist

### 2.1 Erkennung

| Quelle | Preis | Limit | Weitergabe | Deckt ab |
|---|---|---|---|---|
| **Ximilar Collectibles** | Ab **Business 100K = 59 EUR/Monat** (Free-Tier deckt die Collectibles-Endpunkte laut Doku **nicht** ab). Rohkarte 16 Credits = 15 EUR/1.000, Slab 31 Credits = 24 EUR/1.000 | **Nicht dokumentiert**, dafür Abschaltvorbehalt nach alleinigem Ermessen des Anbieters. Batch 10 Bilder/Request | **Unklar** — kein Verbot, aber auch keine Lizenzklausel. Vor Produktivstart schriftlich bestätigen lassen | TCG (15+ Spiele), Sportkarten, Comics/Manga, Slab-Label-OCR (PSA, BGS, CGC, SGC, ACE, MANA, TAG) mit Grader, Cert-Nummer und Note. **Keine Videospiele** |

Ximilars `slab_id` ist der wichtigste Einzelfund der ganzen Recherche: Er ersetzt vier gescheiterte Grader-Zugänge (PSA-Freitier im Juni 2026 abgeschafft, CGC nur für autorisierte Händler, Beckett und SGC ohne Produkt) durch OCR des Labels auf dem Foto. Grader, Zertifikatsnummer und Note kommen aus dem Bild, nicht aus fremden Systemen.

**Nicht nutzen:** Ximilars eigenen Pricing-Endpunkt (10 Credits). Die dokumentierten Quellmärkte sind US-Marktplätze, ein Währungsfeld existiert in der Doku nicht, Cardmarket und eBay.de kommen nirgends vor. 5,90 EUR pro 1.000 Preisabfragen für unbelegte US-Zahlen ist der schlechteste Deal im ganzen Feld.

### 2.2 Preise

| Quelle | Preis | Limit | Weitergabe | Deckt ab |
|---|---|---|---|---|
| **PokemonPriceTracker Business** | **99 USD/Monat flat** (ToS §6: kommerzielle Nutzung erfordert Business oder Enterprise — nicht die 9,99-Stufe) | 500 Req/Min, 200.000 Credits/Tag, **pro Konto**. Bei realistischer Nutzung dauerhaft unter 30 % ausgelastet, auch bei 50.000 Nutzern | **Ausdrücklich erlaubt**, wörtlich: Speichern, Cachen und Ausliefern an die eigenen Endnutzer — "on the plan appropriate to your use" | Pokémon: roh, **Sealed**, **PSA/CGC/BGS/SGC mit getrennten Preisen je Note**, Cardmarket-EUR (Beta, +1 Credit, nur ab Business) |
| **Scryfall** (Bulk) | 0 EUR | Bulk-Dateien ohne Limit, API 2 Req/s | **Problematisch**: Paywall-Verbot. Scryfall-Daten dürfen nicht hinter einer Abo-Schranke liegen | Magic, Cardmarket-EUR-Preise, täglich |
| **YGOPRODeck** | 0 EUR | 20 Req/s **pro IP**, Überschreitung = 1 Std. Sperre. Lokales Caching vom Betreiber vorgeschrieben | Ungeregelt | Yu-Gi-Oh, `cardmarket_price` in EUR — **aber ausdrücklich der niedrigste Preis über alle Druckvarianten**, also nur eine Untergrenze, nie ein Vorschlagspreis |
| **TCGCSV** | 0 EUR | 1 Vollabzug/24h, max. 10.000 Requests/24h, 100 ms Pause, eigener User-Agent, kein Browser-Zugriff (CORS) | **Keine Lizenz veröffentlicht** — gespiegelte TCGplayer-Daten, Rechtsgrundlage ungeklärt | Sealed-Katalog über viele Spiele, US-Dollar. Keine Zustands- oder Notenabstufung |
| **UPCitemdb PRO** | 699 USD/Monat, **0,16 USD je 1.000 Lookups** (DEV 99 USD für 600.000/Monat reicht bis weit über 10.000 Nutzer) | PRO: 6 Lookups/s dauerhaft, 12 im Burst, 6 parallele Verbindungen. DEV: 1 Lookup / 2 s — für tausende Nutzer unbrauchbar | **Restriktiv formuliert**: "non-transferable … solely for Customer's operations". Vorab freigeben lassen | EAN-Identifikation für Sealed Product und versiegelte Spiele |
| **eBay Browse API** | 0 EUR | **5.000 Calls/Tag für die ganze Anwendung**, nicht pro Nutzer | Anzeige als "Public Display" erlaubt — mit drei harten Auflagen, siehe unten | Alle Warenarten, **nur aktive Angebote**, ebay.de, EUR |
| **eBay Fulfillment getOrders** | 0 EUR | 100.000 Calls/Tag pro Keyset. Keyset erst nach validiertem Account-Deletion-Webhook aktiv | Rückspiegelung an denselben Nutzer: in Ordnung. **Aggregation über Nutzer hinweg: verboten** (ALA §8.1(d), gilt ausdrücklich auch für die eigenen Daten der Nutzer) | Echte Verkäufe des jeweils verbundenen Nutzers, bis 2 Jahre zurück |
| **Eigener Verkaufspool** | 0 EUR Datenkosten | Keines | **Erlaubt** — die Daten gehören SERO | Alle Warenarten, EUR, mit exakt bekannter Note und Zustand. Kaltstart bei null |

### 2.3 Die drei eBay-Auflagen, die über allem stehen

Diese drei Klauseln entscheiden die Architektur, nicht die Preisliste:

1. **ALA §8.1(d)** — die Ableitung eines "Average selling price für eine eBay-Kategorie" erfordert eBays ausdrückliche vorherige schriftliche Erlaubnis. **Konsequenz: Aus eBay-Daten darf keine angezeigte Marktwertzahl berechnet werden.** Nicht aus Browse, nicht aus getOrders.
2. **ALA §8.1(b)** — eBay-Inhalte dürfen in der Anzeige nicht mit Nicht-eBay-Inhalten vermischt werden, und angezeigte Angebotsdaten dürfen nicht älter als 6 Stunden sein. **Konsequenz: eBay-Angebote sind ein eigener, optisch getrennter Block, kein Bestandteil der Preisberechnung.**
3. **ALA §9** — Preismodellierung mit eBay Content "alone **or in combination with third-party information**". **Konsequenz: Die Trennung der Datenpfade muss vollständig sein.** Ein einziges eBay-Signal im Bewertungsmodell zieht das ganze Modell unter die Klausel.

Das ist der juristische Trennstrich, an dem sich das Produkt ausrichtet: **Die Zahl kommt aus lizenzierten Preis-APIs und aus SEROs eigenem Pool. eBay liefert Kontext, nie die Zahl.**

---

## 3. Was ausscheidet und warum

Damit diese Wege nicht in einem halben Jahr erneut geprüft werden:

| Quelle | Ausschlussgrund |
|---|---|
| eBay Marketplace Insights | Limited Release, Antrag abgelehnt. Selbst **mit** Zugang wäre ein Preis-Tool nur mit zweiter, gesonderter schriftlicher Zustimmung erlaubt (ALA §8.5). Doppelt zu. eBay besitzt seit 2022 TCGplayer — der Interessenkonflikt ist strukturell |
| TCGplayer API | "We are no longer granting new API access at this time" — eigene Doku. Seit der eBay-Übernahme geschlossen |
| Cardmarket API | "Currently, we are not accepting applications" — eigene Hilfeseite. Zugangsdaten Dritter zu nutzen ist ausdrücklich untersagt. Download-Dateien nur mit eingeloggter Browser-Session, also wieder Scraping-nah |
| CardTrader | ToS trennt sauber: antragsfrei nur für **eigene** Bestandsverwaltung. "We can provide market APIs **upon request** … our staff will evaluate each case". Genehmigungsverfahren. Zusätzlich: nur die 25 billigsten Produkte je Blueprint, also nach unten verzerrt |
| PriceCharting / SportsCardsPro / VGPC | Inhaltlich die beste Einzelquelle für den Warenmix, aber ToS: "Price Data cannot be used in any software, application, or system that is accessible to third parties". Plus 1 Call/s fürs ganze Konto. Dieselbe Firma dreimal. **Nicht anfragen** |
| Collectr | API-Terms verbieten Produkte, die Collectrs eigenen oder **geplanten** Angeboten ähneln. SERO wäre nach dieser Klausel ein Konkurrenzprodukt |
| GoCollect, GemRate, Card Ladder | Antragsformular bzw. Demo-Termin, Preise nicht einmal öffentlich. GemRate ist der einzige legitime Weg zu graderübergreifenden Population-Daten — kostet aber ein Vertriebsgespräch |
| PSA, CGC, Beckett, SGC direkt | PSA-Freitier im Juni 2026 auf ~1 Call/Tag reduziert, bezahlte Stufe nur per E-Mail; Collectors-Lizenz erlaubt nur "personal and non-commercial use". CGC nur für autorisierte Händler (Auskunft im eigenen Forum). Beckett und SGC haben schlicht kein Produkt. **Ersetzt durch Ximilar slab_id** |
| SerpApi | Sold-Filter und ebay.de dokumentiert, Zugang und Preis sauber — aber: **keinerlei Rechteeinräumung an den Ergebnisdaten** in den ToS, Freistellungspflicht gegen SerpApi, kein Verkaufsdatum im Antwortschema (Median ohne Recency), zwei seit 23.07.2026 offene eigene Bug-Tickets nach eBays Login-Schranke, Selbstbedienung endet bei 30.000 Suchen/Monat. Der US Legal Shield greift ausdrücklich nur vor US-Gerichten — für München wertlos |
| Apify-Actor "eBay Sold Listings" | Lebt und liefert, aber: Community-Actor eines Einzelentwicklers, Apify stellt vertraglich ausdrücklich klar, dass Actors "not part of the Services" sind und "AS IS" laufen. Preis nicht mehr öffentlich beziffert. Rechtlich identisch mit 130point |
| Oxylabs | Verpflichtende KYC-Prüfung mit dokumentierter Ablehnungsquote von 25 % |
| Bright Data, Zyte, ScraperAPI, ScrapingBee, Decodo, Nimble, Firecrawl | Keine Verkaufsdaten. Man mietet eine fremde IP für dieselbe Scraping-Logik. Man kauft Tarnung, keine Legalität |
| Google Custom Search | Für Neukunden geschlossen, Bestandskunden bis 01.01.2027 |
| Bing Search API | Am 11.08.2025 endgültig abgeschaltet. Falls im Code noch als Fallback: streichen |
| Gemini Grounding | ToS verbieten wörtlich Modifizieren, Vermengen mit eigenem Inhalt und Cachen der Grounded Results — also exakt das Produktkonzept |
| Alle Web- und KI-Suchen (Brave, Tavily, Exa, Perplexity, You.com, OpenAI, Claude Web Search) | Struktureller Fehlschlag: eBay-Verkaufsseiten stehen nicht im Suchindex. Snippets sind keine Preise. Als **Erkennungs- und Anreicherungsschicht** dagegen brauchbar |
| Google Vision Product Search | Von Google selbst als "in maintenance mode" gekennzeichnet, letzte Änderung März 2023, der empfohlene Nachfolger Vertex AI Vision erreicht am 30.09.2026 End of Life |
| AWS Rekognition Custom Labels | Abrechnung nach Laufzeit: rund 2.880 USD/Monat für ein dauerhaft erreichbares Modell. Falsche Größenordnung, und für hunderttausende Kartenvarianten die falsche Bauform |
| pokemontcg.io | Heute gemessen 5 von 10 Aufrufen mit HTTP 500/502. Cardmarket-Preise waren 33 Tage alt. Projekt ist in Scrydex aufgegangen (kostenpflichtig ab 29 USD) |
| JustTCG | Rechtlich der sauberste Anbieter (Endnutzer-Anzeige ausdrücklich in §7.1 erlaubt), aber: **"Right now NA is the only region with data"** — keine EU-Preise, keine EUR. Free-Tier für kommerzielle Nutzung vertraglich gesperrt. Zusätzlich nach eigener Aussage selbst ein Scraper. Als Option vormerken, EU-Region beobachten |
| Whatnot, Heritage, Goldin, Fanatics Collect, Catawiki, GameValueNow | Kein Zugang, kein Portal, teils Domain nicht erreichbar |

---

## 4. Die Architektur

### 4.1 Grundprinzip: zwei getrennte Datenpfade

```
PFAD A — DIE ZAHL (der angezeigte Marktwert)
  speist sich AUSSCHLIESSLICH aus:
  · lizenzierten Preis-APIs (PokemonPriceTracker, freie Kataloge)
  · SEROs eigenem Verkaufspool
  Kein eBay-Datenpunkt fließt hier ein. Niemals.

PFAD B — DER KONTEXT (getrennter UI-Block)
  · eBay Browse: "Aktuell auf eBay.de angeboten: 4 Stück, 39–58 EUR"
  · eBay getOrders: "Deine eigenen Verkäufe dieses Artikels: 2 × 47 EUR"
  Nicht mit Pfad A vermischt, max. 6 Stunden alt, wird gelöscht,
  sobald das Angebot auf eBay verschwunden ist.
```

Diese Trennung ist kein Formalismus. Sie ist der Unterschied zwischen "eine App, die eBay-Angebote anzeigt" (ausdrücklich erlaubt) und "ein Preistool auf Basis von eBay-Daten" (nur mit schriftlicher Erlaubnis).

### 4.2 Routing nach Warenart

```
FOTO
 └─ Warentyp: vom Nutzer per Tab vorgewählt (Karte / Slab / Sealed / Spiel / Comic)
    → spart Ximilars detect-Aufruf: 6 von 16,6 Credits, also 36 % Erkennungskosten

 ├─ ROHE KARTE  (60 % der Scans)  →  Ximilar tcg_id (10 Cr)
 │   ├─ Pokémon (60 % davon) → PokemonPriceTracker, Cardmarket-EUR
 │   ├─ Magic                → Scryfall Bulk, Feld eur / eur_foil
 │   ├─ Yu-Gi-Oh             → YGOPRODeck cardmarket_price, NUR als Untergrenze
 │   ├─ One Piece / Lorcana  → LÜCKE. Eigener Pool, sonst Pfad B
 │   └─ Rückfall             → Eigener Pool → Pfad B → "zu wenig Daten"
 │
 ├─ SLAB  (20 %)  →  Ximilar slab_id (15 Cr) → Grader + Cert + Note
 │   ├─ Pokémon-Slab → PokemonPriceTracker, getrennter Preis je PSA/CGC/BGS/SGC-Note
 │   └─ ALLE ANDEREN → LÜCKE. Eigener Pool → Pfad B
 │
 ├─ SEALED  (12 %)  →  EAN über UPCitemdb (kein Ximilar-Endpunkt nötig)
 │   ├─ Pokémon-Sealed → PokemonPriceTracker /sealed-products
 │   ├─ andere TCG     → TCGCSV als US-Referenz, mit Währungshinweis
 │   └─ Rückfall       → Eigener Pool → Pfad B
 │
 ├─ RETRO-SPIEL  (5 %)
 │   ├─ versiegelt/lose → EAN über UPCitemdb
 │   ├─ gegradet (WATA/VGA/CGA) → Ximilar card_ocr_id (15 Cr) auf das Label
 │   └─ Preis: VOLLSTÄNDIGE LÜCKE. Nur eigener Pool → Pfad B
 │
 └─ MANGA / COMIC  (3 %)  →  Ximilar comics_id (10 Cr), ggf. + slab_id
     └─ Preis: VOLLSTÄNDIGE LÜCKE. Nur eigener Pool → Pfad B
```

### 4.3 Die Lücke gezielt schließen: Slabs, Retro, Manga

Das sind 28 Prozent der Scans, aber der größte Teil des Warenwerts. Für sie existiert **keine** lizenzierte, sofort buchbare Preisquelle. Der einzige Weg, der nicht in einer Sackgasse endet, ist der eigene Pool — und der lässt sich beschleunigen:

**Baustein 1 — Erfassung ab Tag 1, auch ohne Nutzer.**
Der Betreiber ist selbst gewerblicher Händler mit laufendem Bestand. Jeder eigene Slab-, Retro- und Comic-Verkauf ist ein Datenpunkt. Bei einigen hundert Verkäufen im Jahr ist das der Grundstock. Terapeak Product Research im Seller Hub — kostenlos, lizenziert, drei Jahre Historie — liefert dafür die **manuelle Kalibrierungsreferenz**. Nicht automatisiert auslesen (das wäre wieder Scraping aus dem eingeloggten Bereich), sondern stichprobenartig von Hand gegenprüfen.

**Baustein 2 — Der Slab-Schlüssel ist der Hebel.**
Ein Slab ist über `Karte × Auflage × Grader × Note` eindeutig. Ximilars `slab_id` liefert genau diese vier Felder aus dem Foto. Damit hat SERO etwas, das **keine externe Quelle hat**: exakt bekannte Note ohne Rateanteil. Ein Pool mit 50 Datenpunkten und exakter Note schlägt eine externe Quelle mit 500 Datenpunkten unbekannter Zustandsverteilung.

**Baustein 3 — Ehrliche Kennzeichnung statt Scheingenauigkeit.**
```
n ≥ 8 unabhängige Verkäufe  →  "Marktwert 187 EUR"      (Median)
n = 3–7                     →  "Spanne 160–210 EUR, Basis 4 Verkäufe"
n = 1–2                     →  Einzelverkäufe zeigen, keine Aggregation
n = 0                       →  keine Zahl. Pfad B als getrennter Block.
```
Die Schwelle bei 8 statt 5: Bei Slabs mit eindeutiger Zertifikatsnummer ist das Objekt selbst identifizierend, eine niedrige k-Schwelle schützt datenschutzrechtlich nicht. Im Pool wird deshalb auf `Kartenmodell + Note` vergröbert, **die Zertifikatsnummer wird nie gespeichert**.

**Baustein 4 — Das Datenschutz-Gerüst.**
Eigener, granularer Einwilligungsschalter für das Poolen ("Meine Verkäufe helfen anderen Sammlern"), nicht in den AGB, nicht Bedingung fürs Abo (Kopplungsverbot Art. 7 Abs. 4 DSGVO), jederzeit widerruflich mit Wirkung für die Zukunft. Käufernamen, Adressen und E-Mails werden schon beim Einlesen verworfen. Das ist beherrschbar; die DSGVO ist hier nicht das Hindernis.

**Baustein 5 — Der Preisfakt darf nicht von eBay kommen.**
Das ist die entscheidende technische Bedingung. Der Verkaufspreis im Pool stammt aus dem Angebot, das **SERO selbst eingestellt hat**, plus der Bestätigung des Nutzers in der App. Er wird **nicht** aus `getOrders` geholt. Sobald der Preisfakt aus der eBay-API kommt, ist er eBay Content, und §8.1(d) greift wieder. `getOrders` bleibt ausschließlich für die persönliche Portfolio-Ansicht des jeweiligen Nutzers reserviert.

Ehrlich dazu: Das ist eine juristische Konstruktion ohne Rechtsprechung. Sie stützt sich auf die ALA-Definition selbst ("eBay Content does not include information that you lawfully obtain independent of eBay"), sie ist vertretbar, aber sie ist ungetestet. Sie hält nur, wenn die Trennung technisch sauber durchgezogen wird.

---

## 5. Der Übergang von 130point

Kein Bruch, sondern fünf Stufen. Die Preisqualität bricht nicht ein, weil 130point erst abgeschaltet wird, wenn es messbar überflüssig ist.

### Stufe 0 — Sofort, diese Woche: das größte Einzelrisiko entschärfen

Der gefälschte User-Agent ist der gefährlichste Punkt im ganzen System — nicht wegen eBay, sondern wegen der BGH-Grenze aus *Ryanair/Cheaptickets* (I ZR 224/12): Screen Scraping ist in Deutschland nicht per se wettbewerbswidrig, aber das **Überwinden technischer Schutzvorrichtungen** ist genau die Fallgruppe, die der BGH als möglicherweise unlauter offengelassen hat. Cloudflare-Schutz plus gefälschte Kennung ist wortwörtlich diese Fallgruppe.

Maßnahme: ehrlicher User-Agent mit Produktname und Kontaktadresse. Wenn 130point daraufhin blockt, weiß man, woran man ist — und hat eine Antwort, die vor Gericht trägt. Wenn nicht, ist der schwerste Vorwurf entfallen.

Zweite Maßnahme: 130point-Aufrufe laufen über eine Infrastruktur, die von nichts abhängt, was mit dem gewerblichen eBay-Händlerkonto verknüpft ist.

### Stufe 1 — Woche 1 bis 4: Erfassung anschalten, bevor irgendetwas abgeschaltet wird

Jedes über SERO eingestellte Angebot erzeugt einen Pool-Kandidaten mit vollem Schlüssel (Artikel, Auflage, Grader, Note, Zustand, Startpreis, Sofortkaufpreis). Verkaufsbestätigung in der App. Der Pool füllt sich, während 130point unverändert weiterläuft. Kosten: null.

### Stufe 2 — Monat 1 bis 2: Pokémon abkoppeln

PokemonPriceTracker Business buchen (99 USD). Pokémon ist rund 60 Prozent der rohen Karten und der größte Teil der Slabs — also mit **einem** Vertrag zu 99 USD flat der größte Einzelblock des gesamten Volumens, inklusive Notenabstufung und Sealed. Für Magic und Yu-Gi-Oh die Bulk-Abzüge einrichten.

Nach Stufe 2 läuft 130point nur noch für Slabs außerhalb Pokémon, Retro und Comics. Das Abfragevolumen sinkt sofort um rund 70 Prozent, und damit auch die Blockadewahrscheinlichkeit.

### Stufe 3 — Monat 2 bis 6: 130point in den Schattenbetrieb

130point wird **nicht mehr angezeigt**. Es läuft nur noch als stiller Vergleichswert mit, um das eigene Modell zu kalibrieren. Der Nutzer sieht ab jetzt nur noch Pfad-A-Zahlen und Pfad-B-Kontext. Rechtlich ist das ein qualitativer Sprung: Die fremden Daten werden nicht mehr an Dritte weitergegeben, sondern nur intern zur Qualitätsprüfung verwendet.

Messgröße pro Kategorie:
```
Abweichung = |Median(eigener Pool) − Median(130point)| / Median(130point)

Abschaltkriterium je Kategorie:
  Abweichung < 10 %  bei mindestens 80 % der Scans dieser Kategorie
  UND n ≥ 8 im eigenen Pool bei mindestens 60 % der Scans
```

### Stufe 4 — sobald das Kriterium erfüllt ist: Kategorie für Kategorie abschalten

Nicht alles auf einmal. Wahrscheinliche Reihenfolge: Pokémon-Slabs zuerst (durch PokemonPriceTracker ohnehin abgedeckt), dann Sealed, dann die volumenstarken Nicht-Pokémon-Slabs, zuletzt Retro und Comics — die brauchen am längsten, weil sie die dünnsten Kategorien sind.

### Stufe 5 — Der Endzustand, realistisch beschrieben

Für Retro-Spiele und Manga wird der eigene Pool auf Jahre dünn bleiben. Dort ist der ehrliche Endzustand: keine Marktwertzahl, sondern die aktuellen eBay-Angebote als getrennter Block plus die Aufforderung an den Nutzer, den erzielten Preis zurückzumelden. Das ist unbefriedigend, aber es ist die Wahrheit über die Datenlage — und es ist besser als eine erfundene Zahl aus einer Quelle, die den Betreiber sein Händlerkonto kosten kann.

Falls für diese beiden Kategorien doch eine Zahl nötig wird: Der einzige verbleibende Weg ist ein Gespräch mit GemRate (Population-Daten, kein Preis) oder eine schriftliche Anfrage bei eBay für §8.1(d). Beides widerspricht der Vorgabe "ohne noch mal irgendwas anzufragen" — das ist die Konsequenz, nicht meine Empfehlung.

---

## 6. Was er heute tun kann

1. **130point-Client umstellen**: echter User-Agent mit Produktname und Kontaktadresse, keine gefälschte Kennung mehr. 15 Minuten Arbeit, eliminiert den schwersten Rechtsvorwurf.
2. **Bing aus dem Code und aus den Notizen streichen**, falls noch irgendwo als Fallback hinterlegt. Der Dienst ist seit dem 11.08.2025 abgeschaltet.
3. **Pool-Erfassung bauen**: Bei jedem über SERO eingestellten Angebot einen Datensatz anlegen (Artikel, Auflage, Grader, Note, Zustand, Preise, Datum). Verkaufsbestätigung als Ein-Klick-Aktion in der App. Zertifikatsnummer nicht speichern.
4. **Zwei getrennte Datenpfade im Code anlegen**: Pfad A (Bewertung) darf strukturell nicht auf eBay-Daten zugreifen. Das ist eine Architekturgrenze, keine Konvention — am besten getrennte Module, getrennte Datenbanktabellen.
5. **Ximilar Business 100K buchen** (59 EUR) und mit 30 bis 50 echten Artikeln aus dem eigenen Bestand testen: TCG, Slabs aller vier Grader, ein Manga, ein gegradetes Spiel. Prüfen, ob `slab_id` die Note zuverlässig liest.
6. **Bei Ximilar schriftlich anfragen** (care@ximilar.com), zwei Sätze: (a) Requests pro Sekunde und Parallelität, (b) ausdrückliche Freigabe, Erkennungsergebnisse in einer Endkunden-App anzuzeigen. Das ist eine Rückfrage an einen bereits bezahlten Anbieter, kein Antragsverfahren.
7. **Bei PokemonPriceTracker schriftlich klären lassen**, welche Fassung gilt: Preisseite ("Commercial use license" bei allen Tarifen) oder ToS §6 ("requires an active Business or Enterprise subscription"). Unterschied: 9,99 gegen 99 USD im Monat. Danach Business buchen.
8. **Nutzertyp-Vorauswahl in die UI**: Der Nutzer tippt auf Karte / Slab / Sealed / Spiel / Comic, bevor er fotografiert. Spart Ximilars `detect`-Aufruf, also 36 Prozent der Erkennungskosten. Bei 1.000 Nutzern rund 180 EUR im Monat.
9. **Scryfall- und YGOPRODeck-Bulk einrichten** als tägliche Cronjobs. YGOPRODeck-Preise ausschließlich als Untergrenze verwenden, nie als Vorschlagspreis — das Feld ist der niedrigste Preis über alle Druckvarianten.
10. **UPCitemdb im Free-Tier testen** (keine Anmeldung nötig) mit 20 EANs von Displays und versiegelten Spielen aus dem eigenen Lager. Erst bei belegter Trefferquote DEV buchen.
11. **eBay Browse als Pfad-B-Block einbauen**, mit den drei Auflagen fest verdrahtet: getrennter UI-Bereich, maximal 6 Stunden Cache, Löschung bei Angebotsende. Keine Berechnung darauf.
12. **Web-Checkout über Stripe prüfen.** Bei rund 10.700 zahlenden Nutzern springt Apple von 15 auf 30 Prozent Provision — das kostet 12.600 EUR im Monat, mehr als das Doppelte aller Datenquellen zusammen. Das ist der größte Hebel im ganzen Kostenmodell, wichtiger als jede API-Verhandlung.
13. **Tarife mit Scan-Kontingenten statt Flatrate festlegen.** Die variablen Kosten je Nutzer schwanken zwischen 0,32 und 6,52 EUR — Faktor 20. Ein 9,99-EUR-Einheitsabo ist bei einem Slab-lastigen Händler mit 500 Scans und 30 Prozent Apple-Provision ein Verlustgeschäft. Vorschlag: Sammler 4,99 / 50 Scans, Pro 12,99 / 300 Scans, Händler 39,99 / 1.500 Scans, Zusatzscans 0,05 EUR.
14. **Einwilligungstext, Datenschutzerklärung und Löschkonzept anwaltlich prüfen lassen** — insbesondere den Pool-Schalter und die Frage, ob SERO für gewerbliche Nutzer Auftragsverarbeiter wird. Grobe Größenordnung 1.500 bis 4.000 EUR einmalig; das ist eine Schätzung, keine belegte Zahl. Der einzige Posten, an dem hier nicht gespart werden sollte.
15. **Terapeak-Kalibrierung als feste Monatsroutine**: 20 Artikel aus dem eigenen Bestand von Hand im Seller Hub gegenprüfen und mit der Pfad-A-Zahl vergleichen. Nicht automatisieren. Das ist die einzige lizenzierte, kostenlose Möglichkeit, die eigene Preisqualität zu messen.

---

## Der eine Satz, der bleibt

Bis etwa 2.000 Nutzer trägt dieser Plan ohne eine einzige Anfrage; darüber ist mindestens ein Gespräch unvermeidbar — mit Ximilar über den Tarif, mit eBay über §8.1(d) oder mit niemandem, wenn der eigene Pool bis dahin groß genug geworden ist. Genau deshalb ist der Pool nicht ein netter Zusatz, sondern die einzige Komponente des Systems, die nicht irgendwann jemandem gehört, der nein sagen kann.