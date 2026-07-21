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

> ⚠️ **Les 3 mesures PAR PROJET ci-dessous (`Commission_Projet`, `Ventes_Cumulees_Projet`,
> `Palier_Projet`) ont été RÉVISÉES le 2026-07-14 — voir [§9](#9-correction-mesures-par-projet--règle-complété-2026-07-14).**
> Les versions ci-dessous (cumul via propagation `Dim_Date`, rolling 12 mois, sans filtre
> « complété ») causaient des paliers erronés. **Utiliser les versions de la §9.** Les 3
> mesures par rep (`Commission_Rep`, `Bonus_total`, `Ventes_Cumulees_YTD`) restent valides.

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

## 9. Correction mesures par projet + règle « complété » (2026-07-14)

Après tests sur d'autres reps (Derick Pelletier = `vendeur_code 7`, Q2 2026, qui **dépasse
1 M$** dans le trimestre — le seul cas qui teste le cross-palier), deux bugs des mesures
**par projet** sont ressortis. Ils sont corrigés ici. **Ces versions remplacent celles de la §6.**

### Les deux bugs trouvés

1. **Cumuls incohérents entre colonnes.** `Ventes_Cumulees_Projet`, `Palier_Projet` et
   `Commission_Projet` utilisaient des cumuls différents (rolling 12 mois vs propagation
   `Dim_Date` vs total trimestre) → le palier basculait **avant** que la colonne « Ventes
   Cumulées » n'atteigne 1 M$. **Correctif :** les 3 partagent le **même `Base`**.
2. **Filtre « complété » manquant.** `REMOVEFILTERS('Fact...')` effaçait le filtre de page
   `ProductionStatus = "Complétés"` → le palier comptait des projets **non complétés** (ex.
   projet `1187-25` à 494 k chez Derick) et franchissait 1 M$ trop tôt. `IsProjectCompleted`
   ne suffit **pas** (basé sur la date) — c'est bien **`ProductionStatus = "Complétés"`** qui fait foi.

### Nouvelle règle d'affaires confirmée

> **La commission compte les projets COMPLÉTÉS seulement** (`ProductionStatus = "Complétés"`),
> en plus de `marge >= 0`. Un projet à marge négative **ou** non complété est exclu du cumul,
> du palier et de la commission — mais reste dans la colonne « Facturation » brute (d'où un
> écart normal *cumul < facturation*).

### Le `Base` commun (identique dans les 3 mesures)

```DAX
VAR Base =
    CALCULATETABLE(
        ADDCOLUMNS(
            SUMMARIZE( 'Fact_Maestro depense projet', 'Fact_Maestro depense projet'[code_projet], 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] ),
            "v", CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
        ),
        ALLSELECTED( 'Fact_Maestro depense projet' ),
        'Fact_Maestro depense projet'[marge] >= 0,
        'Fact_Maestro depense projet'[ProductionStatus] = "Complétés"
    )
```

`ALLSELECTED` garde la période + le rep sélectionnés (slicers) mais enlève la ligne courante.
`marge >= 0` et `ProductionStatus = "Complétés"` sont bakés → les 3 colonnes voient **exactement
le même ensemble de projets**, donc le palier bascule **pile** quand « Ventes Cumulées »
franchit 1 M$.

### `Ventes_Cumulees_Projet` (final)
```DAX
Ventes_Cumulees_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR Base = /* voir « Base commun » ci-dessus */
    RETURN SUMX( FILTER( Base, [WorkCompletedOrEndDate] < d || ( [WorkCompletedOrEndDate] = d && [code_projet] <= c ) ), [v] ),
    CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ), 'Fact_Maestro depense projet'[marge] >= 0, 'Fact_Maestro depense projet'[ProductionStatus] = "Complétés" )
)
```

### `Palier_Projet` (final)
```DAX
Palier_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR Base = /* voir « Base commun » ci-dessus */
    VAR incl = SUMX( FILTER( Base, [WorkCompletedOrEndDate] < d || ( [WorkCompletedOrEndDate] = d && [code_projet] <= c ) ), [v] )
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

### `Commission_Projet` (final)
```DAX
Commission_Projet =
IF(
    HASONEVALUE( 'Fact_Maestro depense projet'[code_projet] ),
    VAR d = SELECTEDVALUE( 'Fact_Maestro depense projet'[WorkCompletedOrEndDate] )
    VAR c = SELECTEDVALUE( 'Fact_Maestro depense projet'[code_projet] )
    VAR Base = /* voir « Base commun » ci-dessus */
    VAR incl = SUMX( FILTER( Base, [WorkCompletedOrEndDate] < d || ( [WorkCompletedOrEndDate] = d && [code_projet] <= c ) ), [v] )
    VAR excl = SUMX( FILTER( Base, [WorkCompletedOrEndDate] < d || ( [WorkCompletedOrEndDate] = d && [code_projet] <  c ) ), [v] )
    RETURN
          ( 0.005*MIN(incl,1000000) + 0.010*MAX(0,MIN(incl,2000000)-1000000) + 0.015*MAX(0,MIN(incl,3000000)-2000000) + 0.020*MAX(0,MIN(incl,4000000)-3000000) + 0.025*MAX(0,MIN(incl,5000000)-4000000) + 0.030*MAX(0,incl-5000000) )
        - ( 0.005*MIN(excl,1000000) + 0.010*MAX(0,MIN(excl,2000000)-1000000) + 0.015*MAX(0,MIN(excl,3000000)-2000000) + 0.020*MAX(0,MIN(excl,4000000)-3000000) + 0.025*MAX(0,MIN(excl,5000000)-4000000) + 0.030*MAX(0,excl-5000000) ),
    [Commission_Rep]
)
```

### Le calcul marginal expliqué

Chaque tranche de ventes est taxée à son taux (0-1 M = 0,5 % ; 1-2 M = 1 % ; 2-3 M = 1,5 % ;
3-4 M = 2 % ; 4-5 M = 2,5 % ; 5 M+ = 3 %). La fonction cumulative :

```
marginal(S) = 0,005×MIN(S,1M) + 0,010×MAX(0,MIN(S,2M)−1M) + 0,015×MAX(0,MIN(S,3M)−2M)
            + 0,020×MAX(0,MIN(S,4M)−3M) + 0,025×MAX(0,MIN(S,5M)−4M) + 0,030×MAX(0,S−5M)
```

Commission d'un projet = `marginal(cumul_inclusif) − marginal(cumul_exclusif)` → isole la
tranche apportée par ce projet, au(x) bon(s) taux.

**Exemple — projet pivot `1429-26` (Derick), exclusif 995 997,89 → inclusif 1 027 997,89 :**
```
marginal(1 027 997,89) = 0,005×1 000 000 + 0,010×27 997,89 = 5 000 + 279,98 = 5 279,98
marginal(  995 997,89) = 0,005×995 997,89                  = 4 979,99
commission             = 5 279,98 − 4 979,99               =   299,99 $
```
Vérif « coupé au million » : (1 000 000−995 997,89)×0,5 % + (1 027 997,89−1 000 000)×1 % =
20,01 + 279,98 = **299,99 $**. ✓

Les commissions par projet **se télescopent** : `Σ = marginal(cumul_final)`. Pour Derick,
`marginal(1 339 405,59) = 5 000 + 0,010×339 405,59 = 8 394,06 $` = le Total Commission de sa table. ✓

### Validation (Derick, rep 7, Q2 2026)

Cumul complété + marge ≥ 0 : …873 547,89 (**Palier 1**) → 941 497,89 (**Palier 1**) →
995 997,89 (**Palier 1**) → **1 027 997,89 (Palier 2)**. Le switch tombe **pile** au premier
projet ≥ 1 M$, plus à 941 k. Sans le filtre complété, le projet `1187-25` (494 k, non complété)
faisait basculer le palier ~2 projets trop tôt.
