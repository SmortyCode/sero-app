mcp__ccd_session__mark_chapter is not needed here; I have what I need. Writing up the critique.

---

# ZERLEGUNG DER ENTSCHEIDUNGSVORLAGE

Ich habe die tragenden Behauptungen nachgeprüft. Drei davon halten nicht. Eine davon trägt die gesamte Architektur.

---

## 1. Zu optimistisch — nachgeprüft

### 1.1 TÖDLICH: Pfad A ist nicht eBay-frei. Die Hauptquelle IST eBay.

Die ganze Vorlage steht auf dem Satz „Kein eBay-Datenpunkt fließt hier ein. Niemals." PokemonPriceTracker schreibt auf der eigenen FAQ-Seite wörtlich:

> „We aggregate sales data from multiple sources including eBay, TCGPlayer, and CardMarket."
> — https://www.pokemonpricetracker.com/faq

Die eigene Produktseite heißt sogar „Daily-Updated TCGPlayer & eBay Data" (https://www.pokemonpricetracker.com/pokemon-card-price-api).

Damit ist die zentrale juristische Konstruktion der Vorlage widerlegt. Der als „lizenziert und sauber" verkaufte Pfad A liefert **aus eBay-Verkäufen abgeleitete Durchschnittspreise pro Kartenkategorie** — exakt das, was §8.1(d) ohne schriftliche Erlaubnis verbietet. Die Verteidigung „eBay Content does not include information that you lawfully obtain independent of eBay" hängt am Wort *lawfully*: PPT hat mit an Sicherheit grenzender Wahrscheinlichkeit keine Lizenz von eBay, keine von TCGplayer (eBay-Tochter, API seit Übernahme geschlossen) und keine von Cardmarket (nimmt laut eigener Hilfeseite keine Anträge an). Es ist gescrapt.

**Die Vorlage wendet ihr eigenes Ausschlusskriterium inkonsistent an.** Sie wirft Apify raus mit der Begründung „Rechtlich identisch mit 130point" — und macht dann eine Quelle mit exakt derselben Datenherkunft zum Fundament. Der einzige Unterschied: bei PPT sieht man das Scraping nicht, weil eine Rechnung dazwischenliegt. Das ist keine Risikobeseitigung, das ist Risikoverschleierung. Der Betreiber würde in dem Glauben starten, sauber zu sein, während er dieselbe Rechtsverletzung nur teurer eingekauft hat.

### 1.2 Die §9-Klausel ist schärfer, als die Vorlage zugibt

eBays ALA verbietet nicht nur, ein Modell zu bauen. Der Wortlaut (bestätigt über https://developer.ebay.com/join/api-license-agreement):

> Developers may not use eBay Content, either alone or in combination with third-party information, **to suggest or model prices for items listed on the eBay Site**.

Der Verbotstatbestand ist der *Zweck*, nicht das Modul. SEROs Kernfunktion ist: Marktwert anzeigen → auf eBay.de listen. Wenn die App im selben Bildschirm „Marktwert 187 EUR" und darunter „Aktuell auf eBay.de: 4 Stück, 39–58 EUR" zeigt, dann *schlägt sie einen Preis für ein auf eBay zu listendes Objekt unter Verwendung von eBay Content vor*. Getrennte Datenbanktabellen ändern daran nichts. Die Klausel adressiert Nutzung und Darstellung, nicht Code-Architektur. Die Vorlage verkauft eine Compliance-Maßnahme, die den geregelten Tatbestand verfehlt.

### 1.3 eBay Browse: 5.000 Calls/Tag ist falsch dargestellt

Die Vorlage behandelt 5.000/Tag als harte Architekturgrenze. Das ist nur der Default. Es gibt den kostenlosen **Application Growth Check**; nach bestandener Prüfung werden Limits regelmäßig auf Größenordnungen um 1,5 Mio. Calls/Tag angehoben (https://infiplex.com/eBay-Rate-Limit-Increase-Request, https://www.uvdesk.com/en/blog/increase_ebay_api_usage_limit/). Zwei Konsequenzen, beide unangenehm für die Vorlage: Pfad B skaliert besser als behauptet — **aber** der Growth Check ist ein Prüfverfahren mit Offenlegung des Anwendungsfalls. Damit fällt auch die Kernprämisse „ohne dass ich noch mal irgendwas anfrage": Spätestens bei 5.000 Calls/Tag (das sind bei 1.000 Nutzern fünf Kontext-Abrufe pro Nutzer und Monat) muss er zu eBay — und dann legt er eBay genau die App offen, deren §8.1(d)-Konformität die Vorlage selbst als „juristische Konstruktion ohne Rechtsprechung" bezeichnet.

### 1.4 Zwei weitere Beschönigungen

- **PPT-Zitat unvollständig.** Die Vorlage zitiert nur die erlaubende Hälfte. Vollständig (https://www.pokemonpricetracker.com/terms): *„You may not resell, sublicense, syndicate, or redistribute the raw data itself as a standalone product or data service."* Für die Händler-Stufe (39,99 / 1.500 Scans, Bulk-Bewertung, CSV-Export, eigene API) ist das eine offene Flanke.
- **Ximilar-Limits.** Die Vorlage nennt „nicht dokumentiert" und geht zur Tagesordnung über. Die Preisseite (https://www.ximilar.com/pricing/) bestätigt reines Credit-Modell ohne veröffentlichte Rate Limits. Eine Anforderung des Betreibers lautet wörtlich: „Nenne konkrete Ratenlimits und ob sie pro Konto oder pro Nutzer gelten." Für die *Erkennungsquelle des gesamten Produkts* liefert die Vorlage das nicht — und empfiehlt trotzdem sofortige Buchung.

---

## 2. Übersehene Anbieter

### 2.1 TCGdex liefert Cardmarket-EUR-Preise — kostenlos

Die Vorlage führt TCGdex als reinen Katalog. Falsch, Stand heute:

> „Real-time pricing from Europe's Cardmarket and North America's TCGplayer… Pricing data is automatically included in every card response under the `pricing` field."
> — https://tcgdex.dev/markets-prices

Cardmarket-Durchschnitte, Trends, Tiefstpreise, 7/30-Tage-Historie, EUR, täglich. Das ist genau die Funktion, für die die Vorlage 99 USD/Monat an einen anonymen Anbieter zahlen will. Der verbleibende Mehrwert von PPT schrumpft damit auf **einen** Punkt: Preise je Grading-Note. Für Rohkarten ist die Empfehlung schlicht überteuert.

### 2.2 Scrydex wird in einem Nebensatz erledigt — und schließt drei behauptete Lücken

Die Vorlage erwähnt Scrydex nur als Nachfolger von pokemontcg.io. Tatsächlich (https://scrydex.com/pricing): Starter 29 USD / 5.000 Credits, **Growth 99 USD / 50.000 Credits**, Professional 399 USD / 250.000 Credits, Overage 0,0016–0,006 USD. In **allen** Tarifen enthalten: „All TCGs", „Raw Prices", **„Graded Prices"**, **„Population Reports"**, „Price History", **„Vision: Image Analysis"**. Abgedeckt: Pokémon, MTG, Lorcana, One Piece, Gundam, Riftbound.

Das trifft drei Aussagen der Vorlage direkt:
- „One Piece / Lorcana → LÜCKE" — abgedeckt.
- „SLAB → ALLE ANDEREN → LÜCKE" — für TCG-Slabs teilweise abgedeckt.
- „GemRate ist der einzige legitime Weg zu graderübergreifenden Population-Daten — kostet aber ein Vertriebsgespräch" — Scrydex liefert Population-Reports selbstbedient ab 29 USD.

Zusätzlich ersetzt „Vision: Image Analysis" (5 Credits) potenziell einen Teil von Ximilar. Zu prüfen bleibt: EUR/Cardmarket wird auf der Preisseite **nicht** genannt, und die Lizenzbedingungen habe ich **nicht** verifiziert. Aber ein Anbieter, der drei als unlösbar deklarierte Lücken adressiert, verdient mehr als einen Klammersatz.

### 2.3 Weitere, in der Vorlage nicht einmal genannt

| Anbieter | Warum relevant | Status meiner Prüfung |
|---|---|---|
| **CardHedger** (https://www.cardhedger.com/price_api_business) | Explizites „Business"-Preis-API für Sports Cards + Pokémon | Seite nicht auslesbar — ungeprüft, muss geprüft werden |
| **RetroCharting** (https://retrocharting.com/) | 90.000+ Titel, Werte für Loose/CIB/New/**Graded** — genau die als „VOLLSTÄNDIGE LÜCKE" deklarierte Retro-Kategorie | 403 bei Abruf — ungeprüft, API-Existenz offen |
| **CovrPrice** (https://covrprice.com/) | Comics roh + gegradet, **hat nachweislich eine API** (CLZ nutzt sie produktiv: https://clz.com/comics/covrprice) | API existiert belegt; Konditionen ungeprüft |
| **pokewallet.io, tcgapi.dev, pokemon-api.com** | Weitere TCG-Preis-APIs | Ungeprüft |

Die Vorlage erklärt Retro und Comics zur „VOLLSTÄNDIGEN LÜCKE" und baut darauf ihre gesamte Endzustandsstrategie („keine Marktwertzahl, auf Jahre"). Für beide Kategorien existieren Kandidaten, die nicht einmal aufgeschlagen wurden. Eine als unüberwindbar dargestellte Produktgrenze, hinter der drei ungeprüfte Türen stehen, ist kein Rechercheergebnis.

### 2.4 Der eigentliche blinde Fleck: es gibt den Wettbewerb schon

Kein Wort über **Ludex, CollX, Slabfy, Collectr, CardGrader**. Ludex bietet „List It With Ludex" — Scan → Wert aus eBay-Verkäufen → automatisch befülltes eBay-Listing (https://www.ludex.com/list-it/). Das ist SEROs Produkt, live, seit Jahren, mit Preisen aus eBay-Sold-Daten. Zwei Schlüsse, beide entscheidungsrelevant:

1. Die §8.1(d)-Frage ist offenbar nicht so tödlich, wie die Vorlage sie zeichnet — sonst gäbe es diese Apps nicht. Entweder haben sie die schriftliche Erlaubnis (dann ist der Weg gangbar und muss nicht umgangen werden), oder eBay duldet es (dann ist die gesamte Pfad-A/B-Konstruktion überengineert). Beide Antworten ändern den Bauplan. Die Vorlage hat die Frage nicht gestellt.
2. SEROs Alleinstellung wäre **Deutschland/EUR/eBay.de** — und ausgerechnet dafür empfiehlt die Vorlage überwiegend Quellen mit US-Daten.

---

## 3. Kostenrechnung — was fehlt

**„Rund 700 bis 900 Euro bei 1.000 Nutzern" ist keine Rechnung, sondern eine Setzung.** Die zugrunde liegende Scan-Zahl wird nirgends genannt. Rückgerechnet: PPT (~91 EUR) + UPCitemdb DEV (~91 EUR) lassen ~520 EUR für Ximilar, das sind bei ~17 EUR/1.000 Scans rund **30 Scans pro Nutzer und Monat**. Im selben Dokument, Punkt 13, werden Tarife mit **50 / 300 / 1.500 Scans** vorgeschlagen. Kaufen die Nutzer, was ihnen angeboten wird, liegt die Rechnung um Faktor 5 bis 30 daneben. Punkt 13 nennt selbst 0,32–6,52 EUR je Nutzer — die Vorlage widerspricht ihrer eigenen Kernzahl auf Seite 1 und löst den Widerspruch nicht auf.

Fehlende Kostenstellen, keine davon klein:

1. **Nicht zahlende Nutzer.** Die Rechnung setzt 1.000 Nutzer = 1.000 Zahler. Jede Testphase, jeder Free-Scan, jeder Abbrecher verbrennt Ximilar-Credits. Bei branchenüblicher Conversion sind das 5.000–20.000 Scanner für 1.000 Zahler. Diese Position kann die größte im ganzen Modell sein und ist mit null angesetzt.
2. **Fehlversuche.** Schlechtes Foto, falsche Erkennung, Nutzer scannt neu. Jeder Versuch kostet Credits. Realistisch 1,3–1,8 Aufrufe je erkanntem Objekt. Nirgends eingepreist.
3. **Apple/Stripe — und zwar beide.** Punkt 12 rechnet vor, dass Stripe die 30 % spart. Das stimmt seit den EU-DMA-Bedingungen nicht mehr: Externe Kauflinks kosten **5 % bzw. 13 % Store Services Fee, plus 2 % Initial Acquisition Fee, plus Core Technology Commission** (https://developer.apple.com/support/communication-and-promotion-of-offers-on-the-app-store-in-the-eu/, https://www.revenuecat.com/blog/growth/apple-eu-dma-update-june-2025/). Die tatsächliche Ersparnis liegt bei ~10–15 Punkten, nicht bei 27. Der „größte Hebel im ganzen Kostenmodell" ist etwa halb so groß wie behauptet.
4. **19 % Umsatzsteuer.** Bei B2C-Abo in Deutschland aus dem Bruttopreis. 9,99 EUR sind 8,40 EUR netto, bevor Apple etwas nimmt. Kommt im ganzen Dokument nicht vor.
5. **Betriebskosten.** Hosting, Datenbank, **Bildspeicher und Bandbreite** — eine Foto-App mit 1.000 Nutzern lädt Zehntausende Bilder pro Monat hoch. Null angesetzt.
6. **Support.** Ein Einzelunternehmer neben laufendem Handelsgeschäft für 1.000 zahlende App-Nutzer. Bei 2 % Kontaktquote sind das 20 Tickets im Monat, auf Deutsch, mit Erwartungshaltung. Bei den anvisierten „tausenden Nutzern" ist das eine Stelle. Nicht erwähnt.
7. **Recht laufend, nicht einmalig.** Punkt 14 nennt 1.500–4.000 EUR einmalig. Fehlen: AVV mit gewerblichen Nutzern, AGB-Pflege, Verbraucherrecht/Widerruf, Impressum, laufende Anpassung — plus Apple Developer Program 99 USD/Jahr.
8. **Währungsrisiko.** PPT, UPCitemdb, Scrydex rechnen in USD ab. Einnahmen in EUR. Kein Wort.
9. **Haftung.** „Marktwert" gegenüber *gewerblichen* Nutzern, die danach kaufen und verkaufen. Bei den in Punkt 2.4 genannten Abweichungen (Ludex ~8 %, CollX ~22 % über realen eBay-Sold-Werten) sind Fehlbewertungen der Normalfall, nicht der Ausnahmefall. Haftungsausschluss und dessen AGB-Wirksamkeit gegenüber Unternehmern: nicht behandelt.

---

## 4. Kleingeredete Risiken

**a) Kontosperrung — das falsche Konto wird geschützt.** Stufe 0 trennt die *130point-Infrastruktur* vom Händlerkonto. Das ist die kleinere Hälfte. Die größere: SERO listet für Nutzer auf eBay.de, also braucht es ein eBay-Developer-Keyset, und das ist auf den Betreiber registriert, verifiziert, und mit seiner Identität verknüpft. eBay verknüpft Konten routinemäßig. Der wahre Single Point of Failure ist nicht 130point — es ist **das Keyset**. Wird es wegen §8.1(d) gesperrt, ist gleichzeitig weg: das Listing-Feature (die Kernfunktion), Pfad B, getOrders, jedes zahlende Abo — und möglicherweise das Händlerkonto, das sein Broterwerb ist. Die Vorlage nennt das nirgends als Szenario.

**b) ToS-Verstoß bleibt bestehen, nur unsichtbar.** Siehe 1.1. Nach Umsetzung des Plans wäre er in dem Glauben, sauber zu sein, während PPT eBay-Daten weiterreicht. Das ist gefährlicher als der heutige Zustand, weil er heute wenigstens weiß, dass 130point ein Risiko ist.

**c) Abhängigkeit von einem einzigen Anbieter — der schlimmste denkbare Anbieter.** PokemonPriceTracker trägt laut Vorlage „den größten Einzelblock des gesamten Volumens". Was ich über diesen Anbieter feststellen konnte (https://www.pokemonpricetracker.com/terms):
- Keine Firmierung. Keine Rechtsform. Kein Sitz. Kontakt ist eine **ProtonMail-Adresse** (`pokepricetracker@proton.me`).
- US-Recht.
- **Kein SLA**, ausdrücklich „AS IS", keine Zusage zu „uninterrupted, timely, secure, or error-free".
- **„We may terminate or suspend your account and access to the Service immediately, without prior notice or liability, for any reason whatsoever."**
- Haftung auf null reduziert.
- Datenbasis vollständig fremdgescrapt (1.1) — der Anbieter hat also selbst keine gesicherte Grundlage.
- **Widerspruch zwischen Preisseite und ToS bleibt ungeklärt.** Preisseite: „Commercial use license" bei *allen* Tarifen, inkl. Free (https://www.pokemonpricetracker.com/pricing). ToS §6: „Using PokePriceTracker Data for any commercial purpose requires an active Business or Enterprise subscription." Ein Anbieter, dessen eigene Verkaufsseite seinem eigenen Vertrag widerspricht, ist kein Fundament für ein Geschäftsmodell.

Ein Einzelunternehmer soll sein Produkt auf einen anonymen Betreiber mit Wegwerf-Mailadresse gründen, der fristlos ohne Grund kündigen darf. Das gehört auf Seite 1 der Vorlage, nicht in eine Tabellenzeile mit dem Vermerk „Ausdrücklich erlaubt".

**d) Preiserhöhung.** Kein einziger der empfohlenen Anbieter hat eine Preisgarantie. Ximilar staffelt hart: 100K = 59 EUR, aber 1M = **499 EUR** (https://www.ximilar.com/pricing/) — der Sprung vom 500K- zum 1M-Tarif kostet 214 EUR für die Verdopplung. Wächst die App, wächst die Rechnung überproportional. Die Vorlage rechnet mit dem 59-EUR-Einstiegstarif als Anker und erwähnt die Staffel nicht.

**e) Klumpenrisiko Ximilar.** Erkennung *und* Slab-OCR *und* Comic-Erkennung hängen an einem einzigen tschechischen Anbieter, mit undokumentierten Limits und Abschaltvorbehalt nach alleinigem Ermessen. Fällt Ximilar aus, macht die App gar nichts mehr — sie kann dann nicht einmal mehr erkennen, worüber sie schweigen soll.

---

## 5. Wenn PokemonPriceTracker in sechs Monaten dichtmacht

Die Vorlage hat für diesen Fall **keinen Plan**. Sie hat ihn nicht einmal als Frage gestellt. Wahrscheinlichkeit: hoch — anonymer Betreiber, gescrapte Datenbasis, Rechtsanspruch der Quelleigentümer (TCGplayer gehört eBay), fristlose Kündigung ohne Grund vertraglich zugesichert.

Der Tag danach:
- **Rohe Pokémon-Karten:** auffangbar. TCGdex liefert Cardmarket-EUR kostenlos (https://tcgdex.dev/markets-prices). Ein bis zwei Tage Arbeit.
- **Gegradete Pokémon-Slabs, Preis je Note:** **fällt komplett aus.** Kandidat Scrydex (https://scrydex.com/pricing) — Graded Prices in allen Tarifen, aber EUR/Cardmarket unbestätigt. Ohne Ersatz verliert SERO die wertvollste Warengruppe.
- **Sealed Pokémon:** fällt aus. TCGCSV nur USD, ohne Zustandsabstufung.
- Kaskade: 130point ist nach Stufe 4 bereits abgeschaltet, der eigene Pool ist mit Monaten Verspätung noch dünn. Die App zeigt für ihre Kernkategorie plötzlich „zu wenig Daten" — bei laufenden Abos. Massenkündigung plus App-Store-Bewertungsschaden.

**Was der Plan bräuchte und nicht hat:** eine zweite Preisquelle von Tag eins, hinter derselben internen Schnittstelle, mit täglichem Cache der letzten 30 Tage als Notlauf. Konkret: TCGdex kostenlos parallel als Primärquelle für roh/EUR, PPT oder Scrydex ausschließlich für Noten, und eine schriftlich fixierte Ausstiegsklausel — SERO darf Preisdaten nicht nur cachen, sondern muss die Historie behalten dürfen. Die PPT-ToS erlauben „store and cache", verbieten aber Weiterverbreitung als Datenprodukt; ob der Cache nach Vertragsende weiterlaufen darf, steht dort **nicht** — das muss vor Buchung geklärt werden, nicht danach.

Und das Grundproblem, das die Vorlage in ihrem Schlusssatz selbst formuliert, aber falsch beantwortet: Der eigene Pool wächst nur, wenn die App Nutzer hat. Die App bekommt Nutzer nur, wenn sie Preise zeigt. Die Preise kommen von einem Anbieter, der jederzeit abschalten kann. **Der Pool ist keine Absicherung gegen den Ausfall — er ist von ihm abhängig.**

---

# KORREKTUREN, DIE EINGEARBEITET GEHÖREN

**Fundament**

1. **Kernaussage streichen und ersetzen.** „Kein eBay-Datenpunkt fließt hier ein. Niemals." ist falsch. PokemonPriceTracker aggregiert nach eigener Angabe eBay-Verkäufe. Der ganze Abschnitt 4.1 muss neu geschrieben werden.
2. **PPT-Zeile in Abschnitt 2.2 neu bewerten:** Datenherkunft „eBay, TCGplayer, Cardmarket — gescrapt, ohne erkennbare Lizenz". Damit fällt PPT unter dasselbe Ausschlusskriterium wie Apify und 130point. Entweder das Kriterium fällt für alle, oder PPT fällt mit. Die aktuelle Ungleichbehandlung ist der schwerste handwerkliche Fehler des Dokuments.
3. **§9 korrekt wiedergeben.** Verboten ist, Preise für auf eBay zu listende Artikel vorzuschlagen oder zu modellieren — der Zweck ist der Tatbestand, nicht das Modul. Die Behauptung, getrennte Tabellen lösten das Problem, muss raus.
4. **PPT-ToS vollständig zitieren**, inkl. „You may not resell, sublicense, syndicate, or redistribute the raw data itself as a standalone product or data service." Auswirkung auf die Händler-Stufe prüfen.
5. **PPT-Anbieterrisiko auf Seite 1:** anonym, ProtonMail, keine Firmierung, kein SLA, „AS IS", fristlose Kündigung „for any reason whatsoever", US-Recht. Nicht in eine Tabellenzeile.
6. **Widerspruch Preisseite vs. ToS §6** als Warnsignal werten, nicht als Klärungspunkt Nr. 7.

**Übersehene Optionen**

7. **TCGdex nachtragen** — liefert Cardmarket-EUR-Preise für Pokémon kostenlos (https://tcgdex.dev/markets-prices). Reduziert PPT auf reinen Noten-Lieferanten.
8. **Scrydex vollwertig prüfen** (https://scrydex.com/pricing) — Graded Prices, Population Reports, alle TCGs, Vision, ab 29 USD. Schließt „One Piece/Lorcana → LÜCKE" und relativiert „GemRate ist der einzige Weg".
9. **RetroCharting, CovrPrice, CardHedger prüfen**, bevor Retro und Comics als „VOLLSTÄNDIGE LÜCKE" ins Dokument geschrieben werden. CovrPrice hat belegt eine API (CLZ nutzt sie).
10. **Wettbewerbsabschnitt ergänzen** — Ludex, CollX, Slabfy, Collectr. Ludex macht Scan→eBay-Sold-Wert→eBay-Listing bereits live (https://www.ludex.com/list-it/). Die Frage, wie die das rechtlich machen, ist wertvoller als jede weitere ToS-Lektüre.

**Zahlen**

11. **eBay-Browse-Limit korrigieren:** 5.000/Tag ist Default, Application Growth Check hebt es an — ist aber ein Antragsverfahren und bricht damit die Prämisse „ohne noch mal anzufragen".
12. **Kostenrechnung neu**, mit offengelegter Scan-Annahme je Tarif, und Auflösung des Widerspruchs 0,70–0,90 EUR (Seite 1) gegen 0,32–6,52 EUR (Punkt 13).
13. **Fehlende Kostenstellen aufnehmen:** nicht zahlende Scanner, Fehlversuche (Faktor 1,3–1,8), 19 % USt, Hosting/Bildspeicher/Bandbreite, Support, laufende Rechtskosten, Apple Developer 99 USD/Jahr, USD-Währungsrisiko.
14. **Punkt 12 korrigieren:** Stripe-Checkout im EU-App-Store kostet 5 %/13 % Store Services Fee + 2 % Initial Acquisition Fee + Core Technology Commission. Die Ersparnis ist rund halb so groß wie behauptet.
15. **Ximilar-Preisstaffel zeigen**, nicht nur den 59-EUR-Einstieg: 500K = 285 EUR, 1M = 499 EUR.

**Risiko**

16. **Das eBay-Developer-Keyset als eigentlichen Single Point of Failure benennen** — nicht 130point. Sperrung nimmt Listing, Pfad B, getOrders und potenziell das Händlerkonto gleichzeitig.
17. **Ausfallplan für die Hauptquelle ergänzen:** zweite Preisquelle ab Tag eins hinter derselben Schnittstelle, 30-Tage-Cache als Notlauf, und vor Buchung schriftlich klären, ob gecachte Daten nach Vertragsende weiterverwendet werden dürfen.
18. **Ximilar-Klumpenrisiko benennen** — Erkennung, Slab-OCR und Comics an einem Anbieter mit undokumentierten Limits und Abschaltvorbehalt.
19. **Haftung für Fehlbewertungen gegenüber gewerblichen Nutzern** in Punkt 14 aufnehmen; die Vergleichsapps liegen 8–22 % daneben, Fehlbewertung ist der Normalfall.

**Prozess**

20. **Nicht buchen, bevor 1–6 geklärt sind.** Die Vorlage empfiehlt in Punkt 5 und 7 die Buchung von Ximilar und PPT — also Geld auszugeben — auf Basis von zwei Klauseln, die sie selbst als ungeklärt kennzeichnet, und einer Datenherkunft, die sie nicht geprüft hat.

**Nicht verifiziert, ausdrücklich offen:** exakter Wortlaut und Nummerierung von ALA §8.1(b)/(d), §8.5, §9 (developer.ebay.com war bei jedem Abrufversuch nicht auslesbar — Inhalte nur über Suchtreffer bestätigt, Abschnittsnummern der Vorlage konnte ich **nicht** verifizieren); Scryfall-ToS-Paywall-Klausel (403); UPCitemdb-Preise (404); RetroCharting (403); CardHedger; Scrydex-Lizenzbedingungen und EUR-Abdeckung; die Behauptung der Vorlage, PSA habe im Juni 2026 das Freitier abgeschafft.