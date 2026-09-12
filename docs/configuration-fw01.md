# FW01, configuration du pare-feu

> Document de référence décrivant l'état courant de `FW01`. Le raisonnement
> ayant conduit à ces choix est consigné dans `journal/session-02-pare-feu.md`.

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
| OPT1 `POSTES` | `em1` | `00:50:56:20:FA:68` | VMnet2 | `10.10.10.1/24` | VLAN 10 |
| LAN `SERVEURS` | `em2` | `00:50:56:24:F9:22` | VMnet3 | `10.10.20.1/24` | VLAN 20 |
| OPT2 `INVITES` | `em3` | `00:50:56:2D:F1:25` | VMnet4 | `10.10.30.1/24` | VLAN 30 |

Aucune passerelle amont n'est déclarée sur les interfaces internes. Elles
porteraient une route par défaut concurrente de celle du WAN et rendraient la
table de routage incohérente.

**Le segment SERVEURS est assigné au rôle LAN** et non le segment POSTES.
L'interface LAN reçoit par défaut une règle d'autorisation large et une
protection anti-verrouillage sur l'accès d'administration. Le poste
d'administration étant présent sur ce segment, ce choix garantit l'accès à
l'interface web dès l'installation terminée.

Les adresses MAC ont été relevées dans l'hyperviseur avant le premier démarrage.
L'assignation automatique des interfaces s'appuie sur l'ordre d'énumération et
non sur l'usage prévu : elle s'est trompée, et la liste des MAC a permis de
corriger sans procéder par essais.

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

Le segment INVITES ne reçoit pas de suffixe de domaine : il n'appartiendra
jamais à l'annuaire et n'a pas à se voir suggérer des noms qu'il ne peut pas
résoudre.

### Action en attente

> **À traiter lors de la mise en service de `DC01`.** Le serveur DNS annoncé
> sur les sous-réseaux `10.10.10.0/24` et `10.10.20.0/24` doit être remplacé
> par `10.10.20.10`. Un poste membre d'un domaine Active Directory qui
> interroge un autre résolveur que le DNS de la forêt ne récupère pas les
> enregistrements SRV, et ne peut donc ni rejoindre le domaine, ni
> s'authentifier, ni appliquer ses stratégies de groupe.
>
> Le sous-réseau `10.10.30.0/24` conserve `10.10.30.1`.

---

## 5. Filtrage

État courant, avant écriture de la matrice de flux :

| Interface | Politique effective |
|---|---|
| LAN `SERVEURS` | Règle par défaut autorisant tout le trafic sortant |
| OPT1 `POSTES` | Aucune règle, donc refus total |
| OPT2 `INVITES` | Aucune règle, donc refus total |
| WAN | Refus, blocage des réseaux privés et des réseaux non alloués |

Une interface sans règle bloque l'intégralité du trafic. Ce n'est pas un
dysfonctionnement mais l'application du refus par défaut : ce qui n'est pas
explicitement autorisé est interdit.

Les règles définitives seront établies à partir d'une matrice de flux formalisée
avant toute saisie.

---

## 6. Services résiduels à examiner

`dnsmasq` est actif et écoute le DNS sur le port `53053`, valeur employée par
OPNsense lorsque `unbound` occupe déjà le port 53. Sa partie DHCP a été
désactivée, mais sa raison d'être en tant que résolveur reste à clarifier au
regard de la configuration d'`unbound`.

---

## 7. Sauvegarde et retour arrière

| Mécanisme | Emplacement | Portée |
|---|---|---|
| Export XML de configuration | Hors dépôt, stockage chiffré | Configuration complète d'OPNsense |
| Instantané de machine virtuelle | Hyperviseur | État système complet |

L'export XML contient les empreintes de mots de passe et le matériel
cryptographique de l'interface web : il ne doit en aucun cas être versionné.

Un instantané est pris avant toute intervention susceptible de couper l'accès
d'administration, notamment avant l'écriture des règles de filtrage. Instantané
de référence : `FW01 base fonctionnelle, routage et DHCP`.
