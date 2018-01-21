---
layout: post
title: "NAT66: The good, the bad, the ugly"
---

NAT (and NAPT) is one of those technologies anyone has a strong opinion about. It has been for years the necessary evil and invaluable (yet massive) hack that kept IPv4 from falling apart in the face of its abysmally small 32-bit address space - which was, to be honest, an absolute OK choice for the time the protocol was designed, when computers cost a small fortune, and were as big as lorries.

The Internet Protocol, version 4, has been abused for quite too long now.  We made it into the fundamental building block of the modern Internet, a network of a scale it was never designed for. We are well in due time to put it at rest and replace it with its controversial, yet problem-solving 128-bit grandchild, IPv6.

So, what should be the place for NAT in the new Internet, which makes the return to the end-to-end principle one of its main tenets?

### NAT66 misses the point

Well, none, according to the IETF, which has for years tried to dissuade everyone with dabbing with NAT66 (the name NAT is known on IPv6); this is not without good reasons, though. For too long, the supposedly stateless, connectionless level 3 IP protocol has been made into an impromptu "stateful", connection-oriented protocol by NAT gateways, just for the sake to meet the demands of an infinite number of devices trying to connect to the Internet. 

This is without considering the false sense of security that address masquerading provides; I cannot recall how many times I've heard people say that *(gasp!)* NAT was fundamental piece in the security of their internal networks (it's not). 

Given that the immensity of the IPv6 address space allows providers to give out full `/64`s to customers, I'd always failed to see the point in NAT66: it always felt to me as a feature fundamentally dead in the water, a solution seeking a problem, ready to be misused.

Well, this was before discovering how cheap some hosting services could be.

### Being cheap: the root of all evils

I was quite glad to see a while ago that my VPS provider had announced IPv6 support; thanks to this, I would have been finally able to provide IPv6 access to the guests of the VPNs I host on that VPS, without having to incur into the delay penalties caused by tunneling the traffic on good old services such as Hurrican Electric and SixXS [^1]. Hooray!

My excitement was unfortunately not going to last for long, and it was indeed barbarically butchered when I discovered that despite having been granted a full `/32` (2<sup>96</sup> IPs), my provider decided to give its VPS customers just a single `/128` address.

JUST. A. SINGLE. ONE.


![Oh. God. Why.]({{ "/public/ohgodwhy.png" | absolute_url }} "Y U SO CHEAP?"){: .center-image}


Given that IPv6 connectivity was something I really wished for my OpenVPN setup, this was quite a setback.
I was left with fundamentally only two reasonable choices:

1. Get a free /64 from a Hurricane Electric tunnel, and allocate IPv6s for VPN guests from there;
2. Be a very bad person, set up NAT66, and feel ashamed.

Hurrican Electric is, without doubt, the most orthodox option between the two; it's free of charge, it gives out `/64`s, and it's quite easy to set up.  

The main showstopper here is definitely the increased network latency added by two layers of tunnelling (VPN -> 6to4 -> IPv6 internet), and, given that by default native IPv6 source IPs are preferred to IPv4, it would have been bad if having a v6 public address incurred in a slow down of connections with usually tolerable latencies. Especially if there was a way to get decent RTTs for both IPv6 and IPv4...
 
And so, with a pang of guilt, I shamefully committed the worst crime.

### How to get away with NAT66

The process of setting up NAT usually relies on picking a specially reserved privately-routable IP range, to avoid our internal network structure to get in conflict with the outer networking routing rules (it still may happen, though, if under multiple misconfigured levels of masquerading).  

The IPv6 equivalent to `10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16` has been defined in 2005 by the IETF, not without a whole deal of confusion first, with the Unique Local Addresses (ULA) specification. This RFC defines the unique, not publicly routable `fc00::/7` that is supposed to be used to define local subnets, without the unicity guarantees of `2000::/3` (the range from which Global Unicast Addresses (GUA) - i.e. the Internet - are allocated from for the time being). From it, `fd00::/8` is the only block really defined so far, and it's meant to define all of the `/48`s your private network may ever need. 

The next step was to configure my OpenVPN instances to give out  ULAs from subnets of my choice to clients, by adding at the end of to my config the following lines:

```
server-ipv6 fd00::1:8:0/112
push "route-ipv6 2000::/3"
```

I resorted to picking `fd00::1:8:0/112` for the UDP server and `fd00::1:9:0/112` for the TCP one, due to a limitation in OpenVPN only accepting masks from `/64` to `/112`. 

Given that I also want traffic towards the Internet to be forwarded via my NAT, it is also necessary to instruct the server to push a default route to its clients at connection time.

```
$ ping fd00::1:8:1
PING fd00::1:8:1(fd00::1:8:1) 56 data bytes
64 bytes from fd00::1:8:1: icmp_seq=1 ttl=64 time=40.7 ms
```

The clients and servers were now able to ping each other through their local addresses without any issue, but the outer network was still unreachable.

I continued the creation of this abomination by configuring the kernel to forward IPv6 packets; this is achieved by setting the `net.ipv6.conf.all.forwarding = 1` with `sysctl` or in `sysctl.conf` (from now on, the rest of this article assumes that you are under Linux).

```
# cat /etc/sysctl.d/30-ipforward.conf 
net.ipv4.ip_forward=1
net.ipv6.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1
# sysctl -p /etc/sysctl.d/30-ipforward.conf
```

Afterwards, the only step left was to set up NAT66, which can be easily done by configuring the stateful firewall provided by Linux' packet filter.  
I personally prefer (and use) the newer `nftables` to the `{ip,ip6,arp,eth}tables` mess it is supposed to supersede, because I find it tends to be quite less moronic and clearer to understand (despite the relatively scarce documentation available online, which is sometimes a pain. I wish Linux had the excellent OpenBSD's pf...).  
Feel free to use `ip6tables`, if that's what you are already using, and you don't really feel the need to migrate your ruleset to `nft`. 

This is a shortened, summarised snippet of the rules that I've had to put into my nftables.conf to make NAT66 work; I've also left the IPv4 rules in for the sake of completeness.

```
table inet filter {
  [...]
  chain forward {
    type filter hook forward priority 0;

    # allow established/related connections                                                                                                                                                                                                 
    ct state {established, related} accept
    
    # early drop of invalid connections                                                                                                                                                                                                     
    ct state invalid drop

    # Allow packets to be forwarded from the VPNs to the outer world
    ip saddr 10.0.0.0/8 iifname "tun*" oifname eth0 accept
    # Using fd00::1:0:0/96 allows to match for every fd00::1:xxxx:0/112 
    # I set up
    ip6 saddr fd00::1:0:0/96 iifname "tun*" oifname eth0 accept
  }
  [...]
}
# IPv4 NAT table
table ip nat {
  chain prerouting {
    type nat hook prerouting priority 0; policy accept;
  }
  chain postrouting {
    type nat hook postrouting priority 100; policy accept;
    ip saddr 10.0.0.0/8 oif "eth0" snat to 51.254.131.73
  }
} 

# IPv6 NAT table
table ip6 nat {
  chain prerouting {
    type nat hook prerouting priority 0; policy accept;
  }
  chain postrouting {
    type nat hook postrouting priority 100; policy accept;

    # Creates a SNAT (source NAT) rule that changes the source 
    # address of the outbound IPs with the external IP of eth0
    ip6 saddr fd00::1:0:0/96 oif "eth0" snat to <external IPv6>
  }
}
```

The ip6 nat table and the forwarding inet chain are the most important stuff to notice here, given that they respectively configure the packet filter to perform NAT66 and to forward packets from the `tun*` interfaces to the outer world.

After applying the new ruleset with `nft -f <path/to/ruleset>` command, I was ready to witness the birth of our my little sinful setup. 
The only thing left was to ping a known IPv6 from one of the clients, to ensure that forwarding and NAT are working fine. One of the Google DNS servers would suffice:

```
$ ping 2001:4860:4860::8888
PING 2001:4860:4860::8888(2001:4860:4860::8888) 56 data bytes
64 bytes from 2001:4860:4860::8888: icmp_seq=1 ttl=54 time=48.7 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=2 ttl=54 time=47.5 ms
$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=49.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=50.8 ms
```
Perfect! NAT66 was working, in its full evil glory, and the client was able to reach the outer IPv6 Internet with round-trip times as fast as IPv4. What was left now was to check if the clients were able to resolve AAAA records; given that I was already using Google's DNS in /etc/resolv.conf, it should have worked straight away:

```
$ ping facebook.com
PING facebook.com (157.240.1.35) 56(84) bytes of data.
^C
$ ping -6 facebook.com
PING facebook.com(edge-star-mini6-shv-01-lht6.facebook.com (2a03:2880:f129:83:face:b00c:0:25de)) 56 data bytes
^C
```

What? Why is ping trying to reach Facebook on its IPv4 address by default instead of trying IPv6 first?

### One workaround always leads to another

Well, it turned out that Glibc's getaddrinfo() function, which is generally used to perform DNS resolution, uses a precedence system to correctly prioritise source-destination address pairs.

I started to suspect that the default behaviour of getaddrinfo() could be to consider local addresses (including ULA) as a separate case than global IPv6 ones; so, I tried to check `gai.conf`, the configuration file for the IPv6 DNS resolver.

```
label ::1/128       0  # Local IPv6 address
label ::/0          1  # Every IPv6
label 2002::/16     2 # 6to4 IPv6
label ::/96         3 # Deprecated IPv4-compatible IPv6 address prefix
label ::ffff:0:0/96 4  # Every IPv4 address
label fec0::/10     5 # Deprecated 
label fc00::/7      6 # ULA
label 2001:0::/32   7 # Teredo addresses
```

What is shown in the snippet above is the default label table used by `getaddrinfo()`.   
As I suspected, a ULA address is labelled differently (6) than a global Unicast one (1), and, because the default behaviour specified by RFC 3484 is to prefer pairs of source-destination addresses with the same label, the IPv4 is picked over the IPv6 ULA every time.  
Damn, I was so close to commiting the perfect crime.

To make this mess finally functional, I had to make yet another ugly hack (as if NAT66 using ULAs wasn't enough), by setting a new label table in `gai.conf` that didn't make distinctions between addresses.

```
label ::1/128       0  # Local IPv6 address
label ::/0          1  # Every IPv6
label 2002::/16     2 # 6to4 IPv6
label ::/96         3 # Deprecated IPv4-compatible IPv6 address prefix
label ::ffff:0:0/96 4  # Every IPv4 address
label fec0::/10     5 # Deprecated 
label 2001:0::/32   7 # Teredo addresses
```

By omitting the label for `fc00::/7`, ULAs are now grouped together with GUAs, and natted IPv6 connectivity is used by default.

```
$ ping google.com
PING google.com(par10s29-in-x0e.1e100.net (2a00:1450:4007:80f::200e)) 56 data bytes
```

### In conclusion

So, yes, NAT66 can be done and it works, but that doesn't make it any less than the messy, dirty hack it is.
For the sake of getting IPv6 connectivity behind a provider too cheap to give its customers a `/64`, I had to forgo end-to-end connectivity, hacking Unique Local Addresses to achieve something they weren't really devised for.   


Was it worthy? Perhaps. My ping under the VPN is now as good on IPv6 as it is on IPv4, and everything works fine, but this came at the cost of an overcomplicated network configuration.
This could have been much simpler if everybody had simply understood how IPv6 differs from IPv4, and that giving out a single address is simply not the right way to allocate addresses to your subscribers anymore.

The NATs we use today are relics of a past where the address space was so small that we had to break the Internet in order to save it. They were a mistake made to fix an even bigger one, a blunder whose effects we have now the chance to undo.
We should just start to take the ongoing transition period as seriously as it deserves, to avoid falling into the same wrong assumptions yet again.
  
[^1]: Ironically, SixXS closed last June because "many ISPs offer IPv6 now".  
