---
title: Firewall Evasion
author_1: "Héctor Toral Pallás (798095)"
author_2: "Darío Marcos Casalé (795306)"
---

# Firewall Evasion

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
ssh -4NT -L 0.0.0.0:<port>:192.168.20.5:23 seed@192.168.20.99
```

### Pregunta 1
Cuántas conexiones TCP hay en todo este proceso.

Mediante la herramienta `tcpdump` se ha podido comprobar el número de conexiones TCP que se establecen durante el proceso de `static port forwarding`. Tras ejecutar el comando `tcpdump` en la máquina A, se ha podido comprobar que se establecen 2 conexiones TCP:

```
root@378ae553ce7f:/# tcpdump -i eth0 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:31:07.622591 IP A-10.8.0.99.net-10.8.0.0.41490 > B-192.168.20.99.net-192.168.20.0.ssh: Flags [S], seq 4005893999, win 64240, options [mss 1460,sackOK,TS val 402824571 ecr 0,nop,wscale 7], length 0
22:31:07.622840 IP B-192.168.20.99.net-192.168.20.0.ssh > A-10.8.0.99.net-10.8.0.0.41490: Flags [S.], seq 3721002126, ack 4005894000, win 65160, options [mss 1460,sackOK,TS val 3505122926 ecr 402824571,nop,wscale 7], length 0
```

A continuación se ha ejecutado un telnet a localhost para comprobar que el tunel SSH funciona correctamente y consigue conectarse a la maquina B1

```bash
[10/19/22]seed@VM:~/.../Labsetup$ telnet localhost <port>
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Ubuntu 20.04.1 LTS
e49005169f2e login: seed
Password: ****
```

### Pregunta 2

¿Por qué este túnel puede ayudar con éxito a los usuarios a evadir la regla del cortafuegos especificada en la configuración del laboratorio?

Esto puede ser útil porque el usuario puede acceder a la máquina en la red interna sin tener que pasar por el firewall. Esto se debe a que el usuario se conecta a la máquina en la red interna a través del túnel SSH.

## Tarea 2
Dynamic Port Forwarding

### Tarea 2.1
Setting Up Dynamic Port Forwarding

Para activar el `dynamic port forwarding` se ha ejecutado el siguiente comando en la máquina B:

```bash
ssh -4NT -D 0.0.0.0:4444 seed@10.8.0.99
```

Si ahora intentamos acceder a la página de google desde la máquina A, se puede ver que no nos llega el trafico debido a que **no** hemos usado el **proxy SOCKS5**
```
root@a26cecbc2fd5:/# curl 40.89.244.232 -m 3
curl: (28) Connection timed out after 3001 milliseconds
```

Al usar el **proxy SOCKS5**, el tráfico se redirige correctamente
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

--- 

Para demostrar que no se permite el tráfico a menos que tengamos el tunel entre A y B establecido, se han ejecutado los siguientes comandos en la máquina A:

```
root@8f394f60982b:/$ curl --proxy socks5h://192.168.20.99:4444 google.com
curl: (7) Failed to connect to 192.168.20.99 port 4444: Connection refused
```

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

```bash	
tcpdump -i eth0 'tcp[13] &2 != 0' -n -s 0 -w output.pcap
```

```bash	
root@a26cecbc2fd5:/# tcpdump -i eth0 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
10:54:58.846947 IP 192.168.20.1.58802 > a26cecbc2fd5.4444: Flags [S], seq 3801517113, win 64240, options [mss 1460,sackOK,TS val 3286071150 ecr 0,nop,wscale 7], length 0
10:54:58.846975 IP a26cecbc2fd5.4444 > 192.168.20.1.58802: Flags [S.], seq 2456604960, ack 3801517114, win 65160, options [mss 1460,sackOK,TS val 3129383428 ecr 3286071150,nop,wscale 7], length 0
10:54:58.918391 IP 192.168.20.1.58806 > a26cecbc2fd5.4444: Flags [S], seq 1690393414, win 64240, options [mss 1460,sackOK,TS val 3286071221 ecr 0,nop,wscale 7], length 0
10:54:58.918420 IP a26cecbc2fd5.4444 > 192.168.20.1.58806: Flags [S.], seq 2817506843, ack 1690393415, win 65160, options [mss 1460,sackOK,TS val 3129383499 ecr 3286071221,nop,wscale 7], length 0
10:55:08.060662 IP 192.168.20.1.58810 > a26cecbc2fd5.4444: Flags [S], seq 3394860571, win 64240, options [mss 1460,sackOK,TS val 3286080364 ecr 0,nop,wscale 7], length 0
10:55:08.060684 IP a26cecbc2fd5.4444 > 192.168.20.1.58810: Flags [S.], seq 1601240667, ack 3394860572, win 65160, options [mss 1460,sackOK,TS val 3129392642 ecr 3286080364,nop,wscale 7], length 0
10:55:08.801615 IP 192.168.20.1.58814 > a26cecbc2fd5.4444: Flags [S], seq 10207535, win 64240, options [mss 1460,sackOK,TS val 3286081105 ecr 0,nop,wscale 7], length 0
10:55:08.801652 IP a26cecbc2fd5.4444 > 192.168.20.1.58814: Flags [S.], seq 1732189123, ack 10207536, win 65160, options [mss 1460,sackOK,TS val 3129393383 ecr 3286081105,nop,wscale 7], length 0
```

No aparecen paquetes de conexion ni de trafico hacia duckduckgo pq va todo tunelizado
```
root@378ae553ce7f:/# tcpdump -i eth1 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

### Tarea 2.3 (OPCIONAL)
Writing a SOCKS Client Using Python

## Tarea 3
Virtual Private Network (VPN)

### Tarea 3.1
Bypassing Ingress Firewall

Lanzar el túnel para saltar el firewall en el cliente VPN:
A(client) -> B(server)
```bash
ssh -w 0:0 root@192.168.20.99\
    -o "PermitLocalCommand=yes" \
    -o "LocalCommand= ip addr add 192.168.53.88/24 dev tun0 && \
    ip link set tun0 up" \
    -o "RemoteCommand=ip addr add 192.168.53.99/24 dev tun0 && \
    ip link set tun0 up"
```

En el lado del cliente:
```bash
ip route replace 192.168.20.0/24 via 192.168.53.88 dev tun0
ip route add 192.168.20.99 via 10.8.0.11
```

En el lado del servidor:
```bash
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0
```

```
telnet 192.168.53.88
```

```bash
ip route add 192.168.20.0/24 via 192.168.53.88 dev tun0
```

### Tarea 3.2
Bypassing Egress Firewall

> **Warning**
> Volver a lanzar los contenedores para resetear la configuración de red del apartado 3.1

```bash

`A`: VPN Server
`B`: VPN Client 

```bash	
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0
```

A(server) -> B(client)
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

## Tarea 4
Comparing SOCKS5 Proxy and VPN

## Conclusiones

## Referencias 
