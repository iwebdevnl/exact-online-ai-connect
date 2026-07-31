---
description: Toon de openstaande debiteuren uit Exact Online
argument-hint: "[administratie]"
allowed-tools: ["mcp__plugin_exact-online-ai-connect_exact-online__get_started", "mcp__plugin_exact-online-ai-connect_exact-online__read_operation"]
---
Roep eerst `get_started` aan om te authenticeren en de administratie te kiezen.
Haal de openstaande debiteuren op via `read_operation` op
`Bulk/Cashflow/Receivables` met filter `Status` in `[20, 30]`.
Bulk-endpoints accepteren geen `skip`: pagineer met `page_token` uit
`next_page.page_token` van de vorige respons tot je alles hebt.
Vat samen als tabel: klant, factuurnummer, bedrag, vervaldatum, dagen open.
Optioneel argument (administratie): $ARGUMENTS
