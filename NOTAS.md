# Casos de uso

## Conexión básica de 2 nodos, con multicast

**keywords**: vxlan, multicast

### escenario

Dos nodos conectados por una LAN. Iniciamos un "contenedor" en cada uno y se conectan mediante una VXLan

- Nodo 1: ip 10.10.1.23/16
  - C1: ip 192.168.1.1/16
- Nodo 2: ip 10.10.1.24/16
  - C3: ip 192.168.2.1/16

### Desarrollo

_Nodo 1_
```bash
# creamos el bridge de la vxlan
ip link add br01 type bridge
# creamos el interfaz vxlan con un grupo multicast y lo añadimos al bridge
ip link add vxlan01 type vxlan id 1 group 239.1.1.1 dstport 0 dev ens3
ip link set vxlan01 master br01
# Activamos el bridge y el interfaz vxlan
ip link set br01 up
ip link set vxlan01 up
```

**concepto:** hemos creado una especie de switch (`br01`) conectado con un cable `vxlan01` a una LAN.

```bash
# Creamos un namespace nuevo para el "contenedor"
ip netns add c1
# Creamos un par de veth
ip link add vethc1-1 mtu 1450 type veth peer name vethc1-2 mtu 1450
# El primer extremo lo conectamos al bridge y lo activamos
ip link set vethc1-1 master br01
ip link set vethc1-1 up
# Añadimos el otro extremo al "contenedor"
ip link set vethc1-2 netns c1
# Dentro del "contenedor" activamos el extremo (y también el interfaz local)
ip netns exec c1 ip link set vethc1-2 up
ip netns exec c1 ip link set lo up
# Dentro del "contenedor" le damos una IP
ip netns exec c1 ip addr add 192.168.1.1/16 dev vethc1-2
```

**concepto:** hemos creado un contenedor y lo hemos conectado con un cable (`vethc1-1` a `vethc1-2`) al switch.

_Nodo 2_

_hacemos todo igual, con el contenedor c3_

```bash
# Creacion de los recursos para la vxlan
ip link add br01 type bridge
ip link add vxlan01 type vxlan id 1 group 239.1.1.1 dstport 0 dev ens3
ip link set vxlan01 master br01
ip link set br01 up
ip link set vxlan01 up

# Creación del namespace y el dispositivo conectando a la vxlan
ip netns add c3
ip link add vethc3-1 mtu 1450 type veth peer name vethc3-2 mtu 1450
ip link set vethc3-2 netns c3
ip link set vethc3-1 master br01
ip link set vethc3-1 up
ip netns exec c3 ip link set vethc3-2 up
ip netns exec c3 ip link set lo up
ip netns exec c3 ip addr add 192.168.2.1/16 dev vethc3-2
```

### Comprobación

_Nodo 1_

```bash
ip netns exec c1 ping -c 3 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=1.11 ms
64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=0.844 ms
64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=0.828 ms

--- 192.168.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.828/0.929/1.116/0.134 ms
```

### Comentarios

Esta es el método más sencillo, utilizando multicast ya que todos los protocolos funcionan normal y corriente. El único requisito es que esté activada la política de forwarding de los bridges, para que funcionen como switches. De esta forma, cuando se usa el protocolo ARP, cuando no se encuentra una MAC, se reenviará por el resto de puertos para que se encuentre.

En principio, este método va bien ya que los grupos multicast son relativamente eficientes e irán economizando paquetes.

Podemos revisar las tablas de forwarding de los bridges y veremos que se crean entradas como las siguientes:

_nodo 2_

```console
bridge fdb
(...)
5a:99:da:e7:c0:dd dev vxlan01 vlan 1 master br01 permanent
ae:ec:0e:9a:28:16 dev vxlan01 master br01
5a:99:da:e7:c0:dd dev vxlan01 master br01 permanent
00:00:00:00:00:00 dev vxlan01 dst 239.1.1.1 via ens3 self permanent
ae:ec:0e:9a:28:16 dev vxlan01 dst 10.10.1.23 self
(...)
```

- `5a:99:da:e7:c0:dd` es la MAC de vxlan01.
- `00:00:00:00:00:00` es la MAC que se usa en ARP para preguntar por una IP. En la fdb indica que se ha de enviar por el interfaz ens3 al grupo multicast 239.1.1.1.
- `ae:ec:0e:9a:28:16` es la MAC del interfaz C1 (con IP 192.168.1.1/16), y la tabla de forwarding viene a decir que hay que enviarlo por el interfaz `vxlan01` al equipo con IP `10.10.1.23`.
  - esto ha ocurrido después de hacer un PING y, como vemos, como ya se sabe dónde estaba la MAC, se harán envíos unicast.

## Añadir nuevos "contenedores"

### Escenario

Tendremos el caso anterior, pero añadiremos C2 y C4

- Nodo 1: ip 10.10.1.23/16
  - C1: ip 192.168.1.1/16
  - C2: ip 192.168.1.2/16
- Nodo 2: ip 10.10.1.24/16
  - C3: ip 192.168.2.1/16
  - C4: ip 192.168.2.2/16

### Desarrollo

_nodo 1_

```bash
ip netns add c2
ip link add vethc2-1 mtu 1450 type veth peer name vethc2-2 mtu 1450
ip link set vethc2-1 master br01 up
ip link set vethc2-2 netns c2
ip netns exec c2 ip link set vethc2-2 up
ip netns exec c2 ip link set lo up
ip netns exec c2 ip addr add 192.168.1.2/16 dev vethc2-2
```

_nodo 2_

```bash
ip netns add c4
ip link add vethc4-1 mtu 1450 type veth peer name vethc4-2 mtu 1450
ip link set vethc4-1 master br01 up
ip link set vethc4-2 netns c4
ip netns exec c4 ip link set vethc4-2 up
ip netns exec c4 ip link set lo up
ip netns exec c4 ip addr add 192.168.2.2/16 dev vethc4-2
```

### Verificación

_nodo 2_

```console
# ip netns exec c3 ping -c 1 192.168.2.2
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=64 time=0.055 ms

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.055/0.055/0.055/0.000 ms
root@overnet06:~# ip netns exec c3 ping -c 1 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=1.08 ms

--- 192.168.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.080/1.080/1.080/0.000 ms
```

### Comentarios

Esto en realidad no tienen ningún misterio. Como iba el otro, funciona este.

## Añadir un router

### Escenario

Ahora lo que queremos es que haya un "contenedor" con IP `192.168.1.250` que haga de router para salir al exterior.

### Desarrollo

_nodo 1_

```bash
# Añadimos una IP del rango al bridge
ip addr add 192.168.1.250/16 dev br01
# activamos masquerading
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
```

_nodo 2_

```bash
# Al contenedor c3 le activamos el default route
ip netns exec c3 ip route add default via 192.168.1.250
# Ya nos funciona
ip netns exec c3 ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=13.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=8.38 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=118 time=8.42 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 8.385/10.081/13.434/2.373 ms
```

### Comentarios
Así lo que hemos hecho es establecer un nodo como router, poniéndole una IP de la vxlan a uno de los bridges de la vxlan. Lo que pasa es que esta IP cae "dentro" del nodo, y no dentro de la vxlan.

## Introducir la VXLAN en un namespace

Con esto lo que pretendemos es ordenar un poco las cosas y diferenciar mejor el tráfico.

> En particular, nos permitirá luego establecer una subred para los routers y poder llegar a utilizar una única _ip pública_ para dar salida a todas las redes privadas.

### Escenario

Lo que queremos es, simplemente tener un namespace para la vxlan en el que tengamos el bridge de la vxlan y el interfaz que encapsule el tráfico en la vxlan.

- namespace vxlan01
  - bridge brvxlan01
    - veth contendor 1

### Desarrollo

_nodo 1_

```bash
ip netns add vxlan10
ip netns exec vxlan10 ip link add brvxlan10 type bridge
ip netns exec vxlan10 ip link set dev brvxlan10 up
ip netns exec vxlan10 ip link set dev lo up

# Creamos el interfaz vxlan en el namespace principal, en el que está el dispositivo que usamos para la vxlan (ens3) y luego lo metemos en el namespace
ip link add vxlan10 type vxlan id 10 group 239.1.1.10 dstport 0 dev ens3
ip link set dev vxlan10 netns vxlan10
ip netns exec vxlan10 ip link set dev vxlan10 master brvxlan10 up

# Ahora creamos el dispositivo para el contenedor
ip link add veth-c5-10-1 mtu 1450 type veth peer name veth-c5-10-2 mtu 1450
# un extremo lo metemos en el namespace de la vxlan
ip link set dev veth-c5-10-2 netns vxlan10
ip netns exec vxlan10 ip link set dev veth-c5-10-2 master brvxlan10 up

# Ahora creamos el namespace para el contenedor, y metemos el otro extremo en dicho namespace
ip netns add c5
ip link set dev veth-c5-10-1 netns c5
ip netns exec c5 ip link set dev veth-c5-10-1 up
ip netns exec c5 ip link set dev lo up

# Le damos una IP dentro del contenedor
ip netns exec c5 ip addr add 192.168.10.5/16 dev veth-c5-10-1
```

_nodo 2_

```bash
ip netns add vxlan10
ip netns exec vxlan10 ip link add brvxlan10 type bridge
ip netns exec vxlan10 ip link set dev brvxlan10 up
ip netns exec vxlan10 ip link set dev lo up
ip link add vxlan10 type vxlan id 10 group 239.1.1.10 dstport 0 dev ens3
ip link set dev vxlan10 netns vxlan10
ip netns exec vxlan10 ip link set dev vxlan10 master brvxlan10 up

ip link add veth-c6-10-1 mtu 1450 type veth peer name veth-c6-10-2 mtu 1450
ip link set dev veth-c6-10-2 netns vxlan10
ip netns exec vxlan10 ip link set dev veth-c6-10-2 master brvxlan10 up


ip netns add c6
ip link set dev veth-c6-10-1 netns c6
ip netns exec c6 ip link set dev veth-c6-10-1 up
ip netns exec c6 ip link set dev lo up

ip netns exec c6 ip addr add 192.168.10.6/16 dev veth-c6-10-1
```

## Introducir el router en la VXLAN

Ahora que tenemos la vxlan en un namespace, podemos hacer que haya un router en el namespace y dentro de la vxlan, con lo que será exclusivo de esa vxlan.

### Excenario

_nodo 1_ (**puede hacer ping a 8.8.8.8**)
  - namespace vxlan10
    - bridge brvxlan10 (192.168.1.1 gateway)
      - veth contendor c5 (192.168.10.5)
      - vxlan10
    - veth LAN routers B (172.16.1.10)
  - brrouter (172.16.1.1)
    - veth LAN routers A
  - ens3 (10.10.1.16)

_nodo 2_
  - namespace vxlan10
    - bridge brvxlan10
      - veth contendor c6 (192.168.10.6)
      - vxlan10

Desde dentro de los contenedores se podrá salir a 8.8.8.8

### Desarrollo

_nodo 1_

```bash
# Creamos el interfaz que hara de router para los interfaces que harán SNAT (masquerading) 
ip link add brrouter type bridge
ip link set dev brrouter up
ip addr add 172.16.1.1/16 dev brrouter

# Creamos el interfaz que hará el masquerading para la vxlan10 (con IP 172.16.1.10) y lo metemos dentro del namespace de la vxlan
ip link add r-vxlan10-1 type veth peer name r-vxlan10-2
ip link set dev r-vxlan10-2 netns vxlan10
ip netns exec vxlan10 ip link set dev r-vxlan10-2 up
ip netns exec vxlan10 ip addr add 172.16.1.10/16 dev r-vxlan10-2

# El otro extremo lo metemos en el bridge de los routers
ip link set dev r-vxlan10-1 up
ip link set dev r-vxlan10-1 master brrouter

# Ahora activamos el masquerading dentro del namespace (para que saquen las IPs privadas a la red del router)
ip netns exec vxlan10 iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
# Ponemos el router
ip netns exec vxlan10 ip route add default via 172.16.1.1
# Activamos el masquerading para que el router principal pueda sacar el tráfico al exterior
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
```

_nodo 2_

```bash
$ ip netns exec c6 ping 8.8.8.8
connect: Network is unreachable
$ ip netns exec c6 ping 172.16.1.1
connect: Network is unreachable
$ ip netns exec c6 ping 172.16.1.10
connect: Network is unreachable
$ ip netns exec c6 ip route add default via 192.168.1.1
$ ip netns exec c6 ping -c 1 172.16.1.10
PING 172.16.1.10 (172.16.1.10) 56(84) bytes of data.
64 bytes from 172.16.1.10: icmp_seq=1 ttl=64 time=1.10 ms

--- 172.16.1.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.106/1.106/1.106/0.000 ms
$ ip netns exec c6 ping -c 1 172.16.1.1
PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
64 bytes from 172.16.1.1: icmp_seq=1 ttl=63 time=0.975 ms

--- 172.16.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.975/0.975/0.975/0.000 ms
$ ip netns exec c6 ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=8.77 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 8.771/8.771/8.771/0.000 ms
```

### NOTAS

De esta forma podemos tener vxlan con IPs solapadas y usar un único router para todas ellas.

La única precaución es que los rangos de IPs de dentro de las VXLans no pueden coincidir con el rango de IPs de los routers.

## VXLAN sin multicast, pero con BUM

La idea es evitar el multicast (transformarlo en unicast), ya que hay redes que no lo permiten (lo del grupo 239.1.1.1).

### Notas
En realidad, lo del multicast no es un problema grande... sólo que quien quiera se podría meter en el grupo multicast y enterarse (recibir el tráfico y meter equipos pirata). Al usar unicast, decidimos dónde enviamos el tráfico de las vxlans.

> Esto implica un problemilla de configuración y es que cuando das de alta un nodo, tienes que darlo en **todos** los nodos que forman parte de la red. Hay que hacer lo de `bridge fdb add 00:00:00:00:00:00 ...` para todos los nodos, en todos los nodos.
