# 03 — Règles d'affaires confirmées

Toutes ces règles ont été **validées par Vincent** (business owner du programme).

- La commission se **remet à zéro chaque trimestre** (pas annuellement) → `QUARTER()` partout.
- Les projets à **marge négative sont exclus** des ventes cumulées **ET** de la commission.
  Le seuil est **zéro** (`marge >= 0` → compté ; `marge < 0` → exclu) — c'est « à perte »,
  **pas** « sous la marge cible ». Un projet à +2 % compte ; à −1 % non.
- **Commission = projets COMPLÉTÉS seulement** (`ProductionStatus = "Complétés"`), en plus de
  `marge >= 0`. Un projet non complété est exclu du cumul / palier / commission, mais reste
  dans la facturation brute (d'où *cumul < facturation* possible). Confirmé 2026-07-14 —
  `IsProjectCompleted` (basé sur la date) ne fait **pas** foi, c'est `ProductionStatus`.
- **MargeCible** : priorité à `BudgetExcelWeightedMargin`, fallback sur
  `ProjectBudgetMaestroMargin`.
- Commission calculée en **marginal** (pro-rata à travers les seuils), **pas** en taux plat.
- **Départage des projets de même date** : par ordre de numéro de contrat / `code_projet`.
- Seuls les **projets complétés** apparaissent dans les tableaux (le slicer de statut garde
  un `Blank` pour attraper les dossiers mal remplis).
- **Deux colonnes de cumul** :
  - **inclusive** — avec le projet courant
  - **exclusive** — sans le projet courant
