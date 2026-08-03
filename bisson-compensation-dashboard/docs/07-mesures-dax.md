# 07 — Mesures DAX existantes

> ⚠️ **Mise à jour 2026-07-14 :** plusieurs mesures ont été **reconstruites en DAX** après
> le retrait des colonnes bonus du CSV v3. Le code complet et validé est dans
> [docs/10](10-incident-colonnes-bonus-2026-07.md). Ne pas repartir des anciennes
> définitions (qui pointaient sur `BonusSalesPersonCumulativeAmount`, etc.).

## Mesures

- `Ventes_Cumulees_YTD` — **reconstruite** : rolling 12 mois (voir docs/10)
- `Commission_Rep` — **reconstruite** : marginale par paliers, auto-totalisante (voir docs/10)
- `Bonus_total` — **reconstruite** : `[Boni_Marge_Rep] + [Commission_Rep]`
- `Commission_Projet` — **nouvelle** : commission marginale par projet (voir docs/10)
- `Ventes_Cumulees_Projet` — **nouvelle** : rolling 12 mois par projet (voir docs/10)
- `Palier_Projet` — **nouvelle** : palier du cumul trimestriel par projet (voir docs/10)
- `Commission_Total`
- `Boni_Marge_Rep` — DAX pur, jamais dépendant des colonnes retirées
- `Palier_Mode` / `Palier_Actuel`
- `Ecart_Cible_Reelle (%)`
- `Ecart_Cible_Reelle_Dollar ($)`
- `Multiplicateur_Ecart`
- `Taux_Boni_Moyen_Pondere`
- `Moyenne_Ecart_Cible`

## Colonnes calculées

- `Veille_Statut_Col` — colonne calculée pour le slicer de statut.

## Formatage visuel en place

- Coloration conditionnelle **verte / rouge** sur la marge réelle vs cible.
- Échelle dégradée (**rouge → jaune-orange → vert foncé**) sur la colonne d'écart en %.
