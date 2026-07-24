# Query Patronen voor Afletteren

## Stap 0: Discovery (ALTIJD EERST)

### 0a: Ontdek grootboekrekeningen per type

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Type": [12, 20, 22]},
  "select": "Code,Description,Type,TypeDescription,BalanceSide"
}
```

Resultaat geeft per administratie de juiste codes:
- Type 12 → bankrekening(en) (bijv. "1100")
- Type 20 → debiteurenrekening (bijv. "1300")
- Type 22 → crediteurenrekening (bijv. "1600")

### 0b: Ontdek kruisposten/parkeerrekeningen

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Type": 90, "Description": {"contains": "kruis"}},
  "select": "Code,Description"
}
```

### 0c: Ontdek dagboeken

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
| 10 | Kas |
| 12 | Bank |
| 20 | Verkoop |
| 22 | Inkoop |
| 80 | Memoriaal |

**Let op**: Inkoopboek is Type 22, niet Type 21. Type 21 is niet in gebruik.

### 0d: Ontdek write-off rekeningen

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": {"Description": {"contains": "betalingsverschil"}},
  "select": "Code,Description"
}
```

Herhaal met "koersverschil" voor valutaverschillen.

## Aanpak 1: Exact Online REST API (execute_operation)

### Stap 1: Openstaande debiteuren via Receivables

```json
{
  "service": "Cashflow",
  "entity": "Receivables",
  "operation": "GET",
  "select": "ID,AccountName,InvoiceNumber,AmountDC,Description,EntryNumber,DueDate,InvoiceDate",
  "filters": {"AccountName": "{klantnaam}"}
}
```

Filter resultaten op `AmountDC != 0` (client-side):
- `AmountDC > 0` = openstaande vordering
- `AmountDC < 0` = onafgeletterd tegoed (betaling ontvangen, niet gematcht)
- `AmountDC = 0` = volledig afgeletterd

### Stap 2: Openstaande crediteuren via Payments

**Let op**: Het endpoint heet `Cashflow/Payments`, niet `Cashflow/Payables`.

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Payments",
  "operation": "GET",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,Description,DueDate,InvoiceDate,InvoiceNumber,EntryNumber,Status"
}
```

Status 20 = open, Status 30 = gedeeltelijk betaald, Status 50 = volledig afgeletterd.

### Stap 3: Betalingen zonder inkoopfactuur

**Patroon 1**: Crediteurenrekening + InvoiceNumber = null

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

De `InvoiceNumber: null` filter werkt API-side — retourneert direct alleen posten zonder factuur.

**Patroon 2**: Kruisposten/parkeerrekening

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

### Stap 4: Onafgeletterde facturen via OffsetID

Controleer de factuurzijde. **NIET controleren op het bankdagboek** — daar wijst de OffsetID altijd naar de bankrekening-regel (within-entry).

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

Filter client-side op `OffsetID == null` → onafgeletterde facturen.

### Stap 5: Bijbehorende bankbetalingen zoeken

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

Of via BankEntryLines voor een specifiek entrynummer:

```json
{
  "service": "Financialtransaction",
  "entity": "BankEntryLines",
  "operation": "GET",
  "filters": {"EntryNumber": "{entrynummer}"},
  "select": "ID,EntryNumber,Date,Description,AmountDC,GLAccountCode,AccountName"
}
```

### Stap 6: MatchSets uitvoeren (afletteren)

#### 1:1 Matching (exact bedrag)

```json
{
  "service": "Financial",
  "entity": "MatchSets",
  "operation": "POST",
  "data": {
    "matches": [
      {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
      {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
    ]
  }
}
```

De MCP tool toont eerst een preview. Voeg `"confirmed": true` toe om de aflettering uit te voeren.

#### N:1 Matching (één betaling, meerdere facturen)

```json
{
  "service": "Financial",
  "entity": "MatchSets",
  "operation": "POST",
  "confirmed": true,
  "data": {
    "matches": [
      {"line_id": "{factuur-A-id}", "line_type": "TransactionLine"},
      {"line_id": "{factuur-B-id}", "line_type": "TransactionLine"},
      {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
    ]
  }
}
```

#### Creditnota tegen factuur

```json
{
  "service": "Financial",
  "entity": "MatchSets",
  "operation": "POST",
  "confirmed": true,
  "data": {
    "matches": [
      {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
      {"line_id": "{creditnota-id}", "line_type": "TransactionLine"}
    ]
  }
}
```

#### Met write-off (klein verschil)

```json
{
  "service": "Financial",
  "entity": "MatchSets",
  "operation": "POST",
  "confirmed": true,
  "data": {
    "matches": [
      {"line_id": "{factuur-id}", "line_type": "TransactionLine"},
      {"line_id": "{bank-id}", "line_type": "BankEntryLine"}
    ],
    "write_off": {
      "gl_account_code": "{write-off-code}",
      "date": "2026-02-07"
    }
  }
}
```

Optioneel `type`: 3 = debet (kosten), 4 = credit (opbrengsten).

### Stap 7: Verifieer resultaat

```json
{
  "service": "Cashflow",
  "entity": "Receivables",
  "operation": "GET",
  "filters": {"InvoiceNumber": "{factuurnummer}"},
  "select": "ID,AccountName,InvoiceNumber,AmountDC"
}
```

Na succesvolle aflettering moet `AmountDC = 0` zijn.

## Aanpak 2: analyze_data

### Voordelen boven REST API

- Ondersteunt `IS NULL` filtering op OffsetID
- Ondersteunt OR-condities in één query
- Ondersteunt JOINs voor cross-referencing
- Veel sneller voor grote datasets

### Query: Onafgeletterde verkoopfacturen

```json
{
  "query": {
    "table": "Financial/TransactionLines",
    "filters": [
      {"column": "JournalCode", "operator": "=", "value": "{verkoopdagboek}"},
      {"column": "GLAccountCode", "operator": "=", "value": "{debiteurenCode}"},
      {"column": "FinancialYear", "operator": "=", "value": "{jaar}"}
    ],
    "select": ["EntryNumber", "Date", "Description", "AmountDC", "OffsetID", "AccountName"],
    "orderBy": [{"column": "Date", "direction": "DESC"}]
  }
}
```

Filter resultaten op `OffsetID IS NULL`.

### Query: Betalingen zonder inkoopfactuur

```json
{
  "query": {
    "table": "Financial/TransactionLines",
    "filters": [
      {"column": "JournalCode", "operator": "=", "value": "{bankdagboek}"},
      {"column": "GLAccountCode", "operator": "=", "value": "{crediteurenCode}"},
      {"column": "FinancialYear", "operator": "=", "value": "{jaar}"},
      {"column": "AmountDC", "operator": ">", "value": 0}
    ],
    "select": ["Date", "AccountName", "AmountDC", "Description", "EntryNumber", "InvoiceNumber"],
    "orderBy": [{"column": "Date", "direction": "DESC"}]
  }
}
```

Filter resultaten op `InvoiceNumber IS NULL`.

### Query: Kruisposten

```json
{
  "query": {
    "table": "Financial/TransactionLines",
    "filters": [
      {"column": "JournalCode", "operator": "=", "value": "{bankdagboek}"},
      {"column": "GLAccountCode", "operator": "=", "value": "{kruispostenCode}"},
      {"column": "FinancialYear", "operator": "=", "value": "{jaar}"}
    ],
    "select": ["EntryNumber", "Date", "Description", "AmountDC", "GLAccountCode"],
    "orderBy": [{"column": "Date", "direction": "DESC"}]
  }
}
```

**Let op**: De `IN`-operator in analyze_data kan falen (wordt `1=0`). Gebruik aparte queries per GLAccountCode.

### Query: Samenvatting per GLAccount

```json
{
  "query": {
    "table": "Financial/TransactionLines",
    "filters": [
      {"column": "JournalCode", "operator": "=", "value": "{bankdagboek}"},
      {"column": "FinancialYear", "operator": "=", "value": "{jaar}"}
    ],
    "select": ["GLAccountCode"],
    "aggregations": [
      {"function": "SUM", "column": "AmountDC", "alias": "TotaalBedrag"},
      {"function": "COUNT", "column": "ID", "alias": "AantalRegels"}
    ],
    "groupBy": ["GLAccountCode"],
    "orderBy": [{"column": "AantalRegels", "direction": "DESC"}]
  }
}
```
