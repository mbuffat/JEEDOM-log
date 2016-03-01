# JEEDOM limitation des fichiers logs

Installation pour  un système JEEDOM sur un raspberry pi avec une carte SD

## Introduction
Ayant découvert récemment la domotique avec [JEEDOM](https://www.jeedom.com/site/fr/), un logiciel de domotique open-source pouvant s'installer sur un Raspberry avec une communauté dynamique, j'ai optimisé mon installation Jeedom sous Raspberry utilisant une carte SDi de façon à :

- limiter les écritures sur la carte SD pour fiabiliser le système

En effet, un des problèmes récurrants d'utilisation d'un raspberry est qu'un système Unix écrit  beaucoup d'information dans les fichiers log du système. Le problème est que JEEDOM utilise beaucoup le sudo pour exécuter des commandes en tant que root, ce qui a pour conséquence un fichier log système auth.log qui croit beaucoup. C'est un fonctionnement normal dans le monde unix où tout accès root est logger par le système.
Mais comme tout système unix, on peut aussi contrôler et paramétrer le système, et dans notre cas diminuer la taille de auth.log
C'est ce que j'ai fait dans mon installation JEEDOM et que je partage ici.

## installation du répertoir /var/log en mémoire
Au lieu d'écrire les logs sur la carte SD, on va les écrire en mémoire en créant le répertoire log sur un filesystem tmpfs en mémoire. Pour cela rajouer les lignes suivantes dans le fichier
** /etc/fstab **. On en profite aussi pour créer le fichier /tmp sur un filesystem tmpfs pour optimiser JEEDOM V2.0 (comme indiquer sur la documentation JEEDOM)
```
# fichier log
tmpfs    /var/log    tmpfs    defaults,noatime,nosuid,mode=0755,size=10M    0 0
# jeedom
tmpfs    /tmp 	     tmpfs    defaults,noatime,nosuid,size=64m 0 0
```


