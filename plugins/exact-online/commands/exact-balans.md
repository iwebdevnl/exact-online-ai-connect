---
description: Balans / grootboeksaldi uit Exact Online
argument-hint: "[periode]"
allowed-tools: ["mcp__plugin_exact-online-ai-connect_exact-online__get_started", "mcp__plugin_exact-online-ai-connect_exact-online__read_operation"]
---
Roep eerst `get_started` aan. Haal de balans op via `read_operation` op het
rapportage-endpoint `Financial/ReportingBalance`. Toon per grootboekrekening het
saldo, gegroepeerd naar balanspost. `read_operation` geeft maximaal 60 rijen: kom
je precies op 60 uit, meld dan dat het overzicht is afgekapt en verfijn per
periode of per rekening. Optioneel argument (periode): $ARGUMENTS
