---
name: resultatenrekening-analyse
description: >
  Deze skill moet gebruikt worden wanneer om een formele resultatenrekening uit Exact Online
  wordt gevraagd. Classificatie gebeurt op GLAccount.Type en niet op rekeningnummers, en de
  structuur volgt het categorale model van BW2 Titel 9. Triggers: 'resultatenrekening',
  'winst- en verliesrekening', 'W&V', 'P&L', 'profit and loss', 'income statement',
  'exploitatieoverzicht', 'maandresultaat', 'kwartaalresultaat', 'jaarresultaat',
  'resultatenrekening maken', 'BW2 Titel 9', 'jaarrekening model', 'RGS'.
---

# Resultatenrekening Analyse

Genereer een volledige, correcte resultatenrekening vanuit Exact Online. Classificatie is
volledig gebaseerd op het `Type` veld van de GLAccounts API, NIET op rekeningnummerreeksen.

Classificeren op nummerbereik (4xxx = kosten, 8xxx = omzet) is onbetrouwbaar: administraties
zijn vrij in hun rekeningschema, en 9xxx-rekeningen (koersverschillen, rente, belasting)
worden dan gemist. Het `Type` veld sluit exact aan bij de posten van een categorale
resultatenrekening.

## Rapportagestructuur en Type-mapping

Volgorde conform BW2 Titel 9 (Besluit modellen jaarrekening, Model I). De volgorde staat
vast, uitsplitsen mag.

| Type | Beschrijving (API) | Rapportagesectie |
|------|-----------------------------------------|---------------------------|
| 110 | Revenue | 1. Netto-omzet |
| 111 | Cost of goods | 2. Kostprijs van de omzet |
| 125 | Employee costs | 3. Personeelskosten |
| 122 | Depreciation costs | 4. Afschrijvingen |
| 121 | Sales, general administrative expenses | 5. Overige bedrijfskosten |
| 123 | Research and development | 5. Overige bedrijfskosten |
| 126 | Employment costs | 5. Overige bedrijfskosten |
| 160 | Interest income | 6. Financiële baten en lasten |
| 120 | Other costs | 7. Buitengewone baten en lasten |
| 130 | Exceptional costs | 7. Buitengewone baten en lasten |
| 140 | Exceptional income | 7. Buitengewone baten en lasten |
| 150 | Income taxes | 8. Belastingen |
| 90 | General | 9. Overig |

Tussentellingen tussen de secties: Bruto-marge (na 2), Bedrijfsresultaat (na 5), Resultaat
voor belastingen (na 7), Nettoresultaat (na 9). De exacte formules staan in
`references/bw2-model.md`.

## Sign conventions

Gebruik de `BalanceSide` van de GLAccount om te bepalen of een bedrag geflipt moet worden:
`"C"` (credit) betekent flippen naar positief voor weergave, `"D"` (debit) blijft ongewijzigd.

- Type 110 (omzet): negatief in Exact (credit), flip voor weergave.
- Type 111, 121, 122, 123, 125, 126, 130, 150 (kosten): positief (debit), toon als kosten.
- Type 140 (buitengewone baten): negatief (credit), flip.
- Type 160 (financieel) en Type 90 (general): bevatten zowel baten als lasten, bepaal per
  rekening op `BalanceSide`.

## Stap 1: bepaal de scope

Vraag, of leid af uit de context: welk boekjaar (standaard het huidige), welke periodes
(standaard alle beschikbare) en of de gebruiker wil vergelijken met vorig jaar.

## Stap 2: haal de GLAccounts-classificatie op

Dit is verplicht en gaat vooraf aan alles. Zonder de Type-informatie kun je de rekeningen
niet indelen.

Met `read_operation`:

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": { "BalanceType": "W" },
  "select": "Code,Description,Type,TypeDescription,BalanceSide",
  "top": 60
}
```

Pagineer tot je alle W&V-rekeningen hebt (`skip: 0`, `skip: 60`, `skip: 120`, stop zodra het
aantal records kleiner is dan `top`). Bewaar als lookup: `Code` naar
`{Type, TypeDescription, BalanceSide}`.

Optioneel: verrijk met RGS-subcategorieën voor een fijnmazigere uitsplitsing binnen een
sectie. Zie `references/rgs-classificatie.md`.

## Stap 3: haal de saldi op via read_operation

Dit is de normale route. `Financial/ReportingBalance` en de `Read/`-rapportage-endpoints
zijn niet opgenomen in de analyse-omgeving, dus de REST-weg is hier de werkende weg.

Met `read_operation`:

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "filters": {
    "ReportingYear": 2026,
    "ReportingPeriod": 3,
    "BalanceType": "W"
  },
  "select": "ID,GLAccountCode,GLAccountDescription,ReportingPeriod,Amount,BalanceType",
  "top": 60
}
```

**Kritieke regels:**

1. **Pagineer volledig.** `read_operation` geeft maximaal 60 records per call. ReportingBalance
   bevat 50 tot 120 W&V-records per periode, dus 1 tot 2 pagina's. Stop zodra het aantal
   records kleiner is dan `top`.
2. **Herhaal per periode.** Doe dit voor elke maand in scope.
3. **Aggregeer per GLAccountCode plus ReportingPeriod.** Eén rekening kan meerdere records per
   periode hebben (meerdere dagboeken, kostenplaatsen).
4. Voor cumulatief YTD: laat `ReportingPeriod` weg uit het filter.

## Stap 4 (optioneel): analyze_data als versneller

Alleen als `list_available_tables` `Financial/ReportingBalance` toont met
`sync_status = idle`. Dat is ongebruikelijk, deze tabel wordt doorgaans niet
gesynchroniseerd. Controleer dus eerst, en ga bij twijfel gewoon door met Stap 3.

`analyze_data` bestaat bovendien alleen op het Trial- en Analytics-abonnement.

Met `analyze_data`:

```json
{
  "query": {
    "table": "Financial/ReportingBalance",
    "select": ["GLAccountCode", "GLAccountDescription", "ReportingPeriod", "Amount", "BalanceType"],
    "filters": [
      { "column": "ReportingYear", "operator": "=", "value": 2026 },
      { "column": "BalanceType", "operator": "=", "value": "W" }
    ],
    "orderBy": [
      { "column": "GLAccountCode", "direction": "ASC" },
      { "column": "ReportingPeriod", "direction": "ASC" }
    ],
    "limit": 10000
  }
}
```

Slaagt dit, dan heb je het hele jaar in één call en kun je Stap 3 overslaan.

## Stap 5: classificeer en structureer

Koppel elke `GLAccountCode` aan het Type uit de lookup van Stap 2 en plaats de rekening in de
bijbehorende sectie uit de tabel bovenaan. Onbekende of ontbrekende Types gaan naar sectie 9
(Overig), met een waarschuwing in de uitvoer.

Pas per sectie de sign convention toe. Voor secties 6, 7 en 9 geldt de per-rekening-regel op
`BalanceSide`, de overige secties hebben een vast teken.

Binnen Overige bedrijfskosten (Type 121, 123, 126) is een subsectie-indeling gewenst. Met
RGS-mapping komen die subsecties uit de ClassificationDescription, zonder RGS gebruik je de
`TypeDescription` uit de API als label. Zie `references/rgs-classificatie.md`.

## Stap 6: bereken de tussentellingen

Zie `references/bw2-model.md` voor de vijf verplichte tussentellingen. Bevat een sectie geen
data, sla die sectie en de bijbehorende tussentelling over.

## Stap 7: lever de rapportage

Beschikt de omgeving over een spreadsheet- of xlsx-vaardigheid, lever dan een Excel-bestand
met twee tabbladen ("Per Grootboekrekening" en "Samenvatting"). Ontbreekt die vaardigheid,
lever de resultatenrekening dan als tabel in het antwoord met dezelfde secties, subtotalen en
tussentellingen, en meld dat een Excel-export in deze omgeving niet beschikbaar is.

Ga nooit uit van een vast bestandspad voor een xlsx-vaardigheid. De volledige opmaakspecificatie
staat in `references/excel-opmaak.md`.

## Stap 8: validatie

1. **Type-check:** elke rekening heeft een bekend Type uit GLAccounts. Log onbekende Types.
2. **Sign-check:** omzet (Type 110) is negatief in de brondata, kosten positief.
3. **Balans-check:** nettoresultaat is de som van alle W&V-bedragen met het juiste teken.
4. **YTD-check:** per rekening moet de som van de maandkolommen gelijk zijn aan de YTD-kolom.
5. **Vergelijk met Exact:** levert de gebruiker een export aan, vergelijk regel voor regel.

## Veelvoorkomende valkuilen

1. **Classificeren op rekeningnummer in plaats van GLAccount.Type.** Rekeningnummers zijn niet
   gestandaardiseerd tussen administraties.
2. **De GLAccounts-lookup overslaan.** Zonder Stap 2 is correcte sectie-indeling onmogelijk.
3. **Niet alle pagina's ophalen.** `read_operation` stopt bij 60 records, ook als er meer zijn.
4. **Dubbele records niet aggregeren.** Aggregeer per GLAccountCode plus ReportingPeriod.
5. **Sign convention vergeten.** Gebruik `BalanceSide`, geen aanname op basis van het nummer.
6. **Structuur niet conform BW2.** Volgorde en tussentellingen liggen vast: Omzet, Kostprijs,
   Bruto-marge, Bedrijfskosten, Bedrijfsresultaat, Financieel, Buitengewoon, Belasting,
   Resultaat.

## Verdieping

| Onderwerp | Bestand |
|---|---|
| BW2 Titel 9-model en de tussentellingsformules | `references/bw2-model.md` |
| RGS-subcategorieën via GLAccountClassificationMappings | `references/rgs-classificatie.md` |
| Excel-opbouw, opmaak en de fallback zonder xlsx | `references/excel-opmaak.md` |
