# FW01, configuration du pare-feu

> Document de référence décrivant l'état courant de `FW01`. Le raisonnement
> ayant conduit à ces choix est consigné dans les entrées de journal des
> sessions 2 et 3. Les règles de filtrage découlent de `matrice-de-flux.md`.

---

## 1. Identité

| Élément | Valeur |
|---|---|
| Rôle | Routage inter-VLAN, filtrage, serveur DHCP |
| Système | OPNsense 26.7 (amd64), base FreeBSD |
| Nom d'hôte | `FW01` |
| Domaine | `lab.internal` |
| Fuseau horaire | `Europe/Paris` |
| Ressources | 2 Go de mémoire, 1 processeur virtuel, disque de 20 Go, UFS |

Le choix d'OPNsense plutôt que de pfSense repose sur la disponibilité des
images en téléchargement direct depuis la source officielle, accompagnées des
empreintes et de leur signature. Les deux distributions partagent la même base
FreeBSD et le même moteur de filtrage.

---

## 2. Interfaces

| Rôle OPNsense | Interface | Adresse MAC | Réseau virtuel | Adresse | Segment |
|---|---|---|---|---|---|
| WAN | `em0` | `00:50:56:28:FA:A2` | VMnet8 (NAT) | DHCP, `192.168.9.128/24` | Sortie Internet |
| OPT1 `POSTES` | `em1` | `00:50:56:20:FA:68` | VMnet2 | `10.10.10.1/24` statique | VLAN 10 |
| LAN `SERVEURS` | `em2` | `00:50:56:24:F9:22` | VMnet3 | `10.10.20.1/24` statique | VLAN 20 |
| OPT2 `INVITES` | `em3` | `00:50:56:2D:F1:25` | VMnet4 | `10.10.30.1/24` statique | VLAN 30 |

**Les interfaces internes sont impérativement en adressage statique.** Une
interface interne porte la passerelle de son segment, elle ne la reçoit pas.
Une configuration en client DHCP provoque la panne décrite dans le journal de
la session 3.

Aucune passerelle amont n'est déclarée sur les interfaces internes. Elle
créerait une route par défaut concurrente de celle du WAN et rendrait la table
de routage incohérente.

**Le segment SERVEURS est assigné au rôle LAN** et non le segment POSTES.
L'interface LAN reçoit par défaut une protection anti-verrouillage sur l'accès
d'administration. Le poste d'administration étant présent sur ce segment, ce
choix garantit l'accès à l'interface web en toute circonstance.

### Passerelles

| Nom | Interface | Adresse | Rôle |
|---|---|---|---|
| `WAN_DHCP` | WAN | `192.168.9.2` | Passerelle par défaut IPv4, unique |

Aucune autre passerelle IPv4 ne doit exister. Une passerelle créée
automatiquement sur une interface interne détourne la route par défaut vers une
adresse inexistante.

---

## 3. Résolution de noms

| Paramètre | Valeur |
|---|---|
| Résolveurs amont déclarés sur FW01 | `1.1.1.1` |
| Substitution par le DHCP du WAN | Désactivée |

La désactivation de la substitution empêche le serveur DHCP amont d'imposer ses
propres résolveurs et d'écraser la configuration choisie.

---

## 4. Service DHCP

Serveur retenu : **Kea DHCPv4**, successeur d'ISC DHCP.

| Paramètre général | Valeur |
|---|---|
| Interfaces d'écoute | `LAN`, `POSTES`, `INVITES` |
| Durée de bail | `86400` secondes, soit 24 heures |
| Création automatique des règles de pare-feu | Activée |

L'interface WAN est volontairement exclue. Un serveur DHCP répondant côté
extérieur constitue une faute de configuration grave sur un pare-feu.

### Sous-réseaux déclarés

| Sous-réseau | Plage dynamique | Routeur | DNS annoncé | Domaine |
|---|---|---|---|---|
| `10.10.10.0/24` | `10.10.10.100` à `10.10.10.199` | `10.10.10.1` | `10.10.10.1` | `lab.internal` |
| `10.10.20.0/24` | `10.10.20.100` à `10.10.20.199` | `10.10.20.1` | `10.10.20.1` | `lab.internal` |
| `10.10.30.0/24` | `10.10.30.100` à `10.10.30.199` | `10.10.30.1` | `10.10.30.1` | aucun |

Les plages respectent le découpage figé dans `plan-adressage.md` : la zone
`.10` à `.49`, réservée aux adresses fixes, reste libre de tout bail.

La collecte automatique des options a été désactivée au profit d'une
déclaration explicite du routeur et du serveur DNS. Une configuration explicite
se relit et se compare ; une configuration implicite suppose de connaître le
comportement par défaut de l'outil.

### Action en attente

> **À traiter lors de la mise en service de `DC01` comme serveur DNS.** Le
> serveur annoncé sur `10.10.10.0/24` et `10.10.20.0/24` doit passer à
> `10.10.20.10`. La séquence complète de bascule figure dans
> `matrice-de-flux.md`, section 4.

---

## 5. Alias de filtrage

| Nom | Type | Contenu |
|---|---|---|
| `DC01` | Hôte | `10.10.20.10` |
| `PORTS_WEB` | Ports | `80`, `443` |
| `PORTS_DNS` | Ports | `53` |
| `PORTS_ADMIN` | Ports | `443`, `22` |

Les alias évitent la duplication d'une valeur dans plusieurs règles. Une
modification se répercute alors depuis un point unique, sans relecture de
l'ensemble du jeu de règles.

---

## 6. Règles de filtrage

Toutes les règles sont en mode `Rapide`, direction `Entrée`, famille `IPv4`.
L'ordre indiqué est l'ordre d'évaluation.

### Interface POSTES

| # | Action | Protocole | Source | Destination | Port | Journal |
|---|---|---|---|---|---|---|
| 1 | Autoriser | UDP | POSTES net | any | `67` | non |
| 2 | Autoriser | TCP/UDP | POSTES net | Ce pare-feu | `PORTS_DNS` | non |
| 3 | Autoriser | tous | POSTES net | `DC01` | tous | non |
| 4 | Bloquer | tous | POSTES net | Ce pare-feu | tous | oui |
| 5 | Bloquer | TCP/UDP | POSTES net | any | `PORTS_DNS` | oui |
| 6 | Autoriser | TCP | POSTES net | any | `PORTS_WEB` | non |

La règle 1 a pour destination `any` et non `Ce pare-feu` : une requête DHCP
initiale est émise depuis `0.0.0.0` vers `255.255.255.255`, le client ne
connaissant pas encore l'adresse du serveur. Le filtrage reste sûr, la règle
étant portée par l'interface et limitée au port 67 en UDP.

La règle 2 est transitoire et disparaîtra avec la bascule DNS.

### Interface LAN, segment SERVEURS

| # | Action | Protocole | Source | Destination | Port | Journal |
|---|---|---|---|---|---|---|
| 1 | Autoriser | ICMP echo | LAN net | any | | non |
| 2 | Autoriser | UDP | LAN net | any | `67` | non |
| 3 | Autoriser | TCP | LAN net | Ce pare-feu | `PORTS_ADMIN` | non |
| 4 | Autoriser | TCP/UDP | LAN net | Ce pare-feu | `PORTS_DNS` | non |
| 5 | Autoriser | TCP/UDP | `DC01` | any | `PORTS_DNS` | non |
| 6 | Bloquer | TCP/UDP | LAN net | any | `PORTS_DNS` | oui |
| 7 | Autoriser | TCP | LAN net | any | `PORTS_WEB` | non |
| 8 | Autoriser | UDP | LAN net | any | `123` | non |

L'ordre entre les règles 5 et 6 est critique. `DC01` appartient à `LAN net` et
serait donc concerné par le blocage : seule l'autorisation spécifique placée
au-dessus le préserve. Inversées, la résolution externe s'effondre pour
l'ensemble du domaine.

Les règles `Default allow LAN to any` générées à l'installation ont été
désactivées après validation du jeu de règles ci-dessus.

### Interface INVITES

| # | Action | Protocole | Source | Destination | Port | Journal |
|---|---|---|---|---|---|---|
| 1 | Autoriser | UDP | INVITES net | any | `67` | non |
| 2 | Autoriser | TCP/UDP | INVITES net | Ce pare-feu | `PORTS_DNS` | non |
| 3 | Bloquer | tous | INVITES net | POSTES net | tous | oui |
| 4 | Bloquer | tous | INVITES net | LAN net | tous | oui |
| 5 | Bloquer | tous | INVITES net | Ce pare-feu | tous | oui |
| 6 | Autoriser | tous | INVITES net | any | tous | non |

Les blocages 3, 4 et 5 sont indispensables : la règle 6 autorise `any`, qui
englobe les réseaux internes ainsi que le pare-feu lui-même. La règle 5 couvre
l'adresse portée par `FW01` sur le segment invité, laquelle n'appartient ni à
`POSTES net` ni à `LAN net`.

---

## 7. Services résiduels et points ouverts

`dnsmasq` reste actif comme résolveur sur le port `53053`, valeur employée
lorsque `unbound` occupe déjà le port 53. Sa partie DHCP a été désactivée. Son
utilité réelle reste à clarifier.

Le résolveur renvoie des enregistrements AAAA alors que la maquette ne dispose
d'aucune connectivité IPv6. Les clients tentent donc l'IPv6 avant de basculer
en IPv4, ce qui ajoute une latence sur chaque connexion sans symptôme visible.

---

## 8. Sauvegarde et retour arrière

| Mécanisme | Emplacement | Portée |
|---|---|---|
| Export XML de configuration | Hors dépôt, stockage chiffré | Configuration complète d'OPNsense |
| Instantané de machine virtuelle | Hyperviseur | État système complet |

L'export XML contient les empreintes de mots de passe et le matériel
cryptographique de l'interface web : il ne doit en aucun cas être versionné.

Instantanés de référence :

- `FW01 base fonctionnelle, routage et DHCP`
- `FW01 filtrage actif, regles par defaut desactivees`
