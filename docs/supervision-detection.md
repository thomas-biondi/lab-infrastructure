# Supervision et détection

> Description du dispositif de collecte, des détections en service et de leurs
> limites connues. Le raisonnement figure dans
> `journal/session-09-supervision.md`,
> `journal/session-10-verification-chaine.md` et
> `journal/session-11-temps-audit-postes.md`.

---

## 1. Principe directeur

Un SIEM qui alerte trop ne sert à rien, exactement comme un pare-feu qui
autorise tout.

L'arbitrage entre faux positifs et faux négatifs n'est pas optimisable : réduire
les uns augmente les autres. Et les deux aboutissent au même résultat par des
chemins différents. Un faux négatif manque une menace ; un excès de faux
positifs conduit l'analyste à cesser de lire, et la détection réelle se noie
dans le bruit.

**Le critère retenu n'est donc pas « le moins d'alertes possible » mais « un
volume qu'un humain peut réellement traiter ».**

### Trois niveaux de traitement

| Niveau | Contenu | Volume attendu |
|---|---|---|
| Alerte immédiate | Rare, grave, exige une action humaine | Quelques-unes par mois |
| Collecte | Conservé sans alerter, matériau d'investigation | Tout le reste |
| Agrégation | Alerte sur un motif, non sur l'événement isolé | Selon les seuils |

Le classement ne dépend pas de la gravité de l'événement mais de **ce qu'on
ferait en le recevant**. Si la réponse est « rien, je note », ce n'est pas une
alerte.

**Critère de tri** : est-ce que cet événement justifie d'être réveillé la nuit ?
Si oui, c'est une alerte. Sinon, c'est de la collecte.

### Événement isolé et motif

Un événement rare constitue un signal en soi. Un événement fréquent n'en devient
un que par son motif, sa source ou son contexte.

Constaté sur la maquette : trente-neuf échecs d'authentification n'ont produit
aucune alerte de niveau élevé, tandis qu'une seule authentification réussie en a
produit une. L'échec est du bruit, le succès est le signal.

---

## 2. Plan de détection

| # | Détection | Source | Motif | Conduite à tenir | État |
|---|---|---|---|---|---|
| 1 | Modification d'un groupe privilégié | `DC01` | Étape obligée de la plupart des compromissions | Identifier l'auteur, vérifier qu'une demande existe, retirer si non justifié | En service |
| 2 | Succès après série d'échecs | `SRV01`, `DC01` | Indique qu'un secret a été trouvé | Vérifier l'origine, confirmer avec le titulaire, changer le secret si doute | À faire |
| 3 | Lecture d'un mot de passe LAPS | `DC01` | Doit correspondre à une intervention connue | Rapprocher d'un ticket, forcer la rotation si non justifié | Audit posé, règle à écrire |
| 4 | Refus RADIUS répétés | `SRV01` | Tentative d'accès non autorisé à un équipement | Identifier le compte et la source, vérifier si le compte est compromis | À faire |

La colonne « conduite à tenir » est celle que l'on omet le plus souvent. Une
alerte sans conduite associée produit un analyste qui la regarde, ne sait
qu'en faire, et la ferme.

**Collecté sans alerter** : blocages du pare-feu, échecs d'authentification
individuels, élévations de privilèges, sauvegardes, messages de fonctionnement
des services de détection, créations de processus sur les postes.

---

## 3. Architecture

| Composant | Rôle | État |
|---|---|---|
| `SIEM01`, `10.10.20.30` | Gestionnaire, indexeur et tableau de bord | En service |
| Agent sur `SRV01` | Journaux système par journald, SSH, élévations, `fail2ban` | En service |
| Agent sur `DC01` | Journal de sécurité Windows | En service |
| Agent sur `PC01` | Journal de sécurité Windows, créations de processus | En service |
| Syslog depuis `FW01` | Journal de filtrage, refus | Reçu et décodable, sans remontée en production |

`FW01` n'admet pas d'agent, son système n'étant pas couvert. La collecte passe
par syslog, restreinte à l'application de journalisation du filtrage pour ne pas
transmettre l'ensemble du journal système du pare-feu.

Le gestionnaire n'écoute pas en syslog par défaut. Une connexion dédiée est
déclarée, avec restriction explicite de l'adresse source émettrice. Sans cette
restriction, toute machine du réseau pourrait injecter des événements
arbitraires.

`SRV01` fonctionne sans démon syslog classique, conformément au comportement par
défaut de Debian 13. La collecte système passe par un bloc `journald`.

### Flux ouverts

| Source | Destination | Ports |
|---|---|---|
| Segment SERVEURS | `SIEM01` | 1514, 1515 en TCP |
| Segment POSTES | `SIEM01` | 1514, 1515 en TCP |
| `FW01` | `SIEM01` | 514 en UDP, syslog |

Le port 1514 transporte les événements, le 1515 assure l'enrôlement initial des
agents. Les deux sont nécessaires.

---

## 4. Référence de temps

Une corrélation entre sources n'a de sens que si toutes partagent la même
référence de temps. C'est un prérequis de supervision, pas un réglage annexe.

```
pool NTP public
   └─ FW01 ................ 10.10.20.1, seule sortie NTP
        └─ DC01 ........... 10.10.20.10, émulateur PDC, référence du domaine
             ├─ SRV01
             ├─ SIEM01
             └─ PC01 ....... par la hiérarchie du domaine
```

Structure identique à celle du DNS : un seul point de sortie, une seule
référence interne. Le trafic des serveurs Linux vers `DC01` reste à l'intérieur
de VLAN 20 et ne nécessite aucune règle de filtrage.

**Point de vigilance sur un domaine Active Directory.** Le détenteur du rôle
d'émulateur de contrôleur principal est le sommet de la hiérarchie de temps de
la forêt. Sa configuration par défaut après promotion le fait pointer vers cette
même hiérarchie, c'est-à-dire vers lui-même, ce qui le laisse sur son horloge
matérielle. L'ensemble du domaine dérive alors de façon cohérente, et l'écart ne
se manifeste qu'au contact d'une source externe.

| Indicateur | Ce qu'il dit |
|---|---|
| Source affichée | `Local CMOS Clock` signale une horloge non référencée |
| Délai de racine | Nul signifie que la machine se déclare référence ultime |
| `System clock synchronized` | Rémanent, reste vrai après une synchronisation passée |
| Contenu du message NTP | Compteur de paquets, horodatages, référence. C'est le fait |

### Référentiels dans les journaux

Le journal brut issu de journald est exprimé en temps universel, l'horodatage
d'indexation en heure locale. Deux heures d'écart sur cette maquette, sans aucun
défaut de synchronisation.

**Fixer explicitement le référentiel employé avant toute reconstitution de
chronologie.** Une chronologie mélangeant textes bruts et horodatages indexés
est fausse sans avertissement.

---

## 5. Fonctionnement de l'outil, points structurants

### L'indexation est conditionnée par les règles

**Une ligne collectée qui ne déclenche aucune règle n'est stockée nulle part.**
Elle est lue, décodée, comparée au jeu de règles, et jetée si rien ne
correspond.

Conséquence : brancher une source ne suffit jamais. Il faut vérifier qu'un
décodeur et une règle existent pour son format, sans quoi la collecte se fait
dans le vide.

C'est un choix d'architecture qui économise du stockage mais crée un angle mort.
D'autres SIEM stockent tout et appliquent les règles à la recherche.

### Trois étages indépendants

| Étage | Rôle | Échec possible |
|---|---|---|
| Collecte | L'agent lit la source | Permissions, chemin, source inactive |
| Décodage | Extraction des champs | Format non reconnu |
| Règle | Correspondance et niveau | Aucune règle pour cet événement |

Chacun échoue silencieusement. L'outil de test de journaux indique lequel.

### Indicateurs de vérification

| Indicateur | Où | Ce qu'il dit |
|---|---|---|
| `Analyzing file` | Journal de l'agent | La source est enregistrée, **pas** qu'elle est lue |
| Descripteur de fichier ouvert | `/proc/<pid>/fd` du processus | La source est réellement lue |
| `Total rules enabled` | Journal du gestionnaire | Nombre de règles chargées, révèle un rejet silencieux |
| `wazuh-analysisd -t` | Ligne de commande | Refus explicite avant redémarrage |
| Phases 1 à 3 | Outil de test de journaux | Où s'arrête exactement le traitement |
| Agent actif côté gestionnaire | Liste des agents | La connexion est établie, pas seulement le service démarré |

Une règle référençant un décodeur inexistant est rejetée sans message. Le
compteur de règles chargées est le seul moyen de s'en apercevoir. À l'inverse,
une règle testant un champ statique par la mauvaise balise est refusée avec un
message explicite. Les deux cas coexistent, ce qui justifie de conserver les
deux vérifications.

### Le test hors ligne ne prouve pas la production

L'outil de test de journaux charge les règles pour son propre compte et ne fait
intervenir ni l'agent, ni le transport, ni la file d'analyse, ni l'indexation.
Un résultat correct en test est compatible avec une chaîne inopérante.

Cas constaté : un événement de filtrage du pare-feu produit une alerte de niveau
5 en test et aucune en production.

---

## 6. Décodeur personnalisé

`fail2ban` écrit dans un format qui n'est pas du syslog : horodatage ISO avec
millisecondes, nom de module pointé, absence de nom d'hôte. Le prédécodage
n'extrait aucun nom de programme, et aucun décodeur natif ne se sélectionne.

**Solution retenue** : un décodeur dont le `prematch` s'accroche à la
**structure** de la ligne plutôt qu'à un nom de programme.

```xml
<decoder name="fail2ban-local">
  <prematch>[\d+]:\s*\w+\s*[\w*]</prematch>
</decoder>

<decoder name="fail2ban-local-child">
  <parent>fail2ban-local</parent>
  <regex>[(\d+)]:\s*(\w+)\s*[(\w*)]\s*(\w*)\s*(\d+.\d+.\d+.\d+)</regex>
  <order>pid,level,jail,action,srcip</order>
</decoder>
```

### Moteur d'expressions régulières

L'outil n'emploie pas les expressions régulières usuelles mais un moteur réduit,
qui connaît `\d`, `\w`, `\s`, les alternatives et les groupes de capture. **Les
crochets y sont des caractères littéraux.**

> **Une syntaxe familière dans un outil inconnu n'est pas la même syntaxe.**
> Vérifier quel moteur est en jeu avant d'interpréter un motif, y compris
> lorsqu'il produit le résultat attendu.

### Périmètre réel

| Ligne | Décodage |
|---|---|
| `[sshd] Ban 10.10.20.2` | Cinq champs extraits |
| `[sshd] Flush ticket(s) with nftables` | Parent seul, aucun champ |
| `banTime: 3600` | Aucun décodeur, ligne non indexée |

Le `prematch` exige un mot entre crochets après le niveau, c'est-à-dire un nom
de prison. Les lignes de configuration en sont dépourvues et ne sont pas
collectées.

### Décodeur du pare-feu

Le format de journal de filtrage émis en syslog BSD historique est reconnu
nativement par le décodeur `pf`, qui extrait l'action, la direction, les
adresses, les ports, le protocole et l'identifiant de règle. Aucun décodeur
personnalisé n'est nécessaire.

Le format normalisé récent n'est pas reconnu par ce décodeur. Le choix du format
d'émission conditionne donc le décodage.

---

## 7. Règles de détection personnalisées

### 7.1 Actions du service de blocage automatique

```xml
<group name="fail2ban,local,">

  <rule id="100001" level="3">
    <decoded_as>fail2ban-local</decoded_as>
    <description>Fail2ban: evenement de service</description>
  </rule>

  <rule id="100002" level="7">
    <if_sid>100001</if_sid>
    <srcip>any</srcip>
    <description>Fail2ban: $(action) sur le jail $(jail) contre $(srcip)</description>
  </rule>

</group>
```

| Règle | Niveau | Portée |
|---|---|---|
| 100001 | 3 | Toute ligne décodée, collecte sans alerte |
| 100002 | 7 | Lignes portant une adresse source, actions de bannissement |

**Champs statiques et champs dynamiques.** `srcip` est un champ statique,
interrogé par sa balise propre. Une condition écrite `<field name="srcip">` est
refusée au chargement. La famille du champ n'est pas déductible de la syntaxe du
décodeur.

**Condition portant sur le texte.** Une condition `<match>` posée sur le nom de
module s'est chargée sans erreur et sans jamais correspondre. La comparaison ne
porte pas sur la ligne brute telle qu'elle est lue, mais sur ce qui subsiste
après prédécodage et décodage.

### 7.2 Modification de groupe privilégié

```xml
<rule id="100200" level="12">
  <if_sid>60144,60147</if_sid>
  <field name="win.eventdata.targetUserName">DL_Admin_|GG_|Admins du domaine|Administrateurs de</field>
  <description>Modification d un groupe privilegie : $(win.eventdata.targetUserName) par $(win.eventdata.subjectUserName)</description>
  <mitre><id>T1098</id></mitre>
  <group>privilege_escalation,</group>
</rule>
```

Trois apports par rapport à la règle native : hiérarchisation restreinte aux
groupes sensibles de cette infrastructure, description nommant le groupe et
l'auteur, classification corrigée.

### 7.3 Contrôle de chargement

Base sans règles locales : 8451 règles. Le compteur est la vérification de
référence après toute modification. Un écart révèle un rejet, y compris
silencieux, et signale aussi une suppression accidentelle lors d'une édition.

---

## 8. Audit Windows

**Windows ne journalise pas par défaut ce qu'il faut détecter.** Le canal de
sécurité se remplit, ce qui donne l'illusion d'une couverture, mais les
événements déterminants exigent l'activation explicite de l'audit.

### 8.1 Contrôleurs de domaine

| Catégorie | Sous-catégorie | Réglage | Objet |
|---|---|---|---|
| Gestion de comptes | Groupes de sécurité | Succès | Modification de groupe privilégié |
| Gestion de comptes | Comptes d'utilisateur | Succès et échec | Création, suppression, réinitialisation |
| Connexion de compte | Validation des identifiants | Succès et échec | Authentifications NTLM |
| Connexion de compte | Opérations de ticket Kerberos | Succès et échec | Base de la détection Kerberoasting |
| Connexion de compte | Authentification Kerberos | Succès et échec | Échecs et leurs codes |
| Ouverture de session | Ouverture de session | Succès et échec | Connexions et leur type |
| Accès DS | Accès au service d'annuaire | Succès et échec | Lecture des attributs LAPS |

**Écart de mise en œuvre connu** : ces réglages sont portés par la stratégie par
défaut du domaine et non par un objet dédié, donc appliqués à toutes les
machines. Les sous-catégories propres aux contrôleurs de domaine sont ainsi
actives sur les postes, où elles produisent du volume sans objet. Correction
prévue.

> Modifier une stratégie par défaut plutôt que créer un objet dédié empêche de
> distinguer ce qui a été configuré de ce qui était livré, et rend la portée
> impossible à restreindre. Le moindre privilège s'applique aussi à la
> journalisation.

### 8.2 Postes de travail

Objet dédié `GPO_Postes_Audit`, lié à l'unité d'organisation des postes,
distinct de l'objet de durcissement. Audit et durcissement ont des cycles de vie
différents et doivent pouvoir être désactivés séparément.

| Catégorie | Sous-catégorie | Réglage | Objet |
|---|---|---|---|
| Ouverture/fermeture de session | Ouvrir la session | Succès et échec | Connexions et leur type |
| Ouverture/fermeture de session | Ouverture de session spéciale | Succès | Sessions à privilèges |
| Suivi détaillé | Créer un processus | Succès | Exécution sur le poste |
| Gestion des comptes | Groupes de sécurité | Succès et échec | Administrateurs locaux |
| Gestion des comptes | Comptes d'utilisateur | Succès et échec | Création de compte local |
| Accès aux objets | Partage de fichiers | Échec | Accès refusés |

Un poste n'a pas les mêmes événements intéressants qu'un contrôleur de domaine.
Le contrôleur voit les authentifications du domaine, le poste voit l'exécution.

### 8.3 Ligne de commande dans les créations de processus

L'événement de création de processus n'indique par défaut que le programme
lancé, sans ses arguments. Sans eux, on constate qu'un interpréteur a démarré
sans savoir ce qu'il exécute. L'inclusion de la ligne de commande s'active par
un réglage distinct, sous les modèles d'administration.

**Contrepartie assumée** : tout secret passé en argument se retrouve en clair
dans le journal, donc dans le SIEM, lisible par quiconque y accède. Constaté
directement sur cette maquette lors d'un test employant une commande de création
de compte local.

> La valeur de détection et l'exposition de secrets sont deux faces du même
> réglage. Le choix se documente, et s'accompagne d'une règle de gestion sur les
> commandes qui acceptent un secret en argument.

L'événement porte également l'identifiant du processus créateur, ce qui permet
de reconstituer la filiation. Ce n'est pas le programme qui est suspect, c'est
sa filiation : un interpréteur lancé par un traitement de texte n'a pas la même
signification que lancé par un administrateur.

### 8.4 Vérification

`auditpol /get` interroge la configuration effective de la machine, et non la
stratégie. C'est ce qui fait foi.

Détail de lecture : l'outil écrit « Réussite » pour un audit en succès seul et
« Succès et échec » lorsque les deux sont actifs. Un filtre textuel sur un seul
libellé donne une vue incomplète sans avertissement.

### 8.5 Identifiants d'événements à connaître

| Identifiant | Signification |
|---|---|
| 4728 / 4729 | Ajout / retrait dans un groupe **global** |
| 4732 / 4733 | Ajout / retrait dans un groupe **de domaine local** |
| 4756 / 4757 | Ajout / retrait dans un groupe **universel** |
| 4720 | Création d'un compte |
| 4724 | Réinitialisation de mot de passe |
| 4740 | Verrouillage de compte |
| 4625 | Échec d'ouverture de session |
| 4624 | Ouverture de session réussie |
| 4662 | Opération sur un objet de l'annuaire |
| 4688 | Création d'un processus |

**La portée du groupe détermine l'identifiant.** Une règle ne couvrant que 4728
manque toutes les modifications de groupes de domaine local, ce qui inclut les
groupes d'accès du modèle AGDLP.

---

## 9. Détection de la lecture d'un secret machine

### Nature de l'événement

Il n'existe pas d'événement propre à LAPS. Une lecture de mot de passe est une
lecture d'attribut sur un objet ordinateur, donc un accès au service d'annuaire,
événement 4662.

Version en place : Windows LAPS avec attributs chiffrés, identifiée par le
schéma de l'annuaire. Attributs surveillés : `ms-LAPS-Password` et
`ms-LAPS-EncryptedPassword`.

### Deux niveaux indépendants

> **L'audit d'accès à l'annuaire fonctionne à deux niveaux.** La sous-catégorie
> autorise la journalisation, la liste d'audit de l'objet la déclenche. Activer
> la première seule produit un coût en volume sans aucune couverture.

C'est une variante du composant actif qui ne fait rien, appliquée à un mécanisme
d'audit.

### Entrées d'audit posées

Sur l'unité d'organisation des postes, héritées aux objets ordinateurs :

| Paramètre | Valeur | Raison |
|---|---|---|
| Principal | Tout le monde | On veut savoir qui lit, administrateurs compris |
| Type | Réussite | L'échec produit du bruit sur les accès normaux |
| Droit | Lecture de propriété | C'est la lecture qu'on détecte |
| Portée | Objets Ordinateur descendants | Évite les objets utilisateurs et groupes |
| Attributs | Les deux attributs LAPS uniquement | Sans restriction, tout accès à tout attribut serait audité |

La restriction aux attributs détermine le volume. C'est le traitement concret de
l'arbitrage laissé ouvert sur l'accès au service d'annuaire.

L'héritage se vérifie sur l'objet ordinateur lui-même : une entrée posée sur
l'unité ne garantit pas son application.

### Ce que la détection voit

| Cas | Déchiffrement | Événement produit |
|---|---|---|
| Compte hors du groupe autorisé | Refusé | Oui |
| Compte autorisé | Réussi | Oui |

> **La lecture de l'attribut et son déchiffrement sont deux opérations
> distinctes.** L'audit journalise la première, indépendamment du succès de la
> seconde. Une détection fondée sur cet événement voit donc la tentative, y
> compris lorsqu'elle n'aboutit pas.

C'est le cas qui compte : un attaquant sans droit de déchiffrement produit
exactement la même trace qu'un administrateur autorisé.

### Limites connues

Le nom de l'objet cible apparaît sous forme d'identifiant global et non sous le
nom de la machine. La description d'une règle ne pourra pas nommer la machine
directement.

Aucune règle livrée ne couvre l'événement 4662. La détection doit être écrite
entièrement, avec une condition portant sur l'identifiant de l'attribut lu.

---

## 10. Champs exploitables d'un événement Windows

| Champ | Contenu |
|---|---|
| `subjectUserName`, `subjectUserSid` | Auteur de l'action |
| `subjectLogonId` | Session de connexion de l'auteur |
| `memberName`, `memberSid` | Objet ajouté ou retiré |
| `targetUserName`, `targetSid` | Groupe concerné |
| `commandLine`, `parentProcessName` | Exécution et filiation, événement 4688 |

**Le `subjectLogonId` est le champ le plus utile en investigation.** Il
identifie une session précise et permet de reconstituer l'ensemble des actions
de cette session : connexion initiale, provenance, puis chaque opération.

Le suffixe `-500` d'un identifiant de sécurité désigne le compte administrateur
intégré, identique sur toute installation Windows. C'est la raison de sa
surveillance particulière, et celle de l'existence de LAPS.

---

## 11. Qualification d'une alerte

Cinq questions, dans cet ordre :

1. **Quoi ?** Nature technique de l'événement
2. **Où ?** Machine, segment, criticité
3. **Qui ?** Source, compte concerné
4. **Quand ?** Horaire, jour, contexte d'activité
5. **Est-ce cohérent ?** Confrontation au fonctionnement normal connu

**La cinquième question est celle qui décide, et elle exige la connaissance de
l'infrastructure.**

### Faux positif et cause du faux positif

| Réponse | Effet |
|---|---|
| Corriger l'environnement | Traite la cause |
| Ajuster la règle | Traite le symptôme |
| Exclure | Masque le symptôme et crée un angle mort |

**Lorsqu'un faux positif provient d'une faiblesse de l'environnement, corriger
l'environnement plutôt que la règle.**

---

## 12. Exclusions

| Exclusion | Portée | Motif | Date |
|---|---|---|---|
| Poste d'administration | `fail2ban`, prison `sshd` | Exercices de détection répétés | Session 9 |

**Toute exclusion crée un angle mort permanent.** Si le poste d'administration
était compromis, plus rien venant de lui ne serait détecté.

Règle de gestion : toute exclusion se documente avec sa justification et sa
date, et se révise périodiquement.

Effet de bord constaté : une tentative depuis le poste d'administration produit
bien une détection, mais aucune action de bannissement. La détection subsiste,
la réponse est suspendue.

---

## 13. Méthode d'investigation et de vérification

### La donnée témoin

Une recherche qui ne retourne rien ne prouve pas l'absence de données : elle
prouve que cette recherche-là ne les trouve pas.

**Méthode** : injecter un événement au contenu connu et unique, puis le
rechercher.

Trois conditions de validité, chacune apprise par un témoin invalide :

1. **Le témoin doit déclencher une règle connue.** Sur un dispositif qui
   n'indexe que ce qui correspond, un témoin arbitraire ne distingue pas
   l'absence de collecte de l'absence de règle.
2. **Le témoin doit emprunter le chemin testé.** Un test de connectivité lancé
   depuis la machine cible ne traverse pas le pare-feu.
3. **Le marqueur ne doit pas pouvoir apparaître ailleurs.** Ni identifiant de
   règle, ni numéro de port, ni valeur numérique.

### Recherches et correspondances fortuites

> **Ne jamais rechercher un identifiant ou une valeur numérique par
> correspondance de texte libre dans un fichier d'alertes.** Rechercher sur le
> champ.

Deux causes de correspondance fortuite, également trompeuses :

- Les identifiants internes contiennent des séquences numériques arbitraires.
  Un identifiant d'alerte peut contenir le numéro d'événement recherché.
- Les commandes d'administration sont journalisées avec leurs arguments. Une
  recherche pour un identifiant retourne la commande de recherche elle-même, et
  le compteur augmente à chaque tentative, ce qui ressemble exactement à un flux
  en temps réel.

Corollaire technique : au delà d'une certaine taille, l'outil de recherche
traite le fichier d'alertes comme binaire et cesse d'afficher les
correspondances. Les comptages obtenus sans l'option adéquate sont faux sans
message d'erreur.

### Latence de chaîne

Entre l'événement source et sa disponibilité en recherche, il y a la collecte,
l'analyse, l'écriture fichier, la lecture par l'expéditeur et l'indexation. Un
comptage lancé immédiatement après l'injection d'un témoin retourne zéro sans
que cela signifie quoi que ce soit.

Un redémarrage du gestionnaire coupe les agents une dizaine de secondes. Un test
lancé dans cette fenêtre donne un résultat vide pour une raison sans rapport
avec ce qui est testé.

### Vérification d'une chaîne de bout en bout

| Étage | Vérification | Ce qui fait foi |
|---|---|---|
| Émission | La source écrit | Descripteur de fichier ouvert en écriture |
| Collecte | L'agent lit | Descripteur de fichier ouvert en lecture |
| Transport | L'agent est connecté | Liste des agents côté gestionnaire |
| Réception syslog | Le service écoute | Socket ouverte, puis capture réseau |
| Décodage et règle | Correspondance | Outil de test de journaux, phases 1 à 3 |
| Analyse en production | Alerte produite | Présence dans le fichier d'alertes |
| Indexation | Document écrit | Comptage sur l'index, sur un champ |

### Croiser les critères

Chercher par un champ que tous les événements ne portent pas donne un résultat
incomplet, sans avertissement. Croiser au moins deux critères indépendants et
comparer les volumes.

### Reconstruire une chronologie

Une alerte est un point de départ, non une conclusion. L'analyste reconstitue ce
qui précède et ce qui suit, et cherche ce que l'alerte n'a pas vu.

Séquence observée sur la maquette, qui hors contexte constituerait une
compromission caractérisée : reconnaissance, pause, reprise massive, détection
de force brute, puis authentification réussie soixante-quatre secondes plus
tard. Trois éléments plaidaient pour le faux positif, aucun accessible sans un
journal riche.

C'est la raison pour laquelle le journal brut est conservé en plus des champs
décodés.

---

## 14. Plan de collecte par itération

Les sources ne se déterminent pas intégralement au départ. On en branche
quelques-unes, on investigue, on constate un manque, on l'ajoute.

**Manque constaté par ce mécanisme** : les actions du service de blocage
automatique n'étaient pas collectées. Les actions défensives automatiques sont
souvent les moins bien couvertes, parce qu'on pense à collecter les attaques et
non les réponses.

---

## 15. Éléments de dimensionnement

Relevé au 17 septembre 2026, ordre de grandeur et non régime de croisière :
l'index courant contient les exercices de détection des sessions 9 à 11.

| Élément | Valeur |
|---|---|
| Espace alloué à `/var` | 41 Go, dont 26 disponibles |
| Index d'alertes, deux jours | environ 3300 documents |
| Archives brutes | Désactivées |
| Mémoire de la machine | 7,7 Go, dont 1,6 pour l'indexeur et 1,7 pour le gestionnaire |

### Composition par type d'événement

| Identifiant | Part | Nature |
|---|---|---|
| 4624 | 524 | Ouvertures de session |
| 4688 | 249 | Créations de processus, depuis le raccordement des postes |
| 4769 | 19 | Opérations de tickets Kerberos |
| Autres | moins de 60 | Événements applicatifs et système |

Deux identifiants représentent plus de quatre-vingt-dix pour cent des alertes.
Toute politique de rétention se dimensionne à partir de ces deux familles.

### Facteurs d'amplification

| Événement | Alertes produites |
|---|---|
| Tentative d'authentification SSH sur compte inexistant | 2 |
| Connexion TCP refusée par le pare-feu | 5, une par retransmission |

Un dimensionnement établi en comptant les événements, et non les alertes,
sous-estime le stockage.

---

## 16. Points ouverts

Les événements de filtrage de `FW01` sont reçus et décodables mais ne produisent
aucune alerte en production. Le même événement produit une alerte de niveau 5 en
test hors ligne.

Aucune règle ne couvre l'événement 4662. La détection 3 est préparée côté
annuaire mais n'a pas de règle. Une règle d'observation temporaire est en place
et doit être retirée.

Les détections 2 et 4 du plan ne sont pas implémentées.

La rétention des index n'est pas configurée. L'arbitrage sur l'activation des
archives brutes lui est lié : elles combleraient l'angle mort décrit en
section 5, au prix d'un volume à borner.

Les réglages d'audit des contrôleurs de domaine sont portés par la stratégie par
défaut du domaine et s'appliquent donc aussi aux postes.

`SIEM01` ne dispose pas de pare-feu local, contrairement à `SRV01`. Une machine
qui reçoit du syslog sans filtrage local est un écart de durcissement.

Le niveau de la règle native sur les connexions par compte au nom générique
produit un faux positif dont la cause est un nom de compte non nominatif. La
correction attendue est le renommage du compte, non l'ajustement de la règle.

La création d'un compte local sur un poste ne produit pas d'alerte de niveau
comparable à sa suppression. Candidat à une règle locale.
