# Plan d'adressage

> Document de référence. Toute adresse utilisée dans la maquette est définie ici
> et nulle part ailleurs. Une modification de ce fichier implique de répercuter
> le changement sur les équipements concernés, listés en fin de document.

---

## 1. Choix structurants

| Décision | Choix retenu | Justification |
|---|---|---|
| Plage privée | `10.10.0.0/16` | Évite toute collision avec le réseau domestique (`192.168.1.0/24`), plage la plus courante en entreprise |
| Masque | `/24` par segment | Le masque tombe sur un octet entier : lecture immédiate de la partie réseau |
| Corrélation VLAN / IP | 3ᵉ octet = identifiant de VLAN | Une adresse suffit à identifier le segment, sans consulter la documentation |
| Passerelle | `.1` dans chaque segment | Convention la plus répandue ; cohérence sur l'ensemble des segments |
| Domaine AD | `lab.internal` | `.internal` est réservé par l'ICANN (2024) aux usages privés. `.local` est proscrit (conflit mDNS) |
| Nom NetBIOS | `LAB` | - |

VLAN 1 volontairement inutilisé : c'est le VLAN par défaut des équipements Cisco,
laisser du trafic de production y circuler est une mauvaise pratique constante.

---

## 2. Segments

| VLAN | Nom | Sous-réseau | Passerelle | Diffusion | Rôle |
|---|---|---|---|---|---|
| - | WAN | DHCP (VMware NAT) | - | - | Sortie Internet de la maquette |
| 10 | POSTES | `10.10.10.0/24` | `10.10.10.1` | `10.10.10.255` | Postes clients joints au domaine |
| 20 | SERVEURS | `10.10.20.0/24` | `10.10.20.1` | `10.10.20.255` | Contrôleur de domaine, services, supervision |
| 30 | INVITES | `10.10.30.0/24` | `10.10.30.1` | `10.10.30.255` | Segment isolé, accès Internet uniquement |

Identifiants réservés pour extension : **40** (voix / ToIP), **99** (administration
des équipements réseau).

---

## 3. Découpage interne d'un `/24`

Appliqué à l'identique dans chaque segment.

| Plage | Usage | Mode |
|---|---|---|
| `.1` | Passerelle | Fixe |
| `.2 – .9` | Infrastructure réseau (extension future) | Fixe |
| `.10 – .49` | Serveurs | Fixe |
| `.50 – .99` | Libre | - |
| `.100 – .199` | Pool DHCP dynamique | Dynamique |
| `.200 – .249` | Réservations DHCP par adresse MAC | Dynamique fixée |
| `.250 – .254` | Tests, machines temporaires | Fixe |

**Adresse fixe ou réservation DHCP ?** Une adresse fixe est configurée sur la
machine : elle survit à une panne du serveur DHCP. Réservée aux équipements dont
l'indisponibilité empêcherait le réseau de fonctionner (passerelle, contrôleur de
domaine, DNS). Une réservation DHCP est configurée sur le serveur : la machine
reste en configuration automatique et son adresse se modifie depuis un point
unique. C'est le mode par défaut pour tout le reste.

Séparer les deux plages évite le conflit d'adresse entre une machine configurée
manuellement et un bail attribué automatiquement - incident intermittent et
coûteux à diagnostiquer.

---

## 4. Attribution des machines

| Nom | Segment | Adresse | Mode | Rôle |
|---|---|---|---|---|
| `FW01` | tous | `10.10.10.1` / `10.10.20.1` / `10.10.30.1` | Fixe | pfSense - routage inter-VLAN, pare-feu, DHCP |
| `DC01` | VLAN 20 | `10.10.20.10` | Fixe | Windows Server - AD DS, DNS de la forêt |
| `SRV01` | VLAN 20 | `10.10.20.20` | Fixe | Debian - services, scripts, sauvegardes |
| `SIEM01` | VLAN 20 | `10.10.20.30` | Fixe | Wazuh - collecte et analyse des journaux |
| `PC01` | VLAN 10 | pool DHCP | Dynamique | Client Windows 11 joint au domaine |

Nomenclature : `<ROLE><NN>`, deux chiffres pour permettre l'ajout d'un second
équipement de même rôle sans renommer le premier.

---

## 5. Résolution de noms

| Segment | Serveur DNS annoncé par DHCP | Justification |
|---|---|---|
| VLAN 10 | `10.10.20.10` (DC01) | Un client de domaine **doit** interroger le DNS de la forêt : les enregistrements SRV conditionnent la jonction au domaine, l'authentification Kerberos et l'application des GPO |
| VLAN 20 | `10.10.20.10` (DC01) | Idem |
| VLAN 30 | `10.10.30.1` (FW01) | Segment invité : aucun accès à l'annuaire |

`DC01` transfère les requêtes externes vers un résolveur public. **Aucun poste du
domaine ne doit recevoir un résolveur public en DNS primaire** : c'est la
première cause de dysfonctionnement dans un environnement Active Directory.

---

## 6. Correspondance avec les réseaux virtuels VMware

| Segment | VMnet | Type | DHCP VMware |
|---|---|---|---|
| WAN | `VMnet8` | NAT | Actif (fournit l'adresse WAN de FW01) |
| VLAN 10 | `VMnet2` | Custom | **Désactivé** |
| VLAN 20 | `VMnet3` | Custom | **Désactivé** |
| VLAN 30 | `VMnet4` | Custom | **Désactivé** |

La désactivation du serveur DHCP de VMware sur les segments internes est
impérative : deux serveurs DHCP concurrents sur un même domaine de diffusion
produisent des attributions incohérentes (passerelle et DNS divergents) et des
conflits d'adresses.

**Limite assumée.** Les réseaux virtuels de VMware Workstation sont des
commutateurs plats, sans étiquetage 802.1Q. La segmentation reproduite ici est
fonctionnellement équivalente (sous-réseaux distincts, routage et filtrage
centralisés sur le pare-feu) mais ne met pas en œuvre l'encapsulation 802.1Q
elle-même - trunk, étiquette de 4 octets, VLAN natif. Cette partie est traitée
séparément sur équipement réel ou émulé. Les identifiants de VLAN sont conservés
dans la nomenclature pour que la maquette reste transposable sans renumérotation.

---

## 7. Points de répercussion

En cas de modification de ce document, mettre à jour :

- [ ] Interfaces et adresses de FW01
- [ ] Étendues, options et réservations DHCP sur FW01
- [ ] Règles de filtrage inter-VLAN
- [ ] Adresse fixe et zone DNS sur DC01
- [ ] Adresses fixes de SRV01 et SIEM01
- [ ] Cibles de collecte et agents déclarés sur SIEM01
- [ ] Configuration des VMnet dans le Virtual Network Editor
