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
