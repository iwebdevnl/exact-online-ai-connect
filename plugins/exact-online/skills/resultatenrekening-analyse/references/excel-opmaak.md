# Uitvoer van de resultatenrekening

## Eerst: wat kan deze omgeving?

Deze skill mag niet aannemen dat er een spreadsheet-generator beschikbaar is.

- Beschikt de omgeving over een spreadsheet- of xlsx-vaardigheid (een skill, tool of
  bibliotheek waarmee een `.xlsx` gemaakt kan worden), gebruik die dan en volg de opmaak
  hieronder.
- Zo niet, lever de resultatenrekening dan als tabel in het antwoord zelf, met dezelfde
  secties, subtotalen en tussentellingen. Meld daarbij expliciet dat een Excel-export in deze
  omgeving niet beschikbaar is, zodat de gebruiker weet dat dit een bewuste keuze is en geen
  weglating.

Verwijs nooit naar een vast bestandspad voor een xlsx-vaardigheid. Zo'n pad bestaat niet in
elke omgeving en de instructie faalt dan stil.

## Tabblad 1: "Per Grootboekrekening"

- Rijen: gegroepeerd per sectie, in de volgorde van het BW2-model.
- Kolommen: Code | Omschrijving | per maand | YTD.
- Sectieheaders met het sectielabel.
- Subsectie-headers binnen Overige bedrijfskosten (Type 121, 123, 126). Gebruik de RGS-
  ClassificationDescription als die beschikbaar is, anders `TypeDescription` uit de API.
- Subtotalen per sectie en de tussentellingen uit `bw2-model.md`.
- Gebruik Excel-formules (`=SUM`) voor de totalen, geen hardgecodeerde uitkomsten.

## Tabblad 2: "Samenvatting"

Compacte weergave: alle secties en tussentellingen per maand plus YTD.

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

## Opmaak

| Element | Specificatie |
|---|---|
| Font | Arial 10pt |
| Headers | wit op donkerblauw (#2F5496) |
| Sectieheaders | lichtblauw (#D6E4F0), bold |
| Tussentellingen | lichtgroen (#E2EFDA), bold |
| Resultaatrij | wit op blauw (#4472C4), bold |
| Bedragen | `#,##0.00`, negatief tussen haakjes |
| Kolombreedte | Code 8, Omschrijving 48, Bedragen 15 |

Lever je de rapportage als tabel in het antwoord in plaats van als bestand, houd dan dezelfde
volgorde en dezelfde tussentellingen aan. De opmaakkolom hierboven is dan niet van toepassing.
