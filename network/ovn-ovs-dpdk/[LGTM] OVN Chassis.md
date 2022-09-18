## [LGTM] OVN Chassis

## Chassis作用

The ovn-chassis charm provides the Open Virtual Network (OVN) local controller, Open vSwitch Database and Switch. It is used in conjunction with the [ovn-central](https://jaas.ai/ovn-central) charm.

Open vSwitch bridges for integration, external Layer2 and Layer3 connectivity is managed by the charm.

`Chassis` 可以是 `HV`，也可以是 `VTEP` 网关。 `Chassis` 的信息保存在 `Southbound DB` 里面，由 `ovn-controller/ovn-controller-vtep` 来维护。

## Chassis ：br-int,下面是谁的机翻

The customary name for the integration bridge is  br-int,  but  another name may be used.



Each chassis in an OVN deployment  must  be  configured  with  an  Open vSwitch  bridge dedicated for OVN’s use, called the integration bridge.System startup  scripts  may  create  this  bridge  prior  to  starting
**ovn-controller**  if desired. If this bridge does not exist when ovn-controller starts, it will be created automatically with the default  configuration  suggested  below.  The  ports  on  the  integration  bridge
include:

> The  ovn-controller  process  on  each  chassis receives the updated southbound database, with the updated  nb_cfg.

* On any chassis, tunnel ports that OVN  uses  to  maintain logical   network   connectivity.   ovn-controller  adds, updates, and removes these tunnel ports.

* On a hypervisor, any VIFs that are to be attached to logical  networks.  For instances connected through software emulated ports such as TUN/TAP or VETH pairs, the  hypervisor  itself  will  normally  create ports and plug them into the  integration  bridge.  For  instances  connected through  representor  ports, typically used with hardware offload, the ovn-controller may on CMS direction  consult a  VIF plug provider for representor port lookup and plug them into the integration bridge (please refer  to  Docu‐
  mentation/top‐ics/vif-plug-providers/vif-plug-providers.rst for more information). In  both  cases  the  conventions described  in Documentation/topics/integration.rst in the Open vSwitch source tree is followed  to  ensure  mapping between  OVN  logical port and VIF. (This is pre-existing integration work that has already been done  on  hypervisors that support OVS.)
* On  a gateway, the physical port used for logical network connectivity. System startup scripts add this port to the bridge  prior  to  starting ovn-controller. This can be a patch port to another bridge, instead of a physical port,in more sophisticated setups.

Other  ports  should not be attached to the integration bridge. In particular, physical ports attached to the underlay network (as opposed to gateway  ports,  which are physical ports attached to logical networks)
must not be attached to the integration bridge. Underlay physical ports should instead be attached to a separate Open vSwitch bridge (they need not be attached to any bridge at all, in fact).

OVN部署中的每个Chassis都必须配置一个专门用于ovn的Open vSwitch网桥，称为`集成网桥(integration bridge)`。如果需要，系统启动脚本可以在启动OVN控制器之前创建此网桥。

> System startup scripts add this port to the bridge  prior  to  starting ovn-controller.

当OVN控制器启动时此网桥不存在的话，就将使用下面建议的默认配置来自动创建它。

集成网桥上的端口包括：

- 任何机箱上，OVN用来保持逻辑网络连接的隧道（tunnel）端口:

  OVN控制器将添加、更新和删除这些隧道端口。

- 在hypervisor中，任何要连接到逻辑网络的VIF:
  hypervisor本身，或者vSwitch和hypervisor之间的集成（在integrationguide.rst中描述）会负责处理这个问题。

> 这不是OVN的一部分，也不是OVN的新内容；这些都是预存在的集成工作，早已经在支持OVS的 hypervisor上实现了。

- 在网关上，用于逻辑网络连接的物理端口：
  系统启动脚本在启动ovn控制器之前将此端口添加到网桥。
  在更复杂的设置中，也可能是是另一个网桥的补丁端口（patch port），而不是物理端口。

其他端口不应连接到集成网桥上。
尤其是，附加到底层网络（underlay network）的物理端口（与附加到逻辑网络物理端口的网关端口相反）不能连接到集成网桥。
底层物理端口应该连接到单独的Open vSwitch网桥（事实上，它们根本不需要连接到任何网桥）。

集成网桥应该按照下面描述的方式配置。每个配置的效果都记录在`ovs-vswitchd.conf.db`中：

```mipsasm
fail-mode=secure
    避免在OVN控制器启动之前在隔离的逻辑网络之间交换数据包。
    详细信息请参阅ovs vsctl中的 控制器故障设置。
    
other-config:disable-in-band=true
    抑制集成网桥的带内控制流(in-band  control  flows)。
    由于OVN使用本地控制器（通过Unix域套接字）而不是远程控制器，所以这样的控制流显然是不常见的。
    然而，对于同一系统中的某个其他网桥可能有一个带内（in-band）远程控制器，在这种情况下，这可能对带内控制器正常建立的流量进行抑制。
    请参阅有关文档以获取更多信息。
```

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