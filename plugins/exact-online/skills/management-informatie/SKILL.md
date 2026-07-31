---
name: management-informatie
description: >
  Deze skill moet gebruikt worden bij vragen om commerciële stuurinformatie uit Exact
  Online: dashboard, KPI's, omzet per klant of categorie, marge, cashflow-positie of een
  periodevergelijking. Triggers: 'management informatie', 'dashboard', 'KPI',
  'stuurinformatie', 'omzet per klant', 'klantconcentratie', 'groeiers en dalers',
  'recurring revenue', 'marge', 'cashflow', 'vergelijken met vorig jaar', 'QoQ', 'YoY'.
  Een formele resultatenrekening hoort bij resultatenrekening-analyse.
---

# Management Informatie

Actuele stuurinformatie voor ondernemers, zonder Excel-exports. De ingebouwde rapportage van
Exact Online is goed voor de boekhouder, maar te statisch voor de ondernemer die dagelijks
wil weten hoe het bedrijf ervoor staat.

## Databronnen

### Fundamentele regel: bedragen altijd via TransactionLines

Voor alle omzet-, kosten- en bedraganalyses geldt: gebruik `analyze_data` op
`Financial/TransactionLines` als primaire databron, met een JOIN op `Financial/GLAccounts`
om te filteren op rekeningtype.

TransactionLines bevat de daadwerkelijk geboekte bedragen (excl. BTW, incl.
creditnota-correcties). SalesInvoices bevat bruto factuurbedragen die niet aansluiten op de
boekhouding en mist memoriaalboekingen en correcties. Gebruik SalesInvoices alleen voor
metadata die niet in TransactionLines zit, zoals factuur-aantallen, en behandel die bedragen
als indicatief.

### Toolkeuze

| Doel | Tool | Tabel |
|------|------|-------|
| Omzet- en kostenaggregaties | `analyze_data` | `Financial/TransactionLines` + JOIN `Financial/GLAccounts` |
| Periodevergelijkingen (QoQ, YoY) | `analyze_data` | `Financial/TransactionLines` + JOIN `Financial/GLAccounts` |
| Klant-niveau omzet | `analyze_data` | `Financial/TransactionLines` (veld `AccountName`) |
| Factuur-aantallen (metadata) | `analyze_data` | `SalesInvoice/SalesInvoices` |
| Grootboeksaldi per periode | `read_operation` | `Financial/ReportingBalance` |
| Cashflow, debiteuren | `read_operation` | `Read/ReceivablesList` |
| Cashflow, crediteuren | `read_operation` | `Bulk/Cashflow/Payments` |

`read_operation` doet alleen GET en geeft maximaal 60 records per call. Op Bulk-endpoints
werkt `skip` niet: pagineer daar met `page_token` uit `next_page.page_token`.

### Beschikbaarheid en abonnement

`analyze_data` en `list_available_tables` bestaan alleen op het Trial- en
Analytics-abonnement. Op het Essentials-abonnement zijn de modules 3 tot en met 6 hieronder
niet uitvoerbaar zoals beschreven: val dan terug op REST-rapportages via `read_operation`
(`Financial/ReportingBalance` per periode) en meld de gebruiker dat de aggregaties beperkter
zijn.

Check vóór gebruik met `list_available_tables` of de tabel gesynchroniseerd is. Bij
`sync_status = syncing` loopt de import nog en kunnen resultaten incompleet zijn: waarschuw
dan de gebruiker.

### GLAccount.Type: filter altijd op type, nooit op code-ranges

Rekeningnummers variëren per administratie. Sommige administraties hebben omzet op 8000+,
andere op 4000+. Filter daarom altijd op `GLAccount.Type` via de JOIN:

| GLAccount.Type | Categorie |
|----------------|-----------|
| 110 | Omzet |
| 111 | Kostprijs omzet (COGS) |
| 120 | Overige kosten |
| 121 | Verkoop-, algemene en beheerkosten (SG&A) |
| 122 | Afschrijvingskosten |
| 123 | Research en development |
| 125 | Personeelskosten (lonen, sociale lasten, pensioen) |
| 126 | Overige personeelsgebonden kosten (huisvesting, transport, kantoor) |
| 130 | Bijzondere lasten |
| 140 | Bijzondere baten |
| 150 | Belasting over het resultaat |
| 160 | Rentebaten en -lasten |

Kosten in brede zin zijn dus `t1.Type in (111, 120, 121, 122, 123, 125, 126)`. Filter je
alleen op `BalanceType = 'W'`, dan tel je omzet en kosten door elkaar op.

**Meerdere boekjaren:** voeg altijd `t0.Type != 310` toe. De jaarafsluiting boekt elke
W&V-rekening van een afgesloten jaar tegen zichzelf terug, waardoor dat jaar zonder dit
filter op ongeveer nul uitkomt en elke Δ-berekening onzin wordt.

---

## Aanpak

Start met een intake-vraag om te bepalen wat de ondernemer wil:

> "Welk inzicht heeft u nodig? Ik kan direct ophalen:
> 1. **Cashflow positie**, liquiditeit en verwachte in- en uitstroom
> 2. **Omzet-analyse**, uitsplitsing per klant, categorie of afdeling
> 3. **Marge-analyse**, bruto marge en kostprijzen
> 4. **Vergelijking**, QoQ, MoM of YoY
> 5. **Commerciële KPI's**, recurring vs eenmalig, klantconcentratie, groeiers en dalers"

Meerdere keuzes combineren is mogelijk.

---

## Module 1: formele resultatenrekening (doorverwijzing)

Vraagt de gebruiker om een resultatenrekening, W&V, P&L of een jaarrekening-model, gebruik
dan de skill **`resultatenrekening-analyse`**. Die bouwt de volledige structuur conform BW2
Titel 9, inclusief de juiste teken-conventies, tussentellingen en RGS-uitsplitsing.

Bouw hier geen tweede, afwijkende P&L: dat levert per definitie andere cijfers op dan de
skill die er wel voor bedoeld is.

---

## Module 2: cashflow-positie

### Stap 1: huidige liquiditeit

Bepaal de bank- en kasrekeningen via het grootboektype, niet via een codebereik. Liquide
rekeningen zijn `GLAccount.Type` in {10 = Cash, 12 = Bank, 14 = Credit card, 16 = Payment
services}.

Met `read_operation`:

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": { "Type": [10, 12, 14, 16] },
  "select": "Code,Description,Type,TypeDescription"
}
```

Voed de gevonden codes daarna als filter in ReportingBalance. Laat `ReportingPeriod` weg,
het banksaldo is cumulatief. Neem hieronder de codes over die stap 1 opleverde en typ nooit
zelf een nummer of een nummerbereik: welke nummers een administratie voor bank en kas
gebruikt ligt niet vast.

Met `read_operation`:

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "filters": {
    "GLAccountCode": ["<code1 uit stap 1>", "<code2 uit stap 1>", "<...>"],
    "ReportingYear": <jaar>
  },
  "select": "GLAccountCode,GLAccountDescription,Amount"
}
```

Cumulatief banksaldo is de som van Amount over die rekeningen.

### Stap 2: te ontvangen (debiteuren)

Gebruik `Read/ReceivablesList`, dat bevat het veld `JournalCode` dat nodig is voor correcte
classificatie.

Met `read_operation`:

```json
{
  "service": "Read",
  "entity": "ReceivablesList",
  "select": "AccountName,Amount,JournalCode,JournalDescription,InvoiceNumber,InvoiceDate,DueDate,Description"
}
```

Classificeer altijd op `JournalCode` plus het teken van Amount:

| JournalCode | Amount | Betekenis | Actie |
|---|---|---|---|
| Bankdagboek (bijv. `"20"`, `"23"`) | negatief | Tegoed, ontvangen betaling nog niet afgeletterd | Afletteren |
| Verkoopboek (bijv. `"70"`) | positief | Vordering, openstaande verkoopfactuur | Opvolgen |
| Verkoopboek (bijv. `"70"`) | negatief | Tegoed, openstaande creditnota | Uitbetalen of verrekenen |

Controleer de bankdagboek-codes via `Financial/Journals` met `Type: 12`.

### Stap 3: te betalen (crediteuren)

Met `read_operation`:

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Payments",
  "filters": { "Status": [20, 30] },
  "select": "AccountName,AmountDC,DueDate,InvoiceNumber"
}
```

Groepeer op DueDate-buckets (7d, 8 tot 30d, 31 tot 60d, meer dan 60d) en presenteer:
huidige kasmiddelen plus verwachte in- en uitstroom is de verwachte positie.

---

## Module 3: omzet-analyse

Kernmodule. Gebruik `analyze_data` op `Financial/TransactionLines` met een INNER JOIN op
`Financial/GLAccounts`, gekoppeld op `GLAccountCode` = `Code`.

### Basispatroon: omzet per kwartaal

Met `analyze_data`:

```json
{
  "query": {
    "table": "Financial/TransactionLines",
    "joins": [{
      "table": "Financial/GLAccounts",
      "type": "INNER",
      "on": { "leftColumn": "GLAccountCode", "rightColumn": "Code" },
      "select": ["Code", "Description"]
    }],
    "aggregations": [
      { "function": "SUM", "column": "AmountDC", "alias": "Omzet", "sign": -1 }
    ],
    "filters": [
      { "column": "t1.Type", "operator": "=", "value": 110 },
      { "column": "t0.Type", "operator": "!=", "value": 310 },
      { "column": "Date", "operator": ">=", "value": "2025-01-01" },
      { "column": "Date", "operator": "<=", "value": "2026-03-31" }
    ],
    "dateGroupBy": [
      { "column": "Date", "part": "YEAR", "alias": "Jaar" },
      { "column": "Date", "part": "QUARTER", "alias": "Kwartaal" }
    ],
    "orderBy": [
      { "column": "Jaar", "direction": "ASC" },
      { "column": "Kwartaal", "direction": "ASC" }
    ]
  }
}
```

Omzet staat credit en is dus negatief in TransactionLines. `"sign": -1` op de aggregatie
draait dat direct om, zodat je niet achteraf hoeft te ABS-en. Laat je `sign` weg, gebruik
dan ABS() bij weergave en let op de sorteerrichting (`ASC` = hoogste omzet eerst).

### Varianten op hetzelfde patroon

| Vraag | Aanpassing |
|---|---|
| Omzet per categorie | Voeg `"groupBy": ["t1.Code", "t1.Description"]` toe, geeft bijv. Abonnementen vs Maatwerk vs Overig |
| Omzet per klant | Voeg `"select": ["AccountName"]` en `"groupBy": ["AccountName"]` toe |
| Omzet per maand | Zet `part` in `dateGroupBy` op `MONTH` in plaats van `QUARTER` |
| Omzet per kostenplaats | Voeg `CostCenter` toe aan `select` en `groupBy` |

Kostenplaatsen zijn optioneel per administratie. Controleer eerst met `read_operation` op
`HRM/CostCenters` (`select: "Code,Description,Active"`) of ze zijn ingericht.

---

## Module 4: marge-analyse

Haal omzet (Type 110) en kostprijs (Type 111) in één query op: gebruik het basispatroon uit
Module 3, maar met `{ "column": "t1.Type", "operator": "IN", "values": [110, 111] }` en
`"groupBy": ["t1.Type"]`. Laat `sign` hier weg, zodat omzet negatief en kosten positief
blijven en de tekens onderling vergelijkbaar zijn.

Bruto marge = (ABS(omzet) - inkoopkosten) / ABS(omzet) x 100%.

Sectorbenchmarks staan in `references/benchmarks.md`.

---

## Module 5: vergelijking en trends

Alle vergelijkingen lopen via `analyze_data` op TransactionLines. Het datumfilter bepaalt
welke periodes je vergelijkt:

| Vergelijking | Datumfilter | dateGroupBy |
|---|---|---|
| QoQ (dit vs vorig kwartaal) | Afgelopen 6 maanden | YEAR + QUARTER |
| MoM (deze vs vorige maand) | Afgelopen 2 maanden | YEAR + MONTH |
| YoY (dit vs vorig jaar) | Dit jaar plus vorig jaar | YEAR |
| YTD-vergelijking | Januari tot heden, beide jaren | YEAR + MONTH |

Elk van deze vergelijkingen raakt meer dan één boekjaar zodra de periode een jaargrens
overschrijdt. Neem dan `{ "column": "t0.Type", "operator": "!=", "value": 310 }` op in de
filters, anders staat het afgesloten jaar op ongeveer nul en klopt elke Δ niet.

Voer twee `analyze_data` calls uit: totalen per periode voor de samenvattingstabel, en per
categorie per periode om te zien waar de verandering zit.

Signaleringsdrempels staan in `references/benchmarks.md`.

---

## Module 6: commerciële KPI's

Inzicht in de commerciële gezondheid, voorbij de pure financiële cijfers.

- **Recurring vs eenmalig.** Classificeer de grootboekrekeningen uit de omzet-per-categorie
  query op naam: "Abonnement" is recurring, "Maatwerk", "Project", "Consult" en "Overig" zijn
  eenmalig. Presenteer als "X% van de omzet is recurring", met de trend.
- **Klantconcentratie.** Bereken uit de omzet-per-klant query het aandeel van de top 5 en de
  top 10 in de totale omzet.
- **Groeiers en dalers.** Vergelijk de omzet per klant tussen twee periodes, sorteer op
  absoluut verschil en toon de top 5 groeiers, de top 5 dalers, nieuwe klanten (omzet nu,
  geen omzet in de vorige periode) en verdwenen klanten (omgekeerd).
- **Gemiddeld factuurbedrag.** Een stijgend gemiddelde bij dalend volume duidt op
  verschuiving naar grotere opdrachten.

De bijbehorende signaleringsdrempels staan in `references/benchmarks.md`.

---

## Output en valkuilen

- Rapportsjabloon: `references/rapportformat.md`.
- Bekende API-eigenaardigheden en valkuilen: `references/valkuilen.md`. Lees dit vóór je een
  query bouwt, het dekt de tekens, de JOIN-sleutel, de paginatie en de code-range-val.

## Samenwerking met andere skills

- **Resultatenrekening-analyse**: elke vraag om een formele W&V of P&L.
- **Debiteurenbeheer**: bij signaleren van meer dan 60 dagen openstaand.
- **Cashflow-analyse**: bij diepere liquiditeitsvragen.
- **Periodeafsluiting**: als de cijfers niet kloppen, check of de periode volledig gesloten is.

## Communicatie

- Spreek als financieel adviseur, niet als boekhouder: geef duiding, niet alleen cijfers.
- Geef context bij afwijkingen: wat betekent dit voor het bedrijf?
- Signaleer actief wat aandacht vraagt, de ondernemer heeft niet altijd financiële expertise.
- Gebruik Nederlands bedragformaat (€ 1.234,56) en Nederlandse terminologie.
- Vermeld altijd "stand per [datum]" en de databron ("via analyze_data op TransactionLines").
- Vraag bij eerste gebruik: welk boekjaar, welke periode, en YTD of een specifieke periode?
