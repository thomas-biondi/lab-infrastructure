# Authentification centralisée

> Description de la chaîne d'authentification et d'autorisation reliant les
> équipements à l'annuaire. Le raisonnement figure dans
> `journal/session-07-radius.md`.

---

## 1. Principe

RADIUS sépare trois fonctions distinctes :

| Fonction | Question traitée |
|---|---|
| Authentification | Qui es-tu |
| Autorisation | Qu'as-tu le droit de faire |
| Comptabilisation | Qu'as-tu fait, quand, pendant combien de temps |

**Motif de la centralisation.** Sans elle, chaque équipement porte ses propres
comptes locaux. Trente commutateurs représentent trente mots de passe à modifier
au départ d'une personne, opération que personne n'effectue réellement. Avec un
annuaire central, la désactivation d'un compte retire l'accès à l'ensemble du
parc, immédiatement.

C'est le problème que résout le modèle AGDLP pour les partages de fichiers,
transposé aux équipements réseau.

---

## 2. Chaîne mise en place

```
Administrateur
     |
     v
FW01 (client RADIUS)  --- secret partagé --->  SRV01 (FreeRADIUS)
                                                     |
                                              LDAPS, port 636
                                                     v
                                               DC01 (Active Directory)
```

| Élément | Valeur |
|---|---|
| Serveur RADIUS | `SRV01`, FreeRADIUS 3.2 |
| Annuaire | `DC01`, liaison LDAPS vérifiée |
| Client déclaré | `FW01`, `10.10.20.1` |
| Protocole d'authentification | PAP |
| Groupe autorisé | `DL_Admin_Reseau` |

---

## 3. Autorité de certification

La liaison en clair vers l'annuaire est refusée par le contrôleur de domaine :

```
The server requires binds to turn on integrity checking
if SSL\TLS are not already active on the connection
```

Ce refus est un durcissement par défaut des versions récentes de Windows
Server, et non un défaut de configuration. Sans chiffrement, le mot de passe du
compte de service traverserait le réseau en clair à chaque connexion.

Une autorité de certification d'entreprise a donc été déployée sur `DC01` :

| Paramètre | Valeur |
|---|---|
| Type | Autorité racine d'entreprise |
| Nom commun | `LAB-CA` |
| Longueur de clé | 4096 bits |
| Algorithme d'empreinte | SHA-256 |
| Validité | 10 ans |

Le type « entreprise » est intégré à l'annuaire : le certificat racine est
distribué automatiquement aux machines du domaine par stratégie de groupe, et
les contrôleurs de domaine s'auto-délivrent un certificat de serveur. LDAPS
devient donc disponible sans manipulation supplémentaire.

**Distribution vers un système hors domaine.** `SRV01` n'étant pas membre du
domaine, il ne reçoit pas la racine automatiquement. Le certificat a été exporté
depuis `DC01`, converti du format DER binaire vers PEM, et déposé dans le
magasin de confiance système.

Un certificat interne n'est digne de confiance que pour qui connaît l'autorité
qui l'a émis.

---

## 4. Compte de service

| Paramètre | Valeur | Motif |
|---|---|---|
| Emplacement | `OU=Comptes-de-service` | Isolement des comptes de personnes |
| Appartenance à un groupe | Aucune | Le droit de lecture par défaut sur l'annuaire suffit |
| Expiration du mot de passe | Désactivée | Un compte de service dont le mot de passe expire provoque une panne d'authentification sans cause apparente |
| Modification du mot de passe | Interdite | Évite une rotation accidentelle rompant la configuration |

Ces choix vont à l'encontre de ce qui s'applique à un compte humain. La
contrepartie est un mot de passe long, aléatoire et conservé en coffre.

**Exposition connue.** Un compte de service constitue la cible principale des
attaques de type Kerberoasting : le ticket de service est demandé légitimement,
puis le mot de passe est cassé hors ligne, sans déclencher d'alerte. La
longueur du secret est la seule protection réelle.

---

## 5. Méthode d'authentification

Active Directory ne divulgue jamais le condensat du mot de passe. FreeRADIUS ne
dispose donc d'aucun « mot de passe de référence » à comparer, ce qui rend le
module d'authentification par défaut inopérant.

**Mécanisme retenu, liaison sous l'identité de l'utilisateur :**

1. Recherche de l'objet utilisateur avec le compte de service
2. Vérification de l'appartenance au groupe autorisé
3. Tentative de liaison LDAP avec les identifiants fournis
4. Réussite de la liaison valant validation du mot de passe

Ce mécanisme impose de forcer explicitement le type d'authentification, faute de
quoi aucun module ne se déclare compétent et la demande est rejetée par défaut.

**Conséquence structurante.** Cette méthode n'est compatible qu'avec PAP, où le
mot de passe parvient en clair au serveur RADIUS. Elle est **incompatible avec
MS-CHAPv2**, protocole employé par Windows pour le 802.1X.

Un déploiement 802.1X avec des postes Windows supposerait de joindre le serveur
RADIUS au domaine et de déléguer la vérification à un composant tiers. C'est le
principal obstacle rencontré lors d'une intégration FreeRADIUS avec Active
Directory.

---

## 6. Autorisation par groupe

L'authentification seule accepterait tout compte de l'annuaire, y compris ceux
n'ayant aucune raison d'accéder à un équipement réseau. Une vérification
d'appartenance conditionne donc l'acceptation.

### Résolution de l'imbrication

Le modèle AGDLP place l'utilisateur dans un groupe global, lui-même membre d'un
groupe de domaine local. La lecture directe de l'attribut d'appartenance ne
révèle que les groupes de premier niveau : l'appartenance indirecte n'est pas
détectée et l'accès est refusé à tort.

**Solution retenue, filtre de correspondance en chaîne :**

```
(member:1.2.840.113556.1.4.1941:=<DN de l utilisateur>)
```

Cet identifiant d'objet désigne la règle de correspondance en chaîne d'Active
Directory. Le contrôleur de domaine résout lui-même toute la profondeur
d'imbrication et répond par oui ou non, en une seule requête. Le travail est
effectué côté serveur plutôt que par parcours successifs côté client.

**Point de configuration.** L'attribut d'appartenance direct doit être retiré :
tant qu'il est renseigné, le module le lit et n'emploie jamais le filtre. Les
deux mécanismes s'excluent.

### Chaîne validée

| Élément | Valeur |
|---|---|
| Compte | `tbiondi` |
| Groupe global | `GG_Informatique` |
| Groupe de domaine local | `DL_Admin_Reseau` |
| Appartenance directe au groupe local | Aucune |

L'accès est accordé par la seule appartenance indirecte, et retiré dès que
l'appartenance au groupe global est supprimée.

---

## 7. Sécurité du protocole

RADIUS date de 1991. Le mot de passe est protégé par une opération sur un
condensat MD5 du secret partagé, ce qui ne résiste pas à une analyse moderne. Le
protocole ne doit donc jamais transiter sur un réseau non maîtrisé ; les
déploiements récents l'encapsulent dans TLS.

**BlastRADIUS.** Vulnérabilité publiée en 2024, exploitant cette faiblesse pour
forger une réponse d'acceptation. La contre-mesure consiste à exiger l'attribut
`Message-Authenticator` sur tous les paquets, ce qui est configuré sur chaque
client déclaré.

**Secrets partagés.** Chaque client dispose de son propre secret. Un secret
compromis sur un équipement ne doit pas ouvrir l'accès aux autres. Les secrets
par défaut livrés avec la distribution ont été remplacés.

**Comportement en cas de secret incorrect.** Le serveur ne répond pas et ne
signale rien au client. C'est volontaire : une réponse permettrait de tester des
secrets et de mesurer la progression. Côté client, le symptôme est un simple
délai dépassé, sans indication de cause. Le diagnostic n'est possible que depuis
le serveur.

---

## 8. Intégration au pare-feu

| Paramètre | Valeur |
|---|---|
| Serveur déclaré | `SRV01-RADIUS`, `10.10.20.20:1812` |
| Protocole | PAP |
| Sources d'authentification | Base locale, puis RADIUS |

**L'ordre des sources est délibéré.** La base locale demeure en premier afin de
conserver l'accès au pare-feu si le serveur RADIUS est indisponible. Un
équipement dont l'accès dépend entièrement d'un service distant devient
inaccessible en cas de panne de ce service.

C'est la transposition du principe d'accès hors bande appliqué lors du
durcissement de l'accès distant à `SRV01`.

### Séparation authentification et autorisation

Le serveur RADIUS authentifie mais ne transmet aucune information de groupe
exploitable par le pare-feu. Un objet utilisateur local est donc créé sur
l'équipement, dépourvu de toute donnée d'identité, servant uniquement de point
d'accrochage aux autorisations.

Les privilèges sont portés par un groupe local et non attribués directement à
l'utilisateur, selon le même principe que le modèle AGDLP.

**Limite assumée.** L'équipement impose un mot de passe local sur cet objet.
L'authentification est donc centralisée en théorie, tandis qu'un secret propre à
l'équipement subsiste en pratique. Ce mot de passe est long, aléatoire et
conservé en coffre, mais l'écart entre le modèle et la réalité est réel.

**Point de cycle de vie.** La désactivation d'un compte dans l'annuaire suffit à
retirer l'accès, mais l'objet local subsiste sur chaque équipement. Un processus
de nettoyage est nécessaire, sans quoi les équipements accumulent des objets
sans titulaire.

---

## 9. Validation

| Test | Attendu | Résultat |
|---|---|---|
| Compte du groupe autorisé, mot de passe valide | Acceptation | Conforme |
| Compte du groupe autorisé, mot de passe erroné | Refus, code `52e` | Conforme |
| Compte hors du groupe autorisé | Refus sur autorisation | Conforme |
| Appartenance indirecte retirée | Refus | Conforme |
| Compte désactivé dans l'annuaire | Refus, code `533` | Conforme |
| Connexion réelle à l'interface du pare-feu | Acceptation | Conforme |

### Codes de diagnostic Active Directory

Retournés lors d'un échec de liaison, ils précisent la cause exacte :

| Code | Signification |
|---|---|
| `525` | Utilisateur inexistant |
| `52e` | Identifiants incorrects |
| `530` | Connexion interdite à cette heure |
| `531` | Connexion interdite depuis ce poste |
| `532` | Mot de passe expiré |
| `533` | Compte désactivé |
| `701` | Compte expiré |
| `773` | Mot de passe à changer à la prochaine connexion |
| `775` | Compte verrouillé |

---

## 10. Points ouverts

Le 802.1X n'est pas réalisable en l'état : il suppose un commutateur capable de
bloquer un port jusqu'à authentification, fonction absente des réseaux virtuels
de l'hyperviseur employé. La configuration EAP nécessiterait par ailleurs de
résoudre l'incompatibilité MS-CHAPv2 décrite en section 5.

Le serveur RADIUS présente encore le certificat auto-signé livré par défaut pour
les méthodes EAP. Un certificat émis par l'autorité interne serait requis pour
un déploiement 802.1X.

La comptabilisation RADIUS n'est pas activée. Elle constituerait une source
d'événements pertinente pour la phase de supervision.

Les fichiers de configuration du serveur RADIUS contiennent le mot de passe du
compte de service et les secrets partagés en clair. Ils sont inclus dans la
sauvegarde, ce qui rend l'archive sensible et justifie la restriction d'accès du
répertoire de destination.
