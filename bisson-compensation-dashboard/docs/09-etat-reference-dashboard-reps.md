# 09 — État de référence : dashboard « Sommaire de bonification Rep »

Capture d'écran de référence prise le **2026-07-14** (fichier `Job Cost - Sommaire
Bonification Reps - V2`, dernier enregistrement 09h34). Sert de **baseline** : avant/après
chaque modification, on compare à ces valeurs pour confirmer qu'on n'a rien cassé.

> ⚠️ Contexte de filtrage au moment de la capture : **Année = 2026**, **Mois = plusieurs
> sélections**, **Représentants = Tout**, **Facturation débutée = Tout**. Ce n'est donc PAS
> le contexte Q1 / Rep 20 des valeurs de validation de la [section 05](05-valeurs-reference.md) —
> ne pas confondre les deux jeux de chiffres.

## En-tête / slicers

| Slicer | Valeur |
|---|---|
| Année | 2026 |
| Mois | Plusieurs sélections |
| Représentants | Tout |
| Facturation débutée | Tout |

## KPI globaux (haut droite)

| Mesure | Valeur |
|---|---|
| % cible total | **45,61 %** |
| % réel total | **36,74 %** |

## Cartes KPI (colonne de droite)

| Carte | Valeur |
|---|---|
| Palier Commission Atteint | **Palier 1 — 0,50 %** |
| Ventes Cumulées (1 an) | **$5 702 230,04** |
| Écart Cible Mo… (moyen) | **-6,61 %** |
| Multiplicateur … | **-9,38 %** |
| Bonus sur marge | **47 305,0…** (abrégé à l'écran) |
| Commission | **$30 740,…** (abrégé à l'écran) |
| Bonus Total | **$78 045,75** |

> Rappel piège Card v2 : la section « Valeur » n'est pas expansible, d'où l'affichage
> abrégé sur « Bonus sur marge » et « Commission ». Les montants exacts sont à lire dans
> les tableaux / DAX Studio, pas dans la carte.

## Tableau 1 — Commission / projets

Colonnes : **No de projet · Type · Projet · Adresse travaux**

| No de projet | Type | Projet | Adresse travaux |
|---|---|---|---|
| 0788-24 | 32 | Karine Boulet Gaudreault | 2175-2179, Gauthier à Montréal |
| 0792-24 | 12 | Richard Geoffrion | 4235-4241, Messier à Montréal |
| 0974-25 | 73 | Josée Gaudreault | 9156, Hochelaga à Montréal |
| 0981-25 | 11 | Coop Château Maribert Nina Admo | 4350-4366, Saint-Hubert à Montréal |
| 0986-25 | 11 | Marek Ziolkowski et Als | 4439-4443, De Lorimier à Montréal |
| 0994-25 | 12 | Sylvie Choquette | 4813-4817, Saint-Émilie à Montréal |
| 1004-25 | 41 | Sophie Cazenave | 4824-4828, Clark à Montréal |
| 1012-25 | 63 | Bhushan Bhuvanagiri | 2630-2636, Modugno à Montréal |
| 1013-25 | 41 | Gilles Longtin | 2574, Mcgill à Longueuil |
| 1018-25 | 11 | Remo Barone | 5044, Sainte-Marie à Montréal |

(Ligne **Total** présente en bas du tableau ; barre de défilement horizontale → colonnes
de montants non visibles sur la capture.)

## Tableau 2 — Veille Marge

Titre : **« Veille Marge - <25% - >70% - DIFF >=25% »**
Colonnes : **No de projet · Type · Projet · Adresse travaux · % Cible (Excel) · % Réel**

| No de projet | Type | Projet | Adresse travaux | % Cible (Excel) | % Réel |
|---|---|---|---|---|---|
| 0986-25 | 11 | Marek Ziolkowski et Als | 4439-4443, De Lorimier à Montréal | 47,28 % | 18,52 % |
| 1012-25 | 63 | Bhushan Bhuvanagiri | 2630-2636, Modugno à Montréal | 44,00 % | -32,35 % |
| 1013-25 | 41 | Gilles Longtin | 2574, Mcgill à Longueuil | 50,89 % | -7,01 % |
| 1018-25 | 11 | Remo Barone | 5044, Sainte-Marie à Montréal | 44,22 % | 12,20 % |
| 1036-25 | 11 | Stephane Tchoulack | 2167-2171, Marie-Anne Est à Montréal | 48,18 % | -17,18 % |
| **Total** | | | | **47,12 %** | **1009,70 %** |

> Note : le Total « % Réel » à **1009,70 %** est une somme de pourcentages ligne à ligne
> (pas une moyenne pondérée) — à garder en tête si on retouche l'agrégation de cette colonne.

## Observations à surveiller lors des modifications

- Les deux tableaux ne listent pas exactement les mêmes projets (le Tableau 2 est filtré
  sur les seuils de marge <25 % / >70 % / DIFF ≥25 %).
- `Stephane Tchoulack` (1036-25) apparaît dans la Veille Marge mais pas dans le Tableau 1
  visible → confirmer que c'est attendu selon le contexte de filtre.
