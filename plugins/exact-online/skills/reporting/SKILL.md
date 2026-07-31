---
name: reporting
description: >
  Deze skill moet gebruikt worden wanneer de vraag gaat over de keuze van tool of endpoint in
  Exact Online, niet over de financiële inhoud zelf. Triggers: 'welk endpoint', 'waar haal ik
  dit uit', 'welke tool voor', 'analyze_data of read_operation', 'welke tabel gebruik ik',
  'hoe pagineer ik', 'welk endpoint voor openstaande posten', 'klopt dit endpoint wel'.
  Geeft per vraagsoort het juiste endpoint, de juiste JOIN en filters, plus de paginatie-
  en abonnementsregels.
---

# Endpoint- en toolkeuze

Gebruik deze skill om bij een vraag meteen het juiste endpoint en de juiste tool te kiezen,
in plaats van te gokken of te improviseren.

Deze skill gaat over de routing, niet over de inhoudelijke analyse. Voor een formele
resultatenrekening is `resultatenrekening-analyse` de juiste skill, voor stuurinformatie
en commerciële KPI's is dat `management-informatie`.

## Openstaande debiteuren en crediteuren

- Openstaande debiteuren: `Bulk/Cashflow/Receivables`, filter `Status` in `[20, 30]`.
- Openstaande crediteuren: `Bulk/Cashflow/Payments` (let op: dit endpoint heet "Payments"
  in de API, ook al gaat het om crediteuren), zelfde filter `Status` in `[20, 30]`.

Met `read_operation`:

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "filters": { "Status": [20, 30] },
  "select": "AccountName,AmountDC,DueDate,InvoiceNumber"
}
```

**Paginatie op Bulk-endpoints.** Bulk pagineert alleen via een cursor: `skip` geeft daar een
fout. Neem `next_page.page_token` uit de vorige respons en geef die mee in de volgende
`read_operation`. `top`, `skip`, `filters` en `select` worden dan genegeerd, de cursor draagt
ze al mee.

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "page_token": "<next_page.page_token uit de vorige respons>"
}
```

## Omzet, kosten en marge

Gebruik `analyze_data` op `Financial/TransactionLines` met een JOIN naar
`Financial/GLAccounts` op `GLAccountCode` = `Code`, gefilterd op `t1.BalanceType = 'W'`
(resultatenrekening). Kies daarna de grootboeksoort die bij de vraag hoort:

| Vraag | Filter |
|---|---|
| Omzet | `t1.Type = 110` |
| Kosten | `t1.Type in (111, 120, 121, 122, 123, 125, 126)` |
| Marge (omzet plus kostprijs) | `t1.Type in (110, 111)` |

Filter je alleen op `BalanceType = 'W'` en laat je `Type` weg, dan tel je de hele
resultatenrekening bij elkaar op en presenteer je omzet en kosten door elkaar als één bedrag.

**Meerdere boekjaren.** Voeg dan ook `t0.Type != 310` toe. De jaarafsluiting boekt elke
W&V-rekening van een afgesloten jaar tegen zichzelf terug, waardoor dat jaar zonder dit
filter op ongeveer nul uitkomt.

Gebruik hiervoor NIET `SalesInvoices`: dat endpoint mist creditnota's en handmatige
boekingen, waardoor de omzet onjuist uitvalt.

## Ouderdomsanalyse

Gebruik `Read/AgingReceivablesList` voor een kant-en-klare ouderdomsanalyse per bucket, in
plaats van dit zelf te berekenen uit losse facturen.

## Balans en kolommenbalans

Gebruik `Financial/ReportingBalance` voor het saldo per grootboekrekening, gegroepeerd naar
balanspost.

## analyze_data versus read_operation

Gebruik `analyze_data` zodra er meer dan circa 30 records geaggregeerd moeten worden
(bijvoorbeeld omzet over een heel jaar of over meerdere klanten). Voor kleinere, specifieke
opvragingen volstaat `read_operation`, dat maximaal 60 records per call teruggeeft.

`read_operation` doet alleen GET. Aanmaken, wijzigen en verwijderen gaat via
`write_operation`, met `confirmed: true`.

`analyze_data` en `list_available_tables` bestaan alleen op het Trial- en
Analytics-abonnement. Op het Essentials-abonnement val je terug op REST-rapportages via
`read_operation`.
