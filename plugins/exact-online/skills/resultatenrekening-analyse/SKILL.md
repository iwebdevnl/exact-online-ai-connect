---
name: resultatenrekening-analyse
description: >
  Genereer een correcte resultatenrekening (W&V) vanuit Exact Online via ReportingBalance met
  BalanceType=W filtering en classificatie op basis van GLAccount.Type (niet rekeningnummers).
  Rapportagestructuur volgt BW2 Titel 9 categoraal model.
  Triggers: 'resultatenrekening', 'winst- en verliesrekening', 'W&V', 'P&L', 'profit and loss',
  'income statement', 'exploitatieoverzicht', 'resultaat per maand', 'omzet en kosten overzicht',
  'hoe staan we ervoor', 'wat is ons resultaat', 'maandresultaat', 'kwartaalresultaat',
  'jaarresultaat', 'omzet analyse', 'kosten analyse', 'hoeveel winst maken we',
  'hoeveel omzet hebben we', 'resultatenrekening maken', 'financieel overzicht'.
  Werkt met Exact Online MCP (ReportingBalance, GLAccounts).
---

# Resultatenrekening Analyse

Genereer een volledige, correcte resultatenrekening vanuit Exact Online. Classificatie is
volledig gebaseerd op het `Type` veld van de GLAccounts API — NIET op rekeningnummerreeksen.
Rapportagestructuur volgt het Nederlandse categorale model (BW2 Titel 9, Besluit modellen
jaarrekening).

## Waarom deze skill nodig is

Veel implementaties classificeren grootboekrekeningen op basis van nummerbereiken (4xxx = kosten,
8xxx = omzet). Dit is ONBETROUWBAAR omdat:

- Administraties vrij zijn in hun rekeningschema
- 9xxx rekeningen (koersverschillen, rente, belasting) worden gemist
- Rekeningnummers niet standaard zijn tussen administraties

De Exact Online GLAccounts API bevat een `Type` veld met gestandaardiseerde classificaties die
exact aansluiten bij de posten van een categorale resultatenrekening. Gebruik ALTIJD dit veld.

## Classificatie op basis van GLAccount.Type

De GLAccounts API (`Financial/GLAccounts`) bevat per rekening een `Type` (Int32) met de
volgende W&V-relevante waarden:

### Rapportagestructuur (BW2 Titel 9 — Categoraal model)

De Nederlandse wet (BW2 Titel 9, Besluit modellen jaarrekening) schrijft voor kleine
rechtspersonen het categorale model voor (Model I, of uitgesplitst Model E). De structuur:

```
1. NETTO-OMZET
   └─ Type 110: Revenue

2. KOSTPRIJS VAN DE OMZET (indien van toepassing)
   └─ Type 111: Cost of goods sold
   ─────────────────────────────────────────
   BRUTO-MARGE (= 1 - 2)

3. PERSONEELSKOSTEN
   └─ Type 125: Employee costs (lonen, sociale lasten, pensioen, overig)

4. AFSCHRIJVINGEN
   └─ Type 122: Depreciation costs

5. OVERIGE BEDRIJFSKOSTEN
   ├─ Type 121: Sales, general & administrative expenses (SGA)
   ├─ Type 123: Research and development
   └─ Type 126: Employment costs (huisvesting, transport, kantoor)
   ─────────────────────────────────────────
   BEDRIJFSRESULTAAT (= Bruto-marge - 3 - 4 - 5)

6. FINANCIËLE BATEN EN LASTEN
   └─ Type 160: Interest income (bevat zowel baten als lasten)

7. BUITENGEWONE BATEN EN LASTEN
   ├─ Type 120: Other costs (koersverschillen, betalingsverschillen)
   ├─ Type 130: Exceptional costs
   └─ Type 140: Exceptional income
   ─────────────────────────────────────────
   RESULTAAT VOOR BELASTINGEN

8. BELASTINGEN
   └─ Type 150: Income taxes (VPB)
   ─────────────────────────────────────────
   RESULTAAT NA BELASTINGEN

9. OVERIG
   └─ Type 90: General (resultaatrekening, VPB voorgaande jaren)
   ─────────────────────────────────────────
   NETTORESULTAAT
```

### Sign conventions

- **Type 110 (Omzet)**: Bedragen zijn NEGATIEF in Exact (credit). Flip sign voor weergave.
- **Type 111 (Kostprijs omzet)**: Positief (debit). Toon als kosten.
- **Type 125, 122, 121, 123, 126 (Bedrijfskosten)**: Positief (debit). Toon als kosten.
- **Type 160 (Financieel)**: Baten zijn negatief (credit), lasten positief (debit).
  Check `BalanceSide` van de GLAccount: C = baten (flip), D = lasten (ongewijzigd).
- **Type 120, 130 (Kosten)**: Positief (debit).
- **Type 140 (Buitengewone baten)**: Negatief (credit). Flip sign.
- **Type 150 (Belasting)**: Positief (debit).
- **Type 90 (General)**: Check `BalanceSide`: C = baten (flip), D = lasten.

**Vuistregel**: Gebruik de `BalanceSide` van de GLAccount om te bepalen of een bedrag
geflipped moet worden. `BalanceSide = "C"` (credit) → flip sign voor weergave als positief.
`BalanceSide = "D"` (debit) → ongewijzigd.

## Stap-voor-stap werkwijze

### Stap 1: Bepaal de scope

Vraag (of leid af uit context):
- **Boekjaar**: Welk jaar? (standaard: huidig jaar)
- **Periodes**: Welke maanden? (standaard: alle beschikbare periodes)
- **Vergelijking**: Wil de gebruiker vergelijken met vorig jaar?

### Stap 2: Haal de GLAccounts classificatie op

#### 2a. GLAccounts Type (VERPLICHT)

Haal EERST de grootboekrekeningen op met hun Type-classificatie:

```
execute_operation:
  service: "Financial"
  entity: "GLAccounts"
  operation: "GET"
  filters:
    BalanceType: "W"
  select: "Code,Description,Type,TypeDescription,BalanceSide"
  top: 60
```

Pagineer tot je alle W&V-rekeningen hebt. Bewaar als lookup-map:
`Code → {Type, TypeDescription, BalanceSide}`.

Dit is ESSENTIEEL — zonder deze stap kun je de rekeningen niet correct classificeren.

#### 2b. GLAccountClassificationMappings (OPTIONEEL — verrijking met RGS subcategorieën)

Het endpoint `Financial/GLAccountClassificationMappings` bevat de koppeling tussen
grootboekrekeningen en het Referentie Grootboekschema (RGS). Dit levert granulaire
subcategorieën die de rapportage verrijken.

```
execute_operation:
  service: "Financial"
  entity: "GLAccountClassificationMappings"
  operation: "GET"
  top: 60
```

Pagineer volledig (kan 200-600+ records bevatten, meerdere schemes per rekening).

**Data-structuur per record:**

| Veld                     | Voorbeeld                                        | Gebruik                     |
|--------------------------|--------------------------------------------------|-----------------------------|
| GLAccountCode            | "4020"                                           | Koppeling naar rekening     |
| ClassificationCode       | "WPerLesOvt"                                     | RGS code (hiërarchisch)     |
| ClassificationDescription| "Overige toeslagen lonen en salarissen"           | Leesbare subcategorie       |
| GLSchemeCode             | "RGS versie 3.2"                                 | Welk schema                 |
| GLSchemeDescription      | "Referentie GrootboekSchema versie 3.2"          | Schema-omschrijving         |

**RGS ClassificationCode structuur:**

De eerste letter geeft het hoofdniveau aan:
- `B` = Balans
- `W` = Winst & Verlies

De volgende 3-lettergroepen zijn hiërarchisch, bijv.:
- `WPer` = W&V → Personeelskosten
- `WPerLes` = W&V → Personeelskosten → Lonen en salarissen
- `WPerLesOvt` = W&V → Personeelskosten → Lonen en salarissen → Overige toeslagen
- `WBedKan` = W&V → Bedrijfskosten → Kantoorkosten
- `WBedHui` = W&V → Bedrijfskosten → Huisvestingskosten
- `WOmzNod` = W&V → Omzet → Netto-omzet diensten
- `WFbeRls` = W&V → Financiële baten/lasten → Rentelasten
- `WAfs` = W&V → Afschrijvingen

**Gebruik voor subcategorieën:**

Filter op records met `GLSchemeCode` die begint met "RGS" (negeer het basis
"Grootboekrekeningschema" scheme — dat heeft minder detail). Gebruik het 2e niveau
van de ClassificationCode (eerste 4-6 tekens) als subcategorie-groepering:

```python
# Voorbeeld: groepeer W&V-rekeningen per RGS subcategorie
rgs_subcats = {}
for mapping in classification_mappings:
    if not mapping['GLSchemeCode'].startswith('RGS'):
        continue
    code = mapping['GLAccountCode']
    rgs_code = mapping['ClassificationCode']
    rgs_desc = mapping['ClassificationDescription']
    
    # Neem het 2e hiërarchieniveau als subcategorie (bijv. "WPerLes", "WBedKan")
    if len(rgs_code) >= 4:
        subcat_key = rgs_code[:4]  # bijv. "WPer", "WBed", "WOmz"
    rgs_subcats[code] = {
        "rgs_code": rgs_code,
        "rgs_description": rgs_desc,
        "subcat_key": subcat_key,
    }
```

**Wanneer gebruiken:**
- Gebruik RGS-subcategorieën om binnen een Type-sectie (bijv. Type 125 Personeelskosten)
  verder uit te splitsen naar Lonen, Sociale lasten, Overige personeelskosten
- Gebruik RGS-subcategorieën om binnen Type 121 (SGA) te splitsen naar Kantoorkosten,
  Verkoopkosten, Algemene kosten
- Als de RGS-mapping ontbreekt voor een rekening, val terug op de Type-classificatie

**Let op:**
- Niet elke administratie heeft RGS-mappings geconfigureerd
- Eén rekening kan in meerdere schemes voorkomen (RGS 3.1 én 3.2) — gebruik de
  hoogste versie
- Het basis "Grootboekrekeningschema" (GLSchemeCode="1") bevat minder granulaire
  classificaties (bijv. "GC325 Exploitatiekosten") — bruikbaar als fallback

### Stap 3: Probeer analyze_data (snelste route)

Controleer of `Financial/ReportingBalance` beschikbaar is via `list_available_tables`.
Zo ja, en `sync_status = 'idle'`:

```
analyze_data:
  query:
    table: "Financial/ReportingBalance"
    select: ["GLAccountCode", "GLAccountDescription", "ReportingPeriod", "Amount", "BalanceType"]
    filters:
      - column: "ReportingYear"
        operator: "="
        value: <jaar>
      - column: "BalanceType"
        operator: "="
        value: "W"
    orderBy:
      - column: "GLAccountCode"
        direction: "ASC"
      - column: "ReportingPeriod"
        direction: "ASC"
    limit: 10000
```

Als analyze_data beschikbaar is en werkt, sla Stap 4 over en ga naar Stap 5.

### Stap 4: Fallback — ophalen via execute_operation

Als analyze_data niet beschikbaar is, haal de data op via de REST API met paginering:

```
execute_operation:
  service: "Financial"
  entity: "ReportingBalance"
  operation: "GET"
  filters:
    ReportingYear: <jaar>
    ReportingPeriod: <periode>
    BalanceType: "W"
  select: "ID,GLAccountCode,GLAccountDescription,ReportingPeriod,Amount,BalanceType"
  top: 60
```

**KRITIEKE REGELS:**

1. **Pagineer volledig** — `skip: 0`, `skip: 60`, `skip: 120`, etc. Stop als `count < top`.
2. **Herhaal per periode** — Doe dit voor elke maand in scope.
3. **Aggregeer per GLAccountCode + ReportingPeriod** — Eén rekening kan meerdere records
   per periode hebben (meerdere dagboeken, kostenplaatsen).

### Stap 5: Classificeer en structureer

Koppel elke GLAccountCode aan de Type uit de GLAccounts lookup (Stap 2). Bouw de
rapportagestructuur:

```python
from collections import defaultdict

# GLAccount lookup uit Stap 2
# gl_lookup = { "4000": {"Type": 125, "TypeDescription": "Employee costs", "BalanceSide": "D"}, ... }

# Rapportage-secties gebaseerd op GLAccount.Type
# Volgorde conform BW2 Titel 9 categoraal model
SECTIONS = [
    {
        "key": "omzet",
        "label": "NETTO-OMZET",
        "types": [110],
        "sign_flip": True,  # Credit → positief voor weergave
        "sort_order": 1,
    },
    {
        "key": "kostprijs",
        "label": "KOSTPRIJS VAN DE OMZET",
        "types": [111],
        "sign_flip": False,
        "sort_order": 2,
    },
    {
        "key": "personeelskosten",
        "label": "PERSONEELSKOSTEN",
        "types": [125],
        "sign_flip": False,
        "sort_order": 3,
    },
    {
        "key": "afschrijvingen",
        "label": "AFSCHRIJVINGEN",
        "types": [122],
        "sign_flip": False,
        "sort_order": 4,
    },
    {
        "key": "overige_bedrijfskosten",
        "label": "OVERIGE BEDRIJFSKOSTEN",
        "types": [121, 123, 126],
        "sign_flip": False,
        "sort_order": 5,
        # Subsecties: gebruik RGS ClassificationDescription als die beschikbaar is,
        # anders val terug op Type-based labels:
        "subsections_fallback": {
            121: "Verkoop- en beheerskosten",
            123: "Research & development",
            126: "Huisvestings- en overige kosten",
        }
        # Met RGS-verrijking worden subsecties dynamisch bepaald op basis van de
        # ClassificationCode (bijv. WBedKan=Kantoorkosten, WBedHui=Huisvestingskosten,
        # WBedAlk=Algemene kosten). Dit geeft een fijnmaziger uitsplitsing.
    },
    {
        "key": "financieel",
        "label": "FINANCIËLE BATEN EN LASTEN",
        "types": [160],
        "sign_flip": "per_account",  # Gebruik BalanceSide per rekening
        "sort_order": 6,
    },
    {
        "key": "buitengewoon",
        "label": "BUITENGEWONE BATEN EN LASTEN",
        "types": [120, 130, 140],
        "sign_flip": "per_account",
        "sort_order": 7,
    },
    {
        "key": "belastingen",
        "label": "BELASTINGEN",
        "types": [150],
        "sign_flip": False,
        "sort_order": 8,
    },
    {
        "key": "overig",
        "label": "OVERIG",
        "types": [90],
        "sign_flip": "per_account",
        "sort_order": 9,
    },
]

# Classificeer elke rekening
for code, data in agg.items():
    gl_info = gl_lookup.get(code, {})
    gl_type = gl_info.get("Type", 90)  # default naar "General" als onbekend
    balance_side = gl_info.get("BalanceSide", "D")
    
    for section in SECTIONS:
        if gl_type in section["types"]:
            # Bepaal sign flip
            if section["sign_flip"] is True:
                flip = True
            elif section["sign_flip"] == "per_account":
                flip = (balance_side == "C")
            else:
                flip = False
            # Voeg toe aan sectie...
            break

# RGS-verrijking voor subcategorieën (indien beschikbaar uit Stap 2b)
# Gebruik de RGS ClassificationDescription als subsectie-label.
# Groepeer per RGS 2e niveau (eerste 4-7 tekens van ClassificationCode):
#
# Personeelskosten (Type 125) wordt bijv.:
#   WPerLes → "Lonen en salarissen" (4000, 4010, 4020)
#   WPerSol → "Sociale lasten" (4050, 4055, 4060)
#   WPerPen → "Pensioenlasten" (4080)
#   WPerOpl → "Overige personeelskosten" (4100, 4110)
#
# Overige bedrijfskosten (Type 121/123/126) wordt bijv.:
#   WBedKan → "Kantoorkosten" (4340, 4350)
#   WBedHui → "Huisvestingskosten" (4400, 4460)
#   WBedVkk → "Verkoopkosten" (4600, 4620)
#   WBedAlk → "Algemene kosten" (4810, 4830, 4950)
#
# Als RGS-mapping ontbreekt voor een rekening, gebruik de
# subsections_fallback op basis van Type.
```

### Tussentellingen (BW2 Titel 9)

Bereken de volgende tussentellingen — dit zijn de verplichte subtotalen:

1. **Bruto-marge** = Netto-omzet − Kostprijs van de omzet
2. **Totaal bedrijfskosten** = Personeelskosten + Afschrijvingen + Overige bedrijfskosten
3. **Bedrijfsresultaat** = Bruto-marge − Totaal bedrijfskosten
4. **Resultaat voor belastingen** = Bedrijfsresultaat + Financieel + Buitengewoon
5. **Nettoresultaat** = Resultaat voor belastingen − Belastingen ± Overig

Wanneer een sectie geen data bevat (bijv. geen Type 111 kostprijs), sla die sectie en
bijbehorende tussentelling over.

### Stap 6: Genereer de Excel

Maak een professioneel Excel-bestand met twee tabbladen:

**Tab 1: "Per Grootboekrekening"**
- Rijen: gegroepeerd per sectie (zie structuur boven)
- Kolommen: Code | Omschrijving | Per maand... | YTD
- Sectieheaders met de sectielabel
- Subsectie-headers voor Type 121/123/126 binnen Overige Bedrijfskosten
  (gebruik TypeDescription uit de API als subsectielabel)
- Subtotalen per sectie
- Tussentellingen (Bruto-marge, Bedrijfsresultaat, etc.)
- Gebruik Excel-formules (=SUM) voor totalen

**Tab 2: "Samenvatting"**
- Compacte weergave: alle secties en tussentellingen per maand + YTD
- Structuur conform BW2 Titel 9:
  ```
  Netto-omzet
  Kostprijs van de omzet
  ─── Bruto-marge
  Personeelskosten
  Afschrijvingen
  Overige bedrijfskosten
  ─── Bedrijfsresultaat
  Financiële baten en lasten
  Buitengewone baten en lasten
  ─── Resultaat voor belastingen
  Belastingen
  ─── Nettoresultaat
  ```

**Opmaak:**
- Font: Arial 10pt
- Headers: wit op donkerblauw (#2F5496)
- Sectieheaders: lichtblauw (#D6E4F0), bold
- Tussentellingen: lichtgroen (#E2EFDA), bold
- Resultaatrij: wit op blauw (#4472C4), bold
- Bedragen: #,##0.00 met negatief tussen haakjes
- Kolombreedte: Code=8, Omschrijving=48, Bedragen=15

**Gebruik altijd de xlsx skill** (lees `/mnt/skills/public/xlsx/SKILL.md`) voor het aanmaken.

### Stap 7: Validatie

1. **Type-check**: Elke rekening moet een bekende Type hebben uit GLAccounts. Log onbekende
   Types als waarschuwing.
2. **Sign-check**: Omzet (Type 110) moet negatief zijn in brondata. Kosten positief.
3. **Balans-check**: Nettoresultaat = som van alle W&V-bedragen (met correcte sign).
4. **YTD-check**: Per rekening: som maandkolommen = YTD-kolom.
5. **Vergelijk met Exact**: Als de gebruiker een export aanlevert, vergelijk regel-voor-regel.

## Veelvoorkomende valkuilen

1. **Classificeren op rekeningnummer i.p.v. GLAccount.Type** — Het Type veld uit de
   GLAccounts API is de enige betrouwbare classificatie. Rekeningnummers zijn niet
   gestandaardiseerd tussen administraties.

2. **GLAccounts lookup vergeten** — Zonder de Type-informatie uit GLAccounts kun je de
   rekeningen niet correct in secties indelen. Haal dit ALTIJD op in Stap 2.

3. **Niet alle pagina's ophalen** — ReportingBalance bevat 50-120 W&V-records per periode.
   Bij `top: 60` zijn dat 1-2 pagina's. Pagineer volledig!

4. **Dubbele records niet aggregeren** — Eén grootboekrekening kan meerdere records per
   periode hebben. Aggregeer altijd per GLAccountCode + ReportingPeriod.

5. **Sign convention vergeten** — Gebruik `BalanceSide` van de GLAccount om te bepalen
   of je het teken moet flippen. Niet aannames op basis van rekeningnummer.

6. **Rapportagestructuur niet conform BW2** — De volgorde en tussentellingen zijn
   voorgeschreven. Volg altijd de structuur: Omzet → Kostprijs → Bruto-marge →
   Bedrijfskosten → Bedrijfsresultaat → Financieel → Buitengewoon → Belasting → Resultaat.

## Achtergrondinformatie

### BW2 Titel 9 — Besluit modellen jaarrekening

Het Besluit modellen jaarrekening schrijft voor kleine rechtspersonen Model I voor
(categorale resultatenrekening). De hoofdindeling:

- Bruto-marge (mag als één post)
- Lonen en salarissen
- Sociale lasten
- Afschrijvingen
- Overige bedrijfskosten
- Financiële baten en lasten
- Belastingen
- Resultaat na belastingen

Posten mogen worden uitgesplitst maar de volgorde staat vast. In de praktijk worden
personeelskosten (lonen + sociale lasten + overige personeelskosten) vaak samengevoegd,
wat door NBA en RvT wordt gedoogd.

### Exact Online GLAccount Type referentie

Alle W&V-relevante Types:

| Type | Beschrijving (API)                      | Rapportagesectie          |
|------|-----------------------------------------|---------------------------|
| 110  | Revenue                                 | Netto-omzet               |
| 111  | Cost of goods                           | Kostprijs van de omzet    |
| 120  | Other costs                             | Buitengewone baten/lasten |
| 121  | Sales, general administrative expenses  | Overige bedrijfskosten    |
| 122  | Depreciation costs                      | Afschrijvingen            |
| 123  | Research and development                | Overige bedrijfskosten    |
| 125  | Employee costs                          | Personeelskosten          |
| 126  | Employment costs                        | Overige bedrijfskosten    |
| 130  | Exceptional costs                       | Buitengewone baten/lasten |
| 140  | Exceptional income                      | Buitengewone baten/lasten |
| 150  | Income taxes                            | Belastingen               |
| 160  | Interest income                         | Financiële baten/lasten   |
| 90   | General                                 | Overig                    |

### GLAccountClassificationMappings (RGS) referentie

Het endpoint `Financial/GLAccountClassificationMappings` bevat de RGS-koppeling per
grootboekrekening. Relevante velden:

- **GLAccountCode**: Rekeningcode (koppeling)
- **ClassificationCode**: RGS-code, hiërarchisch opgebouwd
- **ClassificationDescription**: Leesbare naam van de RGS-post
- **GLSchemeCode**: Schema-versie ("RGS versie 3.1", "RGS versie 3.2", of "1" voor basis)

De RGS-codes voor W&V beginnen met `W` en zijn hiërarchisch:

| Prefix   | Betekenis                              | Voorbeeld ClassificationDescription        |
|----------|----------------------------------------|--------------------------------------------|
| WOmz     | Omzet                                  | Netto-omzet uit verleende diensten         |
| WKpv     | Kostprijs van de omzet                 | Kosten grond- en hulpstoffen               |
| WPer     | Personeelskosten                       | —                                          |
| WPerLes  | → Lonen en salarissen                  | Overige toeslagen lonen en salarissen      |
| WPerSol  | → Sociale lasten                       | Overige sociale lasten                     |
| WPerPen  | → Pensioenlasten                       | Pensioenpremies                            |
| WAfs     | Afschrijvingen                         | Afschrijvingen immateriële vaste activa    |
| WBed     | Bedrijfskosten                         | —                                          |
| WBedKan  | → Kantoorkosten                        | Kosten automatisering kantoorkosten        |
| WBedHui  | → Huisvestingskosten                   | Huur onroerende zaak                       |
| WBedVkk  | → Verkoopkosten                        | Reclame- en advertentiekosten              |
| WBedAlk  | → Algemene kosten                      | Accountantskosten                          |
| WFbe     | Financiële baten en lasten             | —                                          |
| WFbeRls  | → Rentelasten                          | Rentelasten obligatieleningen              |
| WRed     | Resultaat deelnemingen                 | Resultaat deelnemingen (dividend)          |
| WBel     | Belastingen                            | Vennootschapsbelasting                     |

Gebruik het prefix (eerste 4-7 tekens) om rekeningen binnen een Type-sectie te
groeperen in betekenisvolle subcategorieën.


