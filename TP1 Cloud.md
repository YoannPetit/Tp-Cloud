# **TP1 Cloud**
*Sieger Jonathan & Petit Yoann*

Une fois après avoir installé Vagrant avec le Wizard Windows, nous l'avons ouvert et modifié le fichier Vagrantfile. Nous avons tout d'abord ajouté un disque de 10 Gb sur une machine, pour cela nous avons créer une variable nommé disk qu'on a affecté à un chemin d'accès ainsi qu'une variable tailledisk =10 (la taille de notre disque):

![](https://i.imgur.com/bEtFh9b.png)
    
Mais pour que cela fonctionne nous avons du rajouter un controlleur SATA dans la partie : *config.vm.provider :virtualboc do |vb|*

![](https://i.imgur.com/p2tLmcA.png)
    
Après cela on doit parametrer internet en allant à la ligne 145 il y a une variable nommé "IP", on y met une adresse IP fixe puis on défini à partir de quel IP elle commence ici 200+i (i=1) donc mon ip sera 201.
Et on met notre réseau en mode Public et pour être sur que vagrant se connecte sur la bonne carte réseau on y met le nom complet de notre carte réseau :
    ![](https://i.imgur.com/iNZfEvo.png)
    
On lance Vagrant dans notre CMD windows avec la commande `vagrant up` puis on se connecte sur notre VM en ssh via la commande `vagrant ssh`
Une fois connecté on regarde sur le disque est bien monté avec la commande `lsblk` : 
    ![](https://i.imgur.com/pW8BeGG.png)

Et pour vérifier que notre IP a bien changé on tape la commande `ip a` :
    ![](https://i.imgur.com/iY22P9J.png)

Maintenant il faut pouvoir lancer 3 VM au lancement de vagrant pour cela il faut modifier la valeur d'une variable à la ligne 26 :
    ![](https://i.imgur.com/iqBOkXr.png)
Mais ça ne suffit pas si on fait juste ça on aura un problème lors de la création des disques car le disque dur qui va se créer aura le même nom que le premier disque crée donc il faut modifier quelque code dans l'initialisation du disque dur.
    ![](https://i.imgur.com/Otfh3nN.png)
    
Le code modifie le nom du disque à l'aide d'une concaténation.

Pour notre swarm nous avons 8 machines, donc notre swarm sera composé de 3 managers et 5 workers.
1 manager et 2 workers sur un PC et sur l'autre 2 manager et 3 workers.

Mais tout d'abord il faut paramétrer les metrics (qui renseigne le port et l'adresse utilisé par defaut par le serveur) dans le fichier : /etc/docker/daemon.json

`{ 
   "metrics-addr":"0.0.0.0:9323", 
   "experimental": true 
}`

Une fois cela fait, le Manager principal doit générer le swarm à l'aide de la commande : ` docker swarm init --advertise-addr 192.168.25.101` et ensuite le même manager tape la commande : `docker swarm join-token manager
` qui génère une commande à taper sur les managers qui doivent rejoindre le swarm : 

`docker swarm join --token SWMTKN-1-1vlubz47tueiwbo0056agmxvp14rwmthas2god794jorrxlhuq-artm2nb4fcmjqcbhy2ovlh019 192.168.25.101:2377 `

Il y a la même chose pour les workers : `docker swarm join-token worker` et pour que les workers rejoignent : 
`docker swarm join --token SWMTKN-1-1vlubz47tueiwbo0056agmxvp14rwmthas2god794jorrxlhuq-18bi1tujyccgchhuvi8z8x66l 192.168.25.101:2377`

Avec la commande `docker node ls` on peut voir les workers et manager présent dans le swarm : 
![](https://i.imgur.com/Ox53Jxz.png)


A l'aide de la commande`docker stack deploy python_dirty_app -c docker-compose.yml` on lance nos services qui sont dans notre docker-compose :
![](https://i.imgur.com/qwqY2NH.png)

Installation docker-compose : 
![](https://i.imgur.com/29mgvIT.png)

![](https://i.imgur.com/nnhK3yZ.png)


Registre (qui permet de déployer le service dans tous les noeuds): 
![](https://i.imgur.com/uC1Nz1g.png)

On modifie le docker-compose pour lui indiquer l'ip du registre:
![](https://i.imgur.com/Jj0LYHt.png)


On build le docker compose : 
![](https://i.imgur.com/HS1laCX.png)

Maintenant qu'il est build on peut le push : 
![](https://i.imgur.com/J1azNbm.png)

***Q1 : comment ce conteneur a-t-il pu lancer des conteneurs sur votre hôte ?***

Weave a utilisé notre swarm pour lancer les conteneurs sur l'hôte.

Une fois cela fait on va commencer la mise en place du cluster sur chaque noeud. Tout d'abord on créer un Monitor à l'aide de la commande on fait ça sur 3 noeuds au total en modifiant le champ "MON_IP par l'ip de la machine" : 
`docker run -d --net=host \
--restart always \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.25.201 \
-e CEPH_PUBLIC_NETWORK=192.168.25.0/24 \
--name="ceph-monitor1" \
ceph/daemon mon` et on peut vérifier que c'est construit à l'aide de la commande : Avec la commande `docker exec ceph-monitor1 ceph status`:
![](https://i.imgur.com/bdRzSmx.png)


Ensuite on met en place un manager via la commande sur 3 noeuds au total :
`docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
--name="ceph-mgr" \
--restart=always \
ceph/daemon mgr`

Ce qui donne : 
![](https://i.imgur.com/MTs5vA3.png)


Pour la mise en place de l'OSD on doit récuperer l'output OSD dans un conteneur Monitor on tape :` ceph auth get client.bootstrap-osd` 
![](https://i.imgur.com/yKKhhWN.png)

Puis on tape ces comandes (ici elles sont dans un fichier texte pour une facilité d'éxécution et éviter les erreurs de copie ou d'écriture) à faire sur TOUS les noeuds : 
`docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /dev/:/dev/ \
-e OSD_FORCE_ZAP=1 \
-e OSD_DEVICE=/dev/sdc \
-e OSD_TYPE=disk \
--name="ceph-osd1" \
--restart=always \
ceph/daemon osd_ceph_disk`

Ce qui donne : 
![](https://i.imgur.com/OmredJK.png)
On en a 6 sur 8 les deux derniers ne veulent pas se monter on les restarts les supprimes et on les refaits mais rien ne change ils n'up pas.

Mise en place de la MDS à l'aide de la commande : `docker run -d --net=host --name ceph-mds --restart always -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph -e CEPHFS_CREATE=1 -e CEPHFS_DATA_POOL_PG=512 -e CEPHFS_METADATA_POOL_PG=512 ceph/daemon mds`

Une fois cela fait on va créer un secret dans un conteneur moniteur, en tapant la commande suivante : `ceph auth get-or-create client.dockerswarm osd 'allow rw' mon 'allow r' mds 'allow' > keyring.dockerswarm`

Puis sur tous les noeuds on tape : 
`mkdir /data `
`echo "192.168.25.201,192.168.25.202,192.168.25.203:6789:/      /data/      ceph      name=dockerswarm,secret=AQClfuFbDmoJMhAAmq6KgGlyuW5NFrF1+8UXNA==,noatime,_netdev 0 2" /etc/fstab`
`mount -a`

On a eu une erreur : *"can't read superblock"*
Après 4 destructions de VM on a réussi à mettre 6 VM sur 8, on va du coup faire le NFS.


***Q2 : expliquer rapidement le principe d'un système de fichiers distribué (distributed filesystem)***

Le principe de fichiers distribué et de pouvoir partager des fichiers sur différents périphérique via un systeme de réplicationn fichier sur un périphérique il est automatiquement répliqué sur les autres périphériques.

***Q3 : proposer une façon d'automatiser le déploiement cette conf CEPH (Vagrant ? Swarm stack ? autres ?) Par ex, si on veut rajouter un noeud ?***

On peut faire un docker compose avec toute les commandes et utiliser la fonctionnalitée du swarm pour que ça le fasse sur tous les noeuds.


# NFS : 
***Q3 : proposez une façon d'automatiser le déploiement cette conf NFS***

On met la configuration dans un DockerFile

Pour installer un NFS nous avons pris une machine CENTOS pour séparer le stockage du reste et nous avons tapé les commandes suivante pour paramétrer le serveur NFS : 
Pour installer l'utilitaier NFS : `yum install nfs-utils`
Ensuite on créer un dossier de partage ici "toto" : `mkdir /toto`
On change ses permissions : `chmod -R 755 /toto`
                            `chown nfsnobody:nfsnobody /toto`
On active et lance les services NFS à l'aide des commandes :
`systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap`

Puis on configure le fichier exports (/etc/exports) qui détermine quel dossier partager sur quel poste ici tous les postes du réseau 192.168.25.0/24 : 
`/srv/data   192.168.25.0/24(rw,sync,no_root_squash,no_all_squash)`

On relance le service : `systemctl restart nfs-server`
et on paramètre le firewall pour autoriser les services NFS : 
`firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload`

Le serveur est à présent configuré, nous allons maintenant faire les clients. Nous avons configuré le Ignition VagrantFile, pour cela on a remarqué que le fichier *config.ign* est le fichier qui comporte les paramètres de boot. Pour cela nous avons fait un copier coller du fichier *config.ign.sample* et l'avons modifié en rajoutant 
`{
        "contents": "[Unit]\nBefore=remote-fs.target\n[Mount]\nWhat=192.168.25.150:/srv/data\nWhere=toto\nType=nfs\n[Install]\nWantedBy=remote-fs.target",
        "enable": true,
        "name": "toto.mount"
      }`
Dans la rubrique "units" et après ça on a modifié le nom du fichier en* config.ign*