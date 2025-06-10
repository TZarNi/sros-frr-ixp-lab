# IXP Lab (FRR and BIRD/OpenBGPd as Route Servers)

$\small{\textsf{Nokia sr os မှာ license file အခက်အခဲကြောင့် FRR ကိုပဲ အသုံးပြုထားတယ်။}}$

$\small{\textsf{A containerlab-based lab designed to offer hands-on experience with IXP technologies and best practices.}}$

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
    rs1: # route server #1
      kind: linux
      image: quay.io/openbgpd/openbgpd:7.9
      binds:
        - configs/openbgpd.conf:/etc/bgpd/bgpd.conf
      exec:
        - "ip address add dev eth1 192.168.0.3/24"
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
    - endpoints: ["rs1:eth1", "ixp-net:port3"]
    - endpoints: ["rs2:eth1", "ixp-net:port4"]
```
$\small{\textsf{Start the lab}}$
```yaml
containerlab deploy --topo ixp.clab.yml
╭────────────────┬───────────────────────────────┬────────────────────┬───────────────────╮
│      Name      │           Kind/Image          │        State       │   IPv4/6 Address  │
├────────────────┼───────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-peer1 │ linux                         │ running            │ 172.20.20.5       │
│                │ quay.io/frrouting/frr:8.4.1   │                    │ 3fff:172:20:20::5 │
├────────────────┼───────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-peer2 │ linux                         │ running            │ 172.20.20.4       │
│                │ quay.io/frrouting/frr:8.4.1   │                    │ 3fff:172:20:20::4 │
├────────────────┼───────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-rs1   │ linux                         │ running            │ 172.20.20.3       │
│                │ quay.io/openbgpd/openbgpd:7.9 │ (health: starting) │ 3fff:172:20:20::3 │
├────────────────┼───────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-rs2   │ linux                         │ running            │ 172.20.20.2       │
│                │ ghcr.io/srl-labs/bird:2.13    │                    │ 3fff:172:20:20::2 │
╰────────────────┴───────────────────────────────┴────────────────────┴───────────────────╯
```
$\small{\textsf{node တွေကို access လုပ်ဖို့}}$
```yaml
docker exec -it clab-ixp-peer1 vtysh
docker exec -it clab-ixp-peer2 vtysh
docker exec -it clab-ixp-rs1 ash
docker exec -it clab-ixp-rs2 birdc
```
## BIRD Config
```yaml
In a BIRD configuration, protocol device {} is a crucial section that doesn't define a routing protocol itself,
but rather serves as a service to gather information about network interfaces from the kernel. 
```
## Checking BGP routes
$\small{\textsf{clab-ixp-peer1}}$
```yaml
peer1#  show ip bgp summary 

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 64501 vrf-id 0
BGP table version 2
RIB entries 3, using 576 bytes of memory
Peers 2, using 1434 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.3     4      64503        91        92        0    0    0 00:04:22            1        2 N/A
192.168.0.4     4      64503       104        92        0    0    0 00:04:22            1        2 N/A

peer1# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/0] via 172.20.20.1, eth0, 00:04:37
C>* 10.0.0.1/32 is directly connected, lo, 00:04:36
B>* 10.0.0.2/32 [20/0] via 192.168.0.2, eth1, weight 1, 00:04:34
C>* 172.20.20.0/24 is directly connected, eth0, 00:04:37
C>* 192.168.0.0/24 is directly connected, eth1, 00:04:36
```
$\small{\textsf{clab-ixp-rs2}}$
```yaml
bird> show route
Table master4:
10.0.0.2/32          unicast [AS64502 14:51:35.953] * (100) [AS64502i]
 via 192.168.0.2 on eth1
                     unicast [AS64501 14:51:36.004 from 192.168.0.1] (100) [AS64502i]
 via 192.168.0.2 on eth1
10.0.0.1/32          unicast [AS64501 14:51:35.985] * (100) [AS64501i]
 via 192.168.0.1 on eth1
                     unicast [AS64502 14:51:36.035 from 192.168.0.2] (100) [AS64501i]
 via 192.168.0.1 on eth1
```
