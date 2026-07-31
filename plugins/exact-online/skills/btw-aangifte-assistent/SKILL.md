---
name: btw-aangifte-assistent
description: >
  Deze skill moet gebruikt worden wanneer de gebruiker een BTW-aangifte in Exact Online wil
  controleren of voorbereiden: BTW-bedragen uit de boekingen vergelijken met de aangifte,
  afwijkingen opsporen en ICP-prestaties nalopen. Ook bij een losse vraag als "check mijn BTW"
  of "klopt mijn aangifte".

  Triggers: 'btw aangifte', 'btw controle', 'btw aansluiting', 'omzetbelasting',
  'kwartaalaangifte', 'btw verschil', 'suppletie', 'ICP opgave', 'voorbelasting', 'vat return'.
---

# BTW-aangifte Assistent

Controleer en bereid BTW-aangiftes voor in Exact Online. Deze skill helpt bij het opsporen van
afwijkingen tussen boekingen en aangifte, zodat je met vertrouwen kunt indienen.

## Waarom deze skill bestaat

De BTW-aangifte is een van de meest voorkomende foutbronnen in de boekhouding. Typische problemen:
boekingen op verkeerde BTW-codes, vergeten facturen, ICP-leveringen niet correct gerapporteerd,
afrondingsverschillen die oplopen, en openstaande boekingsvoorstellen die de aangifte beïnvloeden.
Door systematisch te controleren voorkom je naheffingen en boetes.

## Stap-voor-stap Workflow

### Stap 1: Bepaal de BTW-periode

Vraag de gebruiker welke periode ze willen controleren als dit niet duidelijk is.
Nederlandse BTW-aangifteperiodes:

- **Maandelijks**: Periode 1-12 (januari = 1, december = 12)
- **Per kwartaal**: Q1 = periode 1-3, Q2 = 4-6, Q3 = 7-9, Q4 = 10-12
- **Jaarlijks**: Periode 1-12

Bepaal ook het boekjaar (FinancialYear). Standaard is dit het huidige jaar, tenzij anders aangegeven.
In alle voorbeelden hieronder staat `<jaar>` voor dat boekjaar en staan `<beginperiode>` en
`<eindperiode>` voor de periodegrenzen. Vul die overal in, gebruik geen vast jaartal.

### Stap 2: Haal de BTW-aangifte op

Gebruik het Returns endpoint om de ingediende of concept-aangifte op te halen.

**Tool: `read_operation`** (deze tool doet alleen GET en heeft geen `operation`-parameter)

```json
{
  "service": "Read",
  "entity": "Returns",
  "select": "DocumentID,Period,Year,Status,Amount,Description,Frequency,DueDate,Type"
}
```

**Let op**: Het Returns endpoint heeft beperkte velden. Beschikbare velden zijn:
DocumentID, Amount, Created, Currency, Description, DocumentViewUrl, DueDate, Frequency,
Period, PeriodDescription, Status, Subject, Type, Year.

**NIET beschikbaar**: TotalRevenue, TotalVATDue, TotalVATDeductible, ReportingFrequency.

**Status-waarden**: de volledige reeks die op Returns voorkomt is -10, 0, 5, 10, 20, 30, 40, 50.
Vastgelegde betekenis: 10 = Concept, 20 = Open, 30 = Approved, 40 = Realized,
50 = Verwerkt (Processed). Van -10, 0 en 5 is de betekenis niet betrouwbaar vastgelegd: toon
de ruwe waarde en trek er geen conclusie uit.

**Type-waarden**: 31 = BTW-aangifte, 32 = ICP-opgave, 146 = Loonheffing
**Frequency**: M = Maandelijks, Q = Per kwartaal, Y = Jaarlijks

Filter op Year en Period die overeenkomen met de gewenste aangifteperiode, en Type = 31 voor BTW.

### Stap 3: Haal de BTW-grootboekrekeningen op

BTW-rekeningen hebben `GLAccount.Type = 24` (VAT). Filter daarop, niet op rekeningnummer
(rekeningnummers verschillen per administratie):

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "filters": { "Type": 24 },
  "select": "Code,Description,BalanceType"
}
```

Zo krijg je alle BTW-rekeningen, ongeacht het gebruikte rekeningschema. Typische rubrieken
(de rekeningnummers verschillen per administratie):
- Af te dragen BTW hoog/laag tarief (rubriek 1a/1b)
- Af te dragen BTW ICP-verwervingen (rubriek 4a)
- Af te dragen verlegde BTW (rubriek 4b)
- Te vorderen (voor)belasting (rubriek 5b)

**Bewaar de gevonden `Code`-waarden en koppel ze aan een rubriek.** Leid de rubriek af uit de
`Description`, laat de indeling bij twijfel door de gebruiker bevestigen, en gebruik in alle
volgende stappen deze codes. Nooit een aangenomen rekeningnummer of nummerbereik.

**Fallback**: levert `Type: 24` niets op in een niet-standaard administratie, val dan terug op
een rekeningnummer-bereik (vaak 1500-1599), en controleer de gevonden rekeningen handmatig
voordat je ze in de berekening gebruikt.

### Stap 4: Haal BTW-boekingen op uit de administratie

Dit is de kern van de controle. Gebruik ReportingBalance als primaire bron (altijd beschikbaar).

**Via ReportingBalance (aanbevolen)**:

**Tool: `read_operation`**

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "filters": {
    "GLAccountCode": ["<code1 uit stap 3>", "<code2 uit stap 3>", "<...>"],
    "ReportingYear": "<jaar>",
    "ReportingPeriod": { "$gte": "<beginperiode>", "$lte": "<eindperiode>" }
  },
  "select": "GLAccountCode,GLAccountDescription,ReportingPeriod,Amount,Type"
}
```

Een array in een filter wordt een OR-conditie, dus zo haal je precies de rekeningen uit stap 3 op.

**Let op de limiet van 60 records.** `read_operation` geeft maximaal 60 rijen per call, en
ReportingBalance levert een rij per combinatie van rekening, periode en Type. Bij een kwartaal
met meerdere BTW-rekeningen is 60 snel bereikt, en een afgekapt resultaat geeft een BTW-positie
die klopt op het eerste gezicht en toch te laag is. Tel de rijen: krijg je er precies 60, splits
de call dan op per rekening of per periode en tel de deelresultaten op. Controleer daarna of je
voor elke rekening uit stap 3 en elke periode in de aangifteperiode data hebt.

**CRUCIAAL, het Type-veld op ReportingBalance**:

ReportingBalance groepeert mutaties per dagboektype. De Type-codes zijn:
- **Type 20**: Verkoopboekingen (reguliere BTW op verkoop)
- **Type 21**: Creditnota's verkoop
- **Type 30**: Inkoopboekingen (reguliere BTW op inkoop + ICP)
- **Type 40**: Bank-/kasboekingen
- **Type 50**: Memoriaalboekingen (bevat BTW-afdrachtboekingen van VORIGE periodes)

**Filter Type 50 uit** bij het berekenen van de BTW-positie van de huidige periode.
Type 50 bevat de tegenboeking van de vorige BTW-aangifte en vertekent het beeld.

**Via analyze_data (als beschikbaar en gesynchroniseerd)**:

**Tool: `analyze_data`** (bestaat alleen op Trial en Analytics, niet op Essentials). De
`division`-parameter is optioneel: laat je hem weg, dan gebruikt de tool de geselecteerde
administratie.

```json
{
  "division": "<administratiecode>",
  "query": {
    "table": "Financial/TransactionLines",
    "select": ["GLAccountCode", "GLAccountDescription", "Type"],
    "aggregations": [
      { "function": "SUM", "column": "AmountDC", "alias": "Bedrag" },
      { "function": "SUM", "column": "VATAmountDC", "alias": "BTW_Bedrag" },
      { "function": "COUNT", "column": "ID", "alias": "Aantal_Boekingen" }
    ],
    "groupBy": ["GLAccountCode", "GLAccountDescription", "Type"],
    "filters": [
      { "column": "FinancialYear", "operator": "=", "value": "<jaar>" },
      { "column": "FinancialPeriod", "operator": ">=", "value": "<beginperiode>" },
      { "column": "FinancialPeriod", "operator": "<=", "value": "<eindperiode>" },
      { "column": "GLAccountCode", "operator": "IN", "values": ["<code1 uit stap 3>", "<code2 uit stap 3>", "<...>"] }
    ]
  }
}
```

**Let op analyze_data veldnamen**:
- Gebruik `VATAmountDC` voor BTW-bedrag in divisievaluta (DC = Division Currency)
- `AmountVATFC` bestaat ook maar is in vreemde valuta (FC = Foreign Currency)
- Check altijd of de analyze_data tabel gesynchroniseerd is via `list_available_tables`
- Bij status `syncing`: val terug op ReportingBalance via REST API
- Op Essentials bestaan `analyze_data` en `list_available_tables` niet. Zeg dat dan expliciet
  en doe de hele controle via ReportingBalance

### Stap 5: Haal BTW-codes op

Haal altijd de werkelijke BTW-codes op uit de administratie:

**Tool: `read_operation`**

```json
{
  "service": "Vat",
  "entity": "VATCodes",
  "select": "Code,Description,Percentage,EUSalesListing"
}
```

**Let op**:
- Service is `Vat` (NIET `Financial`)
- Het veld `TransactionType` bestaat NIET op VATCodes
- Gebruik `EUSalesListing` om ICP-gerelateerde codes te identificeren

Dit is nodig om ICP-leveringen, verlegd BTW, en andere speciale situaties te identificeren.

### Stap 6: Vergelijk en analyseer

Bereken de BTW-positie door de ReportingBalance-data te groeperen (excl. Type 50). Groepeer op
de **rubriek die je in stap 3 aan elke rekening hebt toegekend**, niet op rekeningnummer: welke
nummers een administratie gebruikt ligt niet vast, welke rubriek een rekening bedient wel.

| Rubriek | Berekening |
|---------|------------|
| 1a - Verschuldigde BTW hoog tarief | SUM(Amount) over de rekeningen die je als "af te dragen BTW hoog tarief" hebt geclassificeerd, Type 20 + 21 |
| 1b - Verschuldigde BTW laag tarief | SUM(Amount) over de rekeningen "af te dragen BTW laag tarief", Type 20 + 21 |
| 4a - ICP verwervingen | SUM(Amount) over de rekeningen "af te dragen BTW ICP-verwervingen", Type 30 |
| 4b - Verlegde BTW | SUM(Amount) over de rekeningen "af te dragen verlegde BTW", Type 30 |
| 5b - Voorbelasting | SUM(Amount) over de rekeningen "te vorderen (voor)belasting", Type 30 + 40 |
| Per saldo | 1a + 1b + 4a + 4b + 5b (negatief = te betalen, positief = terug te ontvangen) |

Vergelijk het per-saldo bedrag met het Amount-veld uit de Returns.

**Tolerantie**: werk relatief, niet met een vast bedrag. Een afwijking tot circa 0,1% van het
totaal verschuldigde BTW-bedrag is doorgaans afronding over veel boekingsregels. Hanteer daarbij
een ondergrens van een paar euro, zodat een kleine administratie niet op een afwijking van
€ 2 alarm slaat. Noem in het rapport altijd zowel het verschil als de tolerantie waartegen je
hebt getoetst, zodat de gebruiker de beoordeling kan navolgen. Alles daarboven onderzoek je
regel voor regel.

**ICP-boekingen**: Bij ICP verwervingen (4a/4b) wordt de BTW zowel als verschuldigd als
aftrekbaar geboekt, op de bijbehorende rekeningen uit stap 3. Het netto-effect in de aangifte
is € 0, maar de bruto bedragen moeten wel kloppen.

### Stap 7: Check openstaande items

Controleer factoren die de aangifte nog kunnen beïnvloeden:

1. **BTW-afdrachtboekingen (Type 50)**: Controleer of de memoriaalboekingen in de periode
   de BTW-afdracht van het vorige kwartaal bevatten. Het saldo van alle BTW-rekeningen
   na deze boeking moet (nagenoeg) nul zijn voor de vorige periode.

2. **Openstaande boekingsvoorstellen**: Facturen die nog niet geboekt zijn maar in de periode vallen.

3. **Naboekingen**: Boekingen na de periodeafsluiting die aan een afgesloten periode zijn toegerekend.

4. **BTW-tussenrekeningen**: Controleer of deze op nul staan na de afdrachtboeking.

### Stap 8: Genereer rapport

Presenteer de bevindingen als een duidelijk overzicht. Gebruik dit format:

```
BTW-controle rapport, Q4 2025 (oktober t/m december)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stand per [datum] | Aangifte status: [Concept/Ingediend/Verwerkt]

✅ Verschuldigde BTW hoog tarief (1a):     € 47.830,34
   Verkoop:       € 54.700,99
   Creditnota's:  € -6.870,65

✅ ICP verwervingen (4a):                  € 12.073,14
✅ Verlegde BTW (4b):                      €  8.627,39
   Totaal verschuldigd:                    € 68.530,87

✅ Voorbelasting 9% (5b):                  €     81,00
✅ Voorbelasting 21% (5b):                 € 53.429,85
   Totaal voorbelasting:                   € 53.510,85

✅ Per saldo terug te ontvangen:           € 15.020,02
✅ Aangifte bedrag:                        € 15.032,00

⚠️ Afrondingsverschil:                    €     11,98
   → 0,02% van het verschuldigde bedrag, binnen tolerantie

Aanbevelingen:
1. Geen actie nodig, aangifte sluit aan
2. Afrondingsverschil kan worden opgenomen in volgende periode
```

## Veelvoorkomende aandachtspunten

- **Privégebruik auto**: Forfaitaire BTW-correctie moet als aparte boeking in het laatste kwartaal
- **Kleine ondernemersregeling (KOR)**: Check of de ondernemer hiervoor in aanmerking komt
- **Margeregeling**: Bij tweedehands goederen geldt een afwijkende BTW-berekening
- **Factuurdatum vs boekperiode**: Een factuur van 31 maart die in april wordt geboekt hoort
  bij Q1 maar staat misschien in Q2, signaleer dit
- **Verlegde BTW**: Controleer dat bij verlegde BTW zowel de verschuldigde als de te vorderen BTW correct zijn geboekt
- **Memoriaalboekingen (Type 50)**: Bevat altijd de BTW-afdracht van de VORIGE periode.
  Deze moet worden uitgefilterd bij de controle van de huidige periode.

## API-gedrag om rekening mee te houden

| Punt | Toelichting |
|------|-------------|
| VATCodes zit in de `Vat`-service | Niet in `Financial` |
| `TransactionType` bestaat niet op VATCodes | Gebruik `EUSalesListing` om ICP-gerelateerde codes te herkennen |
| Returns heeft `Frequency`, niet `ReportingFrequency` | Het veld heet Frequency |
| BTW-rekeningen herken je aan `GLAccount.Type = 24` | Een nummerbereik (vaak 1500-1599) is hooguit een noodgreep als Type 24 niets oplevert, en moet dan handmatig geverifieerd worden. Type 12 = Bank en Type 22 = Crediteuren zijn nooit BTW-rekeningen |
| analyze_data: gebruik `VATAmountDC` | `AmountVATFC` is vreemde valuta, `VATAmountDC` is divisievaluta |
| TransactionLines zit in de `Financialtransaction`-service | Niet in `Financial`, bij gebruik via `read_operation` |
| `read_operation` levert maximaal 60 records | Splits de call op zodra je precies 60 rijen terugkrijgt |
| analyze_data en list_available_tables bestaan niet op Essentials | Alleen op Trial en Analytics. Doe de controle op Essentials volledig via ReportingBalance |

## Communicatie

- Presenteer als "Stand per [datum]", nooit als "definitief"
- Vermeld als periodes nog open zijn dat cijfers voorlopig zijn
- Gebruik € Nederlands formaat (€ 1.234,56)
- Geef concrete aanbevelingen bij afwijkingen
- Bij afwijkingen buiten de tolerantie: adviseer contact met accountant
- Leg uit dat Type 50 memoriaalboekingen de BTW-afdracht van vorige periodes bevatten
