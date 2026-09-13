# DC01, contrôleur de domaine et annuaire

> Document de référence décrivant l'état courant de `DC01` et de la forêt
> `lab.internal`. Le raisonnement figure dans
> `journal/session-04-active-directory.md`.

---

## 1. Identité

| Élément | Valeur |
|---|---|
| Rôle | Contrôleur de domaine, DNS de la forêt, catalogue global |
| Système | Windows Server 2025 Standard, Desktop Experience |
| Nom d'hôte | `DC01` |
| Adresse | `10.10.20.10/24`, statique |
| Passerelle | `10.10.20.1` |
| Segment | VLAN 20 SERVEURS |
| Ressources | 4 Go de mémoire, 2 processeurs virtuels, disque de 60 Go |

Le renommage de la machine et la fixation de son adresse ont été réalisés
**avant** la promotion. Le nom d'un contrôleur de domaine se retrouve dans
l'annuaire, dans la zone DNS et dans les enregistrements SRV : le modifier
après coup est possible mais délicat.

Une adresse fixe est obligatoire. Les autres machines localisent le contrôleur
par des enregistrements DNS pointant sur cette adresse.

---

## 2. Forêt et domaine

| Élément | Valeur |
|---|---|
| Nom de forêt | `lab.internal` |
| Domaine racine | `lab.internal` |
| Nom NetBIOS | `LAB` |
| Niveau fonctionnel de forêt | `Windows2025Forest` |
| Niveau fonctionnel de domaine | `Windows2025Domain` |
| Site | `Default-First-Site-Name` |

Le suffixe `.internal` est réservé par l'ICANN aux usages strictement internes.
`.local` a été écarté en raison de son conflit avec le protocole mDNS.

Le niveau fonctionnel le plus élevé a été retenu, aucun système antérieur
n'ayant vocation à rejoindre la forêt. Un niveau fonctionnel se monte et ne se
redescend pratiquement jamais : le choix engage.

### Rôles portés

`DC01` étant l'unique contrôleur, il porte l'ensemble des rôles à maître unique :

| Rôle FSMO | Portée |
|---|---|
| Maître de schéma | Forêt |
| Maître d'attribution des noms de domaine | Forêt |
| Émulateur de contrôleur principal | Domaine |
| Maître RID | Domaine |
| Maître d'infrastructure | Domaine |

Il assure également le catalogue global et fait autorité sur le temps pour
l'ensemble de la forêt. Kerberos rejetant toute authentification au-delà de cinq
minutes d'écart, la dérive d'horloge de cette machine affecte le domaine entier.
Point de vigilance sur une machine virtuelle susceptible d'être suspendue.

---

## 3. Résolution de noms

| Paramètre | Valeur |
|---|---|
| Serveurs DNS de `DC01` | `10.10.20.10`, puis bouclage |
| Redirecteurs | `1.1.1.1` |
| Indications de racine en secours | Activées |

Un contrôleur de domaine doit se résoudre lui-même. La configuration initiale
issue de la promotion pointait sur le bouclage seul, puis les redirecteurs
reprenaient l'adresse du pare-feu.

**Le redirecteur a été repointé directement vers un résolveur public.** Le
conserver vers `FW01` aurait créé une dépendance en cascade rompue par la
fermeture du flux DNS entre le segment SERVEURS et le pare-feu, prévue par la
matrice de flux.

### Bascule DNS réalisée

La séquence définie dans `matrice-de-flux.md` a été exécutée dans l'ordre :

1. Vérification de la résolution externe depuis `DC01`
2. Modification du serveur DNS annoncé par Kea sur `10.10.10.0/24` et
   `10.10.20.0/24`, vers `10.10.20.10`
3. Suppression des deux règles `TEMPORAIRE` autorisant le DNS vers `FW01`

Le sous-réseau `10.10.30.0/24` conserve `10.10.30.1`.

---

## 4. Structure de l'annuaire

```
DC=lab,DC=internal
├── OU=Domain Controllers          (créée par la promotion)
└── OU=LAB
    ├── OU=Utilisateurs
    ├── OU=Ordinateurs
    │   ├── OU=Postes
    │   └── OU=Serveurs
    ├── OU=Groupes
    │   ├── OU=Global
    │   └── OU=Local
    └── OU=Comptes-de-service
```

**Les conteneurs `CN=Users` et `CN=Computers` ne sont pas utilisés.** Un
conteneur n'accepte pas de lien de stratégie de groupe : tout objet qui y reste
échappe à toute GPO hors niveau domaine. C'est la première cause du symptôme
« la stratégie ne s'applique pas ».

Le conteneur racine `OU=LAB` isole les objets créés de ceux que Windows
maintient, ce qui simplifie les délégations et les sauvegardes sélectives.

La séparation entre postes et serveurs est structurante : leurs stratégies n'ont
aucun paramètre en commun.

Les comptes de service sont isolés car ils ne se gèrent pas comme des comptes de
personnes et constituent une cible privilégiée des attaques Kerberoasting.

Les unités d'organisation créées sont protégées contre la suppression
accidentelle, comportement par défaut de `New-ADOrganizationalUnit`.

---

## 5. Groupes, modèle AGDLP

Un compte entre dans un groupe global, lequel entre dans un groupe de domaine
local, lequel reçoit la permission sur la ressource.

L'intérêt est de séparer deux questions dont les rythmes de changement sont
indépendants : **qui est cette personne dans l'organisation**, décrit par le
groupe global, et **qui a le droit de faire ceci sur cette ressource**, décrit
par le groupe de domaine local. Un changement d'affectation se traduit alors par
la modification d'une seule appartenance, sans toucher aux ressources.

### Groupes globaux

| Nom | Objet |
|---|---|
| `GG_Informatique` | Personnel du service informatique |
| `GG_Comptabilite` | Personnel du service comptabilité |
| `GG_Direction` | Direction |

### Groupes de domaine local

| Nom | Objet | Membre |
|---|---|---|
| `DL_Partage_Compta_RW` | Écriture sur le partage comptabilité | `GG_Comptabilite` |
| `DL_Partage_Compta_RO` | Lecture seule sur le partage comptabilité | `GG_Direction` |
| `DL_Admin_Postes` | Administration locale des postes | `GG_Informatique` |

**Sens d'imbrication** : le groupe global est membre du groupe de domaine local.
L'inverse ne produit aucune erreur, seulement des droits inopérants.

**Convention de nommage** : le droit figure dans le nom du groupe de domaine
local (`RW`, `RO`), afin qu'une liste de permissions soit lisible sans ouvrir
les propriétés de la ressource.

### Rappel sur les portées

| Portée | Peut contenir | Reçoit des permissions |
|---|---|---|
| Globale | Comptes du domaine propre | Dans toute la forêt |
| Domaine local | Membres de toute la forêt | Dans le domaine propre |
| Universelle | Toute la forêt | Dans toute la forêt, répliquée au catalogue global |

---

## 6. Comptes

| Compte | Groupe | Usage |
|---|---|---|
| `mdupont` | `GG_Comptabilite` | Utilisateur |
| `pmartin` | `GG_Direction` | Utilisateur |
| `tbiondi` | `GG_Informatique` | Utilisateur quotidien |
| `a-tbiondi` | aucun | Administration, usage non quotidien |

**Séparation des usages.** Un compte quotidien est exposé en permanence au
courriel et à la navigation. S'il porte des privilèges d'administration, une
seule compromission suffit à atteindre la forêt entière. Le compte
d'administration est distinct et n'est pas membre des groupes fonctionnels : il
reçoit ses droits par délégation explicite.

Les mots de passe ont été saisis via `Read-Host -AsSecureString`, l'historique
PowerShell étant conservé sur disque.

---

## 7. Délégation

| Objet délégué | Bénéficiaire | Droit accordé |
|---|---|---|
| `OU=Utilisateurs` | `a-tbiondi` | Réinitialisation des mots de passe utilisateur et exigence de changement à la prochaine ouverture de session |

La délégation se traduit par une entrée de contrôle d'accès posée sur l'unité
d'organisation et héritée par les objets qu'elle contient. Aucun privilège
d'administration du domaine n'est accordé.

Ce mécanisme est également celui qu'exploitent les chemins d'élévation de
privilèges en environnement Active Directory : une délégation trop large ou un
droit générique accordé par commodité devient un vecteur. Les outils de
cartographie offensive ne font que parcourir ces listes à la recherche de
chaînes exploitables.

---

## 8. Stratégies de groupe

| Objet | Lien | Contenu |
|---|---|---|
| `Default Domain Policy` | Domaine | Politique de mot de passe et de verrouillage |
| `GPO_Postes_Durcissement` | `OU=Postes` | Bannière d'avertissement à l'ouverture de session |
| `GPO_Test_Erreur` | `OU=Postes` | Objet volontairement mal conçu, support d'exercice |

### Politique de mot de passe

| Paramètre | Valeur |
|---|---|
| Longueur minimale | 14 caractères |
| Durée de vie maximale | 365 jours |
| Historique conservé | 24 mots de passe |
| Complexité | Activée |
| Seuil de verrouillage | 10 tentatives |
| Durée de verrouillage | 15 minutes |

Cette politique s'applique **au niveau du domaine uniquement** : elle est portée
par l'objet domaine et ne peut être différenciée par unité d'organisation sans
recourir aux stratégies de mot de passe affinées. C'est l'un des rares cas
légitimes de modification de la `Default Domain Policy`.

La durée de vie longue est un choix assumé et conforme aux recommandations
actuelles de l'ANSSI comme du NIST : l'expiration fréquente pousse les
utilisateurs vers des variantes prévisibles. La longueur et la détection de
compromission sont privilégiées.

### Bannière d'ouverture de session

Deux paramètres définis dans `Paramètres de sécurité → Stratégies locales →
Options de sécurité`. Au-delà de l'exercice, une bannière d'avertissement
affichée avant authentification est exigée par plusieurs référentiels : elle
retire l'argument de la méconnaissance du caractère restreint de l'accès.

### Objet d'exercice

`GPO_Test_Erreur` contient exclusivement un paramètre de configuration
utilisateur, alors que son lien porte sur une unité d'organisation ne contenant
que des comptes ordinateurs. Le lien est actif, l'objet est téléchargé, et la
moitié utilisateur est ignorée sans erreur ni avertissement.

Cet objet sert de support à l'exercice de diagnostic prévu à la mise en service
de `PC01`.

### Ordre de priorité constaté sur `OU=Postes`

1. `GPO_Postes_Durcissement`
2. `GPO_Test_Erreur`
3. `Default Domain Policy`

L'ordre d'application chronologique est l'inverse de l'ordre de priorité : le
domaine s'applique en premier, l'unité d'organisation écrase ensuite. Plus le
point de liaison est proche de l'objet, plus sa priorité est forte.

---

## 9. Points ouverts

`FW01` porte un nom dans `lab.internal` sans être membre du domaine, et cet
enregistrement n'existe pas dans la zone DNS. Le joindre par son nom depuis un
poste supposerait la création manuelle de l'enregistrement.

L'ordre des serveurs DNS sur la carte réseau place le bouclage IPv6 avant
l'adresse IPv4. Sans conséquence fonctionnelle, mais l'ordre inverse serait plus
rigoureux.

---

## 10. Instantanés de référence

| Nom | État |
|---|---|
| `DC01 installe, a jour, avant promotion` | Système seul |
| `DC01 promu, DNS bascule` | Forêt créée, résolution fonctionnelle |
| `DC01 annuaire structure` | Unités d'organisation, groupes, comptes |
| `DC01 annuaire et GPO` | État courant |
