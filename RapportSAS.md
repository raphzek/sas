# Rapport TME 
# TME 5
## ***Configuration du ssh***
Pour simplifier nos manipulations durant le TME, nous travaillerons en ssh.

Tout d'abord, installons les packages ssh à l'aide de la commande suivante :

    apt install ssh

Par la suite, il faut ajouter : 

*Enable_root ... yes
PermitRootLogin yes*

dans le fichier **/etc/ssh/sshd_config**, et relancer le service avec la commande :
etc/init.d/ssh restart  

On peut maintenant se connecter à notre machine virtuelle en ssh comme ceci :

        ssh root@ipmachine -p 2222
*(2222 est le port de la machine virtuelle)*

Note : Avec la commande :

    ssh-keygen

On va générer une clé publique du client puis on la donnera au serveur SSH pour qu'il soit enregistré dans les *known_hosts*.

## ***Mise en place des containers***
 Installons le package lxc de cette manière :

    apt install lxc
 
 Avec cette commande on obtient le statut du service lxc :

    systemctl status lxc-net.service

Puis, on l'active comme ceci :

    systemctl start lxc-net.service



 ## ***Configuration réseau des containers et de la machine hôte***


le repertoire : **/var/lib/lxc/** contient des repertoires avec la configuration des containers deja existant

On va modifier le fichier: **/etc/lxc/default.conf** et ajouter les lignes suivantes:

    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
    
Mettre à "true" la ligne suivante dans le fichier */etc/default/lxc*:

    USE_LXC_BRIDGE="true"

Redemarrer lxc-net:

    systemctl restart lxc-net.service
    
 Créons maintenant le conteneur c1 comme ceci : 

    lxc-create -n c1 -t debian -- -r buster

On peut alors le Demarer avec la commande :

    lxc-start -n c1
    
Cette ligne de commande :

    lxc-info c1

nous permet de vérifier son état. Lorsqu'on utilise cette commande on obtient ceci :

    Name:           c1
    State:          RUNNING
    PID:            927
    CPU use:        0.35 seconds
    BlkIO use:      0 bytes
    Memory use:     14 MiB
    KMem use:       3.04 MiB
    Link:           vethK756XV
    TX bytes:       705 bytes
    RX bytes:      514 bytes
    

 ### **Le Bridge :**
Ajoutons tout d'abord les lignes suivantes dans **/etc/network/interfaces** :

    auto lxc-bridge-nat
    iface lxc-bridge-nat inet static
    bridge_ports none
    bridge_fd 0
    bridge_maxwait 0
    address 192.168.100.1
    netmask 255.255.255.0

Le Bridge s'appelle : *lxc-bridge-nat*. On va alors maintenant modifier le fichier /var/lib/lxc/c1/config de cette manière :

    lxc.net.0.link = lxc-bridge-nat

A présent, on active le routage IP (forwarding) de cette manière :

    echo 1 > /proc/sys/net/ipv4/ip_forward

Et on le rend permanent comme ceci :

1. On ajoute cette ligne dans */etc/sysctl.conf* qui va activer le "packet forwarding" pour IPv4 : 
    net.ipv4.ip_forward=1
2. On éxecute la commande suivante pour mettre à jours nos changements : 
    sysctl -p /etc/sysctl.conf
3. Enfin, on redemarre c1 :
    lxc-start c1

Pour éviter tout problèmes, on tape la commande suivante dans la machine hôte :

    iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

Pour que ceci sois automatique, on rentre la commande dans **/etc/network/if-up.d/iptables** :

    echo   iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE > /etc/network/if-up.d/iptables


### **Gérer la connexion avec ping :**

Lancer c1 avec la commande ci-dessous : 
 
    lxc-attach c1

Installer la commande ping avec la commande suivante:

    apt install iputils-ping
 
 
 On obtient la réponse suivante de ping :

        ping 8.8.8.8 (adresse de google)
        64 bytes from 8.8.8.8: icmp_seq=1 ttl=51 time=16.1 ms
        
  Une fois ceci fait, on peut lancer la commande suivante :

    ping google.com

Voici ce que l'on obtient :

    PING google.com (216.58.213.174) 56(84) bytes of data.
    64 bytes from par21s04-in-f174.1e100.net (216.58.213.174): icmp_seq=1 ttl=51 time=11.5 ms

Attention : Pour pouvoir faire *ping google.com*, il faut aller dans ***etc/resolv.conf*** et ajouter une ligne :

nameserver (adresse IP de ma box)

Celle-ci va servir de DNS.

### **Comment cloner c1 ?**

Tout d'abord stoppons le comme ceci :

    lxc-stop c1

Puis on le clone de cette manière :

    lxc-copy -n c1 -N c2
    lxc-copy -n c1 -N c3



## ***Installation du serveur DHCP***

Tout d'abord on installe les packages necessaires avec la commande suivante :

    apt-get install isc-dhcp-server 

A présent, configurons les les adresses IP des serveurs de noms (DNS) sur notre serveur DHCP. Pour ceci, nous devons modifier le fichier ***/etc/dhcp/dhcpd.conf*** en fournissant l'adresse IP de notre DNS. Voici la partie du fichier modifié :

    subnet 192.168.100.0 netmask 255.255.255.0 {
    range 192.168.100.51 192.168.100.250;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.100.255;
    option routers 192.168.100.1;
    option domain-name-servers home;
    }
    authoritative; #On décommente cette ligne pour être seul sur le serveur DHCP de ce réseau
   
Le range signifie qu'on peut attribuer des adresses de 192.168.100.51 à 192.168.100.250. On inclu pas les adresses du bridge et du container dedant pour n'être confronté à aucun soucis d'adresses.

Donnons maintenant l'interface dans **/etc/default/isc-dhcp-server** comme ceci :

    INTERFACESv4="eth0" 
    INTERFACESv6=""

Puis on redémarre et on vérifie si cela fonctionne correctement :

    service isc-dhcp-server restart
    service isc-dhcp-server status

A présent, on va donner au réseau un autre appareil qui sera donc le container c2. celui-ci va nous assurer que le serveur est bien configuré. Une adresse DHCP lui sera donné automatiquement. Voici la démarche à suivre dans **/etc/network/interfaces** :

    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet dhcp


Encore une fois on redémarre comme fait précédement :

    service isc-dhcp-server restart
    service isc-dhcp-server status

Puis on vérifie si ca fonctionne comme ceci :

    ifdown eth0
    ifup eth0

De plus, on peut également vérifier l'adresse ip fourni avec la commande :

    ifconfig eth0

Le serveur nous renvoi :

    eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.100.51  netmask 255.255.255.0  broadcast 192.168.100.255
        inet6 fe80::216:3eff:fe9f:fe6a  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:9f:fe:6a  txqueuelen 1000  (Ethernet)
        RX packets 46  bytes 5463 (5.3 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 29  bytes 3490 (3.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

L'adresse inet correspond à la première du range définit précédement. Il n'y a donc pas de problème.

Essayons maintenant d'attribuer une adresse fixe qui sera 192.168.100.64 à c2.
Il faudra modifier le fichier **/etc/dhcp/dhcpd.conf** comme suit :

    host c2 {
    hardware ethernet 00:16:3e:a2:74:3b;
    fixed-address 192.168.100.64;
    }

    host c3 {
    hardware ethernet 00:16:3e:88:4a:27;
    }

Ici c2 aura donc une adresse fixe, tandis que celle de c3 sera variable.

Les adresses mac de c2 et c3 se trouves respectivement dans les fichiers **/var/lib/lxc/c2/config** et **/var/lib/lxc/c3/config** à la ligne "lxc.net.0.hwaddr".



## ***Installation d'un serveur DNS***

Tout d'abord, installons les packages dont nous aurons besoins via la commande suivante :

    apt-get install bind9      

Pour modifier le domaine de chaque machine de notre réseau, nous allons passer par la configuration du serveur DHCP. Il faut alors alors ajouter au fichier ***/etc/dhcp/dhcpd.conf*** les lignes suivantes :

    option domain-name "asr.fr";
    option domain-name-servers asr.fr;

A présent déclarons le bloc zone du domaine *asr.fr* comme ceci :

    zone "asr.fr" IN {
    type master;
    file "/etc/bind/db.asr.fr";
    };

Après avoir créé ce bloc zone pour qui notre serveur est "master", nous allons le configurer dans ***/etc/bind/db.asr.fr*** :

    ; BIND data file for local loopback interface
    ;
    $TTL	604800
    @	IN	SOA	server. server.localhost. (
             4		    ; Serial
             604800		; Refresh
              86400		; Retry
            2419200		; Expire
             604800 )	; Negative Cache TTL
    ;
    @	IN	NS	server.
    server	IN	A	192.168.100.1
    c2	IN	A	192.168.100.64
    c3	IN 	A	192.168.100.4
    www	IN	CNAME	c3  

On vérifie maintenant les le fichier de configuration ci-dessus ainsi que le fichier de zone ne génèrent pas d'erreurs de syntaxe à partir des commandes suivantes (respectivement) :

    named-checkconf /etc/bind/named.conf.local
    # aucune réponse ---> c'est ok !

    named-checkzone asr.fr /etc/bind/db.asr.fr 
    # réponse attendue --> 
    zone asr.fr/IN: loaded serial 2
    OK

    On redémarre comme ceci ce qui va prendre en compte nos modifications :

        /etc/init.d/bind9 restart
    
Dans ***etc/hostname*** de c1,c2 et c3, on modfie les noms des machines pour leur donner le nom de domaine *asr.fr* :

    server.asr.fr
    c2.asr.fr
    c3.asr.fr

Puis on indique dans le fichier ***/etc/resolv.conf*** qu'on souhaite utiliser notre nouveau DNS :

    domain asr.fr
    search asr.fr

Enfin, on modifie les lignes suivantes dans ***/etc/bind/named.conf.options*** :

    forwarders {
	192.168.0.254;
	8.8.8.8;
    };

Celles-ci nous indiques que si il ne trouve pas l'adresse requise dans le cache local il interrogera le DNS du fournisseur d'accès (192.168.0.254). Dans le cas d'un nouvel echec, il se retournera vers celui de google (8.8.8.8).


### **Résolution Inverse Nom :**

Tout d'abord, ajouter le bloc zone suivant dans ***/etc/bind/named.conf.local*** :

    zone "100.168.192.in-addr.arpa"  {
    type master;
    file "/etc/bind/db.asr.fr.inv";
    };

Puis, vérifier le contenue de ***/etc/bind/db.asr.fr.inv*** :

    ;
    ; BIND data file for local loopback interface
    ;
    $TTL	604800
    @	IN	SOA	server. 		server.localhost. (
             4       	; Serial
             604800		; Refresh
              86400		; Retry
            2419200		; Expire
             604800 )	; Negative Cache TTL
    ;
    @	IN	NS	server.asr.fr.
    4	IN	PTR c3.asr.fr.
    5 	IN	PTR	c2.asr.fr.
    2	IN 	PTR	server.asr.fr.

Enfin, on redémarre pour tout prendre en compte :

    systemctl restart bind9


### **Serveur DNS secondaire :**

Modifions tout d'abord le fichier ***/etc/bind/named.conf.options*** afin d'autoriser les transferts vers c2 (DNS secondaire) :

    allow-transfer { 192.168.100.64; };

Ensuite, on ajoute dans ***/etc/bind/db.asr.fr*** :

    IN      NS      192.168.100.64

Modifions maintenant la configuration du serveur principal en en ajoutant un ligne "notify yes" dans ***/etc/bind/named.conf.local*** :

    zone "asr.fr" IN {
    type master;
    file "/etc/bind/db.asr.fr";
    notify yes;
    };


Après avoir travaillé sur c1, faisons de même pour c2. 
Il faut modifier le fichier ***/etc/bind/named.conf.local*** de la manière suivante :

    zone "asr.fr" {
	    type slave;
	    file "/etc/bind/db.asr.fr";
	    masters {192.168.100.1; };# adresse du serveur maitre 
        (c1-->server)
    };


    zone "100.168.192.in-addr.arpa" {
	    type slave;
	    file "/etc/bind/db.asr.fr.inv";
	    masters {192.168.100.1; };# adresse du serveur maitre 
        (c1-->server)
    };

Nos deux serveur DNS fonctionnent maintenant correctement !


# TME 6


## ***Mise en place d'un serveur d'annuaire NIS***

On va se servir de c1 (server) comme serveur et donc installer le package ypserv sur celui-ci. Ce sera notre serveur NIS. NIS permet de centraliser des informations sur un réseau UNIX. Installation du package :

    apt  install nis

Par la suite s'affiche une case ou l'on doit entrer le nom de notre domaine NIS. On l'appellera "asr.fr".
A présent ajoutons dans ***/etc/sysconfig/network*** la ligne suivante :

    NISDOMAIN=asr.fr

A présent, nous devons modifier 2 fichiers. ***/etc/hosts*** où on ajoute la ligne suivante :

    192.168.100.1	server.asr.fr nis

et ***/etc/default/nis*** où on modifie la ligne **NISSERVER** comme ceci :

    NISSERVER=master
Dans ***/var/yp/Makefile***, on remplace la ligne :

    ALL = passwd group hosts rpc services netid protocols netgrp
Par :

    ALL = passwd shadow group hosts rpc services netid protocols netgrp


Maintenant, on créé la base de donnée comme ceci :

    /usr/lib/yp/ypinit -m

Puis on redémarre le NIS :

    /etc/init.d/nis restart


Après s'être occupé de la partie serveur, nous devons gérer la partie client. Comme pour le serveur, on installe tout d'abord le NIS.
Par la suite, on ajoute dans ***/etc/yp.conf*** les lignes suivantes :

    domain asr.fr server server.asr.fr
    ypserver serv.asr.fr

Pour pouvoir choisir les éléments qui seront partagés avec le serveur, on va modifier le fichier ***/etc/nsswitch.conf*** en ajoutant "nis" devant chacun d'eux comme ceci :

    passwd:         compat  nis
    group:          compat  nis
    shadow:         compat nis
    hosts:          files dns nis   

Enfin commme d'habitude, on redémarre :

    systemctl restart rpcbind nis



## ***Lightweight Directory Access Protocol (Ldap)***

Commencons par installer les packages nécessaires à l'installation du serveur openldap :

    apt‐get install slapd ldap‐utils

Nous allons à présent configurer le sldap en deux étapes. Tout d'abord en éxécutant la commande suivante :

    dpkg‐reconfigure slapd

Puis en modifiant le fichier ***/etc/ldap/ldap.conf*** de cette manière :

    BASE     dc=asr,dc=local
    URI      ldap://openldap.mondomaine.local/

### **Nouveau noeud**

Ceci terminé, il faut configurer les fichiers LDIF et DIT.
Créons alors un fichier que nous appellerons *configuration.ldif* et ajoutons dans celui-ci les lignes suivantes :

    dn: cn=config
    changetype: modify
    replace: olcLogLevel
    olcLogLevel: stats

    dn: olcDatabase={1}hdb,cn=config
    changetype: modify
    add: olcDbIndex
    olcDbIndex: uid eq

    add: olcDbIndex
    olcDbIndex: cn eq

    add: olcDbIndex
    olcDbIndex: ou eq

    add: olcDbIndex
    olcDbIndex: dc eq

On enregistre les modifications comme ceci :

    ldapmodify -QY EXTERNAL -H ldapi:/// -f ~/olc-mod1.ldif

Ajoutons maintenant au fichier les Unités d'Organisation. Nous allons créer ici un groupe qui pourra contenir des utilisateurs dont un que l'on va créer aussi. Voici les lignes correspondantes :

    #création du groupe
    dn: ou=Groupe,dc=asr.fr,dc=local
    ou: Groupe
    objectClass: organizationalUnit
    description : Groupe de personnes
    
    #Création de l'utilisateur
    dn: cn=Zekri Raphael,ou=Groupe,dc=asr.fr,dc=local
    objectClass: posixGroup
    givenName: Raphael
    cn: Raphael Zekri
    uid: Raph30
    userPassword: LDAP 

La classe **posixGroup** nous a permis d'ajouter Raphael Zekri au groupe. Nous aurions pu donner d'autres information sur Raphael Zekri comme son numéro par exemple.

On va maintenant prendre en compte cette nouvelle Unité d'Organisation et son utilisateur en tapant les lignes suivantes :

    ldapadd -x -W -D 'cn=admin,dc=asr.fr,dc=com' -H ldap://localhost‐f configuration.ldif
    Enter LDAP Password:
    adding new entry "ou=Groupe,dc=asr.fr,dc=local"
    adding new entry "cn=Zekri Raphael,ou=Groupe,dc=asr.fr,dc=local"

-x : Authentification simple  
-W : Permet de taper le mot de passe du compte de manière intéractive  
-D : Donne le Distinguished Name du compte à connecter 
-H : indique la méthode de connexion

Enfin on vérifie que l'utilisateur a bien intégré le groupe comme ceci :

    ldapsearch -xLLL uid=Raph30














