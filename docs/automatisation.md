# Automatisation

> Description des scripts en service sur la maquette, de leurs choix de
> conception et de leurs limites connues.

---

## Principe retenu

Aucun script n'a été écrit à titre d'exercice. Chacun répond à un besoin réel de
la maquette. Le langage s'acquiert en automatisant une manipulation dont on a
assez, non en traitant des cas artificiels.

---

## `sauvegarde-config.sh`

**Emplacement** : `/usr/local/bin/` sur `SRV01`
**Déclenchement** : minuteur `systemd`, quotidien
**Objet** : archivage des configurations et de la liste des paquets installés

### Structure

1. Journalisation du démarrage
2. Création du répertoire de destination et application des permissions
3. Export de la liste des paquets installés
4. Création de l'archive
5. **Vérification d'intégrité**
6. Rotation des archives antérieures à la rétention
7. Journalisation de la fin

L'ordre des étapes 5 et 6 est structurant : aucune ancienne sauvegarde n'est
supprimée avant validation de la nouvelle.

### Choix de conception

| Choix | Motif |
|---|---|
| `#!/usr/bin/env bash` | L'interpréteur est recherché dans le `PATH`, ce qui rend le script portable |
| `set -euo pipefail` | Comportement par défaut de `bash` : poursuivre après une erreur. Inacceptable pour une opération destructive |
| Journalisation par `logger` | Le message rejoint le journal système, donc horodaté, soumis à rotation et collectable par la supervision. Un fichier propre au script est invisible pour celle-ci |
| Sortie non nulle en cas d'échec | Condition nécessaire pour que `systemd` signale la défaillance |
| Chemins en variables | Une valeur employée deux fois finit par diverger si elle est écrite deux fois |

### Limites connues

Les archives ne quittent pas la machine. La règle des trois copies sur deux
supports dont un hors site n'est pas satisfaite.

Aucune vérification de l'espace disponible avant création de l'archive.

Le contenu de l'archive n'est pas chiffré : il inclut des configurations de
sécurité, dont celle du service SSH.

---

## `New-UtilisateursAD.ps1`

**Emplacement** : `C:\scripts\` sur `DC01`
**Déclenchement** : manuel
**Objet** : création d'utilisateurs depuis un fichier CSV, avec rattachement aux
groupes selon le modèle AGDLP

### Entrée

Fichier CSV à séparateur point-virgule, colonnes `Prenom`, `Nom`,
`Departement`, `Fonction`.

**Aucune colonne de mot de passe.** Un fichier contenant des mots de passe
circulerait par messagerie, subsisterait dans un dossier partagé et resterait
lisible des mois durant. Les mots de passe sont générés par le script et
exportés séparément, avec mention explicite du canal de transmission attendu et
de la suppression du fichier.

### Fonctionnalités

| Fonctionnalité | Motif |
|---|---|
| En-tête de documentation | Alimente `Get-Help`, le script se documente comme une applet native |
| Mode simulation | Permet de constater ce qui serait fait avant de le faire. Non négociable sur un traitement de masse |
| Génération d'identifiants uniques | Les homonymes ne se recouvrent pas |
| Génération de mots de passe conformes | Un caractère de chaque catégorie, puis mélange : sans mélange, l'ordre des catégories serait prévisible |
| Vérification de l'état après création | Contrôle de l'attribut `Enabled` |
| `try/catch` par ligne | Une ligne en échec n'interrompt pas le traitement des suivantes |
| Compte rendu | Statut et motif par ligne, totaux en fin d'exécution |

### Vérification de l'état après création

**Point issu d'un incident constaté.** Lorsque le mot de passe fourni ne
satisfait pas la politique du domaine, l'objet utilisateur est créé mais demeure
**désactivé**, sans erreur bruyante. L'absence d'exception ne vaut donc pas
succès.

Le script contrôle explicitement l'attribut `Enabled` après création et signale
le cas comme un échec.

### Deux contraintes d'unicité distinctes

**Défaut identifié lors des tests de doublon.** Un objet utilisateur Active
Directory est soumis à deux contraintes d'unicité indépendantes :

- `SamAccountName`, unique à l'échelle du domaine
- Le nom commun, unique au sein de l'unité d'organisation

La première version ne traitait que la première. Une homonymie produisait un
identifiant incrémenté correct, puis un échec portant sur un nom déjà utilisé,
message sans rapport apparent avec l'identifiant affiché.

Correction retenue : le nom commun dérive de l'identifiant et devient unique par
construction, l'attribut d'affichage conservant la forme lisible. Pratique
courante dans les organisations de taille importante, précisément en raison des
homonymies.

### Limites connues

Le comportement en cas d'identifiant existant crée systématiquement un compte
supplémentaire. Exact pour deux homonymes réels, erroné lors du rejeu accidentel
d'un même fichier, situation fréquente en exploitation.

Traitement à prévoir : ignorer par défaut une ligne dont le prénom, le nom et le
département correspondent à un compte existant, et subordonner la création d'un
homonyme à un commutateur explicite.

Le fichier d'identifiants produit contient des mots de passe en clair. Sa
suppression après transmission relève actuellement de l'opérateur.

---

## Enseignement transverse

Les deux scripts fonctionnaient au premier essai sur leur cas nominal. Les
défauts ont été révélés par les tests d'échec : archive corrompue pour l'un,
doublons et lignes invalides pour l'autre.

Tester que le traitement échoue lorsqu'il doit échouer révèle ce que tester le
succès ne montre jamais.
