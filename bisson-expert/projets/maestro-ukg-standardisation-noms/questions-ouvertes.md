# Questions ouvertes — à confirmer avant d'exécuter

Liste vivante. Chaque question résolue devient une décision documentée dans le README.

| # | Question | Pourquoi ça compte | Statut | Réponse |
|---|----------|--------------------|--------|---------|
| Q1 | **Quel champ nom écrase-t-on dans UKG : le nom légal (paie/T4/RL-1/CCQ) ou le nom d'affichage ?** | Écraser le nom légal avec un nom non légal brise la fiscalité. (Risque R1) | **À vérifier par Karim** (2026-07-29) | Karim vérifie dans UKG quel champ nom viser avant l'écrasement. |
| Q2 | **Quelle clé secondaire est disponible dans les DEUX systèmes pour matcher ?** (NAS, date de naissance, courriel, numéro d'employé commun) | Détermine la fiabilité et l'effort du crosswalk. | **Reporté à la Phase 1 — Exploration** (2026-07-29) | À déterminer par profilage des données (complétude + unicité de chaque clé candidate dans Maestro ET UKG). Karim et Claude vérifient ensemble. |
| Q3 | **Qui a les droits admin UKG** pour créer un champ personnalisé et lancer les imports ? | Bloque les Phases 3-5 sinon. (Risque R6) | **Résolu** (2026-07-29) | Karim a les droits admin UKG. |
| Q4 | **Format/type du champ « Maestro ID »** : texte ou numérique ? Y a-t-il des zéros de tête ? | Un zéro de tête perdu casse la jointure. Recommandation : texte. | Proposé | Texte (à valider) |
| Q5 | **Combien d'employés** au total (Maestro / UKG) ? | Dimensionne l'effort de matching manuel et le lot pilote. **Décide entre matching manuel (à l'œil) et matching sur clé secondaire.** | **Résolu** (2026-07-29) | **Plus de ~100 employés → matching automatique requis** (le manuel à l'œil n'est pas viable à cette échelle). |
| Q6 | **Les dashboards sont-ils dans Power BI ?** Comment la donnée UKG et Maestro y arrive-t-elle ? | Confirme que le Maestro ID doit devenir la clé de la dimension Employé partagée. | Ouvert | — |
| Q7 | **Que fait-on des doublons UKG** révélés par le crosswalk : fusion, désactivation, ou conservation historique ? | Décision métier, impacte la paie et l'historique. (Risque R5) | Ouvert | — |
| Q8 | **As-tu déjà les exports** Maestro et UKG en main, et dans quel format ? | Détermine par quelle phase on démarre concrètement. | **Résolu** (2026-07-29) | Rien en main → on démarre par l'extraction Maestro (Phase 1). |
| Q9 | **Le champ personnalisé « Maestro ID » existe-t-il déjà dans UKG ?** | Si oui, la Phase 3 est en partie faite ; à confirmer (nom, type, reportable). | **À vérifier par Karim** (2026-07-29) | Karim pense l'avoir déjà créé — à confirmer dans Company Settings → Custom Fields. |
