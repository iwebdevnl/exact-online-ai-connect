---
name: management-informatie
description: >
  Management dashboard voor ondernemers in Exact Online. P&L, omzet per klant/categorie,
  bruto marge, cashflow, periode-vergelijkingen (QoQ, YoY), en commerciële KPI's.
  Bedragen altijd via analyze_data op TransactionLines, nooit SalesInvoices.
  Triggers: 'management informatie', 'P&L', 'winst en verlies', 'omzet overzicht',
  'omzet per klant', 'marge', 'cashflow', 'dashboard', 'KPI', 'stuurinformatie',
  'hoe staat het er financieel voor', 'wat is mijn omzet', 'financieel overzicht',
  'vergelijken met vorig jaar', 'kwartaalvergelijking', 'QoQ', 'YoY',
  'recurring revenue', 'klantconcentratie', 'groeiers en dalers'.
  Werkt met Exact Online MCP (analyze_data TransactionLines + GLAccounts JOIN,
  ReportingBalance, Cashflow/Receivables, Cashflow/Payments).
---

# Management Informatie

Actuele stuurinformatie voor ondernemers — zonder Excel-exports. De ingebouwde rapportage
van Exact Online is goed voor de boekhouder, maar te statisch voor de ondernemer die
dagelijks wil weten hoe het bedrijf ervoor staat.

## Databronnen — Wanneer welke tool gebruiken

### Fundamentele regel: bedragen altijd via TransactionLines

Voor alle omzet-, kosten- en bedraganalyses geldt: gebruik **`analyze_data`** op
**`Financial/Transactionlines`** als primaire databron, met een JOIN op
**`Financial/GLAccounts`** om te filteren op rekeningtype.

De reden: TransactionLines bevat de daadwerkelijk geboekte bedragen (excl. BTW, incl.
creditnota-correcties). Dit is de enige betrouwbare bron voor financiële analyses.
SalesInvoices bevat bruto factuurbedragen die niet aansluiten op de boekhouding en
mist boekingen die niet via verkoopfacturen lopen (memoriaalboekingen, correcties, etc.).

**SalesInvoices** mag alleen gebruikt worden voor metadata die niet in TransactionLines zit:
klantnamen bij factuurdetails, factuur-aantallen, of als er gefilterd moet worden op
velden die alleen in SalesInvoices bestaan (zoals OrderedByName). Maar zelfs dan: de
bedragen die je uit SalesInvoices haalt zijn indicatief, niet leidend.

### Toolkeuze

| Doel | Tool | Tabel |
|------|------|-------|
| Omzet/kosten aggregaties | `analyze_data` | `Financial/Transactionlines` + JOIN `Financial/GLAccounts` |
| Periode-vergelijkingen (QoQ, YoY) | `analyze_data` | `Financial/Transactionlines` + JOIN `Financial/GLAccounts` |
| P&L / ReportingBalance | `execute_operation` | `Financial/ReportingBalance` |
| Cashflow / debiteuren | `execute_operation` | `Read/ReceivablesList` |
| Cashflow / crediteuren | `execute_operation` | `Bulk/Cashflow/Payments` |
| Factuur-aantallen (metadata) | `analyze_data` | `Salesinvoice/SalesInvoices` |
| Klant-niveau omzet | `analyze_data` | `Financial/Transactionlines` (AccountName veld) |

### Beschikbaarheid controleren

Voordat je `analyze_data` gebruikt, check met `list_available_tables` of de tabel
gesynchroniseerd is. Let op de `sync_status`:
- `idle` = data compleet, veilig te gebruiken
- `syncing` = import bezig, resultaten kunnen incompleet zijn — waarschuw de gebruiker

### GLAccount.Type — filter altijd op type, niet op code-ranges

Rekeningnummers variëren per administratie. Sommige administraties hebben omzet op 8000+,
andere op 4000+. Filter daarom altijd op `GLAccount.Type` via de JOIN:

| GLAccount.Type | Categorie | Beschrijving |
|----------------|-----------|--------------|
| 110 | Omzet | Netto-omzet uit normale bedrijfsactiviteiten |
| 111 | Kostprijs omzet | Inkoop-/directe kosten (COGS) |
| 120 | Overige kosten | Overige bedrijfskosten |
| 121 | Verkoop/algemeen/beheer | Verkoop-, algemene en beheerkosten (SG&A) |
| 122 | Afschrijvingskosten | Afschrijvingen op vaste activa |
| 125 | Personeelskosten | Lonen, sociale lasten, pensioen |
| 130 | Bijzondere lasten | Buitengewone/incidentele lasten |
| 140 | Bijzondere baten | Buitengewone/incidentele baten |
| 150 | Belasting over resultaat | Vennootschapsbelasting op het resultaat |
| 160 | Rentebaten/-lasten | Financiële baten en lasten |

Gebruik deze types als filter in de JOIN, niet hardcoded rekeningnummers.

---

## Aanpak

Start altijd met een **intake-vraag** om te bepalen wat de ondernemer wil:

> "Welk inzicht heeft u nodig? Ik kan direct ophalen:
> 1. **P&L (resultaat)** — omzet, kosten, winst voor een periode
> 2. **Cashflow positie** — liquiditeit en verwachte in- en uitstroom
> 3. **Omzet-analyse** — uitsplitsing per klant, categorie of afdeling
> 4. **Marge-analyse** — bruto marge en kostprijzen
> 5. **Vergelijking** — QoQ, MoM, of YoY
> 6. **Commerciële KPI's** — recurring vs eenmalig, klantconcentratie, groeiers/dalers"

Meerdere keuzes combineren is mogelijk.

---

## Module 1: P&L — Winst & Verliesrekening

### Kerndata ophalen

Gebruik **ReportingBalance** via `execute_operation` — dit is het centrale endpoint voor
alle grootboeksaldi per periode:

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "ReportingYear": 2026,
    "ReportingPeriod": 3
  },
  "select": "GLAccountCode,GLAccountDescription,Amount,BalanceSide,ReportingPeriod,Type"
}
```

### Rekeningschema ophalen

Haal de GLAccounts op om de juiste rekeningen per type te identificeren. Let op: het veld
heet `BalanceType` (niet `Classification` — dat veld bestaat niet):

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": { "BalanceType": "W" },
  "select": "Code,Description,BalanceSide,BalanceType,Type"
}
```

- `BalanceType = "W"` = Winst & verliesrekening (exploitatierekeningen voor P&L)
- `BalanceType = "B"` = Balansrekening (voor cashflow en vermogenspositie)
- `BalanceSide = "C"` = credit (omzet) / `BalanceSide = "D"` = debet (kosten)

### P&L berekenen

Groepeer de ReportingBalance-data op GLAccount.Type:

```
OMZET:           SUM(Amount) voor Type 110 (is negatief → ABS voor weergave)
OVERIGE KOSTEN:  SUM(Amount) voor Type 120
COGS:            SUM(Amount) voor Type 111
BRUTO MARGE:     ABS(OMZET) - COGS
PERSONEELSK.:    SUM(Amount) voor Type 125
AFSCHRIJVINGEN:  SUM(Amount) voor Type 122
OVERIGE KOSTEN:  SUM(Amount) voor overige debet-typen
EBIT:            BRUTO MARGE - alle bedrijfslasten
FIN. B&L:        SUM(Amount) voor Type 160
NETTO RESULT:    EBIT +/- financiële baten/lasten - belastingen (Type 150)
```

**Teken-conventie ReportingBalance**:
- Omzetrekeningen: Amount is **negatief** (credit-saldo) → gebruik ABS() voor weergave
- Kostenrekeningen: Amount is **positief** (debet-saldo) → direct bruikbaar
- Winst = ABS(OMZET) - TOTALE_KOSTEN

### YTD (Year-to-Date) P&L

ReportingBalance per periode geeft **periode-mutaties**, niet cumulatief. Voor YTD:
- Laat `ReportingPeriod` **weg** uit het filter → Exact Online geeft automatisch cumulatieve
  YTD-totalen terug. Dit is de snelste route.
- Of: haal periodes 1 t/m huidig op en sommeer Amount per GLAccountCode.

---

## Module 2: Cashflow Positie

### Stap 1: Huidige liquiditeit

Bankrekeningen en kaspositie uit de balans (code-bereik 1000-1099):

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "GLAccountCode": { "$gte": "1000", "$lte": "1099" },
    "ReportingYear": 2026
  },
  "select": "GLAccountCode,GLAccountDescription,Amount"
}
```

Cumulatief banksaldo = SUM van Amount voor codes 1000-1099.

### Stap 2: Te ontvangen (debiteuren) via ReceivablesList

Gebruik **ReceivablesList** (Read service) — dit bevat het `JournalCode` veld dat nodig
is voor correcte classificatie:

```json
{
  "service": "Read",
  "entity": "ReceivablesList",
  "operation": "GET",
  "select": "AccountName,Amount,JournalCode,JournalDescription,InvoiceNumber,InvoiceDate,DueDate,Description"
}
```

**Classificatielogica — gebruik altijd JournalCode + Amount-teken:**

| JournalCode | Amount | Betekenis | Actie |
|---|---|---|---|
| Bankdagboek (bijv. `"20"`, `"23"`) | negatief | Tegoed — ontvangen betaling nog niet afgeletterd | Afletteren |
| Verkoopboek (bijv. `"70"`) | positief | Vordering — openstaande verkoopfactuur | Opvolgen |
| Verkoopboek (bijv. `"70"`) | negatief | Tegoed — openstaande creditnota | Uitbetalen/verrekenen |

Controleer bankdagboek-codes via: `Financial/Journals` met `Type: 12`.

### Stap 3: Te betalen (crediteuren)

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Payments",
  "operation": "GET",
  "filters": { "Status": [20, 30] },
  "select": "AccountName,AmountDC,DueDate,InvoiceNumber"
}
```

### Cashflow-overzicht presenteren

Groepeer op DueDate-buckets (7d, 8-30d, 31-60d, >60d) en presenteer als samenvatting
met huidige kasmiddelen + verwachte in/uitstroom = verwachte positie.

---

## Module 3: Omzet-Analyse (via analyze_data)

Dit is de kernmodule voor omzetanalyses. Gebruik altijd `analyze_data` met de
`Financial/Transactionlines` tabel en een INNER JOIN op `Financial/GLAccounts`.

### Basispatroon: omzet per periode

```json
{
  "table": "Financial/Transactionlines",
  "joins": [{
    "table": "Financial/GLAccounts",
    "type": "INNER",
    "on": { "leftColumn": "GLAccount", "rightColumn": "ID" },
    "select": ["Code", "Description"]
  }],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Omzet" }
  ],
  "filters": [
    { "column": "t1.Type", "operator": "=", "value": 110 },
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
```

De resultaten bevatten negatieve bedragen (credit-boekingen) → gebruik ABS() bij weergave.

### Omzet per categorie (grootboekrekening)

Voeg `groupBy` toe op de GLAccount-velden om de omzet per omzetcategorie te zien:

```json
{
  "table": "Financial/Transactionlines",
  "joins": [{
    "table": "Financial/GLAccounts",
    "type": "INNER",
    "on": { "leftColumn": "GLAccount", "rightColumn": "ID" },
    "select": ["Code", "Description"]
  }],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Omzet" }
  ],
  "filters": [
    { "column": "t1.Type", "operator": "=", "value": 110 },
    { "column": "Date", "operator": ">=", "value": "2025-10-01" },
    { "column": "Date", "operator": "<=", "value": "2026-03-31" }
  ],
  "groupBy": ["t1.Code", "t1.Description"],
  "dateGroupBy": [
    { "column": "Date", "part": "YEAR", "alias": "Jaar" },
    { "column": "Date", "part": "QUARTER", "alias": "Kwartaal" }
  ],
  "orderBy": [
    { "column": "t1.Code", "direction": "ASC" },
    { "column": "Jaar", "direction": "ASC" }
  ],
  "limit": 200
}
```

Dit geeft een uitsplitsing als bijv. "Omzet Abonnementen" vs "Omzet Maatwerk" vs
"Omzet Overig" — precies zoals het rekeningschema van de klant is ingericht.

### Omzet per klant

TransactionLines bevat het veld `AccountName` — gebruik dit voor klant-uitsplitsing:

```json
{
  "table": "Financial/Transactionlines",
  "joins": [{
    "table": "Financial/GLAccounts",
    "type": "INNER",
    "on": { "leftColumn": "GLAccount", "rightColumn": "ID" },
    "select": []
  }],
  "select": ["AccountName"],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Omzet" }
  ],
  "filters": [
    { "column": "t1.Type", "operator": "=", "value": 110 },
    { "column": "Date", "operator": ">=", "value": "2026-01-01" },
    { "column": "Date", "operator": "<=", "value": "2026-03-31" }
  ],
  "groupBy": ["AccountName"],
  "orderBy": [{ "column": "Omzet", "direction": "ASC" }],
  "limit": 50
}
```

Omzet is negatief in TransactionLines (credit), dus `ASC` sortering = hoogste omzet eerst.

### Omzet per maand (detailniveau)

Gebruik hetzelfde basispatroon maar met maand-groepering:

```json
{
  "dateGroupBy": [
    { "column": "Date", "part": "YEAR", "alias": "Jaar" },
    { "column": "Date", "part": "MONTH", "alias": "Maand" }
  ]
}
```

### Omzet per kostenplaats / afdeling

Controleer eerst of kostenplaatsen zijn ingericht:

```json
{
  "service": "HRM",
  "entity": "CostCenters",
  "operation": "GET",
  "select": "Code,Description,Active"
}
```

Zo ja, voeg `CostCenter` toe aan `select` en `groupBy` in de analyze_data query.

---

## Module 4: Marge-Analyse

### Bruto marge berekenen via analyze_data

Haal omzet (Type 110) en inkoopkosten (Type 111) op in één query:

```json
{
  "table": "Financial/Transactionlines",
  "joins": [{
    "table": "Financial/GLAccounts",
    "type": "INNER",
    "on": { "leftColumn": "GLAccount", "rightColumn": "ID" },
    "select": ["Type"]
  }],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Bedrag" }
  ],
  "filters": [
    { "column": "t1.Type", "operator": "IN", "values": [110, 111] },
    { "column": "Date", "operator": ">=", "value": "2026-01-01" },
    { "column": "Date", "operator": "<=", "value": "2026-03-31" }
  ],
  "groupBy": ["t1.Type"],
  "dateGroupBy": [
    { "column": "Date", "part": "YEAR", "alias": "Jaar" },
    { "column": "Date", "part": "QUARTER", "alias": "Kwartaal" }
  ]
}
```

Berekening: Bruto marge = (ABS(Omzet) - Inkoopkosten) / ABS(Omzet) x 100%

**Marge-benchmarks** (indicatief):
- Dienstverlening: 60-80% is gebruikelijk
- Handel/groothandel: 20-40%
- Productie/maakindustrie: 30-50%

---

## Module 5: Vergelijking & Trends

### QoQ, MoM, YoY vergelijkingen

Alle vergelijkingen verlopen via `analyze_data` op TransactionLines. Het datumfilter
bepaalt welke periodes je vergelijkt:

| Vergelijking | Datumfilter | dateGroupBy |
|---|---|---|
| QoQ (dit vs vorig kwartaal) | Afgelopen 6 maanden | YEAR + QUARTER |
| MoM (deze vs vorige maand) | Afgelopen 2 maanden | YEAR + MONTH |
| YoY (dit jaar vs vorig jaar) | Dit jaar + vorig jaar | YEAR |
| YTD vergelijking | Jan-huidig, beide jaren | YEAR + MONTH |

Voer altijd **twee parallelle analyze_data calls** uit:
1. Totalen per periode (voor de samenvattingstabel)
2. Per categorie per periode (voor de uitsplitsing waar de verandering zit)

### Signalering

Bereken deltas en signaleer automatisch:
- Omzetgroei > 20%: sterke groei
- Kostengroei > omzetgroei: druk op de marge
- Negatieve omzetgroei: analyseer oorzaak (welke categorie/klant?)
- Marge-verbetering: efficiëntiewinst
- Marge-verslechtering >3pp: kostenbeheersing vereist aandacht

---

## Module 6: Commerciële KPI's

Deze module geeft inzicht in de commerciële gezondheid van het bedrijf, voorbij de
pure financiële cijfers.

### KPI 1: Recurring vs eenmalig

Gebruik de omzet-per-categorie query (Module 3) en classificeer de GLAccount-codes
in twee groepen op basis van de rekeningnaam:
- **Recurring**: rekeningen met "Abonnement" in de naam
- **Eenmalig**: rekeningen met "Maatwerk", "Project", "Consult", "Overig" in de naam

Presenteer als: "X% van de omzet is recurring" + trend vs vorige periode.
Dit is een cruciale indicator — hoe hoger het recurring-percentage, hoe voorspelbaarder
de inkomsten.

### KPI 2: Klantconcentratie

Gebruik de omzet-per-klant query (Module 3) en bereken:
- Top 5 klanten als % van totale omzet
- Top 10 klanten als % van totale omzet

Signalering:
- Top 5 > 50%: klantconcentratie-risico — afhankelijkheid van enkele grote klanten
- Top 1 > 25%: hoog risico — één klant bepaalt een kwart van de omzet

### KPI 3: Groeiers en dalers

Vergelijk de omzet per klant tussen twee periodes (QoQ of YoY). Sorteer op
absoluut verschil en toon:
- Top 5 groeiers (hoogste absolute groei)
- Top 5 dalers (hoogste absolute daling)
- Nieuwe klanten (omzet in huidige periode, geen omzet in vorige)
- Verdwenen klanten (omzet in vorige periode, geen omzet in huidige)

### KPI 4: Gemiddeld factuurbedrag

Een stijgend gemiddeld factuurbedrag bij dalend volume kan duiden op verschuiving naar
grotere opdrachten — minder voorspelbaar maar hogere waarde per klant.

---

## Management Rapport Output

```
Management Informatie — [Maand/Kwartaal] [Jaar]
Stand per [datum] — Real-time uit Exact Online (analyze_data)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RESULTATENREKENING (YTD t/m [periode])
                          Dit jaar    Vorig jaar    Δ%
Netto-omzet:            €  xxx.xxx   €  xxx.xxx   +xx,x%
Inkoopkosten:           €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
Bruto marge:            €  xxx.xxx   €  xxx.xxx   +xx,x%
Bruto marge %:                xx,x%        xx,x%   +x,xpp

Personeelskosten:       €  xxx.xxx   €  xxx.xxx    +x,x%
Overige bedrijfsl.:     €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
EBIT:                   €  xxx.xxx   €  xxx.xxx   +xx,x%

OMZET PER CATEGORIE (dit kwartaal vs vorig kwartaal)
                          Q huidig    Q vorig       Δ%
Abonnementen:           €  xxx.xxx   €  xxx.xxx   +xx,x%
Maatwerk:               €  xxx.xxx   €  xxx.xxx   +xx,x%
Overig:                 €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
Totaal:                 €  xxx.xxx   €  xxx.xxx   +xx,x%

COMMERCIËLE KPI's
  Recurring %:                xx%  (vorig kwartaal: xx%)
  Top 5 klantconcentratie:    xx%  (vorig kwartaal: xx%)

TOP 5 KLANTEN (omzet periode)
  1. Klant A BV         € xx.xxx  (xx,x%)
  2. Klant B NV         € xx.xxx  (xx,x%)
  ...

TOP 5 GROEIERS                    TOP 5 DALERS
  Klant X  +€ xx.xxx (+xxx%)        Klant Y  -€ x.xxx (-xx%)
  ...                                ...

AANDACHTSPUNTEN:
[Automatisch gegenereerde signaleringen op basis van de data]
```

---

## Bekende API-eigenaardigheden

| Valkuil | Correct | Toelichting |
|---------|---------|-------------|
| SalesInvoices voor bedragen | `analyze_data` op `Financial/Transactionlines` | SalesInvoices bevat bruto factuurbedragen, niet de geboekte omzet |
| GLAccount filter op code-ranges | Filter op `t1.Type = 110` via JOIN | Rekeningnummers variëren per administratie |
| GLAccounts filter `Classification` | Gebruik `BalanceType` (W of B) | `Classification` veld bestaat niet in GLAccounts entity |
| `service: "Financial"` voor TransactionLines | In `execute_operation`: `service: "Financialtransaction"` | In `analyze_data`: gewoon `Financial/Transactionlines` als table |
| ReportingBalance Amount = saldo | Amount = mutatie van die periode | Laat ReportingPeriod weg voor cumulatief YTD |
| Omzet Amount = positief | Omzet Amount = negatief (credit) | ABS() nodig voor weergave; kosten zijn positief |
| ReceivablesList Amount teken = richting | Combineer JournalCode + Amount-teken | Bankdagboek = tegoed, verkoopboek positief = vordering |
| CostCenters altijd beschikbaar | CostCenters optioneel | Niet elke administratie gebruikt kostenplaatsen |
| ReportingBalance voor banksaldo | Codes 1000-1099, laat ReportingPeriod weg | Banksaldo is cumulatief |

---

## Samenwerking met andere skills

- **Debiteurenbeheer**: bij signaleren van >60 dagen openstaand → verwijs naar debiteurenbeheer skill
- **Periodeafsluiting**: als cijfers niet kloppen → check of periode volledig gesloten is
- **Cashflow-analyse**: bij liquiditeitsvragen → verwijs naar de cashflow-analyse skill
- **Resultatenrekening-analyse**: voor een volledige W&V conform BW2 Titel 9

## Communicatie

- Spreek als financieel adviseur, niet als boekhouder: geef duiding, niet alleen cijfers
- Geef altijd context bij afwijkingen: wat betekent dit voor het bedrijf?
- Signaleer actief wat aandacht vraagt — de ondernemer heeft niet altijd financiële expertise
- Gebruik € Nederlands formaat (€ 1.234,56) en Nederlandse terminologie
- Vermeld altijd "stand per [datum]" en de databron ("via analyze_data / TransactionLines")
- Vraag bij eerste gebruik: welk boekjaar, welke periode, en wil men YTD of specifieke periode?
- Bied aan om bevindingen als rapport te exporteren (docx/xlsx) via de bijbehorende skills
