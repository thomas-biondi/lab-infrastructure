# PC01, poste client

> Document de référence décrivant l'état courant de `PC01`. Le raisonnement
> figure dans `journal/session-05-poste-client.md`.

---

## 1. Identité

| Élément | Valeur |
|---|---|
| Rôle | Poste de travail membre du domaine |
| Système | Windows 11 Enterprise, évaluation |
| Nom d'hôte | `PC01` |
| Adresse | `10.10.10.101`, bail DHCP |
| Segment | VLAN 10 POSTES |
| Unité d'organisation | `OU=Postes,OU=Ordinateurs,OU=LAB,DC=lab,DC=internal` |
| Ressources | 4 Go de mémoire, 2 processeurs virtuels, disque de 60 Go |

L'édition Enterprise est requise : l'édition Famille ne peut pas rejoindre un
domaine Active Directory, restriction de licence et non limitation technique.

Windows 11 impose un module de plateforme sécurisée. Sous VMware Workstation,
l'ajout d'un TPM virtuel suppose d'avoir préalablement chiffré la machine
virtuelle, le firmware devant par ailleurs être en UEFI avec démarrage
sécurisé.

---

## 2. Installation

L'assistant de configuration oriente vers une identité cloud et exige une
connexion réseau. Le chemin vers un compte local passe par le lien
**Options de connexion** de l'écran de connexion à l'organisation.

Un compte local `admin-local` a été créé avant la jonction au domaine. Il
constitue le moyen d'accès de secours en cas d'indisponibilité de l'annuaire ou
de blocage d'un compte de domaine.

---

## 3. Validation préalable à la jonction

Contrôles réalisés avant toute tentative de jonction :

| Contrôle | Résultat attendu |
|---|---|
| Adresse obtenue | dans la plage `10.10.10.100` à `.199` |
| Passerelle annoncée | `10.10.10.1` |
| Serveur DNS annoncé | `10.10.20.10` |
| Suffixe DNS | `lab.internal` |
| Résolution SRV `_ldap._tcp.dc._msdcs.lab.internal` | `dc01.lab.internal` port 389 |

**La résolution de l'enregistrement SRV est le contrôle décisif.** C'est
exactement la requête qu'émet Windows pour localiser un contrôleur de domaine.
Si elle aboutit, la jonction aboutira ; si elle échoue, le problème est en amont
et une tentative de jonction ne produirait qu'un message d'erreur générique.

Cette validation confirme simultanément la bascule DNS, le service DHCP sur le
segment POSTES et la règle de filtrage autorisant POSTES vers `DC01`.

---

## 4. Jonction au domaine

Emplacement final du compte machine :
`OU=Postes,OU=Ordinateurs,OU=LAB,DC=lab,DC=internal`.

**Une jonction réalisée sans précision d'emplacement place le compte machine
dans le conteneur `CN=Computers`.** Un conteneur n'accepte aucun lien de
stratégie de groupe : les GPO liées à `OU=Postes` ne s'appliquent alors pas, sans
message d'erreur.

Deux moyens d'obtenir le bon emplacement :

- Préciser l'unité d'organisation cible lors de la jonction en ligne de commande
- Rediriger le conteneur par défaut sur le contrôleur, une fois pour toutes :

```
redircmp "OU=Postes,OU=Ordinateurs,OU=LAB,DC=lab,DC=internal"
```

Toute machine jointe ensuite, y compris par l'interface graphique, atterrit dans
l'unité d'organisation voulue. L'équivalent pour les comptes utilisateurs est
`redirusr`.

Un déplacement de compte machine effectué après coup n'est pris en compte
qu'au **redémarrage** de la machine. Une actualisation de stratégie ne suffit
pas à rattraper un changement d'unité d'organisation.

---

## 5. Stratégies appliquées

Constaté par rapport `gpresult` :

| Portée | Unité d'organisation évaluée | Objets appliqués |
|---|---|---|
| Ordinateur | `lab.internal/LAB/Ordinateurs/Postes` | `Default Domain Policy`, `GPO_Postes_Durcissement` |
| Utilisateur | `lab.internal/LAB/Utilisateurs` | aucun |

`GPO_Test_Erreur` n'apparaît dans aucune des deux sections, ni comme appliquée
ni comme refusée. Analyse détaillée dans le journal de session.

La bannière d'avertissement s'affiche avant l'écran de connexion, confirmant
visuellement le traitement de la moitié ordinateur.

---

## 6. Tests d'isolement

Exécutés depuis `PC01`, en attente depuis l'étape 3 faute de machine hors du
segment SERVEURS.

| Test | Attendu | Résultat | Règle validée |
|---|---|---|---|
| Vers `DC01`, ICMP | réponse | réponse, TTL 127 | Accès complet vers `DC01` |
| Vers `FW01`, ICMP | échec | délai dépassé | Blocage vers le pare-feu |
| Vers `FW01`, port 443 | échec | échec | Blocage vers le pare-feu |
| Vers un résolveur externe | échec | délai dépassé | Blocage du DNS sortant |

**Le TTL décrémenté de 128 à 127** atteste du franchissement d'un routeur : le
trafic traverse effectivement le pare-feu entre les deux segments.

**Les échecs se manifestent par un délai dépassé et non par un refus
immédiat.** Les règles rejettent silencieusement plutôt que de notifier, ce qui
évite de renseigner un attaquant sur l'existence de la cible. Un refus immédiat
et un délai d'attente ne traduisent pas le même comportement de filtrage : c'est
un indice de diagnostic exploitable.

---

## 7. Points ouverts

L'applet `Add-Computer` est dépréciée sous Windows Server 2025 et son
comportement varie selon la version de PowerShell. La jonction a été réalisée
par l'interface graphique, suivie d'un déplacement du compte machine.

Les outils d'administration à distance n'ont pas été installés sur le poste.
Leur mise en place permettrait d'administrer l'annuaire depuis `PC01` plutôt que
d'ouvrir une session sur le contrôleur, conformément au modèle recommandé.

Le groupe `DL_Admin_Postes` n'est pas encore rattaché au groupe Administrateurs
local des postes. Ce rattachement, réalisable par préférence de stratégie de
groupe, matérialiserait la chaîne AGDLP de bout en bout.
