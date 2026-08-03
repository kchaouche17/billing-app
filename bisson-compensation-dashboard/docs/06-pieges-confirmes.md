# 06 — Pièges confirmés : NE JAMAIS REFAIRE

| Piège | Détail |
|---|---|
| **Colonnes calculées dans `Fact_Maestro depense projet`** | Crash systématique de Power BI Desktop, peu importe la formule. IMPOSSIBLE. Ne jamais reproposer. |
| **`FILTER(ALL('Fact...'))` dans une mesure** | `rsQueryMemoryLimitExceeded` (limite 1024 Mo) dans le Service. |
| **`MAX(vendeur_code)` dans `ADDCOLUMNS`** | Perd le contexte → tous les reps affichent le même montant. Utiliser `VALUES()`. |
| **`MAX()` en row context** | Recalcule sur tous les projets au lieu de sommer les lignes → `SUMX(VALUES(code_projet))`. |
| **`MAX()` sur colonne booléenne** | Erreur DAX → `COUNTROWS(FILTER(... = TRUE()))`. |
| **Changer de colonne de date en cours de route** | A coûté ~3 jours. `WorkCompletedOrEndDate` est LA date de référence. `WorkCompletedOn` est corrompu par les SAV de Jérémy. |
| **Locale `fr-CA`** | Met le `$` à droite → envelopper dans une mesure `SUM()` explicite, format Devise `$` à gauche. |
| **Card v2** | Section « Valeur » non expansible → affichage abrégé `K$` non désactivable. Limitation connue. |
| **`QuoteStyle=QuoteStyle.None` sur un CSV avec texte libre** | Une virgule dans un champ (adresse, nom client) décale les colonnes → `Failed to move the data reader to the next row` sur une ligne au-delà de l'aperçu. Utiliser `QuoteStyle=QuoteStyle.Csv`. |
| **Retirer une colonne du `Changed Type` pour faire taire une erreur** | Ça débloque le refresh mais **casse les mesures/visuels** qui en dépendent en aval. Vérifier ce qui utilise la colonne avant de la retirer. |
| **Confondre cumul commission et « ventes cumulées »** | Commission = cumul **trimestriel** (reset). « Ventes cumulées » = **rolling 12 mois** (peut baisser quand une vieille vente sort de la fenêtre — ce n'est pas une vente négative). |

## Enjeux réglés (solutions non documentées)

Ces enjeux ont été réglés mais **les solutions n'ont pas été documentées**. Si un bug de
même famille réapparaît (perte de contexte dans un `SUMX`, formatage de devise), **on
repart de zéro**.

| Enjeu | Statut |
|---|---|
| Formatage du signe `$` sur `Boni_Marge_Rep` | Réglé |
| `Commission_Rep` affichant `--` en mode multi-reps | Réglé |
| Écart entre les cartes KPI et les totaux des tableaux | Réglé |
| Corrections de qualité de donnée dans CRM Dynamics (projets 2025) | Réglé |

## Correctif documenté — colonne Power Query introuvable (2026-07-14)

**Symptôme :** au rafraîchissement, `Expression.Error : ... la colonne
« BonusSalesPersonMarginPercentage » de la table [wasn't found]` (Mashup ErrorCode 10224).

**Cause :** l'étape Power Query **« Changed Type »** (`Table.TransformColumnTypes`) de la
requête `Fact_Maestro depense projet` typait `{"BonusSalesPersonMarginPercentage",
Percentage.Type}`, mais cette colonne **n'existe pas dans le CSV `v3`** de Julien. Seule
`BonusSalesPersonMarginAmount` est livrée. On ne peut pas typer une colonne absente → plantage.

**Correctif appliqué :** retiré la paire `{"BonusSalesPersonMarginPercentage",
Percentage.Type}, ` de la liste du `Table.TransformColumnTypes`. Refresh OK.

**À retenir :** quand le backend change le schéma du CSV (renommage / suppression de
colonne), l'étape *Changed Type* garde l'ancien nom en dur et plante. Vérifier la liste de
typage après chaque nouvelle version du CSV. Si un vrai % de marge est requis plus tard :
soit mesure DAX à partir de `BonusSalesPersonMarginAmount`, soit demander à Julien de
livrer la colonne au backend — **jamais** en colonne calculée dans la Fact.
