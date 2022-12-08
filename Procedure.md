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
