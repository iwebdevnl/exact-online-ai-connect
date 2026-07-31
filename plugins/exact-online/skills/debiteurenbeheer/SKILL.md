---
name: debiteurenbeheer
description: >
  Deze skill moet gebruikt worden wanneer de gebruiker openstaande debiteuren wil opvolgen,
  betaalgedrag wil analyseren of herinneringen wil voorbereiden in Exact Online:
  ouderdomsanalyse, escalatie bij langdurig openstaande posten en concept-herinneringsmails.

  Triggers: 'debiteurenbeheer', 'openstaande facturen opvolgen', 'betalingsherinneringen',
  'wie moet nog betalen', 'achterstallige facturen', 'ouderdomsanalyse', 'wanbetalers',
  'betaalgedrag', 'DSO', 'welke klanten betalen te laat'.
---

# Debiteurenbeheer

Proactief debiteurenbeheer: analyseer betaalgedrag, identificeer risico's, en bereid
herinneringsacties voor. Het doel is niet alleen zien wie te laat betaalt, maar ook
waarom en wat je eraan kunt doen.

## Waarom proactief debiteurenbeheer

Veel ondernemers kijken pas naar debiteuren als het geld opraakt. Door structureel te
monitoren verbeter je de cashflow, verlaag je het afschrijvingsrisico, en versterk je de
klantrelatie (een vriendelijke herinnering is beter dan een boze brief na 90 dagen).

## Analyse Workflow

### Stap 1: Totaaloverzicht ophalen

Start met een quick-scan via OutstandingInvoicesOverview voor de totalen:

**Tool: `read_operation`** (deze tool doet alleen GET en heeft geen `operation`-parameter)

```json
{
  "service": "Read",
  "entity": "OutstandingInvoicesOverview"
}
```

Dit geeft in één call:
- **OutstandingReceivableInvoiceCount/Amount**: Totaal aantal en bedrag openstaande facturen
- **OverdueReceivableInvoiceCount/Amount**: Waarvan verlopen (voorbij vervaldatum)
- **OutstandingPayableInvoiceCount/Amount**: Openstaande crediteuren (ter referentie)
- **OverduePayableInvoiceCount/Amount**: Waarvan verlopen crediteuren

### Stap 2: Ouderdomsanalyse per klant

Haal de ouderdomsanalyse op per klant:

**Tool: `read_operation`**

```json
{
  "service": "Read",
  "entity": "AgingReceivablesList"
}
```

**BELANGRIJK**: Gebruik `AgingReceivablesList` (NIET `AgingReceivablesListByAgeGroup`, dat
endpoint retourneert altijd "Bad Request - Error in query syntax").

Dit geeft per klant een uitsplitsing naar:
- **AgeGroup1** (≤ 30 dagen): AgeGroup1Amount
- **AgeGroup2** (31-60 dagen): AgeGroup2Amount
- **AgeGroup3** (61-90 dagen): AgeGroup3Amount
- **AgeGroup4** (> 90 dagen): AgeGroup4Amount
- **TotalAmount**: Totaal openstaand per klant
- **AccountCode/AccountName**: Klantidentificatie
- **CurrencyCode**: Valuta (typisch EUR)

**Let op, dit endpoint heeft 0 filterbare velden**: sorteren en filteren doe je zelf op de
output. Belangrijker nog: `read_operation` levert **maximaal 60 records per call**. Heeft de
administratie meer dan 60 klanten met openstaande posten, dan is dit overzicht stil onvolledig,
en juist deze skill belooft compleetheid. Tel dus de rijen. Krijg je er precies 60, ga er dan
van uit dat er meer zijn en gebruik deze twee routes naast elkaar:

- **Totalen**: `AgingOverview` (hieronder) geeft de bedragen per ouderdomsgroep in 5 rijen en is
  dus altijd compleet, alleen zonder klantdetail. Gebruik die cijfers voor het totaalbeeld.
- **Detail per factuur**: `Bulk/Cashflow/Receivables` (stap 3) met paginatie doorlopen tot je
  alle regels hebt, en zelf per klant optellen.

Zeg tegen de gebruiker dat het klantoverzicht is afgekapt zolang je de paginatie niet hebt
afgemaakt. Een top-5 uit de eerste 60 rijen is geen top-5.

Voor een compact totaaloverzicht per ouderdomsgroep (zonder klantdetail), gebruik:

**Tool: `read_operation`**

```json
{
  "service": "Read",
  "entity": "AgingOverview"
}
```

Dit geeft 5 rijen: Totaal + 4 ouderdomsgroepen, met zowel AmountReceivable als AmountPayable.

### Stap 3: Detail openstaande posten

Voor klanten met significante openstaande bedragen, haal de individuele facturen op:

**Via Cashflow/Receivables (aanbevolen)**:

**Tool: `read_operation`**

```json
{
  "service": "Bulk",
  "entity": "Cashflow/Receivables",
  "filters": { "Status": [20, 30] },
  "select": "AccountName,AccountCode,AmountDC,DueDate,InvoiceNumber,InvoiceDate,Description,CurrencyCode"
}
```

**Paginatie op Bulk-endpoints**: Bulk-endpoints accepteren geen `skip`. Ze pagineren met een
cursor: neem `next_page.page_token` uit de response over als `page_token` in de volgende call en
herhaal tot er geen `next_page` meer terugkomt. Dit is de enige manier om boven de 60 records
per call uit te komen, en dus de enige manier om een compleet debiteurenoverzicht te krijgen.

**Let op AmountDC teken**: Openstaande debiteuren hebben een **negatief** AmountDC in
Cashflow/Receivables. Gebruik ABS(AmountDC) voor het te ontvangen bedrag.

**DueDate formaat**: De API retourneert datums in OData formaat: `/Date(1772150400000)/`.
Dit zijn milliseconden sinds Unix epoch (1 jan 1970). Converteer naar datum voor de analyse.

**Via ReceivablesList (alternatief)**:

**Tool: `read_operation`**

```json
{
  "service": "Read",
  "entity": "ReceivablesList"
}
```

Dit geeft individuele openstaande regels met Amount (negatief), DueDate, JournalCode,
JournalDescription, InvoiceNumber, InvoiceDate, AccountName, Description, AmountInTransit.
Handig voor het identificeren van welk dagboek/bankrekening de post bevat. Ook hier geldt de
limiet van 60 records per call.

**Let op**: ReceivablesList Amount is ook **negatief** voor openstaande debiteuren.

### Stap 4: Omzetvolume per klant (via analyze_data)

Deze query meet **hoeveel er per klant is gefactureerd**, niet hoe snel er is betaald. Type 20
zijn verkoopboekingen, en daar zit geen betaaldatum in. Je kunt er dus geen betaaltermijn of
betaalgedrag per klant uit afleiden. Gebruik hem als volume-context bij de ouderdomsanalyse:
€ 300 openstaand bij een klant met € 400.000 omzet is een ander gesprek dan € 300 bij een klant
met € 900 omzet.

Check eerst of analyze_data gesynchroniseerd is via `list_available_tables`.

**Tool: `analyze_data`** (bestaat alleen op Trial en Analytics, niet op Essentials). De
`division`-parameter is optioneel: laat je hem weg, dan gebruikt de tool de geselecteerde
administratie. Vul `<jaar>` met het boekjaar dat je analyseert, gebruik geen vast jaartal.

```json
{
  "division": "<administratiecode>",
  "query": {
    "table": "Financial/TransactionLines",
    "select": ["AccountName"],
    "aggregations": [
      { "function": "SUM", "column": "AmountDC", "alias": "Totaal_Gefactureerd" },
      { "function": "COUNT", "column": "ID", "alias": "Aantal_Facturen" }
    ],
    "groupBy": ["AccountName"],
    "filters": [
      { "column": "Type", "operator": "=", "value": 20 },
      { "column": "FinancialYear", "operator": "=", "value": "<jaar>" }
    ]
  }
}
```

**Betaalgedrag per klant is met deze bronnen niet te berekenen.** Daarvoor heb je per factuur
zowel de factuurdatum als de betaaldatum nodig, en die combinatie zit niet in
AgingReceivablesList, Cashflow/Receivables of TransactionLines Type 20. Benader het in plaats
daarvan met de ouderdomsanalyse uit stap 2: het bedrag per klant in AgeGroup3 en AgeGroup4 is de
bruikbaarste indicator van structureel laat betalen die zonder betaaldatums beschikbaar is.
Beloof de gebruiker geen betaaltermijn in dagen per klant.

**Let op**: Bij status `syncing` op analyze_data, val terug op de REST API endpoints
(AgingReceivablesList + Cashflow/Receivables) die altijd actueel zijn. Op Essentials bestaan
`analyze_data` en `list_available_tables` niet: zeg dat dan expliciet en gebruik
ReportingBalance voor de omzetcijfers.

### Stap 5: DSO berekenen (overall, niet per klant)

Days Sales Outstanding (DSO) = (Openstaande debiteuren / Omzet over periode) × Aantal dagen in
de periode. Bereken dit **overall**. Een DSO per klant is met deze bronnen niet te maken: het
openstaande bedrag per klant is een momentopname zonder toerekening aan een periode, dus de
teller en de noemer zouden over verschillende dingen gaan en het getal zou betekenisloos zijn.

Gebruik de OutstandingInvoicesOverview (Stap 1) voor het totale openstaande bedrag, en
analyze_data of ReportingBalance voor de omzet over de periode. Een stijgende DSO is een
waarschuwingssignaal, dus bereken hem over meerdere opeenvolgende periodes en vergelijk.

## Escalatie-workflow

Hanteer een gelaagde benadering afhankelijk van de ouderdom:

| Ouderdom | Actie | Toon |
|----------|-------|------|
| 1-14 dagen over vervaldatum | Vriendelijke herinnering | "Ter herinnering..." |
| 15-30 dagen | Tweede herinnering | "Wij verzoeken u vriendelijk..." |
| 31-60 dagen | Formele aanmaning | "Ondanks eerdere herinneringen..." |
| 61-90 dagen | Laatste waarschuwing | "Bij uitblijven van betaling..." |
| >90 dagen | Escalatie naar incasso | Adviseer externe incasso of juridische stappen |

### Concept-herinneringsmail genereren

Bij het genereren van herinneringsmails, neem altijd op:
- Factuurnummer(s) en bedrag(en)
- Originele factuurdatum en vervaldatum
- Totaal openstaand bedrag
- Bankgegevens voor betaling
- Contactgegevens voor vragen

Pas de toon aan op basis van de ouderdomscategorie en de klantrelatie.

## Rapportage

### Debiteurenrapport genereren

```
Debiteurenrapport, 16 februari 2026
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stand per [datum]

Totaal openstaand:                    € 43.213,48
  ≤ 30 dagen:     € 43.659,15  (101%)  ← incl. nog niet vervallen
  31-60 dagen:    €    240,15   (1%)
  61-90 dagen:    €    145,67   (0%)
  > 90 dagen:     €   -831,49  (-2%)   ← creditnota's/tegoeden

Verlopen facturen:  43 stuks  (€ 25.082,09)
DSO (overall):      XX dagen

Top 5 openstaand:
1. Klant A BV         € 2.500,00   (47 dagen over)  → 2e herinnering
2. Klant B NV         € 1.800,00   (12 dagen over)  → 1e herinnering
3. Klant C            €   950,00   (95 dagen over)  → escalatie
4. Klant D BV         €   750,00   (niet vervallen)
5. Klant E            €   600,00   (3 dagen over)   → afwachten

Aanbevelingen:
1. Stuur 2e herinnering naar Klant A BV (€ 2.500, 47 dagen over)
2. Escaleer Klant C naar incassobureau (€ 950, >90 dagen)
3. Onderzoek negatief saldo >90 dagen (€ -831), mogelijke creditnota's
```

Vermeld onder een top-N altijd hoeveel klanten je hebt beoordeeld. Is de lijst gebaseerd op een
afgekapt resultaat van 60 rijen, zeg dat er dan bij.

## API-gedrag om rekening mee te houden

| Punt | Toelichting |
|------|-------------|
| Gebruik `AgingReceivablesList`, niet `AgingReceivablesListByAgeGroup` | De ByAgeGroup-variant geeft altijd "Bad Request - Error in query syntax" |
| `Cashflow/Receivables` AmountDC is negatief | Gebruik ABS() voor het openstaande bedrag |
| `ReceivablesList` Amount is negatief | Zelfde als Cashflow/Receivables |
| DueDate komt in OData-formaat `/Date(ms)/` | Milliseconden sinds epoch, converteer zelf naar een datum |
| `read_operation` levert maximaal 60 records | Geldt ook voor AgingReceivablesList en ReceivablesList. Precies 60 rijen betekent: waarschijnlijk afgekapt |
| Bulk-endpoints negeren `skip` | Pagineer met `page_token` uit `next_page.page_token` |
| AgingReceivablesList heeft 0 filterbare velden | Sorteren en filteren doe je zelf op de output |
| analyze_data kan status `syncing` hebben | Check `list_available_tables`, val terug op de REST-endpoints die altijd actueel zijn |
| analyze_data en list_available_tables bestaan niet op Essentials | Alleen op Trial en Analytics. Op Essentials vervalt stap 4 en komt de omzet uit ReportingBalance |
| TransactionLines Type 20 bevat geen betaaldatum | Betaalgedrag per klant is er niet uit af te leiden, alleen gefactureerd volume |

## Communicatie

- Gebruik altijd het perspectief van de ondernemer ("uw klanten", niet "accounts")
- Bied aan om concept-herinneringsmails te schrijven
- Bij grote bedragen of lange termijnen: adviseer contact met accountant of jurist
- Bied aan om de bevindingen samen te vatten als takenlijst voor opvolging
- Vermeld altijd dat bedragen "stand per [datum]" zijn, ze kunnen dagelijks wijzigen
- Zeg het expliciet wanneer een overzicht mogelijk is afgekapt op 60 records
- Gebruik € Nederlands formaat (€ 1.234,56)
