# Session 11, référence de temps, audit des postes et sources de supervision

**Objectif.** Raccorder `PC01` et `FW01` comme sources du SIEM, puis implémenter
la détection 3 du plan, la lecture d'un mot de passe LAPS.

**Résultat.** `PC01` raccordé et vérifié. `FW01` raccordé jusqu'à l'analyse, sans
remontée en production. Détection 3 préparée côté annuaire, règle restant à
écrire. Un prérequis non prévu a occupé la première moitié de la session : la
référence de temps de la forêt était fausse.

---

## 1. Prérequis non prévu, la référence de temps

### Constat

La vérification des horloges avant raccordement a révélé que `DC01`, détenteur
des cinq rôles à opérations uniques dont l'émulateur de contrôleur principal,
s'annonçait comme référence de temps de couche 1 tout en lisant son horloge
matérielle.

```
Source : Local CMOS Clock
Couche : 1 (Référence principale)
Délai de racine : 0.0000000s
```

Le délai de racine nul est l'indicateur déterminant : il mesure le trajet
jusqu'à la référence ultime, et une valeur nulle signifie que la machine se
déclare elle-même comme cette référence.

La configuration du service était `Type: NT5DS`, c'est-à-dire synchronisation
sur la hiérarchie du domaine. Correcte pour tout contrôleur de domaine **sauf**
le détenteur de l'émulateur, qui est le sommet de cette hiérarchie et doit donc
pointer vers une source externe. C'est l'état par défaut après promotion, et il
n'est corrigé par personne sur un domaine à contrôleur unique.

### Conséquences

| Domaine | Effet |
|---|---|
| Kerberos | Tolérance de cinq minutes entre client et contrôleur. Toute la forêt dérive ensemble, donc l'écart reste invisible jusqu'au contact d'une source externe |
| Corrélation | `SRV01` et `FW01` étaient sur NTP, `DC01` non. Toute chronologie croisant Windows et Linux aurait comparé deux référentiels indépendants |
| Machine virtuelle | Les horloges d'invités dérivent plus vite que le matériel |

> Une infrastructure peut être cohérente en interne et fausse par rapport au
> temps réel. Un contrôleur de domaine sur `Local CMOS Clock` synchronise tout
> le domaine sur une horloge non référencée. L'écart ne se manifeste qu'au
> contact d'une source externe, et se présente alors comme un défaut
> d'authentification ou une corrélation incohérente, jamais comme un problème
> d'horloge.

### Hiérarchie retenue

```
pool NTP public
   └─ FW01 ................ 10.10.20.1, seule sortie NTP
        └─ DC01 ........... 10.10.20.10, émulateur PDC, référence du domaine
             ├─ SRV01
             ├─ SIEM01
             └─ PC01 ....... par la hiérarchie du domaine
```

Un seul point de sortie, une seule référence interne. Le trafic de `SRV01` et
`SIEM01` vers `DC01` reste à l'intérieur de VLAN 20, donc aucune règle de
filtrage n'est nécessaire.

### Vérification avant configuration

`w32tm /stripchart` interroge une cible en lecture seule, sans modifier ni
configuration ni horloge. Il teste d'un coup la règle de filtrage, l'écoute du
service et la réponse du serveur, ce qu'aucune vérification locale ne permet.

Conflit potentiel écarté par la mesure : les outils de virtualisation avaient
déjà leur synchronisation d'horloge désactivée, et le fournisseur de temps
`VMICTimeProvider` présent dans la configuration est celui de l'intégration
Hyper-V, donc inerte sous VMware. Un seul mécanisme était en jeu.

### Écueil rencontré

L'ajout de la règle de blocage NTP a coupé `SRV01` et `SIEM01`, dont la
synchronisation sortait directement vers un groupe public. L'autorisation ne
couvrait que `DC01`, et le blocage portait sur toute destination, pare-feu
inclus.

Le passage par `DC01` a résolu les deux problèmes en une seule modification,
sans toucher au jeu de règles.

> **`System clock synchronized: yes` est un indicateur rémanent.** Il reste vrai
> après une synchronisation passée et ne prouve pas qu'une synchronisation a
> lieu. Le contenu de `NTPMessage`, avec son compteur de paquets, ses
> horodatages et son champ de référence, est le fait.

Troisième indicateur de ce type rencontré en deux sessions, après
`Analyzing file` et le succès de `w32tm /config`.

---

## 2. Écarts relevés dans le jeu de règles de filtrage

L'analyse du jeu de règles exporté a fait apparaître quatre écarts, dont un
seul a été corrigé pendant la session.

| Écart | État |
|---|---|
| NTP autorisé à tout le segment SERVEURS vers n'importe quelle destination, alors que le DNS est réservé à `DC01` avec blocage explicite des autres | Corrigé, aligné sur le modèle DNS |
| Aucune règle IPv6 active. Le trafic tombe dans le refus implicite, sans décision documentée | Non corrigé |
| Blocage explicite et journalisé vers le pare-feu présent pour POSTES et INVITES, absent pour SERVEURS. Le refus existe par la règle implicite mais n'est pas tracé de la même façon | Non corrigé |
| Remontée des agents déclarée par adresse littérale, alors que le reste de la matrice emploie des alias | Non corrigé |

Deux règles DNS temporaires, marquées à supprimer depuis la mise en service de
l'annuaire, sont toujours présentes en état désactivé.

> Un écart de conception trouvé par relecture vaut une correction. La matrice
> énonce des décisions ; un comportement obtenu par refus implicite n'est pas
> une décision, c'est un effet de bord.

---

## 3. Raccordement de FW01

Le pare-feu n'admet pas d'agent, son système n'étant pas couvert. La collecte
passe donc par syslog.

Configuration réalisée :

- Émission par le service de journalisation distante, restreinte à
  l'application `filterlog` pour ne pas transmettre l'ensemble du journal
  système du pare-feu
- Format BSD historique conservé, celui que les décodeurs livrés reconnaissent
- Réception par un bloc `<remote>` de type syslog sur `SIEM01`, avec restriction
  de l'adresse source émettrice

### État par étage

| Étage | État | Preuve |
|---|---|---|
| Émission | Fonctionne | Capture réseau, paquets reçus, contenu conforme |
| Réception | Fonctionne | Socket UDP ouverte, adresse source autorisée déclarée au démarrage |
| Décodage | Fonctionne | Décodeur `pf`, sept champs extraits, règle native de niveau 5 |
| Analyse en production | **Ne fonctionne pas** | Aucune alerte du pare-feu produite |
| Indexation | Fonctionne pour les agents | Index du jour alimenté en continu |

Le diagnostic n'a pas abouti. Le point ouvert est délimité : un événement
identique produit une alerte en test hors ligne et aucune en production.

> **L'outil de test de journaux ne prouve pas la chaîne de production.** Il
> charge les règles pour son propre compte et ne fait intervenir ni l'agent, ni
> le transport, ni la file d'analyse, ni l'indexation. Un résultat correct en
> test est compatible avec une chaîne inopérante.

Observation de collecte : la journalisation du NTP sortant du pare-feu vers ses
serveurs publics produit un volume régulier sans valeur de détection. Candidat à
l'exclusion ou à un niveau nul.

---

## 4. Audit des postes

### Écart entre le document et la réalité

La session 9 documentait sept sous-catégories activées par stratégie de groupe
sur les contrôleurs de domaine. La vérification a montré qu'elles se trouvent
dans la stratégie par défaut du domaine, donc appliquées à **toutes** les
machines, postes compris.

Deux conséquences. D'une part, `PC01` bénéficiait déjà d'un audit avant cette
session, contrairement à ce que le document laissait entendre. D'autre part, des
sous-catégories propres aux contrôleurs de domaine, comme l'accès au service
d'annuaire ou les opérations de tickets Kerberos, sont actives sur les postes,
où elles produisent du volume sans objet.

> Modifier une stratégie par défaut plutôt que créer un objet dédié empêche de
> distinguer ce qui a été configuré de ce qui était livré, et rend la portée
> impossible à restreindre. Le moindre privilège s'applique aussi à la
> journalisation.

### Objet créé

`GPO_Postes_Audit`, lié à l'unité d'organisation des postes, distinct de
`GPO_Postes_Durcissement`. Audit et durcissement ont des cycles de vie
différents et doivent pouvoir être désactivés séparément.

| Catégorie | Sous-catégorie | Réglage | Objet |
|---|---|---|---|
| Ouverture/fermeture de session | Ouvrir la session | Succès et échec | Connexions et leur type |
| Ouverture/fermeture de session | Ouverture de session spéciale | Succès | Sessions à privilèges |
| Suivi détaillé | Créer un processus | Succès | Exécution sur le poste |
| Gestion des comptes | Groupes de sécurité | Succès et échec | Administrateurs locaux |
| Gestion des comptes | Comptes d'utilisateur | Succès et échec | Création de compte local |
| Accès aux objets | Partage de fichiers | Échec | Accès refusés |

### Arbitrage sur la ligne de commande

L'événement de création de processus n'indique par défaut que le programme
lancé, sans ses arguments. Sans eux, on constate qu'un interpréteur a démarré
sans savoir ce qu'il exécute, ce qui rend la détection d'exécution largement
inopérante. L'inclusion de la ligne de commande s'active par un réglage séparé.

Contrepartie assumée et documentée : tout secret passé en argument se retrouve
en clair dans le journal, donc dans le SIEM, lisible par quiconque y accède.

La démonstration a eu lieu pendant la session même. Deux commandes de création
et de suppression d'un compte local, employées comme donnée témoin, ont produit
des événements contenant le mot de passe en clair, indexés dans le SIEM dans la
minute.

> La valeur de détection et l'exposition de secrets sont deux faces du même
> réglage. Le choix se documente, et s'accompagne d'une règle de gestion sur les
> commandes qui acceptent un secret en argument.

### Vérification

`auditpol /get` interroge la configuration effective de la machine, pas la
stratégie. Les six sous-catégories sont actives sur `PC01`, et la présence de la
ligne de commande a été prouvée par un événement réel portant un marqueur unique,
non par la lecture de la clé de registre correspondante.

Détail de lecture : l'outil écrit « Réussite » pour un audit en succès seul et
« Succès et échec » lorsque les deux sont actifs. Un filtre textuel sur un seul
de ces libellés donne une vue incomplète sans avertissement.

---

## 5. Raccordement de PC01

Agent installé et enrôlé, visible comme actif côté gestionnaire, ce qui
constitue la preuve de connexion et non le seul démarrage du service local. Le
flux inter-VLAN ouvert par anticipation lors de l'étape 3 a été emprunté sans
modification.

Témoin de vérification : création puis suppression d'un compte local, avec un
marqueur choisi pour ne pouvoir apparaître ni dans une commande
d'administration ni dans un identifiant technique.

Alertes produites :

| Règle | Niveau | Événement |
|---|---|---|
| 60111 | 8 | Compte désactivé ou supprimé |
| 60160 | 5 | Modification du groupe Utilisateurs du domaine |
| 60170 | 5 | Modification du groupe Utilisateurs |
| 67027 | 3 | Création de processus |

La création d'un compte local ne produit pas d'alerte de niveau comparable à sa
suppression. Candidat à une règle locale, la création d'un compte sur un poste
étant au moins aussi significative que sa suppression.

---

## 6. Détection 3, lecture d'un mot de passe LAPS

### Nature de l'événement

Il n'existe pas d'événement propre à LAPS. Une lecture de mot de passe est une
**lecture d'attribut** sur un objet ordinateur, donc un accès au service
d'annuaire, événement 4662.

Version en place identifiée par le schéma, et non supposée : Windows LAPS avec
attributs chiffrés. Les deux attributs surveillés sont `ms-LAPS-Password` et
`ms-LAPS-EncryptedPassword`.

### Le mécanisme a deux niveaux

> **L'audit d'accès à l'annuaire fonctionne à deux niveaux indépendants.** La
> sous-catégorie autorise la journalisation, la liste d'audit de l'objet la
> déclenche. Activer la première seule produit un coût en volume sans aucune
> couverture.

Constat de départ : la sous-catégorie était active depuis la session 9, avec son
coût documenté, et aucune entrée d'audit ne visait les attributs LAPS. Les
entrées présentes étaient héritées du domaine, en écriture, et portaient sur des
objets de stratégie de groupe.

### Entrées d'audit posées

Sur l'unité d'organisation des postes, héritées aux objets ordinateurs :

| Paramètre | Valeur | Raison |
|---|---|---|
| Principal | Tout le monde | On veut savoir qui lit, administrateurs compris |
| Type | Réussite | L'échec produit du bruit sur les accès normaux |
| Droit | Lecture de propriété | C'est la lecture qu'on détecte |
| Portée | Objets Ordinateur descendants | Évite les objets utilisateurs et groupes |
| Attributs | Les deux attributs LAPS uniquement | Sans restriction, tout accès à tout attribut serait audité |

La restriction aux deux attributs est le point qui détermine le volume. C'est le
traitement concret de l'arbitrage laissé ouvert en session 9.

Héritage vérifié sur l'objet `PC01` lui-même, l'entrée sur l'unité ne suffisant
pas à garantir son application.

### Résultat du témoin

Deux lectures ont été effectuées, avec deux comptes différents.

| Compte | Déchiffrement | Événement 4662 |
|---|---|---|
| Compte nominatif hors du groupe autorisé | Refusé | Produit |
| Compte administrateur intégré, suffixe `-500` | Réussi | Produit |

> **La lecture de l'attribut et son déchiffrement sont deux opérations
> distinctes.** L'audit journalise la première, indépendamment du succès de la
> seconde. Une détection fondée sur cet événement voit donc la tentative, y
> compris lorsqu'elle n'aboutit pas. C'est ce qui en fait une détection utile
> plutôt qu'un journal d'usage.

C'est le cas qui compte en sécurité opérationnelle : un attaquant sans droit de
déchiffrement produit exactement la même trace qu'un administrateur autorisé.

L'une des lectures provient du compte administrateur intégré, dont le document
de supervision explique la surveillance particulière. Le champ identifiant la
session de connexion permet de regrouper les lectures d'une même session.

### Point ouvert

Le nom de l'objet cible apparaît sous forme d'identifiant global et non sous le
nom de la machine, ce dont la description de la règle devra tenir compte.

Aucune règle livrée ne couvre l'événement 4662. La recherche dans le jeu de
règles ne retourne que des correspondances fortuites. La détection doit donc
être écrite entièrement, avec une condition portant sur l'identifiant de
l'attribut lu.

Une règle d'observation temporaire a été posée pour inspecter la structure de
l'événement décodé. Elle ne s'est pas déclenchée, ni avec une condition sur le
champ d'identifiant d'événement, ni accrochée à la règle mère du canal de
sécurité. La cause n'est pas établie.

Vérifications faites au cours de ce diagnostic, toutes négatives :

- L'agent du contrôleur de domaine est connecté et remonte normalement, plus de
  soixante-dix alertes dans l'heure
- Le bloc de collecte du canal de sécurité n'exclut pas cet identifiant
- La configuration partagée poussée aux agents est vide, donc la configuration
  locale s'applique
- L'événement est bien produit sur le contrôleur, y compris après le dernier
  redémarrage du gestionnaire

---

## 7. Motifs de méthode observés

### Correspondance fortuite en recherche textuelle

Quatre fois dans la session, une recherche d'identifiant numérique en texte
libre dans le fichier d'alertes a retourné des documents sans rapport. Deux
causes distinctes, également trompeuses :

- Les identifiants internes contiennent des séquences numériques arbitraires.
  Un identifiant d'alerte peut contenir le numéro d'événement recherché
- Les commandes d'administration sont journalisées avec leurs arguments. Une
  recherche pour un identifiant retourne la commande de recherche elle-même,
  et le compteur augmente à chaque tentative

Le second cas est le plus dangereux : il produit un compteur qui croît à chaque
vérification, ce qui ressemble exactement à un flux en temps réel.

> **Ne jamais rechercher un identifiant ou une valeur numérique par
> correspondance de texte libre dans un fichier d'alertes.** Rechercher sur le
> champ. Le principe formalisé en session 9 sur les recherches non concluantes
> s'applique aussi aux recherches qui retournent trop.

Corollaire découvert : au delà d'une certaine taille, l'outil de recherche traite
le fichier d'alertes comme binaire et cesse d'afficher les correspondances. Les
comptages obtenus sans l'option adéquate sont faux sans message d'erreur.

### Latence de chaîne

Entre l'événement source et sa disponibilité en recherche, il y a la collecte,
l'analyse, l'écriture fichier, la lecture par l'expéditeur et l'indexation. Un
comptage lancé immédiatement après l'injection d'un témoin retourne zéro sans
que cela signifie quoi que ce soit.

### Contexte d'exécution

Deux occurrences supplémentaires. Un test de connectivité lancé depuis la
machine cible elle-même, dont le paquet n'a jamais traversé le pare-feu. Un bloc
de collecte déclaré sur le serveur de supervision pour un fichier qui n'existe
que sur le serveur applicatif.

Ce dernier a par ailleurs illustré un motif inverse de celui habituellement
rencontré : le bloc avait été commenté avec un caractère sans valeur en XML, ce
qui laissait la déclaration active. Une configuration crue désactivée et
toujours en effet.

---

## 8. Composition du volume

Premier relevé par identifiant d'événement, utile au dimensionnement de la
rétention.

| Identifiant | Part | Nature |
|---|---|---|
| 4624 | 524 | Ouvertures de session, l'essentiel du volume |
| 4688 | 249 | Créations de processus, depuis le raccordement de `PC01` |
| 4769 | 19 | Opérations de tickets Kerberos |
| Autres | moins de 60 | Événements applicatifs et système |

Deux identifiants représentent plus de quatre-vingt-dix pour cent des alertes.
Toute politique de rétention se dimensionne à partir de ces deux familles.

Facteurs d'amplification constatés, à ne pas négliger : une tentative
d'authentification SSH produit deux alertes, un test de connexion TCP refusé
produit une ligne par retransmission, soit cinq.

---

## État à la fin de la session

- Référence de temps unique et vérifiée sur les quatre machines
- Règles NTP alignées sur le modèle du DNS, avec blocage journalisé
- Stratégie d'audit dédiée aux postes, six sous-catégories, ligne de commande
  incluse, vérifiée sur la configuration effective
- `PC01` raccordé, alertes indexées, vérifié par donnée témoin
- `FW01` émettant, reçu et décodable, sans remontée en production
- Entrées d'audit LAPS posées, héritage vérifié, événement produit et confirmé
  pour une lecture refusée comme pour une lecture réussie

## Reste à traiter

- Règle de détection sur l'événement 4662 filtrée sur l'identifiant d'attribut
  LAPS, et retrait de la règle d'observation temporaire
- Diagnostic de la non-remontée des événements du pare-feu en production
- Règles IPv6 explicites et journalisées sur les trois interfaces internes
- Politique de rétention des index et arbitrage sur les archives brutes
- Déplacement des réglages d'audit hors de la stratégie par défaut du domaine,
  vers un objet dédié aux contrôleurs de domaine
- Suppression des règles de filtrage temporaires et de l'objet de stratégie
  résiduel issu des exercices de diagnostic
- Détections 2 et 4 du plan
