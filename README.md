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
iptables -A FORWARD -i eth1 -d 142.250.217.78 -j DROP
iptables -A FORWARD -i eth1 -d 40.89.244.232 -j DROP
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

```bash
ssh -4NT -L 0.0.0.0:<port>:192.168.20.5:23 seed@192.168.20.99
```

How many TCP connections are involved in this entire process

```bash	
tcpdump -i eth0 'tcp[13] &2 != 0' -n -s 0 -w output.pcap
```

```bash
[10/19/22]seed@VM:~/.../Labsetup$ telnet 192.168.20.5 1234
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

### Tarea 2.2
Testing the Tunnel Using Browser

### Tarea 2.3
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
