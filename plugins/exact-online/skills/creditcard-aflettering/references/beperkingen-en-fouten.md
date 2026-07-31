# Beperkingen en Foutafhandeling

Aanvulling op SKILL.md: uitsluitend foutmeldingen, harde beperkingen en workarounds. De regels
zelf (OpeningBalance bepalen, verrekening vorig overzicht) staan in SKILL.md Fase 2d en Fase 3.

## BankEntry aanmaken

### ⛔ Saldovalidatie faalt

Exact Online valideert: `OpeningBalance + som(BankEntryLines) = ClosingBalance`.

**Fout**: `Opening balance plus all bank entry lines should result in the closing balance`
**Oplossing**: Controleer de som van alle AmountFC waarden en pas ClosingBalanceFC aan. Let op dat
betalingen en koersopslagen negatief zijn en dat de verrekening van het vorige overzicht in
`OpeningBalanceFC` zit en niet als losse regel.

### ⛔ BankEntryLines zijn niet verwijderbaar of wijzigbaar na POST

Eenmaal aangemaakt kan een BankEntryLine niet worden verwijderd of gewijzigd
(`DELETE not supported`, `PUT not supported`). Fouten herstellen via:
- DELETE van de hele BankEntry (header) als Status = 20 (open)
- Opnieuw aanmaken met correcte gegevens

**DELETE BankEntry:**

**Tool: `write_operation`**

```json
{
  "service": "Financialtransaction", "entity": "BankEntries",
  "operation": "DELETE",
  "confirmed": true,
  "id": "{EntryID}"
}
```
Alleen mogelijk als Status = 20 (open). Na verwerking (Status = 50) niet meer mogelijk.

### ⛔ Account verplicht bij GLAccount type 22 (crediteuren)

BankEntryLines op een crediteurenrekening (GLAccount.Type = 22) vereisen altijd een `Account`
(leverancier GUID). Zonder Account boekt Exact de regel wel, maar zonder relatiekoppeling, en
mislukt het afletteren via MatchSets later.

---

## MatchSets

### ⛔ TransactionLine ID is niet gelijk aan BankEntryLine ID

Gebruik voor MatchSets altijd het `TransactionLine ID` (via Financialtransaction/TransactionLines),
niet het `BankEntryLine ID`. Ze hebben soms hetzelfde GUID maar dat is toeval.

### ⛔ Periodegrens: factuur in afgesloten periode

**Fout**: `Betalingstermijn: GLAccount=<crediteuren> Account=X finyear=Y finperiod=Z Niet gevonden!`

Optie 1: Periode tijdelijk heropenen, MatchSets uitvoeren, weer sluiten.
Optie 2: Handmatig afletteren via Exact Online UI.

Voorbeeld: een factuur uit een eerdere, al afgesloten periode kan niet zomaar worden afgeletterd
tegen een bankregel in de huidige periode.

### ⛔ MatchSets werkt alleen op type 20/22 rekeningen

Alleen Debiteuren (type 20) en Crediteuren (type 22) kunnen via MatchSets worden afgeletterd.
Koersverschillen- en kostenrekeningen (elk ander GLAccount.Type dan 20/22) kunnen niet worden
afgeletterd, die staan gewoon in de boekhouding zonder koppeling.

---

## PDF-extractie aandachtspunten

- **Koersopslagen**: staan soms als aparte regel, soms verwerkt in het totaalbedrag.
  Check altijd of het af te schrijven bedrag overeenkomt met de factuur + opslag.
- **Valutabedragen**: het PDF toont vaak USD-bedrag + EUR-equivalent. Gebruik altijd
  het EUR-bedrag voor `AmountFC`.
- **Meerdere deelregels, één factuur** (bijvoorbeeld een advertentieplatform dat op verschillende
  data afschrijft): boek elke datum als aparte BankEntryLine op de crediteurenrekening (Type 22),
  match ze samen (N:1) via MatchSets.

---

## Checklist verwerking

```
□ PDF geüpload en alle transacties geëxtraheerd
□ Verrekening vorig overzicht geïdentificeerd (niet als BankEntryLine; sluit aan op vorig slotsaldo)
□ Koersopslagen als aparte regels gemarkeerd (naar koersverschillenrekening)
□ Regels zonder factuur geïdentificeerd (naar Fase 2.5, gebruiker beslist per regel)
□ OpeningBalance bepaald (chronologisch laatste ClosingBalance of 0)
□ ClosingBalance berekend (OpeningBalance + som alle lines)
□ Crediteurenrekening afgeleid via Type 22; koersverschillenrekening door gebruiker opgegeven
□ GLAccount GUIDs opgehaald (crediteuren, koersverschillen, evt. kostenrek)
□ Leverancier GUIDs opgehaald
□ Openstaande facturen volledig opgehaald (alle pagina's via page_token, niet afgekapt op 60)
□ Facturen gematcht aan bankregels (incl. tolerantie bij valutaverschil)
□ BankEntry POST geslaagd (EntryID genoteerd)
□ TransactionLine IDs bankregels opgehaald (filter EntryNumber + GLAccount-GUID crediteuren)
□ TransactionLine IDs facturen opgehaald (per EntryNumber)
□ MatchSets uitgevoerd per factuur
□ N:1 matches afgehandeld
□ Koersverschillen via write_off naar de koersverschillenrekening (zonder handmatig type)
□ Verificatie: openstaande Payments controle
```
