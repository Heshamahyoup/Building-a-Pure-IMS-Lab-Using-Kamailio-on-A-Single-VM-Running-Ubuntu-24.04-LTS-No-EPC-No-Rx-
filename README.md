
# Building a Modern Pure IMS Lab Using Kamailio IMS on A Single VM Running Ubuntu 25.04 LTS  (No EPC, No Rx interface)

## Introduction
In this tutorial we will deploy kamailio IMS on Ubuntu 25.04 LTS, This step-by-step guide to implements a standalone IMS core network conforming to 3GPP standards on a single VM. Unlike traditional mobile network labs that include the Evolved Packet Core (EPC),this configuration focuses exclusively on the IMS service layer, providing Session Initiation Protocol (SIP) based multimedia services and diameter without the dependency on packet core network elements.

The implementation is initially based on the [IMS-Kamailio-Tutorial](https://github.com/anabelen-garcia/IMS-Kamailio-Tutorial#single-vm-ims-setup-with-kamailio-and-fhoss-for-academic-purposes).

## Software Used
In this tutorial, the software we used listed below along with the function.
|Software|Function|
|----|----|
|[kamailio](https://www.kamailio.org/w/)|P/I/S-CSCF|
|[RTPengine](https://github.com/sipwise/rtpengine)|Proxy for RTP media|
|[FHoSS](https://github.com/herlesupreeth/FHoSS)|HSS|
|[bind9](https://gitlab.isc.org/isc-projects/bind9)|DNS Server|
|[PJSUA](https://gitlab.isc.org/isc-projects/bind9)|User Agent|
|[Java7](https://www.oracle.com/java/technologies/javase/javase7-archive-downloads.html)|Prerequisite for FHoSS|

## Topology on a Single Host
Because all components run on the same machine, we will use the loopback addressing in order to separate network elements logically.
![IMS Topology]([./images/Standalon-IMS-Topology.jpg](https://github.com/Heshamahyoup/Building-a-Pure-IMS-Lab-Using-Kamailio-on-A-Single-VM-Running-Ubuntu-24.04-LTS-No-EPC-No-Rx-/blob/main/images/Standalon-IMS-Topology.jpg))


|Component|IP/Port/|Host Name|
|----|----|----|
|P-CSCF|127.0.0.33:5060|pcscf1.domain.imsprovider.org|
|I-CSCF|127.0.0.55:4060|icscf.domain.imsprovider.org|
|S-CSCF|127.0.0.11:6060|scscf1.domain.imsprovider.org|
|HSS|127.0.0.77:3868|hss.domain.imsprovider.org|
|DNS server|127.0.0.66:53|
|RTPengine|127.0.0.88:2223|
|User a|127.0.0.111:5566|
|User b|127.0.0.222:5577|


Createing and persist the loopback addresses 

In a single VM P-CSCF, I-CSCF, S-CSCF, HSS, and DNS ,all run on same machine. in order to simulate separated network nodes, separated DNS records, and independent SIP entities,different loopback IPs are needed.

You can acheive this by using systemd as below:
* Create script
```
sudo nano /usr/local/bin/ims-loopbacks.sh
```
And add:
```
#!/bin/bash

ip address add 127.0.0.11/8 dev lo 2>/dev/null || true
ip address add 127.0.0.33/8 dev lo 2>/dev/null || true
ip address add 127.0.0.55/8 dev lo 2>/dev/null || true
ip address add 127.0.0.66/8 dev lo 2>/dev/null || true
ip address add 127.0.0.77/8 dev lo 2>/dev/null || true
ip address add 127.0.0.88/8 dev lo 2>/dev/null || true
ip address add 127.0.0.111/8 dev lo 2>/dev/null || true
ip address add 127.0.0.222/8 dev lo 2>/dev/null || true
```
Then make it executable.

```
sudo chmod +x /usr/local/bin/ims-loopbacks.sh
```
* Create Service:
```
sudo nano /etc/systemd/system/ims-loopbacks.service
```
Add this to the service.
```
[Unit]
Description=IMS Loopback Addresses
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ims-loopbacks.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Then enable the service.
```
sudo systemctl daemon-reload
sudo systemctl enable ims-loopbacks
sudo systemctl start ims-loopbacks

```
To veriviy:
```
:~#ip addr show lo

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.11/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.33/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.55/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.66/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.77/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.88/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.111/8 scope host secondary lo
       valid_lft forever preferred_lft forever
    inet 127.0.0.222/8 scope host secondary lo
       valid_lft forever preferred_lft forever
```

You can also do it by using netplan file.
## Packages  installatio

First, we need to update Ubuntu APT software repositories (package sources).
```
:~# sudo cat /etc/apt/sources.list
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble main restricted
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble-updates main restricted
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble universe
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble-updates universe
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble multiverse
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble-updates multiverse
deb https://nova.clouds.archive.ubuntu.com/ubuntu/ noble-backports main restricted universe multiverse
deb https://security.ubuntu.com/ubuntu noble-security main restricted
deb https://security.ubuntu.com/ubuntu noble-security universe
deb https://security.ubuntu.com/ubuntu noble-security multiverse
```
Then install following initial packages.
```
sudo apt update && apt upgrade -y && apt install -y mysql-server tcpdump screen ntp ntpdate git-core dkms gcc flex bison libmysqlclient-dev make libssl-dev libcurl4-openssl-dev libxml2-dev libpcre3-dev bash-completion g++ autoconf libmnl-dev libsctp-dev ipsec-tools libradcli-dev libradcli4
```
<br>

#### Downloading, Compiling and Installing kamailio

You can checkout the kamailio branch that will serve as the basis for all the IMS CSCF nodes.
```
:~# mkdir -p /usr/local/src/ 
:~# cd /usr/local/src/ 
:~# git clone https://github.com/herlesupreeth/kamailio 
:~# cd kamailio 
:~# git checkout -b 5.7 origin/5.7
```
You can also cloning kamilio from their repository as below.
```
:~# git clone https://github.com/kamailio/kamailio.git 
```
In this tutorial, I used branch 5.7.

```
:/usr/local/src/kamailio# git branch -a
  5.6
* 5.7
  master
  remotes/origin/3.1
  remotes/origin/3.2
  remotes/origin/3.3
  remotes/origin/4.0
  remotes/origin/4.1
  remotes/origin/4.2
  remotes/origin/4.3
  ...
  ...
  ...

```

After cloning the branch you need,generate build config files.
``` 
:~# cd /usr/local/src/kamailio 
:~# make cfg
```

Edit and modify ```/usr/local/src/kamailio/src/modules.lst```  file to include all modules to be compiled.

```

# this file is autogenerated by make modules-cfg

# the list of sub-directories with modules
modules_dirs:=modules

# the list of module groups to compile
cfg_group_include=

# the list of extra modules to compile
include_modules= cdp cdp_avp db_mysql dialplan ims_auth ims_charging ims_dialog ims_diameter_server ims_icscf ims_ipsec_pcscf ims_isc ims_ocs ims_qos ims_registrar_pcscf ims_registrar_scscf ims_usrloc_pcscf ims_usrloc_scscf outbound presence presence_conference presence_dialoginfo presence_mwi presence_profile presence_reginfo presence_xml pua pua_bla pua_dialoginfo pua_reginfo pua_rpc pua_usrloc pua_xmpp sctp tls utils xcap_client xcap_server xmlops xmlrpc


# the list of static modules
static_modules=

# the list of modules to skip from compile list
skip_modules=

# the list of modules to exclude from compile list
exclude_modules= acc_json acc_radius app_java app_lua app_lua_sr app_mono app_perl app_python app_python3 app_python3s app_ruby app_ruby_proc auth_ephemeral auth_identity auth_radius cdp cdp_avp cnxcc cplc crypto db2_ldap db_berkeley db_cassandra db_mongodb db_mysql db_oracle db_perlvdb db_postgres db_redis db_sqlite db_unixodbc dialplan dnssec erlang evapi geoip geoip2 gzcompress h350 http_async_client http_client ims_auth ims_charging ims_dialog ims_diameter_server ims_icscf ims_ipsec_pcscf ims_isc ims_ocs ims_qos ims_registrar_pcscf ims_registrar_scscf ims_usrloc_pcscf ims_usrloc_scscf jansson janssonrpcc json jsonrpcc jwt kafka kazoo lcr ldap log_systemd lost lwsc memcached misc_radius mqtt nats ndb_cassandra ndb_mongodb ndb_redis nsq osp outbound peering phonenum presence presence_conference presence_dialoginfo presence_mwi presence_profile presence_reginfo presence_xml pua pua_bla pua_dialoginfo pua_json pua_reginfo pua_rpc pua_usrloc pua_xmpp rabbitmq regex rls rtp_media_server ruxc sctp secsipid secsipid_proc slack snmpstats stirshaken systemdops tls tls_wolfssl tlsa topos_redis utils uuid websocket xcap_client xcap_server xhttp_pi xmlops xmlrpc xmpp $(skip_modules)

modules_all= $(filter-out modules/CVS,$(wildcard modules/*))
modules_noinc= $(filter-out $(addprefix modules/, $(exclude_modules) $(static_modules)), $(modules_all))
modules= $(filter-out $(modules_noinc), $(addprefix modules/, $(include_modules) )) $(modules_noinc)
modules_configured:=1
```

Compile and install Kamailio
```
:~# cd /usr/local/src/kamailio 
:~# export RADCLI=1 
:~# make Q=0 all | tee make_all.txt
:~# make install | tee make_install.txt 
:~# ldconfig
```

The binaries and executable scripts are installed in:
```
:~# ls -lh /usr/local/sbin
total 8.9M
-rwxr-xr-x 1 root root 8.8M May  7 21:09 kamailio
-rwxr-xr-x 1 root root  70K May  7 21:09 kamcmd
-rwxr-xr-x 1 root root  64K May  7 21:10 kamctl
-rwxr-xr-x 1 root root  11K May  7 21:10 kamdbctl
```
Modify ```/usr/local/etc/kamailio/kamctlrc``` so that the following domain (domain.imsprovider.org) and dbengine are specified:
```
:~# cat /usr/local/etc/kamailio/kamctlrc
## The Kamailio configuration file for the control tools.
##

## the SIP domain
 SIP_DOMAIN=domain.imsprovider.org

## If you want to setup a database with kamdbctl, you must at least specify
## this parameter.
 DBENGINE=MYSQL

```
<br>

The configuration files are installed at: ```/usr/local/etc/kamailio```

## Configuration files for PCSCF, SCSCF and ICSCF

```
cd
git clone https://github.com/herlesupreeth/Kamailio_IMS_Config
cd Kamailio_IMS_Config/
git checkout 5.3
cp -r kamailio_icscf /etc
cp -r kamailio_pcscf /etc
cp -r kamailio_scscf /etc
```
<br>
Rename the config files to suit our specific CSCF names ( SCSCF1 and PCSCF1).

```
cd /etc/
mv kamailio_scscf/kamailio_scscf.cfg kamailio_scscf/kamailio_scscf1.cfg
mv kamailio_scscf/scscf.xml kamailio_scscf/scscf1.xml
mv kamailio_scscf/scscf.cfg kamailio_scscf/scscf1.cfg
mv kamailio_pcscf/kamailio_pcscf.cfg kamailio_pcscf/kamailio_pcscf1.cfg
mv kamailio_pcscf/pcscf.xml kamailio_pcscf/pcscf1.xml
mv kamailio_pcscf/pcscf.cfg kamailio_pcscf/pcscf1.cfg
```

Start Modification SCSCF1 files.
**kamailio_scscf1.cfg**
```
:/etc# diff kamailio_scscf.BACKUP/kamailio_scscf.cfg kamailio_scscf/kamailio_scscf1.cfg
35c35
< include_file "scscf.cfg"
---
> include_file "scscf1.cfg"
48c48
< listen=tcp:127.0.0.1:6060
---
> listen=tcp:127.0.0.11:6060
67c67
< dns_try_ipv6=on
---
> dns_try_ipv6=off
101c101
< system.service = "Serving-CSCF" desc "Function of this server"
---
> system.service = "Serving-CSCF1" desc "Function of this server1"
195c195
< modparam("jsonrpcs", "fifo_name", "/var/run/kamailio_scscf/kamailio_rpc.fifo")
---
> modparam("jsonrpcs", "fifo_name", "/run/kamailio_scscf1/kamailio_rpc.fifo")
197c197
< modparam("jsonrpcs", "dgram_socket", "/var/run/kamailio_scscf/kamailio_rpc.sock")
---
> modparam("jsonrpcs", "dgram_socket", "/run/kamailio_scscf1/kamailio_rpc.sock")
200c200
< modparam("ctl", "binrpc", "unix:/var/run/kamailio_scscf/kamailio_ctl")
---
> modparam("ctl", "binrpc", "unix:/run/kamailio_scscf1/kamailio_ctl")
242c242
< modparam("cdp","config_file","/etc/kamailio_scscf/scscf.xml")
---
> modparam("cdp","config_file","/etc/kamailio_scscf/scscf1.xml")
```


**scscf1.xml**
```
:/etc# diff kamailio_scscf.BACKUP/scscf.xml kamailio_scscf/scscf1.xml
3,4c3,4
<       FQDN="scscf.ims.mnc001.mcc001.3gppnetwork.org"
<       Realm="ims.mnc001.mcc001.3gppnetwork.org"
---
>       FQDN="scscf1.domain.imsprovider.org"
>       Realm="domain.imsprovider.org"
17c17
<       <Peer FQDN="hss.ims.mnc001.mcc001.3gppnetwork.org" Realm="ims.mnc001.mcc001.3gppnetwork.org" port="3868"/>
---
>       <Peer FQDN="hss.domain.imsprovider.org" Realm="domain.imsprovider.org" port="3868"/>
19c19
<       <Acceptor port="3870" bind="10.4.128.21"/>
---
>       <Acceptor port="3870" bind="127.0.0.11"/>
35c35
<       <DefaultRoute FQDN="hss.ims.mnc001.mcc001.3gppnetwork.org" metric="10"/>
---
>       <DefaultRoute FQDN="hss.domain.imsprovider.org" metric="10"/>
```
**scscf1.cfg**
```
:/etc# diff kamailio_scscf.BACKUP/scscf.cfg kamailio_scscf/scscf1.cfg
2c2
< listen=udp:10.4.128.21:6060
---
> listen=udp:127.0.0.11:6060
5c5
< listen=tcp:10.4.128.21:6060
---
> listen=tcp:127.0.0.11:6060
10,14c10,14
< #!define NETWORKNAME "ims.mnc001.mcc001.3gppnetwork.org"
< #!define NETWORKNAME_ESC "ims\.mnc001\.mcc001\.3gppnetwork\.org"
< #!define HOSTNAME "scscf.ims.mnc001.mcc001.3gppnetwork.org"
< #!define HOSTNAME_ESC "scscf\.ims\.mnc001\.mcc001\.3gppnetwork\.org"
< #!define URI "sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060"
---
> #!define NETWORKNAME "domain.imsprovider.org"
> #!define NETWORKNAME_ESC "domain\.imsprovider\.org"
> #!define HOSTNAME "scscf1.domain.imsprovider.org"
> #!define HOSTNAME_ESC "scscf1\.domain\.imsprovider\.org"
> #!define URI "sip:scscf1.domain.imsprovider.org:6060"
16,17c16,17
< #!subst "/NETWORKNAME/ims.mnc001.mcc001.3gppnetwork.org/"
< #!subst "/HSS_REALM/ims.mnc001.mcc001.3gppnetwork.org/"
---
> #!subst "/NETWORKNAME/domain.imsprovider.org/"
> #!subst "/HSS_REALM/domain.imsprovider.org/"
19c19
< alias=scscf.ims.mnc001.mcc001.3gppnetwork.org
---
> alias="scscf1.domain.imsprovider.org"
22c22,23
< #!define ENUM_SUFFIX "ims.mnc001.mcc001.3gppnetwork.org."
---
> #!define ENUM_SUFFIX "domain.imsprovider.org."
> ##!define ENUM_SUFFIX "ims.mnc001.mcc001.3gppnetwork.org."
29c30
< #!define DB_URL "mysql://scscf:heslo@127.0.0.1/scscf"
---
> #!define DB_URL "mysql://scscf1:heslo@127.0.0.1/scscf1"
```
Start Modifiying PCSCF1 files.

**kamailio_pcscf1.cfg**
```
:/etc# diff kamailio_pcscf.BACKUP/kamailio_pcscf.cfg kamailio_pcscf/kamailio_pcscf1.cfg
12c12
< import_file "pcscf.cfg"
---
> import_file "pcscf1.cfg"
48c48
< listen=tcp:127.0.0.1:5060
---
> listen=tcp:127.0.0.33:5060
136c136
< udp_mtu = 1300
---
> udp_mtu = 0
143c143
< system.service = "Proxy-CSCF" desc "Function of this server"
---
> system.service = "Proxy-CSCF1" desc "Function of this server1"
156a157
> loadmodule "outbound"
179d179
<
264c264
< modparam("jsonrpcs", "fifo_name", "/var/run/kamailio_pcscf/kamailio_rpc.fifo")
---
> modparam("jsonrpcs", "fifo_name", "/var/run/kamailio_pcscf1/kamailio_rpc.fifo")
266c266
< modparam("jsonrpcs", "dgram_socket", "/var/run/kamailio_pcscf/kamailio_rpc.sock")
---
> modparam("jsonrpcs", "dgram_socket", "/var/run/kamailio_pcscf1/kamailio_rpc.sock")
311c311
< modparam("rtimer", "timer", "name=NATPING;interval=5;mode=1;")
---
> modparam("rtimer", "timer", "name=NATPING;interval=300;mode=1;")
331c331
< modparam("rr", "add_username", 1)
---
> modparam("rr", "add_username", 0)
349c349
< modparam("rtpengine", "rtpengine_sock", "1 == udp:localhost:2223")
---
> modparam("rtpengine", "rtpengine_sock", "1 == udp:127.0.0.88:2223")
357c357
< modparam("ctl", "binrpc", "unix:/var/run/kamailio_pcscf/kamailio_ctl")
---
> modparam("ctl", "binrpc", "unix:/var/run/kamailio_pcscf1/kamailio_ctl")
387c387
< modparam("ims_usrloc_pcscf", "db_url", DB_URL)
---
> modparam("ims_usrloc_pcscf", "db_url", "DB_URL")
389c389
< modparam("ims_usrloc_pcscf", "db_mode", 0)
---
> modparam("ims_usrloc_pcscf", "db_mode", 0);
421c421
< modparam("cdp","config_file","/etc/kamailio_pcscf/pcscf.xml")
---
> modparam("cdp","config_file","/etc/kamailio_pcscf/pcscf1.xml")
456c456
< modparam("ims_dialog", "db_url", DB_URL)
---
> modparam("ims_dialog", "db_url", "DB_URL")
931c931
<                       ipsec_destroy_by_contact("location", "$uac_req(ruri);$var(alias)", "$(uac_req(ouri){uri.host})", "$(uac_req(ouri){uri.port})");
---
> #                     ipsec_destroy_by_contact("location", "$uac_req(ruri);$var(alias)", "$(uac_req(ouri){uri.host})", "$(uac_req(ouri){uri.port})");
```
**pcscf1.xml**
```
:/etc# diff kamailio_pcscf.BACKUP/pcscf.xml kamailio_pcscf/pcscf1.xml
3,4c3,4
<       FQDN="pcscf.ims.mnc001.mcc001.3gppnetwork.org"
<       Realm="ims.mnc001.mcc001.3gppnetwork.org"
---
>       FQDN="pcscf1.domain.imsprovider.org"
>       Realm="domain.imsprovider.org"
17c17
<       <Peer FQDN="pcrf.epc.mnc001.mcc001.3gppnetwork.org" Realm="epc.mnc001.mcc001.3gppnetwork.org" port="3868"/>
---
>       <Peer FQDN="pcrf.domain.imsprovider.org" Realm="epc.domain.imsprovider.org" port="3868"/>
19c19
<       <Acceptor port="3871" bind="10.4.128.21"/>
---
>       <Acceptor port="3871" bind="127.0.0.33"/>
24c24
<       <DefaultRoute FQDN="pcrf.epc.mnc001.mcc001.3gppnetwork.org" metric="10"/>
---
>       <DefaultRoute FQDN="pcrf.domain.imsprovider.org" metric="10"/>
```

**pcscf1.cfg**

```
:/etc# diff kamailio_pcscf.BACKUP/pcscf.cfg kamailio_pcscf/pcscf1.cfg
4c4,5
< listen=udp:10.4.128.21:5060
---
> listen=udp:127.0.0.33:5060
> #listen=udp:10.4.128.21:5060
8c9
< listen=tcp:10.4.128.21:5060
---
> listen=tcp:127.0.0.33:5060
15c16
< #!define IPSEC_LISTEN_ADDR "10.4.128.21"
---
> #!define IPSEC_LISTEN_ADDR "127.0.0.33"
21c22
< #!define RX_AF_SIGNALING_IP "10.4.128.21"
---
> #!define RX_AF_SIGNALING_IP "127.0.0.33"
25c26,27
< alias=pcscf.ims.mnc001.mcc001.3gppnetwork.org
---
> alias="pcscf1.domain.imsprovider.org"
> #alias=pcscf.ims.mnc001.mcc001.3gppnetwork.org
30c32
< #!define PCSCF_URL "sip:pcscf.ims.mnc001.mcc001.3gppnetwork.org:5060"
---
> #!define PCSCF_URL "sip:pcscf1.domain.imsprovider.org:5060"
34,36c36,38
< #!subst "/NETWORKNAME/ims.mnc001.mcc001.3gppnetwork.org/"
< #!subst "/HOSTNAME/pcscf.ims.mnc001.mcc001.3gppnetwork.org/"
< #!subst "/PCRF_REALM/epc.mnc001.mcc001.3gppnetwork.org/"
---
> #!subst "/NETWORKNAME/domain.imsprovider.org/"
> #!subst "/HOSTNAME/pcscf1.domain.imsprovider.org/"
> #!subst "/PCRF_REALM/epc.domain.imsprovider.org/"
47c49
< #!define DB_URL "mysql://pcscf:heslo@127.0.0.1/pcscf"
---
> #!define DB_URL "mysql://pcscf1:heslo@127.0.0.1/pcscf1""
50c52
< #!define SQLOPS_DBURL "pcscf=>mysql://pcscf:heslo@127.0.0.1/pcscf"
---
> #!define SQLOPS_DBURL "pcscf=>mysql://pcscf1:heslo@127.0.0.1/pcscf1"
108c110
< #!define FORCE_RTPRELAY
---
> ##!define FORCE_RTPRELAY
113,115c115,117
< #!define WITH_RX
< #!define WITH_RX_REG
< #!define WITH_RX_CALL
---
> ##!define WITH_RX
> ##!define WITH_RX_REG
> ##!define WITH_RX_CALL
```
<br>

Start Modification SCSCF1 files.

**kamailio_icscf.cfg**
```
:/etc# diff kamailio_icscf.BACKUP/kamailio_icscf.cfg kamailio_icscf/kamailio_icscf.cfg
31c31,32
< listen=tcp:127.0.0.1:4060
---
> listen=tcp:127.0.0.55:4060
> #listen=tcp:127.0.0.1:4060
150c151
< modparam("jsonrpcs", "fifo_name", "/var/run/kamailio_icscf/kamailio_rpc.fifo")
---
> modparam("jsonrpcs", "fifo_name", "/run/kamailio_icscf/kamailio_rpc.fifo")
152c153
< modparam("jsonrpcs", "dgram_socket", "/var/run/kamailio_icscf/kamailio_rpc.sock")
---
> modparam("jsonrpcs", "dgram_socket", "/run/kamailio_icscf/kamailio_rpc.sock")
196c197
< modparam("ctl", "binrpc", "unix:/var/run/kamailio_icscf/kamailio_ctl")
---
> modparam("ctl", "binrpc", "unix:/run/kamailio_icscf/kamailio_ctl")
```

**icscf.xml**

```
:/etc# diff kamailio_icscf.BACKUP/icscf.xml kamailio_icscf/icscf.xml
3,4c3,4
<       FQDN="icscf.ims.mnc001.mcc001.3gppnetwork.org"
<       Realm="ims.mnc001.mcc001.3gppnetwork.org"
---
>       FQDN="icscf.domain.imsprovider.org"
>       Realm="domain.imsprovider.org"
18c18
<       <Peer FQDN="hss.ims.mnc001.mcc001.3gppnetwork.org" Realm="ims.mnc001.mcc001.3gppnetwork.org" port="3868"/>
---
>       <Peer FQDN="hss.domain.imsprovider.org" Realm="domain.imsprovider.org" port="3868"/>
20c20
<       <Acceptor port="3869" bind="10.4.128.21"/>
---
>       <Acceptor port="3869" bind="127.0.0.55"/>
33c33
<       <DefaultRoute FQDN="hss.ims.mnc001.mcc001.3gppnetwork.org" metric="10"/>
---
>       <DefaultRoute FQDN="hss.domain.imsprovider.org" metric="10"/>
```

**icscf.cfg**

```
:/etc# diff kamailio_icscf.BACKUP/icscf.cfg kamailio_icscf/icscf.cfg
2c2
< listen=udp:10.4.128.21:4060
---
> listen=udp:127.0.0.55:4060
5c5
< listen=tcp:10.4.128.21:4060
---
> listen=tcp:127.0.0.55:4060
10c10
< alias=ims.mnc001.mcc001.3gppnetwork.org
---
> alias="icscf.domain.imsprovider.org"
12,13c12,13
< #!define NETWORKNAME "ims.mnc001.mcc001.3gppnetwork.org"
< #!define HOSTNAME "icscf.ims.mnc001.mcc001.3gppnetwork.org"
---
> #!define NETWORKNAME "domain.imsprovider.org"
> #!define HOSTNAME "icscf.domain.imsprovider.org"
15,16c15,16
< #!subst "/NETWORKNAME/ims.mnc001.mcc001.3gppnetwork.org/"
< #!subst "/HSS_REALM/ims.mnc001.mcc001.3gppnetwork.org/"
---
> #!subst "/NETWORKNAME/domain.imsprovider.org/"
> #!subst "/HSS_REALM/domain.imsprovider.org/"
18c18
< #!define ENUM_SUFFIX "ims.mnc001.mcc001.3gppnetwork.org."
---
> #!define ENUM_SUFFIX "domain.imsprovider.org."
```
<br>

### Creating PCSCF, SCSCF and ICSCF databases

create the databases for the CSCF nodes (in our case, PCSCF1, SCSCF1 and ICSCF), first run mysql and then execute the below.
```
create database scscf1;
create database pcscf1;
create database icscf;
```
Create the tables of each database, pressing <Enter> each time root pass is asked:

```
cd /usr/local/src/kamailio/utils/kamctl/mysql
mysql -u root -p pcscf1 < standard-create.sql
mysql -u root -p pcscf1 < presence-create.sql
mysql -u root -p pcscf1 < ims_usrloc_pcscf-create.sql
mysql -u root -p pcscf1 < ims_dialog-create.sql
mysql -u root -p scscf1 < standard-create.sql
mysql -u root -p scscf1 < presence-create.sql
mysql -u root -p scscf1 < ims_usrloc_scscf-create.sql
mysql -u root -p scscf1 < ims_dialog-create.sql
mysql -u root -p scscf1 < ims_charging-create.sql
cd /usr/local/src/kamailio/misc/examples/ims/icscf/
mysql -u root -p icscf < icscf.sql
```
**Note:** modify the sql commands based on mysql version you are using.

Next, declare the users that will be able to modify the databases. Typ ```cd``` and ```mysql```. In the current mysql version (mysql Ver 8.0.45), create the users first.

```
-- Create users
CREATE USER 'scscf1'@'localhost' IDENTIFIED BY 'heslo';
CREATE USER 'pcscf1'@'localhost' IDENTIFIED BY 'heslo';
CREATE USER 'icscf'@'localhost' IDENTIFIED BY 'heslo';
CREATE USER 'provisioning'@'localhost' IDENTIFIED BY 'provi';

CREATE USER 'scscf1'@'%' IDENTIFIED BY 'heslo';
CREATE USER 'pcscf1'@'%' IDENTIFIED BY 'heslo';
CREATE USER 'icscf'@'%' IDENTIFIED BY 'heslo';
CREATE USER 'provisioning'@'%' IDENTIFIED BY 'provi';
```

Then grant privileges

```
-- Grant permissions
GRANT DELETE, INSERT, SELECT, UPDATE ON scscf1.* TO 'scscf1'@'localhost';
GRANT DELETE, INSERT, SELECT, UPDATE ON pcscf1.* TO 'pcscf1'@'localhost';
GRANT DELETE, INSERT, SELECT, UPDATE ON icscf.* TO 'icscf'@'localhost';
GRANT DELETE, INSERT, SELECT, UPDATE ON icscf.* TO 'provisioning'@'localhost';

-- Remote/full privileges
GRANT ALL PRIVILEGES ON scscf1.* TO 'scscf1'@'%';
GRANT ALL PRIVILEGES ON pcscf1.* TO 'pcscf1'@'%';
GRANT ALL PRIVILEGES ON icscf.* TO 'icscf'@'%';
GRANT ALL PRIVILEGES ON icscf.* TO 'provisioning'@'%';

FLUSH PRIVILEGES;
```
Now we need to fire some configuration in ICSCF database, add the IMS domain name as trusted, a reference to SCSCF1 and two of that SCSCF1's capabilities, with values 0 and 1:

```
USE icscf;
INSERT INTO nds_trusted_domains VALUES (1, 'domain.imsprovider.org');
INSERT INTO s_cscf VALUES (1,'SCSCF Number 1','sip:scscf1.domain.imsprovider.org');
INSERT INTO s_cscf_capabilities VALUES (1,1,0),(2,1,1);
```
To verify:
```
mysql> show tables;
+---------------------+
| Tables_in_icscf     |
+---------------------+
| nds_trusted_domains |
| s_cscf              |
| s_cscf_capabilities |
+---------------------+
3 rows in set (0.01 sec)

mysql> select * from nds_trusted_domains;
+----+------------------------+
| id | trusted_domain         |
+----+------------------------+
|  1 | domain.imsprovider.org |
+----+------------------------+
1 row in set (0.00 sec)

mysql> select * from s_cscf;
+----+---------------+-----------------------------------+
| id | name          | s_cscf_uri                        |
+----+---------------+-----------------------------------+
|  1 | SCSCF Number1 | sip:scscf1.domain.imsprovider.org |
+----+---------------+-----------------------------------+
1 row in set (0.00 sec)

mysql> select * from s_cscf_capabilities;
+----+-----------+------------+
| id | id_s_cscf | capability |
+----+-----------+------------+
|  1 |         1 |          0 |
|  2 |         1 |          1 |
+----+-----------+------------+
2 rows in set (0.00 sec)
```

## DNS server Installation and configuration
install bind9 DNS server by: 
```sudo apt install bind9```
After finishing installation ,create zones folder and domain.imsprovider.org zone file.

```
sudo mkdir -p /etc/bind/zones
nano /etc/bind/zones/domain.imsprovider.org
```
And paste the data below to the zone file.
```
$ORIGIN domain.imsprovider.org.
$TTL    1W
@       1D IN SOA       localhost. root.localhost. (
                        1       ; serial
                        3H      ; refresh
                        15M     ; retry
                        1W      ; expire
                        1D )    ; minimum
        1D IN NS        ns
ns      1D IN A 127.0.0.66


pcscf1  1D IN A 127.0.0.33
_sip._udp.pcscf1        1D SRV 0 0 5060 pcscf1
_sip._tcp.pcscf1        1D SRV 0 0 5060 pcscf1

icscf   1D IN A 127.0.0.55
_sip._udp       1D SRV 0 0 4060 icscf
_sip._tcp       1D SRV 0 0 4060 icscf

scscf1  1D IN A 127.0.0.11
_sip._udp.scscf1        1D SRV 0 0 6060 scscf1
_sip._tcp.scscf1        1D SRV 0 0 6060 scscf1

hss     1D IN A 127.0.0.77
```
Edit the following bind9 configuration files and modify it accordingly:
```:~# nano /etc/bind/named.conf.local```

```

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "domain.imsprovider.org" {
        type master;
        file "/etc/bind/zones/domain.imsprovider.org";
};
```
Edit and modify the below file contents.
```:~# nano /etc/bind/named.conf.options```

```
options {
  directory "/var/cache/bind";

  // If there is a firewall between you and nameservers you want
  // to talk to, you may need to fix the firewall to allow multiple
  // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

  // If your ISP provided one or more IP addresses for stable
  // nameservers, you probably want to use them as forwarders.
  // Uncomment the following block, and insert the addresses replacing
  // the all-0's placeholder.

  // forwarders {
  //    0.0.0.0;
  // };

        forwarders {
        // Include one line terminated in ; with the IP address of
        // each specific DNS server you want to add.
        };

  //========================================================================
  // If BIND logs error messages about the root key being expired,
  // you will need to update your keys.  See https://www.isc.org/bind-keys
  //========================================================================
  //dnssec-validation auto;
        dnssec-validation no;
        allow-query { any; };
        auth-nxdomain no; # conform to RFC1035
        listen-on { 127.0.0.66; };

  //listen-on-v6 { any; };
};
```

Force this host use the installed DNS server as its nameserver by creating file ```/etc/netplan/60-dns.yaml``` with these contents:
```
network:
  ethernets:
    enp0s3:
      nameservers:
        search: [domain.imsprovider.org]
        addresses:
          - 127.0.0.66
  version: 2
```
Apply the configuration by executing this command : ```sudo netplan apply```

Then restart the service.
```
systemctl restart systemd-resolved.service
systemctl restart bind9
```
Verify the DNS config:


```
:~# dig pcscf1.domain.imsprovider.org

; <<>> DiG 9.18.39-0ubuntu0.24.04.5-Ubuntu <<>> pcscf1.domain.imsprovider.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62411
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 3f3d4aa4e9e22bc4010000006a19f1bbab0625bbb07cad13 (good)
;; QUESTION SECTION:
;pcscf1.domain.imsprovider.org. IN      A

;; ANSWER SECTION:
pcscf1.domain.imsprovider.org. 86400 IN A       127.0.0.33

;; Query time: 0 msec
;; SERVER: 127.0.0.66#53(127.0.0.66) (UDP)
;; WHEN: Fri May 29 20:06:19 UTC 2026
;; MSG SIZE  rcvd: 102
```
```
:~# dig scscf1.domain.imsprovider.org

; <<>> DiG 9.18.39-0ubuntu0.24.04.5-Ubuntu <<>> scscf1.domain.imsprovider.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11381
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 16f58e6ba570e52c010000006a19f1c47e49e40adbaa5ae6 (good)
;; QUESTION SECTION:
;scscf1.domain.imsprovider.org. IN      A

;; ANSWER SECTION:
scscf1.domain.imsprovider.org. 86400 IN A       127.0.0.11

;; Query time: 1 msec
;; SERVER: 127.0.0.66#53(127.0.0.66) (UDP)
;; WHEN: Fri May 29 20:06:28 UTC 2026
;; MSG SIZE  rcvd: 102
```
```
:~# dig icscf.domain.imsprovider.org

; <<>> DiG 9.18.39-0ubuntu0.24.04.5-Ubuntu <<>> icscf.domain.imsprovider.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21964
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7c46f14ffede543d010000006a19f1ce4feeb909eae58944 (good)
;; QUESTION SECTION:
;icscf.domain.imsprovider.org.  IN      A

;; ANSWER SECTION:
icscf.domain.imsprovider.org. 86400 IN  A       127.0.0.55

;; Query time: 0 msec
;; SERVER: 127.0.0.66#53(127.0.0.66) (UDP)
;; WHEN: Fri May 29 20:06:38 UTC 2026
;; MSG SIZE  rcvd: 101
```
```
:~# dig hss.domain.imsprovider.org

; <<>> DiG 9.18.39-0ubuntu0.24.04.5-Ubuntu <<>> hss.domain.imsprovider.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3950
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 08a3262963728651010000006a19f1d6b38229b2781ced23 (good)
;; QUESTION SECTION:
;hss.domain.imsprovider.org.    IN      A

;; ANSWER SECTION:
hss.domain.imsprovider.org. 86400 IN    A       127.0.0.77

;; Query time: 1 msec
;; SERVER: 127.0.0.66#53(127.0.0.66) (UDP)
;; WHEN: Fri May 29 20:06:46 UTC 2026
;; MSG SIZE  rcvd:
```
## RTPEngine Installation and Configuration

```
cd
export DEB_BUILD_PROFILES="pkg.ngcp-rtpengine.nobcg729"
sudo apt install dpkg-dev
sudo git clone https://github.com/sipwise/rtpengine
cd rtpengine/
git checkout mr26.1.0.0
```
Run ```dpkg-checkbuilddeps``` in the same directory and this command checks for dependencies and give you a list of dependencies which are missing in the system. 

Then install these reported packages.
```
apt install debhelper default-libmysqlclient-dev gperf iptables-dev libavcodec-dev libavfilter-dev libavformat-dev libavutil-dev libbencode-perl libcrypt-openssl-rsa-perl libcrypt-rijndael-perl libdigest-crc-perl libdigest-hmac-perl libevent-dev libhiredis-dev libio-multiplex-perl libio-socket-inet6-perl libiptc-dev libjson-glib-dev libnet-interface-perl libpcap0.8-dev libsocket6-perl libspandsp-dev libswresample-dev libsystemd-dev libxmlrpc-core-c3-dev markdown dkms module-assistant keyutils libnfsidmap2 libtirpc1 nfs-common rpcbind
```
After installing dependencies run the below command again and verify that no dependencies are left out
```
dpkg-checkbuilddeps
```
Now, build and install rtpengine packages:
```
:~/rtpengine# dpkg-buildpackage -uc -us
:~/rtpengine# cd ..
:~# dpkg -i *.deb
```
Create file ```/etc/rtpengine/rtpengine.conf``` taking ```/etc/rtpengine/rtpengine.sample.conf``` as a
starting point (```cp /etc/rtpengine/rtpengine.sample.conf /etc/rtpengine/rtpengine.conf```) and
modify it in the manner specified by this diff command:

```
:~$ diff /etc/rtpengine/rtpengine.sample.conf /etc/rtpengine/rtpengine.conf
13c13
< # interface = 123.234.345.456
---
>  interface = 127.0.0.88
23c23
< listen-ng = localhost:2223
---
> listen-ng = 127.0.0.88:2223
221c221
< interface = 10.15.20.121
---
> interface = 127.0.0.88
```
Run this command so that file ```/etc/rtpengine/rtpengine-recording.conf``` is created:
```
cp /etc/rtpengine/rtpengine-recording.sample.conf /etc/rtpengine/rtpengine-recording.conf
```
Create directory ```/var/spool/rtpengine```:
```
mkdir /var/spool/rtpengine
```
Edit file ```/etc/default/ngcp-rtpengine-daemon``` so that it includes ```RUN_RTPENGINE=yes``` and file
```/etc/default/ngcp-rtpengine-recording-daemon.dpkg-new``` so that it includes ```RUN_RTPENGINE_RECORDING=yes```.

Now, install rpcbind.
```
:/etc/rtpengine# apt --fix-broken install
:/etc/rtpengine# apt install rpcbind
```

Restart the following services
```
systemctl restart ngcp-rtpengine-daemon.service ngcp-rtpengine-recording-daemon.service ngcp-rtpengine-recording-nfs-mount.service 
systemctl enable ngcp-rtpengine-daemon.service ngcp-rtpengine-recording-daemon.service ngcp-rtpengine-recording-nfs-mount.service 
```
Stop and disable  depricated rtpproxy.
```
systemctl stop rtpproxy 
systemctl disable rtpproxy 
systemctl mask rtpproxy
```
Verify the rtpengine

```
:~# ss -tuln | grep 2223
udp   UNCONN 0      0                             127.0.0.88:2223       0.0.0.0:*
```
Check the status of RTPengine services.
```
sudo systemctl status ngcp-rtpengine-daemon.service ngcp-rtpengine-recording-daemon.service
```
## Running I-CSCF, P-CSCF, and S-CSCF as separated nodes
Stop the default kamailio SIP server.
```
systemctl stop kamailio 
systemctl disable kamailio 
systemctl mask kamailio
```
Once we launsh the P/I/S-CSCF nodes, we need all traffic coming from SCSCF1, PCSCF1 and ICSCF has its IP address changed adequately (127.0.0.33 in case of PCSCF1, 127.0.0.55 in case of ICSCF, and 127.0.0.11 in case of SCSCF1), in this way we will be able to distinguish the packets/messages sent from/to CSCF nodes in captured traces. 

The orginal [IMS Kamailio Tutorial](https://github.com/anabelen-garcia/IMS-Kamailio-Tutorial#single-vm-ims-setup-with-kamailio-and-fhoss-for-academic-purposes) we follow in this tutorial, use the cgroup on Ubuntu 18 operating system to launch the CSCF processes each inside its cgroup.
Because we are using Ubuntu 24.04 LTS ,we cannot use cgroup (which is deprecated in Ubuntu 24.04 LTS).

Instead of using cgroups and net_cls classids, we leverage Linux system users and nftables user ID  matching to achieve source NAT for each CSCF node.

First, create dedicated system users for each CSCF node.
```
sudo useradd -r -s /usr/sbin/nologin kamailio-pcscf
sudo useradd -r -s /usr/sbin/nologin kamailio-scscf
sudo useradd -r -s /usr/sbin/nologin kamailio-icscf
```
When a process of CSCF node runs under a specific user, all its network packets are tagged with that user's UID. nftables can match packets based on skuid (socket user ID).

Then, create a new nftables table named ```ims``` for IMS traffic by this command.
```
sudo nft add table ip ims
```
Create Postrouting Chain for Source NAT in the NAT table.
```
sudo nft add chain ip ims postrouting '{ type nat hook postrouting priority 100; }'
```
Now, you need to add source NAT rules for each CSCF user as below.
```
sudo nft add rule ip ims postrouting meta skuid "kamailio-pcscf" ip daddr 127.0.0.0/8 snat to 127.0.0.33
sudo nft add rule ip ims postrouting meta skuid "kamailio-scscf" ip daddr 127.0.0.0/8 snat to 127.0.0.11
sudo nft add rule ip ims postrouting meta skuid "kamailio-icscf" ip daddr 127.0.0.0/8 snat to 127.0.0.55
```
Without this steps, all traffic would appear to come from loopback IP 127.0.0.1, making captured traces impossible to distinguish between CSCF nodes.
To verify nftables ruleset
```
sudo nft list ruleset
```
The expected result.
```
table ip ims {
        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                meta skuid 996 ip daddr 127.0.0.0/8 snat to 127.0.0.33
                meta skuid 995 ip daddr 127.0.0.0/8 snat to 127.0.0.11
                meta skuid 994 ip daddr 127.0.0.0/8 snat to 127.0.0.55
        }
}
```
<br>

###### Making IMS CSCF Services Persistent with Systemd

This section explains how to create persistent systemd service files for each IMS node (P-CSCF, S-CSCF, I-CSCF) with proper user isolation. These configurations ensure your IMS services start automatically on boot and run under the dedicated system users created earlier.

Instead of manually starting Kamailio processes with cgexec or running them in the background, we create proper systemd service units.

Configure tmpfiles.d for runtime directories very early in the boot process, before any services start.

**For SCSCF1**
```
sudo nano /etc/tmpfiles.d/kamailio_scscf1.conf
```
add:
```
d /run/kamailio_scscf1 0755 root root -
```

**For PCSCF1**
```
sudo nano /etc/tmpfiles.d/kamailio_pcscf1.conf
```
add:
```
d /run/kamailio_pcscf1 0755 root root -
```
**For ICSCF**
```
sudo nano /etc/tmpfiles.d/kamailio_icscf.conf
```
add:
```
d /run/kamailio_icscf 0755 root root -
```

Now create the systemd services that will run under these dedicated users.

```sudo nano /etc/systemd/system/kamailio-pcscf1.service```

Add this to the file.
```
[Unit]
Description=Kamailio P-CSCF1
After=network.target mysql.service

[Service]
Type=forking
User=kamailio-pcscf
Group=kamailio-pcscf

RuntimeDirectory=kamailio kamailio_pcscf1
RuntimeDirectoryMode=0755

PIDFile=/run/kamailio_pcscf1/kamailio_pcscf1.pid

kamailio-pcscf:kamailio-pcscf /run/kamailio_pcscf1

ExecStart=/usr/local/sbin/kamailio \
    -f /etc/kamailio_pcscf/kamailio_pcscf1.cfg \
    -P /run/kamailio_pcscf1/kamailio_pcscf1.pid
ExecReload=/bin/kill -HUP $MAINPID
Restart=always


[Install]
WantedBy=multi-user.target
```

```sudo nano /etc/systemd/system/kamailio-scscf1.service```

```
[Unit]
Description=Kamailio S-CSCF1
After=network.target mysql.service

[Service]
Type=forking
User=kamailio-scscf
Group=kamailio-scscf

RuntimeDirectory=kamailio kamailio_scscf1
RuntimeDirectoryMode=0755

PIDFile=/run/kamailio_scscf1/kamailio_scscf1.pid

ExecStart=/usr/local/sbin/kamailio \
    -f /etc/kamailio_scscf/kamailio_scscf1.cfg \
    -P /run/kamailio_scscf1/kamailio_scscf1.pid

ExecReload=/bin/kill -HUP $MAINPID
Restart=always

[Install]
WantedBy=multi-user.target
```


```sudo nano /etc/systemd/system/kamailio-icscf.service```

```
[Unit]
Description=Kamailio I-CSCF
After=network.target mysql.service

[Service]
Type=forking
User=kamailio-icscf
Group=kamailio-icscf

RuntimeDirectory=kamailio kamailio_icscf
RuntimeDirectoryMode=0755

PIDFile=/run/kamailio_icscf/kamailio_icscf.pid

ExecStart=/usr/local/sbin/kamailio \
    -f /etc/kamailio_icscf/kamailio_icscf.cfg \
    -P /run/kamailio_icscf/kamailio_icscf.pid

ExecReload=/bin/kill -HUP $MAINPID
Restart=always

[Install]
WantedBy=multi-user.target
```
<br>
Finally, enable and start IMS nodes Services

```
sudo systemctl daemon-reload

sudo systemctl enable kamailio-pcscf1.service
sudo systemctl enable kamailio-scscf1.service
sudo systemctl enable kamailio-icscf.service

sudo systemctl start kamailio-pcscf1.service
sudo systemctl start kamailio-scscf1.service
sudo systemctl start kamailio-icscf.service
```
<br>

Check **P/S/I-CSCF** services status

```
sudo systemctl status kamailio-pcscf1 kamailio-scscf1 kamailio-icscf
```

## Download and install HSS

In order to download and install FoHSS, we need first to download [java](https://www.oracle.com/java/technologies/javase/javase7-archive-downloads.html#license-lightbox) and ant.

First, create this directory ```/usr/lib/jvm``` and download ```jdk-7u79-linux-x64.tar.gz``` from this link and save it in ```/usr/lib/jvm``` Then, run:

```
mkdir -p /usr/lib/jvm/
cd /usr/lib/jvm
tar -zxf jdk-7u79-linux-x64.tar.gz -C /usr/lib/jvm/
update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk1.7.0_79/bin/java 100
update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk1.7.0_79/bin/javac 100
```
Verify that java has been successfully configured by running:

```
:/usr/lib/jvm# update-alternatives --display java
java - auto mode
  link best version is /usr/lib/jvm/jdk1.7.0_79/bin/java
  link currently points to /usr/lib/jvm/jdk1.7.0_79/bin/java
  link java is /usr/bin/java
/usr/lib/jvm/jdk1.7.0_79/bin/java - priority 100
:/usr/lib/jvm# update-alternatives --display javac
javac - auto mode
  link best version is /usr/lib/jvm/jdk1.7.0_79/bin/javac
  link currently points to /usr/lib/jvm/jdk1.7.0_79/bin/javac
  link javac is /usr/bin/javac
/usr/lib/jvm/jdk1.7.0_79/bin/javac - priority 100
:/usr/lib/jvm# update-alternatives --config java
There is 1 choice for the alternative java (providing /usr/bin/java).

  Selection    Path                               Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/jdk1.7.0_79/bin/java   100       auto mode
  1            /usr/lib/jvm/jdk1.7.0_79/bin/java   100       manual mode

Press <enter> to keep the current choice[*], or type selection number:
root@EPC:/usr/lib/jvm# update-alternatives --config javac
There is 1 choice for the alternative javac (providing /usr/bin/javac).

  Selection    Path                                Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/jdk1.7.0_79/bin/javac   100       auto mode
  1            /usr/lib/jvm/jdk1.7.0_79/bin/javac   100       manual mode

Press <enter> to keep the current choice[*], or type selection number:
```

Check java version.
```
:/usr/lib/jvm# java -version
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
```
Now,download the file apache-ant-1.9.14-bin.tar.gz from this [link](http://archive.apache.org/dist/ant/binaries/apache-ant-1.9.14-bin.tar.gz) and store it in ```/root``` directory.

After that, run:
```
cd /root
tar xvfvz apache-ant-1.9.14-bin.tar.gz
mv apache-ant-1.9.14 /usr/local/
sh -c 'echo ANT_HOME=/usr/local/ >> /etc/environment'
ln -s /usr/local/apache-ant-1.9.14/bin/ant /usr/bin/ant
```

Verfiy ant version as follows:
```
:~# ant -version
Apache Ant(TM) version 1.9.14 compiled on March 12 2019
```

<br>
Now, we are ready to Download and install FHoSS

```
mkdir /opt/OpenIMSCore
cd /opt/OpenIMSCore/
git clone https://github.com/herlesupreeth/FHoSS
cd FHoSS/
export JAVA_HOME="/usr/lib/jvm/jdk1.7.0_79"
export CLASSPATH="/usr/lib/jvm/jdk1.7.0_79/jre/lib/"
ant compile deploy | tee ant_compile_deploy.txt
```
Create a file named ```configurator.sh``` inside ```/opt/OpenIMSCore/FHoSS/deploy``` with the following contents.
This file will serve to change the domain name and the IP address in the configuration files, by copying to different paths and running it.
```
#!/bin/bash
# Initialization & global vars
# if you execute this script for the second time
# you should change these variables to the latest
# domain name and ip address
DDOMAIN="open-ims\.test"
DSDOMAIN="open-ims\\\.test"
DEFAULTIP="127\.0\.0\.1"
CONFFILES=`ls *.cfg *.xml *.sql *.properties 2>/dev/null`
# Interaction
printf "Domain Name:"
read domainname
printf "IP Adress:"
read ip_address
# input domain is to be slashed for cfg regexes
slasheddomain=`echo $domainname | sed 's/\./\\\\\\\\\./g'`
if [ $# != 0 ]
then
        printf "changing: "
        for j in $*
        do
                sed -i -e "s/$DDOMAIN/$domainname/g" $j
                sed -i -e "s/$DSDOMAIN/$slasheddomain/g" $j
                sed -i -e "s/$DEFAULTIP/$ip_address/g" $j
                printf "$j "
        done
        echo
else
        printf "File to change [\"all\" for everything, \"exit\" to quit]:"
        # loop
        while read filename ;
        do
                if [ "$filename" = "exit" ]
                then
                        printf "exitting...\n"
                        break ;
                elif [ "$filename" = "all" ]
                then
                        printf "changing: "
                        for i in $CONFFILES
                        do
                                sed -i -e "s/$DDOMAIN/$domainname/g" $i
                                sed -i -e "s/$DSDOMAIN/$slasheddomain/g" $i
                                sed -i -e "s/$DEFAULTIP/$ip_address/g" $i
                                printf "$i "
                        done
                        echo
                        break;
                elif [ -w $filename ]
                then
                        printf "changing $filename \n"
                        sed -i -e "s/$DDOMAIN/$domainname/g" $filename
                        sed -i -e "s/$DSDOMAIN/$slasheddomain/g" $filename
                        sed -i -e "s/$DEFAULTIP/$ip_address/g" $filename
                else
                        printf "cannot access file $filename. skipping... \n"
                fi


                printf "File to Change:"
        done
fi
```

After it is created, provision its permissions.
```
chmod +x /opt/OpenIMSCore/FHoSS/deploy/configurator.sh
```

Start by typing ```cd /opt/OpenIMSCore/FHoSS/deploy/``` and then:

```
root@ims:/opt/OpenIMSCore/FHoSS/deploy# ./configurator.sh
Domain Name:domain.imsprovider.org
IP Adress:127.0.0.77
File to change ["all" for everything, "exit" to quit]:all
changing: DiameterPeerHSS.xml c3p0.properties hibernate.properties hss.properties
log4j.properties
root@ims:/opt/OpenIMSCore/FHoSS/deploy# cp configurator.sh ../scripts/
root@ims:/opt/OpenIMSCore/FHoSS/deploy# cd ../scripts/
root@ims:/opt/OpenIMSCore/FHoSS/scripts# ./configurator.sh
Domain Name:domain.imsprovider.org
IP Adress:127.0.0.77
File to change ["all" for everything, "exit" to quit]:all
changing: hss_db.sql hss_db_migrate_as_register.sql hss_db_migrate_dsai.sql
userdata.sql
root@ims:/opt/OpenIMSCore/FHoSS/scripts# cp configurator.sh ../config/
root@ims:/opt/OpenIMSCore/FHoSS/scripts# cd ../config/
root@ims:/opt/OpenIMSCore/FHoSS/config# ./configurator.sh
Domain Name:domain.imsprovider.org
IP Adress:127.0.0.77
File to change ["all" for everything, "exit" to quit]:all
changing: DiameterPeerHSS.xml c3p0.properties hibernate.properties hss.properties
log4j.properties
```
Now, change the following:
In the file ```/opt/OpenIMSCore/FHoSS/deploy/hibernate.properties```, revert the changed IP address
and leave this line: ```hibernate.connection.url=jdbc:mysql://127.0.0.1:3306/hss_db```.
In the file ```/opt/OpenIMSCore/FHoSS/deploy/webapps/hss.web.console/WEB-INF/web.xml```, change
```open-ims.org``` by ```domain.imsprovider.org```.
In the file ```/opt/OpenIMSCore/FHoSS/src-web/WEB-INF/web.xml```, change ```open-ims.org``` by
```domain.imsprovider.org```.

To enable the web interface of HSS being accessed
from outside the host,mMake Tomcat listen in all interfaces instead of only in 127.0.0.77. To do so, change both ```/opt/OpenIMSCore/FHoSS/deploy/hss.properties``` and
```/opt/OpenIMSCore/FHoSS/config/hss.properties``` as follows:

```
(...)
# host & port : specify the IP address and the port where Tomcat is listening, e.g. host=127.0.0.77; port=8080;

host=0.0.0.0
#host=127.0.0.77
port=8080

(...)
```
Replace ```scscf.domain.imsprovider.org``` by ```scscf1.domain.imsprovider.org``` in the three files below:

```/opt/OpenIMSCore/FHoSS/deploy/DiameterPeerHSS.xml```,
```/opt/OpenIMSCore/FHoSS/scripts/userdata.sql```, and
```/opt/OpenIMSCore/FHoSS/config/DiameterPeerHSS.xml```


Start preparing mysql database (when a pass is asked, press Enter):
```
root@ims:~# mysql
mysql> drop database hss_db;
mysql> create database hss_db;
mysql> exit

cd /opt/OpenIMSCore/
mysql
mysql -u root -p hss_db < FHoSS/scripts/hss_db.sql 
mysql -u root -p hss_db < FHoSS/scripts/userdata.sql

mysql> CREATE USER IF NOT EXISTS 'hss'@'localhost' IDENTIFIED BY 'hss';
mysql> CREATE USER IF NOT EXISTS 'hss'@'%' IDENTIFIED BY 'hss';

mysql> GRANT DELETE, INSERT, SELECT, UPDATE ON hss_db.* TO 'hss'@'localhost';
mysql> GRANT DELETE, INSERT, SELECT, UPDATE ON hss_db.* TO 'hss'@'%';

mysql> ALTER USER 'hss'@'localhost' IDENTIFIED WITH mysql_native_password BY 'hss';
mysql> ALTER USER 'hss'@'%' IDENTIFIED WITH mysql_native_password BY 'hss';

mysql> FLUSH PRIVILEGES;
mysql> exit
```
To verify:
```
mysql> SELECT user,host,plugin FROM mysql.user WHERE user='hss';
+------+-----------+-----------------------+
| user | host      | plugin                |
+------+-----------+-----------------------+
| hss  | %         | mysql_native_password |
| hss  | localhost | mysql_native_password |
+------+-----------+-----------------------+
2 rows in set (0.00 sec)

mysql>exit
```
At this point, run ```cp /opt/OpenIMSCore/FHoSS/deploy/startup.sh /root/hss.sh``` and add to
```/root/hss.sh``` the following three lines just before ```echo "Building Classpath"```:
```
cd /opt/OpenIMSCore/FHoSS/deploy
JAVA_HOME="/usr/lib/jvm/jdk1.7.0_79"
CLASSPATH="/usr/lib/jvm/jdk1.7.0_79/jre/lib/"
```
To launch HSS, you simply run the below script:
```
/root/hss.sh
```
<br>

To access the service provisioning web interface of HSS:
http://<"HOST IP Of IMS VM">:8080/hss.web.console/

Remember the default user name and password.
```
user: hssAdmin 
password: hss
```

Example:
http://192.168.50.3:8080/hss.web.console/

## Add IMS subscription in FoHSS from the Web interface

In order to add the IMS users (sip:b@domain.imsprovider.org and sip:a@domain.imsprovider.org), you can follow the steps on [IMS Kamailio Tutorial](https://github.com/anabelen-garcia/IMS-Kamailio-Tutorial#add-users-using-the-fhoss-web-interface).

You can also refer to this [guid](https://nil.uniza.sk/en/adding-new-user-ims-platform-using-hss-web-gui/).

## Building and installing PJSUA user agent

PJSUA is a high-level API and command-line softphone included in the PJSIP Project, which is an open-source multimedia communication library used to build Voice over IP (VoIP) applications.

It is quite useful for testing and troubleshooting SIP installations, because it prints out all SIP messages sent and received by the IMS nodes to console, so we will use it to test our IMS in this tutorial.

Start running the commands below to build and install it. 
```
cd
apt-get update
apt install python3-dev gcc make binutils build-essential
wget https://github.com/pjsip/pjproject/archive/refs/tags/2.16.tar.gz
tar xvf 2.16.tar.gz
cd pjproject-2.16/
export CFLAGS="$CFLAGS -fPIC"
./configure && make dep && make
cp pjsip-apps/bin/pjsua* /usr/local/bin/pjsua
```

After finishing the installation, download the wave audio file from [here](https://mauvecloud.net/sounds/pcm1608m.wav) and save it in ``/root/audios/pcm1608m.wav`` to play it in pjsua when the called SIP user answer the call.

Now, create a configuration file for 'sip:a@domain.imsprovider.org' and 'sip:b@domain.imsprovider.org' called ``/root/a.cfg`` and ``/root/b.cfg`` respectly. 

**/root/a.cfg**
```
cat /root/a.cfg
# User-dependent parameters:
## Change the proxy parameter if the PCSCF of this UA is different.
--id=sip:a@domain.imsprovider.org
--username=a@domain.imsprovider.org
--password=apass
--local-port=5566
--bound-addr=127.0.0.111
--ip-addr=127.0.0.111
--proxy=sip:pcscf1.domain.imsprovider.org
# Parameters common to all users (of same SIP domain):
## The file to be played can be different, but consider that this has been tested with
## a file with these characteristics: mono, 8000 Hz, PCM (uncompressed), 16 bits/sample
--registrar=sip:domain.imsprovider.org
--realm=domain.imsprovider.org
--null-audio
--add-codec=PCMA/8000
--use-ims
--auto-answer=180
--no-tcp
--color
--play-file=/root/audios/pcm1608m.wav
--auto-play
```

**/root/.cfg**

```
# User-dependent parameters:
## Change the proxy parameter if the PCSCF of this UA is different.
--id=sip:b@domain.imsprovider.org
--username=b@domain.imsprovider.org
--password=bpass
--local-port=5566
--bound-addr=127.0.0.222
--ip-addr=127.0.0.222
--proxy=sip:pcscf1.domain.imsprovider.org
# Parameters common to all users (of same SIP domain):
## The file to be played can be different, but consider that this has been tested with
## a file with these characteristics: mono, 8000 Hz, PCM (uncompressed), 16 bits/sample
--registrar=sip:domain.imsprovider.org
--realm=domain.imsprovider.org
--null-audio
--add-codec=PCMA/8000
--use-ims
--auto-answer=180
--no-tcp
--color
--play-file=/root/audios/pcm1608m.wav
--auto-play
```
<br>

Before starting the SIP UA,make sure that all IMS service are runnimg normally by execute:


```
sudo systemctl status kamailio-pcscf1 kamailio-scscf1.service kamailio-icscf.service rtpengine bind9.service
```


Start the HSS service by executing this command.
```
sudo /root/hss.sh
```
Open a trace on seprated terminal.
```
sudo tcpdump -i lo -s 0 -w ims_lab_trace.pcap
```
Finally, start launching the UA for both calling UA and called UA.

```
sudo pjsua --config-file /root/a.cfg
```
And 
```
sudo pjsua --config-file /root/b.cfg
```
traces/IMS_UA_registration_trace.pcap
traces/IMS_UA_registration_trace.pcap
[Registeration trace download](https://github.com/Heshamahyoup/Building-a-Pure-IMS-Lab-Using-Kamailio-on-A-Single-VM-Running-Ubuntu-24.04-LTS-No-EPC-No-Rx-/blob/main/traces/IMS_UA_registration_trace.pcap)

[Second download](https://github.com/Heshamahyoup/Building-a-Pure-IMS-Lab-Using-Kamailio-on-A-Single-VM-Running-Ubuntu-24.04-LTS-No-EPC-No-Rx-/raw/refs/heads/main/traces/traces/IMS_UA_registration_trace.pcap)

![User a Registeration](
https://github.com/Hesham-Alselwi/hesham-alselwi.github.io/blob/master/assets/images/Standalone_IMS_registration_trace.jpeg)

