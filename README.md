
Voici l'explication détaillée du fichier docker-compose.yml.

1. Informations Générales

version: '3.8'
# version: '3.8' : Cela indique la version du format de fichier Docker Compose utilisée. La version 3.8 est compatible avec Docker 18.06.0 et les versions ultérieures. Elle permet d'utiliser des fonctionnalités avancées comme la gestion des réseaux et des volumes.
# 2. Services
# La section services définit les conteneurs que Docker Compose va créer et gérer.

Service : Firewall
  firewall:
    image: frrouting/frr:latest
    container_name: firewall
    privileged: true
    cap_add:
      - NET_ADMIN
    environment:
      - TINI_SUBREAPER=1
    networks:
      dmz_in:
        ipv4_address: 10.0.0.100
      dmz_out:
        ipv4_address: 10.32.0.100
      internal:
        ipv4_address: 10.64.0.253
    volumes:
      - ./frr-firewall:/etc/frr
    command: >
      bash -c '
      set -e && 
      mkdir -p /etc/frr &&
      echo "zebra=yes\nbgpd=no\nospfd=yes" > /etc/frr/daemons &&
      if [ ! -f /etc/frr/frr.conf ]; then
        echo "frr version 8.4\n        frr defaults traditional\n        hostname firewall\n        service integrated-vtysh-config\n        !\n        router ospf\n          ospf router-id 10.0.0.100\n          network 10.0.0.0/11 area 0\n          network 10.32.0.0/11 area 0\n          network 10.64.0.0/12 area 0\n        !\n        line vty\n        !" > /etc/frr/frr.conf;
      fi &&
      touch /etc/frr/vtysh.conf &&
      chown -R frr:frr /etc/frr &&
      /usr/lib/frr/watchfrr -d -F traditional zebra bgpd ospfd staticd &&
      sleep 10 &&
      tail -f /dev/null'

# Explication :

# image: frrouting/frr:latest : Utilise l'image Docker officielle de FRRouting (FRR), un logiciel de routage réseau. latest signifie qu'il prendra la dernière version disponible.

# container_name: firewall : Nom du conteneur dans Docker.

# privileged: true : Donne des privilèges élevés au conteneur, permettant d'accéder à des fonctionnalités systèmes comme la configuration réseau.

# cap_add: - NET_ADMIN : Ajoute la capacité de gérer les interfaces réseau à l'intérieur du conteneur.

# environment: - TINI_SUBREAPER=1 : Configure l'environnement pour gérer correctement les processus enfants dans le conteneur.

networks :

dmz_in, dmz_out, et internal sont des réseaux virtuels définis plus bas.
Chaque réseau a une adresse IP fixe attribuée :
10.0.0.100 pour le réseau dmz_in
10.32.0.100 pour dmz_out
10.64.0.253 pour internal
volumes: - ./frr-firewall:/etc/frr : Monte le dossier local ./frr-firewall dans le conteneur à l'endroit /etc/frr. Cela permet de persister la configuration de FRR.

command :

bash -c '...' : Exécute une série de commandes bash à l'intérieur du conteneur.
set -e : Arrête l'exécution si une commande échoue.
Création des dossiers et fichiers de configuration pour FRR :
mkdir -p /etc/frr : Crée le dossier /etc/frr s'il n'existe pas.
echo "zebra=yes\nbgpd=no\nospfd=yes" > /etc/frr/daemons : Active le service Zebra (gestion des routes), désactive BGP et active OSPF.
Condition :
Si le fichier de config frr.conf n'existe pas, il est créé avec la configuration OSPF.
touch /etc/frr/vtysh.conf : Crée un fichier vide nécessaire pour l'interface en ligne de commande de FRR.
chown -R frr:frr /etc/frr : Change le propriétaire des fichiers pour l'utilisateur frr.
/usr/lib/frr/watchfrr ... : Lance les services FRR (zebra, ospfd, etc.).
sleep 10 && tail -f /dev/null : Attend 10 secondes, puis garde le conteneur actif sans rien faire (utile pour des tests ou débogage).

Service : Router
# Le service router est similaire au firewall, avec quelques différences :
  router:
    image: frrouting/frr:latest
    container_name: router
    privileged: true
    cap_add:
      - NET_ADMIN
    environment:
      - TINI_SUBREAPER=1
    networks:
      dmz_in:
        ipv4_address: 10.0.0.101
      dmz_out:
        ipv4_address: 10.32.0.101
      internal:
        ipv4_address: 10.64.0.254
    volumes:
      - ./frr:/etc/frr
    command: >
      bash -c '
      set -e && 
      mkdir -p /etc/frr &&
      echo "zebra=yes\nbgpd=yes\nospfd=yes" > /etc/frr/daemons &&
      if [ ! -f /etc/frr/frr.conf ]; then
        echo "frr version 7.5\n        frr defaults traditional\n        hostname router\n        service integrated-vtysh-config\n        !\n        router ospf\n          ospf router-id 10.0.0.101\n          network 10.0.0.0/11 area 0\n          network 10.32.0.0/11 area 0\n          network 10.64.0.0/12 area 0\n        !\n        line vty\n        !" > /etc/frr/frr.conf;
      fi &&
      touch /etc/frr/vtysh.conf &&
      chown -R frr:frr /etc/frr &&
      /usr/lib/frr/watchfrr -d -F traditional zebra bgpd ospfd staticd &&
      sleep 10 &&
      tail -f /dev/null'
    healthcheck:
      test: ["CMD", "vtysh", "-c", "show ip ospf neighbor"]
      interval: 30s
      retries: 3
# Différences principales :

# BGP activé : bgpd=yes dans le fichier daemons, ce qui active le protocole de routage BGP en plus de OSPF.
# Vérification de santé (healthcheck) :
#test: ["CMD", "vtysh", "-c", "show ip ospf neighbor"] : Vérifie si le voisin OSPF est actif.
# interval: 30s : Effectue la vérification toutes les 30 secondes.
# retries: 3 : Si l’échec survient trois fois de suite, le conteneur est marqué comme non sain.

3. Services 
Service : DNS
  dns:
    image: sameersbn/bind:latest
    container_name: dns
    privileged: true
    networks:
      dmz_in:
        ipv4_address: 10.0.0.2
    ports:
      - "15353:53/udp"
      - "15353:53/tcp"
    volumes:
      - ./bind:/data/bind
    command: >
      bash -c "
      set -e && export DEBIAN_FRONTEND=noninteractive && 
      apt update && apt install -y --reinstall iproute2 iputils-ping iptables net-tools && 
      if [ ! -f /data/bind/named.conf ]; then
        echo 'options {
          directory \"/var/cache/bind\";
          recursion yes;
          allow-query { any; };
          forwarders { 8.8.8.8; 8.8.4.4; };
        };' > /data/bind/named.conf;
      fi &&
      /sbin/entrypoint.sh"
# Explication :

# image: sameersbn/bind:latest : Utilise l'image Docker pour BIND, un serveur DNS populaire.

#container_name: dns : Nom du conteneur.

# privileged: true : Donne des privilèges élevés pour gérer des configurations réseau.

# networks: dmz_in : Connecté au réseau dmz_in avec l'IP 10.0.0.2.

ports :

15353:53/udp et 15353:53/tcp : Redirige le port DNS classique 53 (UDP/TCP) vers le port 15353 de l'hôte.
volumes: ./bind:/data/bind : Monte le dossier local ./bind dans /data/bind dans le conteneur, pour stocker les fichiers de configuration DNS.

command :

Met à jour les paquets nécessaires comme iproute2 et iputils-ping.
Vérifie si le fichier de configuration DNS named.conf existe. S'il n'existe pas, il en crée un avec :
recursion yes; : Autorise les requêtes récursives (le serveur va chercher les réponses pour les clients).
allow-query { any; }; : Autorise n'importe quel client à faire des requêtes DNS.
forwarders { 8.8.8.8; 8.8.4.4; }; : Utilise les serveurs DNS de Google comme serveurs de transfert si la réponse n'est pas trouvée localement.
/sbin/entrypoint.sh : Lance le service DNS.
Service : DHCP
yaml
Copier
Modifier
  dhcp:
    image: networkboot/dhcpd:latest
    container_name: dhcp
    privileged: true
    networks:
      dmz_in:
        ipv4_address: 10.0.0.3
    ports:
      - "6767:67/udp"
    volumes:
      - ./dhcp:/data
    command: >
      bash -c "
      set -e && export DEBIAN_FRONTEND=noninteractive && 
      apt update && apt install -y --reinstall iproute2 iputils-ping iptables net-tools && 
      if [ ! -f /data/dhcpd.conf ]; then
        echo 'subnet 10.0.0.0 netmask 255.255.255.0 {
          range 10.0.0.100 10.0.0.200;
          option routers 10.0.0.101;
          option domain-name-servers 10.0.0.2;
        }' > /data/dhcpd.conf;
      fi &&
      /entrypoint.sh"

# Explication :

# image: networkboot/dhcpd:latest : Utilise l'image Docker pour un serveur DHCP (Dynamic Host Configuration Protocol).

# container_name: dhcp : Nom du conteneur.

# privileged: true : Nécessaire pour attribuer des adresses IP via DHCP.

# networks: dmz_in : Connecté au réseau dmz_in avec l'IP 10.0.0.3.

# ports: "6767:67/udp" : Le port 67/udp (utilisé par DHCP) est redirigé vers 6767 sur l'hôte.

# volumes: ./dhcp:/data : Monte le dossier local ./dhcp dans le conteneur pour stocker les configurations DHCP.

command :

Met à jour les paquets nécessaires.
Vérifie si le fichier dhcpd.conf existe. S'il n'existe pas, il en crée un avec la configuration suivante :
subnet 10.0.0.0 netmask 255.255.255.0 : Définit le sous-réseau.
range 10.0.0.100 10.0.0.200 : Plage d'adresses IP attribuées par le DHCP.
option routers 10.0.0.101 : Définit la passerelle par défaut (le routeur).
option domain-name-servers 10.0.0.2 : Définit le serveur DNS (le service DNS décrit plus haut).
/entrypoint.sh : Lance le service DHCP.
Service : Squid Proxy
yaml
Copier
Modifier
  squid:
    image: sameersbn/squid:latest
    container_name: squid
    privileged: true
    networks:
      dmz_out:
        ipv4_address: 10.32.0.4
    ports:
      - "3128:3128"
    command: bash -c "set -e && export DEBIAN_FRONTEND=noninteractive && apt update && apt install -y --reinstall iproute2 iputils-ping iptables net-tools && /sbin/entrypoint.sh"

# Explication :

# image: sameersbn/squid:latest : Utilise l'image Docker pour Squid, un serveur proxy.

# container_name: squid : Nom du conteneur.

# privileged: true : Nécessaire pour gérer les configurations réseau.

# networks: dmz_out : Connecté au réseau dmz_out avec l'IP 10.32.0.4.

# ports: "3128:3128" : Redirige le port 3128 (port par défaut pour Squid Proxy) du conteneur vers l'hôte.

command :

Met à jour les paquets nécessaires et lance le service Squid.
Service : Syslog
yaml
Copier
Modifier
  syslog:
    image: balabit/syslog-ng:latest
    container_name: syslog
    privileged: true
    networks:
      dmz_out:
        ipv4_address: 10.32.0.5
    command: bash -c "set -e && export DEBIAN_FRONTEND=noninteractive && apt update && apt install -y --reinstall iproute2 iputils-ping iptables net-tools && syslog-ng -F"

# Explication :

# image: balabit/syslog-ng:latest : Utilise l'image Docker pour syslog-ng, un serveur de gestion des logs.

# container_name: syslog : Nom du conteneur.

# privileged: true : Nécessaire pour gérer les logs système.

# networks: dmz_out : Connecté au réseau dmz_out avec l'IP 10.32.0.5.

command :

Met à jour les paquets nécessaires et lance le service syslog-ng en mode premier plan (-F).
Service : Workstation
yaml
Copier
Modifier
  workstation:
    image: debian:12
    container_name: workstation
    privileged: true
    networks:
      internal:
        ipv4_address: 10.64.0.2
    command: >
      bash -c "
      set -e && export DEBIAN_FRONTEND=noninteractive && 
      apt update && 
      apt install -y --reinstall iproute2 iputils-ping iptables net-tools traceroute dnsutils isc-dhcp-client curl bsdutils netcat-openbsd && 
      ip route del default && ip route add default via 10.64.0.254 && 
      tail -f /dev/null"

# Explication :

# image: debian:12 : Utilise l'image officielle de Debian 12 comme système de base.

# container_name: workstation : Nom du conteneur.

#privileged: true : Donne des privilèges élevés pour des tests réseau.

# networks: internal : Connecté au réseau internal avec l'IP 10.64.0.2.

command :

Installe des outils réseau (ping, traceroute, curl, etc.).
Change la route par défaut pour passer par le routeur interne (10.64.0.254).
tail -f /dev/null : Garde le conteneur actif.
4. Réseaux
yaml
Copier
Modifier
networks:
  dmz_in:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/11
  dmz_out:
    driver: bridge
    ipam:
      config:
        - subnet: 10.32.0.0/11
  internal:
    driver: bridge
    ipam:
      config:
        - subnet: 10.64.0.0/12

# Explication :

# dmz_in, dmz_out, et internal sont des réseaux définis avec des sous-réseaux spécifiques.

# driver: bridge : Utilise le pilote réseau bridge pour isoler les conteneurs dans des sous-réseaux distincts.

# ipam: config :

# subnet : Définit la plage d'adresses IP pour chaque réseau.
