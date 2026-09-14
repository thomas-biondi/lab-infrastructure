# SRV01, serveur de services Linux

> Document de référence décrivant l'état courant de `SRV01`. Le raisonnement
> figure dans `journal/session-06-serveur-linux.md`.

---

## 1. Identité

| Élément | Valeur |
|---|---|
| Rôle | Services, sauvegardes, automatisation |
| Système | Debian 13 Trixie, installation minimale |
| Nom d'hôte | `SRV01` |
| Domaine | `lab.internal` |
| Adresse | `10.10.20.20/24`, statique |
| Passerelle | `10.10.20.1` |
| Résolveur | `10.10.20.10` |
| Segment | VLAN 20 SERVEURS |
| Ressources | 2 Go de mémoire, 2 processeurs virtuels, disque de 40 Go |

---

## 2. Partitionnement

Gestionnaire de volumes logiques, avec partitions séparées pour `/home`, `/var`
et `/tmp`.

**LVM** insère une couche d'abstraction entre les disques physiques et les
systèmes de fichiers, autorisant l'extension d'un volume à chaud, l'ajout d'un
disque ou la prise d'instantanés. Sans cette couche, une partition saturée se
traite par réinstallation.

**La séparation de `/var`** est la plus importante sur un serveur : ce point de
montage accueille les journaux et les données applicatives. Non séparé, un
service écrivant massivement remplit la racine, et un système de fichiers racine
plein devient inutilisable, souvent irréparable sans redémarrage. La séparation
confine le dégât.

Cette organisation est par ailleurs exigée par la plupart des référentiels de
durcissement, notamment parce qu'elle permet de monter `/tmp` avec des options
restrictives.

---

## 3. Surface d'attaque

Sélection de paquets limitée au serveur SSH et aux utilitaires usuels. Aucun
environnement de bureau, aucun serveur web.

Le principe est celui de la surface minimale : ce qui n'est pas installé ne peut
être exploité, ne nécessite aucun correctif et ne consomme rien. L'installation
résultante occupe moins de 2 Go contre plus de 8 Go avec un environnement
graphique.

La participation aux statistiques d'utilisation de paquets a été refusée : elle
installe un service transmettant périodiquement la liste des paquets installés
vers l'extérieur, soit une divulgation d'information sur l'infrastructure et un
flux sortant non décidé.

---

## 4. Configuration réseau

Gestionnaire retenu : `ifupdown`, via `/etc/network/interfaces`.

```
auto ens33
iface ens33 inet static
    address 10.10.20.20/24
    gateway 10.10.20.1
    dns-nameservers 10.10.20.10
    dns-search lab.internal
```

Le paquet `resolvconf` est requis pour que les directives `dns-nameservers` et
`dns-search` produisent un effet. Sans lui, ces lignes sont lues et
silencieusement ignorées, laissant le système sans résolveur.

**Un seul gestionnaire de configuration réseau doit être actif.** Plusieurs
outils peuvent prétendre gérer une interface : `ifupdown`, `NetworkManager`,
`systemd-networkd`, `dhcpcd`. Leur coexistence produit des pannes intermittentes
difficiles à diagnostiquer, la configuration variant selon le dernier
intervenant. Identifier quel outil gère réellement une interface est le premier
réflexe face à un incident réseau.

---

## 5. Accès distant

### Authentification

Par clé publique Ed25519 exclusivement. Le mot de passe est désactivé.

La clé privée demeure sur le poste d'administration, protégée par une phrase de
passe. Seule la clé publique est déposée sur le serveur. À l'authentification,
le serveur émet un défi que le client signe avec la clé privée : **celle-ci ne
traverse jamais le réseau**, contrairement à un mot de passe transmis à chaque
connexion, fût-ce dans un tunnel chiffré.

Permissions requises, sans lesquelles le serveur ignore la clé sans message
explicite : `700` sur `~/.ssh`, `600` sur `authorized_keys`.

### Durcissement

Fichier `/etc/ssh/sshd_config.d/99-durcissement.conf`, le répertoire d'inclusion
étant préféré au fichier principal afin de survivre aux mises à jour du paquet.

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers user
```

| Directive | Motif |
|---|---|
| `PermitRootLogin no` | Supprime la cible la plus attaquée, dont le nom n'est pas à deviner. Le passage par un compte nommé puis élévation impose de trouver deux éléments et conserve la traçabilité |
| `PasswordAuthentication no` | Rend la force brute inopérante. Sans cette directive, le reste est cosmétique |
| `MaxAuthTries 3` | Limite le nombre d'essais par connexion |
| `LoginGraceTime 30` | Limite la durée d'une tentative |
| `AllowUsers` | Liste blanche : tout compte non listé est refusé, même avec une clé valide |

`PasswordAuthentication no` est réécrit explicitement bien qu'il puisse
correspondre à un défaut : un défaut évolue avec les versions, une directive
non.

Vérification par `sshd -T`, qui affiche la configuration effective toutes
inclusions résolues. Lire les fichiers un par un ne renseigne pas sur la valeur
qui l'emporte.

---

## 6. Mises à jour

`unattended-upgrades` configuré pour appliquer les correctifs de sécurité
uniquement, à l'exclusion des mises à jour fonctionnelles. Un serveur non mis à
jour constitue la première cause de compromission.

**Limite à connaître** : ce mécanisme ne redémarre pas les services utilisant
encore en mémoire une bibliothèque corrigée, et un noyau mis à jour ne prend
effet qu'au redémarrage. La commande `needrestart` signale ce qui reste à
relancer.

---

## 7. Pare-feu local

| Politique | Valeur |
|---|---|
| Entrant | Refus par défaut |
| Sortant | Autorisé |
| Exception | SSH depuis `10.10.20.0/24` uniquement |

**Un pare-feu réseau ne protège pas des machines du même segment.** `DC01`,
`SRV01` et `SIEM01` partagent `10.10.20.0/24` et communiquent directement sans
traverser `FW01`. La compromission de l'un n'est freinée par rien au niveau
réseau.

C'est le principe de défense en profondeur : plusieurs couches indépendantes,
aucune n'étant supposée parfaite. La restriction de source sur SSH fait que la
machine se protège elle-même, indépendamment de l'exactitude des règles du
pare-feu réseau.

Ordre d'application impératif : politiques par défaut, puis autorisations, puis
activation. Activer avant d'autoriser SSH coupe l'accès immédiatement.

---

## 8. Détection d'intrusion

`fail2ban`, prison `sshd` activée.

| Paramètre | Valeur |
|---|---|
| `maxretry` | 3 |
| `findtime` | 10 minutes |
| `bantime` | 1 heure |

Trois échecs dans une fenêtre de dix minutes déclenchent un bannissement d'une
heure. Le bannissement se matérialise par une règle insérée dans le pare-feu
local, vérifiable par `nft list ruleset`.

**Chaîne validée par déclenchement volontaire** : tentatives répétées depuis le
poste d'administration, bannissement constaté, puis levée. Une protection dont
on n'a pas observé le déclenchement ne compte pas.

---

## 9. Sauvegarde des configurations

Script `/usr/local/bin/sauvegarde-config.sh`, déclenché quotidiennement par un
minuteur `systemd`.

### Périmètre

`/etc/ssh`, `/etc/network`, `/etc/fail2ban`, `/etc/ufw`, ainsi que la liste des
paquets installés. Cette dernière est fréquemment omise alors qu'elle seule
permet de reconstituer un serveur identique.

Sur ce serveur, la donnée coûteuse à reconstruire n'est pas un fichier
utilisateur mais une configuration : le coût réside dans le temps passé à
retrouver quel réglage se situait où.

### Mécanismes

| Mécanisme | Effet |
|---|---|
| `set -euo pipefail` | Arrêt à la première erreur, refus des variables non définies, propagation de l'échec dans un enchaînement |
| Vérification d'intégrité | `tar -tzf` avant toute rotation ; une archive illisible est supprimée et le script sort en échec |
| Rotation | Suppression au-delà de 7 jours, **après** validation de la nouvelle archive |
| Journalisation | Via `logger`, donc dans le journal système, horodatée et collectable par la supervision |
| Code de sortie | `1` en cas d'échec, condition nécessaire à toute supervision |

Sans `set -e`, un script échouant à créer son archive poursuivrait jusqu'à la
rotation et supprimerait les anciennes sauvegardes sans en avoir produit de
nouvelle. Sans `set -u`, une faute de frappe dans un nom de variable produit une
chaîne vide, avec les conséquences que l'on imagine sur un chemin de
suppression.

### Planification

Minuteur `systemd` plutôt que `cron` : la sortie est journalisée
automatiquement, un échec apparaît dans l'état du service, et `Persistent=true`
rattrape une exécution manquée si la machine était éteinte.

`RandomizedDelaySec=900` décale l'exécution d'un délai aléatoire. Sans effet sur
une machine isolée, indispensable sur un parc dont les serveurs sauvegarderaient
simultanément vers un même stockage.

### Restauration vérifiée

Extraction dans un répertoire temporaire, puis comparaison récursive avec
l'original. Absence de différence constatée.

Une sauvegarde dont la restauration n'a pas été vérifiée n'est pas une
sauvegarde.

---

## 10. Points ouverts

Le répertoire de destination des archives est en `700`, ce qui rend l'expansion
des motifs impossible depuis un compte non privilégié : l'interprétation d'un
joker relève du shell appelant, avant l'élévation de privilèges. Les opérations
de restauration nomment l'archive explicitement.

Les archives ne sont pas répliquées hors de la machine. La règle des trois
copies sur deux supports dont un hors site n'est pas satisfaite.

`sudo` a été ajouté après l'installation, un mot de passe root ayant été défini
au déploiement. La justification du choix reste valable : `su` ne trace rien,
toutes les actions apparaissant sous l'identité root, tandis que `sudo`
journalise chaque commande avec l'identité réelle de l'appelant et permet une
délégation fine.
