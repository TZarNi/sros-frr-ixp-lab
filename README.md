# IXP Lab (FRR and BIRD as Route Servers)
$\small{\textsf{A containerlab-based lab designed to offer hands-on experience with IXP technologies and best practices.}}$

$\small{\textsf{Nokia sr os မှာ license file အခက်အခဲကြောင့် FRR ကိုပဲ အသုံးပြုထားတယ်။}}$

$\small{\textsf{BIRD config ကို focus လုပ်ချင်တာမို့ လောလောဆယ် openbgpd ကို ခဏဖယ်ထားတယ်။}}$

$\small{\textsf{Lab documentation is available at below link}}$
```yaml
https://containerlab.dev/lab-examples/peering-lab/
```
## Setup

$\small{\textsf{clone the lab}}$
```yaml
git clone https://github.com/hellt/sros-frr-ixp-lab.git
```
$\small{\textsf{linux bridge network တစ်ခု ဖန်တီးမယ်။}}$
```yaml
sudo apt install bridge-utils
sudo brctl addbr ixp-net
ip link show
```
$\small{\textsf{clab yaml file ကို လိုအပ်သလို ပြုပြင်မယ်။}}$
```yaml
name: ixp

topology:
  nodes:
    peer1:
      kind: linux
      image: quay.io/frrouting/frr:8.4.1
      binds:
        - configs/frr1.conf:/etc/frr/frr.conf
        - configs/frr-daemons.cfg:/etc/frr/daemons
    peer2:
      kind: linux
      image: quay.io/frrouting/frr:8.4.1
      binds:
        - configs/frr2.conf:/etc/frr/frr.conf
        - configs/frr-daemons.cfg:/etc/frr/daemons
    rs2: # route server #2
      kind: linux
      image: ghcr.io/srl-labs/bird:2.13
      binds:
        - configs/bird.conf:/etc/bird.conf
      exec:
        - "ip address add dev eth1 192.168.0.4/24"
    ixp-net:
      kind: bridge

  links:
    - endpoints: ["peer1:eth1", "ixp-net:port1"]
    - endpoints: ["peer2:eth1", "ixp-net:port2"]
    - endpoints: ["rs2:eth1", "ixp-net:port4"]
```
$\small{\textsf{Start the lab}}$
```yaml
containerlab deploy --topo ixp.clab.yml
╭────────────────┬─────────────────────────────┬─────────┬───────────────────╮
│      Name      │          Kind/Image         │  State  │   IPv4/6 Address  │
├────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-peer1 │ linux                       │ running │ 172.20.20.3       │
│                │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::3 │
├────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-peer2 │ linux                       │ running │ 172.20.20.2       │
│                │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::2 │
├────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-rs2   │ linux                       │ running │ 172.20.20.4       │
│                │ ghcr.io/srl-labs/bird:2.13  │         │ 3fff:172:20:20::4 │
╰────────────────┴─────────────────────────────┴─────────┴───────────────────╯
```
$\small{\textsf{node တွေကို access လုပ်ဖို့}}$
```yaml
docker exec -it clab-ixp-peer1 vtysh
docker exec -it clab-ixp-peer2 vtysh
docker exec -it clab-ixp-rs2 birdc
```
## BIRD Architecture

$\small{\textsf{bird version 2.0 user manual ကို ရည်ညွှန်းပြီး လုပ်ဆောင်မယ်။}}$
```yaml
https://bird.network.cz/?get_doc&f=bird.html&v=20
```
$\small{\textsf{The heart of BIRD is a routing table. BIRD has several independent routing tables; each of them contains routes of exactly one nettype.}}$
$\small{\textsf{There are two default tables -- master4 for IPv4 routes and master6 for IPv6 routes. Other tables must be explicitly configured.}}$
$\small{\textsf{These routing tables are not kernel forwarding tables. No forwarding is done by BIRD.}}$

## BIRD Config
$\small{\textsf{import all, export all}}$
```yaml
# https://nsrc.org/workshops/2021/riso-pern-apan51/networking/routing-security/en/labs/ixp.html
router id 192.168.0.4;
define myas = 64503;

protocol device { }

#####################
# Protocol template #
#####################
template bgp PEERS {
  local as myas;
  rs client;
}

#############################
# Configuration of BGP peer #
#############################

### AS64501 - Client 1 - FRR ###
protocol bgp AS64501 from PEERS {
  description "Client 1";
  neighbor 192.168.0.1 as 64501;
  ipv4 {
    import all;
    export all;
  };
}
### AS64502 - Client 2 - FRR ###
protocol bgp AS64502 from PEERS {
  description "Client 2";
  neighbor 192.168.0.2 as 64502;
  ipv4 {
    import all;
    export all;
  };
}
```
+ $\small{\textsf{In a BIRD configuration, protocol device {} is a crucial section that doesn't define a routing protocol itself,}}$
  $\small{\textsf{but rather serves as a service to gather information about network interfaces from the kernel.}}$ 
+ $\small{\textsf{bird config မှာ import all, export all သတ်မှတ်ထားတာကြောင့် peer ၂ ခုကြား route တွေအားလုံး ဖလှယ်ထားတာ တွေ့ရမယ်။}}$

$\small{\textsf{Checking BGP routes}}$

$\small{\textsf{clab-ixp-peer1}}$
```yaml
peer1#  show ip bgp summary 
peer1# show ip route
```
$\small{\textsf{clab-ixp-peer2}}$
```yaml
peer2# show ip bgp summary 
peer2# show ip route
```
$\small{\textsf{clab-ixp-rs2}}$
```yaml
bird> show status 
bird> show route
bird> show protocol
bird> show route protocol AS64501 # to check received routes from AS64501
bird> show route protocol AS64502
```
$\small{\textsf{import none, export none}}$
```yaml
# https://nsrc.org/workshops/2021/riso-pern-apan51/networking/routing-security/en/labs/ixp.html
router id 192.168.0.4;
define myas = 64503;

protocol device { }

#####################
# Protocol template #
#####################
template bgp PEERS {
  local as myas;
  rs client;
}

#############################
# Configuration of BGP peer #
#############################

### AS64501 - Client 1 - FRR ###
protocol bgp AS64501 from PEERS {
  description "Client 1";
  neighbor 192.168.0.1 as 64501;
  ipv4 {
    import none;
    export none;
  };
}
### AS64502 - Client 2 - FRR ###
protocol bgp AS64502 from PEERS {
  description "Client 2";
  neighbor 192.168.0.2 as 64502;
  ipv4 {
    import none;
    export none;
  };
}
```
+ $\small{\textsf{bird config မှာ import none, export none သတ်မှတ်ထားတာကြောင့် peer ၂ ခုကြား ဖလှယ်ထားတဲ့ route တစ်ခုမှမရှိတာ တွေ့ရမယ်။}}$

## Communities
$\small{\textsf{IX နဲ့ ချိတ်ဆက်မယ့် peer များသည် အောက်ပါ community များကို အသုံးပြုနိုင်တယ်လို့ သတ်မှတ်ထားတယ် ဆိုပါစို့။}}$
```yaml
|   Community     | Action                        |
|-----------------|-------------------------------|
|  0:<peer-as>    | Do not advertise to <peer-as> |
| 64503:<peer-as> | Advertise to <peer-as>        |
| 0:64503         | Do not advertise to any peer  |
| 64503:64503     | Advertise to all peers        |
| 9654:64503      | Advertise to Akamai           |
```
## Function
$\small{\textsf{BIRD config}}$
```yaml
############
# Function #
############
function bgp_out(int peeras)
{
 if ! (source = RTS_BGP ) then return false;
 if (0,peeras) ~ bgp_community then return false;
 if (myas,peeras) ~ bgp_community then return true;
 if (0, myas) ~ bgp_community then return false;
 return true;
}

#####################################
# Configuration of BGP peer follows #
#####################################

### AS64501 - Client 1 - FRR ###
protocol bgp AS64501 from PEERS {
  description "Client 1";
  neighbor 192.168.0.1 as 64501;
  ipv4 {
    import where bgp_out(64501);
    export none;
  };
}

### AS64502 - Client 2 - FRR ###
protocol bgp AS64502 from PEERS {
  description "Client 2";
  neighbor 192.168.0.2 as 64502;
  ipv4 {
    import none;
    export all;
  };
}
```
$\small{\textsf{RTS BGP:}}$
$\small{\textsf{This RTS indicates that the route was learned from a BGP neighbor. When a route has RTS BGP, BIRD knows that the route was received from another BGP router.}}$
$\small{\textsf{Import refers to routes flowing from a protocol (like BGP) into BIRD's internal routing table.}}$
$\small{\textsf{Export refers to routes flowing from BIRD's routing table into a protocol.}}$

$\small{\textsf{clab-ixp-peer1 (AS64501) config}}$
$\small{\textsf{clab-ixp-peer1 သည် clab-ixp-peer2 နဲ့ peer လုပ်ချင်တယ်။}}$
$\small{\textsf{clab-ixp-peer2 သည် akamai နဲ့ peer လုပ်ချင်တယ်။}}$
$\small{\textsf{clab-ixp-peer3 သည် clab-ixp-peer1, clab-ixp-peer2, akamai အားလုံးနဲ့ peer လုပ်ချင်တယ်။}}$
```yaml
route-map rmap permit 10
match ip address prefix-list pl1
set community 64503:64501
!
router bgp 64501
  bgp router-id 10.0.0.1
  no bgp default ipv4-unicast
  bgp bestpath as-path multipath-relax
  neighbor 192.168.0.3 remote-as 64503
  neighbor 192.168.0.4 remote-as 64503
  !
  address-family ipv4 unicast
    network 10.0.0.1/32
    neighbor 192.168.0.3 activate
    neighbor 192.168.0.4 route-map rmap out
    neighbor 192.168.0.4 activate
  exit-address-family
```
```yaml
peer1# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/0] via 172.20.20.1, eth0, 00:00:40
C>* 10.0.0.1/32 is directly connected, lo, 00:00:40
C>* 172.20.20.0/24 is directly connected, eth0, 00:00:40
C>* 192.168.0.0/24 is directly connected, eth1, 00:00:40

peer2# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/0] via 172.20.20.1, eth0, 00:00:45
B>* 10.0.0.1/32 [20/0] via 192.168.0.1, eth1, weight 1, 00:00:11
C>* 10.0.0.2/32 is directly connected, lo, 00:00:45
C>* 172.20.20.0/24 is directly connected, eth0, 00:00:45
C>* 192.168.0.0/24 is directly connected, eth1, 00:00:45

bird> show route
Table master4:
10.0.0.1/32          unicast [AS64501 14:45:33.846] * (100) [AS64501i]
         via 192.168.0.1 on eth1
```
$\small{\textsf{သတ်မှတ်ထားတဲ့ function အရ BIRD က အောက်ပါအတိုင်း လုပ်ဆောင်တယ်။}}$
+ $\small{\textsf{ AS64502 က route အားလုံးကို main table သို့ import လုပ်တယ်။}}$




