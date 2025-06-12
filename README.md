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
+ $\small{\textsf{initial bird config မှာ import all, export all သတ်မှတ်ထားတာကြောင့် peer ၂ ခုကြား route တွေအားလုံး ဖလှယ်ထားတာ တွေ့ရမယ်။}}$

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
## BIRD Architecture

$\small{\textsf{bird version 2.0 user manual ကို ရည်ညွှန်းပြီး လုပ်ဆောင်မယ်။}}$
```yaml
https://bird.network.cz/?get_doc&f=bird.html&v=20
```
$\small{\textsf{The heart of BIRD is a routing table. BIRD has several independent routing tables; each of them contains routes of exactly one nettype.}}$
$\small{\textsf{There are two default tables -- master4 for IPv4 routes and master6 for IPv6 routes. Other tables must be explicitly configured.}}$
$\small{\textsf{These routing tables are not kernel forwarding tables. No forwarding is done by BIRD.}}$

## function
```yaml
| Evaluation order |   Community     | Action                        |
|------------------|-----------------|-------------------------------|
| 1                |  0:<peer-as>    | Do not advertise to <peer-as> |
| 2                | 47200:<peer-as> | Advertise to <peer-as>        |
| 3                | 0:47200         | Do not advertise to any peer  |
| 4                | 47200:47200     | Advertise to all peers        |

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
    import all;
    export where bgp_out(64501);
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
$\small{\textsf{RTS BGP:}}$
$\small{\textsf{This RTS indicates that the route was learned from a BGP neighbor. When a route has RTS BGP, BIRD knows that the route was received from another BGP router.}}$
$\small{\textsf{Import refers to routes flowing from a protocol (like BGP) into BIRD's internal routing table.}}$
$\small{\textsf{Export refers to routes flowing from BIRD's routing table into a protocol.}}$
$\small{\textsf{Filters are used to control both import and export of routes.}}$ 

## filter

$\small{\textsf{A filter has a header, a list of local variables, and a body. The header consists of the filter keyword followed by a (unique) name of filter.}}$
$\small{\textsf{The list of local variables consists of type name; pairs where each pair declares one local variable.}}$
```yaml
filter accept_community {
         community 65000:10;
     }
```
$\small{\textsf{net; (Network):}}$
$\small{\textsf{This term within BIRD's configuration or logging indicates a network route.}}$
$\small{\textsf{It specifies a range of IP addresses that traffic should be forwarded to based on the configured route.}}$

$\small{\textsf{dead; (Dead Route):}}$
$\small{\textsf{This term signifies that a route has been withdrawn, is no longer valid, or the network is unreachable.}}$
$\small{\textsf{It's a way for BIRD to signal that a specific route is no longer functional and should not be used for forwarding traffic.}}$


