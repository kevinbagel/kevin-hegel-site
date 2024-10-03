---
title: "Containerizing home network core with Docker"
layout: post
tags: [routing, switch, vlan, mikrotik, tnsr, netgear, security, vlans]
---

Containerizing home network core with Docker
=====================================

I moved recently and wanted to take the opportunity to containerize some services that I have been running on a bunch of raspberry pi's and other devices scattered around the house.  I am also adding two ubiquiti access points to alleviate congestion issues caused  by a nearby apartment building.

## Network Setup

| VLAN | Description                    | IP (TNSR interface)            |
|------|--------------------------------|      ------                    |
| 1    | NATIVE                         | 192.168.50.1/24                |
| 6    | DMZ                            | 192.168.8.1/22                 |
| 7    | LAN                            | 192.168.0.1/22                 |
| 8    | WORK                           | 192.168.20.1/22                |
| 9    | IOT                            | 192.168.28.1/22                |
| 10   | DN42                           | 192.168.40.1/22                |


Start by configuring tnsr with new VLAN subinterfaces according to the above table. LAN is the dataplane interface name for your internal LAN interface. Mine is called LAN but yours could be TenGigabitEthernet6/0/0 and the subinterface would be TenGigabitEthernet6/0/0.6 for VLAN 6.

The native VLAN 1 is needed because ubiquiti unifi devices require management traffic to go over the native VLAN in order to communicate with the controller. This VLAN will not route to the internet or have DNS.  It may not be needed on your network if everything is tagged or on a tagged interface.

~~~
fw0 tnsr(config)# interface LAN
fw0 tnsr(config-interface)# ip address 192.168.50.1/24
fw0 tnsr(config-interface)# exit

tnsr(config)# interface subif LAN 6
tnsr(config-subif)# dot1q 6
tnsr(config-subif)# exact-match
tnsr(config-subif)# exit

tnsr(config)# interface subif LAN 7
tnsr(config-subif)# dot1q 7
tnsr(config-subif)# exact-match
tnsr(config-subif)# exit

tnsr(config)# interface subif LAN 8
tnsr(config-subif)# dot1q 8
tnsr(config-subif)# exact-match
tnsr(config-subif)# exit

tnsr(config)# interface subif LAN 9
tnsr(config-subif)# dot1q 9
tnsr(config-subif)# exact-match
tnsr(config-subif)# exit

tnsr(config)# interface subif LAN 10
tnsr(config-subif)# dot1q 10
tnsr(config-subif)# exact-match
tnsr(config-subif)# exit

tnsr(config)# interface LAN.6
tnsr(config-interface)# ip address 192.168.8.1/22
tnsr(config-interface)# description DMZ VLAN6
tnsr(config-interface)# ip nat inside
tnsr(config-interface)# enable
tnsr(config-interface)# exit

tnsr(config)# interface LAN.7
tnsr(config-interface)# ip address 192.168.0.1/22
tnsr(config-interface)# description LAN VLAN7
tnsr(config-interface)# ip nat inside
tnsr(config-interface)# enable
tnsr(config-interface)# exit

tnsr(config)# interface LAN.8
tnsr(config-interface)# ip address 192.168.20.1/22
tnsr(config-interface)# description WORK VLAN8
tnsr(config-interface)# ip nat inside
tnsr(config-interface)# enable
tnsr(config-interface)# exit

tnsr(config)# interface LAN.9
tnsr(config-interface)# ip address 192.168.28.1/22
tnsr(config-interface)# description IOT VLAN9
tnsr(config-interface)# ip nat inside
tnsr(config-interface)# enable
tnsr(config-interface)# exit

tnsr(config)# interface LAN.10
tnsr(config-interface)# ip address 192.168.40.1/22
tnsr(config-interface)# description DN42 VLAN10
tnsr(config-interface)# enable
tnsr(config-interface)# exit
~~~

### DHCP

For now the router will serve DHCP for the native VLAN and the IOT VLAN.  Others can be added in the future by adding additional subnets with a corresponding interface.
~~~
tnsr(config)# dhcp4 server
tnsr(config-kea-dhcp4)# description LAN DHCP Defaults
tnsr(config-kea-dhcp4)# interface listen LAN
tnsr(config-kea-dhcp4)# lease lfc-interval 3600
tnsr(config-kea-dhcp4)# option domain-name
tnsr(config-kea-dhcp4-opt)# data lan
tnsr(config-kea-dhcp4-opt)# exit
tnsr(config-kea-dhcp4)# subnet 192.168.50.0/24
tnsr(config-kea-subnet4)# pool 192.168.50.50-192.168.50.150
tnsr(config-kea-subnet4-pool)# exit
tnsr(config-kea-subnet4)# interface LAN
tnsr(config-kea-subnet4-opt)# exit
tnsr(config-kea-dhcp4)# subnet 192.168.28.0/22
tnsr(config-kea-subnet4)# pool 192.168.28.50-192.168.30.150
tnsr(config-kea-subnet4-pool)# exit
tnsr(config-kea-subnet4)# interface LAN.9
tnsr(config-kea-subnet4)# option domain-name-servers
tnsr(config-kea-subnet4-opt)# data 1.1.1.1
tnsr(config-kea-subnet4-opt)# exit
tnsr(config-kea-subnet4)# option routers
tnsr(config-kea-subnet4-opt)# data 192.168.28.1
tnsr(config-kea-subnet4-opt)# exit
tnsr(config-kea-dhcp4)# exit
~~~

### LAN Subnet Design

| VLAN | Static | Docker MACVLAN | DHCP |
|--------|----------------|------|------|
| 1 | 192.168.50.5 - 192.168.0.40 | 192.168.50.160 - 192.168.50.191 (/27) | 192.168.50.50 - 192.168.50.150 |
| 7 | 192.168.0.5 - 192.168.0.100 | 192.168.0.129 - 192.168.0.254 (/25) | 192.168.2.50 - 192.168.3.250 |

## Docker Host Setup

The docker host is an Ubuntu LTS server connected to a trunk port on the 10g switch

### Local Disks

Create a place to store docker files with a RAID5 array

#### Partition

Follow this stackoverflow guide [stackoverflow](https://unix.stackexchange.com/a/320330) mdadm RAID implementation with GPT partitioning - Unix & Linux Stack Exchange

### Interfaces

#### Shim interface
This bridge interface will allow the host to communicate with the docker containers.
/etc/networkd-dispatcher/routable.d/10-macvlan-interfaces.sh
~~~
#! /bin/bash

ip link add mynet-shim link br0 type macvlan mode bridge
ip addr add 192.168.0.254/32 dev mynet-shim
ip link set mynet-shim up
ip route add 192.168.0.128/25 dev mynet-shim
~~~
~~~
chmod +x /etc/networkd-dispatcher/routable.d/10-macvlan-interfaces.sh
~~~
Netplan
~~~yaml
network:
  version: 2

  ethernets:
    eno6np1:
      dhcp4: false
      match:
        name: eno6np1
      dhcp6: no

  bridges:
    br0:
      addresses: [ "192.168.0.5/22" ]
      interfaces: [ vlan7 ]
      nameservers:
        addresses: [ "192.168.0.130" ]
      routes:
        - to: default
          via: 192.168.0.1  

  vlans:
    vlan7:
      id: 7
      link: eno6np1
      accept-ra: no
~~~


### Docker Compose
~~~yaml
services:

  pihole:
    container_name: pihole
    hostname: pihole
    image: pihole/pihole:latest
    env_file: "pihole.env"
    volumes:
      - /data/docker/pihole/pihole_etc_pihole:/etc/pihole/
    cap_add:
      - NET_ADMIN
    networks:
      lanmacvlan:
        ipv4_address: 192.168.0.130
    dns:
      - 1.1.1.1
    restart: always

  unifi-db:
    image: docker.io/mongo:7.0
    container_name: unifi-db
    env_file: "mongodb.env"
    volumes:
      - /data/docker/mongodb/db:/data/db
    networks:
      lanmacvlan:
        ipv4_address: 192.168.0.131
    restart: always

  unifi-network-application:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi-network-application
    env_file: "unifi.env"
    volumes:
      - /data/docker/unifi/data:/config
    networks:
      lanmacvlan:
        ipv4_address: 192.168.0.132
    restart: always

networks:
  lanmacvlan:
    driver: macvlan
    driver_opts: 
      parent: br0
    ipam:
      config:
        - subnet: "192.168.0.0/22"
          gateway: "192.168.0.1"
          ip_range: "192.168.0.128/25"
          aux_addresses:
            hostmacvlan: "192.168.0.254"

  nativemacvlan:
    driver: macvlan
    driver_opts: 
      parent: eno6np1
    ipam:
      config:
        - subnet: "192.168.50.0/24"
          gateway: "192.168.50.1"
          ip_range: "192.168.50.160/27"
          aux_addresses:
            hostnativemacvlan: "192.168.50.190"
~~~


### Database
/data/docker/mongodb/init-mongo.js
~~~
db.getSiblingDB("MONGO_DBNAME").createUser({user: "MONGO_USER", pwd: "MONGO_PASS", roles: [{role: "dbOwner", db: "MONGO_DBNAME"}]});
db.getSiblingDB("MONGO_DBNAME_stat").createUser({user: "MONGO_USER", pwd: "MONGO_PASS", roles: [{role: "dbOwner", db: "MONGO_DBNAME_stat"}]});
~~~

If the init script does not work on the first run of the container you can docker exec into the mongodb container and manually create the initial unifi database and user.  This will only need to be done once.

### .env files
pihole
~~~
TZ=America/Los_Angeles
WEBPASSWORD='passwordhere'
~~~
mangodb
~~~
MONGO_DBNAME=unifi
MONGO_USER=unifi
MONGO_PASS='passwordhere'
~~~
unifi
~~~
PUID=1000
PGID=1000
TZ=America/Los_Angeles
MONGO_USER=unifi
MONGO_PASS='passwordhere'
MONGO_HOST=unifi-db
MONGO_PORT=27017
MONGO_DBNAME=unifi
~~~

## Docker

~~~
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl stop docker
sudo vim /etc/docker/daemon.json
{
  "data-root": "/data/docker/root"
}
sudo systemctl restart docker
~~~

## Mikrotik 10g Mini Switch

If you only need to access one VLAN on the switch you can keep the factory default configuration and uplink the switch to an access port.  Below is an example if you want to carry all vlans.

~~~
/interface bridge
add name=bridge1 vlan-filtering=no
/interface bridge port
add bridge=bridge1 interface=sfp-sfpplus4 frame-types=admit-only-vlan-tagged
add bridge=bridge1 interface=sfp-sfpplus3 pvid=7 frame-types=admit-only-untagged-and-priority-tagged
add bridge=bridge1 interface=sfp-sfpplus2 pvid=7 frame-types=admit-only-untagged-and-priority-tagged
add bridge=bridge1 interface=sfp-sfpplus1 pvid=7 frame-types=admit-only-untagged-and-priority-tagged
add bridge=bridge1 interface=ether1 pvid=7 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge vlan
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=6
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=7
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=8
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=9
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=10
add bridge=bridge1 tagged=sfp-sfpplus4 vlan-ids=99
add bridge=bridge1 tagged=ether1,bridge1 vlan-ids=99
/interface vlan
add interface=bridge1 vlan-id=99 name=MGMT
/ip address
add address=192.168.99.1/24 interface=MGMT

/interface bridge set bridge1 frame-types=admit-only-vlan-tagged
~~~