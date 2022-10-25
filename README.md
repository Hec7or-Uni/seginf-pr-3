---
title: Firewall Evasion
author_1: "Héctor Toral Pallás (798095)"
author_2: "Darío Marcos Casalé (795306)"
---

# Firewall Evasion

## Tarea 0
Lab Setup

![network](https://github.com/Hec7or-Uni/seginf-pr-3/blob/main/assets/lab.png)

En primer lugar se ha comprobado si el gateway de salida del tráfico a internet está asociado al interfaz eth0. Para ello se ha entrado en la máquina que actúa como router/firewall:
```bash
docker exec -ti <id> bash
```

```
root@7d6166b19dfa:/# ip -br address
lo               UNKNOWN        127.0.0.1/8 
eth1@if24        UP             192.168.20.11/24 
eth0@if26        UP             10.8.0.11/24 
```

- [x] eth0 is the interface connected to the 10.8.0.0/24 network
- [x] eth1 is the one connected to 192.168.20.0/24.

> **Note**: ¿ Añadir CIDR para bloquear una red entera o basta con bloquear una IP ?

```bash
iptables -A FORWARD -i eth1 -d 142.250.217.78 -j DROP # google.com
iptables -A FORWARD -i eth1 -d 40.89.244.232 -j DROP  # duckduckgo.com
```

Para comprobar el bloqueo del tráfico basta con entrar a una de las máquinas de la red B e intentar hacer ping a las IP bloqueadas. Como era de esperar, el ping no llega a resolverse
```
root@a26cecbc2fd5:/# ping 142.250.217.78 -c 1
PING 142.250.217.78 (142.250.217.78) 56(84) bytes of data.

--- 142.250.217.78 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

## Tarea 1
Static Port Forwarding

Ejecutamos el siguiente comando en la máquina A
```bash
ssh -4NT -L 0.0.0.0:<port>:192.168.20.5:23 seed@192.168.20.99
```

How many TCP connections are involved in this entire process

```bash	
tcpdump -i eth0 'tcp[13] &2 != 0' -n -s 0 -w output.pcap
```

```bash
[10/19/22]seed@VM:~/.../Labsetup$ telnet localhost <port>
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Ubuntu 20.04.1 LTS
e49005169f2e login: seed
Password: ****
```

```
root@378ae553ce7f:/# tcpdump -i eth0 'tcp[13] &2 != 0'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:31:07.622591 IP A-10.8.0.99.net-10.8.0.0.41490 > B-192.168.20.99.net-192.168.20.0.ssh: Flags [S], seq 4005893999, win 64240, options [mss 1460,sackOK,TS val 402824571 ecr 0,nop,wscale 7], length 0
22:31:07.622840 IP B-192.168.20.99.net-192.168.20.0.ssh > A-10.8.0.99.net-10.8.0.0.41490: Flags [S.], seq 3721002126, ack 4005894000, win 65160, options [mss 1460,sackOK,TS val 3505122926 ecr 402824571,nop,wscale 7], length 0
```

Why can this tunnel successfully help users evade the firewall rule specified in the lab setup?<br>
This can be helpful because the user can access the machine in the internal network without having to go through the firewall. This is because the user is connecting to the machine in the internal network through the SSH tunnel.

## Tarea 2
Dynamic Port Forwarding

### Tarea 2.1
Setting Up Dynamic Port Forwarding

Ejecutamos el siguiente comando en la máquina B para activar la redirección de puertos dinámica
```bash
ssh -4NT -D 0.0.0.0:4444 seed@10.8.0.99
```

Si ahora intentamos acceder a la página de google desde la máquina A, se puede ver que no nos llega el trafico debido a que no hemos usado el proxy SOCKS5
```
root@a26cecbc2fd5:/# curl 40.89.244.232 -m 3
curl: (28) Connection timed out after 3001 milliseconds
```

Al usar el proxy SOCKS5, el tráfico se redirige correctamente
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

Para demostrar que el mismo comando falla en caso de que el túnel no esté definido, a continuación se incluye el resultado antes y después de establecer el túnel entre B y A:
```bash
root@8f394f60982b:/$ curl --proxy socks5h://192.168.20.99:4444 google.com
curl: (7) Failed to connect to 192.168.20.99 port 4444: Connection refused
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

### Tarea 3.2
Bypassing Egress Firewall

## Tarea 4
Comparing SOCKS5 Proxy and VPN

## Conclusiones

## Referencias 
