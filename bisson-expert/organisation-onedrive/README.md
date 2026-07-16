# Organisation des dossiers OneDrive — Bisson Expert

Projet: simplifier l'arborescence OneDrive de Bisson Expert pour la rendre
plus courte, plus prévisible, et scalable pour les nouveaux projets.

## Constat — arborescence actuelle

- Trop de niveaux d'imbrication.
- Noms de dossiers trop longs (jusqu'à ~40 caractères par segment).
- Beaucoup de dossiers superflus ou en double (parallèles à la structure
  officielle) — candidats à relocaliser ou archiver.

### Dossiers clés à conserver
- **Suivi**
- **Soumissions en cours**
- **Projets actifs**

Tout le reste est jugé superflu ou parallèle et peut être relocalisé
(archivé) plutôt que reproduit dans la nouvelle structure.

## Problème d'intake — WeTransfer et courriel personnel

- Le serveur de Bisson Expert n'accepte plus les envois directs
  (WeTransfer, pièces jointes courriel).
- Résultat: chaque matin, quelqu'un doit manuellement renommer et
  enregistrer les fichiers reçus dans OneDrive. Processus manuel, source
  d'erreurs, non scalable.

## Problème technique observé — erreur d'extraction ZIP

Erreur rencontrée en tentant d'extraire un appel d'offres reçu
(`Bidding_Documents_-_15-07-26.zip`, ~150 fichiers) directement dans
l'arborescence OneDrive existante:

```
Error 0x80010135: Path too long
A-602-CIRCULATIONS-VERTICALES_ESCALIER-2-ET-4_ET-GARDE-CORPS-GRADINS-Rev.02.pdf
```

### Cause racine

- L'extracteur ZIP natif de l'Explorateur Windows est limité à ~260
  caractères de chemin total (`MAX_PATH`). Contrairement à une idée
  répandue, activer la politique Windows *"Enable Win32 long paths"*
  (`LongPathsEnabled`) **ne règle pas** ce cas: l'extracteur natif de
  l'Explorateur n'exploite pas cette option.
- OneDrive impose lui-même une limite d'environ 400 caractères par chemin
  synchronisé — au-delà, le fichier ne synchronise simplement pas, même
  s'il a été extrait localement.
- Le problème est cumulatif: chemin OneDrive racine (souvent 50-70
  caractères à lui seul) + arborescence interne profonde à noms longs
  (jusqu'à 40 car./segment) + noms de fichiers déjà longs fournis par des
  tiers (soumissionnaires, ex. ~78 caractères ci-dessus) = dépassement de
  la limite dès qu'on extrait à quelques niveaux de profondeur.

### Recommandations (bonnes pratiques, scalables)

1. **Ne jamais extraire un ZIP directement dans un dossier OneDrive
   profond via l'outil natif de l'Explorateur.**
   - Extraire d'abord dans un dossier local court, proche de la racine
     (ex. `C:\Extract\`), puis déplacer seulement les fichiers/dossiers
     nécessaires dans OneDrive.
   - Alternative: utiliser 7-Zip, qui gère mieux les chemins longs que
     l'extracteur natif (à valider selon la version/config du poste —
     n'élimine pas la limite OneDrive de ~400 caractères).
2. **Raccourcir l'arborescence cible.** Viser 3-4 niveaux maximum depuis
   la racine du projet, et ≤ 20-25 caractères par segment de dossier —
   voir la structure proposée ci-dessous.
3. **Ne pas conserver les noms de fichiers fournisseurs tels quels** dans
   les niveaux profonds. Renommer selon une convention courte au moment
   du classement (ex. `A602-Escalier-Rev02.pdf`) plutôt que de garder le
   nom original du soumissionnaire.
4. Cette combinaison (arborescence courte + renommage à l'intake) règle
   le problème à la source, contrairement à un correctif ponctuel
   (extraire fichier par fichier, "Skip"), qui ne passe pas à l'échelle
   pour ~150 fichiers par appel d'offres.

## Structure proposée

Voir [`structure-proposee/`](./structure-proposee) pour le squelette de
dossiers. Noms courts, 3 niveaux max depuis la racine du projet:

```
structure-proposee/
├── 01-suivi/
├── 02-soumissions-en-cours/
├── 03-projets-actifs/
├── 04-reception/            ← zone tampon pour WeTransfer / courriel,
│                              vidée chaque matin vers 02 ou 03
└── 05-archives/             ← tout ce qui est superflu/parallèle,
                               relocalisé ici plutôt que supprimé
```

Convention de nommage: dossiers en minuscules, mots séparés par des
tirets, ≤ 25 caractères par segment. Les fichiers reçus de tiers sont
renommés au moment du classement (voir recommandation 3 ci-dessus).

## Prochaines étapes

- [ ] Valider la structure proposée avec Bisson Expert.
- [ ] Identifier les dossiers "superflus/parallèles" à relocaliser vers
      `05-archives`.
- [ ] Documenter et communiquer à l'équipe la procédure d'extraction ZIP
      (extraire en local d'abord, renommer, puis déplacer dans OneDrive).
- [ ] Migrer les projets actifs vers la nouvelle structure.
