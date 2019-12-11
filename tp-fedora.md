# TP Fedora

# I.  `systemd`-basic

## 1. First steps
* **S'assurer que `systemd`est PID1**
```bash
 $ pidof systemd
 1006 1
```

## 2. Gestion du temps
* **Déterminer la différence entre Local Time, Universal Time et RTC Time**
-- **Expliquer dans quels cas il peut être pertinent d'utiliser le RTC time**
```bash
$ timedatectl
               Local time: ven. 2019-11-29 11:48:27 CET
           Universal time: ven. 2019-11-29 10:48:27 UTC
                 RTC time: ven. 2019-11-29 10:48:27
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```
 - Le local time correspond à l'heure de la machine
  - Universal Time correspond au temps universel. (En france il faut rajouter 1h à ce temps pour obtenir l'heure local)
  - Le RTC Time correspond à une horloge en temps réel qui est utilisé pour déclancher des evenements car elle continue de tourner même lorsque la machine est éteinte.

* **Changer de timezone pour un fuseau horaire européen**
```bash
$ timedatectl set-timezone Europe/Prague
```
Vérification :
```bash
$ timedatectl
               Local time: ven. 2019-11-29 11:52:58 CET
           Universal time: ven. 2019-11-29 10:52:58 UTC
                 RTC time: ven. 2019-11-29 10:52:58
                Time zone: Europe/Prague (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no

```
* **Désactiver le service lié à la synchronisation du temps**
```bash
$ timedatectl set-ntp false
```
Vérification : 
```bash
$ timedatectl
               Local time: ven. 2019-11-29 11:54:02 CET
           Universal time: ven. 2019-11-29 10:54:02 UTC
                 RTC time: ven. 2019-11-29 10:54:02
                Time zone: Europe/Prague (CET, +0100)
System clock synchronized: yes
              NTP service: inactive
          RTC in local TZ: no

```

## 3. Gestion de noms
 
**Différence entre les trois noms**

Le hostname **static** est le nom d'hôte traditionnel, qui peut être choisi par l'utilisateur, et est stocké dans le fichier `/etc/hostname`. Le nom d'hôte **transient** est un nom d'hôte dynamique maintenu par le kernel, par défaut, la valeur correspond au hostname static (lors de l'initialisation) soit "localhost". Il peut être modifié par DHCP ou mDNS au moment de l'exécution. Le hostname **pretty** est un nom d'hôte UTF8 de forme libre pour présentation à l'utilisateur.

En prod il faut donc privilégier le **hostname static**

## 4. Gestion du réseau (et résolution de noms)

**Afficher les information DHCP**
```bash
$ nmcli con show ens33
[...]
DHCP4.OPTION[1]:                        broadcast_address = 192.168.10.255
DHCP4.OPTION[2]:                        dhcp_lease_time = 1800
DHCP4.OPTION[3]:                        dhcp_rebinding_time = 1575
DHCP4.OPTION[4]:                        dhcp_renewal_time = 900
DHCP4.OPTION[5]:                        dhcp_server_identifier = 192.168.10.254
DHCP4.OPTION[6]:                        domain_name = localdomain
DHCP4.OPTION[7]:                        domain_name_servers = 192.168.10.2
DHCP4.OPTION[8]:                        expiry = 1575035294
DHCP4.OPTION[9]:                        ip_address = 192.168.10.132
DHCP4.OPTION[10]:                       requested_broadcast_address = 1
DHCP4.OPTION[11]:                       requested_dhcp_server_identifier = 1
DHCP4.OPTION[12]:                       requested_domain_name = 1
DHCP4.OPTION[13]:                       requested_domain_name_servers = 1
DHCP4.OPTION[14]:                       requested_domain_search = 1
DHCP4.OPTION[15]:                       requested_host_name = 1
DHCP4.OPTION[16]:                       requested_interface_mtu = 1
DHCP4.OPTION[17]:                       requested_ms_classless_static_routes = 1
DHCP4.OPTION[18]:                       requested_nis_domain = 1
DHCP4.OPTION[19]:                       requested_nis_servers = 1
DHCP4.OPTION[20]:                       requested_ntp_servers = 1
DHCP4.OPTION[21]:                       requested_rfc3442_classless_static_routes = 1
DHCP4.OPTION[22]:                       requested_root_path = 1
DHCP4.OPTION[23]:                       requested_routers = 1
DHCP4.OPTION[24]:                       requested_static_routes = 1
DHCP4.OPTION[25]:                       requested_subnet_mask = 1
DHCP4.OPTION[26]:                       requested_time_offset = 1
DHCP4.OPTION[27]:                       requested_wpad = 1
DHCP4.OPTION[28]:                       routers = 192.168.10.2
DHCP4.OPTION[29]:                       subnet_mask = 255.255.255.0
[...]
```

**Stopper et désactiver le démarrage de `NetworkManager`**
```bash
$ systemctl stop NetworkManager.service
$ systemctl disable NetworkManager.service
```

**Stopper et désactiver le démarrage de `systemd-networkd`**
```bash
$ systemctl start systemd-networkd.service
$ systemctl enable systemd-networkd.service
```
**Éditer une carte réseau**
Information de la carte réseau ens37 avant édition
```bash
$ ip addr show ens37
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:fe:ee:7a brd ff:ff:ff:ff:ff:ff
    inet 192.168.21.131/24 brd 192.168.21.255 scope global dynamic ens37
       valid_lft 1362sec preferred_lft 1362sec
```
Edition du fichier de configuration :
```bash
$ sudo cat /etc/systemd/network/ens37.network
[Match]
Name=ens37

[Network]
Address=10.33.10.10/24
DNS=10.33.10.1
Gateway=10.33.10.254
```
Résultats :
```bash
$ ip a show ens37
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:fe:ee:7a brd ff:ff:ff:ff:ff:ff
    inet 10.33.10.10/24 brd 10.33.10.255 scope global ens37
       valid_lft forever preferred_lft forever
```

**Activer la résiolution de noms pas `systemd-resolved`**
```bash
$ sudo systemctl start systemd-resolved
$ sudo systemctl enable systemd-resolved
$ sudo systemctl status systemd-resolved
● systemd-resolved.service - Network Name Resolution
   Loaded: loaded (/usr/lib/systemd/system/systemd-resolved.service; e>
   Active: active (running) since Fri 2019-11-29 16:22:23 CET; 14s ago
     Docs: man:systemd-resolved.service(8)
           https://www.freedesktop.org/wiki/Software/systemd/resolved
           https://www.freedesktop.org/wiki/Software/systemd/writing-n>
           https://www.freedesktop.org/wiki/Software/systemd/writing-r>
 Main PID: 2215 (systemd-resolve)
   Status: "Processing requests..."
    Tasks: 1 (limit: 2307)
   Memory: 13.1M
      CPU: 549ms
   CGroup: /system.slice/systemd-resolved.service
           └─2215 /usr/lib/systemd/systemd-resolved

nov. 29 16:22:23 faitdora systemd[1]: Starting Network Name Resolution>
nov. 29 16:22:23 faitdora systemd-resolved[2215]: Positive Trust Ancho>
nov. 29 16:22:23 faitdora systemd-resolved[2215]: . IN DS 19036 8 2 49>
nov. 29 16:22:23 faitdora systemd-resolved[2215]: . IN DS 20326 8 2 e0>
nov. 29 16:22:23 faitdora systemd-resolved[2215]: Negative trust ancho>
nov. 29 16:22:23 faitdora systemd-resolved[2215]: Using system hostnam>
nov. 29 16:22:23 faitdora systemd[1]: Started Network Name Resolution.
```

**Vérifier que le serveur dns tourne localement**
```bash 
$ sudo ss -laptn
State        Recv-Q       Send-Q                Local Address:Port               Peer Address:Port
LISTEN       0            128                   127.0.0.53%lo:53                      0.0.0.0:*            users:(("systemd-resolve",pid=2215,fd=19))
LISTEN       0            128                         0.0.0.0:22                      0.0.0.0:*            users:(("sshd",pid=870,fd=5))
LISTEN       0            128                         0.0.0.0:5355                    0.0.0.0:*            users:(("systemd-resolve",pid=2215,fd=13))
```

Résolution de nom avec `dig` :
```bash
$ dig @127.0.0.53 google.fr

; <<>> DiG 9.11.13-RedHat-9.11.13-2.fc31 <<>> @127.0.0.53 google.fr
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58259
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.fr.                     IN      A

;; ANSWER SECTION:
google.fr.              5       IN      A       172.217.19.227

;; Query time: 94 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: ven. nov. 29 16:39:21 CET 2019
;; MSG SIZE  rcvd: 54
``` 

Résolution de nom avec `systemd-resolve` :
```bash
$ systemd-resolve google.com
google.com: 216.58.215.46                      -- link: ens33

-- Information acquired via protocol DNS in 31.3ms.
-- Data is authenticated: no
```
**Activation permanente du serveur local DNS**
Supprimer le fichier actuel `/etc/resolv.conf` :
`sudo rm /etc/resolv.conf`

Mise en place du lien symbolique :
`sudo ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

Résultat : 
``` bash
cat /etc/resolv.conf

# This file is managed by man:systemd-resolved(8). Do not edit.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs must not access this file directly, but only through the
# symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
# replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 127.0.0.53
options edns0
search auvence.co
```
**Modifier la configuration systemd-resolved**
Ajout d'un DNS dans le fichier `/etc/resolv.conf` : `DNS=1.2.3.4`
Restart `systemd-resolved` : `sudo systemctl restart systemd-resolved`
```bash
$ resolvectl
MulticastDNS setting: yes
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
         DNS Servers: 1.2.3.4
Fallback DNS Servers: 1.1.1.1
                      8.8.8.8
                      1.0.0.1
                      8.8.4.4
                      2606:4700:4700::1111
                      2001:4860:4860::8888
                      2606:4700:4700::1001
                      2001:4860:4860::8844
          DNSSEC NTA: 10.in-addr.arpa
```

**Mise en place de DNS over TLS**
La mise en place du DNS over TLS permet de garantir que personne d'autre ne se fasse passer pour le resolver car ce dernier s'authentifie auprès du client à l'aide d'un certificat.
 ```bash
$ sudo cat /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1
#FallbackDNS=1.1.1.1 8.8.8.8 1.0.0.1 8.8.4.4 2606:4700:4700::1111 2001:4860:4860::8888 2606:4700:4700::1001 2001:4860:4860::8844
#Domains=
#LLMNR=yes
#MulticastDNS=yes
DNSOverTLS=yes
#Cache=yes
#DNSStubListener=yes
#ReadEtcHosts=yes
```

**Activer l'utilisation de DNSSEC :**
On ajoute `DNSSEC=force` au fichier `/etc/systemd/resolved.conf` :
```
sudo cat /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1
#FallbackDNS=1.1.1.1 8.8.8.8 1.0.0.1 8.8.4.4 2606:4700:4700::1111 2001:4860:4860::8888 2606:4700:4700::1001 2001:4860:4860::8844
#Domains=
#LLMNR=yes
#MulticastDNS=yes
DNSSEC=force
DNSOverTLS=yes
#Cache=yes
#DNSStubListener=yes
#ReadEtcHosts=yes
```
## 6. Gestion d'unité basique (services)

**L'unité associée au processus `chronyd`**
```bash
$ sudo cat /var/run/chrony/chronyd.pid
2080

$ systemctl status 485
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-12-05 10:12:46 UTC; 2h25min ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 449 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 465 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/>
 Main PID: 485 (chronyd)
    Tasks: 6
   Memory: 38.1M
      CPU: 1min 4.850ms
   CGroup: /system.slice/chronyd.service
           └─485 /usr/sbin/chronyd
           ...
```

# II. Boot & logs
**Générer un graph de la séquence de boot**

```bash
$ systemd-analyze plot > graph.svg
$ cat graphe.svg | grep "sshd.service"
```
<text class="right" x="2734.235" y="4614.000">sshd.service (197ms)


# III. Mécanismes manipulés par systemd
## 1. cgroups

**Identifier le cgroup utilisé par votre session SSH**
```bash
$ ps -e -o pid,cmd,cgroup   
1371 sshd: kelian@pts/0             0::/user.slice/user-1000.slice/session-21.scope
```

**Modifier la RAM dédiée à la session utilisateur**
```bash
$ sudo systemctl set-property user-1000.slice MemoryMax=510M
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/memory.max
534773760

$ sudo systemctl set-property user-1000.slice MemoryMax=512M
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/memory.max
536870912
```

**Vérifier la création du fichier après la commande `systemctl set-property`**

```bash
$ sudo ls /etc/systemd/system.control/user-1000.slice.d/
50-MemoryMax.conf

$ sudo cat /etc/systemd/system.control/user-1000.slice.d/50-MemoryMax.conf
# This is a drop-in unit file extension, created via "systemctl set-property"
# or an equivalent operation. Do not edit.
[Slice]
MemoryMax=536870912
```