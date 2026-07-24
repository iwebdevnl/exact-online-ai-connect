---
description: Omzet per klant per maand uit Exact Online
argument-hint: "[jaar?]"
---
Roep eerst `get_started` aan. Gebruik `analyze_data` op
`Financial/TransactionLines` met een JOIN naar `Financial/GLAccounts` en filter
`BalanceType='W'` (resultatenrekening). Groepeer op klant en maand, sommeer
`AmountDC`. Toon de top-resultaten als tabel. Als de Analytics-functie niet
beschikbaar is (Essentials-plan), meld dat en val terug op een REST-overzicht.
Optioneel argument (jaar): $ARGUMENTS
