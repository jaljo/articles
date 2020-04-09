# Formation docker

2020-02-05

## Différence container / VM

Kernel <-> composants via des drivers
Process parent: Superviseur
    -> spawn des processes (PHP, slack, etc) : enfants du process

Chaque process peut être alloué à différents threads du CPU

VM: emulation du matériel, du kernel et de l'os, et de tous ses composants
Container docker: un process du systeme comme un autre, avec certaines particularité
    -> profite de certaines features du kernel hôte:
        -> cnames (le FS du container est isolé du FS de l'hôte)
        -> cgroups
        -> interfaces reseaux qui lui sont propre
        -> limitation des ressources (RAM, CPU)

Un container docker = une responsabilité (php, nginx...)

A lire: [Twelve factors](https://12factor.net/fr/)

## dockerd

daemon docker piloté par une api http
`# docker ...` client HTTP de l'api de dockerd

## commandes de base

container: instance de l'image (POO: object instancié)

image: snapshot de FS a un instant t, distribuable via des registry (docker hub,
quay.io...). (POO: classe) + métadonnées (instructions specifiques comme le port
binding, les tags...)

container identifiable par:
    - identifiant hash md5
    - name

`docker run -ti --rm --name [nom] ubuntu:xenial sh`
`--tty --interactive => -ti` redirige tty vers le process

`docker ps` lister les containers qui tournent sur l'hôte

`docker create --name test ...`: créé le container sur l'hôte mais le l'exécute pas
`docker start test`: démarre le container
`docker rm test`: supprime le container
`docker run`: concaténation du create & start & pull si l'image est absente en local
`docker exec [container id]`: exécuter une commande dans un container qui tourne déja

`docker inspect [name]`: obtenir des informations sur le container

Une image: découpée en layers
chaque instruction dans le docker file: un layer
chaque layer identifé par un hash

Chaque layer est mis en cache. si on fait un changement dans le dockerfile tous
les steps apres l'instruction changée sont invalidées dans le cache, et rebuild
car on ne sait pas si les instructions avaient des dépendances fortes avec
l'instruction qui a changée.

`docker build .` prendre pour contexte courant le fichier Dockerfile dans . pour
build une image

`CMD` commande exécutée au moment ou le container démarre. Une par dockerfile

`WORKDIR` répertoire de travail (ou sont exécutées les commandes dans le FS du container)

fichier .dockerignore : similaire au gitignore. Permet d'ignorer les dossiers
a ne pas considérer lors du build

`ARG`
`ENV`
