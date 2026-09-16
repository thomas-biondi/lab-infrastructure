# Supervision et détection

> Description du dispositif de collecte, des détections en service et de leurs
> limites connues. Le raisonnement figure dans
> `journal/session-09-supervision.md`.

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

| # | Détection | Source | Motif | Conduite à tenir |
|---|---|---|---|---|
| 1 | Modification d'un groupe privilégié | `DC01` | Étape obligée de la plupart des compromissions | Identifier l'auteur, vérifier qu'une demande existe, retirer si non justifié |
| 2 | Succès après série d'échecs | `SRV01`, `DC01` | Indique qu'un secret a été trouvé | Vérifier l'origine, confirmer avec le titulaire, changer le secret si doute |
| 3 | Lecture d'un mot de passe LAPS | `DC01` | Doit correspondre à une intervention connue | Rapprocher d'un ticket, forcer la rotation si non justifié |
| 4 | Refus RADIUS répétés | `SRV01` | Tentative d'accès non autorisé à un équipement | Identifier le compte et la source, vérifier si le compte est compromis |

La colonne « conduite à tenir » est celle que l'on omet le plus souvent. Une
alerte sans conduite associée produit un analyste qui la regarde, ne sait
qu'en faire, et la ferme.

**Collecté sans alerter** : blocages du pare-feu, échecs d'authentification
individuels, élévations de privilèges, sauvegardes.

---

## 3. Architecture

| Composant | Rôle |
|---|---|
| `SIEM01`, `10.10.20.30` | Gestionnaire, indexeur et tableau de bord |
| Agent sur `SRV01` | Journaux système, SSH, élévations, `fail2ban` |
| Agent sur `DC01` | Journal de sécurité Windows |

Partitionnement adapté après installation : le schéma par défaut allouait
l'essentiel de l'espace à `/home`, alors que l'indexation écrit dans `/var`. Le
volume a été réduit et l'espace transféré, opération rendue possible par le
choix de LVM.

### Flux ouverts

| Source | Destination | Ports |
|---|---|---|
| Segment SERVEURS | `SIEM01` | 1514, 1515 en TCP |
| Segment POSTES | `SIEM01` | 1514, 1515 en TCP |
| `FW01` | `SIEM01` | 514 en UDP, syslog |

Le port 1514 transporte les événements, le 1515 assure l'enrôlement initial des
agents. Les deux sont nécessaires.

---

## 4. Fonctionnement de l'outil, points structurants

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
| `Analyzing file` | Journal de l'agent | La source est enregistrée, pas qu'elle est lue |
| `Total rules enabled` | Journal du gestionnaire | Nombre de règles chargées, révèle un rejet silencieux |
| Phases 1 à 3 | Outil de test de journaux | Où s'arrête exactement le traitement |

Une règle référençant un décodeur inexistant est rejetée sans message. Le
compteur de règles chargées est le seul moyen de s'en apercevoir.

---

## 5. Décodeur personnalisé

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

Cinq champs extraits : identifiant de processus, niveau, prison, action, adresse
source.

> **Enseignement.** Avec un format non syslog, le `prematch` doit correspondre à
> un motif structurel présent dans la ligne, indépendant des champs prédécodés.
> Un `prematch` sur une chaîne littérale échoue lorsque le prédécodeur n'a pas
> identifié de programme.

---

## 6. Règle de détection personnalisée

La règle native signale les modifications de groupe, sans distinguer les groupes
sensibles des autres, et avec une description peu exploitable.

```xml
<rule id="100200" level="12">
  <if_sid>60144,60147</if_sid>
  <field name="win.eventdata.targetUserName">DL_Admin_|GG_|Admins du domaine|Administrateurs de</field>
  <description>Modification d un groupe privilegie : $(win.eventdata.targetUserName) par $(win.eventdata.subjectUserName)</description>
  <mitre><id>T1098</id></mitre>
  <group>privilege_escalation,</group>
</rule>
```

Trois apports par rapport à la règle native :

- **Hiérarchisation** : niveau 12 et notification, uniquement pour les groupes
  sensibles de cette infrastructure
- **Lisibilité** : la description nomme le groupe et l'auteur, exploitable dans
  une liste d'alertes sans ouvrir l'événement
- **Classification** : `T1098` Account Manipulation, plus juste que `T1484` pour
  ce cas

> La règle générique voyait l'événement mais ne le hiérarchisait pas selon le
> contexte et ne le rendait pas exploitable. C'est précisément le travail
> d'ingénierie de détection.

---

## 7. Audit Windows

**Windows ne journalise pas par défaut ce qu'il faut détecter.** Le canal de
sécurité se remplit, ce qui donne l'illusion d'une couverture, mais les
événements déterminants exigent l'activation explicite de l'audit.

Sous-catégories activées par stratégie sur les contrôleurs de domaine :

| Catégorie | Sous-catégorie | Réglage | Objet |
|---|---|---|---|
| Gestion de comptes | Groupes de sécurité | Succès | Modification de groupe privilégié |
| Gestion de comptes | Comptes d'utilisateur | Succès et échec | Création, suppression, réinitialisation |
| Connexion de compte | Validation des identifiants | Succès et échec | Authentifications NTLM |
| Connexion de compte | Opérations de ticket Kerberos | Succès et échec | Base de la détection Kerberoasting |
| Connexion de compte | Authentification Kerberos | Succès et échec | Échecs et leurs codes |
| Ouverture de session | Ouverture de session | Succès et échec | Connexions et leur type |
| Accès DS | Accès au service d'annuaire | Succès et échec | Lecture des attributs LAPS |

**Arbitrage assumé** : l'accès au service d'annuaire génère un volume important
lorsqu'il est activé largement. En production, il se restreint aux objets qui
comptent par des listes d'audit sur les unités d'organisation. Chaque audit
activé coûte du volume et des performances : on active ce qu'on sait exploiter.

### Identifiants d'événements à connaître

| Identifiant | Signification |
|---|---|
| 4728 / 4729 | Ajout / retrait dans un groupe **global** |
| 4732 / 4733 | Ajout / retrait dans un groupe **de domaine local** |
| 4756 / 4757 | Ajout / retrait dans un groupe **universel** |
| 4720 | Création d'un compte |
| 4724 | Réinitialisation de mot de passe |
| 4740 | Verrouillage de compte |
| 4625 | Échec d'ouverture de session |

**La portée du groupe détermine l'identifiant.** Une règle ne couvrant que 4728
manque toutes les modifications de groupes de domaine local, ce qui inclut les
groupes d'accès du modèle AGDLP. Beaucoup de règles publiées ne couvrent qu'un
des trois identifiants.

---

## 8. Champs exploitables d'un événement Windows

Exemple d'une modification de groupe :

| Champ | Contenu |
|---|---|
| `subjectUserName`, `subjectUserSid` | Auteur de l'action |
| `subjectLogonId` | Session de connexion de l'auteur |
| `memberName`, `memberSid` | Objet ajouté ou retiré |
| `targetUserName`, `targetSid` | Groupe concerné |

**Le `subjectLogonId` est le champ le plus utile en investigation.** Il
identifie une session précise et permet de reconstituer l'ensemble des actions
de cette session : connexion initiale, provenance, puis chaque opération.

Le suffixe `-500` d'un identifiant de sécurité désigne le compte administrateur
intégré, identique sur toute installation Windows. C'est la raison de sa
surveillance particulière, et celle de l'existence de LAPS.

---

## 9. Qualification d'une alerte

Cinq questions, dans cet ordre :

1. **Quoi ?** Nature technique de l'événement
2. **Où ?** Machine, segment, criticité
3. **Qui ?** Source, compte concerné
4. **Quand ?** Horaire, jour, contexte d'activité
5. **Est-ce cohérent ?** Confrontation au fonctionnement normal connu

**La cinquième question est celle qui décide, et elle exige la connaissance de
l'infrastructure.** Une même trace technique peut correspondre à une activité
légitime ou à une intrusion : seul le contexte tranche.

C'est pourquoi un SOC investit autant dans la documentation de l'environnement
supervisé que dans l'outil lui-même.

### Faux positif et cause du faux positif

Une alerte qualifiée en faux positif doit déclencher une seconde question :
pourquoi la règle s'est-elle trompée, et que faut-il changer ?

Trois réponses possibles, hiérarchisées :

| Réponse | Effet |
|---|---|
| Corriger l'environnement | Traite la cause |
| Ajuster la règle | Traite le symptôme |
| Exclure | Masque le symptôme et crée un angle mort |

**Lorsqu'un faux positif provient d'une faiblesse de l'environnement, corriger
l'environnement plutôt que la règle.** Sinon, les exclusions s'accumulent et
masquent des défauts réels.

---

## 10. Exclusions

| Exclusion | Portée | Motif | Date |
|---|---|---|---|
| Poste d'administration | `fail2ban`, prison `sshd` | Exercices de détection répétés | Session 9 |

**Toute exclusion crée un angle mort permanent.** Si le poste d'administration
était compromis, plus rien venant de lui ne serait détecté.

Règle de gestion : toute exclusion se documente avec sa justification et sa
date, et se révise périodiquement. Les SIEM en exploitation accumulent des
exclusions posées par des personnes parties depuis, dont plus personne ne
connaît la raison. C'est un angle mort majeur et rarement audité.

---

## 11. Méthode d'investigation

### La donnée témoin

Une recherche qui ne retourne rien ne prouve pas l'absence de données : elle
prouve que cette recherche-là ne les trouve pas.

**Méthode** : injecter un événement au contenu connu et unique, puis le
rechercher. C'est le seul test qui distingue « les données ne sont pas là » de
« je cherche mal ».

### Croiser les critères

Chercher par un champ que tous les événements ne portent pas donne un résultat
incomplet, sans avertissement.

Croiser au moins deux critères indépendants et comparer les volumes : par
adresse source, par agent et groupe de règles, puis dans le texte brut. Une
divergence révèle un champ manquant.

### Reconstruire une chronologie

Une alerte est un point de départ, non une conclusion. L'analyste reconstitue ce
qui précède et ce qui suit, et cherche ce que l'alerte n'a pas vu.

Séquence observée sur la maquette, qui hors contexte constituerait une
compromission caractérisée : reconnaissance, pause, reprise massive, détection
de force brute, puis authentification réussie soixante-quatre secondes plus
tard.

Trois éléments plaidaient pour le faux positif, et aucun n'est accessible sans
un journal riche :

- Le mode d'authentification de la connexion réussie était incohérent avec le
  scénario d'attaque
- Le compte ayant réussi ne figurait pas parmi ceux qui avaient été testés
- L'empreinte de la clé employée correspondait à une clé connue

C'est la raison pour laquelle le journal brut est conservé en plus des champs
décodés.

---

## 12. Plan de collecte par itération

Les sources ne se déterminent pas intégralement au départ. On en branche
quelques-unes, on investigue, on constate un manque, on l'ajoute. Chaque
investigation améliore la collecte pour la suivante.

C'est l'inverse d'une approche exhaustive initiale, qui produit un volume
ingérable dont personne ne sait ce qui sert.

**Manque constaté par ce mécanisme** : les actions de `fail2ban` n'étaient pas
collectées. Les actions défensives automatiques sont souvent les moins bien
couvertes, parce qu'on pense à collecter les attaques et non les réponses. Or
savoir qu'une contre-mesure s'est déclenchée change la qualification d'un
incident.

---

## 13. Points ouverts

`PC01` et `FW01` ne sont pas encore raccordés comme sources.

Les détections 2, 3 et 4 du plan ne sont pas implémentées.

Le niveau de la règle native sur les connexions par compte au nom générique
produit un faux positif dont la cause est un nom de compte non nominatif. La
correction attendue est le renommage du compte, non l'ajustement de la règle.

La rétention des index n'est pas configurée. Une politique de purge sera
nécessaire pour éviter la saturation du volume d'indexation.
