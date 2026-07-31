# OffsetID Matching Logica

## Wat is OffsetID?

OffsetID is een GUID-veld op `TransactionLines` dat een boekingsregel koppelt aan een tegenpost. De interpretatie verschilt per context: een gevulde OffsetID betekent NIET automatisch dat een post is afgeletterd.

## Twee Soorten OffsetID

### 1. Within-Entry OffsetID (NIET afgeletterd)

Binnen een bankentry linkt de debiteurenregel naar de bankrekening-regel van **dezelfde boeking**. Dit is de standaard dubbelboeking-structuur en betekent NIET dat de betaling is gematcht met een factuur.

```
Bankentry 26200012:
  [Bank]      (ID=cd036d1a, OffsetID=null)       ← bankzijde, altijd null
  [Debiteuren](ID=a7212274, OffsetID=cd036d1a)   ← wijst naar Bank = WITHIN-ENTRY
```

### 2. Cross-Entry OffsetID (WEL afgeletterd)

Na aflettering wijst de debiteurenregel van de verkoopfactuur naar de debiteurenregel van de bankentry (en omgekeerd). Dit is de echte reconciliatie.

```
Verkoopfactuur 26700001:
  [Debiteuren](ID=b35c5e65, OffsetID=a7212274)   ← wijst naar bank-debiteurenregel = AFGELETTERD

Bankentry 26200012:
  [Debiteuren](ID=a7212274, OffsetID=b35c5e65)   ← wijst naar factuur-debiteurenregel = AFGELETTERD
```

## Methoden om Afletterstatus te Bepalen

### Methode 1: Receivables/Payables Endpoint (AANBEVOLEN)

Meest betrouwbaar en eenvoudigst. Geen interpretatie van OffsetID nodig.

**Tool: `read_operation`**

```json
{
  "service": "Cashflow",
  "entity": "Receivables",
  "filters": {"InvoiceNumber": "{factuurnummer}"},
  "select": "ID,AccountName,InvoiceNumber,AmountDC,Description"
}
```

| AmountDC | Status |
|----------|--------|
| `= 0` | Volledig afgeletterd |
| `> 0` | Openstaand (niet of deels betaald) |
| `< 0` | Onafgeletterd tegoed (betaling ontvangen maar niet gematcht) |

Voor crediteuren: gebruik `Cashflow/Payments` (niet `Payables`).

### Methode 2: OffsetID op Factuurzijde

Controleer de debiteurenregel op het **verkoopdagboek**, NIET op het bankdagboek:

**Tool: `read_operation`**

```json
{
  "service": "Financialtransaction",
  "entity": "TransactionLines",
  "filters": {
    "JournalCode": "{verkoopdagboek}",
    "GLAccountCode": "{debiteurenCode}",
    "EntryNumber": "{factuurnummer}"
  },
  "select": "ID,EntryNumber,AmountDC,OffsetID,AccountName"
}
```

- `OffsetID = null` → **NIET afgeletterd**
- `OffsetID = GUID` → **WEL afgeletterd** (GUID wijst naar de bank-debiteurenregel)

### Methode 3: OffsetID op Bankzijde (ONBETROUWBAAR)

De debiteurenregel op het bankdagboek heeft ALTIJD een OffsetID; deze wijst naar de bankrekening-regel van dezelfde entry. Dit is NIET bruikbaar als afletter-indicator.

## Matching Patronen

### 1:1 Betaling, niet afgeletterd

```
Bankentry:
  [Bank]       (ID=AAA, OffsetID=null)
  [Debiteuren] (ID=BBB, OffsetID=AAA)   ← wijst naar Bank = within-entry

Verkoopfactuur:
  [Debiteuren] (ID=CCC, OffsetID=null)  ← null = NIET afgeletterd
  [Omzet]      (ID=DDD, OffsetID=CCC)
```

### 1:1 Betaling, afgeletterd

```
Bankentry:
  [Bank]       (ID=AAA, OffsetID=null)
  [Debiteuren] (ID=BBB, OffsetID=CCC)   ← wijst naar factuur = AFGELETTERD

Verkoopfactuur:
  [Debiteuren] (ID=CCC, OffsetID=BBB)   ← wijst naar bank = AFGELETTERD
  [Omzet]      (ID=DDD, OffsetID=CCC)
```

### Creditnota, afgeletterd tegen factuur

```
Verkoopfactuur:
  [Debiteuren] (ID=CCC, OffsetID=FFF, AmountDC=+1000)   ← AFGELETTERD

Creditnota:
  [Debiteuren] (ID=FFF, OffsetID=CCC, AmountDC=-1000)   ← AFGELETTERD
```

Beide zijn TransactionLine type bij MatchSets (beide komen van het verkoopdagboek).

### N:1 Betaling, meerdere facturen

```
Bankentry:
  [Bank]       (ID=AAA, OffsetID=null)
  [Debiteuren] (ID=BBB, OffsetID=AAA)   ← totaalbedrag

Verkoopfactuur 1:
  [Debiteuren] (ID=CCC, OffsetID=BBB)   ← AFGELETTERD naar bank

Verkoopfactuur 2:
  [Debiteuren] (ID=DDD, OffsetID=BBB)   ← AFGELETTERD naar bank
```

### Incasso Batch (1:N)

```
Bankentry:
  [Bank]           (ID=AAA, OffsetID=null)
  [Incasso]        (ID=CCC, OffsetID=AAA)       ← batch tussenrekening
  [Debiteuren] A   (ID=DDD, OffsetID=CCC)       ← wijst naar batch
  [Debiteuren] B   (ID=EEE, OffsetID=CCC)       ← wijst naar batch
  [Debiteuren] C   (ID=FFF, OffsetID=CCC)       ← wijst naar batch
```

Elke debiteurenregel wordt individueel afgeletterd via MatchSets tegen de bijbehorende factuur.

### Kruisposten (parkeerrekeningen)

```
Bankentry:
  [Bank]         (ID=AAA, OffsetID=null)
  [Kruisposten]  (ID=BBB, OffsetID=AAA)  ← within-entry, NOOIT cross-entry
```

Kruisposten worden niet afgeletterd via MatchSets. Ze worden opgelost door een memoriaalpost die het bedrag uitsplitst naar specifieke kosten- of opbrengstrekeningen.

## GLAccount Types en OffsetID Gedrag

| Type | TypeDescription | OffsetID op bankdagboek | OffsetID op factuur/inkoopdagboek |
|------|----------------|------------------------|----------------------------------|
| 12 | Bank | Altijd null | n.v.t. |
| 90 | General (kruisposten) | Within-entry (→ Bank) | n.v.t. |
| 20 | Accounts receivable | Within-entry (→ Bank) | null = onafgeletterd, GUID = afgeletterd |
| 22 | Accounts payable | Within-entry (→ Bank) | null = onafgeletterd, GUID = afgeletterd |

## Detectie Betalingen Zonder Factuur

**Belangrijk**: OffsetID = null is NIET de juiste indicator voor "betaling zonder factuur" op het bankdagboek. De juiste detectie is:

| Patroon | GLAccount Type | InvoiceNumber | OffsetID | Betekenis |
|---------|---------------|---------------|----------|-----------|
| Betaling zonder factuur | 22 (crediteuren) | `null` | gevuld (within-entry) | Bankbetaling gedaan, geen inkoopfactuur aangemaakt |
| Geparkeerd bedrag | 90 (kruisposten) | `null` | gevuld (within-entry) | Bedrag geparkeerd, moet uitgesplitst |
| Koersverschil | 22 (crediteuren) | gevuld | `null` | Klein bedrag, omschrijving "Koersverschil" |
| Betalingsverschil | 22 (crediteuren) | gevuld | `null` | Klein bedrag, omschrijving "Betalingsverschil" |

**Samenvattend**: Op het bankdagboek geldt:
- `OffsetID = null` → koers-/betalingsverschillen (klein, write-off kandidaten)
- `InvoiceNumber = null` op crediteurenrekening → betaling zonder inkoopfactuur (actie vereist)

## Samenvatting Beslisboom

```
Is het een factuur-/inkoopdagboek-regel?
  ├─ Ja → Check OffsetID:
  │       ├─ null     → NIET afgeletterd
  │       └─ GUID     → WEL afgeletterd (GUID = bank-debiteurenregel)
  └─ Nee (bankdagboek) → OffsetID is ALTIJD gevuld (within-entry)
                          ├─ Check InvoiceNumber:
                          │    ├─ null → betaling zonder factuur (actie vereist)
                          │    └─ gevuld → betaling met factuur
                          └─ Gebruik Receivables/Payments endpoint als alternatief
```
