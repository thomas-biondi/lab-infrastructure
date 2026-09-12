# Session 3, filtrage inter-VLAN

**Objectif.** Formaliser la matrice de flux, écrire les règles de filtrage sur
les trois segments internes, et valider la fondation réseau de la maquette.

**Résultat.** Objectif atteint. Deux erreurs de conception corrigées avant mise
en production, une panne d'accès Internet diagnostiquée et résolue.

---

## Méthode retenue

La matrice de flux a été écrite avant toute saisie de règle, selon le même
principe que le plan d'adressage : décider d'abord, configurer ensuite. Elle
constitue le document de référence du filtrage, chaque règle devant pouvoir s'y
rattacher.

Trois notions ont structuré la rédaction :

- **Le filtrage à état** autorise l'établissement d'une connexion, pas un sens
  de circulation. Aucune règle de retour n'est écrite.
- **Une règle s'évalue sur l'interface d'entrée** du trafic dans le pare-feu.
  La source appartient toujours au segment de l'interface qui porte la règle.
- **La première correspondance décide.** Les exceptions spécifiques se placent
  au-dessus des autorisations générales.

---

## Arbitrage sur les ports Active Directory

Active Directory emploie une quinzaine de ports fixes auxquels s'ajoute une
plage RPC dynamique de 49152 à 65535. Deux approches étaient possibles :
énumérer les services, ou restreindre la destination.

L'énumération a été écartée. Elle suppose d'ouvrir plus de seize mille ports et
reste fragile : une liste incomplète, omettant l'énumérateur de points de
terminaison RPC ou le catalogue global, fait échouer la jonction au domaine sans
message explicite.

**Choix retenu : autoriser tous les ports, mais vers l'hôte `DC01`
exclusivement.** La granularité porte sur la destination plutôt que sur le port.
C'est un compromis assumé et courant en entreprise, pas un raccourci.

---

## Erreur 1, une autorisation large englobe le pare-feu

Détectée à la relecture du jeu de règles exporté, avant toute mise en service.

Les règles d'isolement du segment invité bloquaient le trafic vers
`POSTES network` et `LAN network`, couvrant ainsi les adresses `10.10.10.1` et
`10.10.20.1` portées par le pare-feu. En revanche, l'adresse `10.10.30.1`,
portée par ce même pare-feu sur le segment invité, appartient à
`INVITES network` et n'était couverte par aucun blocage.

La règle d'autorisation générale vers `any` rendait donc l'interface
d'administration accessible depuis le segment invité.

> **Principe à retenir.** Une destination `any` englobe le pare-feu lui-même, y
> compris l'adresse qu'il porte sur le segment source. Bloquer les réseaux
> voisins ne protège pas l'équipement qui porte la règle.

Correction : ajout d'un blocage explicite vers `Ce pare-feu` sur les segments
POSTES et INVITES.

---

## Erreur 2, un blocage placé sous une autorisation

Le blocage correctif ci-dessus a d'abord été inséré **sous** la règle
`Acces Internet`, qui autorise `any` vers `any` en mode `Rapide`.

Conséquence : l'autorisation correspond en premier, décide, et l'évaluation
s'arrête. Le blocage n'est jamais atteint. La correction n'avait donc aucun
effet, tout en donnant l'apparence d'avoir traité le problème.

> **Enseignement.** Une règle de blocage placée sous une autorisation plus large
> ne s'applique jamais. C'est la cause la plus fréquente du symptôme « la règle
> existe mais ne fait rien ». Vérifier l'ordre effectif après chaque ajout, et
> non l'intention supposée.

---

## Panne, perte d'accès Internet

### Symptôme

Depuis `DC01`, la passerelle locale répond mais aucune sortie Internet ne
fonctionne. La résolution de noms retourne une défaillance serveur.

### Première hypothèse, écartée

Le filtrage venait d'être modifié : la chronologie désignait naturellement les
règles. Cette hypothèse était plausible et fausse.

### Test qui a tranché

Un ping vers une adresse publique lancé **depuis le pare-feu lui-même**, et non
depuis la machine qui se plaignait.

```
ping 1.1.1.1
17 packets transmitted, 0 received, 100.0% packet loss
sendto: Host is down
```

Ce résultat déplace le problème : si le pare-feu n'atteint pas Internet, le
filtrage des segments internes n'est pas en cause.

### Indice décisif

`Host is down` n'est pas un délai d'attente dépassé. Le message traduit un
échec de résolution ARP du prochain saut : le paquet ne quitte jamais la
machine, faute de savoir à quelle adresse matérielle le remettre.

### Cause

L'interface POSTES était repassée en client DHCP et avait obtenu `10.10.10.100`
depuis le pool servi par le pare-feu lui-même. Kea annonçant `10.10.10.1` comme
routeur, OPNsense a créé une passerelle correspondante et l'a promue active.

Le pare-feu routait donc son trafic sortant vers `10.10.10.1`, adresse que plus
aucune interface ne portait, l'interface concernée étant passée en `.100`.

### Correction

1. Interface POSTES remise en adressage statique `10.10.10.1/24`, sans
   passerelle amont
2. Suppression de la passerelle créée automatiquement
3. Vérification que la passerelle du WAN est l'unique passerelle IPv4 active
4. Suppression du bail attribué à l'interface du pare-feu

### Enseignements

> Une interface interne est statique par construction. Elle porte la passerelle
> de son segment, elle ne la reçoit pas.
>
> Une passerelle peut être créée sans intervention, à partir d'une option
> annoncée par un serveur DHCP. L'absence de déclaration explicite ne garantit
> pas l'absence de route.
>
> La chronologie ment. La panne est apparue juste après une modification du
> filtrage, sans aucun rapport avec elle.
>
> Face à une panne de connectivité, remonter le chemin depuis l'équipement le
> plus proche de la sortie plutôt que tester depuis celui qui se plaint. Un
> seul test a écarté trois couches d'hypothèses.
>
> Lire le message exact plutôt que le sens général. Un délai dépassé et un
> `Host is down` ne désignent pas le même problème.

---

## Validation

Tests exécutés depuis `DC01` après désactivation des règles par défaut :

| Test | Résultat |
|---|---|
| Passerelle locale | réponse |
| Sortie Internet | réponse |
| Résolution de noms | réponse |
| Ouverture TCP 443 | succès |

Les tests d'isolement inter-segments restent en attente : ils supposent une
machine hors du segment SERVEURS, ce que la maquette ne comporte pas encore.
Ils seront exécutés à la mise en service de `PC01`.

---

## Point ouvert

Le résolveur renvoie des enregistrements AAAA alors qu'aucune connectivité IPv6
n'existe sur la maquette. Les clients tentent donc systématiquement l'IPv6 avant
de basculer en IPv4, ce qui ajoute une latence sans symptôme visible. Situation
comparable à un angle mort déjà rencontré sur un projet précédent, où IPv6
permettait de contourner un chemin de résolution maîtrisé.

---

## État à la fin de la session

- Matrice de flux formalisée et documentée
- Vingt règles de filtrage en service sur les trois segments internes
- Règles par défaut désactivées, filtrage entièrement explicite
- Fondation réseau validée depuis le segment SERVEURS
- `DC01` installé, à jour, adressé en `10.10.20.10`, non promu

## Prochaine session

Promotion de `DC01` en contrôleur de domaine de la forêt `lab.internal`,
configuration du DNS interne, puis bascule du serveur DNS annoncé par Kea selon
la séquence définie dans `matrice-de-flux.md`.
