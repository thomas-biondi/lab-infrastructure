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
| 3 | Matrice de flux et filtrage | À venir |
| 4 | Active Directory, DNS, GPO | À venir |
| 5 | Services Linux et automatisation | À venir |
| 6 | Durcissement et authentification RADIUS | À venir |
| 7 | Supervision et détection | À venir |

## Organisation du dépôt

| Dossier | Contenu |
|---|---|
| `docs/` | Documents de référence : plan d'adressage, configuration des équipements |
| `journal/` | Journal de bord par session : décisions, raisonnement, incidents |

Les documents de `docs/` décrivent l'état courant de la maquette. Le journal
conserve la trace des choix et des erreurs, y compris ce qui a été corrigé
ensuite.

## Conventions

Aucun secret n'est versionné : mots de passe, clés privées, certificats et
sauvegardes de configuration restent hors du dépôt. Les valeurs sensibles sont
remplacées par des marqueurs et la méthode est décrite à la place.
