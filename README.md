---
title: Firewall Evasion
author_1: "Héctor Toral Pallás (798095)"
author_2: "Darío Marcos Casalé (795306)"
---

# Firewall Evasion

> **Warning**
> Las trazas de ejecución mostradas a continuación pueden contener hostnames distintos para las mismas máquinas en distintos experimentos. Esto se debe a que durante la realización de los experimentos se relanzaron varias veces los contenedores de docker, y por tanto, los hostnames cambiaron.

## Requisitos
Lab Setup

![network](https://github.com/Hec7or-Uni/seginf-pr-3/blob/main/assets/lab.png)

En primer lugar se ha comprobado si el gateway de salida del tráfico a internet está asociado al interfaz eth0. Para ello se ha entrado en la máquina que actúa como router/firewall mediante el comando:

```bash
docker exec -ti <id> bash
```

Una vez dentro, se ha comprobado la correcta configuración del gateway de salida mediante el comando:

```
root@7d6166b19dfa:/# ip -br address
lo               UNKNOWN        127.0.0.1/8 
eth1@if24        UP             192.168.20.11/24 
eth0@if26        UP             10.8.0.11/24 
```

- [x] eth0 es la interfaz conectada a la red 10.8.0.0/24
- [x] eth1 es la interfaz conectada a la red 192.168.20.0/24.

Además de bloquear `www.example.com` se han añadido 2 reglas de iptables para bloquear el tráfico a `google.com` y `duckduckgo.com`:

```bash
iptables -A FORWARD -i eth1 -d 142.250.217.78 -j DROP # google.com
iptables -A FORWARD -i eth1 -d 40.89.244.232 -j DROP  # duckduckgo.com
```

Para comprobar el bloqueo del tráfico basta con entrar a una de las máquinas de la red B e intentar hacer ping a las IP bloqueadas. Como era de esperar, el ping no llega a resolverse.

```
root@a26cecbc2fd5:/# ping 142.250.217.78 -c 1
PING 142.250.217.78 (142.250.217.78) 56(84) bytes of data.

--- 142.250.217.78 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

## Tarea 1
Static Port Forwarding

La técnica de `static port forwarding` consiste en redirigir el tráfico de un puerto de la red interna a un puerto de la red externa. Para ello, se ha creado un tunel SSH entre la máquina A y la máquina B. Para ello, se ha ejecutado el siguiente comando en la máquina A:

```bash
ssh -4NT -L 0.0.0.0:4444:192.168.20.5:23 seed@192.168.20.99
```

### Pregunta 1
¿Cuántas conexiones TCP hay en todo este proceso?

Mediante la herramienta `tcpdump` se ha podido comprobar el número de conexiones TCP que se establecen durante el proceso de `static port forwarding`. Tras ejecutar el comando `tcpdump` en la máquina A, se ha podido comprobar que se establecen 2 conexiones TCP:

```
root@378ae553ce7f:/# tcpdump -i eth0 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:31:07.622591 IP A-10.8.0.99.net-10.8.0.0.41490 > B-192.168.20.99.net-192.168.20.0.ssh: Flags [S], seq 4005893999, win 64240, options [mss 1460,sackOK,TS val 402824571 ecr 0,nop,wscale 7], length 0
22:31:07.622840 IP B-192.168.20.99.net-192.168.20.0.ssh > A-10.8.0.99.net-10.8.0.0.41490: Flags [S.], seq 3721002126, ack 4005894000, win 65160, options [mss 1460,sackOK,TS val 3505122926 ecr 402824571,nop,wscale 7], length 0
```
> **Note**
> La opción `tcp[13] &2 != 0` filtra sólo aquellos paquetes con cabecera `SYN`, que sólo aparece durante el establecimiento de una conexión `TCP`.

A continuación se ha ejecutado un telnet a localhost para comprobar que el tunel SSH funciona correctamente y consigue conectarse a la maquina B1

```bash
[10/19/22]seed@VM:~/.../Labsetup$ telnet localhost 4444
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Ubuntu 20.04.1 LTS
e49005169f2e login: seed
Password: ****
```

### Pregunta 2

¿Por qué este túnel puede ayudar con éxito a los usuarios a evadir la regla del cortafuegos especificada en la configuración del laboratorio?

Esto puede ser útil porque el usuario puede acceder a la máquina en la red interna sin que el firewall bloquee el tráfico. Esto se debe a que el usuario se conecta a la máquina en la red interna a través del túnel SSH, y dado que el firewall permite este tipo de tráfico, se puede evadir la regla del firewall que bloquea el resto de conexiones.

## Tarea 2
Dynamic Port Forwarding

### Tarea 2.1
Setting Up Dynamic Port Forwarding

Para activar el `dynamic port forwarding` se ha ejecutado el siguiente comando en la máquina B:

```bash
ssh -4NT -D 0.0.0.0:4444 seed@10.8.0.99
```

Si ahora intentamos acceder a la página de google desde la máquina A, se puede ver que no nos llega el tráfico debido a que **no** hemos usado el **proxy SOCKS5**.
```
root@a26cecbc2fd5:/# curl 40.89.244.232 -m 3
curl: (28) Connection timed out after 3001 milliseconds
```

Al usar el **proxy SOCKS5**, el tráfico se redirige correctamente.
```
root@a26cecbc2fd5:/# curl --proxy socks5h://192.168.20.99:4444 40.89.244.232
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

Cabe destacar que el comportamiento es idéntico si se ejecuta el comando en la máquina B1 y B2.

--- 

Para demostrar que no se permite el tráfico a menos que tengamos el túnel entre A y B establecido, se han ejecutado los siguientes comandos en la máquina A:

- Sin túnel
```
root@8f394f60982b:/$ curl --proxy socks5h://192.168.20.99:4444 google.com
curl: (7) Failed to connect to 192.168.20.99 port 4444: Connection refused
```

- Con túnel
```
root@8f394f60982b:/$ curl --proxy socks5h://192.168.20.99:4444 google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

### Tarea 2.2
Testing the Tunnel Using Browser

Para comprobar el comportamiento relativo al túnel `SSH` se ha empleado la utilidad `tcpdump` para capturar el tráfico. Es de especial relevancia comprobar si se crean conexiones abiertas, por lo que se filtrarán sólo aquellos paquetes `SYN`.

```bash	
tcpdump -i eth0 'tcp[13] &2 != 0'
```

No aparecen paquetes de conexión hacia duckduckgo debido a que todo el tráfico va tunelizado y la conexión se ha realizado previamente al establecer el túnel SSH.

`Máquina Router(Firewall)`
```bash
root@378ae553ce7f:/$ tcpdump -i eth1 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

## Tarea 3
Virtual Private Network (VPN)

### Tarea 3.1
Bypassing Ingress Firewall

En primer lugar, es necesario establecer el túnel SSH en el cliente VPN para poder evadir el firewall.
`A(VPN Client) -> B(VPN Server)`
```bash
ssh -w 0:0 root@192.168.20.99 \
    -o "PermitLocalCommand=yes" \
    -o "LocalCommand= ip addr add 192.168.53.88/24 dev tun0 && \
    ip link set tun0 up" \
    -o "RemoteCommand=ip addr add 192.168.53.99/24 dev tun0 && \
    ip link set tun0 up"
```

A continuación, se debe modificar la configuración de enrutamiento del cliente VPN para que todo el tráfico dirigido a la red B pase a través del interfaz virtual `tun0`. Además, el tráfico con destino a la máquina B debe pasar por el firewall:
```bash
ip route replace 192.168.20.0/24 via 192.168.53.88 dev tun0
ip route add 192.168.20.99 via 10.8.0.11
```

En el lado del servidor es necesario activar el enmascaramiento dinámico para todo el tráfico saliente del servidor VPN:
```bash
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0
```

Una vez realizado el enrutamiento, se puede comprobar que el tráfico se dirige correctamente a la máquina B. Para hacer la prueba se ha realizado un telnet a la máquina B1 desde la máquina A mediante el cliente VPN.

```
root@486738e12260:/# telnet 192.168.20.5
Trying 192.168.20.5...
Connected to 192.168.20.5.
Escape character is '^]'.
Ubuntu 20.04.1 LTS
522f60eaa459 login: seed
Password: 
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-54-generic x86_64)
...
```

Cabe destacar que el comportamiento es idéntico si se intenta hacer telnet a cualquier otro nodo de la red interna (B, B1, B2)

### Tarea 3.2
Bypassing Egress Firewall

> **Warning**
> Volver a lanzar los contenedores para resetear la configuración de red del apartado 3.1

`A(VPN Server) <- B(VPN Client)`
```bash
ssh -w 0:0 root@10.8.0.99 \
    -o "PermitLocalCommand=yes" \
    -o "LocalCommand=ip addr add 192.168.53.88/24 dev tun0 && \
    ip link set tun0 up" \
    -o "RemoteCommand=ip addr add 192.168.53.99/24 dev tun0 && \
    ip link set tun0 up"
```

En el lado del cliente:
```
ip route add 52.142.124.215 dev tun0
```

En el lado del servidor:
```
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0
```

Para comprobar que el funcionamiento de la VPN es correcto, se ha realizado una petición a alguna de las páginas bloqueadas (en este caso google.com)

```
root@444234a44134:/# curl 142.250.217.78
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

## Tarea 4

### Conclusiones

Como comparación, se ha realizado una serie de pruebas con el protocolo `OpenVPN` y otras usando el `proxy SOCKS5` de tal manera que se pueda comprobar el comportamiento de cada uno de ellos.

Tras completar la práctica, hemos podido comprobar que tanto el protocolo `OpenVPN` como el `proxy SOCKS5` nos permiten realizar la técnica de Firewall Evasion. Sin embargo, a nuestro parecer, el protocolo `SOCKS5` es mucho menos complejo a la hora de configurar el túnel, por lo que es mucho más sencillo de implementar.

## Referencias 

- [VPN Masquerade HOWTO](https://tldp.org/HOWTO/VPN-Masquerade-HOWTO-2.html)
