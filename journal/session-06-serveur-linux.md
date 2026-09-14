# Session 6, serveur Linux et automatisation

**Objectif.** Mettre en service `SRV01` sous Debian, durcir l'accès distant,
mettre en place sauvegardes et détection d'intrusion, puis automatiser deux
besoins réels de la maquette.

**Résultat.** Objectif atteint. Plusieurs incidents instructifs, dont deux
défauts de conception révélés par des tests d'échec.

---

## Partitionnement

Choix de LVM avec séparation de `/home`, `/var` et `/tmp`.

La séparation de `/var` est la plus déterminante : ce point de montage accueille
journaux et données applicatives. Non séparé, un service écrivant massivement
remplit la racine, et un système de fichiers racine saturé devient inutilisable,
souvent irréparable sans redémarrage. La séparation confine le dégât à une seule
partition.

---

## Surface d'attaque

Sélection limitée au serveur SSH et aux utilitaires usuels, sans environnement
graphique. Ce qui n'est pas installé ne peut être exploité, ne requiert aucun
correctif et ne consomme rien.

Refus de la participation aux statistiques d'utilisation de paquets : le service
correspondant transmet périodiquement la liste des paquets installés vers
l'extérieur. Divulgation d'information sur l'infrastructure et flux sortant non
décidé.

---

## Incident 1, deux gestionnaires de configuration réseau

**Constat.** Après passage en adressage statique, les routes portaient encore la
mention d'une origine DHCP et une adresse source correspondant à l'ancien bail.

**Analyse.** Recherche du processus maintenant ces routes plutôt que suppression
directe. Aucun client DHCP actif : le client avait été exécuté une fois par
l'installateur et s'était terminé. Les routes constituaient un état résiduel,
non un conflit actif. Un redémarrage a suffi.

> **Enseignement.** Supprimer un symptôme sans identifier ce qui le produit est
> le moyen le plus courant de transformer un incident simple en incident
> récurrent. Si un processus avait réellement maintenu ces routes, leur
> suppression directe les aurait vues réapparaître, et la conclusion aurait été
> celle d'un comportement inexplicable.
>
> Sur un système Linux, plusieurs outils peuvent prétendre gérer une interface.
> Un seul doit être actif. Identifier lequel gère réellement une interface est
> le premier réflexe face à un incident réseau.

---

## Incident 2, directives réseau sans effet

**Constat.** Après redémarrage du service réseau, le fichier de résolution ne
contenait plus aucun serveur de noms.

**Cause.** Les directives de résolution placées dans la configuration
d'interface ne produisent d'effet que si le paquet `resolvconf` est présent. Sur
une installation minimale, il ne l'est pas : les lignes sont lues, valides
syntaxiquement, et silencieusement ignorées.

**Dépendance circulaire.** Le gestionnaire de paquets ne pouvait plus résoudre
le dépôt, et le paquet corrigeant la résolution devait être téléchargé.

**Correction.** Écriture manuelle du fichier de résolution pour rompre la
boucle, puis installation du paquet, puis retour à une configuration unique.

> **Enseignements.**
>
> Une configuration écrite, valide syntaxiquement et sans effet est un motif
> récurrent. Même structure que l'objet de stratégie de groupe lié à une unité
> d'organisation sans utilisateur, rencontré à la session 5.
>
> Lorsqu'une réparation dépend de ce qui est cassé, rompre la boucle par le
> chemin manuel.
>
> Lecture du message d'erreur : le refus de connexion immédiat sur l'adresse de
> bouclage indique un service local absent, à distinguer du délai dépassé
> observé lors des tests d'isolement, qui traduisait un rejet silencieux par le
> pare-feu. Symptôme apparent identique, causes opposées.

---

## Incident 3, durcissement appliqué avant l'installation de la clé

**Constat.** Impossible de déposer la clé publique par le canal prévu, le
serveur n'acceptant plus que l'authentification par clé.

**Première hypothèse, erronée.** Le serveur annonçait `publickey` comme seule
méthode disponible, ce qui a conduit à conclure à un défaut de la distribution.

**Réfutation.** Le journal du service montrait des authentifications par mot de
passe acceptées vingt minutes plus tôt. L'horodatage du fichier de durcissement
coïncidait avec la dernière d'entre elles.

**Cause réelle.** Le fichier de durcissement avait été créé avant le dépôt de la
clé. Le service ayant rechargé sa configuration, la désactivation du mot de passe
a supprimé le seul moyen d'authentification disponible à ce moment.

**Correction.** Dépôt manuel de la clé publique depuis la console de
l'hyperviseur, chemin de secours hors bande.

> **Enseignements.**
>
> On installe le nouveau mécanisme avant de retirer l'ancien, et l'on vérifie
> qu'il fonctionne entre les deux. Même logique d'ordonnancement que la bascule
> DNS de la session 4, où l'ordre inverse aurait interrompu la résolution.
>
> Sur un serveur distant dépourvu d'accès hors bande, l'erreur est définitive.
>
> Une explication plausible n'est pas une cause vérifiée. La configuration
> effective obtenue par `sshd -T` constitue un fait ; un message de refus décrit
> un symptôme compatible avec plusieurs causes. Commencer par le fait.

---

## Signature d'une attaque par force brute

Le journal du service conserve la trace des tentatives infructueuses :

```
Invalid user SRV01 from 10.10.20.2
Failed password for invalid user SRV01 (répété)
Connection reset [preauth]
```

Bien qu'il s'agisse en l'occurrence d'une erreur de nom d'utilisateur, la
signature est identique à celle d'une attaque : compte inexistant, tentatives
rapprochées, même source.

Vocabulaire à retenir pour la phase de supervision : `Invalid user` signale un
compte inexistant, `Failed password` un compte existant dont le secret est
erroné. Le passage de l'un à l'autre indique qu'un attaquant a identifié un nom
valide, et constitue l'événement à détecter.

Les authentifications par clé journalisent l'empreinte de la clé employée, ce
qui permet d'identifier non seulement le compte mais la clé utilisée.

---

## Détection d'intrusion, validation par déclenchement

Prison `sshd` activée, trois échecs en dix minutes entraînant un bannissement
d'une heure.

Chaîne validée de bout en bout par déclenchement volontaire : tentatives
répétées, bannissement constaté, levée effective. Le bannissement se matérialise
par une règle insérée dans le pare-feu local, vérifiable indépendamment de
l'outil qui l'a posée.

Même méthode que la validation des alertes de supervision sur un projet
antérieur : une protection dont on n'a pas observé le déclenchement ne compte
pas.

---

## Sauvegarde et restauration

### Ajout d'une vérification d'intégrité

La rotation initiale supprimait les archives antérieures à la rétention sans
jamais vérifier qu'une archive était lisible. Une archive corrompue demeure une
archive du point de vue du critère d'âge.

Ajout d'un contrôle par listage du contenu, placé **avant** la rotation, avec
sortie en échec si l'archive est illisible. Aucune ancienne sauvegarde n'est
donc supprimée lorsque la nouvelle est invalide.

### Incident 4, expansion des motifs et élévation de privilèges

**Constat.** Une commande d'extraction employant un motif échouait sur un
fichier introuvable, alors que l'archive existait.

**Cause.** L'interprétation d'un joker relève du shell appelant, **avant**
l'élévation de privilèges. Le répertoire de destination étant restreint au
compte privilégié, le shell utilisateur ne trouvait aucune correspondance et
transmettait la chaîne littérale.

**Variante observée.** Exécuté depuis un shell privilégié, le motif
correspondait à deux archives. Le premier argument étant interprété comme
l'archive et les suivants comme les membres à extraire, la commande recherchait
la seconde archive à l'intérieur de la première.

> **Enseignement.** Un motif portant sur plusieurs correspondances modifie le
> sens de la commande. Pour une opération de restauration, l'archive est nommée
> explicitement.

### Restauration vérifiée

Extraction dans un répertoire temporaire, puis comparaison récursive avec
l'original. Aucune différence constatée.

---

## Script de création d'utilisateurs

### Défaut révélé par le test de doublon

Le cas nominal fonctionnait intégralement. Le rejeu du même fichier a mis en
évidence un défaut de conception.

Un objet utilisateur Active Directory est soumis à **deux contraintes d'unicité
indépendantes** : l'identifiant de connexion, unique dans le domaine, et le nom
commun, unique dans son unité d'organisation. La première version ne traitait
que la première.

Conséquence observée : un identifiant incrémenté correct, suivi d'un échec
portant sur un nom déjà utilisé, message sans rapport apparent avec
l'identifiant affiché.

Correction : le nom commun dérive de l'identifiant et devient unique par
construction, l'attribut d'affichage conservant la forme lisible.

> **Enseignements.**
>
> La cause réelle et le message d'erreur ne coïncidaient pas. Sans connaissance
> de la distinction entre les deux attributs, la recherche aurait porté
> durablement sur le mauvais.
>
> Le test négatif a révélé un défaut que le cas nominal ne pouvait pas montrer.
> Ce défaut serait apparu en exploitation à la première homonymie, c'est-à-dire
> au plus mauvais moment.

### Report de la leçon de la session 5

Le script vérifie l'attribut d'activation après création, plutôt que de conclure
au succès sur la seule absence d'exception. Un mot de passe non conforme à la
politique produit un compte créé mais désactivé, sans erreur.

### Limite connue

Le comportement en cas d'identifiant existant crée systématiquement un compte
supplémentaire. Correct pour deux homonymes réels, erroné lors du rejeu
accidentel d'un même fichier.

Traitement à prévoir : ignorer par défaut une ligne correspondant à un compte
existant sur le prénom, le nom et le département, et subordonner la création
d'un homonyme à un commutateur explicite.

---

## Erreurs d'environnement d'exécution

Trois occurrences dans la même session, toutes de même nature : la commande
était correcte, l'environnement ne l'était pas.

- Confusion entre le client et le démon dans le nom d'une commande de
  vérification
- Script lancé depuis l'interpréteur de commandes hérité plutôt que depuis
  l'interpréteur moderne, conduisant à l'ouverture du fichier dans un éditeur
- Tests d'isolement exécutés depuis une console demeurée ouverte sur une autre
  machine

> **Enseignement.** Vérifier l'environnement avant la commande. Une sortie
> cohérente peut répondre à une question qui n'était pas celle posée.

---

## État à la fin de la session

- `SRV01` en service, adressage statique, résolution fonctionnelle
- Accès distant par clé uniquement, durcissement vérifié sur configuration
  effective
- Correctifs de sécurité appliqués automatiquement
- Pare-feu local en refus par défaut, SSH restreint au segment serveurs
- Détection d'intrusion validée par déclenchement
- Sauvegarde quotidienne planifiée, intégrité contrôlée, restauration vérifiée
- Deux scripts en service, documentés, testés sur leurs cas d'échec

## Prochaine étape

Durcissement et authentification réseau : renforcement des règles inter-VLAN,
stratégies de mot de passe, gestion des comptes administrateurs locaux, et mise
en service d'un serveur RADIUS adossé à l'annuaire.
