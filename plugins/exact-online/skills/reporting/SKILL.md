---
name: reporting
description: Kies het juiste Exact Online endpoint voor financiële vragen en rapportages.
---

# Routing voor financiële rapportages

Gebruik deze skill om bij een financiële vraag meteen het juiste endpoint te
kiezen, in plaats van te gokken of te improviseren.

## Openstaande debiteuren en crediteuren

- Openstaande debiteuren: `Bulk/Cashflow/Receivables`, filter `Status` in
  `[20, 30]`.
- Openstaande crediteuren: `Bulk/Cashflow/Payments` (let op: dit endpoint heet
  "Payments" in de API, ook al gaat het om crediteuren), zelfde filter
  `Status` in `[20, 30]`.

## Omzet, kosten en marge

Gebruik `analyze_data` op `Financial/TransactionLines` met een JOIN naar
`Financial/GLAccounts`, gefilterd op `BalanceType='W'` (resultatenrekening).
Gebruik hiervoor NIET `SalesInvoices`: dat endpoint mist creditnota's en
handmatige boekingen, waardoor de omzet onjuist uitvalt.

## Ouderdomsanalyse

Gebruik `Read/AgingReceivablesList` voor een kant-en-klare
ouderdomsanalyse per bucket, in plaats van dit zelf te berekenen uit losse
facturen.

## Balans en kolommenbalans

Gebruik `Financial/ReportingBalance` voor het saldo per grootboekrekening,
gegroepeerd naar balanspost.

## analyze_data versus execute_operation

Gebruik `analyze_data` zodra er meer dan circa 30 records geaggregeerd moeten
worden (bijvoorbeeld omzet over een heel jaar of over meerdere klanten). Voor
kleinere, specifieke opvragingen volstaat `execute_operation`.

`analyze_data` is alleen beschikbaar op het Trial- en Analytics-abonnement. Op
het Essentials-abonnement val je terug op REST-rapportages via
`execute_operation`.
