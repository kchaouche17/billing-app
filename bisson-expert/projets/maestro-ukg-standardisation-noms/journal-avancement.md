# Journal d'avancement

Suivi vivant de l'exécution. **Aucune donnée employé réelle ici** (pas de NAS, pas de noms) —
uniquement méthode, décisions et compteurs. Les fichiers de travail (Maestro, UKG, crosswalk,
import) restent **en local chez Karim, hors dépôt**.

Sessions : **2026-07-29**, **2026-07-30**

---

## État par phase

| Phase | État | Détail |
|---|---|---|
| 0 — Cadrage | ✅ Fait | Décisions arrêtées (voir README §2). |
| 1 — Exploration | ✅ Fait | Clé = NAS ; Maestro ID → `External Id` ; périmètre actifs+inactifs. |
| 2 — Extraction Maestro | ✅ Fait | ~552 lignes (Numéro, Nom, Prénom, NAS, État). |
| 3 — Crosswalk | 🟠 **À REFAIRE sur 429** | Fait sur un sous-ensemble **filtré (114)** ; UKG en a en réalité **429** (filtre caché). |
| 4 — Valider `External Id` | ✅ Fait | Champ **vide** partout → libre (Q10 réglée). Format d'écriture résolu (voir notes). |
| 5 — Import Maestro ID | 🟡 Partiel | Mécanique résolue ; à refaire proprement sur les 429 avec la colonne `ID Maestro`. |
| 6 — Écraser les noms | 🟡 Partiel | **107 noms importés** (sous-ensemble filtré). ~322 restants une fois le crosswalk refait. |
| 7 — Validation / dashboards | ⬜ À faire | |

---

## 🔴 Découverte majeure (2026-07-30) — filtre caché dans UKG

L'export UKG initial avait un **filtre appliqué non vu** → on croyait **114 employés**.
Filtre enlevé : **429 employés** dans UKG.

**Conséquence :** le crosswalk (fait sur 114) doit être **refait sur les 429**. La méthode
est identique (NAS_norm + XLOOKUP), seule la source change. Les **107 noms déjà importés
restent valides** (rien de perdu) ; il reste ~322 employés à traiter.

---

## Ce qui est acquis (méthode crosswalk — à re-rouler sur 429)

Construction **locale** dans Excel (le NAS ne sort jamais du poste de Karim) :

1. Deux onglets : `Maestro` et `UKG`.
2. **NAS normalisé** des deux côtés (`NAS_norm`) — enlève espaces, force 9 chiffres :
   ```
   =IF(E2="";"";TEXT(SUBSTITUTE(E2;" ";"")*1;"000000000"))
   ```
3. **Join** sur `NAS_norm` pour ramener le numéro Maestro (colonne B du tab Maestro) :
   ```
   =IFERROR(XLOOKUP(G2;Maestro!$G:$G;Maestro!$B:$B);"")
   ```
4. Colonne de contrôle `COUNTIF` (0 = pas de match, 1 = ok, >1 = doublon).

### Résultats sur le sous-ensemble filtré (114) — à re-valider sur 429
- 107/114 appariés via le NAS ; 7 sans NAS UKG (à apparier par nom).
- Aucun doublon de NAS dans UKG. Compte générique « Manager » (9001) retiré.
- Noms : 91 déjà identiques, 16 à corriger → décision : écraser tous les appariés.

### 🔑 Découverte : les numéros sont (presque) les mêmes des deux côtés
Vérifié : **`Numéro` Maestro = `Employee Id` UKG** pour la grande majorité.
**Exceptions réelles** (là où ça diffère — et donc où l'`External Id` est indispensable) :
- FOURNIER : UKG `9999` = Maestro `1440`
- PAQUETTE : UKG `1498` = Maestro `4074`
- SINDA : UKG `4646` = **aucun** match Maestro (fait partie des exceptions)

→ La colonne à charger dans `External Id` doit être **`ID Maestro`** (le résultat du XLOOKUP),
**pas** le `Employee Id`. Pour ~104/107 c'est pareil, mais pas pour FOURNIER/PAQUETTE.

---

## ✅ Blocage « Missing header record » — RÉSOLU

L'import UKG veut les **en-têtes anglais exacts** (sensibles à la casse), pas les libellés FR.

**En-têtes valides :** `Employee Id`, `First Name`, `Last Name`, `New External Id`.

Mécanique confirmée par le **Test** (dry-run, ne touche à rien) :
- La clé = `Employee Id` (met à jour, ne crée pas ; Username/Email requis seulement pour une création).
- Pour **écrire** le Maestro ID, utiliser la colonne **`New External Id`** — la colonne
  `External Id` seule sert de **clé de recherche**, pas à écrire (sinon avertissement
  « Cannot change ExternalId »).
- Ne mettre **que** les colonnes voulues (ne pas importer les 250+ du template complet).

**Résultat obtenu :** les **noms des 107** ont été importés avec succès (vérifié sur des fiches
qui étaient « à changer » : accents/casse corrigés).

---

## Questions ouvertes / réglées
- **Q1** ✅ réglé : « Legal First Name is **not supported in Payroll companies** » → écraser
  `First Name`/`Last Name` (le nom principal) est le bon geste, sans risque paie.
- **Q10** ✅ réglé : `External Id` était **vide** partout.
- **7 exceptions** (dont SINDA) — à apparier par nom, à la main.
- **Nouveau** : refaire le crosswalk et les imports sur les **429**.

---

## ▶️ Reprendre à la prochaine session
1. **Ré-exporter UKG complet (429, filtre enlevé)** : `Employee Id`, `First Name`, `Last Name`,
   `Social Insurance Number`.
2. **Refaire le crosswalk** sur les 429 (NAS_norm + XLOOKUP → `ID Maestro`).
3. **Import noms** (colonnes `Employee Id`, `First Name`, `Last Name`) sur les nouveaux appariés
   — les 107 déjà faits sont OK.
4. **Import `New External Id` = `ID Maestro`** (pas `Employee Id`) pour poser le lien, en gérant
   FOURNIER/PAQUETTE et les exceptions sans match.
5. Toujours passer par le **Test** avant « Importer ».
