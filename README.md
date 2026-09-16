# lab-infrastructure

Maquette d'infrastructure d'entreprise virtualisée, construite pour couvrir le
socle technique commun à l'administration systèmes, au réseau, à la sécurité
d'infrastructure et à la sécurité opérationnelle.

L'objectif n'est pas d'empiler des services mais de reproduire les contraintes
réelles d'un système d'information : segmentation, annuaire centralisé,
durcissement, supervision, et documentation des décisions de conception.

## Architecture

```
Internet
   |
 FW01 (OPNsense) ........ routage inter-VLAN, filtrage, DHCP
   |                      authentification des administrateurs par RADIUS
   |
   +-- VLAN 10 POSTES ... PC01, poste joint au domaine
   |                      administration locale et secrets gérés par l'annuaire
   |
   +-- VLAN 20 SERVEURS . DC01 (AD DS, DNS, PKI), SRV01 (services, RADIUS),
   |                      SIEM01 (collecte et détection)
   |
   +-- VLAN 30 INVITES .. segment isolé, accès Internet uniquement
```

## État d'avancement

| Étape | Objet | Statut |
|---|---|---|
| 1 | Plan d'adressage et conventions | Terminé |
| 2 | Pare-feu, routage inter-VLAN, DHCP | Terminé |
| 3 | Matrice de flux et filtrage | Terminé |
| 4 | Active Directory, DNS, GPO, poste client | Terminé |
| 5 | Services Linux et automatisation | Terminé |
| 6 | Durcissement et authentification centralisée | Terminé |
| 7 | Supervision et détection | En cours |

## Documents de référence

| Document | Objet |
|---|---|
| [`docs/plan-adressage.md`](docs/plan-adressage.md) | Segments, adressage, nomenclature, conventions |
| [`docs/matrice-de-flux.md`](docs/matrice-de-flux.md) | Flux autorisés entre segments et justifications |
| [`docs/configuration-fw01.md`](docs/configuration-fw01.md) | Pare-feu : interfaces, DHCP, règles de filtrage |
| [`docs/configuration-dc01.md`](docs/configuration-dc01.md) | Contrôleur de domaine : forêt, annuaire, groupes, GPO |
| [`docs/configuration-pc01.md`](docs/configuration-pc01.md) | Poste client : jonction, stratégies, tests d'isolement |
| [`docs/configuration-srv01.md`](docs/configuration-srv01.md) | Serveur Linux : durcissement, pare-feu local, sauvegardes |
| [`docs/authentification-centralisee.md`](docs/authentification-centralisee.md) | RADIUS, PKI interne, autorisation par groupe |
| [`docs/durcissement-postes.md`](docs/durcissement-postes.md) | Administration locale et gestion des secrets machines |
| [`docs/supervision-detection.md`](docs/supervision-detection.md) | Plan de détection, collecte, règles et méthode d'analyse |
| [`docs/automatisation.md`](docs/automatisation.md) | Scripts en service, choix de conception et limites |

## Journal de bord

| Session | Objet |
|---|---|
| [01](journal/session-01-plan-adressage.md) | Plan d'adressage et conventions |
| [02](journal/session-02-pare-feu.md) | Mise en service du pare-feu |
| [03](journal/session-03-filtrage.md) | Filtrage inter-VLAN |
| [04](journal/session-04-active-directory.md) | Mise en service de l'annuaire |
| [05](journal/session-05-poste-client.md) | Poste client et diagnostic de stratégie |
| [06](journal/session-06-serveur-linux.md) | Serveur Linux et automatisation |
| [07](journal/session-07-radius.md) | Authentification centralisée |
| [08](journal/session-08-durcissement-postes.md) | Durcissement des postes |
| [09](journal/session-09-supervision.md) | Supervision et détection |

Les documents de `docs/` décrivent l'état courant de la maquette. Le journal
conserve la trace des choix, des raisonnements et des incidents, ainsi que la
méthode de diagnostic employée pour les résoudre.

## Conventions

Aucun secret n'est versionné : mots de passe, clés privées, certificats et
sauvegardes de configuration restent hors du dépôt. Les valeurs sensibles sont
remplacées par des marqueurs et la méthode est décrite à la place.
