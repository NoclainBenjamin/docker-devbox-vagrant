# Docker Devbox

Docker devbox est un projet Vagrant intégrant tout le nécessaire pour créer des environments de développement Docker 
sous Windows & Mac.

## Pourquoi ?

Docker for windows et Docker toolbox utilisent des partages VirtualBox/Hyper-V entre l'hôte Windows et la VM Linux ou 
s'execute le daemon docker pour monter le volumes locaux dans le container. Celà entraine une série de problèmes:

- Performance médiocres.
- Droits altérés et difficiles à maitriser.
- Liens symboliques absents provoquant des problèmes pour certains programmes.

## Solution

- VM Docker (Ubuntu Xenial).
- Vagrant pour provisionner Docker, Docker Compose et [nginx-proxy](https://github.com/jwilder/nginx-proxy).
- [nfs4j-daemon](https://github.com/inetum-orleans/nfs4j-daemon) pour partager les fichiers entre l'hôte sous windows et la VM Docker.
- [Smartcd](https://github.com/cxreg/smartcd) (Activation/Désactivation automatique d'alias lors de l'entrée/sortie dans un dossier)

Cette solution est construite de zéro ce qui permet de garder une bonne flexibilité sur l'environnement technique de la VM.

*Note: nginx-proxy permet d'accéder un à container web via `http://mon-appli.test` plutôt que `http://192.168.1.100:<port>`*

## Pré-requis
- [VirtualBox](https://www.virtualbox.org/) (**/!\\** La virtualisation doit être activé dans le BIOS)
- [Vagrant](https://www.vagrantup.com/)
- [Vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) (`vagrant plugin install vagrant-vbguest`)
- [Vagrant-nfs4j](https://github.com/inetum-orleans/vagrant-nfs4j) (`vagrant plugin install vagrant-nfs4j`)
- [vagrant-disksize](https://github.com/sprotheroe/vagrant-disksize) (`vagrant plugin install vagrant-disksize`)
- [vagrant-certificates](https://github.com/inetum-orleans/vagrant-certificates) (Optionnel, `vagrant plugin install vagrant-certificates`)
- [vagrant-persistent-storage](https://github.com/kusnier/vagrant-persistent-storage) (Optionnel, `vagrant plugin install vagrant-persistent-storage`)
- [Acrylic DNS Proxy](https://sourceforge.net/projects/acrylic) (Optionnel, [Aide d'installation sur StackOverflow](https://stackoverflow.com/questions/138162/wildcards-in-a-windows-hosts-file#answer-9695861), Proxy DNS local pour rediriger `*.test` vers 
l'environnement docker, identique au fichier `/etc/hosts` mais supporte les wildcard `*`)

## Installation

- Cloner le repository

```bash
git clone https://github.com/inetum-orleans/docker-devbox-vagrant
cd docker-devbox-vagrant
```

- Lancer vagrant:

```bash
vagrant up
```

Au premier lancement, la box `ubuntu/xenial` est téléchargée depuis le cloud Vagrant, puis provisionnée selon la 
définition du Vagrantfile. Le vagrantfile provisionne grâce aux scripts présents dans le dossier `provision`.

Une fois la machine provisionnée, vous pouvez vous connecter à celle-ci via la commande:

```bash
vagrant ssh
```

Les commandes `docker` et `docker-compose` sont disponibles dans cet environnement.

## Paramétrage

Il est possible de paramétrer la VM avec le fichier `config.yaml`. Copier le fichier `config.example.yaml` vers 
`config.yaml`, et modifier selon vos besoins.

## Configuration de git sur la VM et sur la machine hôte

* Utiliser som prénom & nom comme et adresse mail GFI.

```bash
git config --global user.name "Prénom Nom"
git config --global user.email "prenom.nom@gfi.fr"
```

* Pour éviter de polluer l'historique des commits avec des merge commit.

```bash
git config --global pull.rebase true
```

## Configuration des sauts de ligne sur le projet

Pour éviter tout problème lors du partage de fichier entre Linux et Windows, il faut prendre quelques précautions au 
sujet des caractères de saut de lignes.

- Paramétrer l'option pour git `core.autocrlf false`.

```bash
git config --global core.autocrlf false
```

- Paramétrer l'éditeur de code pour utiliser les sauts de ligne linux uniquement (LF).

## Configuration de Acrylic DNS Proxy (optionnel)

Acrylic DNS Proxy permet de router des ensemble de noms de domaines vers la VM, sans avoir à modifier le fichier 
`/etc/hosts` pour chaque projet.

- Dans les propriétés de la carte réseau, définir le serveur DNS de l'interface IPv4 à `127.0.0.1`, et IPv6 à `::1`.

- Menu Démarrer > Edit Acrylic Configuration File > Modifier les paramètres suivants

```
PrimaryServerAddress=172.16.78.251
SecondaryServerAddress=10.45.6.3
TertiaryServerAddress=10.45.6.2
```

- Menu Démarrer > Edit Acrylic Hosts File > Ajouter la ligne suivante à la fin du fichier

```
192.168.1.100 *.test
```

## Rappel des commandes Vagrant

- Lancer la VM

```bash
vagrant up
```

- Arrêter la VM
```bash
vagrant halt
```

- Redémarrer la VM
```bash
vagrant reload
```

- Provisionner la VM
```bash
vagrant provision
```

## Installation automatique des certificats d'autorité racine

Le plugin `vagrant-certificates` permet d'installer automatiquement sur la VM les certificats racine situés dans un 
dossier de l'hôte.

## Synchronisation des fichiers du projet via NFS

Il est possible d'utiliser un point de montage NFS via le plugin `vagrant-nfs4j`.

Il faut paramétrer la section `synced_folder` dans le fichier `config.yaml` comme décrit dans la section **Paramétrage**.

```yml
synced_folders:
  user:
    source: 'C:\Users\user' # chemin absolu ou relatif au fichier Vagrantfile
    target: '/c/Users/user' # chemin absolu ou relatif au home de l'utilisateur de la VM
  projects:
    source: 'C:\devel\projects' # chemin absolu ou relatif au fichier Vagrantfile
    target: '/c/devel/projects' # chemin absolu ou relatif au home de l'utilisateur de la VM
```

Lorsque la section `synced_folders` est renseignée dans le fichier de configuration, Vagrant va automatiquement 
lancer nfs4j-daemon pour monter les dossiers spécifiés via NFS.

Pour supporter les liens symboliques, il est nécessaire de configurer la [Stratégie de Sécurité Locale pour autoriser la création de liens symboliques](https://github.com/inetum-orleans/nfs4j-daemon#symbolic-links-support-on-windows) 
pour votre utilisateur.

### Libérer de l'espace disque

Attention, car `dc down` supprime les containers ! Avant de lancer ces commandes pour libérer de l'espace disque,
il vaut mieux donc arrêter les projets que l'on souhaite conserver avec `dc stop` pour que les volumes associés
ne soient pas détruits.

 ```
 docker system prune  --filter "until=24h"
 docker volume rm $(docker volume ls -qf dangling=true)
 ```

 Dans certains cas, le dossier `/var/lib/docker` est pollué avec des dossiers `*-removing` et `*-init`. Ils peuvent être supprimés.

 ```
 # A executer avec l'utilisateur root
 cd /var/lib/docker
 find . -name "*-init" -type d -exec rm -R {} +
 find . -name "*-removing" -type d -exec rm -R {} +
 ``` 

### Problèmes liés au VPN

Si le client vpnc ne parvient pas à se connecter, il est nécessaire de vérifier le MTU des cartes réseaux (1500).

Voir [https://www.virtualbox.org/ticket/13847](https://www.virtualbox.org/ticket/13847)

### Interface PORTAINER

Une interface graphique de gestion des containers/volumes docker est accessible via l'url : `portainer.test` (à ajouter dans le fichier hosts de la machine hôte au besoin).
Elle est basée sur [PORTAINER](https://portainer.readthedocs.io/en/stable/index.html) 