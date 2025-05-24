**PPPoE (Point-to-Point Protocol over Ethernet)** is a network protocol that encapsulates PPP frames inside Ethernet frames. It is commonly used for broadband connection to the Internet over DSL, providing authentication (typically through a username and password), encryption, and compression. The protocol is explained in [#RFC2516](https://datatracker.ietf.org/doc/html/rfc2516). In summary the initial discovery phase can be illustrated by the following steps:

1. PADI (PPP Active Discovery Initiation): The client broadcasts a PADI packet to initiate the discovery process, seeking PPPoE servers. 
2. PADO (PPP Active Discovery Offer): PPPoE servers respond to the PADI with PADO packets, offering service. 
3. PADR (PPP Active Discovery Request): The client responds to one of the offers by sending a PADR packet to the chosen PPPoE server. 
4. PADS (PPP Active Discovery Session-confirmation): The server responds to PADR with a PADS packet, confirming the session and providing a session ID. 
5. Session Stage: After the exchange of PAD packets and session establishment, PPP session data is exchanged within PPPoE frames.

![Visual Illustration Of Discovery Process](pppoe_discovery.png)

In essence the router will be the PPPoE client and will need to establish a connection with the PPPoE server which is hosted by the ISP.

My ISP does not use DSL as such I needn't worry about converting between DSL and Ethernet. However I am using FTTH and as such I need a ONT. This is all provided as such I assume I only need to send a PPPoE request to the ISP from for example eth0 on my router through the ONT. 

As mentioned I do not want double NAT, but this is an option

Typical configuration options can be found at VyOS's admin docs under their PPPoE section [(1)](https://docs.vyos.io/en/latest/configuration/interfaces/pppoe.html).

The following should set eth0 to connect to the PPPoE server and form an active session. Further it will instruct the router to not install advertised DNS nameservers into the local system. I will also be using [#RFC3704](https://datatracker.ietf.org/doc/html/rfc3704.html) and enable strict mode to prevent IP spoofing from DDos attacks.

```
set interfaces pppoe pppoe0 no-peer-dns
set interfaces pppoe pppoe0 ip source-validation strict
set interfaces pppoe pppoe0 authentication username 'userid'
set interfaces pppoe pppoe0 authentication password 'secret'
set interfaces pppoe pppoe0 source-interface 'eth0'
```

Further operational commands can be found at [(2)](https://docs.vyos.io/en/latest/configuration/interfaces/pppoe.html#operation)
###### Tags
- #PPPoE 
- #HomeLabSetup