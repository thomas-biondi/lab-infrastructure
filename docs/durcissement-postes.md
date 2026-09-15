# Durcissement des postes de travail

> Description des mécanismes d'administration locale et de gestion des secrets
> sur les postes du domaine. Le raisonnement figure dans
> `journal/session-08-durcissement-postes.md`.

---

## 1. Administration locale par appartenance de groupe

### Principe

Aucun compte n'est ajouté individuellement au groupe Administrateurs local d'un
poste. L'appartenance est portée par un groupe de domaine, lui-même alimenté par
un groupe global décrivant la fonction.

```
tbiondi  ->  GG_Informatique  ->  DL_Admin_Postes  ->  Administrateurs local
```

Une personne rejoignant le service informatique devient administratrice de
l'ensemble du parc par une seule appartenance. Son départ retire l'accès
partout, sans intervention sur aucune machine.

### Mise en œuvre

Préférence de stratégie de groupe, dans `GPO_Postes_Durcissement`, liée à
`OU=Postes`.

| Paramètre | Valeur |
|---|---|
| Action | Mettre à jour |
| Groupe cible | Administrateurs intégré, identifié par `S-1-5-32-544` |
| Membre ajouté | `LAB\DL_Admin_Postes` |
| Suppression des membres existants | Désactivée |

**Le ciblage par identifiant de sécurité est indispensable.** Le groupe
Administrateurs porte un nom différent selon la langue du système. Une
préférence désignant le groupe par son nom littéral échoue sur tout poste dont
la langue diffère, sans message d'erreur.

Les options de suppression des membres existants sont volontairement
désactivées. Les activer figerait la composition du groupe, ce qui constitue la
pratique recommandée en production, mais suppose de lister exhaustivement tous
les membres légitimes, comptes de secours compris.

---

## 2. Gestion des mots de passe administrateurs locaux

### Problème traité

Sur un parc déployé par image, le compte administrateur local partage le même
mot de passe sur toutes les machines. Un attaquant compromettant un poste
extrait le condensat de ce mot de passe et le rejoue sur les autres machines
sans avoir besoin de le casser.

Cette technique transforme la compromission d'un poste en compromission du parc.
C'est le vecteur de mouvement latéral le plus courant.

### Solution retenue

Windows LAPS, intégré nativement au système depuis 2023, sans agent à déployer.

Chaque poste génère lui-même un mot de passe aléatoire pour son compte
administrateur local, l'écrit dans son propre objet ordinateur de l'annuaire, et
le renouvelle périodiquement.

| Conséquence | Effet |
|---|---|
| Mot de passe unique par machine | Le rejeu sur une autre machine est impossible |
| Stockage dans l'annuaire | L'accès est contrôlé par les permissions et traçable |
| Rotation automatique | Un secret divulgué a une durée de vie bornée |

### Extension de schéma

L'ajout des attributs au schéma de la forêt est **irréversible**. Un attribut de
schéma ne se supprime pas, il se désactive au mieux. En production, l'opération
se planifie, se teste hors production et requiert l'appartenance aux
administrateurs du schéma.

Attributs ajoutés :

| Attribut | Objet |
|---|---|
| `ms-LAPS-Password` | Mot de passe en clair |
| `ms-LAPS-EncryptedPassword` | Mot de passe chiffré |
| `ms-LAPS-EncryptedPasswordHistory` | Historique chiffré |
| `ms-LAPS-PasswordExpirationTime` | Échéance de rotation |
| `ms-LAPS-CurrentPasswordVersion` | Version courante |
| `ms-LAPS-EncryptedDSRMPassword` | Mot de passe de restauration d'annuaire |

### Permissions

| Portée | Principal | Droit |
|---|---|---|
| `OU=Postes` | Comptes machines | Écriture de leur propre mot de passe |
| `OU=Postes` | `LAB\DL_Admin_Postes` | Lecture du mot de passe |

Un poste peut écrire dans son objet et rien d'autre. Il ne peut pas relire son
propre mot de passe, ni consulter celui d'une autre machine. Le moindre
privilège appliqué à un compte machine.

Les administrateurs du domaine et de l'entreprise disposent du droit de lecture
de façon implicite.

### Configuration de la stratégie

| Paramètre | Valeur |
|---|---|
| Répertoire de sauvegarde | Active Directory |
| Compte géré | Compte administrateur intégré, par défaut |
| Longueur | 20 caractères |
| Complexité | Majuscules, minuscules, chiffres, caractères spéciaux |
| Âge maximal | 30 jours |
| Chiffrement du mot de passe | Activé |
| Déchiffreurs autorisés | `LAB\DL_Admin_Postes` |
| Actions de post-authentification | Réinitialisation et fermeture de session, délai 8 heures |
| Délai d'expiration supérieur au requis | Interdit |

**Les actions de post-authentification constituent le paramètre le plus
important.** Après qu'un administrateur s'est connecté avec le mot de passe,
celui-ci est automatiquement renouvelé au terme du délai. Un secret utilisé pour
dépanner un poste devient inexploitable le jour même, qu'il ait été noté,
transmis ou intercepté.

Le compte géré est désigné par son identifiant de sécurité et non par son nom,
ce qui rend la configuration indépendante de la langue du système.

---

## 3. Deux couches d'autorisation distinctes

Le chiffrement du mot de passe introduit une séparation qui se configure à deux
endroits différents :

| Couche | Emplacement | Effet |
|---|---|---|
| Droit de lecture | Permissions de l'unité d'organisation | Obtenir l'attribut |
| Droit de déchiffrement | Paramètre de stratégie | Exploiter son contenu |

Un principal disposant de la première sans la seconde obtient l'objet, ses
horodatages et le nom du déchiffreur autorisé, mais aucune valeur exploitable :

```
Source              : EncryptedPassword
DecryptionStatus    : Unauthorized
AuthorizedDecryptor : LAB\DL_Admin_Postes
```

**Comportement vérifié avec un compte administrateur du domaine**, qui dispose
du droit de lecture implicite mais n'appartient pas au groupe déchiffreur : la
lecture aboutit, le déchiffrement est refusé.

Cette séparation entre accéder à une donnée et pouvoir l'utiliser se retrouve
dans tout mécanisme reposant sur du chiffrement au repos : journaux, sauvegardes,
coffres de secrets.

---

## 4. Validation

| Test | Résultat |
|---|---|
| Rattachement du groupe d'administration au groupe local | Conforme |
| Élévation sans saisie d'identifiants pour un membre du groupe | Conforme |
| Génération et écriture du mot de passe dans l'annuaire | Conforme |
| Lecture sans droit de déchiffrement | Refus, objet retourné sans valeur |
| Lecture par un membre du groupe déchiffreur | Mot de passe obtenu |
| Rotation forcée | Nouveau mot de passe généré |

---

## 5. Points ouverts

### Comptes à privilèges élevés sur les postes

Le groupe Administrateurs local des postes contient `LAB\Admins du domaine`,
ajouté automatiquement lors de la jonction au domaine.

Un compte à privilèges élevés peut donc s'authentifier sur un poste de travail
et y laisser un élément de session exploitable en cas de compromission de la
machine. C'est précisément ce que le modèle de tiering cherche à empêcher : un
compte ne s'authentifie que sur des machines d'un niveau de confiance
équivalent au sien.

Le retrait de ce groupe constitue le durcissement suivant, rendu possible par
LAPS qui fournit désormais un accès administrateur local par machine sans
recourir à un compte de domaine.

### Rotation du mot de passe de restauration d'annuaire

Le paramètre de sauvegarde des comptes de restauration des services d'annuaire
n'est pas activé, la stratégie étant liée aux postes et non aux contrôleurs de
domaine.

Il traiterait le mot de passe de restauration défini une seule fois à la
promotion du contrôleur, et jamais renouvelé depuis. C'est un secret que la
plupart des organisations posent au déploiement et n'ont jamais fait tourner.

### Gestion automatique d'un compte dédié

Windows LAPS sait créer et gérer un compte d'administration local dédié plutôt
que le compte intégré, dont l'identifiant de sécurité est universellement connu
et donc énumérable. Non activé afin de ne pas multiplier les variables lors de
la validation du mécanisme de base.
