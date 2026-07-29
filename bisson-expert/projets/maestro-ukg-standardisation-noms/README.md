# Standardisation des noms d'employés — Maestro ↔ UKG Ready

**Client :** Bisson Expert
**Consultant :** Karim Chaouche
**Statut :** Cadrage (Phase 0)
**Dernière mise à jour :** 2026-07-29

---

## 1. Besoin réel

Au-delà de « écraser les noms dans UKG », le besoin de fond est :

> **Pouvoir relier de façon fiable un employé entre Maestro et UKG Ready, pour que les
> données des deux systèmes se joignent proprement dans les dashboards, sans doublons
> ni faux positifs.**

L'uniformisation des noms est *un moyen* d'y arriver ; la **clé de correspondance
stable (Maestro ID)** en est le vrai livrable durable. Les noms peuvent rechanger
demain (mariage, correction d'orthographe, surnom) — un identifiant numérique, non.

## 2. Décisions déjà arrêtées (par le client)

| # | Décision | Détail |
|---|----------|--------|
| D1 | **Source de vérité des noms** | Maestro. On écrase les noms UKG avec ceux de Maestro. |
| D2 | **Nouveau champ dans UKG** | Un champ personnalisé « Maestro ID » sur la fiche employé. |
| D3 | **Finalité** | Dédoublonner et mieux matcher les données dans les dashboards. |

## 3. Insight critique — la clé de correspondance

On **ne peut pas** écraser les noms UKG en ciblant les employés par leur nom :
c'est justement le champ qui diffère et qui change. Chaque import UKG doit cibler
par un **identifiant stable que UKG connaît déjà** : le **UKG Employee ID**
(numéro d'employé interne UKG).

Il faut donc, avant tout import, construire une **table de correspondance** :

```
Maestro ID  ──►  UKG Employee ID
```

Cette table est le pivot du projet. Elle sert à :
1. Charger le bon Maestro ID sur la bonne fiche UKG.
2. Écraser le bon nom sur la bonne fiche UKG.
3. **Révéler les doublons** : si un même employé pointe vers 2 fiches UKG, ou si
   2 employés Maestro pointent vers la même fiche UKG, on a trouvé une anomalie à corriger.

### Comment matcher sans nom fiable ?

On matche sur une **clé secondaire présente dans les deux systèmes**, par ordre de fiabilité :

1. **NAS (numéro d'assurance sociale)** — clé la plus fiable, unique par personne.
2. **Date de naissance + initiales** — bon complément si NAS indisponible.
3. **Courriel professionnel** — s'il est saisi des deux côtés.
4. **Numéro d'employé existant** — si Maestro et UKG partagent déjà un numéro commun.
5. **Nom normalisé** (sans accents, minuscules, espaces réduits) — en dernier recours,
   à valider manuellement.

> ⚠️ Les cas non résolus automatiquement sont réglés **à la main** dans le crosswalk.
> C'est normal et attendu sur un premier passage.

## 4. Séquence d'exécution (l'ordre compte)

```
Phase 1  Exploration & profilage des données (choisir la clé de match)
Phase 2  Extraction Maestro          → gabarit 01
Phase 3  Extraction UKG + crosswalk  → gabarit 02   ← pivot du projet
Phase 4  Créer / valider le champ « Maestro ID » dans UKG
Phase 5  Importer le Maestro ID dans UKG            → gabarit 03
Phase 6  Écraser les noms dans UKG (pilote puis lot) → gabarit 04
Phase 7  Validation, réconciliation, dashboards
```

> **> 100 employés** → le matching manuel « à l'œil » n'est pas viable. On matche
> automatiquement sur une clé secondaire, **choisie à la Phase 1** après profilage.

**Pourquoi charger le Maestro ID (Phase 5) AVANT d'écraser les noms (Phase 6) ?**
Pour que le lien permanent Maestro↔UKG existe même si l'import des noms échoue
partiellement. Une fois le Maestro ID en place dans UKG, on ne dépend plus jamais
du nom pour rejouer un import.

---

## 5. Détail par phase

### Phase 1 — Exploration & profilage des données

Objectif : **découvrir** (pas deviner) quelle clé permet de matcher Maestro et UKG,
avant de figer quoi que ce soit. À faire ensemble, Karim + Claude.

- 1.1 **Inventaire des champs** : lister les colonnes disponibles à l'export côté
  Maestro (fiche employé) et côté UKG (Employee Information / Demographics).
- 1.2 **Clés candidates** : pour chacune (NAS, date de naissance, courriel, numéro
  d'employé existant), mesurer dans **chaque** système :
  - **complétude** = % de fiches où la valeur est renseignée ;
  - **unicité** = la valeur identifie-t-elle une seule personne (pas de collisions).
- 1.3 **Coïncidence d'identifiants** : vérifier si le numéro d'employé UKG égale déjà
  le Maestro ID (parfois le cas → matching trivial).
- 1.4 **Choix de la stratégie** : une clé unique et propre des deux côtés ; sinon une
  **clé composite** (ex. date de naissance + nom de famille) ; sinon nom normalisé +
  revue manuelle des restes.
- 1.5 Profiter de cette phase pour trancher **Q1** (nom légal vs affichage) et **Q9**
  (le champ « Maestro ID » existe-t-il déjà dans UKG, et avec quel paramétrage).

**Sortie :** décision de matching documentée (met à jour Q2 dans `questions-ouvertes.md`).
**Validation :** on a au moins une clé (ou combinaison) fiable pour > 100 employés.

### Phase 2 — Extraction Maestro (gabarit 01)

**Où :** module Ressources humaines / Paie de maestro\*.
**Quoi extraire :** numéro d'employé (Maestro ID), nom, prénom, + clé(s) secondaire(s)
disponibles (NAS, date de naissance, courriel) pour le matching.
**Comment :** export Excel/CSV depuis l'écran de liste des employés, ou via un rapport
maestro\*. Attention à l'**encodage** (voir section Risques — accents français).

→ Remplir `gabarits/01_maestro_export_employes.csv`.

### Phase 3 — Extraction UKG + crosswalk (gabarit 02)

1. **Exporter la liste actuelle des employés UKG** : UKG Employee ID, nom actuel,
   + les mêmes clés secondaires. (Dans UKG Ready : *My Team > Employee Information*,
   ou un rapport « Employee Details », exportable en Excel.)
2. **Sauvegarder cet export** — c'est ton point de restauration (rollback) des noms.
3. **Construire le crosswalk** : joindre Maestro et UKG sur la clé secondaire,
   résoudre les cas ambigus à la main, marquer chaque ligne
   (`statut_match` = auto / manuel / non_resolu / doublon).

→ Remplir `gabarits/02_crosswalk_maestro_ukg.csv`.

### Phase 4 — Créer / valider le champ personnalisé « Maestro ID » dans UKG

Dans UKG Ready (les chemins varient selon la version / les droits admin) :

- **Company Settings → HR Setup / Profiles → Custom Fields** (ou *Employee Custom Fields*).
- Créer un champ :
  - **Nom :** `Maestro ID`
  - **Type :** *Texte* (recommandé — préserve tout zéro de tête ou format ; on ne fait
    pas de calcul dessus).
  - **Cocher « inclus dans les rapports / reportable »** pour qu'il remonte dans les
    exports et dans le dashboard.
  - Portée : fiche employé (niveau employé, pas poste).

> Sans ce champ créé au préalable, l'import de Phase 4 n'a pas de colonne cible.

### Phase 5 — Importer le Maestro ID dans UKG (gabarit 03)

- UKG Ready : **Company Settings → Global Setup → Imports** (outil d'import).
- Fichier **clé = UKG Employee ID**, valeur = `maestro_id`.
- Mapper la colonne vers le champ personnalisé « Maestro ID », valider, puis committer.

→ Remplir `gabarits/03_ukg_import_maestro_id.csv`.

### Phase 6 — Écraser les noms dans UKG (gabarit 04)

- Même outil d'import UKG, **clé = UKG Employee ID**, valeurs = `prenom`, `nom_famille`.
- **Faire d'abord un lot pilote de 5 à 10 employés**, valider visuellement, puis lancer
  le reste.
- ⚠️ **Nom légal vs nom d'affichage** — voir Risques (R1). Confirmer QUEL champ nom on écrase.

→ Remplir `gabarits/04_ukg_import_noms.csv`.

### Phase 7 — Validation & réconciliation

- Compter : nb d'employés Maestro = nb de liens dans le crosswalk = nb de fiches UKG mises à jour.
- Contrôle par sondage : 10 fiches UKG au hasard, nom + Maestro ID conformes à Maestro.
- **Dashboards** : utiliser désormais le **Maestro ID comme clé de jointure** entre la
  source UKG et la source Maestro (au lieu du nom). Dans un modèle Power BI, le Maestro ID
  devient la clé de la dimension « Employé » partagée — c'est ce qui élimine les doublons.
- Documenter les doublons UKG trouvés en Phase 2 et le plan pour les fusionner/désactiver.

---

## 6. Risques et mitigations

| # | Risque | Impact | Mitigation |
|---|--------|--------|------------|
| **R1** | **Nom légal vs nom d'affichage.** UKG distingue le nom légal (paie, T4/RL-1, CCQ) du nom préféré/affichage. Écraser le nom légal avec un nom Maestro non légal = fiscalité brisée. | Élevé | Confirmer quel champ on écrase. Si Maestro porte le **nom légal**, OK. Sinon, n'écraser que le nom d'affichage. |
| **R2** | **Accents / encodage** (é, è, ç, ï…). Maestro/Sybase exporte souvent en Windows-1252 ; UKG attend souvent UTF-8. | Moyen | Fixer l'encodage à l'export, valider un nom accentué dans le lot pilote avant le lot complet. |
| **R3** | **Pas de clé secondaire commune fiable** (ni NAS ni courriel des deux côtés). | Moyen | Matching manuel assisté par nom normalisé ; prévoir du temps ; c'est ponctuel (une fois le Maestro ID chargé, plus jamais requis). |
| **R4** | **Écrasement irréversible des noms UKG.** | Moyen | Export UKG complet **avant** tout import (Phase 2, étape 2) = rollback. |
| **R5** | **Doublons dans UKG** (la raison d'être du projet). Une personne = 2 fiches UKG. | Moyen | Le crosswalk les révèle. Décider : fusionner, désactiver l'ancienne, ou router les données historiques. |
| **R6** | **Droits / accès** aux outils d'import et de création de champ dans UKG. | Faible | Confirmer qui a le rôle admin UKG pour créer le champ et lancer les imports. |

## 7. Questions ouvertes

Voir [`questions-ouvertes.md`](./questions-ouvertes.md).

## 8. Contenu du dossier

```
maestro-ukg-standardisation-noms/
├── README.md                         ← ce document (méthode + plan)
├── questions-ouvertes.md             ← décisions à confirmer avant d'exécuter
└── gabarits/
    ├── 01_maestro_export_employes.csv    ← extraction Maestro (source de vérité)
    ├── 02_crosswalk_maestro_ukg.csv      ← table de correspondance (pivot)
    ├── 03_ukg_import_maestro_id.csv      ← import du Maestro ID dans UKG
    └── 04_ukg_import_noms.csv            ← import d'écrasement des noms UKG
```
