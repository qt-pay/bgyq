## [LGTM] OVN Chassis

## LGTM

https://www.mankier.com/7/ovn-architecture



## What is Chassis:+1:

Hypervisors and gateways are together called *transport node* or ***chassis***.

Each chassis has a bridge mapping for one of the **localnet** physical networks only.

An OVN *logical network* is a network implemented in software that is insulated from physical (and thus virtual) networks by tunnels or other encapsulations. 

```bash
$ ovn-sbctl show
Chassis local_chassis
    hostname: TJ-UNISTACK-YFB-SEC01
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
$ ovn-sbctl chassis-add fakechassis geneve 127.0.0.1
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}


$ ovn-nbctl ls-add sw0
$ ovn-nbctl lsp-add sw0 sp0
$ ovn-nbctl show
    switch d02ad16a-2b74-434e-a26f-aa45ad698b4e (sw0)
        port sp0
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
        
#  bind logical port PORT to CHASSIS
$ ovn-sbctl lsp-bind sp0 fakechassis
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
    # new binding
    Port_Binding "sp0"
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
```



### localnet

  Introduce a new logical port type called "localnet".  A logical port with this type also has an option called "network_name".  A "localnet" logical port represents a connection to a network that is locally accessible from each chassis running ovn-controller.  ovn-controller will use the ovn-bridge-mappings configuration to figure out which patch port on br-int should be used for this port.

https://patchwork.ozlabs.org/project/openvswitch/patch/1441298701-32163-3-git-send-email-rbryant@redhat.com/

### gateway

Gateways provide limited connectivity between logical networks and physical ones.

#### vtep gateway

A `VTEP gateway` connects an OVN logical network to a physical (or virtual) switch that implements the OVSDB VTEP schema that accompanies Open vSwitch. (The `VTEP gateway` term is a misnomer, since a VTEP is just a VXLAN Tunnel Endpoint, but it is a well established name.) 

> misnomer: a name that does not suit what it refers to, or the use of such a name.
>
> vtep gateway，名字起的多少有点问题，它就是一个vxlan tunnel endpoint，用来连接ovn和physical switch.

The main intended use case for VTEP gateways is to attach physical servers to an OVN logical network using a physical top-of-rack switch that supports the OVSDB VTEP schema

#### L2 gateway

A L2 gateway simply attaches a designated physical L2 segment available on some chassis to a logical network. The physical network effectively becomes part of the logical network.

To set up a L2 gateway, the CMS adds an **l2gateway** LSP to an appropriate logical switch, setting LSP options to name the chassis on which it should be bound. **ovn-northd** copies this configuration into a southbound **Port_Binding** record. On the designated chassis, **ovn-controller** forwards packets appropriately to and from the physical segment.

L2 gateway ports have features in common with **localnet** ports. However, with a **localnet** port, the physical network becomes the transport between hypervisors. With an L2 gateway, packets are still transported between hypervisors over tunnels and the **l2gateway** port is only used for the packets that are on the physical network. The application for L2 gateways is similar to that for VTEP gateways, e.g. to add non-virtualized machines to a logical network, but L2 gateways do not require special support from top-of-rack hardware switches.

#### L3 gateway router

OVN supports L3 gateway routers, which are OVN logical routers that are implemented in a designated chassis. Gateway routers are typically used between distributed logical routers and physical networks. The distributed logical router and the logical switches behind it, to which VMs and containers attach, effectively reside on each hypervisor.

### Hypervisors 

A hypervisor, also known as a virtual machine monitor or VMM, is software that creates and runs virtual machines (VMs). 

## Chassis作用

`Chassis` 可以是 `HV`，也可以是 `VTEP` 网关（vxlan tunnel endpoint for Hybird Overlay）。 `Chassis` 的信息保存在 `Southbound DB` 里面，由 `ovn-controller/ovn-controller-vtep` 来维护。

* ovn-controller:  ovn-controller is the local controller daemon for OVN, the Open Virtual Network. It connects up to the OVN Southbound database vSwitch database  over the OVSDB  protocol and to ovs-vswitchd **via OpenFlow.**  

   Each hypervisor and  software gateway in an OVN deployment runs its own independent copy of ovn-controller;

* ovn-controller-vtep:   ovn-controller-vtep is the local controller daemon in OVN, the Open Virtual Network, for VTEP enabled physical switches. It connects up to the OVN Southbound database over the OVSDB protocol, and down to the VTEP database.
   over the OVSDB protocol.

### vtep

vtep - hardware_vtep database schema
This schema specifies relations that a VTEP can use to integrate physical ports into logical switches maintained by a network virtualization controller such as NSX

## Chassis ：br-int

Each chassis has a bridge mapping for one of the **localnet** physical networks only.

Each chassis in an OVN deployment  must  be  configured  with  an  Open vSwitch  bridge dedicated for OVN’s use, called the **integration bridge**.System startup  scripts  may  create  this  bridge  prior  to  starting
**ovn-controller**  if desired. If this bridge does not exist when ovn-controller starts, it will be created automatically with the default  configuration  suggested  below.  The  ports  on  the  integration  bridge
include:

> The  ovn-controller  process  on  each  chassis receives the updated southbound database, with the updated  nb_cfg.

* On any chassis, tunnel ports that OVN  uses  to  maintain logical   network   connectivity.   ovn-controller  adds, updates, and removes these tunnel ports.

* On a hypervisor, any VIFs that are to be attached to logical  networks.  For instances connected through software emulated ports such as TUN/TAP or VETH pairs, the  hypervisor  itself  will  normally  create ports and plug them into the  integration  bridge.  For  instances  connected through  representor  ports, typically used with hardware offload, the ovn-controller may on CMS direction  consult a  VIF plug provider for representor port lookup and plug them into the integration bridge (please refer  to  Docu‐
  mentation/top‐ics/vif-plug-providers/vif-plug-providers.rst for more information). In  both  cases  the  conventions described  in Documentation/topics/integration.rst in the Open vSwitch source tree is followed  to  ensure  mapping between  OVN  logical port and VIF. (This is pre-existing integration work that has already been done  on  hypervisors that support OVS.)
* On  a gateway, the physical port used for logical network connectivity. System startup scripts add this port to the bridge  prior  to  starting ovn-controller. This can be a patch port to another bridge, instead of a physical port,in more sophisticated setups.

Other  ports  should not be attached to the integration bridge. In particular, physical ports attached to the underlay network (as opposed to gateway  ports,  which are physical ports attached to logical networks)
must not be attached to the integration bridge. Underlay physical ports should instead be attached to a separate Open vSwitch bridge (they need not be attached to any bridge at all, in fact).

The integration bridge should be configured as described below. The effect of each of these settings is documented in [ovs-vswitchd.conf.db(5)](https://www.mankier.com/5/ovs-vswitchd.conf.db):

- **fail-mode=secure**

  Avoids switching packets between isolated logical networks before **ovn-controller** starts up. See **Controller Failure Settings** in [ovs-vsctl(8)](https://www.mankier.com/8/ovs-vsctl) for more information.

- **other-config:disable-in-band=true**

  Suppresses in-band control flows for the integration bridge. It would be unusual for such flows to show up anyway, because OVN uses a local controller (over a Unix domain socket) instead of a remote controller. It’s possible, however, for some other bridge in the same system to have an in-band remote controller, and in that case this suppresses the flows that in-band control would ordinarily set up. Refer to the documentation for more information.

The customary name for the integration bridge is  br-int,  but  another name may be used.



### port_binding

ovn-controller will then set up the flows necessary for these ports to be able to communicate each other as defined by the OVN logical topology.

```bash
# lsp-bind logical-port chassis Binds the logical port named logical-port to chassis
$ ovn-sbctl [--may-exist] lsp-bind logical-port chassis
$ ovn-sbctl [--if-exists] lsp-unbind logical-port
```

除了vm/container port， attached underlay network physical port, tunnel port ，

Other  ports  should not be attached to the integration bridge.

### east-west traffic 

The  following happens when a VM sends an east-west traffic which needs to be routed:

1.  The packet first  enters  the  ingress  pipeline,  and  then egress  pipeline of the source localnet logical switch datapath and is sent out via  a  localnet  port  of  the  source localnet  logical  switch  (instead  of sending it to router pipeline).
    
2.  The gateway chassis receives the packet via a localnet  port of  the  source  localnet logical switch and sends it to the integration bridge. The packet then enters the ingress pipeline,  and then egress pipeline of the source localnet logical switch datapath and enters the ingress pipeline  of  the logical router datapath.
    
3.  Routing decision is taken.

4.  From the router datapath, packet enters the ingress pipeline and then egress pipeline of the destination localnet logical switch  datapath. It then goes out of the integration bridge to the provider bridge ( belonging to the destination  logical switch) via a localnet port.
    
5.  The  destination  chassis receives the packet via a localnet port and sends it to  the  integration  bridge.  The  packet enters  the ingress pipeline and then egress pipeline of the destination localnet logical switch and finally delivered to the destination VM port.

### 配置demo

https://www.openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html

在github上将 ovs的分支切换到2.5就能看到shell script

## OVN Southbound DB 

OVN Southbound DB 包括如下表：

The  following  list  summarizes  the  purpose of each of the tables in the OVN_Southbound database.  Each table is described in more detail on a later page.

| Table            | Purpose                                             |
| ---------------- | --------------------------------------------------- |
| SB_Global        | Southbound configuration                            |
| Chassis          | Physical Network Hypervisor and Gateway Information |
| Encap            | Encapsulation Types                                 |
| Address_Set      | Address Sets                                        |
| Logical_Flow     | Logical Network Flows                               |
| Multicast_Group  | Logical Port Multicast Groups                       |
| Datapath_Binding | Physical-Logical Datapath Bindings                  |
| Port_Binding     | Physical-Logical Port Bindings                      |
| MAC_Binding      | IP to MAC bindings                                  |
| DHCP_Options     | DHCP Options supported by native OVN DHCP           |
| DHCPv6_Options   | DHCPv6 Options supported by native OVN DHCPv6       |
| Connection       | OVSDB client connections.                           |
| SSL              | SSL configuration.                                  |
| DNS              | Native DNS resolution                               |
| RBAC_Role        | RBAC_Role configuration.                            |
| RBAC_Permission  | RBAC_Permission configuration.                      |
| Gateway_Chassis  | Gateway_Chassis configuration.                      |

### chassis table

在物理网络中每一行表示一个 hypervisor 或者网关，每一个 chassis 通过 ovn-controler/ovn-controller-vtep 添加和更新自己的行，并保留剩余行的拷贝来确定如何访问其他的 hppervisor。

```bash
$ ovn-sbctl list Chassis
_uuid               : f20ae8d7-800a-46dd-adb2-c2f279cb4456
encaps              : [e94ba8c3-7648-41eb-94cf-5615d83c8182]
external_ids        : {}
hostname            : ""
name                : br-test
nb_cfg              : 0
vtep_logical_switches: []

```

### ovn-sbctl chassis-add:fire:

The `ovn-sbctl` utility can be used to see into the state stored in the `OVN_Southbound` database. 

```bash
# chassis-add chassis encap-type encap-ip  Creates  a  new  chassis  named  chassis. 
$ ovn-sbctl chassis-add br-test geneve 172.21.170.84
$ ovn-sbctl show
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
```

end



## ovs add chassis?:confused:

Let us create an *Integration Bridge* with the Name OVN on kvmhost01 ad kvmhost03:

```bash
## OVN is bridge name. It always is br-int
$ ovs-vsctl add-br OVN -- set Bridge OVN fail-mode=secure
```

We now tell the Hosts, how the *ovn-central* system can be reached and which *Integration Bridge* should be used:

```bash
id_file=/etc/openvswitch/system-id.conf
test -e $id_file || uuidgen > $id_file

ovs-vsctl set open . external_ids:system-id=$(cat $id_file)
ovs-vsctl set open . external_ids:ovn-nb="tcp:172.18.31.1:6641"
ovs-vsctl set open . external-ids:ovn-remote="tcp:172.18.31.1:6642"
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.31.10
ovs-vsctl set open . external_ids:ovn-bridge=OVN
```

On Arch Linux the Package doesn’t generate an system-id.conf file with an system wide uuid, so the first lines will create them if it isn’t already there. The above settings are made on kvmhost01, be sure to change the *ovn-encap-ip* to your Overlay IP of the Host. And start the *ovn-controller* after configure the settings above:

```
/usr/share/openvswitch/scripts/ovn-ctl start_controller
```

If all went well, you can see on your *ovn-central* system the new Chassis with `ovn-sbctl show`:

```
Chassis "bffe235d-b222-49ce-9723-180ad5ba8b90"
    hostname: "kvmhost01"
    Encap geneve
	ip: "172.18.31.10"
	options: {csum="true"}
```

Just repeat the settings on all of your participating hosts and `ovn-sbctl show` and with our lab setup, it should look like:

```
Chassis "bffe235d-b222-49ce-9723-180ad5ba8b90"
    hostname: "kvmhost01"
    Encap geneve
	ip: "172.18.31.10"
	options: {csum="true"}
Chassis "804c7da4-04c8-416e-9420-0345f7335284"
    hostname: "kvmhost03"
    Encap geneve
	ip: "172.18.31.30"
	options: {csum="true"}
```

end

## [Life Cycle of a VTEP gateway](https://www.mankier.com/7/ovn-architecture#Description-Life_Cycle_of_a_VTEP_gateway)

A gateway is a chassis that forwards traffic between the OVN-managed part of a logical network and a physical VLAN, extending a tunnel-based logical network into a physical network.

The steps below refer often to details of the OVN and VTEP database schemas. Please see [ovn-sb(5)](https://www.mankier.com/5/ovn-sb), [ovn-nb(5)](https://www.mankier.com/5/ovn-nb) and [vtep(5)](https://www.mankier.com/5/vtep), respectively, for the full story on these databases.

1. A VTEP gateway’s life cycle begins with the administrator registering the VTEP gateway as a **Physical_Switch** table entry in the **VTEP** database. The **ovn-controller-vtep** connected to this VTEP database, will recognize the new VTEP gateway and create a new **Chassis** table entry for it in the **OVN_Southbound** database.
2. The administrator can then create a new **Logical_Switch** table entry, and bind a particular vlan on a VTEP gateway’s port to any VTEP logical switch. Once a VTEP logical switch is bound to a VTEP gateway, the **ovn-controller-vtep** will detect it and add its name to the *vtep_logical_switches* column of the **Chassis** table in the **OVN_Southbound** database. Note, the *tunnel_key* column of VTEP logical switch is not filled at creation. The **ovn-controller-vtep** will set the column when the correponding vtep logical switch is bound to an OVN logical network.
3. Now, the administrator can use the CMS to add a VTEP logical switch to the OVN logical network. To do that, the CMS must first create a new **Logical_Switch_Port** table entry in the **OVN_Northbound** database. Then, the *type* column of this entry must be set to "vtep". Next, the *vtep-logical-switch* and *vtep-physical-switch* keys in the *options* column must also be specified, since multiple VTEP gateways can attach to the same VTEP logical switch. Next, the *addresses* column of this logical port must be set to "unknown", it will add a priority 0 entry in "ls_in_l2_lkup" stage of logical switch ingress pipeline. So, traffic with unrecorded mac by OVN would go through the **Logical_Switch_Port** to physical network.
4. The newly created logical port in the **OVN_Northbound** database and its configuration will be passed down to the **OVN_Southbound** database as a new **Port_Binding** table entry. The **ovn-controller-vtep** will recognize the change and bind the logical port to the corresponding VTEP gateway chassis. Configuration of binding the same VTEP logical switch to a different OVN logical networks is not allowed and a warning will be generated in the log.
5. Beside binding to the VTEP gateway chassis, the **ovn-controller-vtep** will update the *tunnel_key* column of the VTEP logical switch to the corresponding **Datapath_Binding** table entry’s *tunnel_key* for the bound OVN logical network.
6. Next, the **ovn-controller-vtep** will keep reacting to the configuration change in the **Port_Binding** in the **OVN_Northbound** database, and updating the **Ucast_Macs_Remote** table in the **VTEP** database. This allows the VTEP gateway to understand where to forward the unicast traffic coming from the extended external network.
7. Eventually, the VTEP gateway’s life cycle ends when the administrator unregisters the VTEP gateway from the **VTEP** database. The **ovn-controller-vtep** will recognize the event and remove all related configurations (**Chassis** table entry and port bindings) in the **OVN_Southbound** database.
8. When the **ovn-controller-vtep** is terminated, all related configurations in the **OVN_Southbound** database and the **VTEP** database will be cleaned, including **Chassis** table entries for all registered VTEP gateways and their port bindings, and all **Ucast_Macs_Remote** table entries and the **Logical_Switch** tunnel keys.