# Journal d'avancement

Suivi vivant de l'exécution. **Aucune donnée employé réelle ici** (pas de NAS, pas de noms) —
uniquement méthode, décisions et compteurs. Les fichiers de travail (Maestro, UKG, crosswalk,
import) restent **en local chez Karim, hors dépôt**.

Dernière session : **2026-07-29**

---

## État par phase

| Phase | État | Détail |
|---|---|---|
| 0 — Cadrage | ✅ Fait | Décisions arrêtées (voir README §2). |
| 1 — Exploration | ✅ Fait | Clé = NAS ; Maestro ID → `ID externe` ; périmètre actifs+inactifs. |
| 2 — Extraction Maestro | ✅ Fait | Export réduit aux colonnes utiles (Numéro, Nom, Prénom, NAS, État). ~552 lignes. |
| 3 — Crosswalk | 🟡 Presque fait | 107/114 employés UKG appariés (voir ci-dessous). 7 exceptions à traiter par nom. |
| 4 — Valider `ID externe` | ⬜ À faire | Confirmer que le champ est libre (Q10). |
| 5 — Import Maestro ID | ⬜ À faire | Après le crosswalk + validation `ID externe`. |
| 6 — Écraser les noms | 🟡 En cours — **bloqué** | Fichier prêt ; import UKG en erreur « Missing header record » (voir blocage). |
| 7 — Validation / dashboards | ⬜ À faire | |

---

## Ce qui a été fait (Phase 3 — crosswalk)

Construction **locale** dans Excel (le NAS n'est jamais sorti du poste de Karim) :

1. **Deux onglets** dans un classeur : `Maestro` et `UKG`.
2. **Normalisation du NAS** des deux côtés (colonne `NAS_norm`) — retire les espaces et force
   9 chiffres pour aligner les formats et rétablir les zéros de tête :
   ```
   =IF(E2="";"";TEXT(SUBSTITUTE(E2;" ";"")*1;"000000000"))
   ```
3. **Join** via `XLOOKUP` sur le `NAS_norm`, côté UKG, pour ramener le numéro Maestro :
   ```
   =IFERROR(XLOOKUP(G2;Maestro!$G:$G;Maestro!$B:$B);"")
   ```
4. **Colonne de contrôle** (`COUNTIF`) pour repérer les cas 0 / 1 / doublons.

### Résultats (compteurs)
- Maestro : **~552 lignes** (tout l'historique, beaucoup d'inactifs sur 4-5 ans).
- UKG : **114 employés** (actifs / récemment inactifs seulement).
- **107 appariés sur 114 (94 %)** via le NAS.
- **7 exceptions** UKG sans match (NAS manquant côté UKG — « quelques trous ») → à apparier
  **par nom, à la main**. *(Reporté, à faire plus tard.)*
- **Aucun doublon de NAS** dans UKG (pas de fiche en double sur cette clé).
- Un compte générique **« Manager » (ID 9001)** a été retiré du périmètre.

### Comparaison des noms (sur les 107 appariés)
Formule `EXACT` (sensible casse + accents) entre nom UKG actuel et nom Maestro :
- **91 identiques**, **16 à changer** (accents, traits d'union, prénom complet…).
- Décision : **écraser les 107** (les 91 identiques = sans effet), noms **Maestro** = source de vérité.
- **Fichier d'import des noms préparé** (fichier séparé, originaux intacts) : 3 colonnes
  `ukg_employee_id`, `prenom`, `nom`.

---

## 🚧 Blocage actuel (Phase 6)

Import testé dans UKG via **Importations → Configuration de l'employé → Employés**
(type Excel, bouton **« Test »** = essai à blanc, ne touche à rien).

**Erreur : « Missing header record ».**
Cause : les **en-têtes** du fichier (« ID d'employé / Prénom / Nom de famille ») ne
correspondent pas aux **noms de colonnes exacts** attendus par l'import UKG
(sensibles à la casse). Ce n'est PAS un problème de clé ni de données.

### Prochaine étape pour débloquer
1. Faire un **export des employés** depuis UKG (2-3 lignes) pour récupérer **les en-têtes
   exacts** attendus.
2. **Reformater** le fichier des noms sous ces en-têtes.
3. Relancer le **Test** → si OK, faire un **pilote de 5** (dont un nom accentué + un composé)
   → vérifier → puis les 107.

---

## Notes techniques utiles
- L'import UKG a un bouton **« Test »** (dry-run) : toujours l'utiliser avant « Importer ».
- Ordre d'identification des personnes à l'import UKG : **External ID → Payroll ID → Employee ID**.
  → Une fois le Maestro ID dans `ID externe`, les imports pourront cibler par lui.
- L'`ID externe` et le NAS (« ID national principal ») sont aussi gérables via l'import « Employés ».

## Points encore ouverts
- **Q1** — confirmer que les noms Maestro sont les noms légaux (avant écrasement).
- **Q10** — confirmer que `ID externe` est libre.
- **7 exceptions** — apparier par nom à la main.

---

## Reprendre à la prochaine session
👉 **Débloquer la Phase 6** : exporter des employés UKG pour obtenir les **en-têtes exacts**,
reformater le fichier des noms, relancer le **Test**, puis pilote de 5, puis les 107.
