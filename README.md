# DEPRECATED
## This repo is no longer maintained.<br>For a list of current demos, please visit:<br>https://gitlab.com/cumulus-consulting/goldenturtle/<br><br><br>

# Cumulus NetQ Demo

This demo will install Cumulus Linux [NetQ](https://docs.cumulusnetworks.com/display/NETQ) Fabric Validation System using the Cumulus [reference topology](https://github.com/cumulusnetworks/cldemo-vagrant). Please visit the reference topology github page for detailed instructions on using Cumulus Vx with Vagrant.


![Cumulus Reference Topology](https://raw.githubusercontent.com/CumulusNetworks/cldemo-vagrant/master/documentation/cldemo_topology.png)

_Don't want to run it locally? You can also run this demo in [Cumulus In the Cloud](https://cumulusnetworks.com/try-for-free/)_


Table of Contents
=================
* [Prerequisites](#prerequisites)
* [Using Virtualbox](#using-virtualbox)
* [Using Libvirt KVM](#using-libvirtkvm)
* [Using Cumulus in the Cloud](#using-cumulus-in-the-cloud)
* [Running the Demo](#running-the-demo)
    * [EVPN Demo](#evpn-demo)
    * [Docker Swarm   Routing on the Host Demo](#docker-swarm--routing-on-the-host-demo)
* [Troubleshooting + FAQ](#troubleshooting--faq)


Prerequisites (Note that these are for running the simulation locally, not on CITC)
------------------------
* Running this simulation roughly 10G of RAM.
* Internet connectivity is required from the hypervisor. Multiple packages are installed on both the switches and servers when the lab is created.
* Download this repository locally with `git clone https://github.com/CumulusNetworks/cldemo-netq.git` or if you do not have Git installed, [Download the zip file](https://github.com/CumulusNetworks/cldemo-netq/archive/master.zip)
* Download the NetQ Telemetry Server from https://cumulusnetworks.com/downloads/#product=NetQ%20Virtual&hypervisor=Vagrant. You need to be logged into the site to access this.  Choose NetQ 1.4.
* Install [Vagrant](https://releases.hashicorp.com/vagrant/).  Vagrant 2.0.1+ is needed to support VirtualBox 5.2.x
* Install [Virtualbox](https://www.virtualbox.org/wiki/VirtualBox) or [Libvirt+KVM](https://libvirt.org/drvqemu.html) hypervisors.

Using Virtualbox
------------------------
* Add the downloaded box to vagrant via:  
`vagrant box add cumulus-netq-server-1.4.1-ts-amd64-vbox.box --name=cumulus/ts`  
**Note:** You must first manually download the Telemetry Server Vagrant box file from the [Cumulus Networks Download Page](https://cumulusnetworks.com/downloads/#product=NetQ%20Virtual&hypervisor=Vagrant)

Using Libvirt+KVM
------------------------
* Rename `Vagrantfile-kvm` to `Vagrantfile` replacing the existing Vagrantfile that is used for Virtualbox.
* Install the Vagrant mutate plugin with 
`vagrant plugin install vagrant-mutate`
* Add the downloaded box to vagrant via:  
`vagrant box add cumulus-netq-server-1.4.1-ts-amd64-libvirt.box --name=cumulus/ts`  
**Note:** You must first manually download the Telemetry Server Vagrant box file from the [Cumulus Networks Download Page](https://cumulusnetworks.com/downloads/#product=NetQ%20Virtual&hypervisor=Vagrant)

Using Cumulus in the Cloud
------------------------
Request a "Blank Workbench" on [Cumulus in the Cloud](https://cumulusnetworks.com/try-for-free/). When you receive notice that it is provisioned, connect to the *oob-mgmt-server*

Once connected run  
`git clone -b citc https://github.com/CumulusNetworks/cldemo-netq`

This will set the groundwork to copy the rest of the demo to your workbench.

Next  
`cd cldemo-netq`  
`ansible-playbook setup.yml`

After Ansible finishes two new directories are created: 
[evpn](#evpn-demo)  
[docker](#docker-swarm--routing-on-the-host-demo)

You can access either directory and follow the demo instructions below. 

To switch between demos, please _reprovision_ your Cumulus in the Cloud instance.

Running the Demo
------------------------
* The Telemetry Server replaces the oob-mgmt-server in the topology.
* `cd cldemo-netq`
* `vagrant up oob-mgmt-server oob-mgmt-switch`
* `vagrant up` (bringing up the oob-mgmt-server and switch first prevent DHCP issues)
* `vagrant ssh oob-mgmt-server`

### EVPN Demo
The first demo is based on BGP-EVPN with VxLAN Routing.
![EVPN Logical Topology](https://raw.githubusercontent.com/CumulusNetworks/cldemo-netq/master/images/evpn-topology.png)

Server01 and Server03 are in VLAN13, connected via VxLAN (VNI13). 
Server02 and Server04 are in VLAN24, also connected via VxLAN (VNI24).

All four leaf switches are configured with anycast gateways for both VLAN13 and VLAN24.

Server01 has the IP `10.1.3.101`  
Server02 has the IP `10.2.4.102`  
Server03 has the IP `10.1.3.103`  
Server04 has the IP `10.2.4.104`  

To provision this demo, from the **oob-mgmt-server**
* `cd evpn`
* `ansible-playbook run_demo.yml`

After the playbook finishes, you can run a number of tests to view connectivity   
From **server01**:
* `ping 10.1.3.103` (server03)
* `ping 10.2.4.104` (server04)
* `traceroute 10.1.3.103`
* `traceroute 10.2.4.104`

Notice the path from server01 to server03 is direct, while server01 to server04 passes through an extra hop (the gateway at 10.1.3.1)

From **leaf01**:
* `netq check bgp`
```
cumulus@leaf01:mgmt-vrf:~$ netq check bgp
Total Nodes: 10, Failed Nodes: 0, Total Sessions: 16, Failed Sessions: 0
```
* `netq check evpn`
```
cumulus@leaf01:mgmt-vrf:~$ netq check evpn
Total Nodes: 10, Failed Nodes: 0, Total Sessions: 8, Failed Sessions: 0, Total VNIs: 2
```
* `ip route show | netq resolve` to view the routing table with NetQ hostname resolution
* `netq server03 show ip neighbors` to view the ARP table of server01. This should include an entry for `10.1.3.101` (*note:* the MAC address you see may be different from these examples)
```
cumulus@leaf01:mgmt-vrf:~$ netq server03 show ip neighbors
Matching neighbor records are:
IP Address       Hostname         Interface            Mac Address              VRF              Remote Last Changed
---------------- ---------------- -------------------- ------------------------ ---------------- ------ ----------------
10.1.3.101       server03         uplink               44:38:39:00:00:03        default          no     9m:8.903s
10.1.3.13        server03         uplink               44:38:39:00:00:24        default          no     7m:24.415s
10.1.3.14        server03         uplink               32:3e:76:e2:7b:ae        default          no     6m:58.210s
192.168.0.254    server03         eth0                 44:38:39:00:00:5f        default          no     9m:57.746s
```
* `netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty` (this should be the MAC address of server01's `uplink` bond interface)
```
cumulus@leaf01:mgmt-vrf:~$ netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty
Number of Paths: 4
Number of Paths with Errors: 0
Number of Paths with Warnings: 0
Path MTU: 9000

 leaf03 vni: 13 swp52 -- swp3 spine02 swp2 -- swp52 vni: 13 leaf02 bond01 -- uplink server01 uplink 
                swp52 -- swp3 spine02 swp1 -- swp52 vni: 13 leaf01 bond01 -- uplink server01 uplink 
 leaf03 vni: 13 swp51 -- swp3 spine01 swp2 -- swp51 vni: 13 leaf02 bond01 -- uplink server01 uplink 
                swp51 -- swp3 spine01 swp1 -- swp51 vni: 13 leaf01 bond01 -- uplink server01 uplink 

```

On leaf01 add an incomplete VNI configuration
```
net add bgp l2vpn evpn vni 14
net commit
```

And check that BGP is still working as expected
`netq check bgp`

And that 'check evpn' is indicating a problem with the configuration
`netq check evpn`
```
cumulus@leaf01:mgmt-vrf:~$ netq check evpn
Total Nodes: 10, Failed Nodes: 1, Total Sessions: 8 , Failed BGP Sessions: 0, Total VNIs: 3
Hostname          Peer Name         Peer Hostname     Error           Last Changed
----------------- ----------------- ----------------- --------------- -------------------------
leaf01            -                 -                 VNI 14 not in k 3.38594s
                                                      ernel

```

Correct the EVPN misconfiguration
```
net del bgp l2vpn evpn vni 14
net commit
```

Verify that EVPN is functional (this may take up to 30 seconds)
`netq check evpn`

Now, on leaf01 shut down the link to spine01 
`sudo ifdown swp51`

Wait 5-10 seconds for NetQ to export the data.

With NetQ, check BGP again and you should see two failed sessions. 
`netq check bgp`
```
cumulus@leaf01:mgmt-vrf:~$ netq check bgp
Total Nodes: 10, Failed Nodes: 2, Total Sessions: 16 , Failed Sessions: 2, 
Hostname          VRF             Peer Name         Peer Hostname     Reason                                        Last Changed
----------------- --------------- ----------------- ----------------- --------------------------------------------- -------------------------
leaf01            default         swp51             spine01           Link Admin Down                               0.990985s
spine01           default         swp1              leaf01            Hold Timer Expired                            9.489406s
```

Again, run the NetQ traceroute that was run earlier:  
`netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty`
```
cumulus@leaf01:mgmt-vrf:~$ netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty
Number of Paths: 4
Number of Paths with Errors: 1
  Path: 4 Path Incomplete
Number of Paths with Warnings: 0
Path MTU: 9000

 leaf03 vni: 13 swp52 -- swp3 spine02 swp2 -- swp52 vni: 13 leaf02 bond01 -- uplink server01 uplink 
                swp52 -- swp3 spine02 swp1 -- swp52 vni: 13 leaf01 bond01 -- uplink server01 uplink 
 leaf03 vni: 13 swp51 -- swp3 spine01 swp2 -- swp51 vni: 13 leaf02 bond01 -- uplink server01 uplink 
 vni: 13 leaf03 bond01 -- uplink server01 uplink 
```
and notice that there are two paths through spine02 but only a single path through spine01 now.

View the changes to the fabric as a result of shutting down the interface  
`netq spine01 show bgp changes between 1s and 5m`  
*note* the interface state on spine01 may not change because of the virtual environment, but the BGP peer will still fail.

Next, from **spine02**: 
Change the MTU on the interface
```
net add interface swp3 mtu 1500
net commit
```

If we check BGP again, we still have only two failed sessions: leaf01 and spine01.  
`netq check bgp`
```
cumulus@spine02:mgmt-vrf:~$ netq check bgp
Total Nodes: 10, Failed Nodes: 2, Total Sessions: 16 , Failed Sessions: 2, 
Hostname          VRF             Peer Name         Peer Hostname     Reason                                        Last Changed
----------------- --------------- ----------------- ----------------- --------------------------------------------- -------------------------
leaf01            default         swp51             spine01           Link Admin Down                               0.65267s
spine01           default         swp1              leaf01            RA not configured(?)                          0.65404s
```

If we run the traceroute again, we will see the MTU failure in the path 
`netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty`  
*If you need to get the MAC address again use `netq server03 show ip neighbors` and use the entry for `10.1.3.101`*
```
cumulus@spine02:mgmt-vrf:~$ netq trace 44:38:39:00:00:03 vlan 13 from leaf03 pretty
Number of Paths: 4
  Inconsistent path mtu among paths
Number of Paths with Errors: 1
  Path: 4 Path Incomplete
Number of Paths with Warnings: 2
  Path: 1 MTU mismatch between leaf03:swp52 (9216) and spine02:swp3 (1500)
  Path: 2 MTU mismatch between leaf03:swp52 (9216) and spine02:swp3 (1500)
Path MTU: 1500

 leaf03 vni: 13 swp52 -- swp3 spine02 swp2 -- swp52 vni: 13 leaf02 bond01 -- uplink server01 uplink 
                swp52 -- swp3 spine02 swp1 -- swp52 vni: 13 leaf01 bond01 -- uplink server01 uplink 
 leaf03 vni: 13 swp51 -- swp3 spine01 swp2 -- swp51 vni: 13 leaf02 bond01 -- uplink server01 uplink 
 vni: 13 leaf03 bond01 -- uplink server01 uplink 
```

Again, you can see the changes with  
`netq spine02 show interface changes between 1s and 5m`
```
cumulus@spine02:mgmt-vrf:~$ netq spine02 show interface changes between 1s and 5m

Matching link records:
Hostname          Interface                 Type             State      VRF             Details                             DB State   Last Changed
----------------- ------------------------- ---------------- ---------- --------------- ----------------------------------- ---------- -------------------------
spine02           swp3                      swp              up         default         MTU:1500                            Add        1m:21.883s
```

### Docker Swarm + Routing on the Host Demo
The second demo relies on [Cumulus Host Pack](https://cumulusnetworks.com/products/host-pack/) to install FRR and NetQ on each server. The servers speak eBGP unnumbered to the local top of rack switches.

If any existing demos have already been provisioned, the lab must be rebuilt. On your VM host run  
`vagrant destroy -f leaf01 leaf02 leaf03 leaf04 spine01 spine02 server01 server02 server03 server04`   
then recreate a fresh environment with  
`vagrant up`

![Docker + Routing on the Host](https://raw.githubusercontent.com/CumulusNetworks/cldemo-vagrant/master/documentation/cldemo_topology.png)

Just as described in the Reference Topology diagram, each server is configured with a /32 loopback IP and BGP ASN.


After BGP is configured on the hosts, [Docker CE](https://www.docker.com/community-edition) is automatically installed and [Docker Swarm](https://docs.docker.com/engine/swarm/) is configured.


Within Docker Swarm, server01 acts as the _Swarm Manager_ while server02, server03 and server04 act as _Swarm Workers_.

Swarm deploys an Apache service. The service creates four apache containers which are deployed on various nodes in the swarm.

**To provision this demo**, from the oob-mgmt-server 
* `cd docker`
* `ansible-playbook run_demo.yml`

From **server01**: 
* `sudo docker node ls` to verify that all four servers are in the swarm
* `sudo docker service ps apache_web` to see the four apache containers deployed

Log into to FRR on **server01**: 
* `sudo docker exec -it cumulus-roh /usr/bin/vtysh` to attach to the FRR process
* `show ip bgp summary` to view the BGP peers
* `show ip bgp` to view the BGP routes
* `CTL P CLT Q` to log out of FRR container and keep it running 


Now use NetQ to verify Docker settings. On **spine01**:  
 `netq show docker summary` to see the nodes with docker installed and brief information about them  
 `netq show docker swarm cluster` to see the members of the cluster  
 `netq show docker swarm node` to view the the members of the cluster and their roles  
 `netq show docker container network host` to view the containers with host networking, which shows the Quagga containers  
 `netq show docker service` to view the currently running services (only apache_web in this demo)  
 `netq show docker service name apache_web connectivity` to view all of the containers named `apache_web` and their connectivity  
 `netq leaf02 show docker container adjacent interface swp1` to see the containers that are adjacent to the leaf02, swp1 interface (the containers deployed on server01)

Now, connect to **server03** and shut down the link to leaf04  
`sudo ifdown eth2`

Wait 10-20 seconds for NetQ to export the data and look at the impact of removing leaf03 from the network  
`netq leaf03 show impact docker service apache_web`  
The red indicates that removing leaf03 from service would bring down server03 and the attached containers
```
cumulus@server03:~$ netq leaf03 show impact docker service apache_web
apache_web -- apache_web.2.i46cc2s6249020xw4z8ct0id3 -- server02:eth1:eth1 -- swp2:swp2:leaf01
                                                     -- server02:eth2:eth2 -- swp2:swp2:leaf02
           -- apache_web.3.qwgwu4a5isy9cj1z5snx9ui8n -- server03:eth1:eth1 -- swp1:swp1:leaf03
           -- apache_web.4.ifxyjh86gbz2ycjk6k2qkbsh3 -- server04:eth1:eth1 -- swp2:swp2:leaf03
                                                     -- server04:eth2:eth2 -- swp2:swp2:leaf04
           -- apache_web.1.vxwdzbrrhynol1qdx9lpiiroz -- server01:eth1:eth1 -- swp1:swp1:leaf01
                                                     -- server01:eth2:eth2 -- swp1:swp1:leaf02
```

Now, still on **server03**, run the Docker "hello world" example to create and destroy a container.  
`sudo docker run --name test hello-world` 

And view the changes to the container environment  
`netq server03 show docker container changes`

You will see `apache_web` from the Docker Swarm, `cumulus-roh` the routing on the host container and `test` the container we just created and destroyed. 
```
cumulus@server03:~$ netq server03 show docker container changes

Matching container records:
Container Name       Hostname          Container IP         IP Masq  Network Na Service Name    DBState    Last Changed
                                                                     me
-------------------- ----------------- -------------------- -------- ---------- --------------- ---------- -------------------------
test                 server03          172.17.0.2           True     bridge                     Del        4.503862s
test                 server03          172.17.0.2           True     bridge                     Add        11.463860s
apache_web.3.qwgwu4a server03          10.255.0.8           False    ingress    apache_web      Add        4m:33.722s
5isy9cj1z5snx9ui8n
cumulus-roh          server03          10.0.0.33/32,        False    host                       Add        5m:33.697s
                                       192.168.0.33/24
```

To view changes to Docker Swarm we can change the number of nodes `apache_web` is running on.  
From **server01** run:  
`sudo docker service scale apache_web=2` 

This will change the environment from four apache_web containers to two.

View the updated cluster with `netq show docker service` and notice that only 2 replicas are running.
```
cumulus@server01:~$ netq show docker service

Matching service records:
Service Name    Manager    Cluster    Mode       Replicas                           Running
--------------- ---------- ---------- ---------- ---------------------------------- ----------
apache_web      server01   default    Replicated 2                                  2

```

Next, scale the swarm up to 5 containers. Still on **server01** run:  
`sudo docker service scale apache_web=5`

Wait up to 30 seconds and see the cluster change 
```
cumulus@server01:~$ netq show docker service

Matching service records:
Service Name    Manager    Cluster    Mode       Replicas                           Running
--------------- ---------- ---------- ---------- ---------------------------------- ----------
apache_web      server01   default    Replicated 5                                  5
cumulus@server01:~$ 
```

NetQ also allows us to see the changes to the specific service (note: the specific servers listed here may be different in your environment, but three "Add" entries should exist)

`netq show docker container service apache_web changes between 1s and 5m`
```
cumulus@server01:~$ netq show docker container service apache_web changes between 1s and 5m

Matching container records:
Container Name       Hostname          Container IP         IP Masq  Network Na Service Name    DBState    Last Changed
                                                                     me
-------------------- ----------------- -------------------- -------- ---------- --------------- ---------- -------------------------
apache_web.4.nw4lp75 server03          10.255.0.12          False    ingress    apache_web      Add        21.959066s
xi9n541vsxcnxm8co2
apache_web.5.cr5pvdi server03          10.255.0.13          False    ingress    apache_web      Add        22.957091s
qh4cyw492fmu73kg54
apache_web.3.ixklc4c server04          10.255.0.11          False    ingress    apache_web      Add        23.538899s
9wlcvuhiqlphhskao4
```

Going further back in time we can also see when the cluster was scaled down from four to two:
```
cumulus@server01:~$ netq show docker container service apache_web changes between 1s and 5m

Matching container records:
Container Name       Hostname          Container IP         IP Masq  Network Na Service Name    DBState    Last Changed
                                                                     me
-------------------- ----------------- -------------------- -------- ---------- --------------- ---------- -------------------------
<snip>
apache_web.4.ifxyjh8 server04          10.255.0.7           False    ingress    apache_web      Del        1m:58.385s
6gbz2ycjk6k2qkbsh3
apache_web.3.qwgwu4a server03          10.255.0.10          False    ingress    apache_web      Del        1m:58.920s
5isy9cj1z5snx9ui8n

```

Finally, you can view the service in the past when only two instances were running.  You will likely need a different value than the example below using 3m ago.  How many minutes ago was it when you scaled back to two apache_web services?

`netq show docker container service apache_web around <time>`
```
cumulus@server01:~$ netq show docker container service apache_web around 3m

Matching container records:
Container Name       Hostname          Container IP         IP Masq  Network Na Service Name    Up Time
                                                                     me
-------------------- ----------------- -------------------- -------- ---------- --------------- -------------------------
apache_web.1.w3ipcm5 server02          10.255.0.7           False    ingress    apache_web      0:06:10
gl2k8adb3elid34hm0
apache_web.2.mlh6as1 server01          10.255.0.8           False    ingress    apache_web      0:06:09
jpradrkccldqqwem99
cumulus@server01:~$ 
```


Troubleshooting + FAQ
-------
* The `Vagrantfile` expects the telemetry server to be named `cumulus/ts`. If you get the following error
```The box 'cumulus/ts' could not be found or
could not be accessed in the remote catalog. If this is a private
box on HashiCorp's Atlas, please verify you're logged in via
`vagrant login`. Also, please double-check the name. The expanded
URL and error message are shown below:

URL: ["https://atlas.hashicorp.com/cumulus/ts"]
Error: The requested URL returned error: 404 Not Found
```
Please ensure you have the telemetry server downloaded and installed in Vagrant. Use `vagrant box list` to see the current Vagrant box images you have installed.
* `vagrant ssh` fails to network devices - This is expected, as each network device connects through the `oob-mgmt-server`. Use `vagrant ssh oob-mgmt-server` then ssh to the specific network device.
* If you log into a switch and are prompted for the password for the `vagrant` user, issue the command `su - cumulus` to change to the cumulus user on the oob-mgmt-server
* The Docker demo fails on server01 with an  error similar to the following
```TASK [Deploy Apache Containers] ************************************************
Wednesday 06 September 2017  03:03:56 +0000 (0:00:00.567)       0:00:44.092 ***
fatal: [server01]: FAILED! => {"changed": true, "cmd": ["docker", "service", "create", "--name", "apache_web", "--replicas", "3", "--publish", "8080:80", "php:5.6-apache"], "delta": "0:00:02.673790", "end": "2017-09-06 03:03:58.934894", "failed": true, "rc": 1, "start": "2017-09-06 03:03:56.261104", "stderr": "Error response from daemon: rpc error: code = 3 desc = port '8080' is already in use by service 'apache_web' (vviesw72piif37ip8wplod2dn) as an ingress port", "stdout": "", "stdout_lines": [], "warnings": []}
```
The Docker playbook can only be run once without reprovisioning the environment. The error can be ignored. If you need to rebuild the environment, please `vagrant destroy` and then `vagrant up`
