---
description: Toon de openstaande debiteuren uit Exact Online
argument-hint: "[administratie?]"
---
Roep eerst `get_started` aan om te authenticeren en de administratie te kiezen.
Haal de openstaande debiteuren op via `execute_operation` op
`Bulk/Cashflow/Receivables` met filter `Status` in `[20, 30]`.
Vat samen als tabel: klant, factuurnummer, bedrag, vervaldatum, dagen open.
Optioneel argument (administratie): $ARGUMENTS
