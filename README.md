# lab-infrastructure

Maquette d'infrastructure d'entreprise virtualisée, construite pour couvrir le
socle technique commun à l'administration systèmes, au réseau, à la sécurité
d'infrastructure et à la sécurité opérationnelle.

L'objectif n'est pas d'empiler des services mais de reproduire les contraintes
réelles d'un système d'information : segmentation, annuaire centralisé,
durcissement, supervision, et documentation des décisions de conception.

## Architecture cible

```
Internet
   |
 FW01 (OPNsense) ........ routage inter-VLAN, filtrage, DHCP
   |
   +-- VLAN 10 POSTES ... PC01, poste client joint au domaine
   +-- VLAN 20 SERVEURS . DC01 (AD DS, DNS), SRV01 (services), SIEM01 (Wazuh)
   +-- VLAN 30 INVITES .. segment isolé, accès Internet uniquement
```

## État d'avancement

| Étape | Objet | Statut |
|---|---|---|
| 1 | Plan d'adressage et conventions | Terminé |
| 2 | Pare-feu, routage inter-VLAN, DHCP | Terminé |
| 3 | Matrice de flux et filtrage | Terminé |
| 4 | Active Directory, DNS, GPO | Client et diagnostic restants |
| 5 | Services Linux et automatisation | À venir |
| 6 | Durcissement et authentification RADIUS | À venir |
| 7 | Supervision et détection | À venir |

## Documents de référence

| Document | Objet |
|---|---|
| [`docs/plan-adressage.md`](docs/plan-adressage.md) | Segments, adressage, nomenclature, conventions |
| [`docs/matrice-de-flux.md`](docs/matrice-de-flux.md) | Flux autorisés entre segments et justifications |
| [`docs/configuration-fw01.md`](docs/configuration-fw01.md) | Pare-feu : interfaces, DHCP, règles de filtrage |
| [`docs/configuration-dc01.md`](docs/configuration-dc01.md) | Contrôleur de domaine : forêt, annuaire, groupes, GPO |

## Journal de bord

| Session | Objet |
|---|---|
| [01](journal/session-01-plan-adressage.md) | Plan d'adressage et conventions |
| [02](journal/session-02-pare-feu.md) | Mise en service du pare-feu |
| [03](journal/session-03-filtrage.md) | Filtrage inter-VLAN |
| [04](journal/session-04-active-directory.md) | Mise en service de l'annuaire |

Les documents de `docs/` décrivent l'état courant de la maquette. Le journal
conserve la trace des choix, des raisonnements et des erreurs, y compris celles
qui ont été corrigées ensuite.

## Conventions

Aucun secret n'est versionné : mots de passe, clés privées, certificats et
sauvegardes de configuration restent hors du dépôt. Les valeurs sensibles sont
remplacées par des marqueurs et la méthode est décrite à la place.
