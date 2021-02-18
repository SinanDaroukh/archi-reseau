*Compte-rendu rédigé par Armen Aristakesyan, Florian Bouchut, Sinan Daroukh et Alexis Georges*
# Architecture Sécurisé des Réseaux

- [Architecture Sécurisé des Réseaux](#architecture-sécurisé-des-réseaux)
  * [Préambule](#préambule)
  * [Partie I - Le firewall](#partie-i--le-firewall)
    + [Sécurisation par défaut en entrant depuis internet](#sécurisation-par-défaut-en-entrant-depuis-internet)
    + [PF - Logging](#pf---logging)
    + [NATer la DMZ](#nater-la-dmz)
    + [Configurer le serveur DHCP](#configurer-le-serveur-dhcp)
  * [Partie II - Le VPN](#partie-ii--le-vpn)
    + [Configuration IKEv2](#configuration-ikev2)
    + [Connexion avec un Mac](#connexion-avec-un-mac)
  * [Partie III - CARP / Pfsync](#partie-iii--carp-/-pfsync)
    + [Configuration de PFSYNC](#configuration-de-pfsync)
    + [Load balancing](#load-balancing)
  * [Partie IV - Reverse Proxy](#partie-iv--reverse-proxy)
    + [1 - mode hash [key]](#1---mode-hash-[key])
    + [2 - mode least-states](#2---mode-least-states)
    + [3 - mode loadbalance [key]](#3---mode-loadbalance-[key])
    + [4 - mode random](#4---mode-random)
    + [5 - mode roundrobin](#5---mode-roundrobin)
    + [6 - mode source-hash [key]](#6---mode-source-hash-[key])
  * [Partie V - Proxy sortant](#partie-v--proxy-sortant)
    + [Généralités](#généralités)
    + [Proxy filtrant](#proxy-filtrant)
    + [Proxy cache](#proxy-cache)

## Préambule

1. Installation de la VM OpenBSD avec VirtualBox.

2. Vérification des intérfaces réseaux (1e en mode _pont_ sur `eth0`, la 2e en mode _réseau interne_ sur `dmz`, et la 3e en mode _réseau interne_ sur `lan`).

3. Démarrage de la VM OpenBSD.

Relevé des adresses IP et MAC :

    em0: flags=808843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST,AUTOCONF4> mtu 1500
         lladdr 08:00:27:86:e0:ab
         inet 192.168.0.37 netmask 0xffffff00 broadcast 192.168.0.255
    em1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
         lladdr 08:00:27:1d:54:b7
         inet 10.0.0.1 netmask 0xffffff00 broadcast 10.0.0.255
    em2: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
         lladdr 08:00:27:2a:a1:39
         inet 10.0.1.1 netmask 0xffffff00 broadcast 10.0.1.255

> Par quel fichier de configuration sont configurées les addresses ces 3 interfaces ?
>> Les fichiers hostname.em* dans le dossier /etc/

> Qu'est ce qui détermine la route par défaut utilisée par les paquets sortants de la machine ?
>> La VM est configurée en DHCP sur sa patte de sortie donc c'est le DHCP qui fournit la route par défaut.

## Partie I : Le firewall

### Sécurisation par défaut en entrant depuis internet

Modification du fichier `/etc/pf.conf`, ajout des lignes :

     block in on em0 all
     pass in on em0 proto icmp

Ouverture du port SSH en entrant

     pass in on em0 proto tcp from <clients> to port { 22 }

Puis authorisation de connexion depuis notre PC uniquement (en ligne de commande):

     pfctl -t clients -T add 192.168.0.45

Il faut par la suite recharger le fichier de configuration de PF avec la commande :

     pfctl -f /etc/pf.conf

Pour vérifier les règles il faut taper la commande :

     pfctl -sr

### PF - Logging

Modification de la ligne `block in on em0 all` en `block in log on em0 all` puis ecrire la ligne de commande suivante :

     tcpdump -n -e -ttt -i pflog0

### NATer la DMZ

Modifier le fichier /etc/network/interfaces, ajouter les lignes :

     auto enp0s3
     iface enp0s3 inet static
          address 10.0.0.2/24
          gateway 10.0.0.1

Ne pas oublier de recharger après avoir ajouter l'adresse :

     systemctl restart networking

> Pouvez vous vous connecter en tant que root via ssh depuis le firewall vers le serveur debian ? Pourquoi ?
>> Non, parce que debian est configuré pour laisser passer le ssh que sur l'utilisateur test.

Pour NATer la DMZ il faut taper les commandes : 

     sysctl net.inet.ip.forwarding=1
     echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf

Puis il faut ajouter la ligne suivante dans `/etc/pf.conf` :

     pass out on em0 inet from em1:network to any nat-to (em0)

Finalement il faut taper : 

     pfctl -f /etc/pf.conf

Pour vérifier l'état du NAT il faut taper : 

     pfctl -s state

> Que faut t'il configurer pour que cette machine puisse résoudre des noms DNS ?
>> Il faut faire une résolution DNS avec 8.8.8.8 dans /etc/resolv.conf

### Configurer le serveur DHCP

Modification du fichier `/etc/dhcpd.conf` :

     option deomain-name "my.domain";
     option domain-name-servers 8.8.8.8;

     subnet 10.0.1.0 netmask 255.255.255.0 {
          option routers 10.0.1.1;

          range 10.0.1.2 10.0.1.10;
     }

Pour démarrer le serveur DHCP taper les commandes suivantes :

     rcctl enable dhcpd
     rcctl start dhcpd

## Partie II : Le VPN

### Configuration IKEv2

Création d'une autorité de certification (CA) :

     ikectl ca vpn create

Installation du certificat de l’autorité de certification : 

     ikectl ca cert install

Création d’un certificat pour le serveur VPN :
     
     ikectl ca vpn certificate server1.domain create

Installation du certificat du serveur VPN :
     
     ikectl ca vpn certificate 192.168.0.37 install

Dans le fichier `/etc/iked.conf` ajouter les lignes :

     user 'user' 'password'
     ikev2 'vpn' passive esp \
              from 0.0.0.0/0 to 0.0.0.0 \
              local 192.168.0.37 peer any \
              srcid server1.domain \
              eap "mschap-v2" \
              config address 10.0.2.0/24 \
              config name-server 10.246.200.1 \
              tag "ROADW"

Dans le fichier `/etc/pf.conf` ajouter les lignes :

     pass in log on em0 proto udp from any to (em0) port { isakmp, ipsec-nat-t } tag IKED
     pass in log on em0 proto esp from any to any tag IKED

     pass log on enc0 tagged ROADW
     match out log on em0 inet tagged ROADW nat-to (em0)

Rechargez le fichier avec la commande :

     pfctl -f /etc/pf.conf

Génération du certificat pour le client :

     ikectl ca vpn certificate client1.domain create
     cp /etc/ssl/vpn/client1.domain.crt /etc/iked/certs/
     ikectl ca vpn certificate client1.domain export

> Quelles sont les règles de firewalling à ajouter pour qu'un client puisse se connecter au VPN ?
>> pass in on em0 inet proto udp to port { 500, 4500 }

Elle permet d’autoriser le trafic venant depuis l’extérieur pour le protocole ISAKMP.

> Quels sont les fichiers de clefs de la PKI qui vont être utilisés par le serveur ?
>> Le fichier /etc/ssl/vpn/ca.crt car celui-ci contient la clef publique de l’autorité de certification ce qui permettra au serveur VPN de vérifier la validité du certificat du client.

> Quelles sont les IP utilisées par le VPN ?
>> Il s’agit de la plage d’adresse 10.0.2.0/24.

Afin de démarer le VPN taper les commandes suivants :

     rcctl -f restart iked
     rcctl set iked flags -6

Pour visualiser les connexions VPN taper :

     iked -dv

### Connexion avec un Mac

Pour se connecter au VPN avec un ordinateur sour MacOS :

- Acceder aux paramètres réseaux
- Ajouter un réseau **VPN**
- Choisir **IKEv2**
- Mettre l'IP du serveur `(192.168.0.37)`
- Mettre l'identifiant distant `(server1.domain)`
- Puis cliquer sur **configuration d'authentification**
- Mettre le login `user` et le mot de passe `password`
- Cliquer sur **Connecter**

## Partie III : CARP / Pfsync

Sur la VM OpenBSD `fw1` et `fw1` entrer les commandes suivantes :

     ifconfig em1 10.0.0.x netmask 255.255.255.0
     ifconfig em2 10.0.1.x netmask 255.255.255.0
     ifconfig em3 10.0.10.1 netmask 255.255.255.252

     ifconfig carp1 create
     ifconfig carp0 create
     ifconfig carp2 create

> Attention : **x** est `100` pour `fw1` et `200` pour `fw1`.

Sur la VM `fw1` entrer les commandes suivantes :

     ifconfig carp0 vhid 10 pass root carpdev em0 advskew 1 192.168.1.111 netmask 255.255.255.0
     ifconfig carp1 vhid 11 pass root carpdev em1 advskew 1 10.0.0.1 netmask 255.255.255.0
     ifconfig carp2 vhid 12 pass root carpdev em2 advskew 1 10.0.1.1 netmask 255.255.255.0

Sur la VM `fw2` entrer les commandes suivantes :

     ifconfig carp0 vhid 10 pass root carpdev em0 advskew 100 192.168.1.111 netmask 255.255.255.0
     ifconfig carp1 vhid 11 pass root carpdev em1 advskew 100 10.0.0.1 netmask 255.255.255.0
     ifconfig carp2 vhid 12 pass root carpdev em2 advskew 100 10.0.1.1 netmask 255.255.255.0

Activation de la préemption et le basculement de toutes les interfaces 
     
     sysctl -w net.inet.carp.preempt=1

### Configuration de PFSYNC 

     ifconfig pfsync0 syncdev em3 
     ifconfig pfsync0 up 

     set skip on pfsync0

> Que faut il faire pour que le VPN marche sur l'ip flottante ? Est-ce que le VPN peut gérer le failover ?

Modifier le fichier `iked.conf` pour avoir le contenu suivant :

     user 'android' 'password'
     ikev2 'responder_eap' passive esp \
              from 0.0.0.0/0 to 0.0.0.0 \
              local 192.168.1.111 peer any \
              srcid server1.domain \
              eap "mschap-v2" \
              config address 192.168.2.0/24 \
              config name-server 192.168.1.111 \
              tag "ROADW"

Changer le serveur en `192.168.1.111` dans la config sur le téléphone.

Certains états ne devraient pas être synchronisés, lesquels, pourquoi, et comment s'assurer qu'ils restent locaux à chacun des firewall ?

     Tout ce qui est broadcast et multicast !

Voir le lien suivant : https://sc1.checkpoint.com/documents/R76/CP_R76_ClusterXL_AdminGuide/7288.htm


> Faut t'il modifier la règle de NAT pour les différents réseaux ? Quels sont les conséquences sur les paquets en provenance des sous-réseaux ?
>> Oui il faut adapter chaque règle pour les différents réseaux. Les conséquences sont que chaque réseaux sort du réseaux avec la même adresse IP source.

     pass out on em0 from 10.0.0.0/24 to any nat-to 192.168.1.111

### Load balancing 

Sur `fw1` :

     ifconfig carp0 192.168.1.111 carpdev em0 carpnodes 1:0,2:100 balancing ip
     ifconfig carp1 10.0.0.1 carpdev em1 carpnodes 1:0,2:100 balancing ip
     ifconfig carp2 10.0.1.1 carpdev em2 carpnodes 1:0,2:100 balancing ip

Sur `fw2` :
     
     ifconfig carp0 192.168.1.111 carpdev em0 carpnodes 1:100,2:0 balancing ip
     ifconfig carp1 10.0.0.1 carpdev em1 carpnodes 1:100,2:0 balancing ip
     ifconfig carp2 10.0.1.1 carpdev em2 carpnodes 1:100,2:0 balancing ip

## Partie IV : Reverse Proxy

> Quels sont les fichiers a remodifier sur le clone ? Vérifier que les deux firewall peuvent y accéder.

`/etc/network/interfaces`

     auto enp0s3
     iface enp0s3 inet static
          address 10.0.0.3
          netmask 255.255.255.0
          gateway 10.0.0.1


Tapez les commandes suivantes : 

     systemctl start lighttpd
     echo "<body><h1>Serveur 1 </h1></body>" > /var/www/index.lighttpd.html
     systemctl stop lighttpd
     systemctl start lighttpd

     lighttpd-enable-mod accesslog
     tail -f /var/log/lighttpd/access.log

*Les options suivantes permettent de définir l'algorithme de programmation pour sélectionner un hôte dans le tableau spécifié*

---
### 1 - mode hash [key]

> Balanace les connexions sortantes sur les hôtes actifs en fonction de la clé, de l'adresse IP et du port du relais. Des données supplémentaires peuvent être introduites dans le hachage en examinant les en-têtes HTTP et les variables GET.

### 2 - mode least-states

> Transférer chaque connexion sortante vers l'hôte actif avec les états pf(4) les moins actifs. Ce mode n'est pris en charge que par les redirections.

### 3 - mode loadbalance [key]

> Équilibre les connexions sortantes sur les hôtes actifs en fonction de la clé, de l'adresse IP source du client, ainsi que de l'adresse IP et du port du relais. Ce mode n'est pris en charge que par les relais.

### 4 - mode random

> Distribue les connexions sortantes de façon aléatoire à travers tous les hôtes actifs. Ce mode est pris en charge par des redirections et des relais.

### 5 - mode roundrobin

> Distribue les connexions sortantes à l'aide d'un programmateur de round-robin à travers tous les hôtes actifs. C'est le mode par défaut et il sera utilisé si aucune option n'a été spécifiée. Ce mode est pris en charge par des redirections et des relais.

### 6 - mode source-hash [key]

> Balanace les connexions sortantes sur les hôtes actifs en fonction de la clé et de l'adresse IP source du client. Ce mode est pris en charge par des redirections et des relais.

Dans le fichier `/etc/relayd.conf`, ajouter les lignes suivantes

     rcctl enable relayd
     rcctl start relayd
     rcctl stop relayd

     debian1="10.0.0.2"
     debian2="10.0.0.3"
     table <webhosts> {
          $debian1
          $debian2
     }

     redirect www {
          listen on 192.168.1.53 port 80 interface em0
          forward to <webhosts> check http "/" code 200
          pftag RELAYD
     }
     
 
Dans le fichier `/etc/pf/conf`, rajouter la ligne 

     anchor "relayd/*"

## Partie V : Proxy sortant

### Généralités

### Proxy filtrant

### Proxy cache

