# Workshop 1: Configuring IPv4 and IPv6

[ [Back to Root page](https://github.com/rhjacobsen/CN_workshops/blob/master/README.md) ]

This workshop consists of two parts. The first part is dealing with IPv4 static address configuration, address resolution (ARP), and autoconfiguration (DHCP). The second part is dealing with IPv6 autoconfiguration and address resolution which is part of the Neighbour Discovery Protocol (NDP).

The overall goal of the workshop is to consolidate the material that has been covered in the lectures on IPv4 and IPv6 and get some experience working with basic IP concepts in practice.

## Lab overview

### Setup

![Project](imgs/LAB01.png "Lab overview - setup")

### Networks

| Name         | Network
|--------------|--------------------------
| Shannon      | 10.0.0.0 / 24
| Shannon      | 2001:16d8:dd92:1001::0 /64

## Creating and Setting up the Laboratory
The same GNS3 project is used for all experiments. The first step is to create the project.
 * Start GNS3 and create (or open an existing) project <sup>1</sup>
 * Add an Ethernet Switch, two Nodes (Hosts), and a Router
 * Connect them as shown in the figure above
 * Save the project <sup>2</sup> 
 * Start all devices, wait until all links are green
 * Right click a link, and choose _start capture_, Wireshark protocol analyzer should start

#### Notes:
```
1) You man need to run GNS3 as administrator for the environment to have access to the virtual server.
2) Configurations that you have typed are not persistently stored when you close a project.
```

## Part 1: IPv4 Configuration and Address Resolution
The objective of these experiments is to configure the network with IPv4 addresses using static and dynamic address configuration applying the Dynamic Host Configuration Protocol (DHCP), and observe the behaviour of the ARP and ICMP protocols. DHCP can be used to configure more than just IP-addresses, but for simplicity this will not be covered in this workshop.

The experiments consists of the following steps. When asked to fill in information, do so before continuing to the next step.

### Step 1: Static IPv4 Address Configuration
 * Login to "Router-1" and configure it with the IPv4 address 10.0.0.1 using the command:

    ```
    ip addr add 10.0.0.1/24 dev eth0
    ip link set eth0 up
    ```
 * Login to "Node-1", and configure it with the IPv4 address 10.0.0.2 using the command:

    ```
    ip addr add 10.0.0.2/24 dev eth0
    ip link set eth0 up
    ```
 * Verify the settings via

    ```
    ip addr show eth0
    ```

### Step 2: Observe ARP and Ping Packets

 * From "Router-1" ping "Node-1" 5 times:

   ```    
   # Send 5 ping packets to 10.0.0.2
   ping -c 5 10.0.0.2
   ```

   > ##### Challenge 1.1
   > Identify the ARP and Ping packets captured by Wireshark. Briefly explain the flow of the ARP and Ping packets that have been sent and received as an effect of the ping-command:
   > ```
   > The router with the ip address 10.0.0.1 sends a broadcast request with its mac address and ip address asking for the mac address of the interface with ip 10.0.0.2. The destination mac address is set to all 1's. The host replies with another arp message with its own ip address (10.0.0.2) and MAC address that the router uses to send ping messages. 
   >
   >
   >
   >
   >
   > ```

### Step 3: Setup DHCP Server

 * Create the file /etc/dhcpd.conf on "Router-1" and add the content:

    ```
    # For more info look in /etc/dhcp/dhcpd.conf.example
    default-lease-time 600;
    max-lease-time 7200;
    subnet 10.0.0.0 netmask 255.255.255.0 {
        range 10.0.0.3 10.0.0.255;
    }
    ```

 * This tells the DHCP server to configure the network 10.0.0.0 with the netmask 255.255.255.0 such that addresses ranging from 10.0.0.3 to 10.0.0.255 are assigned to Nodes.
 * Start the DHCP server:

    ```
    # man https://linux.die.net/man/8/dhcpd
    /usr/sbin/dhcpd -4 -f -d eth0
    ```

### Step 4: Automatic IPv4 Address Configuration

 * Login to "Node-2", and run the command

    ```
    # Start the DHCP client for autoconfiguration of eth0.
    dhclient eth0
    ```

    > ##### Challenge 1.2
    > Identify the DHCP packets by using packet capture, and briefly explain the exchange of DHCP packets. What IPv4 address has been assigned to "Node-2"?
    > ```
    > At first client sends a DHCP discover message with source ip 0.0.0.0 and destination ip 255.255.255.255 (broadcast) and gets the ip address 10.0.0.5 back from the DHCP server as a DHCPOFFER message. Afterwards, host-2 requests the new ip address from the DHCP server with a DHCPREQUEST message and gets a DHCPACK message back. Interface eth0 of host-2 now has the ip address 10.0.0.5
    
    Node-2 sends a request with source ip address 0.0.0.0 to the destination ip address 255.255.255.255 (broadcast) to get an ip address. The DHCP server (router-2) replies with the ip address 10.0.0.4 in the YOUR IP ADDRESS field (this is also the address used as destination address). 
    >
    >
    >
    >
    >
    > ```

  * Finally, try to ping both "Node-1" and "Router-1", in order to see if the network has been configured successfully. If all nodes can
  ping each other you have successfully completed Part 1.

## Part 2: IPv6 autoconfiguration 

The objective of these experiments is to observe how stateless IPv6 autoconfiguration and Neighbour Discovery Protocol (NDP) works.
Start this part by opening the project that was created in the beginning of the workshop and connect WireShark to the switch (or a hub).
The experiments consists of the three steps below. When asked to fill in information, do so before continuing to the next step.

### Step 1: Configure IPv6 router with autoconfiguration

In order to see how IPv6 autoconfiguration works with a router present, we need to configure the Router Advertisement Daemon (radvd) on the router node:

 * Login to the "Router-1" node and assign an IPv6 address to eth0 of the router by issuing the following command:

    ```
    ip addr add 2001:16d8:dd92:1001::1/64 dev eth0
    ip link set eth0 up
    ```

 * Edit /etc/radvd.conf using the _nano editor_ (or vim if you prefer), so that it contains the following:

    ```
    interface eth0 {
        AdvSendAdvert on;
        prefix 2001:16d8:dd92:1001::/64 {
        };
    };
    ```

    This tells the radvd daemon to send out router advertisements on interface 0, containing the prefix 2001:16d8:dd92:1001::/64.

 * Start the radvd daemon by:

    ```
    /usr/sbin/radvd -n
    ```

    In Wireshark you will now see a number of router advertisements.

### Step 2: Configure Hosts using Autoconfiguration

We now have a node on the network which acts as an IPv6 router. Next step is to see how the two hosts (Node-1 and Node-2) can be autoconfigured using the prefix advertised by the IPv6 router.

 * Login to Node-1 and bring up the eth0 interface using:

    ```
    ip link set eth0 up
    ```

 * Inspect the interface by typing:

    ```
    ip addr show eth0
    ```

    > ##### Challenge 1.3
    > What IPv6 addresses are configured for the interface prior to autoconfiguration?
    > ```
    > inet6 link local: fe80::24ed:6fff:fe0b:f2d/64
    >
    >
    > ```

 * Enable IPv6 autoconfiguration and Duplicate Address Detection by issuing the following 4 commands:

    ```
    sysctl -w net.ipv6.conf.eth0.autoconf=1
    sysctl -w net.ipv6.conf.eth0.dad_transmits=1
    sysctl -w net.ipv6.conf.eth0.accept_ra=1
    sysctl -w net.ipv6.conf.eth0.router_solicitations=3
    ```

 * Force autoconfiguration by bringing the interface down and up again:

    ```
    ip link set eth0 down
    ip link set eth0 up
    ```

 * Inspect Wireshark  
 * Inspect the interface by typing:

    ```
    ip addr show eth0
    ```

    > ##### Challenge 1.4
    > What IPv6 addresses are configured for eth0?
    > ```
    > inet6 global: 2001:16d8:dd92:1001:24ed:6fff:fe0b:f2d/64
    > 
    >
    > ```

    > ##### Challenge 1.5
    > What is the IPv6-network address (prefix) announced by the Router? Explain how you can identify this address?
    > ```
    > The prefix is 2001:16d8:dd92:1001::/64 and is advertised in the Router Advertisement message broadcasted to all hosts from the router. This message is the responds to the Router solicitation message broadcasted to routers from node-1.
    >
    >
    > ```

    > ##### Challenge 1.6
    > Briefly explain the ICMPv6 packets capture by Wireshark. You can ignore packets destined for ff02::16.
    > ```
    > Node-1 broadcasts a Router Solicitation message to all routers with its MAC address. Router-1 responds with a Router Advertisement message with the network prefix and its MAC address.
    >
    > 
    >
    >
    >
    >
    >
    > ```

 * Ping the router (2001:16d8:dd92:1001::1):

    ```
    ping6 -c 5 2001:16d8:dd92:1001::1
    ```

    > ##### Challenge 1.7
    > Briefly explain the ICMPv6 packets exchanged.
    > ```
    > Node-1 sends a Neighbor Solicitation message to Router-1 with its MAC address using the solicited-node multicast address of the router. Router-1 responds with a Neighbor Advertisement message with its MAC address that can be used to send the ping messages. Afterwards, a Neighbor Solicitation message is send from router-1 to node-1 with a subsequent responds from node-1 with a Neighbor Advertisement. This happens a couple of times in both directions. We cannot explain why this happens after the ping messages have already been sent.
    >
    >
    >
    >
    >
    >
    >
    > ```

### Step 3: IPv6 address calculation

Having configured one of the hosts using IPv6 autoconfiguration, the final step is to predict the IPv6 address that will be assigned to the other node. Recall that the EUI-64 identifier is calculated from the MAC-address by taking the lower 24-bits and the upper 24-bits and inserting FF:FE between them and inverting the UL-bit. The EUI-64 identifier is then used together with the network prefix to construct the global unicast address.

An example:

    MAC address:    00:B0:D0:66:6F:54
    Network prefix: 2001:1:1:1::/64
    EUI-64:         02:B0:D0:FF:FE:66:6F:54
    Global unicast: 2001:1:1:1:02B0:D0FF:FE66:6F54

 * Login to Node-2 and obtain the MAC-address by running
    ```
    ip link show eth0
    ```

   > ##### Challenge 1.8
   > Calculate the global unicast IPv6 address from the MAC-address and the prefix which the router is configured for.
   > ```
   > MAC: 92:a6:c6:b6:bd:b8
   >
   > Network Prefix: 2001:16d8:dd92:1001::/64
   >
   > EUI-64: 90:a6:c6:ff:fe:b6:bd:b8
   >
   > Global Unicast: 2001:16d8:dd92:1001:90a6:c6ff:feb6:bdb8
   >
   > ```

 * Enable IPv6 autoconfiguration and Duplicate Address Detection by issuing the following 4 commands:

    ```
    sysctl -w net.ipv6.conf.eth0.autoconf=1
    sysctl -w net.ipv6.conf.eth0.dad_transmits=1
    sysctl -w net.ipv6.conf.eth0.accept_ra=1
    sysctl -w net.ipv6.conf.eth0.router_solicitations=3
    ```

 * Force autoconfiguration by bringing the interface down and up again:

    ```
    ip link set eth0 down
    ip link set eth0 up
    ```

 * Inspect the assigned IPv6-addresses. Does the address configured for the interface match the one calculated? If not try to calculate it again.

    ```
    ip addr show eth0
    ```

 * As a final test, ping the node which was autoconfigured using the command (replacing the x's with the address):

    ```
    ping6 -c 5 xxxx:xxxx:xxxx:xxxx::x
    ```
