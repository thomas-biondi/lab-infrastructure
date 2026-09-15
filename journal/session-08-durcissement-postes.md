# Session 8, durcissement des postes

**Objectif.** Rattacher l'administration locale des postes au modèle de groupes
de l'annuaire, puis mettre en service la gestion automatique des mots de passe
administrateurs locaux.

**Résultat.** Objectif atteint. Un diagnostic long sur une préférence de
stratégie sans effet, résolu en descendant jusqu'à la donnée manipulée.

---

## Diagnostic principal, préférence sans effet

### Symptôme

La préférence de stratégie destinée à ajouter un groupe de domaine au groupe
Administrateurs local ne produit aucun effet. Le traitement se déclare réussi,
aucune erreur n'est journalisée, et le groupe n'est pas modifié.

### Pistes explorées et écartées

| Piste | Vérification | Conclusion |
|---|---|---|
| Connectivité vers le contrôleur | Localisation du contrôleur, accès au partage de stratégies | Fonctionnelle |
| Emplacement du compte machine | Nom distinctif de l'objet ordinateur | Conforme |
| Existence du groupe de domaine | Interrogation de l'annuaire | Existant, nom exact |
| Réception de la stratégie | Objets appliqués sur le poste | Reçue |
| Filtrage de sécurité de l'objet | Permissions, présence du droit d'application | Conforme |
| Présence du fichier de préférence | Partage du contrôleur et cache local du poste | Présent des deux côtés |

Six vérifications, toutes conformes, et aucun résultat. À ce stade, toutes les
couches d'abstraction affirmaient que le mécanisme devait fonctionner.

### Fausse piste

Une lecture des appartenances de groupe a fait apparaître un compte membre des
quatre groupes les plus privilégiés de la forêt, ce qui semblait constituer un
écart de sécurité majeur.

La commande avait en réalité été exécutée depuis une console élevée sous une
autre identité. L'outil rapportait les appartenances du compte élévateur, non
celles de la session interactive.

> **Enseignement.** En session élevée avec un autre compte, toute commande
> interrogeant « l'utilisateur courant » interroge le compte élévateur :
> identité, variables d'environnement, chemins de profil, résultats de
> stratégie. C'est une cause classique de faux diagnostics.
>
> Réflexe associé : vérifier l'identité effective avant d'interpréter toute
> sortie.

### Cause réelle

Lecture du fichier de préférence dans le partage du contrôleur :

```xml
groupName="Administrateurs (intÃ©grÃ©)"
<Member name="LAB\DL_Admin_Postes" action="ADD" sid=""/>
```

Deux anomalies :

- **Aucun attribut d'identifiant de sécurité pour le groupe cible.** La
  préférence désignait le groupe local par son nom littéral, et ce nom était
  corrompu par un problème d'encodage. L'extension recherchait un groupe
  inexistant sous ce nom, ne trouvait rien, et considérait n'avoir rien à
  faire.
- **Identifiant du membre vide.** La console n'avait pas résolu le groupe de
  domaine au moment de la saisie, signe que la valeur avait été tapée plutôt que
  sélectionnée.

### Correction

Recréation de l'élément en sélectionnant le groupe cible dans la liste proposée
et le membre via le sélecteur d'objets de l'annuaire, au lieu d'une saisie
manuelle.

Fichier obtenu après correction :

```xml
groupSid="S-1-5-32-544"
<Member name="LAB\DL_Admin_Postes" action="ADD" sid="S-1-5-21-...-1108"/>
```

L'identifiant `S-1-5-32-544` désigne universellement le groupe Administrateurs,
indépendamment de la langue du système. Le nom corrompu devient dès lors sans
conséquence.

### Enseignements

> **Quand tous les indicateurs affirment que le mécanisme devrait fonctionner,
> il faut descendre jusqu'à la donnée que l'outil manipule réellement.** Les
> couches d'abstraction affichent l'intention, pas le contenu. Le rapport
> généré par la console présentait une configuration correcte, alors que le
> fichier sous-jacent était inexploitable.
>
> **Sélectionner plutôt que saisir.** Une valeur choisie dans un sélecteur
> d'objets est résolue et accompagnée de son identifiant. Une valeur tapée reste
> une chaîne de caractères, sujette aux fautes de frappe et aux problèmes
> d'encodage.
>
> **Un traitement déclaré réussi ne garantit pas un résultat.** L'extension
> avait effectivement terminé son travail : elle n'avait simplement rien trouvé
> à faire. Vérifier l'effet, non le code de retour.
>
> Quatrième occurrence dans ce projet d'une configuration écrite, valide en
> apparence et dépourvue d'effet. Les précédentes concernaient un objet de
> stratégie lié à une unité d'organisation sans utilisateur, des directives de
> résolution de noms sans le paquet requis, et une directive d'imbrication de
> groupes non reconnue par le module qui la lisait.

### Incident non expliqué

Plusieurs actualisations de stratégie ont expiré sur la partie ordinateur, sans
qu'aucun événement de fin ne soit journalisé. Le comportement a cessé de
lui-même et n'a pas été reproduit.

Hypothèses plausibles, aucune vérifiée : durée accrue de la première application
d'une nouvelle extension, ou contention de ressources sur l'hôte de
virtualisation.

> Une panne transitoire non reproduite demeure une panne non expliquée. Le
> consigner comme tel est préférable à retenir l'hypothèse la plus commode.

---

## Gestion des mots de passe administrateurs locaux

### Problème traité

Sur un parc déployé par image, le compte administrateur local partage le même
mot de passe sur toutes les machines. La compromission d'un poste permet d'en
extraire le condensat et de le rejouer sur les autres machines, sans même avoir
à le casser.

C'est le vecteur de mouvement latéral le plus courant, et il transforme la
compromission d'un poste en compromission du parc.

### Solution

Chaque poste génère son propre mot de passe aléatoire, l'écrit dans son objet
ordinateur de l'annuaire, et le renouvelle périodiquement.

Le mécanisme est intégré au système depuis 2023 et ne requiert aucun agent,
contrairement à la génération précédente.

### Extension de schéma, opération irréversible

L'ajout des attributs au schéma de la forêt ne se défait pas : un attribut de
schéma se désactive au mieux, ne se supprime jamais.

En production, l'opération se planifie, se valide hors production et requiert
l'appartenance aux administrateurs du schéma. Sur une maquette, elle se réalise
en une commande, ce qui ne doit pas faire oublier sa portée.

### Asymétrie des permissions

Un poste dispose du droit d'écrire son propre mot de passe, et de rien d'autre.
Il ne peut ni relire sa propre valeur, ni consulter celle d'une autre machine.

C'est le moindre privilège appliqué à un compte machine, et cela explique
pourquoi la consultation s'effectue depuis l'annuaire et non depuis le poste.

### Refus d'un nom non qualifié

L'attribution du droit de lecture a été refusée sur un nom court, l'outil
exigeant un nom pleinement qualifié.

> Exigence justifiée : un nom court est ambigu dès lors que plusieurs domaines
> coexistent, et une résolution implicite pourrait accorder la lecture de mots
> de passe au mauvais principal. Sur une donnée de cette sensibilité, un refus
> explicite vaut mieux qu'une supposition.

### Deux couches d'autorisation

Le chiffrement du mot de passe introduit une séparation configurée à deux
endroits distincts : le droit de **lire** l'attribut, porté par les permissions
de l'unité d'organisation, et le droit de **déchiffrer** son contenu, porté par
un paramètre de stratégie.

Comportement vérifié avec un compte administrateur du domaine, qui dispose du
droit de lecture implicite sans appartenir au groupe déchiffreur :

```
Source              : EncryptedPassword
DecryptionStatus    : Unauthorized
AuthorizedDecryptor : LAB\DL_Admin_Postes
```

L'objet est retourné avec ses horodatages et le nom du déchiffreur autorisé,
mais sans valeur exploitable.

> **Enseignement.** Pouvoir accéder à une donnée et pouvoir l'utiliser sont deux
> choses séparables. Cette distinction se retrouve dans tout mécanisme reposant
> sur du chiffrement au repos : journaux, sauvegardes, coffres de secrets.
>
> Corollaire de configuration : le chiffrement seul est insuffisant. Sans
> désignation explicite des déchiffreurs, la valeur par défaut s'applique, et un
> groupe autorisé à lire pourrait n'obtenir qu'un contenu inexploitable.

### Actions de post-authentification

Le paramètre le plus significatif du dispositif. Après qu'un administrateur s'est
connecté avec le mot de passe, celui-ci est automatiquement renouvelé au terme
d'un délai de huit heures.

Un secret utilisé pour dépanner un poste devient inexploitable le jour même,
qu'il ait été noté, transmis ou intercepté. La durée de vie du secret est bornée
par son usage, non par un calendrier.

Comportement vérifié par rotation forcée : le mot de passe consulté n'est plus
valide après renouvellement.

---

## Convergence du modèle de groupes

Trois ressources de nature différente sont désormais gouvernées par la même
appartenance :

| Ressource | Mécanisme |
|---|---|
| Administration du pare-feu | RADIUS, autorisation par groupe |
| Administration locale des postes | Préférence de stratégie de groupe |
| Lecture des mots de passe administrateurs locaux | Permissions d'annuaire et stratégie |

Le retrait d'une personne de `GG_Informatique` supprime simultanément les trois
accès, sans intervention sur aucun équipement ni aucune machine.

C'est ce que promet le modèle AGDLP, constaté sur trois types de ressources
plutôt qu'énoncé.

---

## État à la fin de la session

- Administration locale des postes portée par le modèle de groupes
- Mots de passe administrateurs locaux uniques par machine, chiffrés dans
  l'annuaire, renouvelés après usage
- Séparation entre droit de lecture et droit de déchiffrement vérifiée
- Journalisation de diagnostic désactivée après résolution
- Étape 6 terminée

## Prochaine étape

Supervision et détection : centralisation des journaux, règles de détection, et
exploitation des sources d'événements produites par les étapes précédentes.
