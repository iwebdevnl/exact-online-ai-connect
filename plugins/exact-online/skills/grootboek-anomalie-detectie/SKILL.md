---
name: grootboek-anomalie-detectie
description: >
  Detecteer ongebruikelijke boekingen en potentiële fouten in het grootboek van Exact Online.
  Controleert op: afwijkende bedragen, ongebruikelijke rekening-combinaties, tussenrekeningen
  die niet op nul staan, dubbele boekingen, en boekingen buiten kantooruren.

  Triggers: 'anomalie detectie', 'grootboek controle', 'boekingen controleren', 'fouten opsporen',
  'ongebruikelijke boekingen', 'dubbele boekingen', 'boekingsfouten', 'grootboek analyse',
  'continuous monitoring', 'audit controle', 'jaarwerk voorbereiding', 'jaarwerk controle',
  'boekhouding controleren', 'administratie check', 'administratie opschonen',
  'tussenrekeningen controleren', 'kruisposten controleren', 'balans kloppend',
  'verdachte boekingen', 'correctieboekingen nodig'.

  Gebruik deze skill wanneer de gebruiker de kwaliteit van de boekhouding wil controleren,
  bij jaarwerk-voorbereiding, of wanneer ze vermoeden dat er fouten in de administratie zitten.
  Werkt met Exact Online MCP (TransactionLines via analyze_data, GLAccounts, ReportingBalance).
---

# Grootboek Anomalie-detectie

Automatische controle op veelvoorkomende boekingsfouten en onregelmatigheden. Bespaart de
boekhouder uren handmatig speurwerk en verhoogt de betrouwbaarheid van de administratie. Deze
skill werkt in **elke** administratie omdat hij classificeert op **rekeningtype**
(`GLAccount.Type`) en op omschrijving-trefwoorden, niet op administratie-specifieke
grootboeknummers of codebereiken.

## Waarom dit generiek is (lees dit eerst)

Grootboeknummers verschillen per administratie: 1050 is niet altijd een kruispost, 4100 niet
altijd kantoorkosten. Wat wél in elke Exact Online-administratie identiek is, is het veld `Type`
op de grootboekrekening — een vaste, platform-brede classificatie (bijv. 12 = Bank, 24 = VAT,
90 = General). **Classificeer op `Type` en op omschrijving-trefwoorden, nooit op vast nummer of
codebereik.** Zo blijft de detectie correct, ongeacht het rekeningschema van de klant.

> Vuistregel: haal de relevante rekeningen altijd eerst live op met GLAccounts (op `Type`),
> lees hun `Code` uit de response, en gebruik díe codes in de saldo-query's. Nooit een
> aangenomen bereik als `1050-1099` in een filter zetten.

## Waarom anomalie-detectie

Zelfs in goed bijgehouden administraties sluipen fouten erin: een factuur op de verkeerde
grootboekrekening, een dubbele boeking door een importfout, of een tussenrekening die vergeten
is. Handmatig controleren is tijdrovend en foutgevoelig. Geautomatiseerde detectie pakt de
meest voorkomende problemen op.

> In alle voorbeelden hieronder staat `<jaar>` voor het boekjaar dat je controleert. Bepaal dat
> aan het begin (huidig boekjaar of wat de gebruiker vraagt) en vul het overal in — gebruik geen
> vast jaartal.

## Controle-categorieën

### 1. Tussenrekeningen die niet op nul staan

De belangrijkste controle. Tussenrekeningen (kruisposten, vraagposten, tussenrekening
uitgestelde kosten/omzet, PSP-tussenrekeningen) horen op nul te staan aan het einde van een
periode.

**Stap 1a — identificeer de kandidaten via Type:**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": { "Type": [32, 90] },
  "select": "Code,Description,Type,TypeDescription"
}
```

**BELANGRIJK**: Tussenrekeningen worden NIET gevonden via Type [20, 22, 24]:
- Type 20 = Accounts Receivable (Debiteuren)
- Type 22 = Accounts Payable (Crediteuren)
- Type 24 = VAT (BTW-rekeningen)

**Werkelijke tussenrekeningen hebben doorgaans deze Types:**
- **Type 90** (General): uitzoekrekening, tussenrekening lonen, uitgestelde kosten/omzet,
  PSP-tussenrekening, incasso te ontvangen — de meeste tussenrekeningen.
- **Type 32** (Other assets): sommige kruisposten.

**Stap 1b — filter binnen die set op omschrijving-trefwoorden.** Type 90 (General) bevat óók
gewone grootboekrekeningen. Selecteer daarom alleen de rekeningen waarvan de `Description`
(case-insensitive) een van deze trefwoorden bevat:

`tussenrekening`, `tussenrek`, `kruispost`, `uitzoek`, `vraagpost`, `te ontvangen`,
`te betalen`, `nog te`, `psp`, `mollie`, `stripe`, `paypal`, `adyen`, `buckaroo`, `betaalprovider`,
`incasso`, `uitgestelde`, `overloop`, `transitoria`, `clearing`, `suspense`, `interim`,
`wip`, `onderhanden`.

Zo vind je tussenrekeningen ongeacht hun nummer en ongeacht welke PSP de klant gebruikt.
Twijfel je of een Type 90-rekening een tussenrekening is? Toon hem aan de gebruiker en laat het
bevestigen — de trefwoordlijst is een zeef, geen absolute waarheid.

**Stap 2 — controleer het saldo per gevonden rekening.** Gebruik de `Code`-waarden uit stap 1,
niet een aangenomen bereik. Via ReportingBalance (aanbevolen, altijd actueel):

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "GLAccountCode": "<code uit stap 1>",
    "ReportingYear": "<jaar>"
  },
  "select": "GLAccountCode,GLAccountDescription,ReportingPeriod,Amount,Type"
}
```

**CRUCIAAL — ReportingBalance bevat mutaties, NIET saldi:**
Om het werkelijke saldo te bepalen: openingsbalans + SUM(alle mutaties alle periodes t/m huidig).
Voor een snelle check: als de SUM van alle mutaties over ALLE periodes ≠ 0 voor een
kruispostenrekening, dan zijn er openstaande posten. Neem de openingsbalans mee (de
openingsbalans-periode / het beginsaldo), anders mis je een saldo dat uit een vorig jaar
is meegenomen.

**Uitzondering**: rekeningen voor uitgestelde kosten/omzet mogen bewust een saldo hebben bij
doorlopende contracten. Herken die aan trefwoorden als `uitgesteld`, `overloop` of `transitoria`
in de omschrijving (niet aan een vast nummer) en markeer ze als "controle" in plaats van "fout".

**Via analyze_data (als alternatief, check eerst sync-status)** — geef ook hier de dynamisch
gevonden codes mee in plaats van een bereik:

```json
{
  "table": "Financial/TransactionLines",
  "select": ["GLAccountCode", "GLAccountDescription"],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Saldo" }
  ],
  "groupBy": ["GLAccountCode", "GLAccountDescription"],
  "filters": [
    { "column": "GLAccountCode", "operator": "IN", "values": ["<code1>", "<code2>", "..."] }
  ]
}
```

**Let op**: Check altijd `list_available_tables` of analyze_data gesynchroniseerd is.
Bij status `syncing` geeft analyze_data incomplete data — val terug op ReportingBalance.

### 2. Afwijkende bedragen (outlier detectie)

Zoek boekingen die significant afwijken van het gebruikelijke op dezelfde grootboekrekening.
Een factuur van € 50.000 op een rekening waar normaal € 500-facturen staan, is verdacht.

```json
{
  "table": "Financial/TransactionLines",
  "select": ["GLAccountCode", "GLAccountDescription"],
  "aggregations": [
    { "function": "AVG", "column": "AmountDC", "alias": "Gemiddeld" },
    { "function": "MAX", "column": "AmountDC", "alias": "Maximum" },
    { "function": "MIN", "column": "AmountDC", "alias": "Minimum" },
    { "function": "STDDEV", "column": "AmountDC", "alias": "Stdev" },
    { "function": "COUNT", "column": "ID", "alias": "Aantal" }
  ],
  "groupBy": ["GLAccountCode", "GLAccountDescription"],
  "filters": [
    { "column": "FinancialYear", "operator": "=", "value": "<jaar>" }
  ]
}
```

**Kies de drempel robuust.** Een naïeve regel als "MAX > 10× gemiddelde" faalt op rekeningen
waar debet en credit elkaar bijna opheffen: daar ligt het gemiddelde dicht bij nul en wordt
bijna élke boeking onterecht als outlier gemarkeerd. Gebruik daarom:
- Primair een **absolute afwijking**: een regel valt op als hij meer dan ~3× de
  standaarddeviatie van het gemiddelde ligt (`ABS(bedrag − gemiddeld) > 3 × stdev`), of
- Een **absolute-bedragdrempel** die past bij de rekening (bijv. regels > € 10.000 op een
  rekening waar de mediaan onder € 1.000 ligt).
- Sla rekeningen met een gemiddelde ≈ 0 én lage stdev over voor de ratio-check; beoordeel die
  op absolute bedragen.

Filter rekeningen met slechts 1-2 boekingen uit (te weinig data voor een betrouwbare spreiding).
Haal daarna de specifieke boekingen op voor nader onderzoek.

### 3. Potentiële dubbele boekingen

Laat de database het zware werk doen in plaats van 5000 ruwe regels op te halen en met het oog
te vergelijken — bij grotere administraties valt data buiten zo'n limiet weg (stille misser).
Aggregeer server-side op relatie + bedrag + datum en houd alleen de combinaties over die vaker
dan één keer voorkomen:

```json
{
  "table": "Financial/TransactionLines",
  "select": ["AccountName", "AmountDC", "Date"],
  "aggregations": [
    { "function": "COUNT", "column": "ID", "alias": "Aantal" }
  ],
  "groupBy": ["AccountName", "AmountDC", "Date"],
  "having": [
    { "column": "Aantal", "operator": ">", "value": 1 }
  ],
  "filters": [
    { "column": "FinancialYear", "operator": "=", "value": "<jaar>" }
  ]
}
```

Voor elke kandidaat-combinatie haal je daarna de onderliggende regels op (EntryNumber,
Description, factuurnummer) om te bevestigen of het echt een dubbel is. Houd rekening met
terugkerende facturen: maandelijks dezelfde leverancier en bedrag is normaal — check of de
beschrijvingen/factuurnummers verschillen en of de datums minstens ~een maand uit elkaar liggen.
Wil je ook near-duplicates (binnen enkele dagen) vangen, groepeer dan op een afgekapte datum
(bijv. per week) of vergelijk de kandidaten met dezelfde AccountName+AmountDC op datum-afstand.

### 4. Boekingen op ongebruikelijke rekeningen

Zoek rekeningen die maar 1-2 boekingen per jaar hebben — dit kan duiden op een verkeerde
rekeningkeuze:

```json
{
  "table": "Financial/TransactionLines",
  "select": ["GLAccountCode", "GLAccountDescription"],
  "aggregations": [
    { "function": "COUNT", "column": "ID", "alias": "Aantal_Boekingen" },
    { "function": "SUM", "column": "AmountDC", "alias": "Totaal" }
  ],
  "groupBy": ["GLAccountCode", "GLAccountDescription"],
  "filters": [
    { "column": "FinancialYear", "operator": "=", "value": "<jaar>" }
  ],
  "orderBy": [{ "column": "Aantal_Boekingen", "direction": "ASC" }]
}
```

Rekeningen met slechts 1-2 boekingen per jaar zijn verdacht (tenzij het bewust eenmalige
posten zijn zoals afschrijvingen, jaarwerk-boekingen, of BTW-correcties).

### 5. Balans-plausibiliteit

Controleer of de balans "logisch" is. De signalen hieronder betreffen verschillende
rekeningtypes, dus bepaal de betrokken rekeningen via hun `Type` en beoordeel dan het saldo:
- **Bank negatief zonder kredietfaciliteit** → rekeningen met `Type` 12 (Bank), ook 10/14/16.
- **Debiteurensaldo negatief** (vooruitbetalingen zijn ok, maar signaleer) → `Type` 20.
- **Crediteurensaldo positief** (leverancier is jou geld schuldig) → `Type` 22.

Haal per relevant type de rekeningen op via GLAccounts en controleer het cumulatieve saldo via
ReportingBalance (openingsbalans + mutaties), net als bij de tussenrekeningen. Gebruik geen
enkel `Type 40`-filter voor deze controles: Type 40 in ReportingBalance duidt op
bank-/kasboekingen en dekt debiteuren/crediteuren niet.

## Rapportage

Neem rekeningcodes en -omschrijvingen over zoals ze in de administratie heten. Onderstaand
format is een **voorbeeld met fictieve nummers** — vul het met de werkelijke data:

```
Grootboek Anomalie-rapport — <periode> <jaar>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stand per [datum]

Gecontroleerd: X boekingsregels over Y periodes

❌ Tussenrekeningen met saldo:
   • <code> Kruisposten:                   € 498,40  (2 openstaande regels)
   • <code> Kruisposten liquide middelen:  €  31,29  (1 regel)

⚠️ Tussenrekeningen met bewust saldo:
   • <code> Uitgestelde kosten:            € 7.073,51  (doorlopende contracten)
   • <code> Uitgestelde omzet:             € 5.483,20  (doorlopende contracten)
   ✅ <code> Incasso te ontvangen:         € 0,00

⚠️ Afwijkende bedragen:
   • <rek> Kantoorkosten:  € 8.500 op 15-jan (mediaan € 340, > 3× stdev)
     → Boeking #<nr>, omschrijving: "Nieuwe laptops"
     → Mogelijk: Investering i.p.v. kosten? Controleer activering.

⚠️ Potentiële dubbelen:
   • Leverancier X, € 1.250,00 op 10-jan en 12-jan
     → Factuurnr verschilt (F-001 vs F-002) → waarschijnlijk geen dubbel

✅ Balans-plausibiliteit: geen onlogische saldi gevonden
✅ Ongebruikelijke rekeningen: 3 rekeningen met 1 boeking
   → <rek> Kosten Servers/Hosting: € 455,48 — controleer classificatie

Samenvatting:
  Kritiek:      2 items (tussenrekeningen met onverklaard saldo)
  Waarschuwing: 2 items (outlier + mogelijke dubbelen)
  Info:         1 item (ongebruikelijke rekening)
```

## Bekende API-eigenaardigheden

| Verwachting | Correct | Toelichting |
|----------------|---------|-------------|
| GLAccounts Type [20, 22, 24] voor tussenrekeningen | Type [32, 90] + filter op omschrijving-trefwoorden | Type 20=Debiteuren, 22=Crediteuren, 24=VAT — dit zijn GEEN tussenrekeningen |
| Tussenrekeningen via vast codebereik (1050-1099 e.d.) | GLAccounts op Type + trefwoord → codes uit de response in de saldo-query | Codebereiken verschillen per administratie |
| Hardcoded GLAccountCodes | Altijd eerst GLAccounts opzoeken | Rekeningnummers variëren per administratie |
| Outlier = MAX > 10× gemiddelde | Absolute afwijking (>3× stdev) of absolute-bedragdrempel | Bij gemiddelde ≈ 0 markeert de ratio-regel bijna alles onterecht |
| Dubbelen: 5000 regels ophalen en met het oog vergelijken | Server-side GROUP BY relatie+bedrag+datum, HAVING COUNT > 1 | Voorkomt stille missers voorbij de rijlimiet |
| analyze_data SUM = saldo | analyze_data SUM = netto mutaties | Voor actueel saldo: ReportingBalance (openingsbalans + mutaties) |
| analyze_data altijd actueel | analyze_data kan status 'syncing' hebben | Check list_available_tables; val terug op ReportingBalance |
| service: "Financial", entity: "TransactionLines" | service: "Financialtransaction", entity: "TransactionLines" | TransactionLines zit in Financialtransaction-service (bij REST API) |

## Communicatie

- Begin altijd met de meest kritieke bevindingen (tussenrekeningen)
- Geef bij elke anomalie een mogelijke verklaring EN een aanbevolen actie
- Wees voorzichtig met het label "fout" — het kan ook een bewuste boeking zijn
- Vermeld dat uitgestelde kosten/omzet bewust een saldo kunnen hebben; herken die aan de
  omschrijving, niet aan een vast nummer
- Bij jaarwerk: bied aan om bevindingen als checklist te exporteren
- Adviseer contact met accountant bij complexe correcties
- Gebruik € Nederlands formaat (€ 1.234,56)
- Vermeld altijd "Stand per [datum]" — cijfers zijn momentopnames
