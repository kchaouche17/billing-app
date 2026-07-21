# 04 — Architecture : calculs au backend

Julien a livré son implémentation C#. **Les calculs lourds sont faits au backend**,
directement dans `Fact_Maestro depense projet`.

## Ce qui est désormais caduc

- La table calculée `Cumul_Ventes_Rep` (`ADDCOLUMNS` + `TREATAS`).
- Les mesures DAX lourdes qui refaisaient ces calculs côté Power BI.

## Colonnes pré-calculées livrées par le backend

| Colonne | Pilier |
|---|---|
| `BonusSalesPersonCumulativeAmount` | commission (cumulative) |
| `BonusSalesPersonMarginAmount` | boni marge (margin) |
| `SalesPersonYearlyCumulativeInclusiveAmount` | cumul inclusif |
| `SalesPersonYearlyCumulativeExclusiveAmount` | cumul exclusif |
| `BonusSalesPersonCumulativeProjectAvailableAmount` | commission — disponible au projet |

## Convention de nommage

| Suffixe | Signification |
|---|---|
| `*_margin` | boni marge |
| `*_cumulative` | commission ventes cumulées |
| `*_total` | somme des deux |
