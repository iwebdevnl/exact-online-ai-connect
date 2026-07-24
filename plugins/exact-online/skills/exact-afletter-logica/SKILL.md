---
name: exact-afletter-logica
description: >
  Identificeer onafgeletterde posten in Exact Online en voer aflettingen uit via de
  MatchSets API. Ondersteunt 1:1 matching, deelbetalingen, creditnota's, incasso-batches
  en N:1 matches. Gebruikt Receivables/Payables endpoints voor identificatie en
  Financial/MatchSets voor uitvoering.
  Triggers: 'afletter', 'afletteren', 'afgeletterd', 'reconciliatie bankboek', 'onafgeletterde posten',
  'af te handelen bankregels', 'welke bankregels staan nog open', 'bankboek opschonen',
  'kruisposten', 'unmatched bank entries', 'bank reconciliation Exact Online',
  'deelbetaling', 'creditnota matchen', 'write-off boeken', 'betalingen zonder factuur',
  'uitgaande betalingen niet gematcht', 'openstaande crediteuren', 'openstaande debiteuren'.
  Werkt met de Exact Online REST API (execute_operation), analyze_data en
  de MatchSets API (POST Financial/MatchSets) voor het daadwerkelijk afletteren.
---

# Exact Online Afletter-Logica

Identificeer onafgeletterde posten en voer aflettingen uit. Werkt generiek — geen hardcoded grootboekcodes.

## Twee Use Cases

| Use Case | Doel | Primaire methode |
|----------|------|-----------------|
| **A: Bulk Scan** | "Wat moet er nog verwerkt worden?" | Phase 0 → analyze_data / Receivables/Payments |
| **B: Specifieke Match** | "Letter deze factuur af tegen die betaling" | Phase 1-2 → REST API + MatchSets |

Begin altijd met Phase 0 voor een totaaloverzicht. Ga pas naar Phase 1-2 als je specifieke posten wilt matchen.

## Phase 0: Discovery & Overzichtsscan

**Standaard aanpak: REST API** (execute_operation). Werkt altijd, real-time data, geen subscription vereist.
analyze_data is optioneel voor bulk-aggregaties over grote datasets — vereist Analytics of Trial subscription en kan mid-sync incomplete data bevatten.

### Stap 0.1: Ontdek grootboekrekeningen (ALTIJD EERST)

Grootboekcodes verschillen per administratie. Gebruik het `Type` veld op GLAccounts om de juiste rekeningen te vinden:

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Type": [12, 20, 22]},
  "select": "Code,Description,Type,TypeDescription,BalanceSide"
}
```

**Universele GLAccount Types** (werken in elke Exact Online administratie):

| Type | TypeDescription | Betekenis | Voorbeeld codes |
|------|----------------|-----------|-----------------|
| 12 | Bank | Bankrekeningen | 1100, 1110, 1120 |
| 20 | Accounts receivable | Debiteurenrekening | 1300, 1310 |
| 22 | Accounts payable | Crediteurenrekening | 1600, 1400 |
| 90 | General | Algemeen (incl. kruisposten) | 1050, 1290 |

Sla de gevonden codes op voor alle vervolgqueries:
- `debiteurenCode` ← Type 20 (bijv. "1300")
- `crediteurenCode` ← Type 22 (bijv. "1600")
- `bankCodes` ← Type 12 (bijv. ["1100", "1110"])

**Voor kruisposten/parkeerrekeningen** (Type 90): zoek aanvullend op beschrijving:

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Type": 90, "Description": {"contains": "kruis"}},
  "select": "Code,Description"
}
```

### Stap 0.2: Ontdek dagboeken

```json
{
  "service": "Financial",
  "entity": "Journals",
  "operation": "GET",
  "filters": {"Type": [12, 20, 22]},
  "select": "Code,Description,Type"
}
```

| Type | Omschrijving |
|------|-------------|
| 12 | Bank/Kas |
| 20 | Verkoop |
| 22 | Inkoop (zelfde type als Accounts payable GLAccount!) |

**Let op**: Inkoopboek en crediteuren-GLAccount delen Type 22. Onderscheid dagboek vs GLAccount op basis van het entity (Journals vs GLAccounts).

### Stap 0.3: Bulk scan debiteuren (openstaande vorderingen)

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "operation": "GET",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,InvoiceNumber,Description,DueDate,InvoiceDate,EntryNumber"
}
```

Status 20 = open, Status 30 = gedeeltelijk betaald.

**Optioneel via analyze_data** (alleen met Analytics/Trial subscription):

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

**Let op**: In de Exact Online API heet de Payables endpoint `Cashflow/Payments` (niet `Payables`).

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Payments",
  "operation": "GET",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,Description,DueDate,InvoiceDate,InvoiceNumber,EntryNumber,Status"
}
```

### Stap 0.5: Detectie betalingen zonder inkoopfactuur

Dit zijn uitgaande bankbetalingen waarvoor geen inkoopfactuur is aangemaakt. Er zijn twee patronen:

**Patroon 1: Crediteurenrekening + InvoiceNumber = null**

Betaling is op de crediteurenrekening geboekt en aan een relatie gekoppeld, maar er is geen inkoopfactuur aangemaakt in het inkoopboek:

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "operation": "GET",
  "filters": {
    "JournalCode": "{bankdagboek}",
    "GLAccountCode": "{crediteurenCode}",
    "FinancialYear": "{jaar}",
    "InvoiceNumber": null
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,InvoiceNumber"
}
```

De `InvoiceNumber: null` filter werkt API-side — alleen posten zonder gekoppelde inkoopfactuur worden geretourneerd. Geen client-side filtering nodig.

**Patroon 2: Parkeerrekening (kruisposten)**

Betaling is op een tussenrekening geparkeerd, zonder relatie en zonder factuur:

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "operation": "GET",
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
| Geparkeerd | Kruisposten (1050/1290 of equivalent) | `null` | Moet nog uitgesplitst |
| Koersverschil | Type 22 (crediteuren) | gevuld | OffsetID=null, kleine bedragen, omschrijving "Koersverschil" |
| Betalingsverschil | Type 22 (crediteuren) | gevuld | OffsetID=null, kleine bedragen, omschrijving "Betalingsverschil" |

## Phase 1: Gedetailleerde Identificatie (per klant/leverancier)

### Stap 1.1: Receivables/Payables per relatie

```json
{
  "service": "Cashflow",
  "entity": "Receivables",
  "operation": "GET",
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

**Kernregel**: Controleer altijd de **factuurzijde** (verkoop-/inkoopdagboek), niet de bankzijde:

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "operation": "GET",
  "filters": {
    "JournalCode": "{verkoopdagboek}",
    "GLAccountCode": "{debiteurenCode}",
    "FinancialYear": "{jaar}"
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,InvoiceNumber"
}
```

`OffsetID = null` op de factuurzijde → factuur is NIET afgeletterd.

### Stap 1.3: Bijbehorende bankbetalingen zoeken

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "operation": "GET",
  "filters": {
    "JournalCode": "{bankdagboek}",
    "GLAccountCode": "{debiteurenCode}",
    "FinancialYear": "{jaar}"
  },
  "select": "ID,EntryNumber,Date,Description,AmountDC,OffsetID,AccountName,GLAccountCode,InvoiceNumber"
}
```

Match op basis van AccountName, bedrag en/of Description/InvoiceNumber.

## Phase 2: Afletteren via MatchSets API

### Endpoint

- **Service**: `Financial`
- **Entity**: `MatchSets`
- **Methode**: POST
- **Scope**: `Financial receivables payables`

De MCP tool toont een preview met bedragen en vraagt om bevestiging (`confirmed: true`) voordat de aflettering wordt uitgevoerd.

### Schema

```json
{
  "matches": [
    {"line_id": "{guid-factuur-regel}", "line_type": "TransactionLine"},
    {"line_id": "{guid-bank-regel}", "line_type": "BankEntryLine"}
  ],
  "write_off": {
    "gl_account_code": "{write-off-rekening}",
    "date": "2026-01-31",
    "type": 3
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
| `BankEntryLine` | Bankboekingen |
| `CashEntryLine` | Kasboekingen |
| `GeneralJournalEntryLine` | Memoriaalboekingen |

**Let op**: Er is geen `SalesEntryLine` of `PurchaseEntryLine` line_type. Gebruik `TransactionLine` voor factuurregels.

## Matching Scenario's

### Scenario 1: 1:1 Matching (standaard)

Eén factuur wordt volledig betaald door één bankbetaling. Bedragen zijn exact gelijk.

```json
{
  "matches": [
    {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
    {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
  ]
}
```

### Scenario 2: Deelbetaling (N:1 factuurzijde)

Eén factuur wordt in meerdere termijnen betaald. Elke betaling wordt apart afgeletterd. Gebruik GEEN write-off bij deelbetalingen — het restant blijft open.

### Scenario 3: Creditnota matching

Een creditnota creëert een negatieve debiteurenregel. Beide zijn TransactionLine type:

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

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Description": {"contains": "betalingsverschil"}},
  "select": "Code,Description"
}
```

| Situatie | Zoekterm | Standaard RGS |
|----------|----------|---------------|
| Betalingsverschil (< €1) | "betalingsverschil" | 4860 |
| Koersverschil (vreemde valuta) | "koersverschil" | 9280 |
| Korting (skonto) | "korting" of "skonto" | 4860 of specifiek |

## Volledige Workflow

```
Phase 0: Discovery & Scan
  Stap 0.1: GLAccounts ophalen (Type 12/20/22) → codes leren
  Stap 0.2: Journals ophalen (Type 12/20/22) → dagboekcodes leren
  Stap 0.3: Bulk scan Receivables (AmountDC ≠ 0)
  Stap 0.4: Bulk scan Payments (Status 20/30)
  Stap 0.5: Betalingen zonder factuur detecteren
            → crediteurenrekening + InvoiceNumber = null
            → kruisposten/parkeerrekeningen

Phase 1: Gedetailleerde Identificatie (per relatie)
  Stap 1.1: Receivables/Payments per relatie
  Stap 1.2: OffsetID check op factuurzijde
  Stap 1.3: Bijbehorende bankbetalingen zoeken

Phase 2: Uitvoering
  Stap 2.1: MatchSets POST (preview)
  Stap 2.2: MatchSets POST (confirmed: true)
  Stap 2.3: Verifieer resultaat via Receivables/Payments
```

## Resultaat Presentatie

Groepeer onafgeletterde posten per categorie met actiesuggesties:

| Categorie | Actie |
|-----------|-------|
| Betalingen zonder factuur (crediteurenrek. + InvoiceNumber=null) | Inkoopfactuur aanmaken in inkoopboek, dan afletteren |
| Geparkeerde bedragen (kruisposten) | Uitsplitsen via memoriaal naar kostenrekeningen |
| Onafgeletterde debiteuren (AmountDC < 0 op Receivables) | Matchen met openstaande verkoopfactuur via MatchSets |
| Onafgeletterde crediteuren (Payments Status 20/30) | Matchen met bankbetaling via MatchSets |
| Koers-/betalingsverschillen (OffsetID=null, klein bedrag) | Write-off naar betalingsverschillen/koersverschillen rekening |
| Creditnota's tegenover facturen | Matchen via MatchSets (beide TransactionLine) |

## Bekende Beperkingen

- Payables endpoint heet `Cashflow/Payments` in de API, niet `Cashflow/Payables`
- `InvoiceNumber: null` filter werkt API-side voor detectie betalingen zonder factuur
- `OffsetID IS NULL` kan NIET gefilterd worden via de REST API — moet client-side (maar is zelden nodig dankzij InvoiceNumber null)
- `BankEntryLines` endpoint heeft beperkte filtermogelijkheden
- `analyze_data`: `IN`-operator kan falen; gebruik aparte queries per waarde
- analyze_data tabellen kunnen in sync zijn (warning "Data synchronisatie actief") — gebruik dan REST API als fallback
- Er is geen `SalesEntryLine` of `PurchaseEntryLine` line_type beschikbaar voor MatchSets
- Deelbetalingen: controleer of Exact Online automatisch regelsplits maakt na gedeeltelijke aflettering
