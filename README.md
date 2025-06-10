# IXP Lab with Cisco ios, FRR and BIRD/OpenBGPd as Route Servers

$\small{\textsf{Nokia sr os မှာ license file အခက်အခဲကြောင့် cisco ios ကို အသုံးပြုထားတယ်။}}$

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
      kind: cisco_c8000v
      image: vrnetlab/cisco_c8000v:17.09.04a
    peer2:
      kind: linux
      image: quay.io/frrouting/frr:8.4.1
      binds:
        - configs/frr.conf:/etc/frr/frr.conf
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
    - endpoints: ["peer1:Gi2", "ixp-net:port1"]
    - endpoints: ["peer2:eth1", "ixp-net:port2"]
    - endpoints: ["rs1:eth1", "ixp-net:port3"]
    - endpoints: ["rs2:eth1", "ixp-net:port4"]
```
$\small{\textsf{Start the lab}}$
```yaml
containerlab deploy --topo ixp.clab.yml
╭────────────────┬─────────────────────────────────┬────────────────────┬───────────────────╮
│      Name      │            Kind/Image           │        State       │   IPv4/6 Address  │
├────────────────┼─────────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-peer1 │ cisco_c8000v                    │ running            │ 172.20.20.3       │
│                │ vrnetlab/cisco_c8000v:17.09.04a │ (health: starting) │ 3fff:172:20:20::3 │
├────────────────┼─────────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-peer2 │ linux                           │ running            │ 172.20.20.2       │
│                │ quay.io/frrouting/frr:8.4.1     │                    │ 3fff:172:20:20::2 │
├────────────────┼─────────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-rs1   │ linux                           │ running            │ 172.20.20.4       │
│                │ quay.io/openbgpd/openbgpd:7.9   │ (health: starting) │ 3fff:172:20:20::4 │
├────────────────┼─────────────────────────────────┼────────────────────┼───────────────────┤
│ clab-ixp-rs2   │ linux                           │ running            │ 172.20.20.5       │
│                │ ghcr.io/srl-labs/bird:2.13      │                    │ 3fff:172:20:20::5 │
╰────────────────┴─────────────────────────────────┴────────────────────┴───────────────────╯
```
$\small{\textsf{node တွေကို access လုပ်ဖို့}}$
```yaml
ssh admin@clab-ixp-peer1
password: admin
docker exec -it clab-ixp-peer2 vtysh
docker exec -it clab-ixp-rs1 ash
docker exec -it clab-ixp-rs2 birdc
```

