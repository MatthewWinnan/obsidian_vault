NAT turns out to be an extremely useful protocol besides for one of the more common use cases, whereby private addresses are translates to public addresses. [#RFC2663](https://datatracker.ietf.org/doc/html/rfc2663) details standard NAT terminology and considerations. Quoting from the abstract:

> Network Address Translation is a method by which IP addresses are
   mapped from one realm to another, in an attempt to provide
   transparent routing to hosts. Traditionally, NAT devices are used to
   connect an isolated address realm with private unregistered addresses
   to an external realm with globally unique registered addresses. 

NAT works through a few main steps: 
1. Firstly, address binding occurs, mapping private addresses to public ones for session initiation. 
2. Secondly, address lookup and translation adjust the IP addresses in packet headers to ensure they're appropriate for the target network realm. 
3. Finally, address unbinding releases the mapping once the session ends, freeing up the public address for future use.

Now for my specific use case I will only be working with IPv4 addresses. As such I will mainly focus on VyOS's NAT44 implementation [(1)](https://docs.vyos.io/en/latest/configuration/nat/nat44.html). This implementation and other broader strategies are discussed in [#RFC3022](https://datatracker.ietf.org/doc/html/rfc3022#ref-NAT-TERM).  I am only assigned on public address as such I have a many to one relationship and the best use case would be #NAPT. 

RFC3022 defined basic and port NAT as follows: Basic NAT involves mapping internal private addresses to a set of external addresses for communication with external networks. NAPT extends this by also translating TCP/UDP port numbers, allowing multiple internal hosts to share a single external address. 

NAPT is also sometimes referred to as #PAT (Port Address Translation) and in the VyOS ecosystem this role is configured using the #SNAT (Source Network Address Translation) implementation [(2)](https://docs.vyos.io/en/latest/configuration/nat/nat44.html#snat). NAPT can further be extended to allow inbound access by statically mapping a local node per each service TU port of the registered IP address. Traditionally this is referred to as port forwarding. While I am planning on using a VPN to securely access my internal network externally many of these concepts still apply. I also just want to add it to the documentation for extra information. 

The VyOS ecosystem defines port forwarding as #DNAT (Destination Network Address Translation) [(3)](https://docs.vyos.io/en/latest/configuration/nat/nat44.html#dnat). One can see that DNAT and SNAT would both be useful if one wants to allow bi-directional access. This is defined in RFC3022 as Bi-directional NAT and discussed in a more general sense in RFC2663. 

I plan to configure the VyOS router using the following set of commands:

```
set nat source rule 100 outbound-interface name 'pppoe0'
set nat source rule 100 source address 'x.x.x.x/24'
set nat source rule 100 translation address 'masquerade'
```

In VyOS NAT is rule based, as such we configure rule 100 to apply to outbound traffic. We use the static address assigned to the router as the source address to translate. We translate it via #masquerade. This target is effectively an alias to say “use whatever IP address is on the outgoing interface”, rather than a statically configured IP address. This is useful if you use DHCP for your outgoing interface and do not know what the external address will be.

###### Tags
- #NAT
- #HomeLabSetup