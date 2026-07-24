---
name: cashflow-analyse
description: >
  Generiek kasstroomoverzicht en cashflow-analyse uit Exact Online, zonder administratie-specifieke
  grootboeknummers of -namen: classificeert op rekeningtype (platform-breed in Exact, identiek per
  administratie). Bouwt het overzicht volgens de directe en indirecte methode, met operationele,
  investerings- en financieringskasstroom, werkkapitaal-KPI's (DSO, DPO, DIO, kasconversiecyclus),
  vrije kasstroom en een beknopte vooruitblik. Levert een Excel-werkmap met dashboard, beide
  methoden, trends en onderbouwing. Triggers: 'cashflow-analyse', 'kasstroomoverzicht',
  'kasstroom analyse', 'cashflow rapport', 'directe methode', 'indirecte methode',
  'operationele kasstroom', 'vrije kasstroom', 'free cash flow', 'werkkapitaal analyse',
  'kasconversiecyclus', 'DSO DPO', 'waar ging ons geld heen', 'kasstroom per maand',
  'liquiditeitsanalyse'. Voor een gedetailleerde 13-weeks liquiditeitsprognose kun je de
  analyse verder uitwerken.
---

# Cashflow-analyse — generiek kasstroomoverzicht (directe + indirecte methode)

Deze skill bouwt een volledig kasstroomoverzicht en cashflow-analyse uit een Exact Online-
administratie. Hij werkt in **elke** administratie omdat hij classificeert op **rekeningtype**,
niet op grootboeknummers of -namen.

## Waarom dit generiek is (lees dit eerst)

Grootboeknummers en -namen verschillen per administratie: 8000 is niet altijd omzet, 4200 is
niet altijd afschrijving. Wat wél in elke Exact Online-administratie identiek is, is het veld
`Type` op de grootboekrekening — een vaste, platform-brede classificatie (bijv. 12 = Bank,
20 = Debiteuren, 30 = Vaste activa, 110 = Omzet). **Classificeer altijd op `Type`, nooit op
nummer of omschrijving.** Zo blijft de analyse correct, ongeacht het rekeningschema.

> Let op: de richtlijn-tekst van de Exact-koppeling bevat een verkorte, deels afwijkende
> Type-lijst. Gebruik de volledige, officiële lijst uit `reference/rekeningtype-classificatie.md`
> (die komt uit het entiteitsschema en is leidend).

## Het rekenhart: één identiteit die alles sluitend maakt

Elke boeking is in evenwicht (debet = credit). Per periode geldt daarom over álle rekeningen:

```
Σ mutatie(alle rekeningen) = 0
⇒ mutatie(liquide middelen) = − Σ mutatie(alle niet-liquide rekeningen)
```

De mutatie op de liquide rekeningen (Type 10/12/14/16) **is** de netto kasstroom van de
periode. Diezelfde uitkomst wordt door de indirecte methode heropgebouwd uit resultaat,
afschrijvingen, werkkapitaal, investeringen en financiering. Hierdoor sluiten de directe en
indirecte methode per definitie op elkaar aan — en dat is meteen de ingebouwde controle.

**Kasvuistregel:** het kaseffect van elke niet-liquide rekening = **− mutatie (AmountDC)**.
Een toename van een actief (debet, +AmountDC) kost geld (−). Een toename van een schuld
(credit, −AmountDC) levert geld op (+).

## Werkwijze

### Stap 0 — Verbinden en scope bepalen
1. Roep `get_started` aan (regelt inlog en administratiekeuze).
2. Vraag de gebruiker (of leid af): welk jaar/periode, en welke granulariteit (per maand is
   standaard). Vraag of er meerdere jaren vergeleken moeten worden.

### Stap 1 — Liquide rekeningen en beginsaldo
- Bepaal de liquide middelen: rekeningen met `Type` in 10, 12, 14, 16.
- Beginsaldo liquide middelen = openingsbalans + mutaties t/m start van de periode. Gebruik
  `Financial/ReportingBalance` of de openingsbalans; tel de mutaties van eerdere jaren op.
- **Vraag de gebruiker altijd of het berekende beginsaldo klopt met de werkelijke bankstand.**

### Stap 2 — De bewegingsmatrix ophalen (de kern)
Haal in één analyse de mutatie per rekeningtype per maand op (Analytics/Trial-tier vereist):

```json
{
  "table": "Financial/TransactionLines",
  "joins": [{
    "table": "Financial/GLAccounts", "type": "INNER",
    "on": {"leftColumn": "GLAccount", "rightColumn": "ID"},
    "select": ["Type", "TypeDescription", "BalanceType"]
  }],
  "aggregations": [{"function": "SUM", "column": "AmountDC", "alias": "Mutatie"}],
  "groupBy": ["t1.Type", "t1.TypeDescription", "t1.BalanceType"],
  "dateGroupBy": [{"column": "Date", "part": "MONTH", "alias": "Maand"}],
  "filters": [{"column": "FinancialYear", "operator": "=", "value": <jaar>}],
  "limit": 1000
}
```

Controleer: de som van `Mutatie` over alle rijen van een periode moet **0** zijn (double-entry).
Wijkt het af, meld dat en stop — de data is dan onvolledig gesynchroniseerd.

Zonder Analytics-tier: gebruik `Financial/ReportingBalance` per periode en map elke
`GLAccountCode` naar zijn `Type` via `Financial/GLAccounts` (haal de rekeningen één keer op).

### Stap 3 — Openstaande posten (voor KPI's en vooruitblik)
- Debiteuren: `Bulk` / `Cashflow/Receivables`, `Status [20,30]`, select
  `AccountName,AmountDC,DueDate,InvoiceNumber,InvoiceDate`. **AmountDC is negatief → ABS().**
- Crediteuren: `Bulk` / `Cashflow/Payments`, `Status [20,30]` (Payables heet *Payments*).

### Stap 4 — Classificeren en beide methoden opbouwen
Map elk `Type` naar een kasstroomcategorie volgens `reference/rekeningtype-classificatie.md`.
Lees `reference/methodologie.md` voor de exacte opbouw van de indirecte methode (resultaat +
afschrijvingen + werkkapitaal), de afgeleide directe methode, en de KPI-formules.

Kernpunten:
- **Indirect** = nettoresultaat + afschrijvingen (terug) ± werkkapitaalmutaties (operationeel),
  −investeringen (investering), ± vermogen/leningen (financiering).
- **Direct (afgeleid)** = ontvangsten van klanten, betalingen aan leveranciers/voorraad,
  lonen, belastingen — afgeleid uit resultaatposten gecorrigeerd voor de bijbehorende
  werkkapitaalmutaties. Sluit per definitie aan op de operationele kasstroom van de indirecte
  methode.
- **Onbekende types**: elk `Type` dat niet in de map staat → toon als aparte regel "Niet
  geclassificeerd" en meld het. Nooit stil weglaten.

### Stap 5 — Excel-werkmap genereren
Schrijf de opgehaalde data naar een JSON-invoerbestand en draai de generator:

```bash
python scripts/build_cashflow_workbook.py invoer.json uitvoer.xlsx
python /pad/naar/xlsx/scripts/recalc.py uitvoer.xlsx   # herbereken + check op fouten
```

De generator zet de ruwe matrix op een **Brondata**-tab en een **Classificatie**-tab, en bouwt
alle overzichten met **SUMIFS-formules** daarop. Zo is elk getal herleidbaar en herrekent de
werkmap automatisch. Zie het kopje "Invoerformaat" onderaan. Controleer na `recalc.py` dat er
0 formulefouten zijn en dat de aansluitcontrole klopt (operationeel + investering + financiering
= mutatie liquide middelen).

### Stap 6 — Beknopte vooruitblik
Houd dit licht (een gedetailleerde 13-weeks prognose is een aparte, diepere exercitie):
- Neem de gemiddelde operationele maandkasstroom van de laatste 3–6 maanden als run-rate.
- Tel de zekere posten erbij: openstaande debiteuren op vervaldatum (inkomend) en crediteuren
  op vervaldatum (uitgaand), de komende ~3 maanden.
- Toon één base-scenario met het verwachte eindsaldo per maand en markeer waar het saldo onder
  een veilige buffer (≈ 1,5× gemiddelde maanduitgaven) zou komen. Noem dit "indicatief".

## KPI's die in elke analyse horen
- **DSO** (debiteurendagen) = openstaande debiteuren / omzet × dagen-in-periode.
- **DPO** (crediteurendagen) = openstaande crediteuren / inkoop × dagen-in-periode.
- **DIO** (voorraaddagen) = voorraad / inkoopwaarde omzet × dagen (alleen bij voorraad).
- **Kasconversiecyclus (CCC)** = DSO + DIO − DPO.
- **Operationele kasstroom (OCF)** en **vrije kasstroom (FCF)** = OCF − investeringen.
- **Kasstroommarge** = OCF / omzet. **Operationele-kasstroomratio** = OCF / kortlopende schulden.
- **Cash runway** = liquide middelen / gemiddelde netto kasuitstroom per maand (indien negatief).

## Communicatie
- Schrijf "verwacht"/"indicatief" voor de vooruitblik, nooit "zeker".
- Leg uit dat de directe methode de werkelijke geldstromen toont en de indirecte methode laat
  zien hoe het resultaat zich tot de kasstroom verhoudt; benoem dat beide op elkaar aansluiten.
- Presenteer cijfers als "stand per [datum]", niet als definitief; afgesloten periodes kunnen
  nog corrigeren.
- Vraag of het beginsaldo klopt met de werkelijke bankstand.
- Bied de Excel-werkmap aan en vat de 3–5 belangrijkste bevindingen samen.

## Bekende eigenaardigheden (Exact-koppeling)
| Verwachting | Werkelijkheid | Oplossing |
|---|---|---|
| `Type` is een getal | Wordt als **string** opgeslagen in analytics | Filter met `["10","12",...]`, niet `[10,12,...]` |
| `IN`-filter op join-kolom werkt | Geeft soms een cast-fout | Haal de hele matrix op (group by `Type`) en bucket in nabewerking |
| Afschrijving altijd op aparte rekening | Soms direct op de activarekening | Geen Type 35? Leid de terugname af uit Type 122 (afschrijvingskosten) |
| Receivables AmountDC positief | Is **negatief** | ABS() bij presentatie |
| Alle W-posten = operationeel | Rente/belasting soms apart | Standaard operationeel; benoem als de gebruiker IFRS-splitsing wil |
| Banksaldo uit mutaties | ReportingBalance = mutatie, geen saldo | Beginsaldo = openingsbalans + cumulatieve mutaties |

## Invoerformaat voor de generator
`scripts/build_cashflow_workbook.py` verwacht JSON met:
```json
{
  "bedrijf": "Naam", "administratie": "234", "valuta": "EUR",
  "jaar": 2026, "periode_label": "2026 (t/m mei)",
  "beginsaldo_liquide": 0.0, "beginsaldo_bevestigd": false,
  "matrix": [{"type": "110", "omschrijving": "Revenue", "balance_type": "W", "maand": 1, "mutatie": -213645.27}],
  "debiteuren": [{"naam": "...", "bedrag": 1000.0, "vervaldatum": "2026-06-30"}],
  "crediteuren": [{"naam": "...", "bedrag": 1000.0, "vervaldatum": "2026-06-15"}],
  "vorig_jaar": {"omzet": 0.0, "operationele_kasstroom": 0.0}
}
```
De generator bevat zelf de volledige Type→categorie-map en de kasvuistregel; lever dus
uitsluitend de ruwe matrix aan, niet voorberekende totalen. Alle accounting gebeurt in het
script en in de Excel-formules, zodat het deterministisch en controleerbaar is.
