# Unicast Mode

```bash
ip link add vxlan99 type vxlan id 99 dstport 0 nolearning proxy l2miss l3miss
ip link add dev br99 type bridge
ip link set dev vxlan99 master br99
ip link set dev br99 up
ip link set dev vxlan99 up
set -e


if [ "$1" == "1" ]; then
# contenedor 1
ip netns add uni1
ip link add veth-1 mtu 1450 address f4:15:40:05:00:01 type veth peer name veth-2 mtu 1450 address f4:15:40:05:00:02
ip link set dev veth-1 master br99
ip link set dev veth-1 up
ip link set dev veth-2 netns uni1
ip netns exec uni1 ip link set dev veth-2 up
ip netns exec uni1 ip link set dev lo up
ip netns exec uni1 ip addr add 192.168.5.1/16 dev veth-2

# contenedor 2
ip netns add uni2
ip link add veth-3 mtu 1450 address f4:15:40:05:00:03 type veth peer name veth-4 mtu 1450 address f4:15:40:05:00:04
ip link set dev veth-3 master br99
ip link set dev veth-3 up
ip link set dev veth-4 netns uni2
ip netns exec uni2 ip link set dev veth-4 up
ip netns exec uni2 ip link set dev lo up
ip netns exec uni2 ip addr add 192.168.5.2/16 dev veth-4
fi

if [ "$1" == "2" ]; then
# contenedor 1
ip netns add uni3
ip link add veth-1 mtu 1450 address f4:15:40:06:00:01 type veth peer name veth-2 mtu 1450 address f4:15:40:06:00:02
ip link set dev veth-1 master br99
ip link set dev veth-1 up
ip link set dev veth-2 netns uni3
ip netns exec uni3 ip link set dev veth-2 up
ip netns exec uni3 ip link set dev lo up
ip netns exec uni3 ip addr add 192.168.6.3/16 dev veth-2

# contenedor 2
ip netns add uni4
ip link add veth-3 mtu 1450 address f4:15:40:06:00:03 type veth peer name veth-4 mtu 1450 address f4:15:40:06:00:04
ip link set dev veth-3 master br99
ip link set dev veth-3 up
ip link set dev veth-4 netns uni4
ip netns exec uni4 ip link set dev veth-4 up
ip netns exec uni4 ip link set dev lo up
ip netns exec uni4 ip addr add 192.168.6.4/16 dev veth-4
fi
```