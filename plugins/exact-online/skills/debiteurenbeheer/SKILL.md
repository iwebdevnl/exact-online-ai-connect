---
name: debiteurenbeheer
description: >
  Proactief debiteurenbeheer in Exact Online. Ouderdomsanalyse, herinnerings-workflow,
  escalatie bij langdurig openstaande posten, en concept-herinneringsmails genereren.
  Bouwt voort op de afletter-skills maar richt zich op opvolging en communicatie.

  Triggers: 'debiteurenbeheer', 'debiteuren opvolging', 'openstaande facturen opvolgen',
  'betalingsherinneringen', 'herinnering sturen', 'wie moet nog betalen', 'achterstallige facturen',
  'overdue invoices', 'aging report', 'ouderdomsanalyse', 'debiteurenrapport',
  'incasso voorbereiding', 'wanbetalers', 'slechte betalers', 'creditmanagement',
  'debiteurenrisico', 'betaalgedrag', 'DSO', 'days sales outstanding',
  'welke klanten betalen te laat', 'follow up facturen'.

  Gebruik deze skill wanneer de gebruiker iets wil doen met het opvolgen van openstaande
  debiteuren, betaalgedrag wil analyseren, of herinneringen wil voorbereiden. Werkt met
  Exact Online MCP (AgingReceivablesList, Receivables, Accounts, TransactionLines).
---

# Debiteurenbeheer

Proactief debiteurenbeheer: analyseer betaalgedrag, identificeer risico's, en bereid
herinneringsacties voor. Het doel is niet alleen zien wie te laat betaalt, maar ook
waarom en wat je eraan kunt doen.

## Waarom proactief debiteurenbeheer

Veel ondernemers kijken pas naar debiteuren als het geld opraakt. Door structureel te
monitoren verbeter je de cashflow, verlaag je het afschrijvingsrisico, en versterk je de
klantrelatie (een vriendelijke herinnering is beter dan een boze brief na 90 dagen).

## Analyse Workflow

### Stap 1: Totaaloverzicht ophalen

Start met een quick-scan via OutstandingInvoicesOverview voor de totalen:

```json
{
  "service": "Read",
  "entity": "OutstandingInvoicesOverview",
  "operation": "GET"
}
```

Dit geeft in één call:
- **OutstandingReceivableInvoiceCount/Amount**: Totaal aantal en bedrag openstaande facturen
- **OverdueReceivableInvoiceCount/Amount**: Waarvan verlopen (voorbij vervaldatum)
- **OutstandingPayableInvoiceCount/Amount**: Openstaande crediteuren (ter referentie)
- **OverduePayableInvoiceCount/Amount**: Waarvan verlopen crediteuren

### Stap 2: Ouderdomsanalyse per klant

Haal de ouderdomsanalyse op per klant:

```json
{
  "service": "Read",
  "entity": "AgingReceivablesList",
  "operation": "GET"
}
```

**BELANGRIJK**: Gebruik `AgingReceivablesList` (NIET `AgingReceivablesListByAgeGroup` —
dat endpoint retourneert altijd "Bad Request - Error in query syntax").

Dit geeft per klant een uitsplitsing naar:
- **AgeGroup1** (≤ 30 dagen): AgeGroup1Amount
- **AgeGroup2** (31-60 dagen): AgeGroup2Amount
- **AgeGroup3** (61-90 dagen): AgeGroup3Amount
- **AgeGroup4** (> 90 dagen): AgeGroup4Amount
- **TotalAmount**: Totaal openstaand per klant
- **AccountCode/AccountName**: Klantidentificatie
- **CurrencyCode**: Valuta (typisch EUR)

**Let op**: Dit endpoint heeft 0 filterbare velden — het retourneert altijd ALLE klanten
met openstaande posten. Sorteer/filter zelf in de output.

Voor een compact totaaloverzicht per ouderdomsgroep (zonder klantdetail), gebruik:

```json
{
  "service": "Read",
  "entity": "AgingOverview",
  "operation": "GET"
}
```

Dit geeft 5 rijen: Totaal + 4 ouderdomsgroepen, met zowel AmountReceivable als AmountPayable.

### Stap 3: Detail openstaande posten

Voor klanten met significante openstaande bedragen, haal de individuele facturen op:

**Via Cashflow/Receivables (aanbevolen)**:

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "operation": "GET",
  "filters": { "Status": [20, 30] },
  "select": "AccountName,AccountCode,AmountDC,DueDate,InvoiceNumber,InvoiceDate,Description,CurrencyCode"
}
```

**Let op AmountDC teken**: Openstaande debiteuren hebben een **negatief** AmountDC in
Cashflow/Receivables. Gebruik ABS(AmountDC) voor het te ontvangen bedrag.

**DueDate formaat**: De API retourneert datums in OData formaat: `/Date(1772150400000)/`.
Dit zijn milliseconden sinds Unix epoch (1 jan 1970). Converteer naar datum voor de analyse.

**Via ReceivablesList (alternatief)**:

```json
{
  "service": "Read",
  "entity": "ReceivablesList",
  "operation": "GET"
}
```

Dit geeft individuele openstaande regels met Amount (negatief), DueDate, JournalCode,
JournalDescription, InvoiceNumber, InvoiceDate, AccountName, Description, AmountInTransit.
Handig voor het identificeren van welk dagboek/bankrekening de post bevat.

**Let op**: ReceivablesList Amount is ook **negatief** voor openstaande debiteuren.

### Stap 4: Betaalgedrag analyseren (via analyze_data)

Analyseer het historische betaalgedrag per klant. Dit helpt bij het inschatten van risico's.
Check eerst of analyze_data gesynchroniseerd is via `list_available_tables`.

```json
{
  "table": "Financial/TransactionLines",
  "select": ["AccountName"],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Totaal_Gefactureerd" },
    { "function": "COUNT", "column": "ID", "alias": "Aantal_Facturen" }
  ],
  "groupBy": ["AccountName"],
  "filters": [
    { "column": "Type", "operator": "=", "value": 20 },
    { "column": "FinancialYear", "operator": "=", "value": 2026 }
  ]
}
```

**Let op**: Bij status `syncing` op analyze_data, val terug op de REST API endpoints
(AgingReceivablesList + Cashflow/Receivables) die altijd actueel zijn.

### Stap 5: DSO berekenen

Days Sales Outstanding (DSO) = (Openstaande debiteuren / Omzet over periode) × Aantal dagen.
Bereken dit per klant en overall. Een stijgende DSO is een waarschuwingssignaal.

Gebruik de OutstandingInvoicesOverview (Stap 1) voor het totale openstaande bedrag, en
analyze_data of ReportingBalance voor de omzet over de periode.

## Escalatie-workflow

Hanteer een gelaagde benadering afhankelijk van de ouderdom:

| Ouderdom | Actie | Toon |
|----------|-------|------|
| 1-14 dagen over vervaldatum | Vriendelijke herinnering | "Ter herinnering..." |
| 15-30 dagen | Tweede herinnering | "Wij verzoeken u vriendelijk..." |
| 31-60 dagen | Formele aanmaning | "Ondanks eerdere herinneringen..." |
| 61-90 dagen | Laatste waarschuwing | "Bij uitblijven van betaling..." |
| >90 dagen | Escalatie naar incasso | Adviseer externe incasso of juridische stappen |

### Concept-herinneringsmail genereren

Bij het genereren van herinneringsmails, neem altijd op:
- Factuurnummer(s) en bedrag(en)
- Originele factuurdatum en vervaldatum
- Totaal openstaand bedrag
- Bankgegevens voor betaling
- Contactgegevens voor vragen

Pas de toon aan op basis van de ouderdomscategorie en de klantrelatie.

## Rapportage

### Debiteurenrapport genereren

```
Debiteurenrapport — 16 februari 2026
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stand per [datum]

Totaal openstaand:                    € 43.213,48
  ≤ 30 dagen:     € 43.659,15  (101%)  ← incl. nog niet vervallen
  31-60 dagen:    €    240,15   (1%)
  61-90 dagen:    €    145,67   (0%)
  > 90 dagen:     €   -831,49  (-2%)   ← creditnota's/tegoeden

Verlopen facturen:  43 stuks  (€ 25.082,09)
DSO (overall):      XX dagen

Top 5 openstaand:
1. Klant A BV         € 2.500,00   (47 dagen over)  → 2e herinnering
2. Klant B NV         € 1.800,00   (12 dagen over)  → 1e herinnering
3. Klant C            €   950,00   (95 dagen over)  → escalatie
4. Klant D BV         €   750,00   (niet vervallen)
5. Klant E            €   600,00   (3 dagen over)   → afwachten

Aanbevelingen:
1. Stuur 2e herinnering naar Klant A BV (€ 2.500, 47 dagen over)
2. Escaleer Klant C naar incassobureau (€ 950, >90 dagen)
3. Onderzoek negatief saldo >90 dagen (€ -831) — mogelijke creditnota's
```

## Bekende API-eigenaardigheden

| Oorspronkelijk | Correct | Toelichting |
|----------------|---------|-------------|
| AgingReceivablesListByAgeGroup | AgingReceivablesList | ByAgeGroup variant geeft altijd "Bad Request - Error in query syntax" |
| Cashflow/Receivables AmountDC = positief | AmountDC = negatief | Gebruik ABS() voor het openstaande bedrag |
| ReceivablesList Amount = positief | Amount = negatief | Zelfde als Cashflow/Receivables: negatief voor debiteuren |
| DueDate als ISO datum | DueDate in OData format /Date(ms)/ | Milliseconden sinds epoch, converteer naar datum |
| analyze_data altijd actueel | analyze_data kan status 'syncing' hebben | Check list_available_tables; val terug op REST API endpoints |
| AgingReceivablesList filterbaar | 0 filterbare velden | Retourneert altijd alle klanten, sorteer/filter zelf |

## Communicatie

- Gebruik altijd het perspectief van de ondernemer ("uw klanten", niet "accounts")
- Bied aan om concept-herinneringsmails te schrijven
- Bij grote bedragen of lange termijnen: adviseer contact met accountant of jurist
- Bied aan om de bevindingen samen te vatten als takenlijst voor opvolging
- Vermeld altijd dat bedragen "stand per [datum]" zijn — ze kunnen dagelijks wijzigen
- Gebruik € Nederlands formaat (€ 1.234,56)
