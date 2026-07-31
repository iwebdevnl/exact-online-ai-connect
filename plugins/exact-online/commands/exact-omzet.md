---
description: Omzet per klant per maand uit Exact Online
argument-hint: "[jaar]"
allowed-tools: ["mcp__plugin_exact-online-ai-connect_exact-online__get_started", "mcp__plugin_exact-online-ai-connect_exact-online__analyze_data", "mcp__plugin_exact-online-ai-connect_exact-online__read_operation"]
---
Roep eerst `get_started` aan. Gebruik `analyze_data` op
`Financial/TransactionLines` met een JOIN naar `Financial/GLAccounts` op
`GLAccountCode` = `Code`. Filter op `t1.BalanceType = 'W'` én op `t1.Type = 110`
(omzetrekeningen). Zonder die tweede filter tel je de hele resultatenrekening op
en rapporteer je kosten als omzet. Loopt de periode over meerdere boekjaren, voeg
dan `t0.Type != 310` toe: de jaarafsluiting boekt elke resultatenrekening van een
afgesloten jaar tegen zichzelf weg, waardoor dat jaar anders op nul uitkomt.
Groepeer op klant en maand, sommeer `AmountDC`. Toon de top-resultaten als tabel.
Is `analyze_data` niet beschikbaar (Essentials-plan), meld dat dan en val terug
op `read_operation`. Optioneel argument (jaar): $ARGUMENTS
