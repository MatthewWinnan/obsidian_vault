# General Overview

VyOS is a router operating system built on top of #Debian. It uses a config approach to store and load routing configurations utilising the built in Debian networking tool. [VyOS](https://vyos.io/)

In my hardware setup I will detail how I configured VyOS to work on the Odroid H3+ running proxmox. I am currently using VyOS 1.5.x circinus.

My ISP uses #PPPoE in order to authenticate me and thus provide me with internet service. I also want my router to directly be connected to the ISP and be the sole provider of networking to my lab. As such I will not be doing double NAT or anything like that. 

In order to do this the following steps will need to be performed:

1. **Configure PPPoE on WAN Interface**: Use my router’s interface to enter PPPoE credentials and establish a connection to my ISP.
2. **Set up the DHCP Server**: Configure the #DHCP service on the LAN interface to automatically assign IP addresses to connected devices.
3. **Enable NAT**: Ensure #NAT is enabled on my router to allow multiple devices to share the Internet connection.
4. **Connect and Configure the Access Switch**: Connect the switch to the router’s LAN port. 
5. **Test the Connection**: Connect a device to the switch and verify it receives an IP address, can access the network, and reaches the Internet.

###### Tags
- #VyOS
- #HomeLabSetup