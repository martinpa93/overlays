# overlays
MUCNAP UPV - Cloud Computing

Initially two containers start running with networking to none.

$ docker run --rm -dit --network none --name no-net-alpine alpine:latest ash
$ docker run --rm -dit --network none --name no-net-alpine2 alpine:latest ash

Docker (and probably any container technology) uses linux network namespaces to isolate container network from host network. When Docker creates and runs a container; it creates a separate network namespace (container network) and puts the container into it.

Then, Docker connects the new container network to linux bridge docker0 using a veth pair. This also enables container be connected to the host network and other container networks in the same bridge.

I will create a network namespace and a veth pair, then connect host network to container network using the veth pair mentioned.

Create two namespaces

$ ip netns add ns1
$ ip netns add ns2

The interfaces and the route table on namespaces are empty:
$ ip netns exec ns1 ip a 
$ ip netns exec ns2 ip a 
$ ip netns exec ns1 ip route 
$ ip netns exec ns2 ip route 

Create the Veth Pairs

$ ip link add veth0 type veth peer name veth1
$ ip link add veth2 type veth peer name veth3
$ ip link set veth1 netns docker0
$ sudo ip netns exec docker0 ip addr show

Creating the bridge 

$ sudo-apt-get install bridge-utils
$ sudo brctl addbr docker0
$ brctl show
$ ip a add 172.16.1.254/16 dev docker0
$ ip link set dev docker0 up
$ route -n


Connecting host

$sudo brctl addif br0 veth1
$sudo brctl addif br0 veth2

Assign IP to veth

$sudo ip netns exe docker0 ip addr add 172.12.0.10/24 dev veth1
$sudo ip netns exe docker0 ip addr add 172.12.0.12/24 dev veth2

Then , I assign the Ip address to the bridge br0 and set up:

$sudo ip addr add 172.12.0.11/24 dev br0
$sudo ip link set br0 up

Externally acces service exposed within the container

$ sudo iptables -t nat -A PREROUTING ! -i docker0 -p tcp -m tcp --dport 80 -j DNAT --to-destintion 172.12.0.10:80
$sudo ip netns exec docker0 nc -lp 80
