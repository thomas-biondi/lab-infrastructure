# Session 7, authentification centralisée

**Objectif.** Mettre en service un serveur RADIUS adossé à Active Directory, et
l'employer pour l'authentification des administrateurs sur le pare-feu, avec
autorisation fondée sur l'appartenance à un groupe de l'annuaire.

**Résultat.** Objectif atteint. Chaîne complète validée de l'équipement à
l'annuaire, avec cinq obstacles techniques résolus.

---

## Limite posée en préalable

Le 802.1X complet n'est pas réalisable sur cette maquette. Il suppose un
commutateur bloquant un port jusqu'à authentification du poste, fonction absente
des réseaux virtuels de l'hyperviseur, qui se comportent en commutateurs plats.

Le cas d'usage retenu est donc l'authentification des administrateurs sur
l'équipement réseau : chaîne identique, du client au serveur jusqu'à l'annuaire,
avec un usage réellement fonctionnel.

---

## Méthode de travail

Le mode débogage au premier plan a été employé pour l'intégralité de la session.
Il affiche chaque fichier lu, chaque module chargé, chaque étape de traitement et
le motif exact de tout refus.

La réputation de complexité de l'outil tient largement au fait qu'il est souvent
configuré puis redémarré en aveugle, avec ensuite une lecture de journaux
laconiques. Le mode débogage est à ce serveur ce que l'affichage de la
configuration effective est au service d'accès distant : la réalité plutôt que
l'intention.

---

## Obstacle 1, liaison en clair refusée par l'annuaire

**Constat.** L'instanciation du module échoue au démarrage.

```
Bind ... failed: Strong(er) authentication required
Server said: The server requires binds to turn on integrity checking
if SSL\TLS are not already active on the connection
```

**Analyse.** Refus attendu et légitime : les versions récentes de Windows Server
n'acceptent plus de liaison non protégée. Sans chiffrement, le mot de passe du
compte de service traverserait le réseau en clair à chaque connexion.

**Correction.** Déploiement d'une autorité de certification d'entreprise sur le
contrôleur de domaine, ce qui active LDAPS sans manipulation supplémentaire, le
contrôleur s'auto-délivrant un certificat de serveur.

Le certificat racine a ensuite été exporté, converti du format DER binaire vers
PEM, et déposé dans le magasin de confiance du serveur Linux, celui-ci n'étant
pas membre du domaine et ne recevant donc rien par stratégie de groupe.

> **Enseignements.**
>
> Le message d'erreur contenait la totalité du diagnostic et de la solution. Le
> mode débogage l'a fourni en une ligne, là où un redémarrage de service aurait
> laissé chercher.
>
> Un certificat interne n'est digne de confiance que pour qui connaît l'autorité
> qui l'a émis. La distribution de la racine est la moitié du travail.
>
> L'obstacle s'est révélé être une nécessité anticipée : le 802.1X suppose de
> toute façon un certificat côté serveur. L'autorité servira deux fois.

---

## Obstacle 2, aucun module compétent pour l'authentification

**Constat.** L'objet utilisateur est trouvé dans l'annuaire, puis la demande est
rejetée.

```
WARNING: No "known good" password added
ERROR: No Auth-Type found: rejecting the user
```

**Analyse.** Active Directory ne divulgue jamais le condensat du mot de passe.
Le serveur RADIUS ne dispose donc d'aucune valeur de référence à comparer, aucun
module ne se déclare compétent, et le refus par défaut s'applique.

**Correction.** Forçage explicite du type d'authentification, déclenchant une
seconde liaison sous l'identité de l'utilisateur. La réussite de cette liaison
vaut validation du mot de passe.

> **Conséquence structurante.** Ce mécanisme n'est compatible qu'avec le
> protocole transmettant le mot de passe en clair au serveur RADIUS. Il est
> incompatible avec le protocole employé par Windows pour le 802.1X.
>
> C'est l'obstacle principal d'une intégration entre ce serveur RADIUS et
> Active Directory, et il conditionne toute perspective de déploiement 802.1X.

---

## Obstacle 3, renvois d'annuaire inutiles

Chaque recherche déclenchait trois tentatives de connexion vers des partitions
d'application, toutes vouées à l'échec, le certificat du contrôleur ne couvrant
pas ces noms.

Sans conséquence fonctionnelle, mais coûteux : trois négociations chiffrées
inutiles à chaque requête d'authentification.

Correction par désactivation de la poursuite des renvois, configuration
recommandée lorsque la recherche porte sur un domaine unique.

---

## Obstacle 4, imbrication de groupes non résolue

**Constat.** Un compte membre d'un groupe global, lui-même membre du groupe
autorisé, est rejeté.

```
Processing memberOf value "CN=GG_Informatique,..."
User is not a member of "DL_Admin_Reseau"
```

**Analyse.** La lecture directe de l'attribut d'appartenance ne révèle que les
groupes de premier niveau. L'appartenance indirecte, qui est précisément le
principe du modèle AGDLP, n'est pas détectée.

**Première tentative, sans effet.** Une directive d'imbrication a été ajoutée à
la configuration du module. Le débogage a montré que le comportement était
inchangé : la directive n'est pas reconnue et a été ignorée silencieusement.

> C'est la troisième occurrence dans ce projet d'une configuration écrite,
> syntaxiquement valide et dépourvue d'effet. Les deux précédentes concernaient
> un objet de stratégie de groupe lié à une unité d'organisation sans
> utilisateur, et des directives de résolution de noms sans le paquet requis.
>
> Le point commun : aucune erreur, aucun avertissement, seulement un
> comportement inchangé. Seule la comparaison entre le comportement observé et
> le comportement attendu révèle le problème.

**Correction.** Emploi du filtre de correspondance en chaîne d'Active Directory,
identifié par l'OID `1.2.840.113556.1.4.1941`. Le contrôleur de domaine résout
lui-même toute la profondeur d'imbrication et répond par oui ou non, en une
seule requête.

Point de configuration déterminant : l'attribut d'appartenance direct doit être
retiré, faute de quoi le module continue de le lire et n'emploie jamais le
filtre.

Changement de méthode visible dans le débogage, passant d'un parcours de
l'attribut à une recherche sur l'objet groupe.

> **Enseignement.** Déléguer un calcul au composant qui détient la donnée est
> préférable à le reproduire côté client. Une requête au lieu d'un parcours, et
> la logique reste cohérente avec le modèle de l'annuaire.

---

## Obstacle 5, secret partagé incorrect

**Constat.** Les paquets émis par le pare-feu parviennent au serveur, qui les
rejette sans répondre.

```
Dropping packet without response because of error:
invalid Message-Authenticator! (Shared secret is incorrect.)
```

**Cause.** Divergence entre les deux secrets, probablement un caractère invisible
emporté lors d'une copie. Le secret généré en base64 comporte des symboles et un
retour à la ligne final.

**Correction.** Génération d'un secret hexadécimal, dépourvu de symbole ambigu,
reposé des deux côtés.

> **Enseignement.** L'absence totale de réponse est un comportement délibéré du
> protocole : répondre permettrait à un attaquant de tester des secrets et de
> mesurer sa progression.
>
> Côté client, le symptôme est un délai dépassé sans indication de cause. Le
> diagnostic n'est possible que depuis le serveur. C'est la raison pour laquelle
> le mode débogage est indispensable sur ce protocole.

---

## Validation de la chaîne

| Test | Attendu | Résultat |
|---|---|---|
| Compte autorisé, mot de passe valide | Acceptation | Conforme |
| Compte autorisé, mot de passe erroné | Refus, code `52e` | Conforme |
| Compte hors du groupe | Refus sur autorisation | Conforme |
| Appartenance indirecte retirée | Refus | Conforme |
| Compte désactivé dans l'annuaire | Refus, code `533` | Conforme |
| Connexion réelle à l'interface du pare-feu | Acceptation | Conforme |

**Le quatrième test est le plus significatif.** Le retrait de l'appartenance au
groupe global, sans aucune intervention sur l'équipement ni sur le groupe local,
supprime l'accès. Sa restauration le rétablit.

C'est la démonstration complète de ce qu'apporte l'authentification
centralisée : une action unique dans l'annuaire retire l'accès à l'ensemble du
parc, instantanément. Rapporté à trente équipements, c'est la différence entre
un départ traité en dix secondes et un départ jamais réellement traité.

**Distinction authentification et autorisation, visible dans le flux.** Pour un
compte hors du groupe, la recherche aboutit et l'objet est trouvé ; le refus
intervient à l'étape suivante. Un mot de passe correct ne suffit pas.

---

## Attributs transmis par un équipement réel

Le passage de l'outil de test au pare-feu enrichit la demande :

```
Service-Type = Login-User
NAS-Identifier = ...
NAS-Port-Type = Ethernet
```

Ces attributs permettent de distinguer le contexte d'une demande et d'accorder
des droits différenciés selon qu'elle provient d'un pare-feu, d'un commutateur
ou d'un point d'accès sans fil.

---

## Écart entre le modèle et la réalité

Le serveur RADIUS authentifie mais ne transmet aucune information de groupe
exploitable par l'équipement. Un objet utilisateur local est donc nécessaire sur
le pare-feu, dépourvu de données d'identité, servant uniquement de point
d'accrochage aux autorisations.

L'équipement impose par ailleurs un mot de passe local sur cet objet.
L'authentification est donc centralisée en théorie, tandis qu'un secret propre à
l'équipement subsiste en pratique.

> **Enseignement.** Nommer cet écart vaut mieux que de le passer sous silence.
> Un auditeur le relèvera ; savoir l'expliquer et en mesurer la portée est la
> réponse attendue.

Point de cycle de vie associé : la désactivation d'un compte dans l'annuaire
retire l'accès, mais l'objet local subsiste sur chaque équipement. Sans processus
de nettoyage, les équipements accumulent des objets sans titulaire.

---

## Ordre des sources d'authentification

La base locale est conservée en première position, devant le serveur RADIUS.

Un équipement dont l'accès dépend entièrement d'un service distant devient
inaccessible en cas de panne de ce service. Le compte local de secours est
l'équivalent de l'accès par console qui a permis de résoudre l'incident de
durcissement de l'accès distant à la session précédente.

---

## Incident mineur

Le conflit de port entre le service installé et l'instance de débogage s'est
reproduit à plusieurs reprises, avec un message identique à celui du conflit
entre serveurs DHCP rencontré à la session 2.

À relever pour la méthode : lors de la dernière occurrence, la sortie faisait
plusieurs centaines de lignes et l'erreur fatale était la dernière. Sur une
sortie longue, la lecture par la fin est la plus rapide, l'erreur fatale étant
toujours le dernier élément écrit avant l'arrêt.

---

## Point de sécurité opérationnelle

Un mot de passe réel a été exposé en clair dans une sortie de débogage
transmise, l'outil de test affichant le mot de passe soumis. Le compte concerné
a été modifié.

> **Enseignement.** Relire une sortie de débogage avant transmission, y compris
> à un collègue. Les traces d'authentification contiennent par nature ce qu'on
> ne souhaite pas divulguer.

---

## État à la fin de la session

- Autorité de certification d'entreprise opérationnelle sur le contrôleur
- Liaison LDAPS chiffrée et vérifiée entre le serveur RADIUS et l'annuaire
- Compte de service dédié, isolé et sans appartenance de groupe
- Autorisation fondée sur l'appartenance indirecte, résolue côté annuaire
- Pare-feu authentifiant ses administrateurs contre l'annuaire
- Protection contre la vulnérabilité de forge de réponse activée
- Configuration du serveur RADIUS incluse dans la sauvegarde

## Reste à traiter dans cette étape

- Gestion des mots de passe administrateurs locaux des postes
- Rattachement du groupe d'administration aux postes de travail par préférence
  de stratégie de groupe, en suspens depuis la session 5
- Durcissement des stratégies de sécurité
