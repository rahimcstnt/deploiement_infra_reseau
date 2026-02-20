# Déploiement d'infrastructure réseau
Ce projet s’inscrit dans le cadre d’une SAÉ du BUT Informatique au semestre 5. Il consiste à concevoir et mettre en place une infrastructure réseau complète pour une organisation, avec un réseau public, un réseau privé segmenté en VLAN (production, administratif, informatique) et un pare-feu centralisé assurant le routage et la sécurité.

# Compte Rendu Final équipe Blanc :

- **Boughezal Abderrahim**
- **Khudoev Revaz**
- **Kochiev Mickhail**
 
## Introduction -  Réseau :

Le FAI nous a attribué le réseau 192.168.1.0/24 pour cette infrastructure. Nous l'avons découpé en sous-réseaux /27 avec masque 255.255.255.224, créant 8 sous-réseaux de 32 adresses chacun (30 hôtes utilisables, 1 adresse réseau, 1 broadcast). 

Ce choix offre une grande flexibilité pour des extensions futures et réduit les marges d'erreur, même si nous n'utilisons que 5 sous-réseaux : Informatique, Administratif, Production, DMZ, et le lien Firewall-Routeur.

## Sous-Réseaux Utilisés

| Sous-réseau       | Adresse réseau     | Intervalle d'adresses (hôtes) | Broadcast        | Masque             |
|-------------------|--------------------|-------------------------------|------------------|--------------------|
| Informatique     | 192.168.1.0/27    | 192.168.1.1 - 192.168.1.30   | 192.168.1.31    | 255.255.255.224   |
| Administratif    | 192.168.1.32/27   | 192.168.1.33 - 192.168.1.62  | 192.168.1.63    | 255.255.255.224   |
| Production       | 192.168.1.64/27   | 192.168.1.65 - 192.168.1.94  | 192.168.1.95    | 255.255.255.224   |
| DMZ              | 192.168.1.96/27   | 192.168.1.97 - 192.168.1.126 | 192.168.1.127   | 255.255.255.224   |
| Lien Firewall    | 192.168.1.224/27  | 192.168.1.225 - 192.168.1.254| 192.168.1.255   | 255.255.255.224   |


## Configuration VLAN et Connectivité

Chaque sous-réseau principal est isolé dans un VLAN dédié :  
- VLAN 20 : réseau Informatique  
- VLAN 30 : réseau Administratif  
- VLAN 40 : réseau Production  
- VLAN 10 : DMZ  

Un port trunk du switch 2 relie les VLAN 20, 30, 40 à l'interface **IN** du pare-feu Stormshield. Un second port trunk du switch 3 connecte le VLAN 10 à l'interface **DMZ** du pare-feu.

L'interface **OUT** du pare-feu possède l'adresse IP **192.168.1.253** et est directement branchée sur l'interface **WLAN4** du routeur (IP **192.168.1.254**) dans le sous-réseau de lien Firewall.

Les machines virtuelles sont configurées en mode **bridge** sur l'interface physique **enp3s0** :  
- douglas02 pour Informatique  
- douglas03 pour Administratif  
- douglas04 pour Production  
- douglas01 pour DMZ  

Chaque machine a deux routes configurées :  
1. Route par défaut vers sa passerelle (dernière adresse hôtes : .30, .62, .94, .126)  
2. Route vers **192.168.0.0/16**  
Cela évite d'écrire une route distincte pour chaque réseau ; le pare-feu Stormshield assure toute la sécurité inter-réseaux.

![Diagramme infrastructure réseau](/images/infra_reseau_finale.jpg "Diagramme infrastructure réseau")

![Infrastructure réseau physique](/images/infra_physique.jpg "Infrastructure réseau physique")
"
## Politiques de Sécurité Réseau

Pour respecter les consignes du projet, des règles strictes ont été établies sur le pare-feu :  

**Accès autorisés vers notre infrastructure :**  
- Réseaux 192.168.2.0/24, 192.168.3.0/24, 192.168.4.0/24  
- Réseau d'interconnexion 192.168.10.0/24  
**→ Accès DMZ uniquement, interdit vers Informatique/Administratif/Production**

**Accès sortants depuis nos réseaux privés :**  
- Accès autorisé vers les DMZ des autres organisations  
- Accès autorisé vers le réseau d'interconnexion 192.168.10.0/24  

**Contrôle spécifique Production :**  
Les machines du réseau Production n'accèdent aux réseaux externes **qu'à travers un proxy web** situé dans le réseau Informatique, via un navigateur **Firefox** préconfiguré avec les paramètres proxy.

![Règles pare-feu](/images/regles_parefeu.png "Règles pare-feu")
## Routage Inter-Organisations

**Sur le routeur :**  
- Routes statiques vers 192.168.2.0/24, 192.168.3.0/24, 192.168.4.0/24  
- Réseau 192.168.10.0/24 directement connecté (adresses 192.168.10.1 à 192.168.10.4, passerelle 192.168.10.4)

**Sur le pare-feu Stormshield :**  
- Les mêmes routes incluant 192.168.10.0/24 avec passerelle 192.168.1.254 (interface OUT)  

**Relais DHCP :** Configuré sur le pare-feu pour transmettre les trames DHCP Discover vers tous les VLANs.


## Méthodologie d'Installation des Services

Pour chaque sous-réseau, nous avons adopté une **structure de dossiers organisée** basée sur la machine physique hôte :

**Organisation des dossiers :**
- **douglas01** → DMZ
- **douglas02** → Informatique  
- **douglas03** → Administratif
- **douglas04** → Production

**Structure dans chaque dossier de sous-réseau :**
~~~~
douglasXX/  
├── Vagrantfile # Création et provisionnement des VMs  
├── dns/ # Fichiers config + script déploiement  
├── web/ # Fichiers config + script déploiement  
├── mail/ # Fichiers config + script déploiement  
├── dhcp/ # Fichiers config + script déploiement  
└── ...
~~~~
**Fonctionnement :** 
1. **Vagrantfile** à la racine : définit et provisionne toutes les machines virtuelles du sous-réseau 
2. **Répertoires services/** : contiennent pour chaque service : - Les **fichiers de configuration** spécifiques - **Scripts d'installation/déploiement** qui copient automatiquement les fichiers vers les bons répertoires après démarrage des VMs Cette approche **modulaire** permet une maintenance facile, des déploiements reproductibles et une organisation claire des configurations par service et par sous-réseau. 

## Services - DMZ

### DNS et Dynamic DNS

J'ai configuré un **serveur DNS autoritaire** pour le domaine **blanc.iut**, qui fait **forward** vers **192.168.4.20** (DNS du FAI faisant autorité sur le TLD **iut**).

**Deux zones DNS configurées :**
1. **Zone blanc.iut** : contient les enregistrements des serveurs publics (web, mail, etc.)
2. **Zone prive.blanc.iut** : contient les enregistrements des machines des réseaux privés

**Dynamic DNS avec TSIG :** J'ai configuré une **clé TSIG** partagée entre le serveur DNS et le serveur DHCP. Lorsqu'une machine demande une adresse DHCP, le serveur DHCP **écrit automatiquement** l'adresse attribuée dans la zone **prive.blanc.iut**, permettant d'interroger directement le **FQDN** lié à cette machine.

**Sécurité avec ACLs :**
- **Zone prive.blanc.iut** : interrogation **limitée uniquement au réseau privé**
- **Zone blanc.iut** : interrogation **autorisée pour les réseaux concernés**

### Serveur Web

J'ai installé un **serveur Apache** et configuré une **page d'accueil index.php** présentant l'organisation **blanc.iut**.

### Serveur Mail

J'ai installé **Postfix** en tant que **serveur SMTP** et **Dovecot** en tant que **serveur IMAP** avec authentification basée sur les **utilisateurs système (PAM)**.

**Utilisateurs système créés sur cette machine :**
- `userinfo`
- `useradmin1` 
- `useradmin2`
- `userprod`

**Clients mail installés :**
- **Roundcube** (interface webmail)
- **Firefox** (navigateur web)

**Redirection de port** : Port **8080** de la machine hôte vers le service pour effectuer les tests.

## Services - Réseau Administratif

J'ai configuré des **machines utilisateurs** équipées de :  
- **Postfix** (SMTP)
- **Dovecot** (IMAP)
- **Roundcube** (webmail)
- **Firefox** (navigateur)

**Redirections de ports** : **8080** et **8081** vers la machine physique pour effectuer les tests.

## Services - Réseau Production

J'ai configuré des **machines utilisateurs** avec :  
- **Postfix** (SMTP)
- **Dovecot** (IMAP)
- **Roundcube** (webmail)
- **Firefox** (navigateur)

**Configuration proxy obligatoire :**  
J'ai créé un fichier **`/usr/lib/firefox-esr/distribution/policies.json`** qui **force Firefox** à utiliser le **forward proxy** du sous-réseau Informatique situé à l'adresse **`192.168.1.5`**.

**Sécurité renforcée :**
- **Route vers l'extérieur supprimée**
- **Dépendance DNS FAI uniquement**
- **Accès réseau local 192.168.1.0 maintenu**
- **Redirection graphique Firefox** (pas de port forwarding)

## Services - Réseau Informatique

### Serveur DHCP

J'ai configuré un **serveur DHCP** qui attribue des **adresses dynamiques** dans une plage spécifique pour **éviter les adresses IP statiques**. 

**Configuration des baux :** **10 secondes** (pour les tests rapides).  

Le serveur DHCP **enregistre automatiquement** toutes les nouvelles adresses dans la zone **prive.blanc.iut** grâce à la **clé TSIG partagée** avec le serveur DNS.

### NFS avec Ansible

**Authentification SSH :** J'ai généré une **paire de clés SSH publique/privée**. La clé publique de la machine de gestion a été **copiée sur toutes les machines utilisateurs**.

**Inventaire Ansible :** Contient les **FQDN des machines** qui sont résolus automatiquement via la zone privée du DNS.

**Playbooks Ansible créés :**
1. **nfs-server** : Installe et configure le **serveur NFS** sur la machine locale
2. **nfs-client** : Ansible **interroge le DNS** pour récupérer les adresses IP des machines cibles, puis crée les **répertoires partagés** et configure les **clients NFS**

### LDAP

#### Mise en place du serveur LDAP

**Objectif**

**Mettre en place :**

- Un annuaire centralisé

- Une gestion cohérente des utilisateurs

**Une gestion des groupes correspondant aux services :**

- Informatique

- Administratif

- Production

**Le domaine LDAP configuré est :**

dc=blanc,dc=iut

#### Structure de l’annuaire

**L’annuaire est organisé de manière hiérarchique :**

```
dc=blanc,dc=iut
├── ou=people
├── ou=groups
└── ou=services
```

**Units organisationnelles :**

ou=people → contient les comptes utilisateurs

ou=groups → contient les groupes POSIX

ou=services → réservé aux services futurs

##### Groupes LDAP

**Trois groupes principaux ont été créés :**

**Groupe	GID	Service:**
- info	5000	Informatique
- admin	5001	Administratif
- prod	5002	Production

Les groupes sont de type posixGroup, permettant une compatibilité avec les systèmes Linux (uid/gid).

##### Utilisateurs LDAP

**Les utilisateurs suivants ont été créés :**

Utilisateur	UID	Groupe principal
Mickhail 10000 admin
Revaz 10001 admin
Abderrahim 10002 admin
userinfo	11003	info
useradmin1	11001	admin
useradmin2	11002	admin
userprod	11000	prod

**Chaque utilisateur possède :**

objectClass: inetOrgPerson

objectClass: posixAccount

objectClass: shadowAccount

UID unique

GID correspondant au service

homeDirectory

loginShell

adresse mail (ex: userinfo@blanc.iut
)

**Cette configuration permet :**

- Une compatibilité Linux complète
- Une future intégration NFS
- Une centralisation des comptes

##### Sécurité

Mot de passe administrateur LDAP configuré via SSHA

Authentification locale via socket ldapi:/// pour la configuration interne

Les accès LDAP sont limités au réseau Informatique.

Automatisation

**La configuration LDAP est automatisée grâce :**

**À des fichiers LDIF :**

- base.ldif
- groups.ldif
- users.ldif

**À un script Bash permettant :**

- Configuration de slapd
- Définition du suffixe
- Définition du RootDN
- Import automatique des fichiers LDIF

Cela permet de reconstruire l’annuaire rapidement en cas de suppression de la machine.

### Apache - Forward Proxy

J'ai configuré un **fichier forward-proxy.conf** placé dans **`/etc/apache2/sites-available/`**.

**Configuration détaillée :**
- **VirtualHost sur port 80** pour "**apache.local**"
- Agit comme **proxy forward** (proxy direct)
- Les **clients du sous-réseau Production (192.168.1.64/27)**, c'est-à-dire les adresses **.65 à .94**, peuvent envoyer leurs requêtes **HTTP/HTTPS**
- Le proxy **relaye ces requêtes vers Internet**

### Machine Utilisateur `userinfo`

Machine équipée de :  
- **Postfix** (SMTP)
- **Dovecot** (IMAP)  
- **Roundcube** (webmail)
- **Firefox** (navigateur)

**Redirection de port** : **8080** vers la machine physique pour les tests.


## Tableau Récapitulatif des Services

| Service/Composant          | **FAIT** | **NON FAIT** |
|----------------------------|----------|--------------|
| **DMZ**                    |          |              |
| DNS   | ✓        |              |
| Serveur Web Apache         | ✓        |              |
| Serveur Mail  | ✓      |              |
| **Administratif**          |          |              |
| Machine utilisateur 1 (Client mail)  | ✓        |              |
| Machine utilisateur 2 (Client mail)  | ✓        |              |
| **Production**             |          |              |
| Machine utilisateur (Client mail)  | ✓        |              |
| **Informatique**           |          |              |
| DHCP               | ✓        |              |
| NFS | ✓        |              |
| Forward Proxy       | ✓        |              |
| Machine utilisateur (Client mail) | ✓        |              |
| LDAP                       | ✓         |       |


## Difficultés Rencontrées

La **principale difficulté** rencontrée concerne **l'installation de l'infrastructure réseau**, qui s'est avérée techniquement complexe à plusieurs niveaux :

### Configuration VLAN et Trunking
- **Problème** : Les ports trunk entre switch et pare-feu Stormshield refusaient initialement de propager les VLANs 10, 20, 30, 40
- **Solution** : Rajouter les passerelles par défaut sur des interfaces virtuelles du pare-feu Stormshield

### Routage Inter-VLAN via Pare-feu
- **Problème** : Les VLANs internes (20, 30, 40) n'avaient pas de connectivité inter-sous-réseaux
- **Solution** : Configuration des règles de routage sur Stormshield avec **NAT approprié** + validation des passerelles par défaut


### Problèmes DNS sur la Zone Privée

- **Problème : Mise à jour dynamique DHCP → DNS**  
  Les mises à jour dynamiques n’apparaissaient pas dans la zone privée malgré la configuration TSIG.  
  **Cause** : Incohérence entre le nom de la clé TSIG côté DHCP et côté DNS, et droits insuffisants sur la zone.  
  **Solution** : Harmoniser le nom et  la clé TSIG, lier explicitement cette clé à la zone `prive.blanc.iut` (allow-update), puis vérifier dans les logs DNS que les mises à jour sont acceptées.

## Conclusion

L'infrastructure **blanc.iut** est **entièrement opérationnelle** :

**Objectifs atteints :**
- Segmentation VLAN 10/20/30/40
- Sécurité : externes → DMZ, Production via proxy
- Services : DNS dynamique, mail, web, DHCP, NFS, proxy
- Routage inter-organisations fonctionnel


