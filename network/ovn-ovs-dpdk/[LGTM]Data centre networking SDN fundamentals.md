## [LGTM]Data centre networking: SDN fundamentals

![img](https://secure.gravatar.com/avatar/4e4228f082a34fa7e679708bc66f043d?s=96&d=mm&r=g)

### [Fouaz Bouguerra](https://ubuntu.com/blog/author/fbouguerr)

on 11 October 2021

- Share on:
-  

- [Facebook](https://www.facebook.com/sharer/sharer.php?u=https://www.ubuntu.com/blog/data-centre-networking-sdn-fundamentals)
-  

- [Twitter](https://twitter.com/share?text=Data centre networking%3A SDN fundamentals&url=https://www.ubuntu.com/blog/data-centre-networking-sdn-fundamentals&hashtags=ubuntu)
-  

- [LinkedIn](https://www.linkedin.com/shareArticle?mini=true&url=https://www.ubuntu.com/blog/data-centre-networking-sdn-fundamentals&title=Data centre networking%3A SDN fundamentals)

**Tags:** [data center ](https://ubuntu.com/blog/tag/data-center), [Data center networking ](https://ubuntu.com/blog/tag/data-center-networking), [model-driven networking ](https://ubuntu.com/blog/tag/model-driven-networking), [Private cloud ](https://ubuntu.com/blog/tag/private-cloud), [sdn ](https://ubuntu.com/blog/tag/sdn), [software-defined networking ](https://ubuntu.com/blog/tag/software-defined-networking), [Virtualisation](https://ubuntu.com/blog/tag/virtualisation)

![img](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_720/https://ubuntu.com/wp-content/uploads/65e2/pexels-manuel-geissinger-325229.jpg)

This blog post is part of our data centre networking series:

- [Data centre networking : What is SDN](https://ubuntu.com/blog/data-centre-networking-what-is-sdn)
- **Data centre networking : SDN fundamentals**
- [Data centre networking : SDDC](https://ubuntu.com/blog/data-centre-networking-software-defined-data-centre)
- [Data centre networking : What is OVS](https://ubuntu.com/blog/data-centre-networking-what-is-ovs)
- [Data centre networking : What is OVN](https://ubuntu.com/blog/data-centre-networking-what-is-ovn)
- [Data centre networking : SmartNICs](https://ubuntu.com/blog/data-centre-networking-smartnics)

In the precedent [blog](https://ubuntu.com/blog/data-centre-networking-what-is-sdn), we provided an introduction to **Software-Defined Networking (SDN)** and the main reasons which compelled the industry to adopt it. We’ve seen how impactful it can be to leverage scalable automation and the power of software to define and run key networking components. Being sufficiently granular to address functions such as switching, routing, security and QoS, provides strong benefits to the organisations’ IT teams. Going further, making those networking functions model-driven and tied to the end-user’s application intent will highly improve the business applications’ outcomes. In this blog, we will cover the principal components of SDN, its architecture and different types.

## Software-defined networking architecture

The SDN network components can be represented into three layers:

- **The application layer**, which provides network “applications” that can be used on demand by users to transmit information about network-specific requests.
- **The control layer**, which includes the SDN controllers and interacts with the applications to provide the destination of the data traffic.
- **The infrastructure layer**, which includes all the SDN networking devices responsible for the traffic forwarding.

![img](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_585,h_310/https://lh4.googleusercontent.com/hCdg_uWOUVoBbUA8E6YEtwMsXYKO6VlBenisqrEWN-6D1yDKGQyF1YsDZdUSFiqMA1zPDBAxyBsfuHnt6rWYbHK5ISexbEii42PNKTPfA3hkSspM3iTl8GdYVT8b99Ax1PyHBmXy=s0)

The communication between the different layers is done using [OpenFlow](https://en.wikipedia.org/wiki/OpenFlow), a standardised and programmable networking protocol used in SDN to direct traffic and exchange information between the middle and bottom layers.

A management and orchestration plan is also necessary to manage this infrastructure. It allows the configuration of the SDN controller and the initialisation of the “network elements”:

## SDN controllers

The architecture of an SDN controller includes the following components:

![img](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_375,h_301/https://lh5.googleusercontent.com/DUUv5_jYfs4bKWGFzgv-DDo_56h-W475K3CNDmxMOwtk3Dkp3md-0lY6MCLgmY8cWgX6bGmJWdkPhyuhcYC-Liu6mhtmSBDJwDdjAcE3Iv-pNv3KwTftEp87YIMhBXRoTuU2o9VB=s0)

- The Northbound interface: It is not standardised and can take different forms. APIs using languages such as Java, Python, REST (Representational State Transfer) are most often available and allow applications to interact with the SDN network.
- The Southbound interface: It is standardized and uses the OpenFlow protocol. Other protocols like SNMP, CLI can be implemented in “closed SDN” solutions.
- Modules for managing network components (flow table, topology, a network element discovery module, an SDN network element management module, a table of usage statistics …etc.)

SDN leverages the usage of “standardised” common off-the-shelf switches which are controlled by an SDN controller. This takes the computational complexity away from the SDN network devices, and makes them more “commoditised”, which is a synonym for lower costs.

![img](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_720/https://lh5.googleusercontent.com/kBu5LtTa4tUeWTaPlSd3xQ06MbdQV-e6o-b65DVCos3kSYbRkCeW47vP1sIJLqi2FUwhGk82Zr9lJoazBbiTlho2lsmGatDw8dtEEoX1AN0PECTEdwkoTiTQF9Cw6G3wGlnzCCe1=s0)

The SDN Controller builds a global view of the network so that it can make decisions about routing flows from one point of the network to another. It communicates with applications (which will make requests to open and close flows) via APIs on its Northbound interface. It controls the network nodes (SDN devices) via its Southbound interface and the OpenFlow protocol.

## One or many SDNs?

The concept of SDN has evolved from its original invention and has since seen several implementations depending on how the controller layer is connected to the SDN devices. There are four types of SDN which we can classify as follows :

- **Open SDN**: Open SDN has a centralised control plane and uses OpenFlow for its southbound API. It requires evolutionary, hybrid deployment strategies to succeed. ONOS and BigSwitch are good examples of its implementation.

- **Overlay-based SDN**: It operates on an overlay network and represents a practical solution to solve data centre connectivity issues, but doesn’t address physical networks underneath. As an example of this implementation, we’ll find: Juniper Contrail, NEC VTN and NSX (VMWare).

- **API-based SDN** (or unopened SDN): The focus here is on network programming. It uses NetConf / OpFlex / SSH for southbound APIs to allow programming of network nodes including existing hardware infrastructure. OpenDaylight, APIC-DC,EM and Tail-f (NSO) are examples of its implementation.
- The focus here is on network programming. It uses NetConf / OpFlex / SSH for southbound APIs to allow programming of network nodes including existing hardware infrastructure. OpenDaylight, APIC-DC,EM and Tail-f (NSO) are examples of its implementation.

- **Automation-based SDN** (or hybrid SDN): It covers SDN-enabled as well as legacy networking equipment. One of its goals is to reduce the cost of network components. It uses automation tools (agents, Python, etc.) and generic white label components supporting several types of OS and leveraging ONIE (Open Network Install Environment)

![img](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_720/https://lh3.googleusercontent.com/hm4k0qlNatXw8o7KISLiCw9h7_hPDljiaejDGpE0-adWhG3Nlt2UquHsk2jDcrc8Bu7dieMgpCk9GFury9l5uLUjRyGdFogyv6xUiiSyyiVPow4GfrsRLy11JIyLv5y8vP1cRXYX=s0)

The four SDN families

Among the widely deployed SDNs in the [private cloud](https://ubuntu.com/cloud/private-cloud) infrastructure arena, there is open source SDN as part of OpenStack, which supports the best of breed of overlay protocols such as GENEVE and VXLAN. Canonical’s [Charmed Openstack](https://ubuntu.com/openstack) uses Neutron to provide network connectivity between OpenStack instances, enabling multi-VM deployments. For this purpose, Neutron supports various software defined networking technologies, including Open Virtual Network (OVN), Open vSwitch (OVS), Juniper Contrail, Cisco ACI, etc. In addition to carrying the fundamentals of SDN, Charmed OpenStack is also integrated in the application’s larger automated deployment and life-cycle management ecosystem, provided by [Juju](https://juju.is/).

SDN hasn’t stopped developing and there are many efforts and open source projects which work to address the ever evolving networking landscape, influenced by the ever changing application space and user expectations. The journey is certainly long but exciting to get to model-driven and why not self-healing networks. 

