# IXP Lab (FRR and BIRD as Route Server)
$\small{\textsf{A containerlab-based lab designed to offer hands-on experience with IXP route distributing using bgp community and filter.}}$

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
    peer3:
      kind: linux
      image: quay.io/frrouting/frr:8.4.1
      binds:
        - configs/frr3.conf:/etc/frr/frr.conf
        - configs/frr-daemons.cfg:/etc/frr/daemons
    akamai:
      kind: linux
      image: quay.io/frrouting/frr:8.4.1
      binds:
        - configs/akamai.conf:/etc/frr/frr.conf
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
    - endpoints: ["peer3:eth1", "ixp-net:port5"]
    - endpoints: ["akamai:eth1", "ixp-net:port6"]
```
$\small{\textsf{Start the lab}}$
```yaml
cd sros-frr-ixp-lab/
containerlab deploy 
╭─────────────────┬─────────────────────────────┬─────────┬───────────────────╮
│       Name      │          Kind/Image         │  State  │   IPv4/6 Address  │
├─────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-akamai │ linux                       │ running │ 172.20.20.5       │
│                 │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::5 │
├─────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-peer1  │ linux                       │ running │ 172.20.20.2       │
│                 │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::2 │
├─────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-peer2  │ linux                       │ running │ 172.20.20.6       │
│                 │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::6 │
├─────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-peer3  │ linux                       │ running │ 172.20.20.4       │
│                 │ quay.io/frrouting/frr:8.4.1 │         │ 3fff:172:20:20::4 │
├─────────────────┼─────────────────────────────┼─────────┼───────────────────┤
│ clab-ixp-rs2    │ linux                       │ running │ 172.20.20.3       │
│                 │ ghcr.io/srl-labs/bird:2.13  │         │ 3fff:172:20:20::3 │
╰─────────────────┴─────────────────────────────┴─────────┴───────────────────╯
```
$\small{\textsf{node တွေကို access လုပ်ဖို့}}$
```yaml
docker exec -it clab-ixp-peer1 vtysh
docker exec -it clab-ixp-peer2 vtysh
docker exec -it clab-ixp-peer3 vtysh
docker exec -it clab-ixp-akamai vtysh
docker exec -it clab-ixp-rs2 birdc
```
## BIRD Architecture

$\small{\textsf{bird version 2.0 user manual ကို ရည်ညွှန်းမယ်။}}$
```yaml
bird> show status 
BIRD 2.13
Router ID is 192.168.0.4
Hostname is rs2
Current server time is 2025-06-13 16:32:55.876
Last reboot on 2025-06-13 16:24:53.902
Last reconfiguration on 2025-06-13 16:24:53.902
Daemon is up and running

https://bird.network.cz/?get_doc&f=bird.html&v=20
```
$\small{\textsf{The heart of BIRD is a routing table. BIRD has several independent routing tables; each of them contains routes of exactly one nettype.}}$
$\small{\textsf{There are two default tables -- master4 for IPv4 routes and master6 for IPv6 routes. Other tables must be explicitly configured.}}$
$\small{\textsf{These routing tables are not kernel forwarding tables. No forwarding is done by BIRD.}}$

## Lab topology
$\small{\textsf{အောက်ပါ setup အတွက် route policy ကို config လုပ်ကြည့်မယ်။}}$

![peering drawio (1)](https://github.com/user-attachments/assets/9265474e-cedc-4bed-b586-d3a0ced5ee27)

+ $\small{\textsf{clab-ixp-peer1 သည် clab-ixp-peer2, clab-ixp-peer3 တို့နဲ့ peer ဖြစ်ချင်တယ်။}}$<br>
+ $\small{\textsf{clab-ixp-peer2 သည် clab-ixp-peer1, clab-ixp-peer3, akamai အားလုံးနဲ့  peer ဖြစ်ချင်တယ်။}}$<br>
+ $\small{\textsf{clab-ixp-peer3 သည် clab-ixp-peer1, clab-ixp-peer2, akamai အားလုံးနဲ့ peer ဖြစ်ချင်တယ်။}}$

## BGP Communities
$\small{\textsf{IX နဲ့ ချိတ်ဆက်မယ့် peer များသည် အောက်ပါ community များကို အသုံးပြုနိုင်တယ်လို့ သတ်မှတ်ထားတယ် ဆိုပါစို့။}}$
```yaml
|   Community     | Action                        |
|-----------------|-------------------------------|
|  0:<peer-as>    | Do not advertise to <peer-as> |
| 64504:<peer-as> | Advertise to <peer-as>        |
| 0:64504         | Do not advertise to any peer  |
| 64504:64510     | Advertise to Akamai           |
```
## BIRD Config
```yaml
#################
# Filter Config #
#################
define EXPORT_COMMUNITY_64501 = (64504,64501);
define EXPORT_COMMUNITY_64502 = (64504,64502);
define EXPORT_COMMUNITY_64503 = (64504,64503);
define EXPORT_COMMUNITY_64510 = (64504,64510);

filter export_filter_64501 {
 if (EXPORT_COMMUNITY_64501 ~ bgp_community) then accept;
 else reject;
}

filter export_filter_64502 {
 if (EXPORT_COMMUNITY_64502 ~ bgp_community) then accept;
 else reject;
}

filter export_filter_64503 {
 if (EXPORT_COMMUNITY_64503 ~ bgp_community) then accept;
 else reject;
}

filter export_filter_64510 {
 if (EXPORT_COMMUNITY_64510 ~ bgp_community) then accept;
 else reject;
}

######################################
# Configuration of BGP peer follows  #
######################################

### AS64501 - Client 1 - FRR ###
protocol bgp AS64501 from PEERS {
  description "Client 1";
  neighbor 192.168.0.1 as 64501;
  ipv4 {
    import all;
    export filter export_filter_64501 ;  
  };
}

### AS64502 - Client 2 - FRR ###
protocol bgp AS64502 from PEERS {
  description "Client 2";
  neighbor 192.168.0.2 as 64502;
  ipv4 {
    import all;
    export filter export_filter_64502;
  };
}

### AS64503 - Client 3 - FRR ###
protocol bgp AS64503 from PEERS {
  description "Client 3";
  neighbor 192.168.0.3 as 64503;
  ipv4 {
    import all;
    export filter export_filter_64503;
  };
}

### AS64510 - Akamai - FRR ###
protocol bgp AS64510 from PEERS {
  description "Akamai";
  neighbor 192.168.0.10 as 64510;
  ipv4 {
    import all;
    export filter export_filter_64510;
  };
}
```
+ $\small{\textsf{In a BIRD configuration, protocol device {} is a crucial section that doesn't define a routing protocol itself,}}$
  $\small{\textsf{but rather serves as a service to gather information about network interfaces from the kernel.}}$
+ $\small{\textsf{Import refers to routes flowing from a protocol (like BGP) into BIRD's internal routing table.}}$<br>
+ $\small{\textsf{Export refers to routes flowing from BIRD's routing table into a protocol.}}$<br>
  
## Peers Config
$\small{\textsf{peer1(AS64501)}}$
```yaml
ip prefix-list pl1 permit 10.0.0.1/32
route-map rmap permit 10
match ip address prefix-list pl1
set community 64504:64502 64504:64503
```
$\small{\textsf{peer2(AS64502)}}$
```yaml
ip prefix-list pl1 permit 10.0.0.2/32
route-map rmap permit 10
match ip address prefix-list pl1
set community 64504:64501 64504:64503 64504:64510
```
$\small{\textsf{peer3(AS64503)}}$
```yaml
ip prefix-list pl1 permit 10.0.0.3/32
route-map rmap permit 10
match ip address prefix-list pl1
set community 64504:64501 64504:64502 64504:64510
```
$\small{\textsf{akamai(AS64510)}}$
```yaml
ip prefix-list pl1 permit 10.0.0.10/32
route-map rmap permit 10
match ip address prefix-list pl1
set community 64504:64502 64504:64503
```
## Checking peers for route filtering

$\small{\textsf{Checking received routes and communities on route server}}$
```yaml
bird> show route 
Table master4:
10.0.0.3/32          unicast [AS64503 16:25:25.691] * (100) [AS64503i]
 via 192.168.0.3 on eth1
10.0.0.2/32          unicast [AS64502 16:25:26.763] * (100) [AS64502i]
 via 192.168.0.2 on eth1
10.0.0.1/32          unicast [AS64501 16:25:25.628] * (100) [AS64501i]
 via 192.168.0.1 on eth1
10.0.0.10/32         unicast [AS64510 16:25:26.763] * (100) [AS64510i]
 via 192.168.0.10 on eth1

bird> show route all
Table master4:
10.0.0.3/32          unicast [AS64503 16:25:25.691] * (100) [AS64503i]
 BGP.community: (64504,64501) (64504,64502) (64504,64510)
10.0.0.2/32          unicast [AS64502 16:25:26.763] * (100) [AS64502i]
 BGP.community: (64504,64501) (64504,64503) (64504,64510)
10.0.0.1/32          unicast [AS64501 16:25:25.628] * (100) [AS64501i]
 BGP.community: (64504,64502) (64504,64503)
10.0.0.10/32         unicast [AS64510 16:25:26.763] * (100) [AS64510i]
 BGP.community: (64504,64502) (64504,64503)
```
$\small{\textsf{clab-ixp-peer1}}$
```yaml
peer1# show ip bgp summary 
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.4     4      64504       642       562        0    0    0 00:27:57            2        1 N/A
Total number of neighbors 1

peer1# show ip route bgp
B>* 10.0.0.2/32 [20/0] via 192.168.0.2, eth1, weight 1, 00:27:59
B>* 10.0.0.3/32 [20/0] via 192.168.0.3, eth1, weight 1, 00:28:01
```
$\small{\textsf{clab-ixp-peer2}}$
```yaml
peer2# show ip bgp summary 
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.4     4      64504       679       592        0    0    0 00:29:26            3        1 N/A
Total number of neighbors 1

peer2# show ip route bgp
B>* 10.0.0.1/32 [20/0] via 192.168.0.1, eth1, weight 1, 00:29:35
B>* 10.0.0.3/32 [20/0] via 192.168.0.3, eth1, weight 1, 00:29:35
B>* 10.0.0.10/32 [20/0] via 192.168.0.10, eth1, weight 1, 00:29:33
```
$\small{\textsf{clab-ixp-peer3}}$
```yaml
peer3# show ip bgp summary 
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.4     4      64504       711       622        0    0    0 00:30:55            3        1 N/A
Total number of neighbors 1

peer3# show ip route bgp
B>* 10.0.0.1/32 [20/0] via 192.168.0.1, eth1, weight 1, 00:30:58
B>* 10.0.0.2/32 [20/0] via 192.168.0.2, eth1, weight 1, 00:30:56
B>* 10.0.0.10/32 [20/0] via 192.168.0.10, eth1, weight 1, 00:30:56
```
$\small{\textsf{clab-ixp-akamai}}$
```yaml
akamai# show ip bgp summary 
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.4     4      64504       738       644        0    0    0 00:32:01            2        1 N/A
Total number of neighbors 1

akamai# show ip route bgp
B>* 10.0.0.2/32 [20/0] via 192.168.0.2, eth1, weight 1, 00:32:03
B>* 10.0.0.3/32 [20/0] via 192.168.0.3, eth1, weight 1, 00:32:03
```
$\small{\textsf{some BIRD commands}}$
```yaml
bird> show status 
bird> show route
bird> show protocol
bird> show route protocol AS64501 # to check received routes from AS64501
bird> show route filter export_filter_64510
```
## Reference
```yaml
https://bird.network.cz/?get_doc&f=bird.html&v=20
https://github.com/fingon/bird-ext-lsa/blob/master/doc/bird.conf.example
https://gist.github.com/tonusoo/005d9ae8fe24432c733be8f12785ee9c
https://indico.csnog.eu/event/1/contributions/22/attachments/18/29/Ondrej_Filip.pdf
https://blog.apnic.net/2024/12/09/ixp-from-scratch-part-1-building-a-new-ixp/
https://docs.frrouting.org/en/stable-7.2/bgp.html
```


