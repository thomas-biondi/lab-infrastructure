# Session 10, vérification de la chaîne de collecte et hiérarchisation

**Objectif.** Lever le point ouvert de la session 9 sur la cohérence de la
collecte `fail2ban`, puis vérifier la chaîne de bout en bout avant d'étendre le
périmètre de collecte à `PC01` et `FW01`.

**Résultat.** Point ouvert levé, chaîne vérifiée de la source jusqu'à la
recherche, règle `fail2ban` hiérarchisée après trois itérations. Le raccordement
des nouvelles sources et la politique de rétention sont reportés : la session a
révélé assez d'écarts pour justifier de ne pas étendre avant d'avoir corrigé.

---

## Principe de la session

Étendre une collecte avant d'avoir vérifié la chaîne existante revient à ajouter
des sources à un dispositif dont on ignore s'il fonctionne. La session a donc
été construite à l'envers de l'intuition : on ne branche rien tant que ce qui
est déjà branché n'est pas prouvé.

> Une vérification préalable qui ne trouve rien coûte une heure. Une extension
> posée sur une chaîne défaillante coûte un diagnostic à deux variables
> inconnues.

---

## 1. Cohérence de la collecte fail2ban

Le point ouvert portait sur une configuration reconfigurée plusieurs fois entre
écriture fichier et écriture syslog, sans certitude sur l'état final.

Trois étages vérifiés indépendamment, chacun par un fait et non par une
déclaration :

| Étage | Vérification retenue | Constat |
|---|---|---|
| Sortie de `fail2ban` | Descripteur de fichier ouvert en écriture par le processus | `/var/log/fail2ban.log`, descripteur 3 |
| Collecte de l'agent | Descripteur de fichier ouvert en lecture par `wazuh-logcollector` | Même chemin, descripteur 7 |
| Décodage | Test de journal sur une ligne réelle | Décodeur `fail2ban-local` sélectionné |

Les trois concordent. La voie syslog explorée en session 9 a bien été
abandonnée, et aucune déclaration concurrente ne subsiste dans la configuration
poussée par le gestionnaire.

> **Le descripteur de fichier ouvert est la preuve de lecture.** La mention
> `Analyzing file` du journal de l'agent signifie que la source est enregistrée,
> pas qu'elle est lue. Le contenu de `/proc/<pid>/fd` tranche entre les deux, et
> c'est la vérification à retenir pour tout diagnostic de collecte.

Observation annexe : la prison `sshd` lit le journal systemd et non un fichier,
ce que confirme un descripteur pointant vers `system.journal`. Cohérent avec
l'absence de `rsyslog` sur Debian 13.

---

## 2. Erreurs horodatées et séquence de démarrage

Le journal du gestionnaire portait quatorze échecs d'initialisation de
l'indexeur et des messages de synchronisation d'agents en échec. Lecture initiale :
panne du dispositif d'indexation.

La confrontation des horodatages a donné une autre lecture :

```
22:55:32   gestionnaire démarré
22:55:36   échecs d'initialisation de l'indexeur
22:55:50   indexeur réellement démarré
23:08:18   gestionnaire redémarré, indexeur déjà actif
```

Le gestionnaire a cherché l'indexeur pendant les quatorze secondes de démarrage
de sa machine virtuelle Java. Le message le disait lui-même en annonçant de
nouvelles tentatives jusqu'au succès. Après le redémarrage de 23:08, aucune
occurrence.

> **Un journal conserve les erreurs passées.** Une erreur horodatée n'est pas une
> erreur actuelle. Avant de conclure à une panne, dater l'erreur et la confronter
> à la séquence de démarrage des services concernés.

---

## 3. Vérification de la chaîne par donnée témoin

Premier témoin injecté par `logger` sur `SRV01` avec une chaîne unique.
Recherche dans l'index : aucun résultat.

Ce résultat n'établit pas une défaillance. La chaîne ne stocke que ce qui
déclenche une règle, et aucune règle ne correspondait à cette ligne arbitraire.
Le témoin testait donc simultanément la collecte, l'existence d'une règle et la
recherche, sans permettre de distinguer laquelle était en cause.

> **Une donnée témoin n'est valide que si elle déclenche une règle connue.** Sur
> un dispositif qui n'indexe que ce qui correspond, un témoin arbitraire ne
> distingue pas l'absence de collecte de l'absence de règle. Le témoin se choisit
> parmi les événements dont on sait qu'ils alertent.

Second témoin : tentative d'authentification SSH avec un nom d'utilisateur unique
et inexistant, qui déclenche une règle native de niveau 5. Résultat immédiat dans
l'index et dans l'interface.

Chaîne vérifiée de bout en bout : production de l'événement, collecte par
l'agent, transmission au gestionnaire, décodage, correspondance de règle,
indexation, recherche.

---

## 4. Facteur d'amplification d'un événement

Le témoin SSH a produit **deux** alertes pour une seule tentative, à la
milliseconde près, même règle et même agent.

Vérification avant conclusion : lecture du journal brut des deux documents. Les
textes diffèrent. Le serveur SSH produit deux lignes distinctes pour une même
tentative, une pour l'utilisateur inexistant et une pour la fermeture de la
connexion, et les deux correspondent à la même règle.

Il ne s'agit donc pas d'une double collecte mais du comportement propre de la
source. La conséquence est un facteur d'amplification de deux sur cette famille
d'événements.

> Le nombre d'alertes produites par un événement n'est pas de un par défaut. Un
> dimensionnement de volume établi en comptant les événements, et non les
> alertes, sous-estime le stockage nécessaire.

---

## 5. Référentiel de temps

Sur la même alerte :

| Origine | Valeur |
|---|---|
| Journal brut de la source | `Sep 16 22:03:34` |
| Horodatage indexé | `Sep 17 00:03:36` |

Deux heures d'écart, sans défaut de synchronisation : `timedatectl` confirme la
synchronisation sur `SIEM01` et `SRV01`. Le journal brut issu de journald est
exprimé en temps universel, l'horodatage de l'alerte en heure locale.

> **Fixer explicitement le référentiel de temps avant toute reconstitution de
> chronologie.** Le texte brut d'une source et son horodatage d'indexation
> peuvent exprimer le même instant dans deux fuseaux différents. Une chronologie
> établie en mélangeant les deux est fausse sans aucun avertissement.

Le point deviendra déterminant au raccordement de `FW01`, qui émettra ses
propres horodatages en syslog.

---

## 6. Hiérarchisation de la règle fail2ban

### Écart constaté

Le test de journal sur une ligne de démarrage de service produisait une alerte de
niveau 5 avec une description aux champs vides :

```
Fail2ban: Action détectée () sur le jail  contre l'IP
```

Le décodeur parent s'accroche à la structure de la ligne, donc il accepte toute
ligne portant un nom de prison, y compris les messages de fonctionnement. La
règle unique ne posait aucune condition sur les champs extraits et alertait au
même niveau dans tous les cas.

Une action défensive automatique et un message de vidage de table se
présentaient donc à l'identique dans la liste d'alertes.

### Périmètre réel du décodeur

Vérification faite sur trois lignes, le décodeur est moins permissif
qu'initialement supposé :

| Ligne | Décodage |
|---|---|
| `[sshd] Ban 10.10.20.2` | Cinq champs extraits |
| `[sshd] Flush ticket(s) with nftables` | Parent seul, aucun champ |
| `banTime: 3600` | Aucun décodeur, ligne jamais indexée |

Le `prematch` exige un mot entre crochets après le niveau, c'est-à-dire un nom de
prison. Les lignes de configuration sans prison ne sont pas collectées du tout.

### Trois itérations

| Version | Condition | Résultat |
|---|---|---|
| 1 | `<match>fail2ban.actions</match>` | Chargée sans erreur, sans effet |
| 2 | `<field name="srcip">` | Refusée au chargement, champ statique |
| 3 | `<srcip>any</srcip>` | Fonctionne |

La première version illustre à nouveau le motif dominant de ce projet : une
condition syntaxiquement valide, chargée sans message, et qui ne correspond
jamais. La cause tient à ce que la comparaison ne porte pas sur la ligne brute
telle qu'elle est lue, mais sur ce qui subsiste après prédécodage et décodage.

La deuxième a été refusée avec un message explicite au test de configuration.

> **Wazuh distingue champs statiques et champs dynamiques.** `srcip` est un champ
> statique, interrogé par sa balise propre et non par `<field name="...">`. La
> famille du champ n'est pas déductible de la syntaxe du décodeur, où les deux
> s'écrivent de la même façon dans `<order>`.

> **Le test de configuration détecte ce que le redémarrage masque.** Lancer
> `wazuh-analysisd -t` avant tout redémarrage a évité un service dégradé. La
> session 9 avait rencontré un rejet silencieux, détectable seulement par le
> compteur de règles ; ici le refus est explicite. Les deux cas coexistent, ce
> qui justifie de conserver les deux vérifications.

### État final

| Règle | Niveau | Portée |
|---|---|---|
| 100001 | 3 | Toute ligne décodée, collecte sans alerte |
| 100002 | 7 | Lignes portant une adresse source, actions de bannissement |

Description produite : `Fail2ban: Ban sur le jail sshd contre 10.10.20.2`.

La classification MITRE initialement posée a été retirée. `T1110` désigne
l'attaque par force brute, or cette règle signale une réponse défensive et non
l'attaque.

> Une classification approximative est moins utile qu'une absence de
> classification. Elle place l'événement dans une catégorie où un analyste ira
> le chercher pour une autre raison.

### Compteur de règles

Passage de 8451 à 8454, soit les trois règles locales chargées. Confirmation par
l'indicateur numérique plutôt que par la lecture du fichier, méthode établie en
session 9.

Écart relevé au passage : une suppression accidentelle de la règle 100200 lors
d'une édition a fait chuter le compteur à 8452. L'écart a été constaté par le
compteur avant d'avoir produit un effet en exploitation.

---

## 7. Commentaire de configuration sans effet

Le bloc de collecte déclaré par erreur sur `SIEM01`, qui pointe vers un fichier
`fail2ban` inexistant sur cette machine, a été commenté avec le caractère `#`.

L'erreur de lecture de fichier a persisté après redémarrage. Le caractère `#`
n'est pas un marqueur de commentaire en XML : il est interprété comme du texte
et le bloc qui suit reste valide pour le parseur.

> **Une syntaxe familière transposée dans un langage qui ne la partage pas
> produit une configuration crue désactivée et toujours active.** Symétrique du
> motif habituel de ce projet, où une configuration écrite reste sans effet.
> Dans les deux cas, l'écart entre l'intention et l'effet ne produit aucun
> message.

Le commentaire XML s'écrit `<!-- -->`, et n'admet pas de double tiret interne.

---

## 8. Hypothèses formulées et démenties

Six hypothèses ont été formulées au cours de la session à partir de traces ou de
connaissances générales, puis démenties par une vérification.

| Hypothèse | Ce qui l'a démentie |
|---|---|
| Les crochets du décodeur sont des classes de caractères | Test négatif : une ligne au format voisin n'est pas décodée |
| Le décodeur est trop permissif | Une ligne sans nom de prison n'est pas décodée du tout |
| L'indexeur est en panne | Horodatages antérieurs au démarrage effectif |
| La double alerte est une double collecte | Deux textes bruts différents |
| `<match>` sur le nom de module filtre la règle | Chargée sans effet |
| `<field name="srcip">` teste le champ extrait | Refusée, champ statique |

Le moteur d'expressions régulières de l'outil est volontairement réduit et ne
suit pas les conventions usuelles. Les crochets y sont des caractères
littéraux, ce qui rend le décodeur de la session 9 correct pour une raison
différente de celle qui avait été documentée.

> **Une explication plausible produite rapidement n'est pas une cause vérifiée**,
> quelle que soit l'assurance avec laquelle elle est formulée. Six fois sur une
> seule session, une vérification d'une commande a suffi à écarter une
> hypothèse raisonnable. Le coût de la vérification est systématiquement
> inférieur au coût de la correction appliquée dans la mauvaise direction.

C'est la récurrence documentée depuis la session 3, observée ici dans sa forme
la plus dense.

---

## État à la fin de la session

- Point ouvert sur la cohérence `fail2ban` levé, les trois étages concordent
- Chaîne de traitement vérifiée de bout en bout par donnée témoin valide
- Deux agents actifs, confirmés côté gestionnaire et côté agents
- Règle `fail2ban` hiérarchisée sur deux niveaux, description exploitable
- Horloges synchronisées sur `SIEM01` et `SRV01`
- Archives brutes confirmées désactivées

## Reste à traiter

- Retrait effectif du bloc de collecte étranger sur `SIEM01`, à confirmer par
  l'absence de l'erreur 1103 après redémarrage
- Vérification des horloges de `DC01` et de `FW01`
- Politique de rétention des index et arbitrage sur les archives brutes
- Raccordement de `PC01`, qui suppose d'étendre la stratégie d'audit aux postes
- Raccordement de `FW01`, périmètre limité aux refus, et traitement de l'absence
  de règles IPv6 dans la matrice de flux
- Détections 2, 3 et 4 du plan
