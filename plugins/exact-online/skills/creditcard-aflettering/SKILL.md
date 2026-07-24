---
name: creditcard-aflettering
description: >
  Verwerk een creditcard- of PSP-afschrift volledig in Exact Online: van PDF inlezen tot
  afletteren. Extraheert transacties, matcht openstaande inkoopfacturen, importeert als
  BankEntry in het creditcarddagboek en lettert af via MatchSets. Koersverschillen en
  koersopslagen naar de koersverschillenrekening. Regels zonder factuur worden NOOIT
  automatisch op een kostenrekening geboekt — altijd eerst overzicht tonen en wachten op
  expliciete instructie per regel van de gebruiker.

  Triggers: 'creditcard afletteren', 'creditcard verwerken', 'creditcard afschrift
  importeren', 'PSP afletteren', 'Mollie verwerken', 'Stripe verwerken',
  'bankafschrift creditcard', 'creditcard facturen koppelen', 'creditcard dagboek',
  'creditcard PDF inlezen', 'creditcard transacties boeken', 'bankregel aanmaken'.

  Vereist: Exact Online MCP, PDF-afschrift als bijlage.
---

# Creditcard-aflettering — van PDF tot afgeletterd

Volledige verwerking van een creditcard- of PSP-afschrift in vijf fasen:

```
Fase 0 → Intake                 Dagboek en rekeningnummers ophalen bij de gebruiker
Fase 1 → PDF inlezen            Transacties extraheren en structureren
Fase 2 → Voorbereiding          GUIDs en openstaande facturen ophalen
Fase 2.5 → Ontbrekende facturen Overzicht presenteren + wachten op actie gebruiker
Fase 3 → BankEntry import       Transacties aanmaken in Exact Online dagboek
Fase 4 → Afletteren             MatchSets per factuur
```

> ⚠️ **Regel**: Regels zonder factuur worden **nooit automatisch op een kostenrekening
> geboekt**. Ze worden altijd eerst in een overzicht gepresenteerd. De gebruiker beslist
> per regel: factuur uploaden/aanmaken, alvast op crediteuren boeken, of expliciet
> toestemming geven voor directe kostenboeking — inclusief de gewenste rekening.

---

## Fase 0: Intake — configuratie ophalen

Deze skill werkt in elke administratie. Bepaal daarom de dagboeken en rekeningen live uit de
administratie op basis van hun **type**, en gebruik geen vaste "standaard"-nummers — die
verschillen per organisatie en leiden bij een andere klant tot een verkeerde boeking.

**Creditcarddagboek.** Vraag welk dagboek het is (code of naam) of leid het af. Bankdagboeken
hebben `Type` 12; een creditcard-/PSP-dagboek staat soms als een apart banktype. Verifieer:

```json
{
  "service": "Financial", "entity": "Journals", "operation": "GET",
  "filters": {"Code": "{dagboekcode}"},
  "select": "Code,Description,Type,GLAccount,GLAccountCode"
}
```

Verwacht een bankdagboek (Type 12; een correct ingericht creditcard-/PSP-dagboek kan afwijken).
Meld het gevonden type aan de gebruiker in plaats van hard te stoppen bij ≠ 12, zodat een
legitiem creditcarddagboek niet onterecht wordt geweigerd.

**Crediteurenrekening — afleiden via Type 22, niet aannemen.** Crediteuren hebben
`GLAccount.Type` = 22 (Accounts payable). Haal ze op en kies bij meerdere de juiste samen met
de gebruiker:

```json
{
  "service": "Financial", "entity": "GLAccounts", "operation": "GET",
  "filters": { "Type": [22] },
  "select": "ID,Code,Description,Type"
}
```

**Koersverschillenrekening — verplicht opvragen, geen default.** Koersverschil is een
W&V-kostenrekening zonder onderscheidend `Type`, dus een type-lookup is hier niet mogelijk.
Vraag de gebruiker expliciet welke grootboekrekening voor koersverschillen gebruikt wordt en
toon geen gesuggereerd standaardnummer. Bevestig de gekozen code voordat je verder gaat.

Bewaar voor de rest van de verwerking:
- `dagboekcode` — code van het creditcarddagboek
- `rek_crediteuren` / `guid_crediteuren` — code én GUID van de crediteurenrekening (Type 22)
- `rek_koersverschillen` / `guid_koersverschillen` — code én GUID, door de gebruiker opgegeven

---

## Fase 1: PDF inlezen en transacties structureren

Vraag de gebruiker het PDF-afschrift te uploaden als de bijlage ontbreekt.

Extraheer per transactieregel:

| Veld | Beschrijving | Voorbeeld |
|------|-------------|---------|
| `datum` | Transactiedatum | 2026-02-03 |
| `omschrijving` | Originele omschrijving uit PDF | "Supplier invoice" |
| `valuta` | Transactievaluta | USD / EUR |
| `bedrag_vreemd` | Bedrag in transactievaluta | 100.00 |
| `bedrag_eur` | Bedrag in EUR (afgeschreven) | 95.80 |
| `koersopslag` | Apart vermelde koersopslag | 0.38 |
| `leverancier` | Herkende leveranciersnaam | "Supplier B.V." |
| `factuurnummer` | Herkend factuurnummer uit omschrijving | "INV-2026-001" |
| `type` | `factuur` / `kosten` / `koersopslag` / `verrekening` | factuur |

**Speciale regels:**
- **Verrekening vorig overzicht**: positief bedrag — overslaan bij BankEntry aanmaken, verwerken via OpeningBalance
- **Koersopslagen**: apart vermeld in het afschrift → boek op `rek_koersverschillen` als losse BankEntryLine
- **Regels zonder factuur**: markeer als `ontbrekend` — worden **niet** automatisch verwerkt

Presenteer een gestructureerde tabel ter bevestiging vóór je verder gaat:

```
Datum      | Leverancier         | EUR     | Type         | Factuur / Actie
-----------|---------------------|---------|--------------|---------------------
2026-02-03 | Supplier A          | 1010,05 | factuur      | INV-2026-010
2026-02-05 | Supplier B          |   17,05 | ⚠️ ontbreekt  | factuur aanmaken?
2026-02-07 | Koersopslag Bank    |    0,38 | koersopslag  | → rek_koersverschillen
...
```

Vraag bevestiging of correcties vóór Fase 2.

---

## Fase 2: Voorbereiding in Exact Online

Voer de volgende lookups parallel uit:

### 2a. GUIDs ophalen voor vaste rekeningen
```json
{
  "service": "Financial", "entity": "GLAccounts", "operation": "GET",
  "filters": {"Code": ["{rek_crediteuren}", "{rek_koersverschillen}"]},
  "select": "ID,Code,Description"
}
```
Bewaar: `guid_crediteuren`, `guid_koersverschillen`.
Kostenrekeningen worden pas opgezocht nádat de gebruiker ze expliciet heeft aangewezen in Fase 2.5.

### 2b. Leverancier GUIDs ophalen
```json
{
  "service": "CRM", "entity": "Accounts", "operation": "GET",
  "filters": {"Name": ["{leverancier1}", "{leverancier2}", "..."]},
  "select": "ID,Name"
}
```
Maak een lookup-tabel: `leveranciersnaam → Account GUID`.

### 2c. Openstaande inkoopfacturen ophalen
```json
{
  "service": "Bulk", "entity": "Cashflow/Payments", "operation": "GET",
  "filters": {"Status": [20, 30]},
  "select": "ID,AccountName,AmountDC,InvoiceNumber,EntryNumber,Description,InvoiceDate"
}
```

Match elke transactie uit Fase 1 aan een factuur. Match op **leverancier + factuurnummer +
bedrag**, waarbij het bedrag binnen een tolerantie mag vallen in plaats van exact gelijk te
zijn. Dat is belangrijk: juist bij vreemde valuta wijkt het in EUR afgeschreven bankbedrag
structureel af van het factuurbedrag (de reden dat de koersverschil-route in Fase 4 bestaat).
Een exacte-bedrag-eis zou precies die valutafacturen onterecht als "ontbrekend" bestempelen en
de koersverschil-afhandeling nooit laten aankomen.

Match-logica:
- **Exacte match** (leverancier + bedrag gelijk): direct koppelen.
- **Match met verschil** (leverancier klopt, factuurnummer herkenbaar, bedrag binnen tolerantie
  — bijv. een paar procent of een vast bedrag door koersopslag): koppelen én markeren als
  "koersverschil", zodat Fase 4 het restant via `write_off` naar de koersverschillenrekening
  boekt. Toon het verschil in het overzicht zodat de gebruiker het kan controleren.
- **Geen betrouwbare match** (leverancier onbekend, of bedrag wijkt te veel af zonder plausibele
  valuta-/opslagverklaring): categorie "ontbrekend" → Fase 2.5.

Noteer per match: `factuur_EntryNumber`, `factuur_ID` en het eventuele bedragverschil.

### 2d. OpeningBalance ophalen
```json
{
  "service": "Financialtransaction", "entity": "BankEntries", "operation": "GET",
  "filters": {"JournalCode": "{dagboekcode}"},
  "select": "EntryID,EntryNumber,ClosingBalanceFC,FinancialYear,FinancialPeriod"
}
```
`OpeningBalanceFC` = `ClosingBalanceFC` van de chronologisch laatste boeking. Sorteer daarvoor
op FinancialYear + FinancialPeriod (en pas daarbinnen op EntryNumber) — `EntryNumber desc` alleen
geeft niet gegarandeerd de laatste boeking, want de nummering hoeft niet gelijk te lopen met de
datumvolgorde bij terugwerkende boekingen. Bij twijfel: laat de gebruiker het slotsaldo van het
vorige afschrift bevestigen. Bij de eerste boeking in het dagboek: gebruik 0.

**Verrekening vorig overzicht.** Een positief "verrekening"-bedrag bovenaan het afschrift is het
openstaande slotsaldo van het vorige overzicht. Dat hoort in `OpeningBalanceFC` te zitten en
wordt niet als losse BankEntryLine geboekt. Controleer expliciet dat het verrekenbedrag aansluit
op het `ClosingBalanceFC` van de vorige boeking; wijkt het af, meld dat en zoek uit waar het
verschil zit voordat je de BankEntry aanmaakt — anders faalt de saldovalidatie of wordt de
verrekening dubbel/niet geboekt.

---

## Fase 2.5: Ontbrekende facturen — overzicht en beslissing

**Stop hier** als er na Fase 2 nog transacties zijn zonder gekoppelde factuur.

Presenteer een overzicht van alle ontbrekende facturen:

```
ONTBREKENDE FACTUREN — actie vereist
══════════════════════════════════════════════════════════════════
 #  Datum      Leverancier      Bedrag    Omschrijving afschrift
────────────────────────────────────────────────────────────────
 1  05-02-2026 Supplier B       €  17,05  Subscription jan
 2  11-02-2026 Supplier C       €  84,13  Invoice feb 2026
══════════════════════════════════════════════════════════════════
Totaal ontbrekend: € 101,18

Voor elke regel, geef aan wat er moet gebeuren:
  A) Factuur wordt nog geüpload/aangemaakt in Exact Online
     → Geef aan wanneer dit gedaan is, dan gaan we verder
  B) Alvast op crediteuren boeken — factuur koppelen zodra beschikbaar
     → Geef aan en we boeken de regel op {rek_crediteuren} met leverancier
  C) Rechtstreeks op kostenrekening boeken (geen factuur beschikbaar)
     → Geef per regel de gewenste rekeningcode op
```

**Wacht op expliciete instructie van de gebruiker per regel** voordat je verdergaat naar Fase 3.

| Actie gebruiker | Wat de skill doet |
|-----------------|-------------------|
| "Factuur is aangemaakt" | Opnieuw ophalen via Cashflow/Payments, dan matchen |
| "Boek alvast op crediteuren" | BankEntryLine op `guid_crediteuren` + leverancier GUID |
| "Boek op rekening X" | GUID van X ophalen, BankEntryLine op X boeken |
| Geen reactie / onduidelijk | Niet verder gaan — nogmaals vragen |

> **Nooit zelf een rekening kiezen of aanname doen.** De gebruiker moet altijd expliciet
> de rekening of actie benoemen. Bij twijfel: vragen.

Pas als alle regels een actie hebben gekregen: doorgaan naar Fase 3.

---

## Fase 3: BankEntry aanmaken in Exact Online

Maak één BankEntry aan met alle transacties als geneste BankEntryLines.

**Saldoberekening:**
```
ClosingBalance = OpeningBalance + som_alle_regels
```
Betalingen zijn **negatief**. Koersopslagen ook **negatief**.
Verrekening vorig overzicht: **niet** als BankEntryLine — verwerkt via OpeningBalance.

### POST BankEntry met geneste lines

```json
{
  "service": "Financialtransaction",
  "entity": "BankEntries",
  "operation": "POST",
  "confirmed": true,
  "data": {
    "JournalCode": "{dagboekcode}",
    "FinancialYear": {jaar},
    "FinancialPeriod": {periode},
    "OpeningBalanceFC": {vorig_slotbalans},
    "ClosingBalanceFC": {berekend_slotbalans},
    "Currency": "EUR",
    "BankEntryLines": [
      {
        "Date": "{datum}",
        "Description": "{leverancier} {factuurnummer}",
        "AmountFC": -{bedrag},
        "GLAccount": "{guid_crediteuren}",
        "Account": "{guid_leverancier}"
      },
      {
        "Date": "{datum}",
        "Description": "Koersopslag {leverancier}",
        "AmountFC": -{koersopslag},
        "GLAccount": "{guid_koersverschillen}"
      }
    ]
  }
}
```

**Regels per type:**

| Type | GLAccount | Account | AmountFC |
|------|-----------|---------|----------|
| Factuur | `guid_crediteuren` | `guid_leverancier` | negatief |
| Alvast op crediteuren (geen factuur) | `guid_crediteuren` | `guid_leverancier` | negatief |
| Kosten (na expliciete bevestiging + rekening) | `guid_kostenrekening` | optioneel | negatief |
| Koersopslag (apart op afschrift) | `guid_koersverschillen` | leeg | negatief |

> **Koersopslagen** zijn apart vermelde bankkosten op het afschrift. Ze worden als losse
> BankEntryLines op `guid_koersverschillen` geboekt en horen altijd in de BankEntry.
> Ze worden **niet** in Fase 2.5 gepresenteerd als ontbrekende factuur.

Na aanmaken: noteer `EntryID` en `EntryNumber` uit de response.

**Losse BankEntryLine toevoegen achteraf** (bijv. vergeten koersopslag):
```json
{
  "service": "Financialtransaction", "entity": "BankEntryLines", "operation": "POST",
  "confirmed": true,
  "data": {
    "EntryID": "{EntryID}",
    "Date": "{datum}", "Description": "{omschrijving}",
    "AmountFC": -{bedrag}, "GLAccount": "{guid}"
  }
}
```

---

## Fase 4: Afletteren via MatchSets

### 4a. TransactionLine IDs bankregels ophalen

Filter op de crediteurenrekening via de al opgehaalde GUID (`guid_crediteuren`) in plaats van een
los ingevoerd nummer — zo hangt de afletter niet af van een handmatig correct getypt rekeningnummer.

```json
{
  "service": "Financialtransaction", "entity": "TransactionLines", "operation": "GET",
  "filters": {"EntryNumber": "{EntryNumber}", "GLAccount": "{guid_crediteuren}"},
  "select": "ID,AmountDC,AccountName,Description"
}
```

### 4b. TransactionLine IDs facturen ophalen

```json
{
  "service": "Financialtransaction", "entity": "TransactionLines", "operation": "GET",
  "filters": {"EntryNumber": ["{entryNr1}", "{entryNr2}", "..."], "GLAccount": "{guid_crediteuren}"},
  "select": "ID,AmountDC,AccountName,EntryNumber"
}
```

### 4c. MatchSets per factuur

```json
{
  "service": "Financial", "entity": "MatchSets",
  "operation": "POST", "confirmed": true,
  "data": {
    "matches": [
      {"line_id": "{bankregel_TL_ID}", "line_type": "TransactionLine"},
      {"line_id": "{factuur_TL_ID}", "line_type": "TransactionLine"}
    ]
  }
}
```

Bij **koersverschil** (bankbedrag ≠ factuurbedrag) boek je het restant weg via `write_off` naar
de door de gebruiker opgegeven koersverschillenrekening:
```json
"write_off": {
  "gl_account_code": "{rek_koersverschillen}",
  "date": "{datum_afschrift}",
  "type": 3
}
```
`type: 3` = debet (afschrift > factuur) · `type: 4` = credit (afschrift < factuur).

> Controleer de debet/credit-richting vóór je dit op grote schaal draait: een verkeerde `type`
> boekt het koersverschil met het omgekeerde teken. Verifieer de richting één keer met een echte
> testmatch (of tegen de MatchSets-documentatie) en houd het teken van het bedragverschil uit
> Fase 2c ernaast: afschrift > factuur hoort bij het ene type, afschrift < factuur bij het andere.

### 4d. N:1 matching (meerdere bankregels → 1 factuur)

```json
{
  "matches": [
    {"line_id": "{bankregel_1}", "line_type": "TransactionLine"},
    {"line_id": "{bankregel_2}", "line_type": "TransactionLine"},
    {"line_id": "{factuur_TL_ID}", "line_type": "TransactionLine"}
  ]
}
```

Bij N:1 met resterend verschil: match alleen de beschikbare factuur af. De bankregelregel
blijft voor het restant open staan totdat de bijbehorende factuur beschikbaar is.

---

## Verificatie

```json
{
  "service": "Bulk", "entity": "Cashflow/Payments", "operation": "GET",
  "filters": {"Status": [20, 30]},
  "select": "AccountName,AmountDC,InvoiceNumber"
}
```

Presenteer een eindoverzicht:
- ✅ Afgeletterd
- ⚠️ Koersverschil geboekt op koersverschillenrekening
- 📋 Geboekt op kostenrekening (geen factuur)
- 🔗 Op crediteuren geboekt, wacht op factuur
- ❌ Niet verwerkt (periodegrens of ontbrekende factuur)

---

## Snelle referentie

| Actie | Service | Entity | Methode |
|-------|---------|--------|---------|
| Dagboek verifiëren | Financial | Journals | GET |
| GLAccount GUIDs | Financial | GLAccounts | GET |
| Leverancier GUIDs | CRM | Accounts | GET |
| Openstaande facturen | Bulk | Cashflow/Payments | GET |
| Vorig banksaldo | Financialtransaction | BankEntries | GET |
| BankEntry aanmaken | Financialtransaction | BankEntries | POST |
| Losse BankEntryLine toevoegen | Financialtransaction | BankEntryLines | POST |
| TL IDs ophalen | Financialtransaction | TransactionLines | GET |
| Afletteren | Financial | MatchSets | POST |

Zie `references/beperkingen-en-fouten.md` voor foutmeldingen en workarounds.
