# 03 — Règles d'affaires confirmées

Toutes ces règles ont été **validées par Vincent** (business owner du programme).

- La commission se **remet à zéro chaque trimestre** (pas annuellement) → `QUARTER()` partout.
- Les projets à **marge négative sont exclus** des ventes cumulées **ET** de la commission.
- **MargeCible** : priorité à `BudgetExcelWeightedMargin`, fallback sur
  `ProjectBudgetMaestroMargin`.
- Commission calculée en **marginal** (pro-rata à travers les seuils), **pas** en taux plat.
- **Départage des projets de même date** : par ordre de numéro de contrat / `code_projet`.
- Seuls les **projets complétés** apparaissent dans les tableaux (le slicer de statut garde
  un `Blank` pour attraper les dossiers mal remplis).
- **Deux colonnes de cumul** :
  - **inclusive** — avec le projet courant
  - **exclusive** — sans le projet courant
