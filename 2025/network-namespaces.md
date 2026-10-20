# Using Network Namespaces on Linux

### Date: 2025-11-09

I have been doing some experiments with networking on Linux recently and have been
looking for a nice way to simulate a network on a single host and stumbled upon network namespaces, which are an alternative
to setting up networking using full-fledged Docker containers or even virtual machines.

### What are network namespaces?

To quote the [man page for network namespaces](https://man7.org/linux/man-pages/man7/network_namespaces.7.html):

> Network namespaces provide isolation of the system resources associated
> with networking: network devices, IPv4 and IPv6 protocol stacks, IP
> routing tables, firewall rules, the /proc/net directory (which is a
> symbolic link to /proc/pid/net), the /sys/class/net directory, various
> files under /proc/sys/net, port numbers (sockets), and so on. In addition,
> network namespaces isolate the UNIX domain abstract socket namespace (see unix(7)).

In other words, with network namespaces, we obtain virtual networks
that are isolated from the main network resources of our system.

Notably, if we execute a socket application inside a namespace, that
application sees only the network environment of the namespace, not that
of the host system.

### Setting network namespaces up

This can be achieved as follows:

```bash
root@debian-2025-11-05:~# ip netns add nns-a
root@debian-2025-11-05:~# ip netns add nns-b
```

This command adds a new network namespace called `nns-a` and another one called `nns-b`. Existing network namespaces can be displayed using:

```bash
root@debian-2025-11-05:~# ip netns list
nns-b
nns-a (id: 0)
```

Deletion of a network namespace can be achieved as follows:

```bash
root@debian-2025-11-05:~# ip netns delete nns-a
```

We can execute an application inside a network namespace as follows:

```bash
root@debian-2025-11-05:~# ip netns exec nns-a /bin/bash
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
root@debian-2025-11-05:~# exit
exit
```

### Connecting network namespaces

We add [virtual Ethernet devices](https://man7.org/linux/man-pages/man4/veth.4.html) on the host system as follows:

```bash
root@debian-2025-11-05:~# ip link add veth1-nns-a type veth peer name veth1-nns-b
```

This creates a veth device `veth1-nns-a` and its peer `veth1-nns-b`:

```bash
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    altname enx525400123456
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute ens3
       valid_lft 85308sec preferred_lft 74508sec
    inet6 fec0::64ad:59c5:464c:882e/64 scope site dynamic mngtmpaddr noprefixroute 
       valid_lft 86396sec preferred_lft 14396sec
    inet6 fe80::5e56:554a:16d7:d837/64 scope link 
       valid_lft forever preferred_lft forever
3: veth1-nns-b@veth1-nns-a: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:a2:02:f0:e7:7b brd ff:ff:ff:ff:ff:ff
4: veth1-nns-a@veth1-nns-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 8e:e6:bb:20:f9:c5 brd ff:ff:ff:ff:ff:ff
```

Next, we move `veth1-nns-a` into the network namespace `nns-a`:

```bash
root@debian-2025-11-05:~# ip link set dev veth1-nns-a netns nns-a
```

Analogously, `veth1-nns-b` is moved to `nns-b`:

```bash
root@debian-2025-11-05:~# ip link set dev veth1-nns-b netns nns-b
```

Inside `nns-a`, the new veth device shows up:

```bash
root@debian-2025-11-05:~# ip netns exec nns-a /bin/bash
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1-nns-a@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 8e:e6:bb:20:f9:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

In another terminal window, we observe the same for `nns-b`:

```bash
root@debian-2025-11-05:~# ip netns exec nns-b /bin/bash
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1-nns-b@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:a2:02:f0:e7:7b brd ff:ff:ff:ff:ff:ff link-netns nns-a
```

The only remaining step is to bring both devices up in their respective namespaces:

```bash
root@debian-2025-11-05:~# ip link set dev veth1-nns-a up
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1-nns-a@if3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 8e:e6:bb:20:f9:c5 brd ff:ff:ff:ff:ff:ff link-netns nns-b
```

```bash
root@debian-2025-11-05:~# ip link set dev veth1-nns-b up
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1-nns-b@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:a2:02:f0:e7:7b brd ff:ff:ff:ff:ff:ff link-netns nns-a
    inet6 fe80::50a2:2ff:fef0:e77b/64 scope link tentative proto kernel_ll 
       valid_lft forever preferred_lft forever
```

We now have a working connection between the two network namespaces and
can send and receive Ethernet frames between the two network interfaces
of the different namespaces.
We can achieve this by using the MAC addresses `8e:e6:bb:20:f9:c5` of `veth1-nns-a` and `52:a2:02:f0:e7:7b` of `veth1-nns-b` in the Ethernet header and sending them through their respective network interfaces.

If we assign IP addresses in the same subnet (e.g. `172.16.0.2/24` and
`172.16.0.3/24`) to the two network interfaces, we can also send IP
packets.

Inside `nns-a`:

```bash
root@debian-2025-11-05:~# ip addr add 172.16.0.2/24 dev veth1-nns-a
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1-nns-a@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 8e:e6:bb:20:f9:c5 brd ff:ff:ff:ff:ff:ff link-netns nns-b
    inet 172.16.0.2/24 scope global veth1-nns-a
       valid_lft forever preferred_lft forever
    inet6 fe80::8ce6:bbff:fe20:f9c5/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

And inside `nns-b`:

```bash
root@debian-2025-11-05:~# ip addr add 172.16.0.3/24 dev veth1-nns-b
root@debian-2025-11-05:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1-nns-b@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:a2:02:f0:e7:7b brd ff:ff:ff:ff:ff:ff link-netns nns-a
    inet 172.16.0.3/24 scope global veth1-nns-b
       valid_lft forever preferred_lft forever
    inet6 fe80::50a2:2ff:fef0:e77b/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
root@debian-2025-11-05:~# ping 172.16.0.2
PING 172.16.0.2 (172.16.0.2) 56(84) bytes of data.
64 bytes from 172.16.0.2: icmp_seq=1 ttl=64 time=0.287 ms
64 bytes from 172.16.0.2: icmp_seq=2 ttl=64 time=0.234 ms
64 bytes from 172.16.0.2: icmp_seq=3 ttl=64 time=0.396 ms
64 bytes from 172.16.0.2: icmp_seq=4 ttl=64 time=0.386 ms
64 bytes from 172.16.0.2: icmp_seq=5 ttl=64 time=0.373 ms
^C
--- 172.16.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4015ms
rtt min/avg/max/mdev = 0.234/0.335/0.396/0.063 ms
```

A useful tool for network debugging is [tcpdump](https://www.tcpdump.org/), which we can use to observe the incoming ICMP ping packets inside `nns-a`:

```bash
root@debian-2025-11-05:~# tcpdump -i veth1-nns-a -p -l
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth1-nns-a, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:40:15.013325 IP 172.16.0.3 > 172.16.0.2: ICMP echo request, id 1401, seq 1, length 64
19:40:15.013437 IP 172.16.0.2 > 172.16.0.3: ICMP echo reply, id 1401, seq 1, length 64
19:40:16.015896 IP 172.16.0.3 > 172.16.0.2: ICMP echo request, id 1401, seq 2, length 64
19:40:16.016169 IP 172.16.0.2 > 172.16.0.3: ICMP echo reply, id 1401, seq 2, length 64
19:40:16.522802 IP6 fe80::8ce6:bbff:fe20:f9c5 > ip6-allrouters: ICMP6, router solicitation, length 16
19:40:17.017312 IP 172.16.0.3 > 172.16.0.2: ICMP echo request, id 1401, seq 3, length 64
19:40:17.017437 IP 172.16.0.2 > 172.16.0.3: ICMP echo reply, id 1401, seq 3, length 64
19:40:18.020752 IP 172.16.0.3 > 172.16.0.2: ICMP echo request, id 1401, seq 4, length 64
19:40:18.020900 IP 172.16.0.2 > 172.16.0.3: ICMP echo reply, id 1401, seq 4, length 64
19:40:19.049993 IP 172.16.0.3 > 172.16.0.2: ICMP echo request, id 1401, seq 5, length 64
19:40:19.050203 IP 172.16.0.2 > 172.16.0.3: ICMP echo reply, id 1401, seq 5, length 64
19:40:20.105023 ARP, Request who-has 172.16.0.3 tell 172.16.0.2, length 28
19:40:20.105382 ARP, Request who-has 172.16.0.2 tell 172.16.0.3, length 28
19:40:20.105410 ARP, Reply 172.16.0.2 is-at 8e:e6:bb:20:f9:c5 (oui Unknown), length 28
19:40:20.105539 ARP, Reply 172.16.0.3 is-at 52:a2:02:f0:e7:7b (oui Unknown), length 28
^C
15 packets captured
15 packets received by filter
0 packets dropped by kernel
```

### References

[This blog post by Giles Thomas](https://www.gilesthomas.com/2021/03/fun-with-network-namespaces) explains network namespaces quite well and in more detail.
