# Maschinelle Erkennung gegradeter Sammlerstücke (Slabs) — Recherche & Pipeline-Empfehlung

*Read-only-Recherche, 03.08.2026. Nichts am Code geändert. Belege als URLs; alles ohne belastbare Primärquelle ist als **unbestätigt** markiert.*

---

## 0. Kurzfazit — die fünf wichtigsten Punkte

1. **Es gibt genau EINEN offiziellen, dokumentierten Cert-Lookup mit API: PSA.** `GET https://api.psacard.com/publicapi/cert/GetByCertNumber/{certNumber}`, Bearer-Token, Free-Tier **100 Calls/Tag**. Alle anderen Grader (Beckett, CGC, SGC, WATA, VGA/CGA) bieten öffentlich nur Web-Formulare, keine dokumentierte API.
2. **Der 603-€-Fehler war kein Grading-Problem, sondern ein Klassen-Problem.** Das Slab-Label sagt selbst, *was* drin ist (Comic/Manga-Label: Titel + Heftnummer + Erscheinungsdatum + Verlag; Karten-Label: Jahr/Set/Spieler/Kartennummer). Diese *item_class* muss vor jeder Preisabfrage aus dem Label abgeleitet werden und die Preisquelle bestimmen — ein Manga darf nie in den PriceCharting-Kartenindex laufen.
3. **Barcode/QR schlägt OCR.** PSA (QR seit 2020), CGC und SGC haben QR-Codes auf dem Slab, PSA zusätzlich einen Barcode mit der Cert-Nummer. Ein QR-/Barcode-Decoder (ZXing/pyzbar/OpenCV) auf dem Slab-Foto ist um Größenordnungen zuverlässiger als Text-OCR auf Plastik mit Reflexionen.
4. **Cert-Nummer + Grader → Lookup → exakte Identität** ist der Königsweg, funktioniert aber heute nur bei PSA vollautomatisch. Für die anderen: multimodales LLM als Label-Parser + strenge Format-Validierung + Konfidenz-Gate.
5. **Skalen sind NICHT vergleichbar.** BGS 9.5 ≈ PSA 10, BGS 10 Black Label ≪ 0,5 % der Einsendungen, BCCG ist notorisch 2–3 Stufen überbewertet, CGC Cards hat *kein* 9.8 (das ist CGC **Comics**). Ein gemeinsamer `card_key × grade`-Katalog braucht den Grader als Teil des Schlüssels — was in `web/catalog.py` zu prüfen wäre.

---

## 1. Label-Aufbau und Kürzel der großen Grader

### 1.1 Korrekte Kürzel (häufigste Fehlerquelle)

| Firma | Korrektes Kürzel auf dem Label | Anmerkung |
|---|---|---|
| Professional Sports Authenticator | **PSA** | auch PSA/DNA für Autogramme |
| Beckett Grading Services | **BGS** | „Beckett" ist die *Firma*, BGS der Service für moderne Karten |
| Beckett Vintage Grading | **BVG** | Karten vor ~1970er, **keine Subgrades** |
| Beckett Collectors Club Grading | **BCCG** | eigene, viel mildere Skala — nicht mit BGS vergleichen |
| Beckett Authentication Services | **BAS** | nur Autogramme/COA, kein Kartengrade |
| Certified Guaranty Company | **CGC** | Sub-Marken: CGC Cards, CGC Comics, CGC Video Games — unterschiedliche Skalen! |
| Sportscard Guaranty | **SGC** | |
| Wata Games | **WATA** | Videospiele |
| Video Game Authority (unter CGA) | **VGA** | Teil der Collectible Grading Authority |

Quelle Beckett-Familie: [allvintagecards.com/beckett-card-grading-guide](https://allvintagecards.com/beckett-card-grading-guide/), [cardgrading.app/beckett-grading-system](https://cardgrading.app/beckett-grading-system)

**Bezug zum Code:** `/Users/smorty/ebay-bot/bot/claude_client.py:139-142` (`_GRADER_KUERZEL`) mappt heute nur `beckett→BGS, bgs, psa, cgc, sgc, wata, vga, cga, ace`. **Es fehlen `bvg`, `bccg`, `bas`, `cbcs`, `tag`, `mana`, `ace`(vorhanden), `csg`.** Besonders kritisch: `beckett → BGS` ist bei einem Vintage- oder BCCG-Slab schlicht falsch und macht das Listing unauffindbar bzw. suggeriert einen falschen Wert.

### 1.2 Feldanordnung

**PSA** — Front-Label horizontal oben am Slab. Felder: Jahr, Hersteller/Set, Spieler bzw. Kartenname, Kartennummer, Varianten-/Attribut-Zusätze (z. B. „ROOKIE"), rechts die Note (z. B. `GEM MT 10`). Cert-Nummer numerisch, meist im oberen/rechten Bereich, zusätzlich als Barcode und seit 2020 als QR-Code codiert.
Quellen: [mastergrade.ai/blog/psa-label-guide](https://mastergrade.ai/blog/psa-label-guide), [pregradecards.com/blog/psa-certificate-number-verify-authenticity](https://pregradecards.com/blog/psa-certificate-number-verify-authenticity), [figoca.com/blog/psa/psa-cert-checker](https://figoca.com/blog/psa/psa-cert-checker)
Label-Farbe kodiert Sondertypen (Red/Gold/Black u. a.) — **die konkrete Farbsemantik konnte ich nicht aus einer PSA-Primärquelle belegen → unbestätigt.**

**BGS** — Front-Label mit Gesamtnote groß, darunter/rechts die **vier Subgrades** Centering / Corners / Edges / Surface. Label-Farbe ist selbst ein Datenfeld:
- **Silber** = Note 9 und darunter
- **Gold** = 9.5 Gem Mint und „10 Pristine" (Gold Label)
- **Schwarz** = 10 Black Label (alle vier Subgrades exakt 10)
BGS 9.5 verlangt alle vier Subgrades ≥ 9 und mindestens drei ≥ 9.5; BGS 10 Pristine alle ≥ 9.5 und mindestens drei = 10.
Quellen: [gradingmetric.com/bgs-card-grading](https://www.gradingmetric.com/bgs-card-grading), [zeropop.app/blog/bgs-subgrades-explained](https://zeropop.app/blog/bgs-subgrades-explained), [cardcenteringtool.com/grading-standards/bgs](https://www.cardcenteringtool.com/grading-standards/bgs)

**CGC Cards** — 10-Punkte-Skala mit Halbstufen (1.5 … 9.5), Spitzen: **Pristine 10** > **Gem Mint 10** > **Mint+ 9.5**. Sonderkennzeichnungen: `Authentic (AU)`, `Authentic Altered (AA)`, `Authentic ART`. Autogramme separat 5–10 ohne Halbstufen. **Kein 9.8/9.9 bei Karten** — retirierte Altbezeichnungen „Perfect 10"/„Gem Mint 9.5" können aber auf alten Slabs stehen.
Quelle: [cgccards.com/card-grading/grading-scale](https://www.cgccards.com/card-grading/grading-scale/)

**CGC Comics/Manga** (der Fall „One Piece Band 103") — Label zeigt **Titel, Heftnummer, Erscheinungsdatum, Verlag** plus Note 0.5–10.0; **Farbband ist Statusinformation**: blau = Universal, gelb/gold = Signature Series, grün = Qualified, lila = Restored (S/M/E). Hier ist 9.8 die faktische Standard-Topnote.
Quellen: [gocollect.com/blog/label-color-madness](https://gocollect.com/blog/label-color-madness-a-breakdown-of-the-different-cgc-labels), [mycomicscollection.com/en/blog/cgc-label-colors-meaning](https://mycomicscollection.com/en/blog/cgc-label-colors-meaning/)

**SGC** — Cert-Nummer auf dem goldenen Label, zusätzlich QR-Code am Holder.
Quelle: [packz.io/blog/sgc-cert-lookup-guide](https://www.packz.io/blog/sgc-cert-lookup-guide)

**WATA** — Neue Holder (ab Juni 2023): 9-stellige Cert-Nummer **unten links auf dem Front-Label**; Legacy-Holder (vor 06/2023): Nummer auf der **Rückseite links neben der „Grade Details"-Box**. WATA vergibt **zwei getrennte Werte**: Box-Note auf 10-Punkte-Skala (realistische Obergrenze 9.8) **und** ein Seal Rating **C … A++**. Ein Parser, der nur eine Zahl liest, verliert die halbe Information.
Quellen: [blog.watagames.com/2023/10/17/cert-verification](https://blog.watagames.com/2023/10/17/cert-verification/), [ballerstatus.com/collectibles/wata-vs-vga-grading-retro-video-games](https://ballerstatus.com/collectibles/wata-vs-vga-grading-retro-video-games/)

**VGA / CGA** — Seriennummer auf dem Grade-Label; Sicherheitsmerkmale: Microprinting, UV-Farbe, QR-Code.
Quellen: [cgagrading.com/blog/serial-verification-](https://www.cgagrading.com/blog/serial-verification-/), [vgagrading.com](https://vgagrading.com/)

### 1.3 Cert-Nummern-Formate (für Regex-Validierung)

| Grader | Format laut Quelle | Belastbarkeit |
|---|---|---|
| PSA | 8–10 Ziffern, rein numerisch | Drittquellen einig, **keine PSA-Primärquelle gefunden → als Heuristik behandeln** |
| BGS/Beckett | alphanumerisch, ~6–15 Zeichen, auch im Barcode codiert | Drittquelle ([cardgrade.io/tools/bgs-cert-lookup](https://cardgrade.io/tools/bgs-cert-lookup)) |
| CGC Cards | 7–12 Ziffern auf dem Front-Label | [cgccards.com](https://www.cgccards.com/CERTLOOKUP/card-grading/grading-scale/) |
| CGC Comics | typisch 10 Ziffern, teils mit kurzem Präfix | Drittquelle |
| SGC | moderne Black Labels: 7 Ziffern; alte grüne/weiße Labels: 11 Zeichen **inklusive Bindestrich** `XXXXXXX-XXX` | [packz.io](https://www.packz.io/blog/sgc-cert-lookup-guide) |
| WATA | 9 Ziffern | [blog.watagames.com](https://blog.watagames.com/2023/10/17/cert-verification/) |
| VGA/CGA | Seriennummer, Format nicht öffentlich dokumentiert | **unbestätigt** |

**Praxisregel:** Format-Validierung nur als *Plausibilitäts-Gate* nutzen (verwirft OCR-Halluzinationen wie 6-stellige PSA-Nummern), niemals als Beweis. Der einzige Beweis ist der Lookup.

---

## 2. Verifikations-Endpunkte / Cert-Lookups — was es wirklich gibt

### 2.1 PSA Public API — der einzige dokumentierte offizielle Weg

- Base: `https://api.psacard.com/publicapi`
- Endpunkt: `GET /cert/GetByCertNumber/{certNumber}`
- Auth: `Authorization: Bearer <token>`, Token im PSA-Konto erzeugbar, **läuft nicht ab**, jederzeit neu generierbar
- Antwortfelder u. a.: `CertNumber`, `Subject`, `CardGrade`, `YearIssued`, `Brand`, `Variety`, `CardAttributes`, `ImageURL`, `IsValidRequest`, `ServerMessage`; `TotalPopulation` / `PopulationHigher` kommen **immer `null`**
- **Free Tier: 100 Calls/Tag**, darüber HTTP 429; höhere Kontingente nur nach direkter Absprache mit PSA (Preise nicht öffentlich)
- Bilder erst für Certs **ab Oktober 2021** vorhanden
- Keine Pop-Reports, keine Preise, keine Katalogsuche — **nur Lookup per Cert-Nummer**

Quellen: [psacard.com/publicapi/documentation](https://www.psacard.com/publicapi/documentation), [api.psacard.com/publicapi/swagger/ui/index](https://api.psacard.com/publicapi/swagger/ui/index), [github.com/maccann-24/sports-card-research – 02-PSA-API.md](https://github.com/maccann-24/sports-card-research/blob/master/02-PSA-API.md), [github.com/brad-newman/fetch-psa-api](https://github.com/brad-newman/fetch-psa-api)

> ⚠️ **Skalierungs-Realität für „zehntausende Nutzer":** 100 Calls/Tag reichen nicht. Konsequenz: (a) **Cache jeden Cert-Lookup permanent** — eine Cert-Nummer ist unveränderlich, das ist ein perfekt cachebarer Schlüssel; (b) Lookup nur auslösen, wenn Vision-Konfidenz niedrig ist ODER der Wert eine Schwelle überschreitet; (c) mit PSA über ein kommerzielles Kontingent sprechen, bevor die App skaliert.

### 2.2 Alle anderen Grader: kein dokumentierter API-Zugang

- **Beckett**: Web-Formular [beckett.com/grading/card-lookup](https://www.beckett.com/grading/card-lookup), Autogramme separat [beckett-authentication.com/verify-certificate](https://www.beckett-authentication.com/verify-certificate). Keine öffentliche API auffindbar → **unbestätigt, ob eine Partner-API existiert**; müsste direkt angefragt werden.
- **CGC**: Lookup unter `cgccards.com/certlookup/…`, Comics `cgccomics.com/certlookup/{cert}/`, Videospiele [cgcvideogames.com/en-US/cert-lookup](https://www.cgcvideogames.com/en-US/cert-lookup). Entwickler fragen im CGC-Forum seit Jahren nach API-Zugang ([boards.cgccomics.com Thread 541624](https://boards.cgccomics.com/topic/541624-api-for-accessing-card-and-image-data-from-cert-number/) — Seite lieferte mir 403, Inhalt nur aus Suchtreffer-Snippet). **Keine öffentliche API belegbar.**
- **SGC / WATA / VGA-CGA**: nur Web-Lookups.

> **Rechtlicher Hinweis:** Die Cert-Lookup-Seiten von CGC/Beckett lieferten meinen Fetch-Versuchen HTTP 403 — sie haben aktiven Bot-Schutz. Scraping dieser Seiten für zehntausende Nutzer ist technisch fragil und ToS-seitig riskant. Ich empfehle es **nicht** als Produktionspfad.

### 2.3 Kommerzielle Aggregatoren (der realistische Weg für Nicht-PSA)

**Ximilar Collectibles Recognition** — deckt exakt Svens Use Case ab und hat als einziger Anbieter öffentliche, konkrete API-Doku:
- `/v2/slab_id` — Slab-Label-Analyse mit OCR
- `/v2/slab_grade` — schnelle Erkennung von Grading-Firma + Note
- `/v2/tcg_id`, `/v2/sport_id`, `/v2/comics_id` (Comics/Magazine/**Manga**), `/v2/card_ocr_id`, `/v2/analyze`
- Liefert `Company` (PSA, BECKETT, CGC, SGC, ACE, MANA, TAG), `Grade`, **Zertifikatsnummer**, „verbal grade", `Side` (Vorder-/Rückseite) und ein Tag **`Graded: yes/no`** — genau die raw-vs-graded-Unterscheidung aus deiner Frage (4)
- Unterstützt 15+ TCGs inkl. **One Piece**, plus Comics/Manga, Banknoten, Briefmarken, Münzen
- Preis: Credit-Modell, Free-Plan 1.000–3.000 Credits/Monat (Angaben je Quelle uneinheitlich), Business ab **59 €/Monat für 100 k Credits**; Slab-Label-Analyse und Grading kosten **zusätzliche** Credits; Abrechnung **pro erkannter Karte**, nicht pro Bild
Quellen: [docs.ximilar.com/collectibles/recognition](https://docs.ximilar.com/collectibles/recognition), [ximilar.com/pricing](https://www.ximilar.com/pricing/), [ximilar.com/blog/get-an-ai-powered-trading-card-price-checker-via-api](https://www.ximilar.com/blog/get-an-ai-powered-trading-card-price-checker-via-api/)

**GemRate Partner API** — vereinheitlichte **Population-Daten** über PSA, Beckett, SGC, CGC, täglich aktualisiert. Kein Cert-Lookup, aber relevant für Seltenheits-/Preis-Plausibilisierung. Preise nur auf Anfrage. [gemrate.com/partner](https://www.gemrate.com/partner)

---

## 3. OCR / Vision für Slab-Labels — was praktisch funktioniert

### 3.1 Reihenfolge der Verfahren (nach Zuverlässigkeit)

1. **QR-Code decodieren.** PSA (seit 2020), CGC und SGC drucken QR-Codes, die direkt auf die Verifikationsseite zeigen — die Cert-Nummer steckt in der URL. Kein OCR-Risiko, keine Halluzination. Bibliotheken: `pyzbar`, `opencv.QRCodeDetector`, ZXing. **Das ist der mit Abstand billigste und sicherste Kanal und fehlt in der aktuellen Pipeline vollständig.**
2. **Barcode decodieren.** PSA und BGS codieren die Cert-Nummer im Barcode ([cardgrade.io/tools/bgs-cert-lookup](https://cardgrade.io/tools/bgs-cert-lookup), [figoca.com](https://figoca.com/blog/psa/psa-cert-checker)).
3. **Multimodales LLM (Claude) auf dem entzerrten Label-Crop.** Für die semantischen Felder (Set, Kartenname, Sprache, Subgrades, Label-Farbe) klar überlegen, weil es Layout und Semantik gemeinsam versteht und mit Reflexionen robust umgeht.
4. **Klassisches OCR als Kreuzprüfung** für die Cert-Nummer. VLM-basierte OCR ist 2026 deutlich vorn: PaddleOCR-VL-1.5 erreicht 94,5 % auf OmniDocBench v1.5; klassische Pipelines (Tesseract/EasyOCR/PaddleOCR) arbeiten stufenweise und kaskadieren Fehler, sind dafür aber billig und lokal. Tesseract ist am schnellsten (CPU), PaddleOCR präziser aber GPU-hungrig (~120 S./min auf RTX 3090 vs. 25 S./min Tesseract-CPU).
Quellen: [codesota.com/ocr/paddleocr-vs-tesseract](https://www.codesota.com/ocr/paddleocr-vs-tesseract), [imagetotable.ai/blog/best-open-source-ocr-tools-2026](https://imagetotable.ai/blog/best-open-source-ocr-tools-2026), [intuitionlabs.ai/articles/non-llm-ocr-technologies](https://intuitionlabs.ai/articles/non-llm-ocr-technologies)

> **Empfehlung:** Zwei-Kanal-Prinzip für die Cert-Nummer. Kanal A = QR/Barcode, Kanal B = LLM-Ablesung. Stimmen sie überein → hohe Konfidenz, kein PSA-Call nötig. Weichen sie ab → PSA-Lookup entscheidet. So bleibt das 100/Tag-Budget für die wirklich unsicheren Fälle.

### 3.2 Vorverarbeitung gegen Reflexionen auf Plastik

- **Physikalisch schlägt algorithmisch:** Ein Polarisationsfilter ist numerisch nicht simulierbar; gleichmäßige, diffuse Beleuchtung ist das wirksamste Mittel. In einer Endkunden-App nicht erzwingbar → stattdessen **Capture-Guidance**: Live-Hinweis „Slab leicht kippen, Lichtquelle nicht spiegeln", und **Mehrfachaufnahme aus 2–3 leicht verschiedenen Winkeln** mit Auswahl/Fusion des glanzfreiesten Label-Bereichs. Das ist die App-taugliche Variante des Multi-Pose-Illumination-Prinzips (US-Patent 9,619,686).
- **Glanzdetektion:** Glanzpixel sind **hell + entsättigt** — Schwellwert in HSV auf hohem V und niedrigem S liefert eine brauchbare Maske ([amphident.de – Removing Glare](http://www.amphident.de/en/blog/preprocessing-for-automatic-pattern-identification-in-wildlife-removing-glare.html)).
- **Inpainting** (OpenCV Navier-Stokes / Fast-Marching) repariert **nur kleine** Glanzflächen; großflächiger Glanz ist damit nicht zu retten ([stackgod – Glare removal with inpainting](https://stackgod.blogspot.com/2021/06/glare-removal-with-inpaintingopencv.html)).
- **Textbewusste Verfahren** existieren als Forschung: *Text-Aware Single Image Specular Highlight Removal* ([arXiv:2108.06881](https://arxiv.org/pdf/2108.06881)), *M2-Net* ([arXiv:2207.09965](https://arxiv.org/pdf/2207.09965)). Für die Produktion eher als Fallback interessant, nicht als Basis.
- **Perspektivische Entzerrung:** Slabs sind starre Rechtecke mit bekanntem Seitenverhältnis → Ecken finden, `cv2.getPerspectiveTransform` + `warpPerspective`. **Das existiert bei Sven bereits** (`/Users/smorty/ebay-bot/web/cardscan.py:154-200`, `slab_recut()` ab `:287` mit Plausibilitätsprüfung auf Hochkant-Rechteck bei `:311-322`). Der fehlende Schritt ist ein **zweiter, engerer Crop nur auf das Label-Band** — das Label ist ein schmaler horizontaler Streifen am oberen Rand, und ein Label-Crop mit 4–6× mehr Pixeln pro Zeichen verbessert sowohl OCR als auch LLM-Ablesung drastisch.

---

## 4. „Graded vs. lose Karte vs. Toploader/Sleeve" zuverlässig unterscheiden

Sven hat diese Klassifikation bereits als Vision-Klasse (`kind ∈ {slab, sleeve, raw}` in `web/cardscan.py:71-79,101`). Zur Härtung:

1. **Label-Band als Primärmerkmal.** Ein Slab hat ein **opakes, bedrucktes Band** mit Text am oberen Rand; ein Toploader/Sleeve ist über die gesamte Fläche transparent. Das ist das robusteste einzelne Signal — deutlich robuster als Dicke oder Kantenoptik.
2. **Seitenverhältnis.** Karte 2,5″ × 3,5″ (≈ 0,714). PSA-Slab ca. 86 mm breit × 68 mm tief × 10 mm; BGS Flip Type 1 ca. 87 mm × 15 mm Tiefe; CGC/CSG ca. 86 mm × 12 mm — BGS-Slabs sind spürbar dicker als PSA. Es gibt **keinen Industriestandard** für Slab-Maße, das Verhältnis ist also nur ein weiches Signal. Quellen: [mycollex.com/slab-sizing-library](https://www.mycollex.com/slab-sizing-library), [quirkshelv.com – Slab Sizes](https://www.quirkshelv.com/blogs/news/top-8-trading-card-grading-companies-slab-sizes-standard-size-guide)
3. **Positive Evidenz erzwingen.** Regel: `graded = true` **nur**, wenn (Grader-Logo/Name erkannt) **UND** (plausible Note) **UND** (plausible Cert-Nummer ODER QR/Barcode dekodiert). Fehlt eines davon → `graded = null` und Fallback auf Raw-Preis, statt zu raten. Der bestehende Prompt-Satz „Eine LOSE Karte ist NIEMALS graded" (`bot/claude_client.py:85`) geht in die richtige Richtung, ist aber nur eine Instruktion, keine Validierung im Code.
4. **Fertige Alternative:** Ximilar liefert das Tag `Graded: yes/no` plus `Side` (Vorder-/Rückseite) direkt — als unabhängige zweite Meinung für Grenzfälle nutzbar.
5. **Sleeve/Toploader ≠ Zustand.** Ein Toploader sagt **nichts** über den Kartenzustand aus. Er darf keinen Preisaufschlag und keinen „graded"-Pfad auslösen.

---

## 5. Bekannte Fallstricke

### 5.1 Item-Klasse — die eigentliche Ursache des 603-€-Fehlers
Das Slab-Label enthält die Klasse implizit: Comic-/Manga-Label = *Titel + Heftnummer + Erscheinungsdatum + Verlag*; Karten-Label = *Jahr + Set + Spieler/Kartenname + Kartennummer*. Ein Manga „One Piece Vol. 103" darf nicht gegen einen TCG-Index gematcht werden, wo „One Piece OP01-016 Nami" existiert. **Empfehlung:** `item_class ∈ {tcg_card, sport_card, comic_manga, video_game, coin, other}` als *hartes Routing-Feld* aus dem Label ableiten und in `web/pricecharting.py` / `web/prices.py` als Filter durchreichen. Zusätzlich ein **Match-Gate**: wenn der Kandidat der Preisquelle eine andere Klasse hat als das Label, wird der Treffer verworfen — nicht bepreist.

### 5.2 Skalen sind nicht ineinander umrechenbar
- **BGS 9.5 ≈ PSA 10** in der Sache; PSA 10 erzielt dennoch regelmäßig höhere Preise (Markenprämie). BGS 9 ≈ PSA 8–9. PSA 10 erlaubt Zentrierung bis 60/40 vorn, BGS Black Label verlangt 50/50.
  Quellen: [cardlines.com/bgs-10-vs-psa-9-5](https://cardlines.com/bgs-10-vs-psa-9-5/), [cardgrader.ai/blog/card-grading-scale-explained](https://cardgrader.ai/blog/card-grading-scale-explained)
- **CGC Cards kennt kein 9.8** — 9.8/9.9 sind CGC **Comics**. Ein „CGC 9.8" bei einer Karte ist ein Erkennungsfehler oder ein Comic.
- **BCCG-Noten sind typischerweise 2–3 Stufen zu hoch** ([cardgrading.app](https://cardgrading.app/beckett-grading-system)). Ein BCCG 10 darf **niemals** wie ein BGS 10 bepreist werden.
- **WATA hat zwei Achsen** (Box-Note + Seal Rating C…A++). Nur die Zahl zu übernehmen verliert den wertbestimmenden Teil.

### 5.3 Granularität der Preisquelle
PriceCharting kennt bei Karten nur **9 / 9.5 / 10** — der Code weiß das und interpoliert (`web/pricecharting.py:5-6,49-73,107-112`). Deshalb landen Beckett 9.0 und 9.4 im selben Feld. Sauberer Umgang: bei Zwischennoten (9.4, 9.2, 8.5 …) ausdrücklich als **Schätzung mit Spanne** ausweisen statt als Punktpreis, und die Note **abrunden** statt zu interpolieren, wenn die Datenbasis dünn ist.

### 5.4 Subgrades
BGS-Subgrades sind vier separate Zahlen. Wird nur „die größte Zahl auf dem Label" gelesen, entsteht systematischer Unsinn (eine 10 in „Corners" bei Gesamtnote 9). **Regel:** Gesamtnote ist die typografisch größte/hervorgehobene Zahl; Subgrades erscheinen als Vierergruppe mit Labels C/E/CO/S. Immer alle fünf Werte gemeinsam extrahieren oder gar keinen.

### 5.5 Japanisch vs. Englisch
Japanische Sets haben **eigene Set-Codes und Symbole**, die *nicht* 1:1 auf englische Sets abbilden; Promo-Nummerierung unterscheidet sich strukturell (`SM103` vs. `005/SV-P`). Kartennummer und HP sind sprachunabhängig ablesbar, aber der Set-Schlüssel ist es nicht → **eigene Katalogzweige je Sprache** und `language` als Teil des `card_key` in `web/catalog.py`. Japanische Karten notieren am Markt anders als englische — ein sprachagnostischer Preis ist grundsätzlich falsch.
Quellen: [tcg-placeholder.com – Set-Codes](https://tcg-placeholder.com/en-US/blog/how-to-identify-pokemon-card-sets-complete-guide), [tallytcg.com – Japanese Pokémon cards](https://tallytcg.com/blogs/news/how-to-read-a-japanese-pokemon-card-a-beginner-s-guide)

### 5.6 Nebenbefund zur eBay-Datenlage (war als „zu prüfen" markiert)
Die Vermutung ist **bestätigt**: `findCompletedItems` wurde am **15.10.2020** eingeschränkt/deprecated, die **Finding API insgesamt im Januar 2024 deprecated und im Februar 2025 abgeschaltet**. Verkaufte Artikel sind offiziell nur noch über die **Marketplace Insights API** erreichbar, die eine **Limited Release** ist, Business-Level-Freigabe verlangt und laut eBay-Doku aktuell **nicht für neue Nutzer geöffnet** ist. Die Nutzung von 130point in `web/sold.py` ist damit sachlich begründet — aber ein dauerhaftes Klumpenrisiko ohne Vertrag.
Quellen: [developer.ebay.com – API Deprecation Status](https://developer.ebay.com/develop/get-started/api-deprecation-status), [community.ebay.com – Finding & Shopping API decommissioned](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062)

---

## 6. Empfohlene Pipeline (priorisiert, ohne manuellen Eingriff)

### P0 — verhindert falsche Preise, geringer Aufwand
1. **`item_class` aus dem Label ableiten und als hartes Routing-Feld durchreichen.** Preisquelle nur innerhalb der Klasse abfragen. Match-Gate: Klassen-Mismatch → Treffer verwerfen, nicht bepreisen. *(Direkte Ursache des 603-€-Falls.)*
2. **Grader-Kürzel-Tabelle erweitern** um `BVG`, `BCCG`, `BAS`, `CBCS`, `TAG`, `CSG`, `MANA` und **`beckett` nicht mehr blind auf `BGS`** mappen, sondern anhand des Labeltyps entscheiden (`bot/claude_client.py:139-142`).
3. **Positive-Evidenz-Regel für `graded`**: Grader + Note + (Cert-Nummer ODER Code) — sonst `graded = null`.
4. **Zwischennoten nicht interpolieren, sondern als Spanne ausgeben**, wenn die Quelle nur 9/9.5/10 kennt (`web/pricecharting.py:49-73`).

### P1 — größter Genauigkeitsgewinn pro Aufwand
5. **QR-/Barcode-Decoder** auf den entzerrten Slab-Crop (pyzbar/ZXing). Trifft PSA, CGC, SGC und BGS-Barcodes.
6. **Zweiter Crop nur auf das Label-Band**, dann LLM-Ablesung auf diesem hochaufgelösten Ausschnitt statt auf dem Gesamtbild.
7. **Format-Validierung der Cert-Nummer je Grader** als Halluzinationsfilter (Tabelle 1.3), inklusive SGC-Sonderfall mit Bindestrich.

### P2 — der Königsweg, wo er verfügbar ist
8. **PSA Public API integrieren**, mit **permanentem Cache pro Cert-Nummer** (Cert-Daten sind unveränderlich) und Aufruf nur bei niedriger Konfidenz oder hohem Wert. Free-Tier 100/Tag ist die harte Grenze — vor dem Skalieren mit PSA über ein Kontingent sprechen.
9. **Für Nicht-PSA-Slabs: Ximilar `/v2/slab_id`** als zweite Meinung bzw. als Primärkanal. Deckt BECKETT, CGC, SGC, TAG, ACE, MANA ab, liefert Cert-Nummer und `Graded`-Tag, unterstützt auch Comics/Manga — also genau die Lücke des PSA-Wegs.

### P3 — Qualität und Skalierung
10. **Capture-Guidance in der PWA**: Winkel-Hinweis gegen Spiegelungen, optional 2–3 Frames und Auswahl des glanzärmsten Label-Bereichs (HSV-Glanzmaske als Score).
11. **Sprachzweig im Katalog** (`language` als Teil des `card_key`), getrennte JP/EN-Preisreihen.
12. **Konfidenz-Gate statt Rückfrage:** Bei Konfidenz unter Schwelle *kein* Listing erzeugen, sondern Preis als Spanne + Hinweis. Das erhält den „kein manueller Eingriff"-Anspruch, ohne falsche Punktpreise zu produzieren.
13. **Optional: GemRate Partner API** für Pop-Daten zur Plausibilisierung (ein PSA-10 mit Pop 3 verhält sich preislich anders als einer mit Pop 40.000).

---

## 7. Was ich NICHT belegen konnte (unbestätigt)

- PSA-Cert-Nummernlänge aus einer PSA-Primärquelle (nur Drittanbieter, 8–10 Ziffern).
- Semantik der PSA-Labelfarben (Red/Gold/Black) aus PSA-Primärquelle.
- Existenz irgendeiner Partner-/Enterprise-API bei **CGC**, **Beckett**, **SGC**, **WATA**, **VGA** — weder bestätigt noch widerlegt; müsste direkt angefragt werden.
- Konkrete Preise der PSA-Paid-Tiers (nicht öffentlich).
- Exakte Ximilar-Credit-Kosten pro `slab_id`-Call (Doku nennt nur „zusätzliche Credits"; das Free-Kontingent wird in zwei Quellen mit 1.000 bzw. 3.000 Credits/Monat angegeben).
- VGA/CGA-Seriennummernformat.

---

## 8. Referenzierte Dateien im Projekt (nur gelesen, nichts verändert)

- `/Users/smorty/ebay-bot/bot/claude_client.py:85` — `graded_info`-Feld im Vision-Prompt (grader/grade/cert_number)
- `/Users/smorty/ebay-bot/bot/claude_client.py:139-161` — `_GRADER_KUERZEL` + Titel-Korrektur, unvollständige Kürzelliste
- `/Users/smorty/ebay-bot/web/cardscan.py:71-79,101,118-120` — Klassifikation `slab/sleeve/raw` und Crop-Margins
- `/Users/smorty/ebay-bot/web/cardscan.py:154-200` — Konturensuche/Rechteck-Reparatur
- `/Users/smorty/ebay-bot/web/cardscan.py:287-337` — `slab_recut()` mit Plausibilitätsprüfungen
- `/Users/smorty/ebay-bot/web/pricecharting.py:5-6,21-45,49-73,107-114` — Grade→Preisfeld-Mapping und Zwischenstufen-Interpolation
- `/Users/smorty/ebay-bot/web/catalog.py`, `/Users/smorty/ebay-bot/web/prices.py`, `/Users/smorty/ebay-bot/web/sold.py` — Katalogschlüssel bzw. Preisquellen

---

## Quellen

- [PSA Public API Documentation](https://www.psacard.com/publicapi/documentation) · [PSA Swagger](https://api.psacard.com/publicapi/swagger/ui/index) · [sports-card-research 02-PSA-API.md](https://github.com/maccann-24/sports-card-research/blob/master/02-PSA-API.md) · [fetch-psa-api](https://github.com/brad-newman/fetch-psa-api)
- [Beckett Graded Card Lookup](https://www.beckett.com/grading/card-lookup) · [Beckett Authentication Verify](https://www.beckett-authentication.com/verify-certificate) · [allvintagecards – Beckett Guide](https://allvintagecards.com/beckett-card-grading-guide/) · [cardgrading.app – Beckett System](https://cardgrading.app/beckett-grading-system) · [GradingMetric – BGS](https://www.gradingmetric.com/bgs-card-grading) · [zeropop – BGS Subgrades](https://zeropop.app/blog/bgs-subgrades-explained) · [Card Centering Tool – BGS](https://www.cardcenteringtool.com/grading-standards/bgs)
- [CGC Cards Grading Scale](https://www.cgccards.com/card-grading/grading-scale/) · [CGC Video Games Cert Lookup](https://www.cgcvideogames.com/en-US/cert-lookup) · [GoCollect – CGC Label Colors](https://gocollect.com/blog/label-color-madness-a-breakdown-of-the-different-cgc-labels) · [mycomicscollection – CGC Label Colors](https://mycomicscollection.com/en/blog/cgc-label-colors-meaning/)
- [packz.io – SGC Cert Lookup Guide](https://www.packz.io/blog/sgc-cert-lookup-guide)
- [WATA – Cert Verification](https://blog.watagames.com/2023/10/17/cert-verification/) · [BallerStatus – WATA vs VGA](https://ballerstatus.com/collectibles/wata-vs-vga-grading-retro-video-games/) · [CGA – Serial Verification](https://www.cgagrading.com/blog/serial-verification-/) · [VGA Grading](https://vgagrading.com/)
- [MasterGrade – PSA Label Guide](https://mastergrade.ai/blog/psa-label-guide) · [PreGradeCards – PSA Cert Number](https://pregradecards.com/blog/psa-certificate-number-verify-authenticity) · [figoca – PSA Cert Checker](https://figoca.com/blog/psa/psa-cert-checker)
- [Ximilar API Docs – Collectibles Recognition](https://docs.ximilar.com/collectibles/recognition) · [Ximilar Pricing](https://www.ximilar.com/pricing/) · [Ximilar – Card Price Checker via API](https://www.ximilar.com/blog/get-an-ai-powered-trading-card-price-checker-via-api/) · [GemRate Partner API](https://www.gemrate.com/partner)
- [CodeSOTA – PaddleOCR vs Tesseract](https://www.codesota.com/ocr/paddleocr-vs-tesseract) · [imagetotable – Best OSS OCR 2026](https://imagetotable.ai/blog/best-open-source-ocr-tools-2026) · [IntuitionLabs – Non-LLM OCR](https://intuitionlabs.ai/articles/non-llm-ocr-technologies)
- [Glare removal with inpainting (OpenCV)](https://stackgod.blogspot.com/2021/06/glare-removal-with-inpaintingopencv.html) · [AmphIdent – Removing Glare](http://www.amphident.de/en/blog/preprocessing-for-automatic-pattern-identification-in-wildlife-removing-glare.html) · [arXiv 2108.06881 – Text-Aware Specular Highlight Removal](https://arxiv.org/pdf/2108.06881) · [arXiv 2207.09965 – M2-Net](https://arxiv.org/pdf/2207.09965) · [US 9,619,686 – Multi-pose illumination vs. glare](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9619686)
- [COLLEX – Slab Sizing Library](https://www.mycollex.com/slab-sizing-library) · [QuirkShelv – Slab Sizes](https://www.quirkshelv.com/blogs/news/top-8-trading-card-grading-companies-slab-sizes-standard-size-guide)
- [Cardlines – BGS 9.5 vs PSA 10](https://cardlines.com/bgs-10-vs-psa-9-5/) · [CardGrader – Grading Scale Explained](https://cardgrader.ai/blog/card-grading-scale-explained)
- [tcg-placeholder – Pokémon Set-Codes](https://tcg-placeholder.com/en-US/blog/how-to-identify-pokemon-card-sets-complete-guide) · [TallyTCG – Japanese Pokémon Cards](https://tallytcg.com/blogs/news/how-to-read-a-japanese-pokemon-card-a-beginner-s-guide)
- [eBay – API Deprecation Status](https://developer.ebay.com/develop/get-started/api-deprecation-status) · [eBay Community – Finding/Shopping API decommissioned](https://community.ebay.com/t5/Traditional-APIs-Search/Alert-Finding-API-and-Shopping-API-to-be-decommissioned-in-2025/td-p/34222062)