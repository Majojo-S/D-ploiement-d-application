### LEPRETRE Guerric et SANDRAS Marine
_______
# Sujet 1 Mise en place

Se connecter à la machine de virtualisation: 
```
user@phys$ ssh marine.sandras.etu@frene11.iutinfo.fr
```

## 1. Simplifier la connection ssh
Pour éviter de taper notre mot de passe à chaque connection, on crée une clé qui sera reconnu par la machine de virtualisation.

On générera une clé public connu et une clé privée.

1. Générer une clée: 
```
user@phys$ ssh-keygen
```
On vous demandera un chemin pour le fichier de la clé puis une passphrase (un mot de passe) qui sera la clé privée.

2. Copier la clée publique générer dans autorized_key de la machine de virtualisation : 
```
user@phys$ ssh-copy-id (-i "cheminCleePublique") marine.sandras.etu@frene11.iutinfo.fr
```

La première connection, suite à ceci, on vous demandera la clé privée et normalement aucun mot de passe et passphrase ne sera demander pendant les prochaines connections.

La clé sera utiliser à travers un agent ssh. Cette agent tourne en arrière plan pendant toute la durée de votre session.

## 2. Creer des vm

Il nous faut des commandes pour créer des vm, on va les chercher en tapant :
```
user@virtu$ source /home/public/vm/vm.env
```
*(Pour éviter de taper cette commande à chaque fois qu'on se connecte en ssh, on écrit cette commande dans le **.bashrc**)*

### Liste des commandes :

Créer la machine: 
```
user@virtu$ vmiut creer matrix
```
Afficher les vm: 
```
user@virtu$ vmiut lister
```
Démarrer une vm: 
```
user@virtu$ vmiut demarrer matrix
```
Arreter une vm: 
```
user@virtu$ vmiut arreter matrix
```
Supprimer une vm (la vm doit être arréter): 
```
user@virtu$ vmiut supprimer matrix
```
Info sur une vm: 
```
user@virtu$ vmiut info matrix
```

## 3. Configurer la vm
### Changer l'ip

*On ne peut pas faire de ssh pour changer l'ip !*
*On utilise ip pour se connecter donc on ne peut pas la désactiver*

Se connecter a la vm: 
```
user@virtu$ vmiut console matrix
```
Se mettre en root
```
user@vm$ su -
```
Désactiver l'ip
```
root@vm# ifdown ensp0s3
```
Configurer l'ip dans **/etc/network/interfaces** : 
```
iface ensp0s3 inet static
	address 192.168.194.3/24
	gateway 192.168.164.2
```
Redémarrer l'ip
```
root@vm# ifup ensp0s3
```
*(si il y a une erreur de ifup essayer ```ip add del 192.168.194.3/24 dev ensp0s3``` et refaire ```ifup ensp0s3```)*

### Configuration du proxy

Puisqu'il y a un proxy sur les machines de l'IUT, on doit le configurer pour que la vm puisse avoir accès à internet.

Le proxy étant **http://cache.univ-lille.fr:3128**.

Ajouter dans **/etc/environment**:
```
HTTP_PROXY=http://cache.univ-lille.fr:3128
HTTPS_PROXY=http://cache.univ-lille.fr:3128
http_proxy=http://cache.univ-lille.fr:3128
https_proxy=http://cache.univ-lille.fr:3128
NO_PROXY=localhost,192.168.194.0/24,172.18.48.0/22
```
*(NO_PROXY désigne les adresses n'ayant pas besoin du proxy)*

### Mise à jour

Mettre à jour tout les packets: 
```
root@vm# apt update && apt full-upgrade
```
*(Commande à effectuer de temps en temps)*
si une fenêtre s'ouvre cocher la case []/dev/sda (avec espace) et appuyer sur entrer.

Redémarrer la machine au cas où on a une nouvelle version du kernel
```
root@vm# reboot
```

### Installation de nouveaux packets

On installe des outils
```
root@vm# apt install vim less tree rsync
```

## 4. Aller plus loin

Pour éviter de taper tout l'adresse des machines en se connectant en ssh, on crée des alias dans le fichier **HOME/.ssh/config** :
```
Host virtu
		HostName frene11.iutinfo.fr
		User marine.sandras.etu

Host vm
		HostName 192.168.194.3
		User user
```

Pour se connecter directement en ssh à la vm de notre machine physique (donc sans passer par la machine de virtualisation).

On écrit dans le fichier **.bashrc** la ligne :
```
alias vmjump = 'ssh -J virtu vm'
```

_______
# Sujet 2 Dernière configuration sur VM

## 1. Changer le nom de la VM
*(il faut être en root)*

La VM se nomme par défaut *debian* et on veut l'appeler *matrix*
```
Changer son nom dans le fichier **/etc/hostname** en replaçant *debian* par *matrix*.
```

Pour l'appeler par matrix en effectuant par exemple un **ping**
```
Changer le fichier **/etc/hosts** en replaçant *127.0.1.1   debian* par *127.0.1.1  matrix*.
```

## 2. Commande sudo

Pour installer la commande **sudo** :
```
root@vm# apt install sudo
```

On veut que user est accès à la commande **sudo**, on effectue :
```
root@vm# usermod -aG sudo user
```
*(-aG permet d'ajouter user au groupe sudo)*


Si on veut créer un autre user ayant accès à sudo, on fait :
```
root@vm# adduser user sudo
```

## 3. Configuration de l'horloge

En effectuant la commande **date** sur la vm, la machine de virtualisation et physique, on remarque que la vm est en avance d'un heure.

```
user@vm$ journalctl -u systemd-timesyncd
```
Cette commande permet de voir quand la vm a été créé et la dernière fois qu'elle a été démarré.

Pour arranger le problème d'heure, on modifie dans le fichier **/etc/systemd/timesyncd.conf** la ligne **NYP** :
```
NTP=ntp.univ-lille.fr
```
*(si vous êtes sur user n'oublier pas de mettre sudo avant la commande pour accèder au fichier)*

On redémarre le système pour que l'heure se règle :
```
user@vm$ sudo systemctl restart systemd-timesyncd.service
```
*ou*
```
root@vm$ systemctl restart systemd-timesyncd.service
```

## 4. Installation d'un serveur de base de donnée

On installe PostgreSQL :
```
user@vm$ sudo -E apt install postgresql
```
*(-E est pour préserver les configurations du proxy avec sudo)*

Vérifier l'état de postgreSQL :
```
user@vm$ systemctl status postgresql
```

S'y connecter :
```
user@vm$ sudo -u postgres psql
```

### Sur postgreSQL

Créer un utilisateur : 
```
postgres=# CREATE USER matrix WITH PASSWORD 'matrix';
```

Créer une base de donnée :
```
postgres=# CREATE DATABASE matrix;
```

Donner l'accès à matrix pour qu'il soit propriétaire de la base de donnée matrix :
```
postgres=# GRANT ALL privileges ON DATABASE matrix TO matrix;
```

### Sur le shell

On quitte postgreSQL pour retourner sur le shell d'user :
```
postgres=# \q
```

On va créer une table test dans la base de donnée matrix :
```
user@vm$ psql -h localHost -c "create table test(id int);" matrix matrix
```
*-h* demande le serveur qui est le localHost

*-c* demande la requète à éxécuter

*Le premier matrix* représente l'utilistaeur

*Le dernier matrix* représente la base de donnée

____________
# Sujet 3 Installation et configuration de Synapse

## 1. Accès à un service HTTP sur la VM
On installe un serveur HTTP sur la vm :

```
user@vm$ sudo -E apt install nginx
```
On vérifie qu'il est démarré avec la commande :
```
user@vm$ systemctl status nginx.service
```
On installe curl (un client HTTP) qui sert à ...
```
user@vm$ sudo -E apt install curl
```
On vérifie qu'on peut accéder au serveur **nginx** depuis la vm :
```
user@vm$ curl http://localhost
```

Vous devez obtenir ceci :
```HTML
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
--------------------
On essaye la même chose mais depuis la machine de virtualisation :
```
user@virtu$ curl --noproxy '*' http://192.168.194.3:80
```
Si on n'utilise pas le *noproxy* cela ne marche pas.
La vm est configurer pour ne pas passer par le proxy pour accèder au localhost, mais la machine de virtualisation est configurer pour passer par le proxy pour una adresse http. Le proxy ne connait pas la vm et ne peut donc pas le trouver.

--------------------
On veut maintenant accèder au service par notre machine physique.

*(Ce n'est pas posible directement puisque seul la machine de virtualisation connait et peut accèder au réseau de la vm)*

Pour ce faire, on doit utiliser la fonction tunnel de **ssh**. 

*(on tape la commande sur la machine de virtualisation)*
```
user@virtu$ ssh -L 0.0.0.0:9090:192.168.194.3:80 vm
```
*(vm est alias de user@192.168.194.3)*

On tape ensuite l'URL *http://machine-virtualisation.iutinfo.fr:9090* dans un navigateur de notre **machine physique**, et on accède au service de la vm. 
*machine-virtualisation* a remplacer par le nom de votre machine de virtualisation.

Pour éviter d'utiliser l'option **-L** à chaque fois, on écrit dans le fichier **.ssh/config** de la machine de virtualisation (rajouter les lignes dans l'host vm):
```
    LocalForward 0.0.0.0:9090 192.168.198.3:80
```

## 2. Installation de Synapse

### installation du paquet
Installation de paquet nécessaire pour synapse :
```
user@vm$ sudo -E apt install -y lsb-release wget apt-transport-https
```
Récupération du paquet de synapse :
```
user@vm$ sudo -E wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
```

écrire dans un fichier les sources pour le paquet
```
user@vm$ echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/matrix-org.list
```
Faire une mise à jour du système
```
user@vm$ sudo -E apt update
```

installation de synapse :
```
user@vm$ sudo -E apt install matrix-synapse-py3
```
Pendant l'installation du packet un message vas apparaitre, il faut mettre machine_virtualisation.iutinfo.fr:8008 et valider. 
Mettre *non* pour la recuperation de donné pour améliorer le logiciel (ce n'est pas necéssaire pour notre vm).

### Paramétrage spécifique pour une instance dans un réseau privé

Les paramètres par défaut de Synapse considèrent que le serveur est accessible par internet et qu’il ne cherche pas à contacter des éléments situés sur un réseau privé, or ce réseau est privé.

dans **/etc/matrix-synapse/homeserveur.yaml** modifier les dernières lignes
```
trusted_key_servers: 
    - serveur_name: "matrix.org"
```

changer en :
```
trusted_key_servers: []
```
*(supprimer le serveur_name)*

### Utilisation d’une base Postgres

Puisque Synapse utilise *sqlite* par défaut pour la base de donnée, on doit le configurer pour qu'il utilise postgreSQL à la place.

Modifier au fichier **/etc/matrix-synapse/homeserveur.yaml** les lignes :
```
database:
  name: sqlite3
  args:
```
En :
```
database:
  name: psycopg2
  args:
    user: matrix
    password: matrix
    database: matrix
    host: localhost
    cp_min: 5
    cp_max: 10
```

La base de donnée "matrix" créée précédemment avait les options par défaut, elle doit être réécrite puisque Synapse ne les comprend pas.

Se connecter à PosgreSQL :
```
user@vm$  sudo su - postgres
```

Détruire l'ancienne base de donnée :
```
postgres@vm$ dropdb matrix;
```

Créer une base de donnée :
```
postgres@vm$ createdb --encoding=UTF8 --locale=C --template=template0 --owner=matrix matrix;
```
Quitter :
```
postgres@vm$ exit
```

On vérifie si la base PostgreSQL est utilisée par Synapse. Pour cela on redémarre Synapse:
```
user@vm$ sudo systemctl restart matrix-synapse.service
```
On regarde le contenu de la base de donnée matrix :
```
user@vm$ psql -h localhost -c "\d" matrix matrix
```
Tout est bon si il y a aucune erreur et qu'il y a un affichage de table comme celui-ci:
```
                              List of relations
 Schema |                      Name                      |   Type   | Owner
--------+------------------------------------------------+----------+--------
 public | access_tokens                                  | table    | matrix
 public | account_data                                   | table    | matrix
 public | account_data_sequence                          | sequence | matrix
 public | account_validity                               | table    | matrix
 public | application_services_state                     | table    | matrix
 public | application_services_txn_id_seq                | sequence | matrix
 public | application_services_txns                      | table    | matrix
```


### Création d’utilisateurs

Pour créer un utilisateur du serveur, il faut utiliser le script **register_new_matrix_user** et ajouter dans **/etc/matrix-synapse/homeserveur.yaml** la ligne suivante, servant à partager la clé avec le script :
```
registration_shared_secret: "zesrdtyfihojpighjfchgxc"
```
On redémarre Synapse(commande vu précédemment)

On tape ensuite :
```
user@vm$ register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml
```
Puis on suit les instruction pour créer l'utilisateur.

### Connexion à votre serveur Matrix

On passe par un client element à l'adresse *http://tp.iutinfo.fr:8888* pour se connecter à notre serveur.
Sélectionner se connecter puis modifier le serveur d'accueil matrix.org par votre url de votre machine de virtualisation (*http://machine-virtualisation.iutinfo.fr:9090*).

Vous remarquerez qu'il ne trouve pas le serveur, c'est normal car l'url mène au serveur nginx configuré plus tôt.
Il faut changer la ligne *LocalForward* du fichier (de votre machine de virtualisation) **.ssh/config** de votre host vm avec :
```
LocalForward 0.0.0.0:9090 localhost:8008
```

Connecté vous à votre vm et le serveur d'accueil sera reconnu.
Connecté vous avec l'utilisateur créé plus tôt a votre serveur.

Vous atteignez l'interface de synapse et maintenant vous pouvez créer un salon pour discuter avec les utilisateur enregistrer sur votre serveur Synapse.

### Activation de l’enregistrement des utilisateurs 

Pour activer l'enregistration d'utilisateur non connu sans vérification, il faut rajouter les lignes suivant dans le fichier **/etc/matrix-synapse/homeserveur.yaml**
```
enable_registration: true
enable_registration_without_verification: true
```

Il suffit de créer un nouveau compte, grâce à l'element web, en mettait le serveur visé.

_______
# Sujet 4 Installation et configuration de Element Web

## Element Web

![comparaisonElementWeb](apachevsnginx.png "Comparaison").

On a choisie Apache puisqu'on le connait mieux et qu'on préfère séparer le serveur web du reverse proxy.


_____________
Installation :
```
user@vm$ sudo -E apt install apache2
```

Configuartion pour que l'Element soit accessible sur le port 8080 sur la vm.

Aller dans le fichier **/etc/apache2/ports.conf** pour remplacer :
```
Listen 80
```
En :
```
Listen 8080
```

Redirection ssh pour que le serveur web soit accessible sur le port 9090 de la machine de virtualisation :

Changer la ligne *LocalForward* du fichier (de votre machine de virtualisation) **.ssh/config** de votre host vm avec :
```
LocalForward 0.0.0.0:9090 localhost:8080
```

## Reverse Proxy pour Synapse
