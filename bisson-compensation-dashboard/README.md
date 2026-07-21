# Bisson — Dashboard de rémunération (Power BI)

Projet de référence pour les **dashboards de rémunération** développés dans Power BI
pour Bisson Expert en Fondation.

Ce dossier n'est **pas** le fichier Power BI lui-même : c'est la **base de connaissances
versionnée** du projet. On y consigne les règles d'affaires confirmées, l'architecture des
calculs, les valeurs de validation, les pièges à ne jamais refaire et les mesures DAX
existantes. L'objectif : ne plus jamais repartir de zéro quand un bug de même famille
réapparaît ou qu'un nouveau dashboard du même type doit être bâti.

## Contexte rapide

- **Entreprise :** Bisson Expert en Fondation
- **Responsable :** Karim Chaouche (dashboards Power BI de rémunération)
- **Objet :** rémunération des représentants des ventes, programme à deux piliers
  1. **Commission progressive par paliers** — basée sur les ventes cumulées
  2. **Boni sur marge** — basé sur l'écart entre la marge réelle et la marge cible
- **Source de données :** `maestroProjectSaleSummaries_v3.csv` (Azure Blob Storage),
  alimenté par le backend C# de Julien, dates provenant du CRM Dynamics.

## Convention de nommage (colonnes backend)

| Suffixe          | Signification                       |
|------------------|-------------------------------------|
| `*_margin`       | boni marge                          |
| `*_cumulative`   | commission ventes cumulées          |
| `*_total`        | somme des deux                      |

## Index de la documentation

| # | Document | Contenu |
|---|----------|---------|
| 01 | [Contexte d'affaires](docs/01-contexte-affaires.md) | Entreprise, parties prenantes, objectif |
| 02 | [Modèle de données](docs/02-modele-donnees.md) | Tables, relations, colonnes clés |
| 03 | [Règles d'affaires confirmées](docs/03-regles-affaires.md) | Règles validées par Vincent |
| 04 | [Architecture — calculs au backend](docs/04-architecture-backend.md) | Colonnes livrées par Julien, ce qui est caduc |
| 05 | [Valeurs de référence de validation](docs/05-valeurs-reference.md) | Chiffres à retrouver pour valider |
| 06 | [Pièges confirmés — ne jamais refaire](docs/06-pieges-confirmes.md) | Erreurs qui ont coûté du temps |
| 07 | [Mesures DAX existantes](docs/07-mesures-dax.md) | Inventaire des mesures et colonnes calculées |
| 08 | [Méthode de travail](docs/08-methode-travail.md) | Règles fermes de développement |
| 09 | [État de référence — dashboard Reps](docs/09-etat-reference-dashboard-reps.md) | Baseline de validation (capture 2026-07-14) |
| 10 | [Incident & reconstruction — colonnes bonus v3](docs/10-incident-colonnes-bonus-2026-07.md) | Cause racine, correctifs Power Query, 6 mesures reconstruites (2026-07-14) |

## Statut du projet

Tous les chantiers identifiés sont **complétés** (voir historique dans la doc). Aucun
travail en cours. Ce dossier sert désormais de **référence** pour tout futur dashboard
de compensation du même type.
