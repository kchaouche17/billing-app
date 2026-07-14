# 10 — Incident & reconstruction : colonnes bonus retirées du CSV v3 (2026-07-14)

Journée de dépannage majeure. Le dashboard Reps s'est brisé après une nouvelle livraison
du backend. Ce document capture **la cause racine, les correctifs Power Query, la décision
d'architecture et les 6 mesures DAX reconstruites** — pour ne jamais refaire ce marathon.

## 1. Symptômes (dans l'ordre d'apparition)

1. Au refresh : `Expression.Error : … la colonne « BonusSalesPersonMarginPercentage » …
   introuvable` (Mashup 10224).
2. Après correction : `Failed to move the data reader to the next row`.
3. Puis erreurs DAX `Missing_References` + « Ce champ a été supprimé du modèle » sur
   plusieurs cartes et les deux tableaux.

## 2. Cause racine

Le CSV **`maestroProjectSaleSummaries_v3.csv`** de Julien **ne livre plus les colonnes
bonus pré-calculées** dont dépendait tout l'étage commission / cumul / palier :

- `BonusSalesPersonCumulativeAmount`
- `BonusSalesPersonMarginAmount`
- `BonusSalesPersonMarginPercentage`
- `SalesPersonYearlyCumulativeInclusiveAmount`
- `SalesPersonYearlyCumulativeExclusiveAmount`
- `BonusSalesPersonCumulativeProjectAvailableAmount`

Confirmé par un courriel de Julien : il a **retravaillé le calcul des jours (pro-rata) et
des bonus au backend**, avec de nouveaux champs Kronos/UKG (`WorkedDaysMaestroUkgReady`,
`ActualLabourQtyHoursMaestroUkgReady`, `ExpenseAmountLaborMaestroUkgRead`,
`TotalExpenseRealMaestroUkgReady`). Sa refonte a changé le schéma du CSV et **retiré** les
colonnes bonus. Ce n'était pas un accident côté Power BI.

> Le rapport « fonctionnait » à l'écran juste avant parce qu'il affichait des **données en
> cache**. Dès le refresh, le modèle a rechargé le v3 et les colonnes ont disparu.

## 3. Correctifs Power Query (requête `Fact_Maestro depense projet`)

| Problème | Correctif |
|---|---|
| `Changed Type` typait `BonusSalesPersonMarginPercentage` (absente) → plantage | Retiré la paire `{"BonusSalesPersonMarginPercentage", Percentage.Type}` du `Table.TransformColumnTypes`. |
| `Failed to move the data reader to the next row` au chargement complet | L'import lisait le CSV avec `QuoteStyle=QuoteStyle.None`. Le v3 contient des colonnes texte libres (`ContractClientNameCalculated`, `ContractAddress`) **avec des virgules** (adresses) → décalage de colonnes → lecture cassée sur une ligne au-delà de l'aperçu. **Corrigé : `QuoteStyle=QuoteStyle.Csv`** dans l'étape `Imported CSV`. |

> **Piège :** retirer les autres colonnes bonus du `Changed Type` faisait *taire* l'erreur
> de refresh, mais **cassait ensuite les mesures/visuels** qui en dépendaient. Soigner le
> symptôme ≠ soigner la cause.

## 4. Décision d'architecture

**On recalcule la commission en DAX**, à partir de `total_facture_sans_taxe` — on n'attend
pas que Julien remette les colonnes.

Justification :
- La commission repose sur le **cumul des ventes $** (règles stables de Vincent), pas sur
  les jours/marge que Julien refait. Les deux ne se collisionnent pas.
- `total_facture_sans_taxe` et les paliers existent toujours.
- Recalculer au backend *et* en DAX créerait deux vérités qui divergent.
- Le **boni de marge** (`Boni_Marge_Rep`) était déjà 100 % DAX et n'a jamais dépendu des
  colonnes retirées — il fonctionne tel quel.

## 5. Règles d'affaires précisées ce jour-là

- **Commission → cumul TRIMESTRIEL** (reset chaque trimestre), calcul **marginal** :
  chaque tranche de ventes taxée à son propre taux (0 – 1 M = 0,5 % ; 1 – 2 M = 1 % ;
  2 – 3 M = 1,5 % ; 3 – 4 M = 2 % ; 4 – 5 M = 2,5 % ; 5 M+ = 3 %). Bornes codées en dur
  (il n'existe **pas** de table `Commission_Paliers` dans le modèle actuel).
- **« Ventes cumulées » → rolling 12 mois** (les 12 derniers mois glissants, peu importe
  l'année civile), **PAS** l'année civile ni le reset trimestriel. Une colonne rolling
  **peut baisser** d'une ligne à l'autre : ce n'est pas une vente négative, c'est une
  vieille vente positive qui **sort** de la fenêtre de 12 mois.
- **Par projet** : contribution marginale = `marginal(cumul_inclusif) − marginal(cumul_exclusif)`,
  cumul ordonné par `WorkCompletedOrEndDate` puis **départage par `code_projet`**.
- **Marge < 0 exclue** partout (`'Fact...'[marge] >= 0`).

## 6. Les 6 mesures reconstruites

Toutes validées dans DAX Studio avant création (voir valeurs §7).

### `Commission_Rep` (par rep, auto-totalisante)
```DAX
Commission_Rep =
SUMX(
    VALUES( 'Fact_Maestro depense projet'[vendeur_code] ),
    VAR S =
        CALCULATE(
            SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ),
            'Fact_Maestro depense projet'[marge] >= 0
        )
    RETURN
          0.005 * MIN( S, 1000000 )
        + 0.010 * MAX( 0, MIN( S, 2000000 ) - 1000000 )
        + 0.015 * MAX( 0, MIN( S, 3000000 ) - 2000000 )
        + 0.020 * MAX( 0, MIN( S, 4000000 ) - 3000000 )
        + 0.025 * MAX( 0, MIN( S, 5000000 ) - 4000000 )
        + 0.030 * MAX( 0, S - 5000000 )
)
```

### `Bonus_total`
```DAX
Bonus_total = [Boni_Marge_Rep] + [Commission_Rep]
```

### `Ventes_Cumulees_YTD` (carte — rolling 12 mois finissant à la dernière date du contexte)
```DAX
Ventes_Cumulees_YTD =
VAR dMax = MAX( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
VAR borneBasse = EDATE( dMax, -12 )
RETURN
CALCULATE(
    SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ),
    'Fact_Maestro depense projet'[marge] >= 0,
    REMOVEFILTERS( Dim_Date ),
    'Fact_Maestro depense projet'[WorkCompletedOrEndDate] > borneBasse,
    'Fact_Maestro depense projet'[WorkCompletedOrEndDate] <= dMax
)
```

### `Commission_Projet` (par projet — marginal, cumul TRIMESTRIEL)
```DAX
Commission_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d   = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c   = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR rep = SELECTEDVALUE( 'Fact_Maestro depense projet'[vendeur_code] )
    VAR venteProjet =
        CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ), 'Fact_Maestro depense projet'[marge] >= 0 )
    VAR ProjetsRep =
        CALCULATETABLE(
            ADDCOLUMNS(
                SUMMARIZE( 'Fact_Maestro depense projet', 'Fact_Maestro depense projet'[code_projet], 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] ),
                "v", CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
            ),
            REMOVEFILTERS( 'Fact_Maestro depense projet' ),
            'Fact_Maestro depense projet'[vendeur_code] = rep,
            'Fact_Maestro depense projet'[marge] >= 0
        )
    VAR excl =
        SUMX(
            FILTER(
                ProjetsRep,
                'Fact_Maestro depense projet'[WorkCompletedOrEndDate] < d
                    || ( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] = d && 'Fact_Maestro depense projet'[code_projet] < c )
            ),
            [v]
        )
    VAR incl = excl + venteProjet
    VAR commIncl =
          0.005 * MIN( incl, 1000000 ) + 0.010 * MAX( 0, MIN( incl, 2000000 ) - 1000000 )
        + 0.015 * MAX( 0, MIN( incl, 3000000 ) - 2000000 ) + 0.020 * MAX( 0, MIN( incl, 4000000 ) - 3000000 )
        + 0.025 * MAX( 0, MIN( incl, 5000000 ) - 4000000 ) + 0.030 * MAX( 0, incl - 5000000 )
    VAR commExcl =
          0.005 * MIN( excl, 1000000 ) + 0.010 * MAX( 0, MIN( excl, 2000000 ) - 1000000 )
        + 0.015 * MAX( 0, MIN( excl, 3000000 ) - 2000000 ) + 0.020 * MAX( 0, MIN( excl, 4000000 ) - 3000000 )
        + 0.025 * MAX( 0, MIN( excl, 5000000 ) - 4000000 ) + 0.030 * MAX( 0, excl - 5000000 )
    RETURN commIncl - commExcl,
    [Commission_Rep]
)
```

### `Ventes_Cumulees_Projet` (par projet — rolling 12 mois)
```DAX
Ventes_Cumulees_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d   = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c   = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR rep = SELECTEDVALUE( 'Fact_Maestro depense projet'[vendeur_code] )
    VAR borneBasse = EDATE( d, -12 )
    VAR ProjetsFenetre =
        CALCULATETABLE(
            ADDCOLUMNS(
                SUMMARIZE( 'Fact_Maestro depense projet', 'Fact_Maestro depense projet'[code_projet], 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] ),
                "v", CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
            ),
            REMOVEFILTERS( 'Fact_Maestro depense projet' ),
            REMOVEFILTERS( Dim_Date ),
            'Fact_Maestro depense projet'[vendeur_code] = rep,
            'Fact_Maestro depense projet'[marge] >= 0,
            'Fact_Maestro depense projet'[WorkCompletedOrEndDate] > borneBasse,
            'Fact_Maestro depense projet'[WorkCompletedOrEndDate] <= d
        )
    RETURN
        SUMX(
            FILTER(
                ProjetsFenetre,
                'Fact_Maestro depense projet'[WorkCompletedOrEndDate] < d
                    || ( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] = d && 'Fact_Maestro depense projet'[code_projet] <= c )
            ),
            [v]
        ),
    BLANK()
)
```

### `Palier_Projet` (par projet — palier du cumul TRIMESTRIEL inclusif)
```DAX
Palier_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d   = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c   = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR rep = SELECTEDVALUE( 'Fact_Maestro depense projet'[vendeur_code] )
    VAR venteProjet =
        CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ), 'Fact_Maestro depense projet'[marge] >= 0 )
    VAR ProjetsRep =
        CALCULATETABLE(
            ADDCOLUMNS(
                SUMMARIZE( 'Fact_Maestro depense projet', 'Fact_Maestro depense projet'[code_projet], 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] ),
                "v", CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
            ),
            REMOVEFILTERS( 'Fact_Maestro depense projet' ),
            'Fact_Maestro depense projet'[vendeur_code] = rep,
            'Fact_Maestro depense projet'[marge] >= 0
        )
    VAR excl =
        SUMX(
            FILTER(
                ProjetsRep,
                'Fact_Maestro depense projet'[WorkCompletedOrEndDate] < d
                    || ( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] = d && 'Fact_Maestro depense projet'[code_projet] < c )
            ),
            [v]
        )
    VAR incl = excl + venteProjet
    RETURN
        SWITCH( TRUE(),
            incl >= 5000000, "Palier 6 — 3,00%",
            incl >= 4000000, "Palier 5 — 2,50%",
            incl >= 3000000, "Palier 4 — 2,00%",
            incl >= 2000000, "Palier 3 — 1,50%",
            incl >= 1000000, "Palier 2 — 1,00%",
            "Palier 1 — 0,50%"
        ),
    BLANK()
)
```

> **Note sur le cumul par projet :** on relit les projets du rep avec
> `REMOVEFILTERS('Fact...')` (enlève le filtre de la ligne courante) tout en gardant la
> propagation de `Dim_Date` (le trimestre) pour la commission/palier. Pour le rolling
> 12 mois, on ajoute `REMOVEFILTERS( Dim_Date )` et on borne à `EDATE( d, -12 )`.
> Le filtre **rep** provient du slicer `Représentants` (sur `Dim_Vendeurs`), qui survit à
> `REMOVEFILTERS('Fact...')`.

## 7. Validation (rep 20 = Stéphane Taillefer, Q1 2026)

| Métrique | Recalcul DAX | Référence mémo (instantané) |
|---|---|---|
| Cumul ventes trimestriel | **768 404,63** | 768 964,62 |
| Commission | **3 842,02** | 3 844,82 |
| Commission totale (tous reps) | **16 253,86** | 16 271,66 |
| Boni sur marge | 3 981,20 | — |
| Bonus Total | 7 823,22 | — |
| Ventes cumulées 12 mois (au 2026-03-24) | 1 827 444,15 | — |

L'écart de ~0,1 % avec les références du mémo est de la **dérive de données** (la
facturation évolue au fil des jours) — pas une erreur de formule. Les valeurs recalculées
sont cohérentes entre elles (commission = 0,5 % × cumul pour le palier 1 ; carte
`Ventes_Cumulees_YTD` = dernière ligne du tableau).

## 8. Suivi

- FYI possible à Julien : le v3 a retiré les colonnes bonus pré-calculées ; on a recalculé
  côté Power BI. À réévaluer s'il les réintroduit (aligner les noms ou retirer le DAX).
- Ce recalcul DAX est **la nouvelle source de vérité** de la commission côté rapport tant
  que le backend ne re-livre pas les colonnes.
