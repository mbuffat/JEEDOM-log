# JEEDOM limitation de la taille des fichiers logs

Installation pour  un système JEEDOM sur un raspberry pi avec une carte SD

## Introduction
Ayant découvert récemment la domotique avec [JEEDOM](https://www.jeedom.com/site/fr/), un logiciel de domotique open-source pouvant s'installer sur un Raspberry avec une communauté dynamique, j'ai optimisé mon installation Jeedom sous Raspberry utilisant une carte SDi de façon à :

- limiter les écritures sur la carte SD pour fiabiliser le système

 Le problème est que JEEDOM utilise beaucoup le sudo pour exécuter des commandes en tant que root, ce qui a pour conséquence un fichier log système auth.log qui croit beaucoup. C'est un fonctionnement normal dans le monde unix où tout accès root est logger par le système.
Mais comme tout système unix, on peut aussi contrôler et paramétrer le système, et dans notre cas diminuer la taille de auth.log
C'est ce que j'ai fait dans mon installation JEEDOM en utilisant des informations en provenance de différents blogs que j'ai testé sur mon installation et que je partage ici.

## installation du répertoire /var/log en mémoire
Au lieu d'écrire les logs sur la carte SD, on va les écrire en mémoire en créant le répertoire log sur un filesystem tmpfs en mémoire. Pour cela on doit rajouter les lignes suivantes dans le fichier
** /etc/fstab **. On en profite aussi pour créer le fichier /tmp sur un filesystem tmpfs pour optimiser JEEDOM V2.0 (comme indiquer sur la documentation JEEDOM)
```
# fichier log
tmpfs    /var/log    tmpfs    defaults,noatime,nosuid,mode=0755,size=20M    0 0
# optimisation jeedom
tmpfs    /tmp 	     tmpfs    defaults,noatime,nosuid,size=64m 0 0
```
En rebootant le système , on a maintenant des logs écrits en mmémoire, mais à chaque reboot le système de fichiers logs est éffacé, ce qui dans mon cas n'est pas un problème (on pourrait mettre en place un système cron pour sauvegarder régulièrement les logs).
Cependant il y un petit problème, car certains programmes (nginx par ex.) utilisent des sous-répertoires (** /var/log/nginx **)  qui ne sont pas recrées automatiquement. Il faut donc les recréer lors du boot du système.

Pour cela on lance au démarage un service ** prepare-dirs** , que j'ai trouvé sur ce blog [Ramdisk for the Raspbzerry]( https://www.a-netz.de/blog/2013/02/ramdisks-for-the-raspberry)
```
#!/bin/bash
#
### BEGIN INIT INFO
# Provides:          prepare-dirs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Required-Start:  
# Required-Stop:   
# Short-Description: Create /var/log/nginx directory on tmpfs at startup
# Description:       Create /var/log/nginx directory on tmpfs at startup
### END INIT INFO
DIR=/var/log/nginx
#
# main()
#
case "${1:-''}" in
  start)
    touch /var/log/lastlog
    # create the /var/log/nginx needed by webserver
    if [ ! -d ${DIR} ]; then
      mkdir ${DIR}
      chmod 755 ${DIR}
    fi
    ;;
  stop)
    ;;
  restart)
   ;;
  reload|force-reload)
   ;;
  status)
   ;;
  *)
   echo "Usage: $SELF start"
   exit 1
   ;;
esac
```
On écrit ce fichier dans /etc/init.d avec les bons droits (root rwx) 
```
chmod 0755 /etc/init.d/prepare-dirs
```
et on l'installe dans le système unix avec la commande 
```

update-rc.d prepare-dirs defaults 01 99
```
pour qu'il soit automatiquement executer au démarrage.

## limitation des logs systèmes

Pour éliminer les logs des sudo par www-data il faut modifier **/etc/pam.d/sudo** comme ci dessous en ajoutant la ligne session [sucess=1....
```
#%PAM-1.0
@include common-auth
@include common-account
# remove log for www-data
session [success=1 default=ignore] pam_succeed_if.so service in sudo quiet uid = 0 ruser = www-data
@include common-session-noninteractive
```
puis en utilisant l'utilitaire **visudo** on ajoute la ligne suivante :

```
Defaults:www-data    !syslog
```
On peut faire la mame chose pour les logs su par www-data dans **/etc/pam.d/su** en modifiant la fin du fichier comme suit:

```
# The standard Unix authentication modules, used with
# NIS (man nsswitch) as well as normal /etc/passwd and
# /etc/shadow entries.
@include common-auth
@include common-account
# remove log for www-data
session [success=1 default=ignore] pam_succeed_if.so service in su quiet ruser = www-data
@include common-session
```

## modification de rsyslog.conf

On peut aussi réduire les logs standards en modifiant le fichier ** /etc/rsyslog.conf ** . Pour cela on édite le fichier
et on modifie la ligne
```
*.*;auth,authpriv.none         -/var/log/syslog
```
par
```
# First some standard log files.  Log by facility.
#
# reduce auth log
:msg, contains, "pam_unix(su:session)" ~
:msg, contains, "Successful su for www-data by root" ~
:msg, contains, "pam_unix(cron:session)" ~
auth,authpriv.*                 /var/log/auth.log
#*.*;auth,authpriv.none         -/var/log/syslog
*.*;auth,authpriv,cron.none     -/var/log/syslog
```

## réduction du niveau de log de systemd

par défaut sur mon installation, le niveau de log de systemd (utilisé par JEEDOM) est trop important.

dans **/etc/systemd/system.conf** , j'ai mis le Loglevel à  **warning** au lieu de **info**
```
[Manager]
#LogLevel=info
LogLevel=warning
```
et de même dans **/etc/systemd/system.conf** ,
```
[Manager]
#LogLevel=info
LogLevel=warning
```

## Conclusion 

Avec ces manips  (que je ne conseille qu'à des utilisateurs avertis) j'ai réduit de façon drastique la taille des fichiers log
et donc la fiabilité de mon système.

- Pour fiabiliser encore plus le système, j'ai récupérer un vieux disque USB, qui est la majorité du temps en hibernation et je l'utilise pour faire une copie avec rsync de ma carte SD.
 
- Il faut cependant une carte SD de bonne qualité pour fiabiliser le système

Enfin si vous avez des astuces pour diminuer encore la taille des logs, je suis preneur!!

Marc BUFFAT, mars 2016
