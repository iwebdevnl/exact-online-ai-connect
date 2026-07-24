---
name: periodeafsluiting
description: >
  Maand- of kwartaalafsluiting checklist voor Exact Online. Loopt systematisch alle controles
  door: bankboekingen verwerkt, boekingsvoorstellen afgehandeld, terugkerende boekingen aangemaakt,
  tussenrekeningen op nul, BTW-aansluiting. Bereidt de afsluiting voor; het daadwerkelijk sluiten
  van de periode gebeurt handmatig door de gebruiker in Exact Online (geen API-endpoint hiervoor).

  Triggers: 'periodeafsluiting', 'maandafsluiting', 'kwartaalafsluiting', 'periode afsluiten',
  'month-end close', 'maand afsluiten', 'afsluiting checklist', 'periodes sluiten',
  'dagboeken afsluiten', 'financiële afsluiting', 'close checklist', 'einde maand',
  'einde kwartaal', 'afsluiting controleren', 'maandwerk', 'periode dicht',
  'openstaande periodes', 'welke periodes staan open'.

  Gebruik deze skill ook wanneer de gebruiker vraagt "wat moet ik nog doen voor de maandafsluiting"
  of "is alles klaar voor het afsluiten van de maand". Werkt met Exact Online MCP voor
  JournalStatusList, ReportingBalance, GLAccounts, en TransactionLines.
---

# Periodeafsluiting

Systematische maand- of kwartaalafsluiting voor Exact Online. Voorkomt dat je een periode
afsluit terwijl er nog openstaande items zijn. Deze skill werkt in **elke** administratie omdat
hij classificeert op **rekeningtype** (`GLAccount.Type`) en op dagboeksoort, niet op
administratie-specifieke grootboeknummers, dagboeknummers of -namen.

## Waarom dit generiek is (lees dit eerst)

Grootboeknummers, dagboeknummers en -namen verschillen per administratie: 1050 is niet altijd
een kruispost, dagboek 20 is niet altijd de Rabobank. Wat wél in elke Exact Online-administratie
identiek is, is het veld `Type` op de grootboekrekening — een vaste, platform-brede classificatie
(bijv. 12 = Bank, 24 = VAT, 90 = General). **Classificeer op `Type` en op omschrijving-trefwoorden,
nooit op vast nummer of naam.** Zo blijft de afsluiting correct, ongeacht het rekeningschema van
de klant.

> Let op: haal de dagboeken en grootboekrekeningen altijd eerst live op uit de administratie
> zelf. Ga nooit uit van vaste nummers uit een voorbeeld — die zijn illustratief.

## Waarom systematisch afsluiten

Een periode afsluiten zonder controle is risicovol: vergeten boekingsvoorstellen, onverwerkte
bankregels, tussenrekeningen die niet op nul staan, en terugkerende boekingen die ontbreken.
Deze skill loopt een checklist door en geeft per onderdeel een duidelijke status.

## Afsluiting Checklist

### Stap 0: Verbinden

Roep `get_started` aan (regelt inlog en administratiekeuze). Werk vanaf dit punt uitsluitend met
data uit de gekozen administratie.

### Stap 1: Bepaal de periode

Vraag de gebruiker welke periode(s) ze willen afsluiten. Bepaal:
- **Boekjaar** (FinancialYear)
- **Periode(s)** (1-12 voor maanden)
- **Type**: maandafsluiting of kwartaalafsluiting

### Stap 2: Check periodes — wat is al gesloten?

Controleer de status van de financiële periodes via de dagboekstatus:

```json
{
  "service": "Read",
  "entity": "JournalStatusList",
  "operation": "GET",
  "select": "Journal,JournalDescription,Period,Year,Status"
}
```

Filter op de gewenste Year en Period. Status 0 = open, Status 1 = gesloten.
Alle dagboeken moeten Status=1 hebben voor een volledig gesloten periode.
Als één dagboek nog open is, is de periode niet volledig afgesloten.

**Toon de dagboeken zoals ze in déze administratie heten en genummerd zijn** — neem de
`Journal` en `JournalDescription` rechtstreeks uit de response over. Verzin geen namen.

Voor een indeling naar soort kun je de dagboeken groeperen op hun aard: bank/kas, inkoop,
verkoop en memoriaal. Als je die groepering nodig hebt, leid ze af uit het dagboektype in de
administratie (of vraag het de gebruiker eenmalig), niet uit een vast nummer.

### Stap 3: Controleer bankboekingen

Zijn alle bankafschriften verwerkt voor de periode? Gebruik ReportingBalance (altijd beschikbaar):

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "ReportingYear": 2026,
    "ReportingPeriod": 1,
    "Type": 40
  },
  "select": "GLAccountCode,GLAccountDescription,Amount"
}
```

Type 40 = Bank-/kasboekingen in ReportingBalance. Dit geeft een overzicht van de netto mutaties
per bankrekening in de periode.

**Bepaal de bank-/kasrekeningen generiek** via het grootboektype, niet via een codebereik.
Liquide rekeningen zijn `GLAccount.Type` in {10 = Cash, 12 = Bank, 14 = Credit card,
16 = Payment services}:

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": { "Type": [10, 12, 14, 16] },
  "select": "Code,Description,Type,TypeDescription"
}
```

**Alternatief via analyze_data** (als beschikbaar):
```json
{
  "table": "Financial/TransactionLines",
  "filters": [
    { "column": "FinancialYear", "operator": "=", "value": 2026 },
    { "column": "FinancialPeriod", "operator": "=", "value": 1 },
    { "column": "Type", "operator": "=", "value": 40 }
  ],
  "aggregations": [
    { "function": "SUM", "column": "AmountDC", "alias": "Totaal_Bank" },
    { "function": "COUNT", "column": "ID", "alias": "Aantal_Regels" }
  ],
  "groupBy": ["JournalCode", "JournalDescription"]
}
```

Controleer ook of de banksaldi aansluiten bij de werkelijke banksaldi (dit vereist handmatige
bevestiging van de gebruiker).

### Stap 4: Controleer tussenrekeningen

Tussenrekeningen (kruisposten, vraagposten, uitzoekrekening, tussenrekeningen voor uitgestelde
kosten/omzet, PSP-tussenrekeningen) moeten aan het einde van een periode op nul staan.

**BELANGRIJK — Correcte, generieke aanpak voor tussenrekeningen:**

Tussenrekeningen worden NIET gevonden via GLAccounts Type [20, 22, 24]. Die types zijn:
- Type 20 = Accounts Receivable (Debiteuren)
- Type 22 = Accounts Payable (Crediteuren)
- Type 24 = VAT (BTW-rekeningen)

**Werkelijke tussenrekeningen hebben doorgaans deze GLAccounts Types:**
- **Type 90** (General): uitzoekrekening, tussenrekening lonen, uitgestelde kosten/omzet,
  tussenrekening PSP (Mollie/Stripe), incasso te ontvangen — de meeste tussenrekeningen.
- **Type 32** (Other assets): sommige kruisposten.

**Stap 4a — haal de kandidaten op via Type, filter op omschrijving-trefwoorden:**

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": { "Type": [32, 90] },
  "select": "Code,Description,Type,TypeDescription"
}
```

Type 90 (General) bevat óók echte grootboekrekeningen. Selecteer daarom uit deze lijst alleen
de rekeningen waarvan de **omschrijving** (case-insensitive) een van deze trefwoorden bevat:

`tussenrekening`, `tussenrek`, `kruispost`, `uitzoek`, `vraagpost`, `te ontvangen`,
`te betalen`, `nog te`, `mollie`, `stripe`, `paypal`, `psp`, `incasso`, `uitgestelde`,
`overloop`, `transitoria`, `clearing`, `suspense`, `wip`, `onderhanden`.

Zo vind je tussenrekeningen ongeacht hun nummer. **Toon de gevonden lijst aan de gebruiker**
zodat die kan bevestigen of aanvullen — een enkele afwijkend benoemde rekening kan zo alsnog
worden meegenomen. Bewaar de bevestigde lijst niet hard in de skill; hij verschilt per klant.

**Stap 4b — controleer de saldi via ReportingBalance** per gevonden `GLAccountCode`:

```json
{
  "service": "Financial",
  "entity": "ReportingBalance",
  "operation": "GET",
  "filters": {
    "GLAccountCode": "<code uit stap 4a>",
    "ReportingYear": 2026,
    "ReportingPeriod": 1
  },
  "select": "GLAccountCode,GLAccountDescription,Amount,Type"
}
```

**CRUCIAAL — ReportingBalance bevat mutaties, NIET saldi:**

ReportingBalance geeft de mutaties per periode per dagboektype. Om het werkelijke saldo te
bepalen moet je ALLE periodes t/m de huidige periode optellen, inclusief de openingsbalans.

Voor een snelle check op openstaande posten: tel per tussenrekening de mutaties op t/m de te
sluiten periode (openingsbalans + Σ mutaties). Is dat cumulatieve saldo ≠ 0, dan staan er nog
posten open op die rekening.

**ReportingBalance Type-codes die op tussenrekeningen kunnen voorkomen:**
- Type 40: Bank-/kasboekingen (kruisposten door bankafschriften)
- Type 84: Automatische boekingen uitgestelde omzet
- Type 86: Automatische boekingen uitgestelde kosten
- Type 95/96: Tegenboekingen uitgestelde omzet/kosten
- Type 50: Memoriaalboekingen (o.a. BTW-afdracht vorige periode)

Tussenrekeningen met een cumulatief saldo ≠ 0 zijn een aandachtspunt. **Uitzondering:**
rekeningen voor uitgestelde kosten/omzet mogen bewust een saldo hebben als er doorlopende
contracten zijn die over meerdere periodes lopen. Herken die aan trefwoorden als `uitgesteld`,
`overloop` of `transitoria` in de omschrijving en markeer ze als "controle" in plaats van "fout".

### Stap 5: Controleer terugkerende boekingen

Check of alle terugkerende boekingen (abonnementen, vaste lasten) voor de periode zijn aangemaakt.
Dit kan via ReportingBalance door te kijken naar memoriaalboekingen (Type 50, 90, 95, 96) voor de
periode, of via TransactionLines als analyze_data beschikbaar is. Vergelijk met een eerdere,
representatieve periode of vraag de gebruiker welke terugkerende boekingen verwacht worden.

### Stap 6: Controleer boekingsvoorstellen

Zijn er nog openstaande inkoopfactuur-voorstellen die in de te sluiten periode thuishoren?
Signaleer deze aan de gebruiker — ze moeten eerst verwerkt of afgekeurd worden.

### Stap 7: BTW-aansluiting

Voer een snelle BTW-check uit (verwijs naar de btw-aangifte-assistent skill voor een
uitgebreide controle). Controleer minimaal of de BTW-rekeningen na afdracht op nul staan.

**Bepaal de BTW-rekeningen generiek** via het grootboektype, niet via een codebereik.
BTW-rekeningen hebben `GLAccount.Type` = 24 (VAT):

```json
{
  "service": "Financial",
  "entity": "GLAccounts",
  "operation": "GET",
  "filters": { "Type": [24] },
  "select": "Code,Description,Type,TypeDescription"
}
```

Controleer per gevonden BTW-rekening het saldo via ReportingBalance (zoals in stap 4b).

**Filter Type 50 uit** voor de BTW-positie van de huidige periode. Type 50 bevat de
BTW-afdracht van het vorige tijdvak en vertekent het beeld. Zie btw-aangifte-assistent
skill voor details.

Na een BTW-afdrachtboeking (Type 50) moeten de BTW-rekeningen voor het VORIGE tijdvak
nagenoeg op nul staan.

### Stap 8: Genereer afsluitingsrapport

Presenteer de bevindingen als een duidelijk overzicht. Neem de dagboeken en rekeningen over
zoals ze in de administratie heten. Onderstaand format is een **voorbeeld met fictieve namen
en nummers** — vul het met de werkelijke data uit de klant-administratie:

```
Periodeafsluiting rapport — Januari 2026 (periode 1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stand per [datum]

Dagboeken status (P1 2026):
  ✅ <Bankdagboek A>: gesloten
  ✅ <Bankdagboek B>: gesloten
  ✅ <Inkoopboek>: gesloten
  ✅ <Verkoopboek>: gesloten
  ⚠️ <Memoriaal>: nog open

Tussenrekeningen:
  ❌ <code> Kruisposten:              € 498,40  (2 openstaande regels)
  ✅ <code> Kruisposten pin:          € 0,00
  ⚠️ <code> Uitgestelde kosten:       € 2.713,40  (doorlopende contracten)
  ⚠️ <code> Uitgestelde omzet:        € 836,27   (doorlopende contracten)
  ✅ <code> Incasso te ontvangen:     € 0,00

BTW-aansluiting:
  ✅ BTW-afdracht vorig tijdvak verwerkt (Type 50 memoriaalboekingen)
  ✅ BTW-rekeningen vorig tijdvak: nagenoeg nul

Controles:
  ✅ Bankboekingen: verwerkt
  ✅ Terugkerende boekingen: aangemaakt
  ✅ Boekingsvoorstellen: geen openstaand

Aanbevelingen:
1. Onderzoek kruisposten <code> (€ 498,40) — letter af of boek af
2. Uitgestelde kosten/omzet: normaal bij doorlopende contracten
3. Sluit memoriaaldagboek af na correctie kruisposten
4. Daarna kan de periode definitief gesloten worden
```

### Stap 9: Periode afsluiten — dit doet de gebruiker zelf, handmatig

**De Exact Online API biedt geen endpoint om een periode te sluiten.** Deze skill kan de
afsluiting dus niet uitvoeren — de rol van de skill eindigt bij stap 8: een volledig checklist-
rapport met status per dagboek, tussenrekening en BTW-positie.

Als alle controles uit stap 1-8 groen zijn (of de openstaande punten zijn bewust geaccepteerd),
informeer de gebruiker dat zij de periode zelf moeten afsluiten in de Exact Online-webinterface:

1. Navigeer naar **Financieel → Periodes** (of **Instellingen → Boekhouding → Periodes afsluiten**,
   afhankelijk van de indeling van de administratie).
2. Selecteer het boekjaar en de periode die volgens dit rapport klaar is om te sluiten.
3. Bevestig het afsluiten in de Exact Online-interface zelf.

Vermeld er expliciet bij dat een gesloten periode in Exact Online desgewenst weer heropend kan
worden. Bied nooit aan om de periode "voor" de gebruiker te sluiten en doe geen `execute_operation`-
aanroep die dit suggereert — er bestaat geen schrijfbare entity hiervoor.

## Bekende API-eigenaardigheden

| Verwachting | Correct | Toelichting |
|----------------|---------|-------------|
| GLAccounts Type [20, 22, 24] voor tussenrekeningen | Type [32, 90] + filter op omschrijving-trefwoorden | Type 20=Debiteuren, 22=Crediteuren, 24=VAT — dit zijn GEEN tussenrekeningen |
| Bank/kas via vast dagboek- of codebereik | GLAccount.Type in {10, 12, 14, 16} | Bank/kas staat per administratie op andere nummers; het Type is platform-breed |
| BTW via vast codebereik (bijv. 1500-1599) | GLAccount.Type = 24 | Codebereik verschilt per administratie; Type is platform-breed |
| ReportingBalance = saldo | ReportingBalance = mutaties per periode | Om werkelijk saldo te berekenen: openingsbalans + SUM(mutaties alle periodes) |
| service: "Financial", entity: "TransactionLines" | service: "Financialtransaction", entity: "TransactionLines" | TransactionLines zit in de Financialtransaction-service |
| ReportingBalance Type 50 = BTW huidige periode | Type 50 = memoriaal (BTW-afdracht VORIG tijdvak) | Altijd uitfilteren bij controle huidige periode |

## Communicatie

- Presenteer de checklist altijd met duidelijke ✅/⚠️/❌ statusaanduidingen
- Toon dagboeken en rekeningen zoals ze in de betreffende administratie heten; verzin geen namen
- Geef concrete aanbevelingen bij elk aandachtspunt
- Wees expliciet dat het daadwerkelijk afsluiten van de periode een handmatige stap is die de
  gebruiker zelf in Exact Online uitvoert — er is geen API-endpoint hiervoor, dus deze skill
  sluit nooit zelf een periode af
- Vermeld dat gesloten periodes in Exact Online weer geopend kunnen worden indien nodig
- Vermeld dat tussenrekeningen voor uitgestelde kosten/omzet bewust een saldo kunnen hebben
  bij doorlopende contracten — dit is niet per se een fout
