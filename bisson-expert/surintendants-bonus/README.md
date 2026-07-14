# Dashboard Bonification Surintendants — Bisson Expert en Fondation

> Document de référence / mémoire du projet. Versionné dans le repo pour garder une
> trace persistante du contexte, des décisions validées et du backlog.
> Toute amélioration future part de ce document.
> Dernière mise à jour : 14 juillet 2026

---

## 1. Contexte et objectif

- **Entreprise :** Bisson Expert en Fondation
- **Utilisateur :** Karim Chaouche (kchaouche)
- **Fichier Power BI :** `Sommaire Bonification Surintendants.pbix`
- **Dossier OneDrive :** `...IT Backlog\Projects\New Bonus Structure\`
- **Outils :** Power BI Desktop + Power BI Service, DAX Studio, Power Query (M)
- **Objectif :** Dashboard Power BI de suivi de la bonification des **surintendants**
  (chargés de projet), affiché sur un **écran dédié** en salle. Style visuel inspiré
  des **cartes de joueurs NHL** (gamification).

Ce projet est le « petit frère » d'un dashboard existant pour les **représentants
(reps)** — commission + boni marge. Beaucoup de leçons viennent de là (voir §8).

---

## 2. Les 6 surintendants actifs

| Nom | # employé | Dynamics ID | Fichier avatar |
|---|---|---|---|
| Fatma Bououni | 26 | `100000026` | `1778526725048_image.png` |
| Richard St-Jean | 05 | `100000005` | `__2_.png` |
| Ramzi Hosni | 28 | `100000028` | `__3_.png` |
| Gabriel Primeau-Dufresne | 32 | `100000032` | `__4_.png` |
| Justin Tremblay-Houde | 30 | `100000030` | `__5_.png` |
| Étienne Desmeules | 29 | `100000029` | `__7_.png` |

- ⚠️ **Roger Doucet a été retiré** — il ne travaille plus chez Bisson.
- ⚠️ `1778519268785_image.png` = la **vraie photo** de Fatma, **PAS** son avatar. Ne pas confondre.
- Tous les avatars partagent le préfixe `ChatGPT_Image_May_11__2026__03_20` (générés
  par ChatGPT, style cartoon NHL, chandail rouge/noir Bisson).
- Emplacement : `C:\Users\kchaouche\OneDrive - BissonExpert\IT EXTERNE _ BISSON EXPERT - IT Backlog\Projects\New Bonus Structure\Photos Sur Intendants`
- ❌ **Pas accessibles par URL réseau** → doivent être embarqués en **base64** dans le HTML.
- ❌ Les avatars SVG ont été tentés puis **abandonnés** (qualité insuffisante). Ne pas y revenir.

---

## 3. Modèle de données

### Table de faits : `Fact_Maestro depense projet`

| Colonne | Rôle |
|---|---|
| `marge` | Marge **réelle** du projet |
| `BudgetExcelWeightedMargin` | Marge **cible — source prioritaire** |
| `ProjectBudgetMaestroMargin` | Marge cible — **fallback** si Excel est BLANK |
| `total_facture_sans_taxe` | Ventes (base de calcul du boni) |
| `code_projet` | Identifiant projet (ex. `1065-25`) |
| `ActiveProjectManagerDynamicsId` | Code surintendant |
| `WorkCompletedOrEndDate` | **Date de référence — colonne de la relation avec Dim_Date** |
| `IsProjectCompleted` | Filtre projets complétés |
| `ProductionStatus` | Statut de production |

**Colonnes de coûts (réel vs budget) :**
- `cout_reel_mo` / `cout_budget_mo`
- `cout_reel_materiel` / `cout_budget_materiel`
- `cout_reel_sous_traitant` / `cout_budget_sous_traitant`
- `cout_reel_autre_total` / `cout_budget_autre_total`

### Table de dimension : `Dim_Charges de projet`

Colonnes : `nom`, `dynamics id`, `actif`.
Liée à la table de faits via `dynamics id` ↔ `ActiveProjectManagerDynamicsId`.

### Table de paliers : `Bonification_Surintendants`

Créée en DAX via `DATATABLE`. 35 lignes, de **−7 % à +10 %** par incréments de **0,5 %**.
Colonnes : `Ecart_Cible`, `Taux_Boni`.

Source : `Courbe_bonification_surintendant__05072026.xlsx` — **colonne D = « Taux de
base surintendant (New) »** (⚠️ pas la colonne C qui est l'ancien taux CP).

Valeurs exactes (`Ecart_Cible` → `Taux_Boni`) :

```
-0.070 → 0.0006104651   -0.005 → 0.0067151163   0.040 → 0.0137790698
-0.065 → 0.0008720930    0.000 → 0.0075000000   0.045 → 0.0144767442
-0.060 → 0.0011337209    0.005 → 0.0081976744   0.050 → 0.0151744186
-0.055 → 0.0014825581    0.010 → 0.0089825581   0.055 → 0.0157848837
-0.050 → 0.0018313953    0.015 → 0.0097674419   0.060 → 0.0163953488
-0.045 → 0.0021802326    0.020 → 0.0105523256   0.065 → 0.0169186047
-0.040 → 0.0026162791    0.025 → 0.0113372093   0.070 → 0.0174418605
-0.035 → 0.0030523256    0.030 → 0.0122093023   0.075 → 0.0178779070
-0.030 → 0.0036627907    0.035 → 0.0130813953   0.080 → 0.0183139535
-0.025 → 0.0041860465                            0.085 → 0.0186627907
-0.020 → 0.0047965116                            0.090 → 0.0189244186
-0.015 → 0.0053197674                            0.095 → 0.0191860465
-0.010 → 0.0060174419                            0.100 → 0.0193604651
```

- Taux à la cible (écart 0 %) : **0,75 %**
- Taux plafond (écart ≥ +10 %) : **1,936 %** (`0.01936046511627907`)

---

## 4. Logique métier du boni — ⚠️ VALIDÉE, NE PAS RÉINVENTER

```
Écart       = MargeR − MargeC        ← EN POINTS DE %, PAS divisé par la cible
Boni projet = Taux(Écart) × Ventes du projet
Boni surint = Σ des bonis de ses projets
```

**Règles :**

| Condition | Résultat |
|---|---|
| `MargeC` est BLANK **ou** = 0 | Boni = **BLANK** |
| Écart < −7 % | Boni = **0 $** |
| −7 % ≤ Écart ≤ +10 % | Lookup du taux dans `Bonification_Surintendants` |
| Écart > +10 % | Taux plafond **1,936 %** |

**Arrondi obligatoire :** `ROUND( EcartCap / 0.005, 0 ) * 0.005`
→ ⚠️ **`MROUND` ne fonctionne PAS correctement** dans ce contexte. Ne pas l'utiliser.

> 🚨 **CONTRADICTION À CONNAÎTRE** — Le vieux « guide dashboard surintendants » (hérité
> du projet reps) dit que l'écart = `(MargeR − MargeC) / MargeC` et que le boni
> s'applique sur les **ventes cumulées**. **C'EST FAUX pour les surintendants.** La
> logique ci-dessus (points de %, ventes **par projet**) a été validée en DAX Studio et
> confirmée par Karim. Si le guide et ce document se contredisent, **ce document gagne**.

**Validations manuelles réussies (Q1 2025) :**

| Projet | Marge R | Marge C | Écart | Boni |
|---|---|---|---|---|
| 1065-25 | 40,10 % | 37,00 % | +3,10 % | ~950 $ |
| 1069-25 | 42,68 % | 39,21 % | +3,47 % | ~650 $ |
| 1004-25 | 50,50 % | 48,00 % | +2,50 % | ~209 $ |
| 0986-25 | 25,17 % | 47,28 % | −22,11 % | 0 $ |

**Totaux par surintendant (Q1 2025, `IsProjectCompleted = TRUE`) :**

| Dynamics ID | Projets | Ventes | Boni |
|---|---|---|---|
| 100000004 | 15 | 367 574,75 $ | 4 530,70 $ |
| 100000026 | 14 | 293 736,00 $ | 2 591,49 $ |
| 100000005 | 10 | 518 353,53 $ | 948,14 $ |
| 100000024 | 2 | 254 753,60 $ | 0 $ |

---

## 5. Mesures DAX validées (en production)

### `Boni_Surint_Projet` — par projet (colonne de tableau)

```dax
Boni_Surint_Projet =
VAR Ventes    = CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
VAR MargeR    = CALCULATE( MAX( 'Fact_Maestro depense projet'[marge] ) )
VAR Excel     = CALCULATE( MAX( 'Fact_Maestro depense projet'[BudgetExcelWeightedMargin] ) )
VAR Maestro   = CALCULATE( MAX( 'Fact_Maestro depense projet'[ProjectBudgetMaestroMargin] ) )
VAR MargeC    = IF( NOT ISBLANK( Excel ), Excel, Maestro )
VAR EcartBrut = IF( NOT ISBLANK( MargeC ) && MargeC <> 0, MargeR - MargeC, BLANK() )
VAR EcartCap  = MIN( 0.10, EcartBrut )
VAR EcartArrondi = ROUND( EcartCap / 0.005, 0 ) * 0.005
VAR Taux =
    IF( ISBLANK( EcartBrut ), BLANK(),
    IF( EcartBrut < -0.07, 0,
        CALCULATE(
            MAX( Bonification_Surintendants[Taux_Boni] ),
            Bonification_Surintendants[Ecart_Cible] = EcartArrondi
        )
    ))
RETURN IF( NOT ISBLANK( Taux ), Taux * Ventes, BLANK() )
```

### `Boni_Surint_Total` — wrapper pour cartes KPI et HTML

```dax
Boni_Surint_Total =
SUMX(
    VALUES( 'Dim_Charges de projet'[dynamics id] ),
    SUMX(
        VALUES( 'Fact_Maestro depense projet'[code_projet] ),
        VAR Ventes    = CALCULATE( SUM( 'Fact_Maestro depense projet'[total_facture_sans_taxe] ) )
        VAR MargeR    = CALCULATE( MAX( 'Fact_Maestro depense projet'[marge] ) )
        VAR Excel     = CALCULATE( MAX( 'Fact_Maestro depense projet'[BudgetExcelWeightedMargin] ) )
        VAR Maestro   = CALCULATE( MAX( 'Fact_Maestro depense projet'[ProjectBudgetMaestroMargin] ) )
        VAR MargeC    = IF( NOT ISBLANK( Excel ), Excel, Maestro )
        VAR EcartBrut = IF( NOT ISBLANK( MargeC ) && MargeC <> 0, MargeR - MargeC, BLANK() )
        VAR EcartCap  = MIN( 0.10, EcartBrut )
        VAR EcartArrondi = ROUND( EcartCap / 0.005, 0 ) * 0.005
        VAR Taux =
            IF( ISBLANK( EcartBrut ), BLANK(),
            IF( EcartBrut < -0.07, 0,
                CALCULATE(
                    MAX( Bonification_Surintendants[Taux_Boni] ),
                    Bonification_Surintendants[Ecart_Cible] = EcartArrondi
                )
            ))
        RETURN IF( NOT ISBLANK( Taux ), Taux * Ventes, 0 )
    )
)
```

### Autres mesures existantes

- `Taux_Boni_Surint` — même logique, retourne le **taux** (formaté en % 2 décimales)
- `Ecart_Cible_Surint_$` — écart en dollars
- `Ecart_Cible_Surint` — écart brut avant cap (`MargeR − MargeC`, formaté en %)

---

## 6. Visuel HTML — état actuel et bug ouvert

### Architecture

```
HTML_Template (table Power Query, 3 chunks de ~28 000 chars)
        ↓
Mesure DAX  HTML_Dashboard_Surintendants
   = Chunk1 & Chunk2 & [JSON live] & Chunk3
        ↓
Visual "HTML Content" (Daniel Marsh-Patrick, v1.6.0)
```

- Le visual rend **n'importe quelle string HTML complète** retournée par une mesure.
- ✅ **JavaScript fonctionne** dans le visual (testé et confirmé).
- Colonnes de `HTML_Template` : `Ordre` (Int64), `Contenu` (Text).
- Placeholder `__DATA__` remplacé par le JSON injecté.
- Limite Power BI : **32 766 chars par cellule** → chunks de 28 000 max.

### Layout du dashboard

- **Panneau gauche (38 %)** : classement des 6 surintendants — rang, # employé, nom,
  boni réel, potentiel, laissé sur la table.
- **Panneau droit** : fiche joueur qui **défile automatiquement toutes les 12 secondes**
  — avatar, KPIs (boni réel, laissé sur la table, efficacité), tableau des **4 paliers**
  de boni avec indicateur **« vous êtes ici »**.
- Palette : noir `#1a1a1a`, rouge Bisson `#C41230`, blanc. Police Oswald.
- Potentiel max = `Ventes × 0.01936`.

### Mesure DAX actuelle

```dax
HTML_Dashboard_Surintendants =
VAR _MaxTaux = 0.01936
VAR _IdsActifs = {100000026, 100000005, 100000028, 100000032, 100000030, 100000029}
VAR _Surint =
    CALCULATETABLE(
        ADDCOLUMNS(
            FILTER( 'Dim_Charges de projet',
                    'Dim_Charges de projet'[dynamics id] IN _IdsActifs ),
            "@Boni", [Boni_Surint_Total],
            "@Max",  CALCULATE(
                        SUMX(
                            VALUES('Fact_Maestro depense projet'[code_projet]),
                            CALCULATE(SUM('Fact_Maestro depense projet'[total_facture_sans_taxe])) * _MaxTaux
                        )
                     )
        ),
        'Fact_Maestro depense projet'[IsProjectCompleted] = TRUE()
    )
VAR _JSON =
    "[" &
    CONCATENATEX(
        _Surint,
        "{""key"":""" & 'Dim_Charges de projet'[nom] &
        """,""boni"":" & FORMAT([@Boni], "0") &
        ",""max"":"  & FORMAT([@Max],  "0") & "}",
        ","
    ) & "]"
VAR _B1 = CONCATENATEX(FILTER(HTML_Template, HTML_Template[Ordre]=1), HTML_Template[Contenu], "")
VAR _B2 = CONCATENATEX(FILTER(HTML_Template, HTML_Template[Ordre]=2), HTML_Template[Contenu], "")
VAR _A1 = CONCATENATEX(FILTER(HTML_Template, HTML_Template[Ordre]=3), HTML_Template[Contenu], "")
RETURN _B1 & _B2 & _JSON & _A1
```

### 🐛 Bug ouvert #1 — le seul blocage restant

**Symptôme :** le layout s'affiche (header, colonnes, sections) mais **les données sont vides**.

**Cause :** DAX **double les guillemets** dans la string de sortie → le JSON arrive
comme `""key""` au lieu de `"key"` → `JSON.parse` échoue **silencieusement** en JS.

**Fix préparé (PAS ENCORE APPLIQUÉ dans Power BI) :** dans le JS du template HTML,
remplacer le parse direct par :

```js
var DATA = (function(){
  var raw = '__DATA__';
  try { return JSON.parse(raw); }
  catch(e) { return JSON.parse(raw.replace(/""/g, '"')); }
})();
```

**Action :** Power BI → Transformer les données → `HTML_Template` → Éditeur avancé →
coller le nouveau script M contenant ce fix → Fermer et appliquer.
**La mesure DAX ne change pas.**

### 🐛 Bug ouvert #2 — Colonne `IsPostBonusReferenceDate` introuvable (14 juillet 2026)

**Symptôme :** plusieurs visuels affichent *« Désolé, erreur lors de la récupération
des données pour cet élément visuel »*. **Différent du Bug #1** (qui, lui, affiche le
layout mais vide).

**Erreur exacte :**
```
QueryUserError — Erreur OLE DB ou ODBC: Query (18, 46)
La colonne 'IsPostBonusReferenceDate' de la table 'Fact_Maestro depense projet'
est introuvable ou ne peut pas être utilisée dans cette expression.
```

**Cause :** une ou plusieurs mesures filtrent sur `Fact_Maestro depense projet[IsPostBonusReferenceDate]`,
mais **la colonne n'existe plus** (renommée ou supprimée). Cette colonne **n'est pas
dans le modèle documenté au §3** → ajoutée après coup. Probablement retirée lors d'un
`Fermer et appliquer` de Power Query pendant les travaux sur la source des **6–7 juillet 2026**
(voir logs de facturation : « Intégration WorkedDays de Julien », « Ajustements et Tests Power BI »).

**Diagnostic (DAX Studio) — localiser les mesures fautives :**
```dax
EVALUATE
FILTER( INFO.VIEW.MEASURES(), SEARCH( "IsPostBonusReferenceDate", [Expression], 1, 0 ) > 0 )
```
Confirmer que la colonne a disparu :
```dax
EVALUATE
FILTER( INFO.VIEW.COLUMNS(), SEARCH( "PostBonus", [Column], 1, 0 ) > 0 )
```

**CAUSE RACINE CONFIRMÉE (14 juillet 2026) — dérive de schéma de la source :**
`Fact_Maestro depense projet` est en **DirectQuery**, alimentée par le backend C# de
**Julien** (source `maestroProjectSaleSummaries_v3.csv` / Azure Blob). Le **6–7 juillet**,
un **nouveau schéma `v3`** a **supprimé/renommé plusieurs colonnes**. Ce n'est PAS un
filtre isolé : c'est une dérive de schéma côté source.

> **Cross-référence :** le dashboard **reps** a subi la même classe de bug le même jour.
> Voir branche `claude/bisson-compensation-dashboard-xiu8mz`,
> `docs/06-pieges-confirmes.md` → « Correctif documenté — colonne Power Query introuvable
> (2026-07-14) » : la colonne `BonusSalesPersonMarginPercentage` a disparu du CSV `v3`.
> Principe retenu : **à chaque nouvelle version du CSV, une référence en dur casse.**

**Colonnes disparues qui cassaient les visuels surintendants** (toutes des références
**orphelines dans la couche RAPPORT** — filtres/champs, PAS des mesures, d'où les
requêtes de dépendance vides) :

| Colonne disparue | Où elle était référencée |
|---|---|
| `IsPostBonusReferenceDate` | Filtre « sur cette page », `= True` |
| `is_probably_incomplete` | Filtre « sur toutes les pages », `= False ou True` |
| `heures_budget_main_doeuvre` | Champ/filtre de visuel |
| *(possiblement d'autres au fil du nettoyage)* | Visuels tableau / matrice |

**✅ Impact sur le boni : NUL.** Aucune de ces colonnes n'appartient à la logique validée
(§4/§5). Le doc `02-modele-donnees.md` de l'autre branche confirme que **toutes les
colonnes cœur du boni ont survécu au v3** (`total_facture_sans_taxe`, `marge`,
`BudgetExcelWeightedMargin`, `ProjectBudgetMaestroMargin`, `WorkCompletedOrEndDate`,
`IsProjectCompleted`, `ActiveProjectManagerDynamicsId`, `code_projet`).

**Résolution :**
1. **Retirer les cartes de filtre orphelines** (niveau visuel / page / rapport) qui
   pointent vers les colonnes disparues. Sans risque pour le calcul du boni.
2. **Retirer les champs disparus** des visuels tableau/matrice qui les affichaient encore.
3. **Réimplémenter l'exclusion « post-date de référence »** (rôle de `IsPostBonusReferenceDate`)
   via un filtre de date sur la colonne survivante :
   `'Fact_Maestro depense projet'[WorkCompletedOrEndDate] >= DATE(aaaa, mm, jj)`.
   → Date de démarrage du programme **À CONFIRMER par Karim**.

**Décision d'architecture (14 juillet 2026) — Option A retenue par Karim :**
On **garde la connexion DirectQuery au modèle sémantique partagé** — pas de bascule en
Import, pas de reconstruction de la couche données. Le fichier
`Sommaire Bonification Surintendants.pbix` est un **rapport mince** en DirectQuery/live sur
ce modèle. Un **backup** du fichier a été fait avant intervention.

**Source de vérité confirmée :** modèle sémantique **« Données seulement »** (propriétaire
**Julien Bergeron**), et non « Job Cost – Sommaire Bonification Reps - V2 » (qui est un
autre modèle du même espace de travail). La chaîne de connexion se lit
`DirectQuery à AS – Modèle sémantique Power BI;Données seulement`. Le fait
`Fact_Maestro depense projet` est présent et identique dans tous les modèles (tous
alimentés par le backend de Julien).

**✅ LARGEMENT RÉSOLU (14 juillet 2026) — par re-synchronisation de la connexion.**
Correction : mon hypothèse « reconnecter ne ramènera pas les colonnes » était **fausse**.
La cause réelle était des **métadonnées DirectQuery périmées en cache local** (le `.pbix`
gardait l'ancien schéma). **Fix appliqué :** `Transformer les données → Paramètres de la
source de données → Changer la source… → re-sélectionner le modèle « Données seulement » →
Se connecter`. Le re-sync a nettoyé les références périmées : la majorité des visuels se
rendent à nouveau. **Restants :** quelques visuels (le tableau du bas + ceux du côté droit)
gardent une référence cassée **au niveau du visuel** (filtre ou champ) que le re-sync
n'atteint pas → à nettoyer un par un sur chaque visuel concerné (bouton **« Corriger »**
natif de Power BI = retrait automatique du filtre cassé du visuel).

### 🐛 Bug ouvert #3 — `Boni_Surint_Total` corrompu : `/ MargeC` au lieu de points de % (14 juillet 2026)

**Symptôme :** en inspectant les mesures en prod, la version de `Boni_Surint_Total`
**réellement dans le `.pbix`** calcule l'écart en **relatif** :
```dax
VAR EcartBrut = IF( NOT ISBLANK( MargeC ) && MargeC <> 0, ( MargeR - MargeC ) / MargeC, BLANK() )
```
alors que **toute la logique validée** (handoff §4, mesures de référence, écart affiché
`Ecart_Cible_Surint_$`, validations manuelles) utilise les **points de %** :
```dax
VAR EcartBrut = IF( NOT ISBLANK( MargeC ) && MargeC <> 0, MargeR - MargeC, BLANK() )
```

**Impact :** le `/ MargeC` **gonfle l'écart** → surpaie les projets à écart modéré.
Exemple validé **1065-25** (MargeR 40,10 %, MargeC 37,00 %) : points → écart +3,10 %,
boni ≈ 950 $ (montant validé) ; relatif → écart +8,38 %, boni ≈ 1 452 $ (**+53 %**).
Le bug était **invisible** jusqu'ici parce que tous les projets affichés avaient des
écarts extrêmes (> +10 % ou < −7 %) → plafond/plancher identiques dans les deux versions.
Incohérence interne aussi : le tableau **affiche** un écart en points mais **calcule** le
boni sur un écart relatif.

**Fix :** retirer le `/ MargeC` dans `Boni_Surint_Total` → revenir à `MargeR - MargeC`.
**À vérifier ensuite :** que `Boni_Surint_Projet` et `Taux_Boni_Surint` **dans le fichier**
ne soient pas corrompus de la même façon (les versions de référence, elles, sont en points).

**Statut :** correction communiquée à Karim, en cours d'application.

**Validation visuelle après fix :** le tableau projet affiche correctement les écarts et
bonis — plafond à **1,94 %** pour les écarts > +10 %, **0 %** pour le projet `0886-24`
(écart −13,17 %, règle du plancher −7 %). Cohérent avec la logique validée (§4).

**À retenir pour la prochaine fois :** quand Julien republie/actualise le modèle
sémantique et que des visuels tombent en « colonne introuvable » (erreur OLE DB/ODBC),
le premier réflexe est **re-synchroniser la connexion** (Changer la source → re-sélection
du même modèle), PAS supprimer des filtres. Le retrait du filtre de page
`IsPostBonusReferenceDate` avant le re-sync était donc **inutile** — à réévaluer si ce
filtre servait une vraie logique métier (exclusion des projets pré-programme).

---

## 7. À faire (backlog)

0. **[BLOQUANT] Réparer la référence `IsPostBonusReferenceDate`** (Bug #2) —
   les visuels plantent tant que ce n'est pas réglé.
1. **[PRIORITÉ] Appliquer le fix `.replace(/""/g, '"')`** dans `HTML_Template` →
   débloque l'affichage des données.
2. **Visuel « tier reward »** — UI de progression style jeu mobile : paliers de boni
   potentiel + $ laissé sur la table.
3. **Colonne « Facteur principal d'écart »** — identifie quelle catégorie de coût
   (MO, matériel, sous-traitant, autre) explique le plus gros écart vs budget, affichée
   en **$ et en %**. Utiliser les paires `cout_reel_*` / `cout_budget_*`.

---

## 8. Contraintes techniques dures — NE PAS TESTER À NOUVEAU

| Contrainte | Détail |
|---|---|
| ❌ **Colonnes calculées dans `Fact_Maestro depense projet`** | Power BI Desktop **crash systématiquement**, même avec `= 1`. Tout doit se faire en **mesures**. |
| ❌ **`MROUND`** | Ne fonctionne pas pour l'arrondi des paliers. Utiliser `ROUND(x/0.005,0)*0.005`. |
| ❌ **Base64 direct dans une mesure DAX** | Les string literals en `VAR` plafonnent vers **4 000–8 000 chars**. D'où les chunks Power Query. |
| ❌ **Avatars SVG** | Tentés, abandonnés. PNG cartoon ChatGPT seulement. |
| ⚠️ **`FILTER(ALL(Fact), ...)` dans une mesure** | Table 10 000+ lignes × 50 lignes de tableau = crash mémoire `rsQueryMemoryLimitExceeded`. Éviter. |
| ⚠️ **Total d'un tableau en contexte multi-surintendant** | Peut afficher `--`. **Normal et accepté.** Utiliser une carte KPI avec `Boni_Surint_Total` pour le total. |
| ⚠️ **Colonne de date** | `WorkCompletedOrEndDate` est **LA** colonne de la relation `Dim_Date`. **Ne jamais en changer** — 3 jours perdus là-dessus sur le projet reps. |

---

## 9. Workflow de travail obligatoire

1. **Toujours prototyper dans DAX Studio d'abord.** Jamais écrire une mesure
   directement dans Power BI Desktop.

   ```dax
   EVALUATE
   CALCULATETABLE(
       ADDCOLUMNS(
           VALUES( 'Fact_Maestro depense projet'[ActiveProjectManagerDynamicsId] ),
           "@Boni", [logique à tester]
       ),
       'Dim_Date'[Year] = 2026,
       'Dim_Date'[Quarter] = 1,
       'Fact_Maestro depense projet'[IsProjectCompleted] = TRUE()
   )
   ORDER BY [@Boni] DESC
   ```

2. Valider les totaux et 2–3 projets individuels à la main.
3. **Seulement ensuite** créer la mesure dans Power BI.
4. Si tu modifies la même mesure 5 fois sans progrès → le problème est **structurel**,
   pas syntaxique. Arrête et remonte à l'architecture.

---

## 10. Préférences de travail de Karim

- **Lire le contexte au complet AVANT de poser des questions.** Ne jamais reposer une
  question déjà répondue. C'est le principal irritant.
- Communication en **français**, direct, sans préambule inutile.
- Ne pas proposer de solutions qui impliquent des **saisies manuelles récurrentes** —
  tout doit être dynamique et branché sur les données live.
- Utiliser la **dernière version** de chaque script/mesure. Jamais une version antérieure.
- Approche **validate-first** : DAX Studio → validation → Power BI.
