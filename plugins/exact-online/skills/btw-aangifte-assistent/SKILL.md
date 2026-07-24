---
name: btw-aangifte-assistent
description: >
  BTW-aangifte controle en voorbereiding in Exact Online. Vergelijkt BTW-bedragen in boekingen
  met de aangifte, detecteert afwijkingen, controleert intracommunautaire prestaties (ICP),
  en genereert een rapport met bevindingen en actiepunten.

  Triggers: 'btw aangifte', 'btw controle', 'btw check', 'btw aansluiting', 'btw rapport',
  'btw-aangifte voorbereiden', 'btw-aangifte controleren', 'omzetbelasting', 'btw verschil',
  'btw afsluiting', 'kwartaalaangifte', 'btw kwartaal', 'vat return', 'btw aansluitingsrapport',
  'btw periode', 'suppletie', 'btw suppletie', 'intracommunautair', 'ICP opgave',
  'btw terugvragen', 'voorbelasting', 'btw verplichtingen'.

  Gebruik deze skill wanneer de gebruiker iets met BTW-aangifte of BTW-controle wil doen
  in Exact Online, ook als ze alleen zeggen "check mijn BTW" of "klopt mijn aangifte".
  Werkt met de Exact Online MCP (execute_operation voor Returns, ReportingBalance,
  en analyze_data voor grote aggregaties).
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

### Stap 2: Haal de BTW-aangifte op

Gebruik het Returns endpoint om de ingediende of concept-aangifte op te halen:

```json
{
  "service": "Read",
  "entity": "Returns",
  "operation": "GET",
  "select": "DocumentID,Period,Year,Status,Amount,Description,Frequency,DueDate,Type"
}
```

**Let op**: Het Returns endpoint heeft beperkte velden. Beschikbare velden zijn:
DocumentID, Amount, Created, Currency, Description, DocumentViewUrl, DueDate, Frequency,
Period, PeriodDescription, Status, Subject, Type, Year.

**NIET beschikbaar**: TotalRevenue, TotalVATDue, TotalVATDeductible, ReportingFrequency.

**Status-waarden**: 10 = Concept, 20 = Open, 30 = Approved, 40 = Realized, 50 = Verwerkt
**Type-waarden**: 31 = BTW-aangifte, 32 = ICP-opgave, 146 = Loonheffing
**Frequency**: M = Maandelijks, Q = Per kwartaal, Y = Jaarlijks

Filter op Year en Period die overeenkomen met de gewenste aangifteperiode, en Type = 31 voor BTW.

### Stap 3: Haal de BTW-grootboekrekeningen op

BTW-rekeningen hebben `GLAccount.Type = 24` (VAT). Filter daarop, niet op rekeningnummer
(rekeningnummers verschillen per administratie):

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
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

**Fallback**: levert `Type: 24` niets op in een niet-standaard administratie, val dan terug op
een rekeningnummer-bereik (vaak 1500-1599) en controleer de gevonden rekeningen handmatig.

### Stap 4: Haal BTW-boekingen op uit de administratie

Dit is de kern van de controle. Gebruik ReportingBalance als primaire bron (altijd beschikbaar).

**Via ReportingBalance (aanbevolen)**:

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "GLAccountCode": { "$gte": "1500", "$lte": "1599" },
    "ReportingYear": 2025,
    "ReportingPeriod": { "$gte": 10, "$lte": 12 }
  },
  "select": "GLAccountCode,GLAccountDescription,ReportingPeriod,Amount,Type"
}
```

**CRUCIAAL — Het Type-veld op ReportingBalance**:

ReportingBalance groepeert mutaties per dagboektype. De Type-codes zijn:
- **Type 20**: Verkoopboekingen (reguliere BTW op verkoop)
- **Type 21**: Creditnota's verkoop
- **Type 30**: Inkoopboekingen (reguliere BTW op inkoop + ICP)
- **Type 40**: Bank-/kasboekingen
- **Type 50**: Memoriaalboekingen (bevat BTW-afdrachtboekingen van VORIGE periodes!)

**Filter Type 50 uit** bij het berekenen van de BTW-positie van de huidige periode.
Type 50 bevat de tegenboeking van de vorige BTW-aangifte en vertekent het beeld.

**Via analyze_data (als beschikbaar en gesynchroniseerd)**:

```json
{
  "table": "Financial/TransactionLines",
  "select": ["GLAccountCode", "GLAccountDescription", "Type"],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Bedrag" },
    { "function": "SUM", "column": "VATAmountDC", "alias": "BTW_Bedrag" },
    { "function": "COUNT", "column": "ID", "alias": "Aantal_Boekingen" }
  ],
  "groupBy": ["GLAccountCode", "GLAccountDescription", "Type"],
  "filters": [
    { "column": "FinancialYear", "operator": "=", "value": 2025 },
    { "column": "FinancialPeriod", "operator": ">=", "value": 10 },
    { "column": "FinancialPeriod", "operator": "<=", "value": 12 },
    { "column": "GLAccountCode", "operator": ">=", "value": "1500" },
    { "column": "GLAccountCode", "operator": "<=", "value": "1599" }
  ]
}
```

**Let op analyze_data veldnamen**:
- Gebruik `VATAmountDC` voor BTW-bedrag in divisievaluta (DC = Division Currency)
- `AmountVATFC` bestaat ook maar is in vreemde valuta (FC = Foreign Currency)
- Check altijd of de analyze_data tabel gesynchroniseerd is via `list_available_tables`
- Bij status `syncing`: val terug op ReportingBalance via REST API

### Stap 5: Haal BTW-codes op

Haal altijd de werkelijke BTW-codes op uit de administratie:

```json
{
  "service": "Vat",
  "entity": "VATCodes",
  "operation": "GET",
  "select": "Code,Description,Percentage,EUSalesListing"
}
```

**Let op**:
- Service is `Vat` (NIET `Financial`)
- Het veld `TransactionType` bestaat NIET op VATCodes
- Gebruik `EUSalesListing` om ICP-gerelateerde codes te identificeren

Dit is nodig om ICP-leveringen, verlegd BTW, en andere speciale situaties te identificeren.

### Stap 6: Vergelijk en analyseer

Bereken de BTW-positie door ReportingBalance data te groeperen (excl. Type 50):

| Rubriek | Berekening |
|---------|------------|
| 1a - Verschuldigde BTW 21% | SUM(Amount) op rek. 1502, Type 20 + 21 |
| 4a - ICP verwervingen | SUM(Amount) op rek. 1506, Type 30 |
| 4b - Verlegde BTW | SUM(Amount) op rek. 1507, Type 30 |
| 5b - Voorbelasting | SUM(Amount) op rek. 1510 + 1511, Type 30 + 40 |
| Per saldo | 1a + 4a + 4b + 5b (negatief = te betalen, positief = terug te ontvangen) |

Vergelijk het per-saldo bedrag met het Amount-veld uit de Returns.

**Tolerantie**: Afrondingsverschillen tot €50 zijn acceptabel bij administraties met veel
boekingsregels. Bij kleinere administraties: tot €5 is normaal.

**ICP-boekingen**: Bij ICP verwervingen (4a/4b) wordt de BTW zowel als verschuldigd (1506/1507)
als aftrekbaar (1511) geboekt. Het netto-effect in de aangifte is €0, maar de bruto bedragen
moeten wel kloppen.

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
BTW-controle rapport — Q4 2025 (oktober t/m december)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
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
   → Binnen tolerantie

Aanbevelingen:
1. Geen actie nodig — aangifte sluit aan
2. Afrondingsverschil kan worden opgenomen in volgende periode
```

## Veelvoorkomende aandachtspunten

- **Privégebruik auto**: Forfaitaire BTW-correctie moet als aparte boeking in het laatste kwartaal
- **Kleine ondernemersregeling (KOR)**: Check of de ondernemer hiervoor in aanmerking komt
- **Margeregeling**: Bij tweedehands goederen geldt een afwijkende BTW-berekening
- **Factuurdatum vs boekperiode**: Een factuur van 31 maart die in april wordt geboekt hoort
  bij Q1 maar staat misschien in Q2 — signaleer dit
- **Verlegde BTW**: Controleer dat bij verlegde BTW zowel de verschuldigde als de te vorderen BTW correct zijn geboekt
- **Memoriaalboekingen (Type 50)**: Bevat altijd de BTW-afdracht van de VORIGE periode.
  Deze moet worden uitgefilterd bij de controle van de huidige periode.

## Bekende API-eigenaardigheden

| Oorspronkelijk | Correct | Toelichting |
|----------------|---------|-------------|
| service: "Financial", entity: "VATCodes" | service: "Vat", entity: "VATCodes" | VATCodes zit in de Vat-service |
| select: "TransactionType" op VATCodes | Niet beschikbaar | Gebruik EUSalesListing i.p.v. TransactionType |
| select: "ReportingFrequency" op Returns | select: "Frequency" | Veld heet Frequency, niet ReportingFrequency |
| GLAccounts Type 12/22 voor BTW | GLAccountCode $gte 1500 $lte 1599 | Type 12=Bank, Type 22=Crediteuren |
| analyze_data: AmountVATFC | analyze_data: VATAmountDC | VATAmountDC = Division Currency (juist), AmountVATFC = Foreign Currency |
| Status 30 = Verwerkt op Returns | Status 50 = Processed | Volledige reeks: -10/0/5/10/20/30/40/50 |
| service: "Financial", entity: "TransactionLines" | service: "Financialtransaction", entity: "TransactionLines" | TransactionLines zit in Financialtransaction-service |

## Communicatie

- Presenteer als "Stand per [datum]", nooit als "definitief"
- Vermeld als periodes nog open zijn dat cijfers voorlopig zijn
- Gebruik € Nederlands formaat (€ 1.234,56)
- Geef concrete aanbevelingen bij afwijkingen
- Bij grote afwijkingen (>€500): adviseer contact met accountant
- Leg uit dat Type 50 memoriaalboekingen de BTW-afdracht van vorige periodes bevatten
