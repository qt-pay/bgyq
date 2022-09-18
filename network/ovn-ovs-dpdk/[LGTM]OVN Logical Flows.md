## [LGTM]OVN Logical Flows 

## 原文

https://blog.russellbryant.net/2016/11/11/ovn-logical-flows-and-ovn-trace/

You can use the `ovn-nbctl` utility to see an overview of the logical topology.

The `ovn-sbctl` utility can be used to see into the state stored in the `OVN_Southbound` database. 

## OpenFlow Basics

Before getting to Logical Flows, it is helpful to have a general understanding of OpenFlow. OpenFlow is the protocol used to program the packet processing pipeline of Open vSwitch. It lets you define a series of tables with rules (flows) that contain a priority, match, and a set of actions. For each table, the highest priority (larger number is higher priority) flow that matches is executed.

Let’s imagine a trivial virtual switch with two ports, port1 and port2.

```
                        +--------+
            (1)         |        |          (2)
           port1 -------| br-int |-------- port2
                        |        |
                        +--------+
```

We can create a bridge with 2 ports using the following commands:

> The ovs-ofctl program is a command line tool for monitoring and administering OpenFlow switches. 

```bash
$ ovs-vsctl add-br br-int
$ ovs-vsctl add-port br-int port1
$ ovs-vsctl add-port br-int port2

$ ovs-vsctl show
3b1995d8-9683-45db-8929-36c62abdbd31
    Bridge br-int
        Port "port1"
            Interface "port1"
        Port br-int
            Interface br-int
                type: internal
        Port "port2"
            Interface "port2"
```

A trivial example is to define a single table where we forward all packets from port1 to port2, and all packets from port2 to port1.

```bash
    Table  Priority  Match      Actions
    -----  --------  ---------- -------
    0      0         in_port=1  output:2
    0      0         in_port=2  output:1
```

We can program this pipeline in OVS using the ovs-ofctl command:

创建一个简单的流表，定义port数据流

```bash
$ ovs-ofctl del-flows br-int
$ ovs-ofctl add-flow br-int
$ ovs-ofctl add-flow br-int "table=1, priority=0, in_port=1,actions=output:2"
$ ovs-ofctl add-flow br-int "table=1, priority=0, in_port=2,actions=output:1"

$ ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
  cookie=0x0, duration=9.679s, table=1, n_packets=0, n_bytes=0, idle_age=9, priority=0,in_port=1 actions=output:2
  cookie=0x0, duration=2.287s, table=1, n_packets=0, n_bytes=0, idle_age=2, priority=0,in_port=2 actions=output:1
```

We can extend this example to add a second table and demonstrate the use of different priorities. Let’s only allow packets from port1 if the source MAC address is 00:00:00:00:01 and only allow packets from port2 with a source MAC address of 00:00:00:00:00:02 (basic source port security). We’ll use table 0 to implement port security and then use table 1 to decide the packet’s destination.

(Yes, this could be done in a single table, but then it wouldn’t be demonstrating tables and priorities, which is the main point here.)

```bash
    Table  Priority  Match                               Actions
    -----  --------  ----------------------------------- ------------
    0      10        in_port=1,dl_src=00:00:00:00:00:01  resubmit(,1)
    0      10        in_port=2,dl_src=00:00:00:00:00:02  resubmit(,1)
    0      0                                             drop
    1      0         in_port=1                           output:2
    1      0         in_port=2                           output:1
```

Again, we can program this pipeline using the ovs-ofctl command line utility.

```bash
$ ovs-ofctl del-flows br-int
$ ovs-ofctl add-flow br-int "table=0, priority=10, in_port=1,dl_src=00:00:00:00:00:01,actions=resubmit(,1)"
$ ovs-ofctl add-flow br-int "table=0, priority=10, in_port=2,dl_src=00:00:00:00:00:02,actions=resubmit(,1)"
$ ovs-ofctl add-flow br-int "table=0, priority=0, actions=drop"
$ ovs-ofctl add-flow br-int "table=1, priority=0, in_port=1,actions=output:2"
$ ovs-ofctl add-flow br-int "table=1, priority=0, in_port=2,actions=output:1"

$ ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=72.132s, table=0, n_packets=0, n_bytes=0, idle_age=72, priority=10,in_port=1,dl_src=00:00:00:00:00:01 actions=resubmit(,1)
 cookie=0x0, duration=60.565s, table=0, n_packets=0, n_bytes=0, idle_age=60, priority=10,in_port=2,dl_src=00:00:00:00:00:02 actions=resubmit(,1)
 cookie=0x0, duration=28.127s, table=0, n_packets=0, n_bytes=0, idle_age=28, priority=0 actions=drop
 cookie=0x0, duration=13.887s, table=1, n_packets=0, n_bytes=0, idle_age=13, priority=0,in_port=1 actions=output:2
 cookie=0x0, duration=4.023s, table=1, n_packets=0, n_bytes=0, idle_age=4, priority=0,in_port=2 actions=output:1
```

Open vSwitch also provides a mechanism to trace a sample packet through a configured pipeline. Here we will trace a packet from port1 with an expected source MAC address. The output of this trace shows that the packet is resubmitted to table 1 and is then output to port 2.  The output is a bit verbose.  Just look at the “Rule” and “OpenFlow actions” lines to see which flows were executed.

通过ovs-ofctl命令行可以定制single node上port流量。

```bash
#  ovs-appctl - utility for configuring running Open vSwitch daemons
$ ovs-appctl ofproto/trace br-int in_port=1,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02 -generate
Bridge: br-int
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000

Rule: table=0 cookie=0 priority=10,in_port=1,dl_src=00:00:00:00:00:01
OpenFlow actions=resubmit(,1)

    Resubmitted flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000
    Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0 reg8=0x0 reg9=0x0 reg10=0x0 reg11=0x0 reg12=0x0 reg13=0x0 reg14=0x0 reg15=0x0
    Resubmitted  odp: drop
    Resubmitted megaflow: recirc_id=0,in_port=1,dl_src=00:00:00:00:00:01,dl_type=0x0000
    Rule: table=1 cookie=0 priority=0,in_port=1
    OpenFlow actions=output:2

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_src=00:00:00:00:00:01,dl_type=0x0000
Datapath actions: 3
```

OpenFlow can be used to build up much more complex pipelines, as well. See the [ovs-ofctl(8) man page](http://openvswitch.org/support/dist-docs/ovs-ofctl.8.html) for a lot more detail.

## OVN Logical Flows

The previous section recapped the basics of OpenFlow. It showed **how OpenFlow can be used to build packet processing pipelines of a single switch.** Manually programming these pipelines on one host, much less hundreds or thousands of hosts, can be tedious. That’s where an SDN controller that programs flows across many switches to accomplish a task is helpful. That’s the role that OVN plays for the Open vSwitch project. **OVN does all of the OpenFlow programming necessary to implement the network topologies and security policies you define using its high level configuration interface.**

How does OVN determine the flows required on each host? The central abstraction to solving this problem in OVN is Logical Flows. Logical Flows are conceptually similar to OpenFlow in that they are made up of tables of flows with a priority, match, and actions. **The major difference is that logical flows describe the detailed behavior of an entire network that can span any number of hosts.**(OpenFlow can be used to build packet processing pipelines of a single switch.) It provides us with separation between defining detailed network behavior and having to worry about the actual physical layout of the environment (how many hosts exist and which hosts ports reside on).

OVN centrally programs networks in logical flows. These logical flows are distributed throughout the whole environment to ovn-controller running on each host. **ovn-controller then knows how to compile logical flows into OpenFlow using the current state of the physical environment (what ports reside locally, and how to reach other hosts).**

Let’s create an example OVN configuration similar to the one in the section on OpenFlow basics. We will create a single OVN logical switch with two logical ports.

```bash
$ ovn-nbctl ls-add sw0

$ ovn-nbctl lsp-add sw0 sw0-port1
$ ovn-nbctl lsp-set-addresses sw0-port1 00:00:00:00:00:01
$ ovn-nbctl lsp-set-port-security sw0-port1 00:00:00:00:00:01

$ ovn-nbctl lsp-add sw0 sw0-port2
$ ovn-nbctl lsp-set-addresses sw0-port2 00:00:00:00:00:02
$ ovn-nbctl lsp-set-port-security sw0-port2 00:00:00:00:00:02

$ ovn-nbctl show sw0
    switch 48d5f699-7ffe-4627-a369-2fc905e44b32 (sw0)
        port sw0-port1
            addresses: ["00:00:00:00:00:01"]
        port sw0-port2
            addresses: ["00:00:00:00:00:02"]
```

OVN defines the logical switch, sw0, using two pipelines: an ingress pipeline and an egress pipeline. When a packet enters the network, the ingress pipeline is executed on the host where the packet originated. If the destination is on the same host, the egress pipeline will be executed as well.

```
sw0-port1 and sw0-port2 on the same host:

    +--------------------------------------------------------------------------+
    |                                                                          |
    |                               Host A                                     |
    |                                                                          |
    |   +---------+                                              +---------+   |
    |   |sw0-port1| --> ingress pipeline --> egress pipeline --> |sw0-port2|   |
    |   +---------+                                              +---------+   |
    |                                                                          |
    +--------------------------------------------------------------------------+
```

If the destination is remote, the packet will be sent over a tunnel before executing the egress pipeline on the remote host.

> `ovn-trace --detail`可以看到ingress 和egress匹配规则

```
sw0-port1 and sw0-port2 on separate hosts:

    +--------------------------------------+
    |                                      |
    |             Host A                   |
    |                                      |
    |   +---------+                        |
    |   |sw0-port1| --> ingress pipeline   |
    |   +---------+           ||           |
    |                         ||           |
    +-------------------------||-----------+
                              ||
                              \/
                         geneve tunnel
                              ||
                              ||
    +-------------------------||-----------+
    |                         ||           |
    |             Host B      ||           |
    |                         ||           |
    |   +---------+           \/           |
    |   |sw0-port2| < -- egress pipeline   |
    |   +---------+                        |
    |                                      |
    +--------------------------------------+
```

You can use the “**ovn-sbctl lflow-list**” command to view the full set of logical flows. The structure will feel somewhat familiar to OpenFlow, but there are some key differences:

1. Ports are logical entities that reside somewhere on a network, not physical ports on a single switch.

   Ovn port可以定位到这个vm在整个ovn network中的位置，而不是像传统的switch上的一个固定物理端口。

2. Each table in the pipeline is given a name in addition to its number. The name describes the purpose of that stage in the pipeline.

3. The match syntax is far more flexible. It supports complex boolean expressions and will feel very familiar to programmers.

4. The actions supported in OVN logical flows extend beyond what you would expect from OpenFlow. We are able to implement higher level features, such as DHCP, in the logical flow syntax. See the documentation for the Logical_Flow table in [ovn-sb(5)](https://www.openvswitch.org//support/dist-docs/ovsdb.5.html) for details on match and action syntax.

   ovn比OpenFlow支持更多的特性

There are several additional stages in the pipeline reserved for features not being used in this example, so the flows in many of the tables are not doing anything interesting.

```bash
 $ ovn-sbctl lflow-list
    Datapath: "sw0" (d7bf4a7b-e915-4502-8f9d-5995d33f5d10)  Pipeline: ingress
      table=0 (ls_in_port_sec_l2  ), priority=100  , match=(eth.src[40]), action=(drop;)
      table=0 (ls_in_port_sec_l2  ), priority=100  , match=(vlan.present), action=(drop;)
      table=0 (ls_in_port_sec_l2  ), priority=50   , match=(inport == "sw0-port1" && eth.src == {00:00:00:00:00:01}), action=(next;)
      table=0 (ls_in_port_sec_l2  ), priority=50   , match=(inport == "sw0-port2" && eth.src == {00:00:00:00:00:02}), action=(next;)
      table=1 (ls_in_port_sec_ip  ), priority=0    , match=(1), action=(next;)
      table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && arp.sha == 00:00:00:00:00:01), action=(next;)
      table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip6 && nd && ((nd.sll == 00:00:00:00:00:00 || nd.sll == 00:00:00:00:00:01) || ((nd.tll == 00:00:00:00:00:00 || nd.tll == 00:00:00:00:00:01)))), action=(next;)
      table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "sw0-port2" && eth.src == 00:00:00:00:00:02 && arp.sha == 00:00:00:00:00:02), action=(next;)
      table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "sw0-port2" && eth.src == 00:00:00:00:00:02 && ip6 && nd && ((nd.sll == 00:00:00:00:00:00 || nd.sll == 00:00:00:00:00:02) || ((nd.tll == 00:00:00:00:00:00 || nd.tll == 00:00:00:00:00:02)))), action=(next;)
      table=2 (ls_in_port_sec_nd  ), priority=80   , match=(inport == "sw0-port1" && (arp || nd)), action=(drop;)
      table=2 (ls_in_port_sec_nd  ), priority=80   , match=(inport == "sw0-port2" && (arp || nd)), action=(drop;)
      table=2 (ls_in_port_sec_nd  ), priority=0    , match=(1), action=(next;)
      table=3 (ls_in_pre_acl      ), priority=0    , match=(1), action=(next;)
      table=4 (ls_in_pre_lb       ), priority=0    , match=(1), action=(next;)
      table=5 (ls_in_pre_stateful ), priority=100  , match=(reg0[0] == 1), action=(ct_next;)
      table=5 (ls_in_pre_stateful ), priority=0    , match=(1), action=(next;)
      table=6 (ls_in_acl          ), priority=0    , match=(1), action=(next;)
      table=7 (ls_in_qos_mark     ), priority=0    , match=(1), action=(next;)
      table=8 (ls_in_lb           ), priority=0    , match=(1), action=(next;)
      table=9 (ls_in_stateful     ), priority=100  , match=(reg0[1] == 1), action=(ct_commit(ct_label=0/1); next;)
      table=9 (ls_in_stateful     ), priority=100  , match=(reg0[2] == 1), action=(ct_lb;)
      table=9 (ls_in_stateful     ), priority=0    , match=(1), action=(next;)
      table=10(ls_in_arp_rsp      ), priority=0    , match=(1), action=(next;)
      table=11(ls_in_dhcp_options ), priority=0    , match=(1), action=(next;)
      table=12(ls_in_dhcp_response), priority=0    , match=(1), action=(next;)
      table=13(ls_in_l2_lkup      ), priority=100  , match=(eth.mcast), action=(outport = "_MC_flood"; output;)
      table=13(ls_in_l2_lkup      ), priority=50   , match=(eth.dst == 00:00:00:00:00:01), action=(outport = "sw0-port1"; output;)
      table=13(ls_in_l2_lkup      ), priority=50   , match=(eth.dst == 00:00:00:00:00:02), action=(outport = "sw0-port2"; output;)
    Datapath: "sw0" (d7bf4a7b-e915-4502-8f9d-5995d33f5d10)  Pipeline: egress
      table=0 (ls_out_pre_lb      ), priority=0    , match=(1), action=(next;)
      table=1 (ls_out_pre_acl     ), priority=0    , match=(1), action=(next;)
      table=2 (ls_out_pre_stateful), priority=100  , match=(reg0[0] == 1), action=(ct_next;)
      table=2 (ls_out_pre_stateful), priority=0    , match=(1), action=(next;)
      table=3 (ls_out_lb          ), priority=0    , match=(1), action=(next;)
      table=4 (ls_out_acl         ), priority=0    , match=(1), action=(next;)
      table=5 (ls_out_qos_mark    ), priority=0    , match=(1), action=(next;)
      table=6 (ls_out_stateful    ), priority=100  , match=(reg0[1] == 1), action=(ct_commit(ct_label=0/1); next;)
      table=6 (ls_out_stateful    ), priority=100  , match=(reg0[2] == 1), action=(ct_lb;)
      table=6 (ls_out_stateful    ), priority=0    , match=(1), action=(next;)
      table=7 (ls_out_port_sec_ip ), priority=0    , match=(1), action=(next;)
      table=8 (ls_out_port_sec_l2 ), priority=100  , match=(eth.mcast), action=(output;)
      table=8 (ls_out_port_sec_l2 ), priority=50   , match=(outport == "sw0-port1" && eth.dst == {00:00:00:00:00:01}), action=(output;)
      table=8 (ls_out_port_sec_l2 ), priority=50   , match=(outport == "sw0-port2" && eth.dst == {00:00:00:00:00:02}), action=(output;)
```

The easiest way to understand logical flows is to use the ovn-trace command. ovn-trace allows you to see how OVN would process a sample packet.

ovn-trace has two required arguments:

```bash
 $ ovn-trace DATAPATH MICROFLOW
```

**DATAPATH identifies the logical datapath (a logical switch or a logical router) where the sample packet will begin.** MICROFLOW describes the sample packet to be simulated. 

Given our sample OVN configuration, let’s see how OVN would process a packet from sw0-port1 that is intended for sw0-port2. ovn-trace has a few different levels of detail to choose from. The first is –minimal, which tells you what happens to a packet, but omits a lot of unnecessary detail. In this case, we see that the final result is that the packet will be delivered to sw0-port2, as expected.

```bash
# This utility simulates packet forwarding within an OVN logical network.
# --minimal These options control the form and level of detail in ovn-trace output
$ ovn-trace --minimal sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
    # reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000
    output("sw0-port2");
```

The next level of detail is given if you use the –summary option. In this mode, we get more detail about packet processing, including which pipeline is being executed. If we run ovn-trace with the same sample packet, we get a better idea of how the packet is processed. We see that:

1. The packet enters the network (sw0) from port sw0-port1 and runs the ingress pipeline.
2. We can see the value “sw0-port2” set to the “outport” variable, indicating that the intended destination for this packet is “sw0-port2”.
3. The packet is output from the ingress pipeline, which brings it to the egress pipeline for “sw0” with the outport variable set to “sw0-port2”.
4. The output action is executed in the egress pipeline, which outputs the packet to the current value of the “outport” variable, which is “sw0-port2”.

```bash
# Summary output includes the logical pipelines visited by a packet and the logical actions executed on it. 
$ ovn-trace --summary sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
    # reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000
    ingress(dp="sw0", inport="sw0-port1") {
        outport = "sw0-port2";
        output;
        egress(dp="sw0", inport="sw0-port1", outport="sw0-port2") {
            output;
            /* output to "sw0-port2", type "" */;
        };
    };
```

While debugging a problem or modifying the code, you may want even more detailed output. ovn-trace has a –detailed option. In this case you get more details about each meaningful logical flow encountered. You see the table number, pipeline stage name, full match, and priority number from the flow. You also get a reference to the location in the OVN source code that is responsible for the creation of that logical flow.

```bash
# The detailed form of output is also the default form. 
$ ovn-trace --detailed sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
    # reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000

    ingress(dp="sw0", inport="sw0-port1")
    -------------------------------------
     0. ls_in_port_sec_l2 (ovn-northd.c:2827): inport == "sw0-port1" && eth.src == {00:00:00:00:00:01}, priority 50
        next(1);
    13. ls_in_l2_lkup (ovn-northd.c:3095): eth.dst == 00:00:00:00:00:02, priority 50
        outport = "sw0-port2";
        output;

    egress(dp="sw0", inport="sw0-port1", outport="sw0-port2")
    ---------------------------------------------------------
     8. ls_out_port_sec_l2 (ovn-northd.c:3170): outport == "sw0-port2" && eth.dst == {00:00:00:00:00:02}, priority 50
        output;
        /* output to "sw0-port2", type "" */
```

Another good example of using ovn-trace would be to see why a packet is getting dropped.  We’ve enabled port security, so let’s get a detailed trace of what would happen to a packet sent from sw0-port1 that contained an unexpected source MAC address.  The output will show us that the packet entered sw0 and failed to match any flow in table 0, meaning the packet is dropped.  We also see that table 0 is named “ls_in_port_sec_l2”, short for “Logical Switch ingress L2 port security”.

```bash
$ ovn-trace --detailed sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:ff && eth.dst == 00:00:00:00:00:02'
# reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:ff,dl_dst=00:00:00:00:00:02,dl_type=0x0000

    ingress(dp="sw0", inport="sw0-port1")
    -------------------------------------
    0. ls_in_port_sec_l2: no match (implicit drop)
```

A similar example would be if a packet contained an unknown destination MAC address.  In this case, we’ll see that the packet successfully passed table 0, but failed to match in table 13, “ls_in_l2_lkup”, short for “Logical Switch ingress L2 lookup”.

```bash
$ ovn-trace --detailed sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:ff'
    # reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:ff,dl_type=0x0000

    ingress(dp="sw0", inport="sw0-port1")
    -------------------------------------
     0. ls_in_port_sec_l2 (ovn-northd.c:2827): inport == "sw0-port1" && eth.src == {00:00:00:00:00:01}, priority 50
        next(1);
    13. ls_in_l2_lkup: no match (implicit drop)
```

### demo

#### install and config

```bash
## ubuntu 22.04 安装
$ apt-get -y install build-essential fakeroot
$ apt-get install python-six openssl -y
$ apt-get install openvswitch-switch openvswitch-common -y
$ apt-get install ovn-central ovn-common ovn-host -y
$ apt-get install ovn-host ovn-common -y

## ovn启动文件 ovn-central.service
$ systemctl cat  ovn-central.service
# /lib/systemd/system/ovn-central.service
[Unit]
Description=Open Virtual Network central components
After=network.target
Requires=network.target
Wants=ovn-northd.service
Wants=ovn-ovsdb-server-sb.service
Wants=ovn-ovsdb-server-nb.service

[Service]
Type=oneshot
ExecStart=/bin/true
ExecStop=/bin/true
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target


## openvswitch启动文件
$ systemctl cat  openvswitch-switch.service
# /lib/systemd/system/openvswitch-switch.service
[Unit]
Description=Open vSwitch
Before=network.target
After=network-pre.target ovsdb-server.service ovs-vswitchd.service
PartOf=network.target
Requires=ovsdb-server.service
Requires=ovs-vswitchd.service

[Service]
Type=oneshot
ExecStart=/bin/true
ExecReload=/usr/share/openvswitch/scripts/ovs-systemd-reload
ExecStop=/bin/true
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
Also=ovs-record-hostname.service


## ovsdb-server 启动命令
ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info --remote=punix:/var/run/openvswitch/db.sock --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log --pidfile=/var/run/openvswitch/ovsdb-server.pid --detach
## ovs-vswitchd 启动命令
ovs-vswitchd unix:/var/run/openvswitch/db.sock -vconsole:emer -vsyslog:err -vfile:info --mlockall --no-chdir --log-file=/var/log/openvswitch/ovs-vswitchd.log --pidfile=/var/run/openvswitch/ovs-vswitchd.pid --detach
## ovn-controller 启动命令
ovn-controller unix:/var/run/openvswitch/db.sock -vconsole:emer -vsyslog:err -vfile:info --no-chdir --log-file=/var/log/ovn/ovn-controller.log --pidfile=/var/run/ovn/ovn-controller.pid --detach

## 命令报错？
$ ovn-nbctl show
ovn-nbctl: unix:/usr/local/var/run/openvswitch/ovnnb_db.sock: database connection failed (No such file or directory)
## 很奇怪，apt-get install ovn-northd 监听的socket在/var/run/下
ovn-northd -vconsole:emer -vsyslog:err -vfile:info --ovnnb-db=unix:/var/run/ovn/ovnnb_db.sock --ovnsb-db=unix:/var/run/ovn/ovnsb_db.sock --no-chdir --log-file=/var/log/ovn/ovn-northd.log --pidfile=/var/run/ovn/ovn-northd.pid --detach
root       32751   24467  0 12:27 pts/1    00:00:00 grep --color=auto ovn-nor
## 默认ovn-northd使用的socket在 /usr/local/var/run/下
$ ovn-northd  -h
ovn-northd: OVN northbound management daemon
usage: ovn-northd [OPTIONS]

Options:
  --ovnnb-db=DATABASE       connect to ovn-nb database at DATABASE
                            (default: unix:/usr/local/var/run/openvswitch/ovnnb_db.sock)
  --ovnsb-db=DATABASE       connect to ovn-sb database at DATABASE
                            (default: unix:/usr/local/var/run/openvswitch/ovnsb_db.sock)
  -h, --help                display this help message

## 临时解决方法：创建软连接
$ ln -sf /var/run/ovn/ovnnb_db.sock /usr/local/var/run/openvswitch/ovnnb_db.sock
$ ln -sf /var/run/ovn/ovnsb_db.sock  /usr/local/var/run/openvswitch/ovnsb_db.sock

$ ovn-nbctl show && ovn-sbctl show && echo "nb and sb are ok."
nb and sb are ok.

```



So far, we have only looked at examples of a single L2 logical network.  Let’s create a new environment that shows how ovn-trace works across multiple networks.  We will create 2 networks, each with 2 ports, and connect them with a logical router.

```bash
# Create the first logical switch and its two ports.
# ls-add [SWITCH]           create a logical switch named SWITCH
$ ovn-nbctl ls-add sw0
$ ovn-nbctl ls-list
064594e0-f900-49c3-bd22-65cc94449cb4 (sw0)



$ ovn-nbctl lsp-add sw0 sw0-port1
$ ovn-nbctl lsp-list sw0
0a09ddfb-72a7-4090-9026-6f5fe76ce46e (sw0-port1)

# set port security addresses for PORT
$ ovn-nbctl lsp-set-addresses sw0-port1 "00:00:00:00:00:01 10.0.0.51"
#  set port security addresses for PORT.
# Port security limits the addresses from which a logical port may send packets and to which it may receive packets.
$ ovn-nbctl lsp-set-port-security sw0-port1 "00:00:00:00:00:01 10.0.0.51"

ovn-nbctl lsp-add sw0 sw0-port2
ovn-nbctl lsp-set-addresses sw0-port2 "00:00:00:00:00:02 10.0.0.52"
ovn-nbctl lsp-set-port-security sw0-port2 "00:00:00:00:00:02 10.0.0.52"

$ ovn-nbctl lsp-list sw0
0a09ddfb-72a7-4090-9026-6f5fe76ce46e (sw0-port1)
943846aa-6e42-4437-a61b-bcf51f0edff2 (sw0-port2)


# Create the second logical switch and its two ports.
ovn-nbctl ls-add sw1

ovn-nbctl lsp-add sw1 sw1-port1
ovn-nbctl lsp-set-addresses sw1-port1 "00:00:00:00:00:03 192.168.1.51"
ovn-nbctl lsp-set-port-security sw1-port1 "00:00:00:00:00:03 192.168.1.51"

ovn-nbctl lsp-add sw1 sw1-port2
ovn-nbctl lsp-set-addresses sw1-port2 "00:00:00:00:00:04 192.168.1.52"
ovn-nbctl lsp-set-port-security sw1-port2 "00:00:00:00:00:04 192.168.1.52"

# Create a logical router between sw0 and sw1.
# ovn-nbctl create Logical_Router name=lr0
$ ovn-nbctl lr-add lr0
$ ovn-nbctl lr-list
751f8954-cc91-41a5-9327-41a7221f02af (lr0)

# 逻辑路由器上添加一个端口用来连接 sw0
$ ovn-nbctl lrp-add lr0 lrp0 00:00:00:00:ff:01 10.0.0.1/24
# 配置sw0连接vRouter
$ ovn-nbctl lsp-add sw0 sw0-lrp0 \
    -- set Logical_Switch_Port sw0-lrp0 type=router \
    options:router-port=lrp0 addresses='"00:00:00:00:ff:01"'
$  ovn-nbctl lsp-list sw0
25231ba4-657b-41be-8249-65c861846cc8 (sw0-lrp0)
0a09ddfb-72a7-4090-9026-6f5fe76ce46e (sw0-port1)
943846aa-6e42-4437-a61b-bcf51f0edff2 (sw0-port2)
# 同理配置sw1和vRouter连接
ovn-nbctl lrp-add lr0 lrp1 00:00:00:00:ff:02 192.168.1.1/24
ovn-nbctl lsp-add sw1 sw1-lrp1 \
    -- set Logical_Switch_Port sw1-lrp1 type=router \
    options:router-port=lrp1 addresses='"00:00:00:00:ff:02"'
```

We can then use “ovn-nbctl show” to view the resulting logical network configuration.

```bash
$ ovn-nbctl show
    switch bf4ba6c6-91c5-4f56-9981-72643816f923 (sw1)
        port sw1-lrp1
            addresses: ["00:00:00:00:ff:02"]
        port sw1-port2
            addresses: ["00:00:00:00:00:04 192.168.1.52"]
        port sw1-port1
            addresses: ["00:00:00:00:00:03 192.168.1.51"]
    switch 13b80127-4b36-46ea-816a-1ba4ffd6ac57 (sw0)
        port sw0-port1
            addresses: ["00:00:00:00:00:01 10.0.0.51"]
        port sw0-lrp0
            addresses: ["00:00:00:00:ff:01"]
        port sw0-port2
            addresses: ["00:00:00:00:00:02 10.0.0.52"]
    router 68935017-967a-4c4a-9dad-5d325a9f203a (lr0)
        port lrp0
            mac: "00:00:00:00:ff:01"
            networks: ["10.0.0.1/24"]
        port lrp1
            mac: "00:00:00:00:ff:02"
            networks: ["192.168.1.1/24"]
```

#### ovn-trace

The  microflow  argument  describes the packet whose forwarding is to be simulated, in the syntax of an OVN logical expression, as described in ovn-sb(5),to  express  constraints.

For reasonable L2 behavior, **the microflow should include at least inport and eth.dst**, **plus eth.src if port security is enabled.** For example:

> port security 避免arp欺骗？？？

```bash
inport == "lp11" && eth.src == 00:01:02:03:04:05 && eth.dst == ff:ff:ff:ff:ff:ff
```

 For  reasonable L3 behavior, microflow should also include **ip4.src and ip4.dst** (or ip6.src and ip6.dst) **and ip.ttl**. For example:

> 如果不加`ip.ttl = N`，则默认ttl值为0，ovn会直接丢弃该package
>
> nw_ttl=ttl Matches IP TTL or IPv6 hop limit value ttl, which  is  specified as a decimal number between 0 and 255, inclusive.
>
> ```bash
> ## nw_ttl是0
> $ ovn-trace --detail sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip4.src == 10.0.0.51 && eth.dst == 00:00:00:00:ff:01 && ip4.dst == 192.168.1.51'
> # ip,reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:ff:01,nw_src=10.0.0.51,nw_dst=192.168.1.51,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=0
> 
> ```
>
> end

```bash
inport == "lp111" && eth.src == f0:00:00:00:01:11 && eth.dst == 00:00:00:00:ff:11 && ip4.src == 192.168.11.1 && ip4.dst == 192.168.22.2 && ip.ttl == 64
```

We should be able to trace a packet sent from sw0-port1 destined for sw1-port2, which requires going through the router. The minimal output will confirm that the end result is that the packet should be output to sw1-port2. We will also see what modifications the packet received along the way. As the packet traversed the logical router, the TTL was decremented and then the source and destination MAC addresses were updated for the next hop.

```bash
# ovn-trace - Open Virtual Network logical network tracing utility

$ ovn-trace --minimal sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip4.src == 10.0.0.51 && eth.dst == 00:00:00:00:ff:01 && ip4.dst == 192.168.1.52 && ip.ttl == 32'
# ip,reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:ff:01,nw_src=10.0.0.51,nw_dst=192.168.1.52,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=32
ip.ttl--;
eth.src = 00:00:00:00:ff:02;
eth.dst = 00:00:00:00:00:04;
output("sw1-port2");
```

If you’d like to take an even closer look, you can experiment with the previous ovn-trace command by changing the verbosity to –summary or –detailed. You could also start to make changes to the sample packet to see what would happen.

可以使用`--detail`参数看到消息的输出

```bash
$ ovn-trace --detail sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip4.src == 10.0.0.51 && eth.dst == 00:00:00:00:ff:01 && ip4.dst == 192.168.1.51 && ip.ttl == 32'
# ip,reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:ff:01,nw_src=10.0.0.51,nw_dst=192.168.1.51,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=32

ingress(dp="sw0", inport="sw0-port1")
-------------------------------------
 0. ls_in_port_sec_l2 (ovn-northd.c:2979): inport == "sw0-port1" && eth.src == {00:00:00:00:00:01}, priority 50, uuid bcf1fac0
    next;
 1. ls_in_port_sec_ip (ovn-northd.c:2113): inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip4.src == {10.0.0.51}, priority 90, uuid b66c72e2
    next;
13. ls_in_l2_lkup (ovn-northd.c:3274): eth.dst == 00:00:00:00:ff:01, priority 50, uuid b700ed25
    outport = "sw0-lrp0";
    output;

egress(dp="sw0", inport="sw0-port1", outport="sw0-lrp0")
--------------------------------------------------------
 8. ls_out_port_sec_l2 (ovn-northd.c:3399): outport == "sw0-lrp0", priority 50, uuid ca1e1506
    output;
    /* output to "sw0-lrp0", type "patch" */

ingress(dp="lr0", inport="lrp0")
--------------------------------
 0. lr_in_admission (ovn-northd.c:3799): eth.dst == 00:00:00:00:ff:01 && inport == "lrp0", priority 50, uuid df571895
    next;
 5. lr_in_ip_routing (ovn-northd.c:3527): ip4.dst == 192.168.1.0/24, priority 49, uuid 65597d52
    ip.ttl--;
    reg0 = ip4.dst;
    reg1 = 192.168.1.1;
    eth.src = 00:00:00:00:ff:02;
    outport = "lrp1";
    flags.loopback = 1;
    next;
 6. lr_in_arp_resolve (ovn-northd.c:4838): outport == "lrp1" && reg0 == 192.168.1.51, priority 100, uuid 1b2b76f9
    eth.dst = 00:00:00:00:00:03;
    next;
 8. lr_in_arp_request (ovn-northd.c:5010): 1, priority 0, uuid 5210dfd3
    output;

egress(dp="lr0", inport="lrp0", outport="lrp1")
-----------------------------------------------
 3. lr_out_delivery (ovn-northd.c:5038): outport == "lrp1", priority 100, uuid 08821f4b
    output;
    /* output to "lrp1", type "patch" */

ingress(dp="sw1", inport="sw1-lrp1")
------------------------------------
 0. ls_in_port_sec_l2 (ovn-northd.c:2979): inport == "sw1-lrp1", priority 50, uuid 76e32c0b
    next;
13. ls_in_l2_lkup (ovn-northd.c:3274): eth.dst == 00:00:00:00:00:03, priority 50, uuid 483520df
    outport = "sw1-port1";
    output;

egress(dp="sw1", inport="sw1-lrp1", outport="sw1-port1")
--------------------------------------------------------
 7. ls_out_port_sec_ip (ovn-northd.c:2113): outport == "sw1-port1" && eth.dst == 00:00:00:00:00:03 && ip4.dst == {255.255.255.255, 224.0.0.0/4, 192.168.1.51}, priority 90, uuid 0236786a
    next;
 8. ls_out_port_sec_l2 (ovn-northd.c:3399): outport == "sw1-port1" && eth.dst == {00:00:00:00:00:03}, priority 50, uuid 9bc44267
    output;
    /* output to "sw1-port1", type "" */
root@Dodo:~#

```



#### arp address all 0

Target MAC address是全0，说明当前还不知道目标主机的MAC地址，从而进行arp广播

如下，如果`ip4.dst`设置一个不存在的`192.168.1.0/24`网段IP，则会发生arp请求

```bash
## 没有port的IP是192.168.1.58
$ ovn-trace --minimal sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && ip4.src == 10.0.0.51 && eth.dst == 00:00:00:00:ff:01 && ip4.dst == 192.168.1.58 && ip.ttl == 32'
# ip,reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:ff:01,nw_src=10.0.0.51,nw_dst=192.168.1.58,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=32
ip.ttl--;
eth.src = 00:00:00:00:ff:02;
eth.dst = 00:00:00:00:00:00;
arp {
    eth.dst = ff:ff:ff:ff:ff:ff;
    arp.spa = 0xc0a80101;
    arp.tpa = 0xc0a8013a;
    arp.op = 1;
    output("sw1-port1");
    output("sw1-port2");
};

```



## A Powerful Abstraction

I mentioned at the very beginning of this post that I find OVN Logical Flows to be a powerful abstraction that has made adding features to OVN much easier than I anticipated. Now that we’ve gone through logical flows in some detail, I’d like to point to a couple of recent feature developments that help demonstrate how easy logical flows make adding features to OVN.

### SOURCE BASED ROUTING

**OVN has support for L3 gateways that can be used to provide connectivity between OVN logical networks and physical networks.** A typical OVN network might have an L3 gateway that resides on a single host. The downside to using a single L3 gateway is that all traffic destined for that physical network must go through the single host where the L3 gateway resides.

Recently, [Gurucharan Shetty added support for multiple L3 gateways on an OVN logical network](https://github.com/openvswitch/ovs/commit/440a9f4b32bead4fa82d93b1fdceed3e55c20b4b). The method supported for distributing traffic among the gateways is based on source IP address.

It’s not important to understand all of the details in this change. I mainly want to draw attention to how little of a code change was required. Let’s take a look at the diffstat, organizied by the type of change.

```
Documentation:
 NEWS                          |    1 
 ovn/ovn-nb.xml                |   28 +++++
 ovn/utilities/ovn-nbctl.8.xml |    8 +

Database schema update:
 ovn/ovn-nb.ovsschema          |    8 +

Command line utility support for new db schema additions:
 ovn/utilities/ovn-nbctl.c     |   43 ++++++--

Changes to how OVN builds logical flows to add support for this feature:
 ovn/northd/ovn-northd.c       |   24 +++-

Test code:
 tests/ovn-nbctl.at            |   42 ++++----
 tests/ovn.at                  |  219 ++++++++++++++++++++++++++++++++++++++++++

 8 files changed, 334 insertions(+), 39 deletions(-)
```

At the very core of this feature is the 24 lines of code changed in ovn-northd.c. This is where the code that generates logical flows was updated for this feature. That is amazing to me. This feature has a significant impact on network behavior, yet was accomplished in very few lines of C code.

### DSCP

 DSCP差分服务代码点（Differentiated Services Code Point)

Another example of adding a feature using logical flows is [this patch that adds support for setting the DSCP field of IP packets based on an arbitrary traffic classifier](https://github.com/openvswitch/ovs/commit/1a03fc7da32e74aacbf638d3617a290ddffaa069).

The patch added a new QoS table to the OVN northbound database. In this table you define a match (or traffic classifier) and the corresponding DSCP value to set on packets that match this classifier.

The key changes to the code are the 80 lines changed in ovn-northd.c. The patch looks a bit bigger than it really is because it created a new pipeline stage for QoS and had to renumber the stages that followed it.

Implementing this feature using logical flows just requires inserting flows matching the configured match (traffic classifier), and then using the OVN logical flow actions “ip.dscp = VALUE; next;”.

### DHCP

OVN supports DHCPv4 and DHCPv6, but all of the details are controlled through logical flows. In most cases, if behavior needs to be changed, it’s only a change to the code that generates the DHCP related logical flows.

A recent example was [this patch which added stateless DHCPv6 support](https://github.com/openvswitch/ovs/commit/40df456680fc1e760d9cc666fef1761b5234cf00). With stateless DHCPv6, we want to provide some configuration entries (such as a DNS address), but not assign an IPv6 address. Implementing this was just a small tweak to the DHCPv6 logical flows to optionally not include the IPv6 address option in the response generated by OVN.

## Conclusion

I hope you now have a better understanding of OVN Logical Flows, a match-action pipeline for defining the behavior of logical networks that can span many hosts.

Thanks again to Ben Pfaff for more recently writing ovn-trace, which makes it easier to read, understand, and modify OVN logical flows. Ben talked about ovn-trace as a part of our OVN talk at the OpenStack Summit in Barcelona. You can find a video of that talk here:

