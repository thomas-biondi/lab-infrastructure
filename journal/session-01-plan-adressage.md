# Session 1 - Plan d'adressage et conventions

**Objectif.** Figer l'adressage, la nomenclature et le nom de domaine avant tout
déploiement. Aucune machine virtuelle créée à ce stade.

---

## Pourquoi commencer par là

Une adresse IP ne reste jamais à un seul endroit : elle se retrouve dans le DNS,
les baux DHCP, les règles de pare-feu, les fichiers de configuration, les
certificats et les agents de supervision. Modifier un sous-réseau en cours de
projet ne consiste pas à changer une ligne, mais à retrouver tous les endroits
où la valeur a été recopiée.

Un plan d'adressage est par ailleurs un livrable en soi. C'est le premier
document demandé lors d'une prise en main de système d'information, et son
absence est le premier symptôme d'une infrastructure non maîtrisée.

---

## Décisions prises et raisonnement

### Plage privée : `10.10.0.0/16`

Les trois blocs réservés à l'usage privé sont définis par la RFC 1918 :
`10.0.0.0/8`, `172.16.0.0/12` et `192.168.0.0/16`.

`192.168.1.0/24` et `192.168.0.0/24` sont écartés : ce sont les plages par défaut
de la quasi-totalité des box, dont celle du réseau domestique hébergeant la
maquette. Un routeur ayant le même réseau des deux côtés ne peut plus déterminer
si une destination est locale ou distante. La panne qui en résulte est
particulièrement trompeuse, puisque chaque configuration prise isolément paraît
correcte.

### Masque : `/24` uniformément

254 adresses utilisables par segment, très au-delà du besoin. Le critère retenu
n'est pas l'économie d'adresses mais la lisibilité : un masque aligné sur un
octet entier permet de lire directement la partie réseau et la partie hôte, sans
calcul. Un découpage plus fin serait justifié en production réelle.

### Corrélation entre identifiant de VLAN et troisième octet

VLAN 10 → `10.10.10.0/24`, VLAN 20 → `10.10.20.0/24`, VLAN 30 → `10.10.30.0/24`.

Bénéfice opérationnel : une adresse lue dans un journal d'événements suffit à
identifier le segment d'origine. Convention courante en entreprise.

VLAN 1 laissé inutilisé - VLAN par défaut des équipements Cisco, qu'il est
déconseillé d'employer pour du trafic de production. Identifiants 40 (voix) et
99 (administration) réservés dès maintenant, sans être créés.

### Passerelle en `.1`

Convention la plus répandue. Le point important n'est pas la valeur choisie mais
son uniformité sur l'ensemble des segments.

### Séparation des plages statiques et dynamiques

Sans cloisonnement, le serveur DHCP finit par attribuer une adresse déjà employée
en configuration manuelle. Le conflit qui en découle est intermittent et difficile
à rattacher à sa cause.

Distinction retenue entre les deux modes :

- **Adresse fixe** (`.10 – .49`) - configurée sur la machine, elle survit à une
  panne du DHCP. Réservée à ce dont l'indisponibilité empêcherait le réseau de
  fonctionner : passerelle, contrôleur de domaine, DNS.
- **Réservation DHCP** (`.200 – .249`) - configurée sur le serveur, associée à
  une adresse MAC. La machine reste en configuration automatique et son adresse
  se pilote depuis un point unique. Mode par défaut pour tout le reste.

### Nom de domaine : `lab.internal`

`.local` est écarté : ce suffixe est réservé au protocole mDNS et provoque des
comportements de résolution imprévisibles selon les systèmes d'exploitation.
Un domaine public non détenu est également écarté.

`.internal` a été réservé par l'ICANN en 2024 pour les usages strictement
internes, ce qui en fait le choix recommandé aujourd'hui.

### DNS annoncé aux clients

Les segments 10 et 20 reçoivent `10.10.20.10` - le contrôleur de domaine - et
lui seul. Un client de domaine qui interroge un résolveur public ne récupère pas
les enregistrements SRV de la forêt : il ne peut alors ni rejoindre le domaine,
ni s'authentifier en Kerberos, ni appliquer ses stratégies de groupe. C'est la
première cause de dysfonctionnement dans un environnement Active Directory.

Le segment invité reçoit la passerelle comme résolveur : il n'a aucune raison
d'accéder à l'annuaire.

---

## Correspondance VMware et limite assumée

Un réseau virtuel (VMnet) par segment, les VMnet internes en type *Custom* avec
**le serveur DHCP de VMware désactivé**. Deux serveurs DHCP concurrents sur un
même domaine de diffusion produisent des attributions incohérentes et des
conflits d'adresses.

Les réseaux virtuels de VMware Workstation sont des commutateurs plats : ils ne
gèrent pas l'étiquetage 802.1Q. La segmentation reproduite ici est
fonctionnellement équivalente - sous-réseaux distincts, routage et filtrage
centralisés - mais l'encapsulation elle-même (trunk, étiquette de 4 octets, VLAN
natif) devra être pratiquée séparément sur équipement réel ou émulé. Les
identifiants de VLAN sont conservés dans la nomenclature pour que la maquette
reste transposable sans renumérotation.

Le choix du NAT plutôt que du pont pour le WAN isole complètement la maquette du
réseau domestique, sur lequel un serveur assure déjà le DHCP de l'ensemble du
foyer.

---

## Incidents

Aucun - session de conception.

---

## État à la fin de la session

- Plan d'adressage figé et versionné (`docs/plan-adressage.md`)
- Nomenclature et nom de domaine arrêtés
- Aucune machine virtuelle déployée

## Prochaine session

Création des réseaux virtuels dans le *Virtual Network Editor*, puis déploiement
et configuration de `FW01` (pfSense) : interfaces, adressage, étendues DHCP,
premières règles de filtrage inter-VLAN.
