# Query Patronen voor Afletteren

Aanvulling op SKILL.md. De discovery-, scan- en MatchSets-queries staan daar; dit bestand bevat
alleen wat er niet in staat: de losse BankEntryLines-lookup en de analyze_data-varianten.

## BankEntryLines opzoeken per entrynummer

Handig wanneer je van één bankboeking de losse regels nodig hebt in plaats van de
TransactionLines-selectie uit SKILL.md stap 1.3.

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "BankEntryLines",
  "filters": {"EntryNumber": "{entrynummer}"},
  "select": "ID,EntryNumber,Date,Description,AmountDC,GLAccountCode,AccountName"
}
```

Let op: voor MatchSets heb je het `TransactionLine ID` nodig, niet het `BankEntryLine ID`. Ze
kunnen toevallig gelijk zijn, maar daar mag je niet op rekenen.

## analyze_data-varianten

`analyze_data` is beschikbaar op Trial en Analytics, niet op Essentials. Voordelen boven
`read_operation`: OR-condities in één query, JOINs voor cross-referencing, en veel sneller op
grote datasets. De DSL gaat altijd onder een `query`-wrapper. Ondersteunde aggregaties: SUM,
COUNT, AVG, MIN, MAX, COUNT_DISTINCT.

`OffsetID` kan ook hier niet op leegheid worden gefilterd. Selecteer het veld en filter
client-side, net als bij `read_operation`.

### Query: Onafgeletterde verkoopfacturen

**Tool: `analyze_data`**

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

Filter de resultaten client-side op een lege `OffsetID`.

### Query: Betalingen zonder inkoopfactuur

**Tool: `analyze_data`**

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

Filter de resultaten client-side op een lege `InvoiceNumber`.

### Query: Kruisposten

**Tool: `analyze_data`**

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

**Tool: `analyze_data`**

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
