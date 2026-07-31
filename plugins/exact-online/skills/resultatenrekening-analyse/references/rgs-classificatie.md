# RGS-verrijking via GLAccountClassificationMappings

Optionele stap. Levert granulaire subcategorieën binnen een Type-sectie, bijvoorbeeld
Personeelskosten (Type 125) uitgesplitst naar Lonen, Sociale lasten en Pensioenlasten.

Niet elke administratie heeft RGS-mappings geconfigureerd. Ontbreken ze, val dan terug op de
Type-classificatie uit de hoofdskill.

## Ophalen

Met `read_operation`:

```json
{
  "service": "Financial",
  "entity": "GLAccountClassificationMappings",
  "top": 60
}
```

Pagineer volledig met `skip`, dit endpoint kan 200 tot 600 records bevatten (meerdere schemes
per rekening).

## Datastructuur per record

| Veld | Voorbeeld | Gebruik |
|---|---|---|
| GLAccountCode | `"4020"` | Koppeling naar de grootboekrekening |
| ClassificationCode | `"WPerLesOvt"` | RGS-code, hiërarchisch |
| ClassificationDescription | `"Overige toeslagen lonen en salarissen"` | Leesbare subcategorie |
| GLSchemeCode | `"RGS versie 3.2"` | Welk schema |
| GLSchemeDescription | `"Referentie GrootboekSchema versie 3.2"` | Schema-omschrijving |

## Opbouw van de ClassificationCode

De code begint met een letter voor het hoofdniveau: `B` = Balans, `W` = Winst en Verlies.
Daarna volgen blokken van drie letters, elk een niveau dieper.

**Eén regel voor het groeperen:** neem de eerste 7 tekens als subcategorie (`WPerLes`,
`WBedKan`). Is de code korter dan 7 tekens, neem dan de eerste 4 als hoofdcategorie (`WPer`,
`WBed`, `WOmz`). Gebruik overal dezelfde regel, anders vallen dezelfde rekeningen in
verschillende runs in verschillende subsecties.

Voorbeeld van de hiërarchie:

- `WPer` = W&V, Personeelskosten
- `WPerLes` = W&V, Personeelskosten, Lonen en salarissen
- `WPerLesOvt` = W&V, Personeelskosten, Lonen en salarissen, Overige toeslagen

## Filtering

Gebruik alleen records waarvan `GLSchemeCode` begint met "RGS". Het basisschema
(`GLSchemeCode = "1"`, "Grootboekrekeningschema") bevat minder granulaire classificaties
zoals "GC325 Exploitatiekosten" en is alleen bruikbaar als fallback.

Komt een rekening in meerdere RGS-versies voor (3.1 en 3.2), gebruik dan de hoogste versie.

## Prefix-referentie (W&V)

| Prefix | Betekenis | Voorbeeld ClassificationDescription |
|---|---|---|
| WOmz | Omzet | Netto-omzet uit verleende diensten |
| WKpv | Kostprijs van de omzet | Kosten grond- en hulpstoffen |
| WPer | Personeelskosten | |
| WPerLes | Lonen en salarissen | Overige toeslagen lonen en salarissen |
| WPerSol | Sociale lasten | Overige sociale lasten |
| WPerPen | Pensioenlasten | Pensioenpremies |
| WAfs | Afschrijvingen | Afschrijvingen immateriële vaste activa |
| WBed | Bedrijfskosten | |
| WBedKan | Kantoorkosten | Kosten automatisering kantoorkosten |
| WBedHui | Huisvestingskosten | Huur onroerende zaak |
| WBedVkk | Verkoopkosten | Reclame- en advertentiekosten |
| WBedAlk | Algemene kosten | Accountantskosten |
| WFbe | Financiële baten en lasten | |
| WFbeRls | Rentelasten | Rentelasten obligatieleningen |
| WRed | Resultaat deelnemingen | Resultaat deelnemingen (dividend) |
| WBel | Belastingen | Vennootschapsbelasting |

## Toepassing in de rapportage

- Binnen Type 125 (Personeelskosten): splits naar `WPerLes` Lonen en salarissen, `WPerSol`
  Sociale lasten, `WPerPen` Pensioenlasten, `WPerOpl` Overige personeelskosten.
- Binnen Type 121, 123 en 126 (Overige bedrijfskosten): splits naar `WBedKan` Kantoorkosten,
  `WBedHui` Huisvestingskosten, `WBedVkk` Verkoopkosten, `WBedAlk` Algemene kosten.
- Ontbreekt de RGS-mapping voor een rekening, gebruik dan het Type-label als subsectie:
  121 = Verkoop- en beheerskosten, 123 = Research en development, 126 = Huisvestings- en
  overige kosten.
