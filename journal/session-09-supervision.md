# Session 9, supervision et détection

**Objectif.** Déployer un SIEM, raccorder les premières sources, formaliser un
plan de détection, puis écrire un décodeur et une règle personnalisés.

**Résultat.** Objectif atteint. Un diagnostic long sur le décodage d'une source
non standard, résolu après élimination méthodique. Plusieurs enseignements de
méthode.

---

## Arbitrage initial sur la détection

La formulation spontanée d'un objectif de supervision est « le moins de faux
positifs et aucun faux négatif ». Les deux ne sont pas simultanément
atteignables : réduire les uns augmente les autres.

L'arbitrage n'est pas symétrique pour autant. Un faux négatif manque une menace,
mais **un excès de faux positifs produit le même résultat par un autre
chemin** : l'analyste cesse de lire, ferme les alertes en masse, et la détection
réelle se noie. C'est le mode de défaillance dominant des centres opérationnels,
documenté dans les analyses post-incident de compromissions majeures, où
l'alerte avait été levée sans être lue.

**Objectif reformulé** : un volume d'alertes qu'un humain peut réellement
traiter.

Le tri ne s'effectue pas après coup mais par hiérarchisation à la conception,
sur trois niveaux : alerte immédiate, collecte sans alerte, agrégation sur
motif. Le classement ne dépend pas de la gravité de l'événement mais de ce que
l'on ferait en le recevant.

---

## Correction du premier plan de détection

Trois candidats avaient été retenus initialement. Deux ne passaient pas le
critère de tri.

**Force brute SSH.** Sur un serveur exposé, c'est un bruit de fond permanent de
plusieurs milliers de tentatives quotidiennes. Alerter dessus garantit la
saturation, et une contre-mesure automatique traite déjà le problème.

Ce qui mérite une alerte n'est pas la tentative mais **le succès après les
échecs**.

**Échecs d'authentification sur le domaine.** Même raisonnement. Un utilisateur
qui se trompe de mot de passe est un événement quotidien, constaté sur cette
maquette même.

Ce qui change la nature du signal : un même compte échouant sur de nombreuses
machines, ou de nombreux comptes échouant depuis une même source. Le motif fait
le signal, non l'événement isolé.

**Création d'un compte administrateur.** Retenu, et élargi à toute modification
d'appartenance à un groupe privilégié : ajouter un compte existant est plus
discret que d'en créer un nouveau, et tout aussi efficace.

> **Principe.** Un événement rare est un signal. Un événement fréquent n'en est
> un que par son motif, sa source ou son contexte.

Confirmation quantitative obtenue plus tard dans la session : trente-neuf échecs
d'authentification, aucune alerte de niveau élevé ; une seule authentification
réussie, une alerte.

---

## Partitionnement inadapté

Le schéma d'installation par défaut allouait 43 Go à `/home` et 2,8 Go à `/var`,
alors que l'indexation écrit exclusivement dans `/var`.

Redimensionnement réalisé grâce à LVM, dont le choix avait été justifié à la
session 6 sans avoir encore servi.

**Contraintes de l'opération**, à ne pas inverser : pour réduire, on rétrécit
d'abord le système de fichiers puis le volume logique ; pour étendre, l'inverse.
Le système de fichiers ne doit jamais, à aucun moment, être plus grand que le
volume qui le porte.

Le démontage a nécessité une session dont le répertoire de travail n'était pas
sur le volume concerné : une élévation depuis une session utilisateur conserve
le shell parent actif et maintient le point de montage occupé.

> Le schéma d'installation par défaut suppose une machine de bureau avec des
> utilisateurs. Sur un serveur, la répartition attendue est l'inverse. En
> production, on partitionne manuellement selon le rôle de la machine, et l'on
> conserve de l'espace non alloué dans le groupe de volumes afin d'étendre plus
> tard sans réduire.

---

## Diagnostic principal, source non décodée

### Symptôme

Les actions d'un service de blocage automatique n'apparaissaient pas dans le
SIEM, alors que l'agent déclarait surveiller son journal.

### Hypothèses successives, toutes écartées

| Hypothèse | Vérification | Résultat |
|---|---|---|
| Permissions du fichier journal | Lecture sous l'identité du service | Confirmée, corrigée, insuffisante |
| Lecture démarrée après les événements | Comparaison des horodatages | Écartée |
| Configuration de l'agent mal placée | Lecture du fichier complet | Écartée |
| Syntaxe du décodeur | Test de configuration sans erreur | Écartée |
| Gestionnaire non redémarré | Redémarrage explicite | Écartée |
| Convention de nommage du fichier | Renommage avec préfixe numérique | Écartée |
| Déclaration explicite du décodeur | Ajout dans la configuration | Provoque un refus de démarrage, doublon |

Sept hypothèses, toutes plausibles, toutes fausses. Chacune a coûté un cycle
complet de modification et de test.

### Fait déterminant

Le compteur de règles chargées est passé de 8451 à 8453 après l'ajout de trois
règles locales. Une règle sur trois était donc rejetée : celle qui référençait
le décodeur.

**Cet indicateur numérique a tranché là où sept raisonnements avaient échoué.**
Le répertoire était bien chargé, les règles aussi, et le décodeur ne l'était
pas.

### Résolution

La cause tenait à la construction du `prematch`. Un motif fondé sur une chaîne
littérale échoue lorsque le prédécodage n'a extrait aucun nom de programme, ce
que l'outil de test indiquait depuis le début par un champ vide.

Un `prematch` s'accrochant à la **structure** de la ligne, indépendamment des
champs prédécodés, fonctionne immédiatement.

> **Enseignement technique.** Avec un format non syslog, faire correspondre la
> structure plutôt qu'un nom.
>
> **Enseignement de méthode.** Sept explications plausibles successives, aucune
> vérifiée avant d'être appliquée. C'est exactement le travers que ce journal
> documente depuis la session 3. Partir de ce que le système déclare charger,
> qui est un fait, plutôt que de raisonner sur ce qu'il devrait faire.

### Voie alternative explorée

Reconfigurer la source pour qu'elle écrive au format syslog a été tentée : le
prédécodage devient correct et des décodeurs natifs existent. Cette voie n'a pas
abouti dans le temps imparti, mais reste la solution structurellement préférable.

> **Corriger à la source plutôt qu'en aval.** Lorsqu'une source produit un
> format non standard, il est souvent plus simple de lui demander d'écrire
> autrement que d'écrire un décodeur pour son format propriétaire.

---

## Recherches non concluantes et donnée témoin

Plusieurs conclusions d'absence de données ont été tirées de recherches dont la
syntaxe n'était pas fiable. Une recherche par motif partiel retournait des
événements sans rapport, correspondant à des fragments de commandes
d'administration.

> **Une recherche qui ne retourne rien ne prouve pas l'absence de données, elle
> prouve que cette recherche-là ne les trouve pas.**
>
> Méthode : injecter un événement au contenu connu et unique, puis le
> rechercher. C'est le seul test qui distingue les deux cas.

Corollaire associé : chercher par un champ que tous les événements ne portent
pas donne un résultat incomplet sans avertissement. Croiser au moins deux
critères indépendants et comparer les volumes.

---

## Reconstitution de chronologie

Une investigation menée sur les événements produits pendant la session a donné
la séquence suivante :

```
15:18   6 échecs, utilisateur inexistant
        pause de 32 minutes
15:50   8 échecs en moins d'une seconde
15:50   détection de force brute
15:51   authentification réussie
15:54   reprise de l'activité
```

**Hors contexte, cette séquence constitue une compromission caractérisée** :
reconnaissance, reprise massive, accès réussi en soixante-quatre secondes,
poursuite de l'activité. Un analyste découvrant cela sur une infrastructure
inconnue escaladerait immédiatement.

Trois éléments plaidaient pour le faux positif, et aucun n'aurait été accessible
sans un journal complet :

- Le mode d'authentification de la connexion réussie était incohérent avec le
  scénario d'attaque supposé
- Le compte ayant réussi ne figurait pas parmi ceux testés lors des échecs
- L'empreinte de la clé employée était identifiable et connue

> La trace technique d'une activité légitime et celle d'une attaque peuvent être
> identiques. Seul le contexte les distingue, et le contexte suppose la
> connaissance de l'infrastructure.
>
> C'est la raison pour laquelle le journal brut est conservé en plus des champs
> décodés : le décodage peut se tromper, et l'on veut toujours pouvoir revenir
> au texte d'origine.

---

## Angle mort constaté et comblé

Les actions du service de blocage automatique n'étaient pas collectées, alors
qu'un bannissement avait effectivement eu lieu pendant les exercices.

> **Les actions défensives automatiques sont souvent les moins bien
> collectées**, parce que l'attention porte sur les attaques et non sur les
> réponses. Or savoir qu'une contre-mesure s'est déclenchée modifie la
> qualification d'un incident.

Ce manque a été identifié par l'investigation elle-même, ce qui illustre le
fonctionnement réel d'un plan de collecte : il se construit par itérations,
chaque investigation révélant un angle mort à combler pour la suivante.

---

## Exclusion posée

Le poste d'administration a été placé en liste blanche du service de blocage,
afin de permettre les exercices répétés.

> **Toute exclusion crée un angle mort permanent.** Si ce poste était compromis,
> plus rien venant de lui ne serait détecté.
>
> Règle de gestion : documenter chaque exclusion avec sa justification et sa
> date, et les réviser périodiquement. Les SIEM en exploitation accumulent des
> exclusions posées par des personnes parties depuis, dont plus personne ne
> connaît la raison.

---

## Audit Windows

**Le canal de sécurité se remplit par défaut, ce qui donne l'illusion d'une
couverture.** Les événements déterminants exigent l'activation explicite de
l'audit, sous-catégorie par sous-catégorie.

État initial constaté : presque toutes les sous-catégories sur « Pas d'audit ».

Sept sous-catégories activées par stratégie de groupe sur les contrôleurs de
domaine, détaillées dans `docs/supervision-detection.md`.

**Arbitrage assumé sur l'accès au service d'annuaire** : activé largement, il
génère un volume considérable. En production, il se restreint aux objets qui
comptent. Chaque audit activé coûte du volume et des performances ; on active ce
que l'on sait exploiter.

### Écart entre l'identifiant attendu et l'identifiant obtenu

Un ajout de membre dans un groupe de domaine local a produit l'événement 4732 et
non 4728, lequel concerne les groupes globaux.

> **La portée du groupe détermine l'identifiant de l'événement.** Une règle ne
> couvrant que 4728 manque toutes les modifications de groupes de domaine local,
> c'est-à-dire précisément les groupes d'accès du modèle AGDLP.
>
> Beaucoup de règles publiées ne couvrent qu'un des trois identifiants.

---

## Première règle de détection

La règle native voyait l'événement mais ne le hiérarchisait pas selon le
contexte et produisait une description inexploitable, mentionnant un identifiant
de sécurité brut plutôt que des noms.

Règle personnalisée écrite, filtrant sur le nom du groupe cible et apportant
trois améliorations :

| Apport | Effet |
|---|---|
| Hiérarchisation | Niveau 12 et notification, uniquement pour les groupes sensibles de cette infrastructure |
| Lisibilité | Description nommant le groupe et l'auteur, exploitable sans ouvrir l'événement |
| Classification | Technique MITRE corrigée vers celle qui correspond réellement au cas |

Comparaison des descriptions produites :

```
Native : Security Enabled Local Group Member Added S-1-5-21-...-1109
Locale : Modification d un groupe privilegie : DL_Admin_Reseau par Administrateur
```

> C'est le passage de consommateur de règles à auteur de règles, et c'est ce qui
> distingue un analyste d'un ingénieur détection.

---

## Champs d'investigation d'un événement Windows

L'événement de modification de groupe porte l'auteur, sa session, l'objet
concerné et le groupe cible, chacun avec son identifiant de sécurité. La
question « qui a fait quoi à qui » se résout sans recherche complémentaire.

**Le champ identifiant la session de connexion de l'auteur est le plus utile en
investigation** : il permet de reconstituer l'ensemble des actions de cette
session, de la connexion initiale à chaque opération suivante. C'est ce qui
permet de retracer le parcours complet d'un intervenant.

---

## Erreurs de contexte d'exécution

Plusieurs commandes ont été exécutées sur une machine autre que celle
concernée : recherche d'un fichier journal sur le serveur de supervision au lieu
du serveur qui le produit, syntaxe de requête du SIEM saisie dans un terminal.

> Constante de ce projet, cinquième occurrence. Vérifier l'environnement
> d'exécution avant d'interpréter un résultat.

---

## État à la fin de la session

- SIEM déployé, partitionnement corrigé
- Deux agents raccordés et actifs
- Plan de détection formalisé avec conduite à tenir par détection
- Audit Windows activé sur les contrôleurs de domaine
- Un décodeur personnalisé en service
- Une règle de détection personnalisée en service, validée de bout en bout
- Un angle mort identifié et comblé par itération

## Reste à traiter

- Raccordement de `PC01` et de `FW01`
- Détections 2, 3 et 4 du plan
- Politique de rétention des index
