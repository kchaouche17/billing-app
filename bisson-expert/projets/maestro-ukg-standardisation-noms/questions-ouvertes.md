# Questions ouvertes — à confirmer avant d'exécuter

Liste vivante. Chaque question résolue devient une décision documentée dans le README.

| # | Question | Pourquoi ça compte | Statut | Réponse |
|---|----------|--------------------|--------|---------|
| Q1 | **Quel champ nom écrase-t-on dans UKG : le nom légal (paie/T4/RL-1/CCQ) ou le nom d'affichage ?** | Écraser le nom légal avec un nom non légal brise la fiscalité. (Risque R1) | **Ouvert — à vérifier par Karim** (2026-07-29) | Cible probable = **nom principal `Prénom`/`Nom`** (ce que lisent les dashboards). UKG a aussi un `Prénom légal` séparé (vide sur la fiche vue). En attente : Karim confirme que les noms Maestro sont les **noms légaux**. **Dernier point bloquant avant la Phase 6.** |
| Q2 | **Quelle clé de match est disponible dans les DEUX systèmes ?** | Détermine la fiabilité et l'effort du crosswalk. | **Résolu — Phase 1** (2026-07-29) | **Clé = NAS.** Rempli pour tous dans Maestro ; champ `Numéro d'assurance sociale` présent dans UKG. Matching **local**, valeurs jamais partagées ni committées. Secours : nom normalisé + revue manuelle. Écartés : courriel (absent de Maestro), date de naissance (trous dans Maestro). |
| Q3 | **Qui a les droits admin UKG** pour lancer les imports ? | Bloque les Phases 4-6 sinon. (Risque R6) | **Résolu** (2026-07-29) | Karim a les droits admin UKG. |
| Q4 | **Format/type de l'identifiant Maestro stocké dans UKG** : texte ou numérique ? | Un zéro de tête perdu casse la jointure. Recommandation : texte. | Proposé | Texte (à valider). Stocké dans `ID externe` (champ texte natif). |
| Q5 | **Combien d'employés** au total (Maestro / UKG) ? | Dimensionne l'effort de matching et le lot pilote. | **Résolu** (2026-07-29) | **~300 à 800 employés** (actifs + inactifs) → matching automatique sur NAS requis. |
| Q6 | **Les dashboards sont-ils dans Power BI ?** Comment la donnée UKG et Maestro y arrive-t-elle ? | Confirme que le Maestro ID doit devenir la clé de la dimension Employé partagée. | Ouvert | — |
| Q7 | **Que fait-on des doublons UKG** révélés par le crosswalk : fusion, désactivation, ou conservation historique ? | Décision métier, impacte la paie et l'historique. (Risque R5) | Ouvert | — |
| Q8 | **As-tu déjà les exports** Maestro et UKG en main, et dans quel format ? | Détermine par quelle phase on démarre concrètement. | **Résolu** (2026-07-29) | Rien en main → on démarre par l'extraction Maestro (Phase 1). |
| Q9 | **Où loger le Maestro ID dans UKG : champ perso ou champ natif ?** | Évite de créer un champ inutile. | **Résolu** (2026-07-29) | Section « Champs supplémentaires » **vide** → aucun champ perso existant. **Décision : utiliser le champ natif `ID externe`** (vide, reportable, conçu pour un ID de système externe). |
| Q10 | **Le champ `ID externe` est-il libre** (non utilisé par une intégration UKG existante) ? | S'il sert déjà à autre chose, on l'écraserait. | **Ouvert — à vérifier par Karim** (2026-07-29) | Vide sur la fiche vue. Confirmer qu'aucune intégration ne s'en sert avant l'import (Phase 5). Plan B : champ perso « Maestro ID ». |
| Q11 | **Le NAS est-il rempli pour (quasi) tous dans UKG ?** | Détermine la couverture du match automatique. | **Résolu** (2026-07-29) | Quelques trous → **107/114 appariés (94 %)** ; 7 exceptions sans NAS UKG à apparier par nom. |
| Q12 | **Les noms UKG sont-ils stockés en MAJUSCULES / sans accents ?** | Sur la fiche vue : « SARAH » / « LAGACE ». Confirme la valeur ajoutée de la standardisation (casse + accents). | Ouvert | Observation à confirmer sur l'export UKG complet. |
