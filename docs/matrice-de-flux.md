# Matrice de flux

> Document de référence. Le filtrage appliqué sur `FW01` découle directement de
> cette matrice. Toute règle qui ne s'y rattache pas doit être remise en cause.

---

## 1. Principes retenus

**Refus par défaut.** Ce qui n'est pas explicitement autorisé est interdit. Une
interface dépourvue de règle bloque l'intégralité du trafic. Ce n'est pas un
dysfonctionnement mais le comportement attendu.

**Filtrage à état.** Une règle autorise l'établissement d'une connexion, pas un
sens de circulation. Les paquets de réponse sont reconnus comme appartenant à
une session existante. Aucune règle de retour n'est donc écrite : en ajouter
ouvrirait des accès non maîtrisés.

**Point d'application.** Une règle s'évalue sur l'interface par laquelle le
trafic entre dans le pare-feu. La source d'une règle appartient toujours au
segment de l'interface qui la porte.

**Ordre d'évaluation.** Les règles sont lues de haut en bas et la première
correspondance décide. Les exceptions spécifiques se placent au-dessus des
autorisations générales.

**Un blocage explicite ne se justifie que dans deux cas** : faire exception à
une autorisation plus large située au-dessus, ou journaliser un trafic précis.
Ailleurs, le refus par défaut suffit.

---

## 2. Matrice

| Source, vers | POSTES | SERVEURS | INVITES | FW01 | Internet |
|---|---|---|---|---|---|
| **POSTES** | non filtré | tout, vers `DC01` seul | refus | DHCP, DNS (transitoire) | HTTP, HTTPS. DNS refusé |
| **SERVEURS** | refus | non filtré | refus | DHCP, administration, DNS (transitoire) | HTTP, HTTPS, NTP, DNS depuis `DC01` seul |
| **INVITES** | refus | refus | non filtré | DHCP, DNS | tout sauf les réseaux internes |

Les cases diagonales portent la mention « non filtré » : deux machines d'un même
sous-réseau communiquent au niveau 2 sans passer par le pare-feu. Il n'y a rien
à autoriser et rien à interdire à ce niveau. Isoler des postes au sein d'un même
segment relèverait du commutateur, par private VLAN ou isolation de port.

---

## 3. Justification des choix

### POSTES vers SERVEURS, restreint à `DC01`

Un poste de travail a besoin de son contrôleur de domaine pour s'authentifier,
résoudre les noms, appliquer ses stratégies et synchroniser son horloge. Il n'a
aucune raison d'atteindre le serveur de services ni le collecteur de journaux.

**Tous les ports sont autorisés, mais vers cet hôte uniquement.** Active
Directory emploie une quinzaine de ports fixes auxquels s'ajoute une plage RPC
dynamique de 49152 à 65535. Un filtrage port par port reviendrait à ouvrir plus
de seize mille ports, ce qui viderait l'exercice de son sens tout en restant
fragile. La granularité porte donc sur la destination plutôt que sur le port.
C'est le compromis retenu par la majorité des organisations, et c'est une
décision assumée.

### SERVEURS vers POSTES, refus

Les stratégies de groupe, les mises à jour et la synchronisation horaire sont
toutes tirées par le client, jamais poussées par le serveur. Un serveur n'a donc
pratiquement jamais besoin d'initier une connexion vers un poste.

C'est précisément le chemin qu'emprunte un attaquant ayant compromis un serveur
pour rebondir vers les postes utilisateurs. Le fermer coûte peu et retire un
mouvement latéral.

### DNS, chemin de résolution unique

L'architecture impose un résolveur unique pour les membres du domaine :
`DC01`. Un poste interrogeant un autre résolveur ne récupère pas les
enregistrements SRV de la forêt, et ne peut alors ni rejoindre le domaine, ni
s'authentifier, ni appliquer ses stratégies.

Trois contrôles concourent au même objectif :

1. Le DNS sortant vers Internet est bloqué depuis POSTES et depuis SERVEURS.
2. Seul `DC01` est autorisé à interroger un résolveur public, afin d'assurer la
   résolution des noms externes pour l'ensemble du domaine.
3. Le DNS vers `FW01` sera fermé pour POSTES et SERVEURS à la mise en service de
   `DC01`.

**Limite connue.** Ces contrôles portent sur le port 53. Le DNS chiffré sur
HTTPS emprunte le port 443 et reste indétectable par un filtrage de port. Le
traiter supposerait un filtrage applicatif ou le blocage des résolveurs connus
par liste.

### Accès au pare-feu lui-même

Un pare-feu est à la fois un équipement de transit et une machine destinataire.
Les deux se traitent séparément : la destination `Ce pare-feu` désigne la
seconde.

Tous les segments doivent joindre `FW01` pour le DHCP. Seul le segment SERVEURS
accède à l'interface d'administration, le poste d'administration s'y trouvant.

**Point de vigilance identifié lors de la mise en œuvre.** Une autorisation vers
`any` englobe le pare-feu lui-même, y compris l'adresse qu'il porte sur le
segment source. Bloquer les réseaux voisins ne suffit donc pas : un blocage
explicite vers `Ce pare-feu` est nécessaire, et il doit être placé au-dessus de
l'autorisation générale pour être atteint.

### ICMP

L'écho ICMP sortant est autorisé depuis SERVEURS à des fins de diagnostic. Il
n'est pas ouvert depuis INVITES : un segment invité n'a pas à sonder le réseau.

---

## 4. Régime transitoire

Tant que `DC01` n'assure pas la résolution de noms, POSTES et SERVEURS
interrogent `FW01`. Les règles concernées portent la mention `TEMPORAIRE` dans
leur description.

**Séquence de bascule, à exécuter dans cet ordre le jour de la mise en service
de `DC01` :**

1. Modifier le serveur DNS annoncé par Kea sur les sous-réseaux
   `10.10.10.0/24` et `10.10.20.0/24`, de `10.10.x.1` vers `10.10.20.10`
2. Vérifier que la règle autorisant `DC01` à interroger un résolveur externe est
   bien active
3. Supprimer les deux règles `TEMPORAIRE` autorisant le DNS vers `FW01` depuis
   POSTES et SERVEURS

Inverser cet ordre coupe la résolution de noms sur l'ensemble de la maquette.

---

## 5. Flux à ouvrir ultérieurement

| Échéance | Flux | Motif |
|---|---|---|
| Mise en service de `SRV01` | à définir selon les services hébergés | |
| Supervision | tous segments vers `SIEM01`, ports 1514 et 1515 | Remontée des agents |
| Authentification réseau | équipements vers serveur RADIUS, ports 1812 et 1813 | 802.1X |
| Temps | postes vers `DC01`, port 123 | Couvert par l'autorisation globale vers `DC01` |

---

## 6. Tests de validation

| Test | Depuis | Attendu | Statut |
|---|---|---|---|
| Passerelle locale | `DC01` | réponse | Validé |
| Sortie Internet | `DC01` | réponse | Validé |
| Résolution de noms | `DC01` | réponse | Validé |
| Ouverture TCP 443 | `DC01` | succès | Validé |
| Résolution externe autorisée | `DC01` | succès | Validé |
| Résolution externe refusée | second serveur du segment | échec | En attente de `SRV01` |
| Isolement du segment invité | `PC01` placé sur INVITES | échec vers les réseaux internes | En attente de `PC01` |
| Accès restreint à `DC01` | `PC01` sur POSTES | succès vers `DC01`, échec ailleurs | En attente de `PC01` |
