# Session 2, mise en service du pare-feu

**Objectif.** Déployer `FW01`, assigner ses quatre interfaces conformément au
plan d'adressage, et mettre en service le routage inter-VLAN ainsi que la
distribution d'adresses sur les trois segments internes.

**Résultat.** Objectif atteint. Un incident bloquant rencontré et résolu.

---

## Préparation des réseaux virtuels

Un réseau virtuel par segment, de type privé, avec **le serveur DHCP de
l'hyperviseur désactivé** sur chacun. Deux serveurs DHCP concurrents sur un même
domaine de diffusion produisent des attributions incohérentes et des conflits
d'adresses.

L'adaptateur hôte n'est connecté que sur le segment SERVEURS, avec l'adresse
`10.10.20.2` attribuée manuellement et **sans passerelle par défaut**. Deux
raisons :

- Le poste d'administration n'a rien à faire dans les segments POSTES et
  INVITES. L'y raccorder créerait un chemin contournant le pare-feu, soit
  exactement ce qu'une segmentation cherche à empêcher.
- L'absence de passerelle évite qu'une route par défaut concurrente n'entre en
  conflit avec la connexion réseau réelle de l'hôte.

L'adresse `.2` a été retenue parce que l'hyperviseur attribue par défaut `.1` à
son adaptateur hôte, adresse déjà réservée au pare-feu dans le plan
d'adressage.

Le WAN passe par le réseau NAT de l'hyperviseur plutôt que par un pont. La
maquette est ainsi totalement isolée du réseau domestique, sur lequel un serveur
assure déjà le DHCP de l'ensemble du foyer.

---

## Incident 1, image système périmée et de provenance incertaine

**Constat.** L'image initialement téléchargée était une version datant de 2021,
soit quatre versions majeures de retard, en fin de vie et sans correctif de
sécurité. L'éditeur ne distribuant plus d'image en téléchargement direct, le
fichier provenait selon toute vraisemblance d'un miroir tiers, sans vérification
d'empreinte.

**Portée.** Il s'agissait d'installer, comme pare-feu de l'infrastructure, une
image système non maintenue et d'origine non vérifiée. C'est le scénario type
d'une compromission par la chaîne d'approvisionnement.

**Correction.** Bascule vers OPNsense 26.7, distribué en téléchargement direct
depuis la source officielle, avec empreintes et signature publiées. Les deux
distributions partagent la même base FreeBSD et le même moteur de filtrage : les
compétences sont transférables.

**Limite de la vérification effectuée.** L'empreinte a été calculée sur le
fichier `.iso` décompressé, alors que les valeurs publiées portent sur l'archive
`.iso.bz2` telle qu'elle est distribuée. La comparaison était donc impossible.

> **Enseignement.** Une empreinte se vérifie sur le fichier tel qu'il est
> distribué, jamais après transformation. Et calculer une empreinte ne prouve
> rien tant qu'elle n'est pas confrontée à une référence authentique.
>
> À distinguer par ailleurs : une empreinte détecte une corruption accidentelle,
> une signature cryptographique prouve l'origine. L'éditeur publie les deux.

---

## Assignation des interfaces

Les adresses MAC des quatre cartes ont été relevées dans l'hyperviseur avant le
premier démarrage, et associées à leur réseau virtuel respectif.

**L'assignation automatique s'est trompée.** Le système a retenu les deux
premières interfaces énumérées comme LAN et WAN, laissant les deux autres non
assignées. L'auto-détection s'appuie sur l'ordre d'énumération, pas sur l'usage
prévu. Sur un équipement réel disposant de plusieurs ports, elle se trompe
presque systématiquement.

La liste des MAC a permis de corriger en une passe, sans procéder par essais.
C'est la transposition d'un réflexe de salle serveur : relever les adresses
avant le brassage plutôt que débrancher des câbles pour observer ce qui tombe.

Assignation finale documentée dans `docs/configuration-fw01.md`.

**Piège de séquence.** Contrairement à d'autres distributions, l'installateur
demande l'interface LAN en premier et la WAN ensuite. Répondre au rythme plutôt
qu'à la question conduit à inverser les deux.

L'adresse LAN par défaut, `192.168.1.1`, correspond à celle de la passerelle du
réseau domestique. Elle a été remplacée par `10.10.20.1` avant tout autre
réglage.

---

## Vérification hors bande du certificat

L'empreinte du certificat de l'interface web, affichée sur la console de la
machine virtuelle, a été comparée à celle présentée par le navigateur avant
d'accepter l'exception de sécurité.

Le canal fournissant la référence, à savoir la console, est distinct du canal à
valider, à savoir la connexion réseau. C'est ce qui rend une interposition
impossible sur cette connexion. Un certificat auto-signé accepté sans cette
comparaison n'apporte aucune garantie d'identité.

---

## Incident 2, conflit sur le port DHCP

**Symptôme.** Après configuration des trois sous-réseaux, le journal du service
DHCP affichait, à chaque tentative de démarrage :

```
DHCPSRV_OPEN_SOCKET_FAIL failed to open socket on interface em1,
  reason: failed to bind fallback socket to address 10.10.10.1, port 67,
  reason: Address already in use - is another DHCP server running?
DHCPSRV_NO_SOCKETS_OPEN no interface configured to listen to DHCP traffic
```

**Lecture.** Le port UDP 67 est celui du serveur DHCP, et un seul processus peut
l'occuper par interface. L'échec portait sur les trois interfaces
simultanément, ce qui écartait un problème propre à un segment. La seconde ligne
confirmait que le service tournait sans écouter nulle part : actif, mais sourd.

**Diagnostic.** Identification du processus détenteur plutôt que supposition
sur son identité :

```
sockstat -4 -l | grep ':67'
```

Avant correction :

```
root  dnsmasq  10004  4  udp4  *:67
```

L'astérisque indique une liaison sur l'ensemble des interfaces, ce qui explique
l'échec simultané sur les trois.

**Cause.** Les versions récentes de la distribution activent `dnsmasq` comme
service DNS et DHCP par défaut sur une nouvelle installation. Deux serveurs DHCP
coexistaient donc sur le même hôte sans qu'aucun n'ait été installé
explicitement.

**Correction.** Suppression des plages DHCP déclarées dans `dnsmasq`, le service
n'ouvrant le port 67 que s'il a au moins une plage à servir. Le service DNS
n'a pas été arrêté. Désactivation de la création automatique des règles de
pare-feu associées, devenue sans objet.

Après correction :

```
root  kea-dhcp4  46126  18  udp4  10.10.10.1:67
root  kea-dhcp4  46126  20  udp4  10.10.20.1:67
root  kea-dhcp4  46126  22  udp4  10.10.30.1:67
```

> **Enseignements.**
>
> Deux serveurs DHCP ne cohabitent pas, sur un même réseau comme sur un même
> hôte. Le message d'erreur posait lui-même la bonne question.
>
> Un service peut être actif sans rendre le moindre service. La vérification
> utile porte sur les sockets réellement ouvertes, pas sur l'état du démon.
>
> Différence de comportement notable entre les deux implémentations : l'une se
> lie à toutes les interfaces d'un coup, l'autre à chaque adresse nommément. La
> seconde approche rend visible et intentionnel ce qui écoute où.
>
> Corollaire de méthode : tant que la commande n'a pas répondu, l'explication
> la plus plausible reste une hypothèse. Agir sur une hypothèse plausible est
> la façon la plus courante de couper le mauvais service.

---

## Point en suspens

`dnsmasq` reste actif comme résolveur, sur le port `53053`. Cette valeur est
employée lorsque `unbound` occupe déjà le port 53 standard, ce qui laisse
supposer que `dnsmasq` ne sert de résolveur à personne. À clarifier avant la
mise en service du DNS de l'annuaire, pour éviter de conserver un service
résiduel dont l'usage ne serait plus compris dans six mois.

---

## État à la fin de la session

- `FW01` opérationnel, quatre interfaces assignées et adressées
- Routage et traduction d'adresses fonctionnels, validés par un test de
  connectivité vers une adresse publique
- Distribution d'adresses active sur les trois segments internes
- Segments POSTES et INVITES en refus total, aucune règle de filtrage écrite
- Configuration exportée hors dépôt, instantané de machine virtuelle pris

## Prochaine session

Formalisation de la matrice de flux, puis écriture des règles de filtrage
inter-VLAN. La matrice est établie avant toute saisie, selon le même principe
que le plan d'adressage : décider d'abord, configurer ensuite.
