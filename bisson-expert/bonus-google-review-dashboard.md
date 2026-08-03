# Dashboard — Bonus Google Review (Bisson Expert)

> Page Power BI **« Sommaire Bonification Google Review »** dans `Marge et bonus.pbix`
> (connecté LIVE au modèle sémantique). Documentée le **2026-08-03**, avant validation
> de Vincent (prévue le lendemain).

## Contexte

Le bonus Google Review récompense les contributeurs d'un projet lorsqu'un **review 5★**
est rattaché au projet. Flux d'activation :

1. Un review 5★ est identifié à un projet.
2. Le chef de production ouvre le projet dans le **CRM → onglet Suivi** et coche
   **« Admissible au bonus sur reviews »**.
3. Le master de données Power BI capte le changement et applique le bonus aux personnes
   ayant travaillé sur le projet.

Le calcul est fait **en amont** (CRM / modèle) — la page ne fait que **surfacer** la donnée.
Historiquement ce bonus était **dormant (0 $)** ; il est maintenant **actif**.

## Source de données

- **Table** : `Fact_Bonus par employe` (grain **employé × projet** ; table sans accent : `employe`).
- **Colonne clé** : `TotalBonusAmountCustomerPublicReview` = la **part par employé**, déjà
  auto-agrégée en **Somme** (icône Σ). → à glisser directement, aucune mesure requise.
  - ⚠️ **SUM**, pas MAX : c'est un **montant** (part de chaque personne), donc additif —
    contrairement aux *jours* de complétion (répétés par employé → MAX).
- Tables **InputKit** (`Fact_InputKit reponses`, `metriques annuelles`, `metriques globales`)
  = source des reviews. Exposent `nb_google_review`, `google_review_1_5`, `taux_reponse`.
  Grain = **technicien / réponse de sondage** — **pas de `ProjectCode`** (voir « En suspens »).

## Composition de la page

- **Slicers** : `Année` · `Mois` · `Nom affichage` (Employé) · `ProjectCode` (Projet).
- **Carte** : « Bonus Google Review — à payer ».
- **Table par employé** : `Nom affichage` (Employé) · `EmployeeJobCategory` (Rôle) · bonus.
- **Table par projet** : `ProjectCode` (Projet) · bonus.
- **Histogramme** : bonus par `Mois` (trié chronologiquement, pas en ordre alpha).

## Filtres obligatoires (sur les 4 visuels de données)

Ces exclusions retirent des données sales connues du modèle — **sinon le total gonfle et
un doublon touche de l'argent réel** :

- `Nom affichage` **≠** `"Employé"` (ligne fantôme MaestroId = 0, sans nom).
- `Nom affichage` **≠** `"GREEN DUPE HENRY"` (doublon d'Henry Green).

## Validation des chiffres

Réconciliation faite au montage : **carte = table employé = table projet = Σ histogramme**.

- Total courant (2026, pré-validation Vincent) : **1 475 $**.
- Les **175 $** d'écart initial (carte 1 750 $ vs table 1 575 $) venaient de la ligne
  fantôme + du doublon, non filtrés sur la carte au départ → corrigé.
- ⚠️ Le total a ensuite glissé **1 575 $ → 1 475 $** (probable refresh ou review dé-cochée) —
  **à confirmer comme chiffre courant** avant paiement.

## Contrôles à refaire à chaque refresh (avant de payer)

1. Les **4 visuels concordent** sur le même total.
2. Les **2 filtres d'exclusion** sont présents sur les 4 visuels de données (pas juste un).
3. Le **vrai Henry Green** touche son bonus review (et non le doublon `GREEN DUPE HENRY`) —
   sinon **dédoublonner à la source**.

## En suspens

- **KPI ratio « projets avec review éligible ÷ total projets avec review Google »** :
  bloqué. `Fact_InputKit reponses` n'a **pas de clé projet** (grain technicien/sondage),
  donc « nombre de projets ayant un review » n'est pas calculable depuis InputKit.
  Dénominateur réaliste si voulu : `nb_google_review` (compte de **reviews**, pas de projets).
- **Validation Vincent** (prévue le lendemain de ce montage).
