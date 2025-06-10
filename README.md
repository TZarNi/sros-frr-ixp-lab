# IXP Lab with Nokia SR OS, FRR and BIRD/OpenBGPd as Route Servers

$\small{\textsf{A containerlab-based lab designed to offer hands-on experience with IXP technologies and best practices.}}$

$\small{\textsf{Lab documentation is available at below link}}$
```yaml
<https://containerlab.dev/lab-examples/peering-lab/>
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
$\small{\textsf{start the lab}}$
```yaml
cd sros-frr-ixp-lab
containerlab deploy --topo ixp.clab.yml
```
