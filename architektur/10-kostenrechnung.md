# SERO — Betriebskostenrechnung, Stand 03.08.2026

## 0. Was hier belegt ist und was Annahme ist

| Kategorie | Status |
|---|---|
| Listenpreise Ximilar, PokemonPriceTracker, JustTCG, UPCitemdb, SerpApi, eBay-Limits | **belegt** aus den gegengeprüften Quellen |
| Credit-Kosten je Ximilar-Operation (6/10/15/…) | **belegt** (korrigierte Fassung der Prüfung) |
| Warenmix, Nutzermix, Katalog-Universen, Cache-Modell | **meine Annahme**, unten offengelegt und veränderbar |
| Ximilar-Preise oberhalb 3 Mio. Credits | **nicht belegt** — die Prüfung nennt Stufen bis 40M, aber Preise nur bis 3M (1.263 EUR). Alles darüber ist Extrapolation und ausdrücklich gekennzeichnet |
| Infrastrukturkosten | **meine Schätzung** (Hetzner Frankfurt), nicht recherchiert |
| Wechselkurs 1 EUR = 1,09 USD | **Annahme**, nicht geprüft |

---

## 1. Annahmen, aus denen alles folgt

### 1.1 Nutzermix

```
95 % Normalnutzer × 30 Scans   = 28,5
 5 % Vielnutzer   × 500 Scans  = 25,0
─────────────────────────────────────
Durchschnitt                   = 53,5 Scans/Nutzer/Monat
```

Die 5 % Vielnutzer erzeugen **47 % des gesamten Scanvolumens**. Das ist die wichtigste Zahl der ganzen Rechnung — sie entscheidet später über die Tarifstruktur.

### 1.2 Warenmix je Scan

| Warenart | Anteil | Begründung |
|---|---|---|
| Rohe Sammelkarten | 60 % | Masse des Sammlermarkts |
| Gegradete Slabs | 20 % | hoher Warenwert, hohe App-Relevanz |
| Sealed Product | 12 % | Displays, ETBs, Booster-Boxen |
| Retro-Videospiele | 5 % | |
| Manga/Comics | 3 % | |

Innerhalb "rohe Karten": Pokémon 60 %, MTG 15 %, Yu-Gi-Oh 12 %, One Piece/Lorcana 8 %, Sport/sonstige 5 %.

### 1.3 Cache-Modell (der entscheidende Teil)

Der 24-Stunden-Zwischenspeicher wirkt nur, wenn zwei Nutzer **dasselbe Preis-Objekt** am selben Tag scannen. Das Preis-Objekt ist nicht "eine Karte", sondern:

- Rohkarte: Karte × Auflage × Zustand
- Slab: Karte × Auflage × Gradingfirma × Note
- Sealed: EAN
- Retro: Spiel × Plattform × Zustandsstufe (loose/CIB/sealed/graded)

Ich modelliere je Kategorie ein **effektives Schlüssel-Universum** `U` (nach Popularitätskonzentration; die Long-Tail-Karten, die nie gescannt werden, zählen nicht mit) und rechne mit der Coupon-Collector-Formel:

```
Eindeutige externe Abfragen pro Tag:  D = U × (1 − e^(−S/U))
S = Scans dieser Kategorie pro Tag
```

| Kategorie | U (effektiv) | warum |
|---|---|---|
| Rohe Karten | 40.000 | Pokémon ~20.500 Karten (belegt via pokemontcg.io), plus MTG/YGO/OP-Kernbestand, × wenige Zustandsstufen, stark konzentriert auf Chase-Karten |
| Slabs | 250.000 | Karte × 4 Grader × 6 relevante Noten — multiplikativ, kaum Überlappung |
| Sealed | 6.000 | endliche Zahl aktueller Displays/ETBs pro Sprache, sehr starke Überlappung |
| Retro | 120.000 | ~24.000 Spiele × 5 Zustandsstufen, breit gestreut |
| Manga/Comics | 150.000 | Serie × Band × Auflage × ggf. Note, praktisch keine Überlappung |

---

## 2. Externe Abfragen je Szenario

### 2.1 Szenario 1 — 1.000 Nutzer

```
Scans/Monat = 1.000 × 53,5           = 53.500
Scans/Tag   = 53.500 / 30            =  1.783
```

| Kategorie | Scans/Tag | Rechnung D = U×(1−e^(−S/U)) | D/Tag | extern/Monat | Cache-Treffer |
|---|---|---|---|---|---|
| Roh | 1.070 | 40.000×(1−e^−0,02675) = 40.000×0,026395 | 1.056 | 31.674 | 1,3 % |
| Slabs | 357 | 250.000×(1−e^−0,001427) | 356 | 10.694 | 0,1 % |
| Sealed | 214 | 6.000×(1−e^−0,035667) | 210 | 6.306 | 1,8 % |
| Retro | 89 | 120.000×(1−e^−0,000743) | 89 | 2.673 | 0,04 % |
| Manga | 54 | 150.000×(1−e^−0,000357) | 53 | 1.605 | 0,03 % |
| **Summe** | **1.783** | | **1.765** | **52.950** | **1,0 %** |

**Befund: Bei 1.000 Nutzern ist der Cache praktisch wirkungslos.** 99 % aller Scans lösen eine externe Abfrage aus. Wer bei dieser Größe mit Cache-Ersparnis kalkuliert, rechnet sich reich.

### 2.2 Szenario 2 — 10.000 Nutzer

```
Scans/Monat = 535.000   Scans/Tag = 17.833
```

| Kategorie | Scans/Tag | Rechnung | D/Tag | extern/Monat | Cache-Treffer |
|---|---|---|---|---|---|
| Roh | 10.700 | 40.000×(1−e^−0,2675) = 40.000×0,234721 | 9.389 | 281.670 | **12,3 %** |
| Slabs | 3.567 | 250.000×(1−e^−0,014267) | 3.542 | 106.245 | 0,7 % |
| Sealed | 2.140 | 6.000×(1−e^−0,356667) = 6.000×0,300016 | 1.800 | 54.003 | **15,9 %** |
| Retro | 892 | 120.000×(1−e^−0,007433) | 888 | 26.646 | 0,4 % |
| Manga | 535 | 150.000×(1−e^−0,003567) | 534 | 16.020 | 0,2 % |
| **Summe** | **17.833** | | **16.153** | **484.578** | **9,4 %** |

### 2.3 Szenario 3 — 50.000 Nutzer

```
Scans/Monat = 2.675.000   Scans/Tag = 89.167
```

| Kategorie | Scans/Tag | Rechnung | D/Tag | extern/Monat | Cache-Treffer |
|---|---|---|---|---|---|
| Roh | 53.500 | 40.000×(1−e^−1,3375) = 40.000×0,73750 | 29.500 | 885.000 | **44,9 %** |
| Slabs | 17.833 | 250.000×(1−e^−0,071333) | 17.212 | 516.366 | 3,5 % |
| Sealed | 10.700 | 6.000×(1−e^−1,783333) = 6.000×0,831923 | 4.992 | 149.745 | **53,4 %** |
| Retro | 4.458 | 120.000×(1−e^−0,037153) | 4.377 | 131.295 | 1,8 % |
| Manga | 2.675 | 150.000×(1−e^−0,017833) | 2.651 | 79.539 | 0,9 % |
| **Summe** | **89.167** | | **58.732** | **1.761.945** | **34,1 %** |

**Kernbefund zum Cache:** Er zahlt sich ausschließlich bei rohen Karten und Sealed Product aus, und erst ab etwa 10.000 Nutzern spürbar. Bei Slabs, Retro-Spielen und Comics liegt die Trefferquote selbst mit 50.000 Nutzern unter 4 % — genau bei den Warenarten mit den höchsten Warenwerten und den teuersten Datenquellen.

---

## 3. Die Kostentreiber sind nicht die, die man erwartet

Drei strukturell verschiedene Kostenarten:

| Typ | Beispiel | Skaliert mit |
|---|---|---|
| **A — Fixkosten (Katalog endlich)** | PokemonPriceTracker, Scryfall-Bulk, TCGCSV, JustTCG, UPCitemdb nach EAN-Aufbau | dem Katalog, **nicht** der Nutzerzahl |
| **B — variabel, gecacht** | Preisabfragen für Slabs/Retro/Manga | Nutzerzahl, gedämpft durch Cache (hier: kaum) |
| **C — variabel, nicht cachebar** | **Bilderkennung (Ximilar)** | linear mit jedem einzelnen Scan |

**Jedes Foto ist neu.** Der Cache kann Preise sparen, aber niemals Erkennung. Deshalb ist die Bilderkennung — nicht die Preisabfrage — der größte Posten.

### 3.1 Ximilar-Credits je Scan (Rechenweg)

Belegte Credit-Sätze: detect 6 · tcg_id 10 · comics_id 10 · slab_id 15 · card_ocr_id 15.
Der Ximilar-Pricing-Endpunkt (10 Credits) wird **nicht** genutzt — laut Prüfung sind Quellmärkte US und die Währung undokumentiert, für eBay.de wertlos.

```
Rohkarte  (60 %):  detect 6 + tcg_id 10                = 16 Cr
Slab      (20 %):  detect 6 + slab_id 15 + tcg_id 10   = 31 Cr
Sealed    (12 %):  kein Ximilar-Endpunkt → Barcode     =  0 Cr
Retro      (5 %):  80 % Barcode = 0 Cr
                   20 % gegradet: detect 6 + ocr 15    = 21 Cr
Manga      (3 %):  detect 6 + comics_id 10 (+ teilw. slab_id) ≈ 20 Cr

Gewichtet:
0,60×16 = 9,60
0,20×31 = 6,20
0,12× 0 = 0,00
0,05×(0,8×0 + 0,2×21) = 0,21
0,03×20 = 0,60
──────────────────────
        = 16,61 Credits je Scan
```

---

## 4. Szenario 1 — 1.000 Nutzer

### 4.1 Ximilar

```
53.500 Scans × 16,61 Cr = 888.635 Credits/Monat
→ Tarif Professional 1M = 1.000.000 Credits für 499 EUR   [belegt]
   (Business 500K zweimal wären 570 EUR — schlechter)
Effektiv: 499 EUR / 888.635 Cr = 0,000562 EUR/Credit
         = 0,0093 EUR je Scan
```

### 4.2 Preisquellen

| Quelle | Deckt ab | Rechnung | Kosten |
|---|---|---|---|
| PokemonPriceTracker **Business** | Pokémon roh + Sealed + Slabs mit PSA/CGC/BGS-Note, Cardmarket-EUR (Beta) | 31.674 × 0,60 = 19.004 Abfragen × 3 Credits = 57.012 Cr/Monat; Kontingent 6 Mio/Monat → 1 % ausgelastet | **99 USD = 91 EUR** |
| Scryfall Bulk (MTG) + YGOPRODeck | MTG-EUR, YGO-Untergrenze | Tagesabzug, keine Einzelabfragen | 0 EUR |
| TCGCSV | Sealed-Katalog USD | 1 Vollabzug/Tag | 0 EUR |
| JustTCG **Starter** | One Piece/Lorcana/Sport — **nur USD, keine EU-Daten** | 31.674 × 0,13 = 4.118 Kartenpreise, gebatcht à 100 = 42 Calls/Monat | **19 USD = 17 EUR** |
| UPCitemdb **DEV** | EAN-Identifikation Sealed + Retro | 53.500 × 0,16 = 8.560 Lookups/Monat; Kontingent 600.000 | **99 USD = 91 EUR** |
| **Slabs + Retro + Manga** | keine lizenzierte Quelle vorhanden | 10.694 + 2.673 + 1.605 = **14.972 Abfragen/Monat** | siehe zwei Varianten |

**Variante A — eBay Browse (aktive Angebote, legal, kostenlos):**
```
Nicht-Karten-Abfragen/Tag = 356 + 210 + 89 + 53 = 708
eBay-Limit = 5.000 Calls/Tag pro Anwendung  → 14 % ausgelastet
Kosten: 0 EUR
Preis-Aussage: Angebotspreise, nicht Verkaufspreise
```

**Variante B — SerpApi Sold (echte eBay.de-Verkäufe):**
```
14.972 Abfragen → Tarif Production 150 USD / 15.000 Suchen  [belegt]
Passt haarscharf. 150 USD = 138 EUR
Durchsatz: 14.972/30/24 = 21 Abfragen/h — Limit 3.000/h, unkritisch
```

### 4.3 Gesamt Szenario 1

| Posten | Variante A | Variante B |
|---|---|---|
| Ximilar Erkennung | 499 | 499 |
| PokemonPriceTracker | 91 | 91 |
| JustTCG | 17 | 17 |
| UPCitemdb | 91 | 91 |
| Slabs/Retro/Manga | 0 (eBay Browse) | 138 (SerpApi) |
| **Datenkosten** | **698 EUR** | **836 EUR** |
| Infrastruktur (Schätzung) | 60 | 60 |
| Apple Developer (99 USD/Jahr) | 8 | 8 |
| **GESAMT/Monat** | **766 EUR** | **904 EUR** |
| **je Nutzer/Monat** | **0,77 EUR** | **0,90 EUR** |

Ximilar-Anteil an den Datenkosten: **71 % (A) bzw. 60 % (B)**.

---

## 5. Szenario 2 — 10.000 Nutzer

### 5.1 Ximilar

```
535.000 Scans × 16,61 = 8.886.350 Credits/Monat

Belegbarer Weg: 3 × Professional 3M (1.263 EUR) = 9.000.000 Cr für 3.789 EUR
Extrapoliert:   Stufen bis 40M existieren, Preise unbelegt.
                Trend 1M→3M: 0,000499 → 0,000421 EUR/Cr (−16 %)
                Fortschreibung auf 9M: ≈ 0,00038 EUR/Cr → 3.377 EUR

PLANWERT: 3.500 EUR/Monat   (Spanne 3.100 – 3.800)
```
**Wichtig:** Hier verlässt man den belegten Preisbereich. Vor Skalierung ist ein Angebot einzuholen — was streng genommen bereits Anforderung 1 berührt.

### 5.2 Preisquellen

| Quelle | Rechnung | Kosten |
|---|---|---|
| PokemonPriceTracker Business | 281.670 × 0,60 = 169.002 Abfragen × 3 Cr = 507.006 Cr/Monat; Tagesbedarf 16.900 Cr vs. 200.000 Limit → 8 % | **91 EUR** (unverändert) |
| Scryfall / YGOPRODeck / TCGCSV | Bulk | 0 EUR |
| JustTCG Starter | 281.670 × 0,13 = 36.617 → 367 Calls gebatcht; Kontingent 10.000 | **17 EUR** (unverändert) |
| UPCitemdb DEV | 85.600 Lookups/Monat, davon ≥90 % aus lokalem EAN-Cache; Kontingent 600.000 | **91 EUR** (unverändert) |

**Variante A — eBay Browse: KIPPT HIER.**
```
Nicht-Karten-Abfragen/Tag = 3.542 + 1.800 + 888 + 534 = 6.764
eBay-Limit 5.000/Tag → 135 % Auslastung → Kontingent gesprengt
Erhöhung nur über "Application Growth Check" = Genehmigungsverfahren
→ Ohne Antrag müssten Slab-, Retro- und Comic-Preise abgeschaltet werden
```

**Variante B — SerpApi:**
```
106.245 + 26.646 + 16.020 = 148.911 Abfragen/Monat
Größter Selbstbedienungstarif: Big Data 275 USD / 30.000 Suchen
→ Bedarf ist das 5-fache. Nur noch "Enterprise — Contact sales".
Kosten bei fortgeschriebenem Big-Data-Satz 9,17 USD/1.000:
148.911 × 0,00917 = 1.365 USD = 1.253 EUR/Monat   [Satz unbelegt]
Durchsatz: 148.911/30/24 = 207/h Schnitt, Spitzen 4× = 830/h — Limit 6.000/h, unkritisch
```

### 5.3 Gesamt Szenario 2

| Posten | Variante A | Variante B |
|---|---|---|
| Ximilar Erkennung | 3.500 | 3.500 |
| PokemonPriceTracker | 91 | 91 |
| JustTCG | 17 | 17 |
| UPCitemdb | 91 | 91 |
| Slabs/Retro/Manga | **nicht mehr möglich** | 1.253 |
| **Datenkosten** | **3.699 EUR** | **4.952 EUR** |
| Infrastruktur | 250 | 250 |
| Apple | 8 | 8 |
| **GESAMT/Monat** | **3.957 EUR** | **5.210 EUR** |
| **je Nutzer/Monat** | **0,40 EUR** | **0,52 EUR** |

Ximilar-Anteil: **95 % (A) bzw. 71 % (B)**.

---

## 6. Szenario 3 — 50.000 Nutzer

### 6.1 Ximilar

```
2.675.000 Scans × 16,61 = 44.431.750 Credits/Monat

Belegbarer Weg: 15 × Professional 3M = 45 Mio Cr für 18.945 EUR
Größte in der Recherche genannte Stufe: 40M — Preis nicht belegt.
44,4 Mio liegt DARÜBER → Individualvertrag zwingend.
Extrapoliert bei ~0,00030 EUR/Cr: 13.330 EUR

PLANWERT: 15.000 EUR/Monat   (Spanne 13.300 – 18.900)
```
**Anforderung 1 ist hier verletzt** — es gibt keinen buchbaren Selbstbedienungstarif mehr.

### 6.2 Preisquellen

| Quelle | Rechnung | Kosten |
|---|---|---|
| PokemonPriceTracker Business | 885.000 × 0,60 = 531.000 Abfragen × 3 Cr = 1,593 Mio Cr/Monat vs. 6 Mio Kontingent (27 %); Tagesbedarf 53.100 Cr vs. 200.000 (27 %) | **91 EUR — unverändert!** |
| Scryfall / YGOPRODeck / TCGCSV | Bulk | 0 EUR |
| JustTCG Starter | 885.000 × 0,13 = 115.050 → 1.151 Calls gebatcht | **17 EUR — unverändert** |
| UPCitemdb DEV | 428.000 Lookups/Monat theoretisch vs. 600.000 Kontingent; real ≈ 5 % wegen EAN-Cache | **91 EUR — unverändert** |
| **Slabs/Retro/Manga (SerpApi Enterprise)** | 516.366 + 131.295 + 79.539 = **727.200 Abfragen/Monat**<br>727.200 × 0,00917 USD = 6.668 USD | **6.117 EUR** |

Durchsatzprüfung SerpApi bei 50.000 Nutzern:
```
727.200 / 30 / 24 = 1.010 Abfragen/h im Schnitt
Tagesspitze (abends, Faktor 3–4)      ≈ 3.000 – 4.000/h
Big-Data-Durchsatz 6.000/h → technisch tragbar, Volumen aber weit über Tarif
```

### 6.3 Gesamt Szenario 3

| Posten | Variante A (ohne Slab-Preise) | Variante B |
|---|---|---|
| Ximilar Erkennung | 15.000 | 15.000 |
| PokemonPriceTracker | 91 | 91 |
| JustTCG | 17 | 17 |
| UPCitemdb | 91 | 91 |
| Slabs/Retro/Manga | 0 | 6.117 |
| **Datenkosten** | **15.199 EUR** | **21.316 EUR** |
| Infrastruktur | 900 | 900 |
| Apple | 8 | 8 |
| **GESAMT/Monat** | **16.107 EUR** | **22.224 EUR** |
| **je Nutzer/Monat** | **0,32 EUR** | **0,44 EUR** |

Ximilar-Anteil: **99 % (A) bzw. 70 % (B)**.

---

## 7. Zusammenfassung der drei Szenarien

| | 1.000 Nutzer | 10.000 Nutzer | 50.000 Nutzer |
|---|---|---|---|
| Scans/Monat | 53.500 | 535.000 | 2.675.000 |
| Externe Abfragen/Monat | 52.950 | 484.578 | 1.761.945 |
| Cache-Trefferquote | 1,0 % | 9,4 % | 34,1 % |
| Ximilar (Erkennung) | 499 € | 3.500 € | 15.000 € |
| Kartenpreise (fix) | 199 € | 199 € | 199 € |
| Slab-/Retro-/Comic-Preise | 138 € | 1.253 € | 6.117 € |
| Infrastruktur + Apple | 68 € | 258 € | 908 € |
| **Gesamt/Monat** | **904 €** | **5.210 €** | **22.224 €** |
| **je Nutzer/Monat** | **0,90 €** | **0,52 €** | **0,44 €** |

Die Kosten je Nutzer **fallen** mit der Größe — von 0,90 auf 0,44 EUR. Aber die Ersparnis kommt zu 60 % aus den fixen Katalogquellen und nur zu 40 % aus dem Cache.

---

## 8. Wo genau welche Quelle kippt

| # | Quelle | Kipppunkt | Was bricht | Rechenweg |
|---|---|---|---|---|
| **1** | **eBay Browse API** | **~2.800 Nutzer**, wenn auch für rohe Karten genutzt<br>**~7.000 Nutzer**, nur für Slabs/Sealed/Retro/Comics | 5.000 Calls/Tag **pro Anwendung**. Erhöhung nur über Application Growth Check = Genehmigungsverfahren | 1.765 Abfragen/Tag je 1.000 Nutzer → 5.000/1.765 = 2,8<br>708/Tag je 1.000 Nutzer → 5.000/708 = 7,1 |
| **2** | **SerpApi Selbstbedienung** | **~2.000 Nutzer** | Größter buchbarer Tarif Big Data = 30.000 Suchen/Monat. Darüber nur "Contact sales" | 14.972 Abfragen bei 1.000 Nutzern → 30.000/14.972 = 2,0 |
| **3** | **Ximilar belegter Preisbereich** | **~3.400 Nutzer** | Größte belegte Stufe = 3 Mio Credits (1.263 EUR). Darüber nur Paketstapelung oder unbelegte Stufen | 3.000.000 / 16,61 = 180.614 Scans → /53,5 = 3.376 |
| **4** | **Ximilar Preisliste insgesamt** | **~45.000 Nutzer** | Auch die größte genannte Stufe (40M) ist erschöpft → Individualvertrag | 40.000.000 / 16,61 / 53,5 = 45.010 |
| **5** | UPCitemdb DEV (99 USD) | ~70.000 Nutzer ohne Cache, praktisch nie mit EAN-Cache | 600.000 Lookups/Monat | 428.000 bei 50.000 Nutzern → ×1,4 |
| **6** | **PokemonPriceTracker Business** | **kippt strukturell nie** | Selbst bei vollständiger Sättigung des Kartenuniversums: 40.000 × 0,6 × 3 = 72.000 Credits/Tag vs. 200.000 Limit | dauerhaft 99 USD flat |
| **7** | Scryfall / YGOPRODeck / TCGCSV | nie (Bulk-Abzüge) | O(Katalog), nicht O(Nutzer) | 0 EUR dauerhaft |

**Die Reihenfolge des Zerbrechens ist das Ergebnis:**
1. Bei **rund 2.000 Nutzern** verliert man die letzte sofort buchbare Quelle für echte eBay-Verkäufe.
2. Bei **rund 2.800 Nutzern** reicht eBay Browse nicht mehr für alles.
3. Bei **rund 3.400 Nutzern** verlässt man Ximilars belegte Preisliste.
4. Bei **rund 7.000 Nutzern** ist auch der eingeschränkte eBay-Browse-Betrieb tot.

Das heißt: **Der Kostenplan trägt bis etwa 2.000 Nutzer ohne eine einzige Anfrage. Danach ist mindestens ein Vertriebsgespräch unvermeidbar — mit Ximilar, mit SerpApi oder mit eBay.** Das ist keine Preisfrage, sondern eine Strukturfrage, und sie ist mit den heute verfügbaren Anbietern nicht wegzurechnen.

---

## 9. Welcher Abo-Preis das trägt

### 9.1 Erlös nach Abzügen

```
Apple 15 % (Small Business Program, < 1 Mio USD/Jahr):
  Nettoerlös = Brutto ÷ 1,19 (USt) × 0,85 = Brutto × 0,7143

Apple 30 % (ab 1 Mio USD Jahreserlös):
  Nettoerlös = Brutto ÷ 1,19 × 0,70 = Brutto × 0,5882
```

| Bruttopreis | Erlös bei 15 % | Erlös bei 30 % |
|---|---|---|
| 4,99 € | 3,56 € | 2,94 € |
| 9,99 € | 7,14 € | 5,88 € |
| 14,99 € | 10,71 € | 8,82 € |
| 39,99 € | 28,56 € | 23,52 € |

### 9.2 Der eigentliche Kostenschock ist nicht die API — es ist Apple

```
Schwelle 1 Mio USD ≈ 917.000 EUR Jahreserlös
Erlös je Nutzer/Jahr bei 9,99 € brutto, 15 %: 7,14 × 12 = 85,71 €
917.000 / 85,71 = 10.700 Nutzer
```

**Genau im Szenario 2 kippt Apple von 15 % auf 30 %.** Wirkung:

```
Erlösverlust je Nutzer: 7,14 − 5,88 = 1,26 €
Bei 10.000 Nutzern:     12.600 €/Monat

Zum Vergleich: sämtliche Datenkosten in Szenario 2 = 5.210 €/Monat
→ Der Provisionssprung kostet das 2,4-fache aller Datenquellen zusammen.
```

Konsequenz: Ein Web-Checkout außerhalb des App Store (Stripe, ~3 % statt 30 %) ist finanziell wichtiger als jede API-Verhandlung. Nach den Recherchenotizen im Projektstand (App-Store-Regel 3.1.3(f)) ist das für ein Reader-/Abo-Modell prüfenswert.

### 9.3 Grenzkosten je Scan — die Zahl, aus der Tarife folgen

```
Ximilar:      16,61 Cr × 0,000499 €/Cr (Professional 1M) = 0,00829 €
              16,61 Cr × 0,000421 €/Cr (Professional 3M) = 0,00699 €
Preisabfrage: 28 % der Scans sind Slab/Retro/Comic
              davon 97 % Cache-Miss → 0,272 externe Abfragen je Scan
              × 0,00917 USD = 0,0025 USD                 = 0,00229 €
Barcode:      nach EAN-Katalogaufbau                     ≈ 0 €
Kartenpreise: Fixkosten                                  = 0 €
────────────────────────────────────────────────────────────────
GRENZKOSTEN JE SCAN ≈ 0,0106 € (kleiner Tarif) bis 0,0093 € (großer Tarif)
```

### 9.4 Deckungsbeitrag je Nutzertyp

| Nutzertyp | Scans | Variable Kosten | Erlös bei 9,99 € (15 %) | DB | Erlös bei 9,99 € (30 %) | DB |
|---|---|---|---|---|---|---|
| Normalnutzer | 30 | 30 × 0,0106 = **0,32 €** | 7,14 € | **+6,82 €** | 5,88 € | **+5,56 €** |
| Vielnutzer (Sammler) | 500 | 500 × 0,0106 = **5,30 €** | 7,14 € | **+1,84 €** | 5,88 € | **+0,58 €** |
| Vielnutzer (Händler, 40 % Slabs) | 500 | Ximilar 19,6 Cr/Scan → 4,89 € + Preise 1,63 € = **6,52 €** | 7,14 € | **+0,62 €** | 5,88 € | **−0,64 €** |

**Ein gewerblicher Vielnutzer mit 500 Slab-lastigen Scans ist bei einem 9,99-EUR-Einheitsabo und 30 % Apple-Provision ein Verlustgeschäft.** Und das ist exakt die Zielgruppe, die zahlen soll.

### 9.5 Empfohlene Tarifstruktur

| Tarif | Brutto | Scan-Kontingent | Max. variable Kosten | Erlös (15 %) | DB | Erlös (30 %) | DB |
|---|---|---|---|---|---|---|---|
| **Sammler** | 4,99 € | 50 | 0,53 € | 3,56 € | +3,03 € | 2,94 € | +2,41 € |
| **Pro** | 12,99 € | 300 | 3,18 € | 9,28 € | +6,10 € | 7,64 € | +4,46 € |
| **Händler** | 39,99 € | 1.500 | 19,56 € (Slab-lastig) | 28,56 € | +9,00 € | 23,52 € | +3,96 € |
| Zusatzscans | 0,05 €/Stück | — | 0,0106 € | — | +0,04 € | — | — |

Rechenweg Händlertarif:
```
1.500 Scans × 40 % Slab-Anteil:
  Ximilar: 1.500 × 19,6 Cr = 29.400 Cr × 0,000499 = 14,67 €
  Preise:  1.500 × 0,40 × 0,97 × 0,00917 USD = 5,34 USD = 4,89 €
  Summe: 19,56 €
Erlös 39,99 € brutto × 0,7143 = 28,56 €  →  DB 9,00 €
```

**Das tragende Modell:**
- Ein Einheitspreis funktioniert nicht. Die Spanne der variablen Kosten reicht von 0,32 € bis 6,52 € je Nutzer — Faktor 20.
- **Scan-Kontingente statt Flatrate** sind zwingend, nicht optional.
- **4,99 € ist der Mindestpreis**, der die Fixkosten bei 1.000 Nutzern trägt: 1.000 × 3,56 € = 3.560 € Erlös gegen 904 € Kosten.
- Reine Kostendeckung wäre schon bei **1,26 € brutto** erreicht (0,90 € ÷ 0,7143). Der Abstand zu 4,99 € ist der Puffer für Entwicklung, Support, Steuerberatung und die im Rechtsteil genannte anwaltliche Prüfung (geschätzt 1.500–4.000 € einmalig, unbelegt).

---

## 10. Die fünf Sätze, auf die es ankommt

1. **Die Bilderkennung, nicht die Preisabfrage, ist der Kostentreiber** — 60 bis 99 % der Datenkosten. Jedes Foto ist neu; kein Cache der Welt hilft dagegen. Wer Kosten senken will, senkt Ximilar-Credits (z. B. `detect` weglassen, wenn der Nutzer den Warentyp selbst wählt: spart 6 von 16,61 Credits = 36 %).

2. **Der Cache rettet weniger, als er verspricht.** Bei 1.000 Nutzern 1 %, bei 10.000 Nutzern 9 %, bei 50.000 Nutzern 34 % — und dort fast ausschließlich bei Pokémon-Karten und Sealed Product. Bei Slabs bleibt er unter 4 %, weil Karte × Grader × Note ein zu großes Schlüsseluniversum aufspannt.

3. **Alle Quellen mit endlichem Katalog sind Fixkosten.** PokemonPriceTracker kostet bei 1.000 wie bei 50.000 Nutzern 99 USD. Scryfall, YGOPRODeck und TCGCSV kosten null. Zusammen 199 EUR/Monat für die gesamte Rohkartenwelt, in jedem Szenario. Das ist die stabilste Säule.

4. **Bei rund 2.000 Nutzern endet die antragsfreie Welt.** Nicht wegen des Geldes, sondern wegen der Kontingente: SerpApi-Selbstbedienung, eBay-Browse-Tageslimit und Ximilars belegte Preisliste laufen alle zwischen 2.000 und 3.500 Nutzern aus. Die Anforderung "eine Lösung, ohne noch mal etwas anzufragen" ist bis 2.000 Nutzern erfüllbar — darüber nicht.

5. **Der teuerste Kostensprung kommt nicht von einem Datenanbieter, sondern von Apple.** Bei etwa 10.700 zahlenden Nutzern springt die Provision von 15 auf 30 % und kostet 12.600 EUR im Monat — mehr als das Doppelte aller Datenquellen zusammen. Ein Web-Abo außerhalb des App Store spart mehr als jede Verhandlung mit Ximilar oder SerpApi.