# Session 5, poste client et diagnostic de stratégie

**Objectif.** Mettre en service `PC01`, le joindre au domaine, diagnostiquer
l'objet de stratégie volontairement défectueux, et exécuter les tests
d'isolement inter-segments en attente depuis l'étape 3.

**Résultat.** Objectif atteint. Quatre points de blocage rencontrés, tous
instructifs.

---

## Contrainte matérielle de Windows 11

Le module de plateforme sécurisée est obligatoire. Sous VMware Workstation, le
TPM virtuel ne peut être ajouté qu'à une machine préalablement chiffrée, avec
un firmware UEFI et le démarrage sécurisé actif. À défaut, l'installateur
s'interrompt sur un message peu explicite relatif à la configuration matérielle.

---

## Identité locale contre identité cloud

L'assistant de configuration oriente vers un compte Microsoft Entra ID et exige
une connexion réseau. Le chemin vers un compte local passe par le lien
**Options de connexion**, dont le libellé prête à confusion.

Au-delà de la manipulation, cet écran illustre une évolution réelle du métier.
L'annuaire local reste la fondation, mais la majorité des infrastructures sont
aujourd'hui hybrides : une machine peut appartenir au domaine local, à
l'identité cloud, ou aux deux. La distinction entre jonction Active Directory
classique, jonction Entra pure et jonction hybride constitue un sujet courant.

---

## Validation avant jonction

Contrôle systématique de la résolution de l'enregistrement SRV avant toute
tentative de jonction :

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.internal
```

C'est la requête exacte qu'émet Windows pour localiser un contrôleur de
domaine. Une réponse garantit que la jonction aboutira ; un échec situe le
problème en amont, sans passer par un message d'erreur générique de jonction.

Cette seule commande valide simultanément le service DHCP sur le segment, la
bascule du serveur DNS annoncé, et la règle de filtrage autorisant POSTES vers
le contrôleur.

Le message `Serveur : UnKnown` retourné par l'outil traduit l'absence de zone de
recherche inversée, sans conséquence fonctionnelle. Point noté pour traitement
ultérieur.

---

## Blocage 1, emplacement du compte machine

**Constat.** La jonction ayant été réalisée par l'interface graphique, le compte
machine s'est trouvé placé dans le conteneur `CN=Computers`.

**Conséquence.** Un conteneur n'accepte aucun lien de stratégie de groupe : les
objets liés à `OU=Postes` ne s'appliquaient pas, sans message ni avertissement.

**Correction.** Déplacement du compte machine vers l'unité d'organisation cible,
suivi d'un **redémarrage**. Une simple actualisation de stratégie ne rattrape
pas un changement d'unité d'organisation : la machine ne réévalue son
appartenance qu'au démarrage.

> **Enseignement.** C'est l'incident le plus courant du métier : une machine
> jointe, apparemment normale, sur laquelle aucune stratégie ne s'applique.
>
> La parade structurelle est la redirection du conteneur par défaut via
> `redircmp`, qui fait atterrir toute jonction ultérieure au bon endroit, y
> compris celles réalisées à la souris. L'équivalent `redirusr` existe pour les
> comptes utilisateurs.

---

## Blocage 2, comptes créés mais désactivés

**Constat.** Aucun compte utilisateur créé lors de la session précédente
n'autorisait d'ouverture de session.

**Cause.** Lorsque le mot de passe fourni à la création ne satisfait pas la
politique du domaine, l'objet utilisateur est créé mais reste **désactivé**.
Aucune erreur bruyante n'est produite. La politique appliquée impose quatorze
caractères et la complexité.

**Défaut de la vérification initiale.** La commande de contrôle employée en fin
de session 4 affichait le nom et le service, sans l'attribut `Enabled`. Elle
confirmait donc l'existence des objets sans rien dire de leur utilisabilité.

**Correction.** Activation des comptes et réinitialisation des mots de passe
conformément à la politique.

> **Enseignement.** Une vérification qui ne porte pas sur l'attribut déterminant
> ne vérifie rien. Contrôler `Enabled`, `LockedOut` et `PasswordExpired` plutôt
> que la seule présence de l'objet.
>
> Formulation à retenir : la création d'un objet et son caractère opérationnel
> sont deux états distincts.

---

## Blocage 3, verrouillage de compte

Le seuil de dix tentatives infructueuses a été atteint, déclenchant le
verrouillage pour quinze minutes.

Test involontaire mais réel de la politique de verrouillage définie à la session
précédente : seuil respecté, blocage effectif, déblocage par administration
centrale via `Unlock-ADAccount`.

---

## Blocage 4, droits d'ouverture de session sur un contrôleur

**Constat.** Le compte d'administration délégué ne peut ouvrir de session sur
`DC01`, ni directement, ni via `runas`, cette dernière échouant avec l'erreur
1385 : type d'ouverture de session non autorisé sur cet ordinateur.

**Cause.** La stratégie par défaut des contrôleurs de domaine restreint les
droits d'ouverture de session, y compris secondaire, à quelques groupes
privilégiés. Le compte délégué n'en fait partie d'aucun.

**Ce n'est pas un défaut mais le durcissement attendu.** Aucun compte non
privilégié n'a vocation à exécuter quoi que ce soit sur un contrôleur de
domaine.

> **Enseignement.** Deux couches d'autorisation coexistent et sont fréquemment
> confondues.
>
> Les **permissions d'annuaire** déterminent ce qu'un compte peut faire sur des
> objets. Les **droits d'ouverture de session**, portés par les stratégies,
> déterminent où un compte peut se connecter.
>
> Un compte peut disposer de permissions étendues sur l'annuaire sans aucun
> droit de session nulle part. C'est même la configuration recherchée pour un
> compte de service.
>
> Corollaire pratique : on administre l'annuaire à distance depuis un poste
> d'administration, non depuis le contrôleur.

---

## Diagnostic de l'objet défectueux

### Premier rapport, portée utilisateur

Exécuté sans élévation, donc limité à la moitié utilisateur.

| Élément | Valeur |
|---|---|
| Unité d'organisation de l'utilisateur | `lab.internal/LAB/Utilisateurs` |
| Objets appliqués | aucun |
| Paramètres | aucun paramètre défini |

### Second rapport, portée ordinateur

Exécuté avec élévation. L'outil rapportant par défaut sur l'utilisateur du
processus, une première tentative a retourné une absence de données : le compte
élevé ne s'était jamais connecté sur la machine. Le paramètre de portée ou
d'utilisateur permet de dissocier **qui exécute** de **sur qui l'on rapporte**.

| Élément | Valeur |
|---|---|
| Unité d'organisation de la machine | `lab.internal/LAB/Ordinateurs/Postes` |
| Objets appliqués | `Default Domain Policy`, `GPO_Postes_Durcissement` |
| Objets refusés | stratégie locale uniquement |
| Erreurs | aucune |

### Conclusion

`GPO_Test_Erreur` **n'apparaît dans aucune des deux sections**. Ni appliquée,
ni refusée, ni mentionnée.

> **Enseignement central de la session.** Il existe deux façons pour une
> stratégie de ne rien produire.
>
> Elle est **évaluée puis écartée** : elle figure alors parmi les objets
> refusés, assortie d'un motif, typiquement un filtrage de sécurité ou un lien
> désactivé.
>
> Elle n'est **jamais évaluée**, l'objet ne se trouvant pas dans le périmètre du
> point de liaison. Elle est alors purement absente du rapport.
>
> Le second cas est le plus déroutant : on cherche la stratégie dans le rapport,
> on ne la trouve pas, et l'on conclut à un problème de réplication ou de
> réseau. La seule méthode consiste à comparer l'emplacement de l'objet avec le
> point de liaison de la stratégie.

Deux corrections possibles selon l'intention : lier la stratégie à l'unité
d'organisation contenant les comptes utilisateurs, ou activer le mode de
bouclage. Ce dernier force le traitement de la moitié utilisateur en fonction de
l'emplacement de la machine, usage courant sur les postes partagés, salles de
formation et bornes.

### Lecture complémentaire du rapport

Les durées de traitement par extension figurent dans l'état des composants.
Sur un poste dont l'ouverture de session est anormalement longue, c'est là que
se situe le diagnostic : une extension consommant plusieurs dizaines de secondes
au lieu de quelques centaines de millisecondes.

La mention d'une liaison rapide détectée renvoie à la mesure de bande passante
vers le contrôleur : sur une liaison jugée lente, certaines extensions coûteuses
sont écartées. Cause classique de scripts de démarrage ne s'exécutant jamais sur
les sites distants.

---

## Erreur de contexte d'exécution

Les tests d'isolement ont d'abord été exécutés depuis une console restée ouverte
sur `DC01`, et non sur `PC01`. Les résultats, tous conformes aux règles du
segment SERVEURS, ne validaient rien de l'isolement recherché.

Détecté par le champ d'adresse source affiché par l'outil de test de connexion.

> **Enseignement.** Vérifier `hostname` avant toute commande sensible lorsque
> plusieurs consoles sont ouvertes sur des machines différentes. Une sortie
> cohérente peut répondre à une question qui n'était pas celle posée.

---

## Tests d'isolement

Exécutés depuis `PC01`, adresse `10.10.10.101`.

| Test | Attendu | Résultat |
|---|---|---|
| Vers `DC01`, ICMP | réponse | réponse, TTL 127 |
| Vers `FW01`, ICMP | échec | délai dépassé |
| Vers `FW01`, port 443 | échec | échec |
| Vers un résolveur externe | échec | délai dépassé |

**Le TTL décrémenté de 128 à 127** atteste du franchissement d'un routeur. La
même commande exécutée depuis le segment SERVEURS retournait 128, sans routage.
Le TTL de retour renseigne sur la distance et sur la nature de ce qui répond.

**Les échecs se manifestent par un délai dépassé** et non par un refus immédiat,
confirmant que les règles rejettent silencieusement. Comportement recherché sur
un pare-feu : un refus explicite renseignerait sur l'existence de la cible.

Sont validés simultanément : le routage inter-VLAN, le filtrage à état, le
moindre privilège vers le contrôleur de domaine, la protection du pare-feu
vis-à-vis de ses propres segments, et le caractère contraint du chemin de
résolution DNS.

---

## État à la fin de la session

- `PC01` membre du domaine, compte machine dans l'unité d'organisation cible
- Stratégies de la moitié ordinateur appliquées et vérifiées par rapport
- Objet défectueux diagnostiqué, mécanisme compris et documenté
- Quatre tests d'isolement exécutés et conformes
- Étape 4 terminée

## Prochaine étape

Services Linux et automatisation : mise en service de `SRV01`, durcissement de
l'accès distant, sauvegardes automatisées, et travail de scripting appliqué à
des besoins réels de la maquette plutôt qu'à des exercices isolés.
