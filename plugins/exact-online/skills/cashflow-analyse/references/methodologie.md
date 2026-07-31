# Methodologie: wat een goede cashflow-analyse moet bevatten

Onderbouwing van de opbouw, gebaseerd op de standaardpraktijk voor het kasstroomoverzicht
(operationeel/investerings-/financieringsactiviteiten; directe vs. indirecte methode) en de
gangbare werkkapitaal- en kasstroom-KPI's.

## 1. Een goede analyse heeft drie activiteiten en twee methoden

**Drie activiteiten** (verplichte indeling van elk kasstroomoverzicht):
- **Operationeel**: kasstroom uit de dagelijkse bedrijfsvoering (resultaat + non-cash posten +
  mutaties in werkkapitaal).
- **Investering**: aan- en verkoop van vaste activa en overige langlopende activa.
- **Financiering**: eigen vermogen (stortingen, uitkeringen) en leningen (opname, aflossing).

**Twee methoden** (verschillen alleen in de operationele sectie; investering en financiering zijn
identiek):
- **Directe methode**: werkelijke ontvangsten en uitgaven per categorie. Intuïtief voor pure
  liquiditeit.
- **Indirecte methode**: begint bij het nettoresultaat en corrigeert voor non-cash posten en
  werkkapitaalmutaties. Meest gebruikt en geeft het meeste inzicht in de samenhang met balans
  en W&V.

Een goede analyse toont beide en laat zien dat ze op elkaar aansluiten.

## 2. De aansluitidentiteit (de ruggengraat)

Per periode geldt over alle grootboekrekeningen: Σ mutatie = 0 (debet = credit). Splits in
liquide (L) en niet-liquide (NL):

```
Σ_L mutatie + Σ_NL mutatie = 0  ⇒  Σ_L mutatie = − Σ_NL mutatie
```

De mutatie op de liquide rekeningen is de netto kasstroom. De indirecte methode is niets anders
dan − Σ_NL mutatie, gehergroepeerd in operationeel/investering/financiering. Daarom:

> **Kaseffect van elke niet-liquide rekening = − mutatie(AmountDC).**

Tekencontrole: omzet staat credit (−AmountDC) → kaseffect +; kosten staan debet (+AmountDC) →
kaseffect −; een hogere debiteurenstand (+AmountDC) → kaseffect − (geld zit vast); een hogere
crediteurenstand (−AmountDC) → kaseffect + (betaling uitgesteld).

## 3. Indirecte methode, opbouw

```
Nettoresultaat                                    = − Σ kaseffect omkeren? nee: = − Σ mutatie(W)
  (alle W-rekeningen samen; winst = − Σ mutatie(W))
+ Afschrijvingen en overige non-cash posten        (Type 35 terugname; bij ontbreken: Type 122)
± Mutatie werkkapitaal:
    − toename debiteuren (20,21)
    − toename voorraad (40)
    − toename vooruitbetaalde/overlopende posten (26,27)
    + toename crediteuren (22)
    + toename btw/belasting te betalen (24,29,100)
    + toename personeel te betalen (25)
= Operationele kasstroom

− Investeringen in vaste/overige activa (30,32, bruto)
= Investeringskasstroom

± Mutatie eigen vermogen excl. resultaat (50,52) en leningen (55,60)
= Financieringskasstroom

Netto kasstroom = operationeel + investering + financiering
Eindsaldo liquide middelen = beginsaldo + netto kasstroom   (moet gelijk zijn aan Σ_L mutatie)
```

In de praktijk hoeft niets handmatig: bucket elke rekening op `Type`, neem kaseffect = −mutatie,
en tel per bucket op. De identiteit garandeert dat het totaal sluit; alleen de verdeling tussen
operationeel en investering is conventie-afhankelijk (zie afschrijving in de classificatie).

## 4. Directe methode, afgeleid en sluitend

De werkelijke geldstromen per categorie worden afgeleid uit de resultaatposten, gecorrigeerd
voor de bijbehorende werkkapitaalmutatie. Dit sluit per definitie aan op de operationele
kasstroom van de indirecte methode:

```
Ontvangen van klanten        = omzet (110)                       − Δ debiteuren (20,21)
Betaald aan leveranciers      = − (kostprijs 111 + overige inkoop) + Δ crediteuren (22) − Δ voorraad (40)
Betaald aan personeel         = − (personeelskosten 125,126)      + Δ personeel te betalen (25)
Overige operationele uitgaven = − (overige kosten 120,121,123,130) − Δ vooruitbetaald (26) + Δ overlopend (27)
Belastingen                   = − (belasting 150) + Δ btw (24) + Δ belasting te betalen (29,100)
Rente                         = rente (160)
= Operationele kasstroom (gelijk aan indirecte methode)
```

Wil de gebruiker een "zuivere" directe methode (werkelijke bankregels per tegenrekening), dan kan
dat door de regels op de liquide rekeningen te groeperen per tegenrekening-`Type`; dat is
bewerkelijker en zelden nodig voor managementinzicht.

## 5. Verplichte controles (kwaliteit)

1. **Double-entry-check**: Σ mutatie over alle rekeningen per periode = 0.
2. **Aansluitcontrole**: operationeel + investering + financiering = mutatie liquide middelen
   = eindsaldo − beginsaldo. Tolerantie ± €1 (afronding).
3. **Methodecontrole**: operationele kasstroom (direct) = operationele kasstroom (indirect).
4. **Volledigheid**: geen "Niet geclassificeerd"-restpost van betekenis; anders melden.
5. **Beginsaldo**: laten bevestigen tegen de werkelijke bankstand.

## 6. KPI's en wat ze betekenen

| KPI | Formule | Signaal |
|---|---|---|
| DSO (debiteurendagen) | openstaande debiteuren / omzet × dagen | Lager = sneller geïnd |
| DPO (crediteurendagen) | openstaande crediteuren / inkoop × dagen | Hoger = langer betalen (meer cash) |
| DIO (voorraaddagen) | voorraad / kostprijs omzet × dagen | Lager = minder kapitaalbeslag |
| Kasconversiecyclus | DSO + DIO − DPO | Lager = minder werkkapitaal nodig |
| Operationele kasstroom (OCF) | uit overzicht | Kern van duurzame cashgeneratie |
| Vrije kasstroom (FCF) | OCF − investeringen | Wat overblijft voor schuld/uitkering/groei |
| Kasstroommarge | OCF / omzet | Kwaliteit van de winst (cash vs. papier) |
| Operationele-kasstroomratio | OCF / kortlopende schulden | Dekking korte verplichtingen |
| Cash runway | liquide middelen / netto kasuitstroom p/m | Maanden tot kasnood (bij negatieve OCF) |

Goede analyse zet KPI's in perspectief: vergelijk meerdere periodes (MoM/QoQ/YoY), wijs op
trends en uitschieters, en koppel ze aan acties (incasso versnellen, voorraad verlagen,
betaaltermijnen heronderhandelen, investering faseren).

## 7. Vooruitblik (licht)

Een analyse mag een korte doorkijk geven, maar de gedetailleerde 13-weeks liquiditeitsprognose
is een aparte, diepere exercitie. Houd het hier bij: run-rate van de operationele kasstroom
(laatste 3 tot 6 maanden) + zekere openstaande posten op vervaldatum, één base-scenario, en een
markering wanneer het saldo onder een veilige buffer komt. Label het expliciet als indicatief.
