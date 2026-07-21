# 02 — Modèle de données

## Tables

| Table | Rôle |
|---|---|
| `Fact_Maestro depense projet` | Table de faits principale |
| `Dim_Date` | Dimension date — relation via **`WorkCompletedOrEndDate`** |
| `Commission_Paliers` | Lookup manuelle : `Seuil_Min`, `Seuil_max`, `Taux` (7 lignes) |
| `BoniMarge_Palliers` | Lookup manuelle : `Borne_Inf` (points de %), `Taux_Boni` |

### Relation Dim_Date

La relation avec `Dim_Date` se fait via **`WorkCompletedOrEndDate`**.

- **PAS** `date_fin_projet`
- **PAS** `WorkCompletedOn` (corrompu par les dossiers SAV de Jérémy — voir
  [pièges](06-pieges-confirmes.md))

### BoniMarge_Palliers — sentinelle

La valeur `Borne_Inf = -1000000000` est une **sentinelle** : elle capture tout ce qui
est sous -15 points de pourcentage, à un taux de boni de **0 %**.

## Colonnes clés de la Fact

- `vendeur_code`
- `nom_vendeur`
- `code_projet`
- `total_facture_sans_taxe`
- `marge`
- `BudgetExcelWeightedMargin`
- `ProjectBudgetMaestroMargin`
- `WorkCompletedOrEndDate`
- `WorkCompletedOn`
- `ProductionStatus`
- `IsProjectCompleted`

## Source des données

`maestroProjectSaleSummaries_v3.csv` sur **Azure Blob Storage**, alimenté par le backend
C# de Julien. Les dates proviennent du **CRM Dynamics**.
