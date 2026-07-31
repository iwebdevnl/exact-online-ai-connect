---
name: exact-afletter-logica
description: >
  Deze skill moet gebruikt worden wanneer de gebruiker onafgeletterde posten in Exact Online wil
  opsporen of wil afletteren: 1:1 matches, deelbetalingen, creditnota's, incasso-batches en N:1
  matches, inclusief write-off bij koers- en betalingsverschillen. Triggers: 'afletteren',
  'onafgeletterde posten', 'reconciliatie bankboek', 'welke bankregels staan nog open',
  'kruisposten', 'deelbetaling', 'creditnota matchen', 'betalingen zonder factuur'.
---

# Exact Online Afletter-Logica

Identificeer onafgeletterde posten en voer aflettingen uit. Werkt generiek, zonder hardcoded
grootboekcodes.

## Welke tool wanneer

| Tool | Waarvoor |
|------|----------|
| `read_operation` | Alle GET-queries: discovery, scans, regels opzoeken. Levert maximaal 60 records per aanroep. |
| `write_operation` | Afletteren via MatchSets (POST). Vereist `confirmed: true`. |
| `analyze_data` | Optioneel voor bulk-aggregaties. Alleen op Trial of Analytics, niet op Essentials. |

`read_operation` geeft maximaal 60 records terug. Op Bulk-endpoints pagineer je met de
`page_token` uit `next_page.page_token` van de vorige respons, niet met `skip`. Haal bij een
bulk-scan altijd alle pagina's op voordat je conclusies trekt: een afgekapte lijst laat posten
stil verdwijnen.

Zie [references/query-patterns.md](references/query-patterns.md) voor de analyze_data-varianten
van deze queries en voor de BankEntryLines-lookup per entrynummer.

## Twee Use Cases

| Use Case | Doel | Primaire methode |
|----------|------|-----------------|
| **A: Bulk Scan** | "Wat moet er nog verwerkt worden?" | Phase 0, read_operation of analyze_data |
| **B: Specifieke Match** | "Letter deze factuur af tegen die betaling" | Phase 1-2, read_operation plus write_operation op MatchSets |

Begin altijd met Phase 0 voor een totaaloverzicht. Ga pas naar Phase 1-2 als je specifieke posten wilt matchen.

## Phase 0: Discovery & Overzichtsscan

**Standaard aanpak: `read_operation`.** Werkt altijd, real-time data, geen extra abonnement vereist.
`analyze_data` is optioneel voor bulk-aggregaties over grote datasets, vereist Analytics of Trial
en kan mid-sync incomplete data bevatten.

### Stap 0.1: Ontdek grootboekrekeningen (ALTIJD EERST)

Grootboekcodes verschillen per administratie. Gebruik het `Type` veld op GLAccounts om de juiste rekeningen te vinden:

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": {"Type": [10, 12, 20, 22]},
  "select": "Code,Description,Type,TypeDescription,BalanceSide"
}
```

**Universele GLAccount Types** (werken in elke Exact Online administratie):

| Type | TypeDescription | Betekenis | Voorbeeld codes |
|------|----------------|-----------|-----------------|
| 10 | Cash | Kasrekeningen | 1000 |
| 12 | Bank | Bankrekeningen | 1100, 1110, 1120 |
| 20 | Accounts receivable | Debiteurenrekening | 1300, 1310 |
| 22 | Accounts payable | Crediteurenrekening | 1600, 1400 |
| 90 | General | Algemeen, inclusief kruisposten | 1050, 1290 |

Sla de gevonden codes op voor alle vervolgqueries:
- `debiteurenCode` uit Type 20 (bijvoorbeeld "1300")
- `crediteurenCode` uit Type 22 (bijvoorbeeld "1600")
- `liquideCodes` uit Type 10 en 12 (bijvoorbeeld ["1000", "1100", "1110"])

**Voor kruisposten/parkeerrekeningen** (Type 90): zoek aanvullend op beschrijving:

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": {"Type": 90, "Description": {"contains": "kruis"}},
  "select": "Code,Description"
}
```

### Stap 0.2: Ontdek dagboeken

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "Journals",
  "filters": {"Type": [10, 12, 20, 22, 80]},
  "select": "Code,Description,Type"
}
```

| Type | Omschrijving |
|------|-------------|
| 10 | Kas |
| 12 | Bank |
| 20 | Verkoop |
| 22 | Inkoop |
| 80 | Memoriaal |

Vraag altijd alle vijf de types op. Kasdagboeken (10) leveren `CashEntryLine`-regels en
memoriaaldagboeken (80) leveren `GeneralJournalEntryLine`-regels; beide zijn geldige line_types
bij MatchSets, en het memoriaal is de route om kruisposten uit te splitsen. Filter je alleen op
12, 20 en 22, dan blijven die twee dagboeksoorten onzichtbaar.

**Let op**: Inkoopboek en crediteuren-GLAccount delen Type 22. Onderscheid dagboek van GLAccount op
basis van de entity (Journals of GLAccounts). Type 21 bestaat niet als dagboeksoort.

### Stap 0.3: Bulk scan debiteuren (openstaande vorderingen)

**Tool: `read_operation`**

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,InvoiceNumber,Description,DueDate,InvoiceDate,EntryNumber"
}
```

Status 20 = open, Status 30 = gedeeltelijk betaald. Krijg je 60 records terug, herhaal de aanroep
met `page_token` uit `next_page.page_token` tot er geen vervolgpagina meer is.

**Optioneel via analyze_data** (alleen op Trial of Analytics):

**Tool: `analyze_data`**

```json
{
  "query": {
    "table": "Cashflow/Receivables",
    "filters": [
      {"column": "AmountDC", "operator": "!=", "value": 0}
    ],
    "select": ["AccountName"],
    "aggregations": [
      {"function": "SUM", "column": "AmountDC", "alias": "TotaalOpen"},
      {"function": "COUNT", "column": "ID", "alias": "AantalPosten"}
    ],
    "groupBy": ["AccountName"],
    "orderBy": [{"column": "TotaalOpen", "direction": "ASC"}]
  }
}
```

### Stap 0.4: Bulk scan crediteuren (openstaande schulden)

**Let op**: In de Exact Online API heet de Payables endpoint `Cashflow/Payments`, niet `Payables`.

**Tool: `read_operation`**

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Payments",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,Description,DueDate,InvoiceDate,InvoiceNumber,EntryNumber,Status"
}
```

Ook hier geldt de 60-recordgrens: pagineer met `page_token` tot de lijst compleet is.

### Stap 0.5: Detectie betalingen zonder inkoopfactuur

Dit zijn uitgaande bankbetalingen waarvoor geen inkoopfactuur is aangemaakt. Er zijn twee patronen:

**Patroon 1: Crediteurenrekening + InvoiceNumber = null**

Betaling is op de crediteurenrekening geboekt en aan een relatie gekoppeld, maar er is geen inkoopfactuur aangemaakt in het inkoopboek:

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "filters": {
    "JournalCode": "{bankdagboek}",
    "GLAccountCode": "{crediteurenCode}",
    "FinancialYear": "{jaar}",
    "InvoiceNumber": null
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,InvoiceNumber"
}
```

De `InvoiceNumber: null` filter werkt API-side: alleen posten zonder gekoppelde inkoopfactuur worden geretourneerd. Geen client-side filtering nodig.

**Patroon 2: Parkeerrekening (kruisposten)**

Betaling is op een tussenrekening geparkeerd, zonder relatie en zonder factuur:

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "filters": {
    "JournalCode": "{bankdagboek}",
    "GLAccountCode": "{kruispostenCode}",
    "FinancialYear": "{jaar}"
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,AccountName,GLAccountCode"
}
```

Kruisposten hebben vaak geen AccountName en geen InvoiceNumber. Ze moeten handmatig worden uitgesplitst via een memoriaalpost.

**Samenvatting detectielogica:**

| Indicator | GLAccount | InvoiceNumber | Betekenis |
|-----------|-----------|---------------|-----------|
| Betaald, geen factuur | Type 22 (crediteuren) | `null` | Inkoopfactuur ontbreekt |
| Geparkeerd | Kruisposten (Type 90) | `null` | Moet nog uitgesplitst |
| Koersverschil | Type 22 (crediteuren) | gevuld | OffsetID=null, kleine bedragen, omschrijving "Koersverschil" |
| Betalingsverschil | Type 22 (crediteuren) | gevuld | OffsetID=null, kleine bedragen, omschrijving "Betalingsverschil" |

## Phase 1: Gedetailleerde Identificatie (per klant/leverancier)

### Stap 1.1: Receivables/Payables per relatie

**Tool: `read_operation`**

```json
{
  "service": "Cashflow",
  "entity": "Receivables",
  "select": "ID,AccountName,InvoiceNumber,AmountDC,Description,EntryNumber,DueDate,InvoiceDate",
  "filters": {"AccountName": "{klantnaam}"}
}
```

| AmountDC | Status | Actie |
|----------|--------|-------|
| `= 0` | Volledig afgeletterd | Geen actie nodig |
| `> 0` | Openstaande vordering | Zoek bijbehorende betaling |
| `< 0` | Onafgeletterd tegoed | Betaling ontvangen maar niet gematcht met factuur |

### Stap 1.2: OffsetID op factuurzijde (alternatief)

Raadpleeg [references/offsetid-matching.md](references/offsetid-matching.md) voor de volledige OffsetID-logica.

**Kernregel**: Controleer altijd de **factuurzijde** (verkoop- of inkoopdagboek), niet de bankzijde:

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "filters": {
    "JournalCode": "{verkoopdagboek}",
    "GLAccountCode": "{debiteurenCode}",
    "FinancialYear": "{jaar}"
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,InvoiceNumber"
}
```

`OffsetID = null` op de factuurzijde betekent: factuur is NIET afgeletterd. Filter daar client-side
op, `OffsetID` kan niet server-side op leegheid worden gefilterd.

### Stap 1.3: Bijbehorende bankbetalingen zoeken

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "filters": {
    "JournalCode": "{bankdagboek}",
    "GLAccountCode": "{debiteurenCode}",
    "FinancialYear": "{jaar}"
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,GLAccountCode,InvoiceNumber"
}
```

Match op basis van AccountName, bedrag en/of Description/InvoiceNumber. Zoek je liever op één
bankboeking, gebruik dan de BankEntryLines-variant uit
[references/query-patterns.md](references/query-patterns.md).

## Phase 2: Afletteren via MatchSets

### Endpoint

- **Tool**: `write_operation`
- **Service**: `Financial`
- **Entity**: `MatchSets`
- **Operation**: POST
- **Scope**: `Financial receivables payables`

Zonder `confirmed: true` toont de tool eerst een preview met bedragen. Pas met `confirmed: true`
wordt de aflettering uitgevoerd. `execute_operation` kan dit niet: dat is een verouderde, GET-only
alias die op een POST een fout teruggeeft zonder iets uit te voeren.

### Volledige aanroep

**Tool: `write_operation`**

```json
{
  "service": "Financial",
  "entity": "MatchSets",
  "operation": "POST",
  "confirmed": true,
  "data": {
    "matches": [
      {"line_id": "{guid-factuur-regel}", "line_type": "TransactionLine"},
      {"line_id": "{guid-bank-regel}", "line_type": "BankEntryLine"}
    ],
    "write_off": {
      "gl_account_code": "{write-off-rekening}",
      "date": "2026-01-31"
    }
  }
}
```

| Parameter | Verplicht | Beschrijving |
|-----------|-----------|-------------|
| `matches` | Ja | Array van minimaal 2 regels met `line_id` (GUID) en `line_type` |
| `write_off` | Nee | Alleen bij verschil: `gl_account_code`, `date`, optioneel `type` (3=debet, 4=credit) |

### Geldige line_type waarden

| line_type | Bron |
|-----------|------|
| `TransactionLine` | Generiek (alle transacties, inclusief facturen) |
| `BankEntryLine` | Bankboekingen (dagboektype 12) |
| `CashEntryLine` | Kasboekingen (dagboektype 10) |
| `GeneralJournalEntryLine` | Memoriaalboekingen (dagboektype 80) |

**Let op**: Er is geen `SalesEntryLine` of `PurchaseEntryLine` line_type. Gebruik `TransactionLine` voor factuurregels.

## Matching Scenario's

De blokken hieronder tonen alleen de `data`-payload. Verstuur ze met `write_operation`, service
`Financial`, entity `MatchSets`, operation `POST` en `confirmed: true`, zoals in de volledige
aanroep hierboven.

### Scenario 1: 1:1 Matching (standaard)

Eén factuur wordt volledig betaald door één bankbetaling. Bedragen zijn exact gelijk.

**Tool: `write_operation`, veld `data`**

```json
{
  "matches": [
    {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
    {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
  ]
}
```

### Scenario 2: Deelbetaling (N:1 factuurzijde)

Eén factuur wordt in meerdere termijnen betaald. Elke betaling wordt apart afgeletterd. Gebruik
GEEN write-off bij deelbetalingen, het restant blijft open.

### Scenario 3: Creditnota matching

Een creditnota creëert een negatieve debiteurenregel. Beide zijn TransactionLine type:

**Tool: `write_operation`, veld `data`**

```json
{
  "matches": [
    {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
    {"line_id": "{creditnota-id}", "line_type": "TransactionLine"}
  ]
}
```

### Scenario 4: Incasso-batch (1:N bankzijde)

Eén bankboeking bevat meerdere debiteuren. Elke debiteurenregel wordt apart gematcht met de bijbehorende factuur.

### Scenario 5: N:1 betalingszijde

Eén betaling dekt meerdere facturen:

**Tool: `write_operation`, veld `data`**

```json
{
  "matches": [
    {"line_id": "{factuur-A-id}", "line_type": "TransactionLine"},
    {"line_id": "{factuur-B-id}", "line_type": "TransactionLine"},
    {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
  ]
}
```

## Write-off Regels

Gebruik write-off alleen wanneer het verschil verklaarbaar is. Zoek de juiste write-off rekening dynamisch:

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": {"Description": {"contains": "betalingsverschil"}},
  "select": "Code,Description"
}
```

| Situatie | Zoekterm | Vaak gebruikte code (altijd verifiëren) |
|----------|----------|------------------------------------------|
| Betalingsverschil (< €1) | "betalingsverschil" | wisselt per administratie, gebruik de gevonden Code |
| Koersverschil (vreemde valuta) | "koersverschil" | wisselt per administratie, gebruik de gevonden Code |
| Korting (skonto) | "korting" of "skonto" | wisselt per administratie, gebruik de gevonden Code |

Neem nooit een rekeningnummer uit een voorbeeld over. Gebruik uitsluitend de `Code` die de
zoekopdracht hierboven in déze administratie oplevert, en laat de gebruiker die bevestigen als er
meerdere kandidaten zijn.

## Volledige Workflow

```
Phase 0: Discovery & Scan
  Stap 0.1: GLAccounts ophalen (Type 10/12/20/22) via read_operation, codes leren
  Stap 0.2: Journals ophalen (Type 10/12/20/22/80) via read_operation, dagboekcodes leren
  Stap 0.3: Bulk scan Receivables (Status 20/30), alle pagina's via page_token
  Stap 0.4: Bulk scan Payments (Status 20/30), alle pagina's via page_token
  Stap 0.5: Betalingen zonder factuur detecteren
            crediteurenrekening + InvoiceNumber = null
            kruisposten/parkeerrekeningen

Phase 1: Gedetailleerde Identificatie (per relatie)
  Stap 1.1: Receivables/Payments per relatie
  Stap 1.2: OffsetID check op factuurzijde
  Stap 1.3: Bijbehorende bankbetalingen zoeken

Phase 2: Uitvoering
  Stap 2.1: write_operation MatchSets zonder confirmed (preview)
  Stap 2.2: write_operation MatchSets met confirmed: true
  Stap 2.3: Verifieer resultaat via read_operation op Receivables/Payments
```

## Resultaat Presentatie

Groepeer onafgeletterde posten per categorie met actiesuggesties:

| Categorie | Actie |
|-----------|-------|
| Betalingen zonder factuur (crediteurenrek. + InvoiceNumber=null) | Inkoopfactuur aanmaken in inkoopboek, dan afletteren |
| Geparkeerde bedragen (kruisposten) | Uitsplitsen via een memoriaalboeking naar kostenrekeningen |
| Onafgeletterde debiteuren (AmountDC < 0 op Receivables) | Matchen met openstaande verkoopfactuur via MatchSets |
| Onafgeletterde crediteuren (Payments Status 20/30) | Matchen met bankbetaling via MatchSets |
| Koers- of betalingsverschillen (OffsetID=null, klein bedrag) | Write-off naar betalingsverschillen- of koersverschillenrekening |
| Creditnota's tegenover facturen | Matchen via MatchSets (beide TransactionLine) |

## Bekende Beperkingen

- Payables endpoint heet `Cashflow/Payments` in de API, niet `Cashflow/Payables`
- `read_operation` levert maximaal 60 records; Bulk-endpoints pagineer je met `page_token`, niet met `skip`
- `InvoiceNumber: null` filter werkt API-side voor detectie betalingen zonder factuur
- `OffsetID` kan niet server-side op leegheid worden gefilterd, niet via `read_operation` en niet via `analyze_data`. Selecteer het veld en filter client-side
- `BankEntryLines` endpoint heeft beperkte filtermogelijkheden
- `analyze_data`: de `IN`-operator kan falen; gebruik aparte queries per waarde
- `analyze_data` is niet beschikbaar op Essentials, alleen op Trial en Analytics
- analyze_data tabellen kunnen in sync zijn (waarschuwing "Data synchronisatie actief"); gebruik dan `read_operation` als fallback
- Er is geen `SalesEntryLine` of `PurchaseEntryLine` line_type beschikbaar voor MatchSets
- Deelbetalingen: verifieer na elke deelaflettering via Receivables/Payments dat `AmountDC` het resterende bedrag toont, en boek geen write-off zolang er nog termijnen volgen
