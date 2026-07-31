# Management rapport, uitvoerformaat

Gebruik dit sjabloon als de gebruiker om een samenhangend managementrapport vraagt. Laat
secties weg waarvoor geen data is opgehaald, verzin nooit rijen.

```
Management Informatie, [Maand/Kwartaal] [Jaar]
Stand per [datum], real-time uit Exact Online (analyze_data)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RESULTATENREKENING (YTD t/m [periode])
                          Dit jaar    Vorig jaar    Δ%
Netto-omzet:            €  xxx.xxx   €  xxx.xxx   +xx,x%
Inkoopkosten:           €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
Bruto marge:            €  xxx.xxx   €  xxx.xxx   +xx,x%
Bruto marge %:                xx,x%        xx,x%   +x,xpp

Personeelskosten:       €  xxx.xxx   €  xxx.xxx    +x,x%
Overige bedrijfsl.:     €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
EBIT:                   €  xxx.xxx   €  xxx.xxx   +xx,x%

OMZET PER CATEGORIE (dit kwartaal vs vorig kwartaal)
                          Q huidig    Q vorig       Δ%
Abonnementen:           €  xxx.xxx   €  xxx.xxx   +xx,x%
Maatwerk:               €  xxx.xxx   €  xxx.xxx   +xx,x%
Overig:                 €  xxx.xxx   €  xxx.xxx   +xx,x%
────────────────────────────────────────────────────────
Totaal:                 €  xxx.xxx   €  xxx.xxx   +xx,x%

COMMERCIËLE KPI's
  Recurring %:                xx%  (vorig kwartaal: xx%)
  Top 5 klantconcentratie:    xx%  (vorig kwartaal: xx%)

TOP 5 KLANTEN (omzet periode)
  1. Klant A BV         € xx.xxx  (xx,x%)
  2. Klant B NV         € xx.xxx  (xx,x%)
  ...

TOP 5 GROEIERS                    TOP 5 DALERS
  Klant X  +€ xx.xxx (+xxx%)        Klant Y  -€ x.xxx (-xx%)
  ...                                ...

AANDACHTSPUNTEN:
[Signaleringen op basis van de opgehaalde data, zie references/benchmarks.md]
```

De blokletter-samenvatting boven in het rapport is een verkorte weergave, geen formele
resultatenrekening. Vraagt de gebruiker om een W&V conform BW2 Titel 9, gebruik dan de skill
`resultatenrekening-analyse`.

**Vergelijkingen over meerdere boekjaren:** de kolom "Vorig jaar" komt uit een `analyze_data`
query die ook `t0.Type != 310` filtert. Zonder dat filter staat er voor een afgesloten jaar
ongeveer nul en zijn alle Δ-percentages onzin.
