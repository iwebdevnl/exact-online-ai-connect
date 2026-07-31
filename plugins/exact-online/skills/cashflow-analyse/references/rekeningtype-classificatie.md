# Rekeningtype → kasstroomcategorie (Exact Online)

Dit is de **officiële, volledige** `GLAccount.Type`-lijst (uit het entiteitsschema). Deze codes
zijn platform-breed identiek in elke administratie. Classificeer hierop, niet op rekeningnummer
of -naam. De generator
(`${CLAUDE_PLUGIN_ROOT}/skills/cashflow-analyse/scripts/build_cashflow_workbook.py`) bevat exact
dezelfde map.

Kasvuistregel: **kaseffect = − mutatie(AmountDC)** voor elke niet-liquide rekening.

| Type | Omschrijving (EN) | Balans/W&V | Categorie | Subcategorie |
|---|---|---|---|---|
| 10 | Cash | B | Liquide middelen | Kas |
| 12 | Bank | B | Liquide middelen | Bank |
| 14 | Credit card | B | Liquide middelen | Creditcard |
| 16 | Payment services | B | Liquide middelen | Payment service (PSP) |
| 20 | Accounts receivable | B | Operationeel | Werkkapitaal: debiteuren |
| 21 | Prepayment accounts receivable | B | Operationeel | Werkkapitaal: vooruitbetaald aan debiteuren |
| 22 | Accounts payable | B | Operationeel | Werkkapitaal: crediteuren |
| 24 | VAT | B | Operationeel | Werkkapitaal: btw |
| 25 | Employees payable | B | Operationeel | Werkkapitaal: personeel te betalen |
| 26 | Prepaid expenses | B | Operationeel | Werkkapitaal: vooruitbetaalde kosten |
| 27 | Accrued expenses | B | Operationeel | Werkkapitaal: nog te betalen kosten |
| 29 | Income taxes payable | B | Operationeel | Werkkapitaal: vennootschapsbelasting te betalen |
| 40 | Inventory | B | Operationeel | Werkkapitaal: voorraad |
| 100 | Tax payable | B | Operationeel | Werkkapitaal: belasting te betalen |
| 35 | Accumulated depreciation | B | Operationeel | Afschrijving (terugname, non-cash) |
| 90 | General | B | Operationeel | Overig werkkapitaal (controleer) |
| 30 | Fixed assets | B | Investering | (Des)investering vaste activa |
| 32 | Other assets | B | Investering | Overige langlopende activa |
| 50 | Capital stock | B | Financiering | Eigen vermogen: kapitaal |
| 52 | Retained earnings | B | Financiering | Eigen vermogen: winstreserve/uitkering |
| 55 | Long term debt | B | Financiering | Langlopende schuld |
| 60 | Current portion of debt | B | Financiering | Kortlopend deel langlopende schuld |
| 110 | Revenue | W | Operationeel (resultaat) | Omzet |
| 111 | Cost of goods | W | Operationeel (resultaat) | Kostprijs omzet |
| 120 | Other costs | W | Operationeel (resultaat) | Overige kosten |
| 121 | Sales, general & admin. expenses | W | Operationeel (resultaat) | Verkoop/algemeen/beheer |
| 122 | Depreciation costs | W | Operationeel (resultaat) | Afschrijvingskosten (non-cash) |
| 123 | Research and development | W | Operationeel (resultaat) | R&D |
| 125 | Employee costs | W | Operationeel (resultaat) | Personeelskosten |
| 126 | Employment costs | W | Operationeel (resultaat) | Werkgeverslasten |
| 130 | Exceptional costs | W | Operationeel (resultaat) | Bijzondere lasten |
| 140 | Exceptional income | W | Operationeel (resultaat) | Bijzondere baten |
| 150 | Income taxes | W | Operationeel (resultaat) | Belasting over resultaat |
| 160 | Interest income | W | Operationeel (resultaat) | Rentebaten/-lasten |
| 300 | Year end reflection | B/W | Technisch | Jaareinde-spiegeling (uitsluiten) |
| 301 | Indirect year end costing | B/W | Technisch | Jaareinde-kostenverdeling (uitsluiten) |
| 302 | Direct year end costing | B/W | Technisch | Jaareinde-kostenverdeling (uitsluiten) |

## Aandachtspunten bij classificatie

- **Afschrijving (35 vs 122).** Voor een nette presentatie hoort de afschrijving als terugname
  in de operationele kasstroom en de bruto-investering in de investeringskasstroom. Daarom:
  - Type 35 (cumulatieve afschrijving) → operationeel (terugname).
  - Type 30/32 (activa) → investering, bruto.
  - Type 122 (afschrijvingskosten, W) staat in het resultaat; de terugname via 35 compenseert
    dit. **Heeft de administratie geen Type 35** (afschrijving wordt direct op 30 geboekt), leid
    de terugname dan af uit Type 122 en presenteer de investeringskasstroom netto, met een notitie.
- **Winstbestemming (52).** Het overboeken van het resultaat naar de winstreserve is non-cash en
  valt binnen het eigen vermogen weg (beide benen non-cash). Alleen werkelijke uitkeringen
  (dividend) zijn een financieringsuitstroom. Bij een analyse binnen het lopende jaar speelt dit
  meestal niet; markeer grote 52-mutaties ter controle.
- **Type 90 (General).** Verzamel-/tussenrekeningen. Standaard operationeel, maar markeer grote
  mutaties: ze kunnen investering of financiering betreffen.
- **Rente (160) en belasting (150).** Standaard operationeel. Wil de gebruiker een IFRS-stijl
  presentatie, dan kunnen rente en belasting als aparte regels of secties getoond worden.
- **Technisch (300 tot 302).** Jaareinde-rubriceringen; deze zijn non-cash en moeten worden
  uitgesloten zodat ze de kasstroom niet vervuilen.
- **Onbekend Type.** Komt een `Type` voor dat hier niet staat, classificeer als "Niet
  geclassificeerd", neem het mee in de aansluitcontrole en meld het expliciet aan de gebruiker.
