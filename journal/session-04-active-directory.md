# Session 4, mise en service de l'annuaire

**Objectif.** Promouvoir `DC01` en contrôleur de domaine de la forêt
`lab.internal`, basculer la résolution de noms, puis structurer l'annuaire :
unités d'organisation, groupes, comptes, délégation et stratégies de groupe.

**Résultat.** Objectif atteint. Deux dépendances non anticipées identifiées et
corrigées, une erreur de manipulation détectée et rattrapée.

---

## Ce qu'installe réellement une promotion

Trois services distincts, souvent confondus en un seul.

**Un annuaire.** Base hiérarchique stockant les objets du système
d'information, consultable en LDAP. Chaque objet possède un nom distinctif
unique décrivant son emplacement dans l'arborescence.

**Un service d'authentification.** Kerberos, où le contrôleur joue le rôle de
tiers de confiance et délivre des tickets prouvant l'identité sans faire
circuler le mot de passe. Les tickets étant horodatés, un écart d'horloge
supérieur à cinq minutes fait échouer l'authentification.

**Un serveur DNS.** Un client ne localise pas son contrôleur de domaine
autrement qu'en interrogeant le DNS pour des enregistrements SRV, lesquels
indiquent quelle machine porte quel service sur quel port. C'est pourquoi le
rôle DNS est installé par la promotion elle-même et n'a pas à être ajouté au
préalable.

Le rôle à sélectionner est `Services de domaine Active Directory`, à ne pas
confondre avec `Services AD LDS` (annuaire LDAP allégé sans domaine),
`Services AD RMS` (gestion des droits sur les documents) ou
`Services de fédération AD FS` (authentification unique vers des applications
externes).

---

## Choix de promotion

| Décision | Choix | Motif |
|---|---|---|
| Périmètre | Nouvelle forêt | Premier contrôleur, aucune forêt existante |
| Niveau fonctionnel | Le plus élevé disponible | Aucun système antérieur à intégrer |
| Mot de passe DSRM | Défini et conservé en coffre | Seul moyen d'accès si l'annuaire est corrompu |

L'avertissement relatif à l'impossibilité de créer une délégation DNS est
attendu : aucun DNS parent ne gère `.internal` et ne peut donc déléguer la zone.
La forêt est isolée.

---

## Dépendance 1, le redirecteur DNS hérité

**Constat.** Après promotion, le redirecteur DNS de `DC01` pointait vers
`10.10.20.1`, valeur reprise de la configuration de la carte réseau au moment de
la promotion. La résolution externe empruntait donc deux résolveurs en cascade.

**Problème.** La matrice de flux prévoit la fermeture du flux DNS entre le
segment SERVEURS et le pare-feu. Cette fermeture aurait coupé la résolution
externe de l'ensemble du domaine, sans lien apparent avec la modification.

**Correction.** Redirecteur repointé directement vers un résolveur public,
avant exécution de la séquence de bascule.

> **Enseignement.** Une valeur héritée d'un état transitoire survit à cet état.
> Relire les configurations générées automatiquement plutôt que supposer
> qu'elles reflètent la cible.

---

## Bascule DNS

Séquence exécutée dans l'ordre défini par `matrice-de-flux.md` :

1. Vérification de la résolution externe depuis `DC01`
2. Modification du serveur annoncé par Kea vers `10.10.20.10`, sur les
   sous-réseaux POSTES et SERVEURS
3. Suppression des deux règles `TEMPORAIRE` autorisant le DNS vers `FW01`

L'ordre inverse aurait interrompu la résolution sur l'ensemble de la maquette.
La règle réservant le DNS sortant à `DC01` seul se trouve validée par ce
fonctionnement.

---

## Structure de l'annuaire

**Motif du refus des conteneurs par défaut.** `CN=Users` et `CN=Computers` sont
des conteneurs et non des unités d'organisation. Un conteneur n'accepte pas de
lien de stratégie de groupe : tout objet qui y demeure échappe à toute GPO hors
niveau domaine. C'est la première cause du symptôme « la stratégie ne s'applique
pas ».

Structure retenue, de type fonctionnel, détaillée dans
`docs/configuration-dc01.md`.

Création en PowerShell plutôt qu'en console graphique : l'opération devient
reproductible et documentée par son propre code.

---

## Modèle AGDLP

Présenté en formation comme une règle à retenir, il se comprend mieux par le
problème qu'il résout.

Accorder un droit directement à une personne fonctionne à petite échelle. À
quelques centaines de comptes, plus personne ne sait qui accède à quoi, et un
changement d'affectation impose de parcourir chaque ressource.

Le modèle insère deux niveaux d'indirection et sépare ainsi deux questions dont
les rythmes de changement sont indépendants : **qui est cette personne**, porté
par le groupe global et modifié lors des recrutements et mutations, et **qui a
le droit de faire ceci sur cette ressource**, porté par le groupe de domaine
local et modifié lors de l'apparition d'une ressource.

Conséquence pratique : une mutation se traduit par la modification d'une seule
appartenance, sans intervention sur aucune ressource.

> **Point de vigilance.** Le groupe global est membre du groupe de domaine
> local, jamais l'inverse. L'imbrication incorrecte ne produit aucune erreur,
> seulement des droits inopérants.

---

## Séparation des comptes d'administration

Un compte quotidien est exposé en permanence au courriel et à la navigation.
S'il porte des privilèges d'administration, une seule pièce jointe suffit à
compromettre la forêt.

Deux comptes distincts ont donc été créés pour le même utilisateur : l'un pour
travailler, l'autre pour administrer. Le compte d'administration n'est membre
d'aucun groupe fonctionnel et reçoit ses droits par délégation explicite.

C'est la base du modèle de tiering, pratique fréquemment énoncée et rarement
appliquée.

---

## Délégation

Droit de réinitialisation des mots de passe accordé au compte d'administration
sur `OU=Utilisateurs`, sans privilège d'administration du domaine.

L'assistant écrit une entrée de contrôle d'accès sur l'unité d'organisation,
héritée par les objets contenus. Rien d'autre qu'une permission posée sur un
objet d'annuaire.

C'est exactement cette mécanique qu'exploitent les chemins d'élévation de
privilèges : une délégation excessive, ou un droit générique accordé par
commodité, devient un vecteur. Les outils de cartographie offensive parcourent
ces listes à la recherche de chaînes exploitables.

---

## Erreur de manipulation, GPO par défaut modifiée

**Constat.** Les paramètres de bannière d'ouverture de session ont d'abord été
saisis dans la `Default Domain Policy`, l'éditeur ayant été ouvert sur cet objet
plutôt que sur un objet dédié.

**Correction.** Paramètres repassés à `Non défini`, puis créés dans
`GPO_Postes_Durcissement`.

> **Règle.** Les deux GPO par défaut ne se modifient pas, à l'exception de la
> politique de mot de passe et de verrouillage, qui ne peut structurellement pas
> résider ailleurs. Elles constituent une référence saine, réinitialisable par
> `dcgpofix` en cas de dérive : y placer ses propres réglages fait perdre cette
> possibilité.
>
> Repasser un paramètre à `Non défini` est le geste correct, plus sûr que la
> suppression de l'objet.

**Point de méthode associé.** Le chemin initialement suivi pour la bannière ne
correspondait à aucun paramètre existant. Les modèles d'administration
comportent plusieurs milliers d'entrées aux libellés proches : activer une
entrée approchante pour avancer produit un effet impossible à rattacher à son
origine des semaines plus tard. Vérifier avant de cliquer.

---

## Objet volontairement défectueux

`GPO_Test_Erreur` contient exclusivement un paramètre de configuration
utilisateur, alors que son lien porte sur une unité d'organisation ne contenant
que des comptes ordinateurs.

Une GPO comporte deux moitiés indépendantes : la configuration ordinateur
s'applique selon l'emplacement du compte machine, la configuration utilisateur
selon l'emplacement du compte utilisateur. Ici, aucun compte utilisateur ne se
trouve sous le point de liaison.

Le lien est actif, l'objet est téléchargé, la moitié utilisateur est ignorée.
**Aucune erreur, aucun avertissement, aucun symptôme visible sinon un réglage
qui ne prend pas.** C'est le scénario réel le plus fréquent.

Cet objet servira de support à l'exercice de diagnostic à la mise en service de
`PC01`.

---

## Lecture de l'ordre de priorité

`Get-GPInheritance` retourne deux listes distinctes : les liens directs, et
l'ensemble de ce qui s'applique, héritage compris.

La seconde est ordonnée **par priorité décroissante**, ce qui constitue
l'inverse de l'ordre d'application chronologique. Le domaine s'applique en
premier, l'unité d'organisation écrase ensuite et se retrouve donc en tête.

La console graphique numérote par ailleurs les liens en plaçant `1` au plus
prioritaire, ce qui achève de brouiller les repères. Principe à retenir plutôt
que numérotation : plus le point de liaison est proche de l'objet, plus sa
priorité est forte.

---

## État à la fin de la session

- Forêt `lab.internal` opérationnelle, `DC01` unique contrôleur
- DNS interne fonctionnel, résolution externe par redirecteur direct
- Bascule DNS exécutée, règles transitoires supprimées
- Dix unités d'organisation, six groupes selon AGDLP, quatre comptes
- Délégation d'administration accordée à un compte non privilégié
- Trois stratégies de groupe en service, dont une défectueuse par conception

## Reste à traiter dans cette étape

- Jonction de `PC01` au domaine
- Diagnostic de `GPO_Test_Erreur` par rapport `gpresult`
- Exécution des trois tests d'isolement inter-segments en attente depuis
  l'étape 3, `PC01` constituant la première machine hors du segment SERVEURS
