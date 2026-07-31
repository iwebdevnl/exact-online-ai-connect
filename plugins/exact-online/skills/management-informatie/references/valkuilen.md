# Bekende API-eigenaardigheden en valkuilen

| Valkuil | Correct | Toelichting |
|---------|---------|-------------|
| SalesInvoices voor bedragen | `analyze_data` op `Financial/TransactionLines` | SalesInvoices bevat bruto factuurbedragen, niet de geboekte omzet |
| GLAccount filteren op code-ranges | Filter op `t1.Type` via de JOIN | Rekeningnummers variëren per administratie |
| Alleen filteren op `BalanceType = 'W'` | Voeg altijd de `t1.Type`-selectie toe | Zonder `Type` tel je de hele resultatenrekening op en meng je omzet met kosten |
| Meerdere boekjaren zonder extra filter | Voeg `t0.Type != 310` toe | De jaarafsluiting boekt elke W&V-rekening van een afgesloten jaar tegen zichzelf terug, dat jaar komt anders op ongeveer nul uit |
| GLAccounts filteren op `Classification` | Gebruik `BalanceType` (W of B) | Het veld `Classification` bestaat niet in de GLAccounts entity |
| `service: "Financial"` voor TransactionLines | In `read_operation`: `service: "FinancialTransaction"` | In `analyze_data` is het gewoon `Financial/TransactionLines` als table |
| JOIN op `GLAccount` = `ID` | JOIN op `GLAccountCode` = `Code` | Dit is de gedocumenteerde sleutel voor de JOIN tussen TransactionLines en GLAccounts |
| ReportingBalance Amount = saldo | Amount = mutatie van die periode | Laat `ReportingPeriod` weg voor cumulatief YTD |
| Omzet Amount = positief | Omzet Amount = negatief (credit) | ABS() nodig voor weergave, kosten zijn positief. In `analyze_data` kan `sign: -1` op de aggregatie dit direct omdraaien |
| ReceivablesList Amount-teken = richting | Combineer `JournalCode` met het Amount-teken | Bankdagboek is een tegoed, verkoopboek positief is een vordering |
| CostCenters altijd beschikbaar | CostCenters zijn optioneel | Niet elke administratie gebruikt kostenplaatsen |
| Banksaldo via codes 1000-1099 | Haal de liquide rekeningen op via `GLAccount.Type` in {10, 12, 14, 16} | Een codebereik mist rekeningen buiten het bereik en vergelijkt bovendien als tekst: `"10001"` en `"1050A"` vallen binnen `$gte: "1000"` / `$lte: "1099"` |
| `skip` gebruiken op een Bulk-endpoint | Pagineer met `page_token` uit `next_page.page_token` | Bulk-endpoints pagineren alleen via een cursor en geven een fout op `skip` |
