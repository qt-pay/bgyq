## [LGTM]ovs cheat sheet

## Look up the table

```
ovs-vsctl list bridge ovs-br
```

## About Bridge and Port

| OpenVswitch                                     | About Bridge and Port                                        |
| :---------------------------------------------- | :----------------------------------------------------------- |
| Add Bridge                                      | `ovs-vsctl add-br ovs-br`                                    |
| Corresponds to the interface on ovs-br          | `ovs-vsctl add-port ovs-br eth0`                             |
| (1) + (2) can be written                        | `ovs−vsctl add−br ovs-br -- add−port ovs-br eth0`            |
| Remove Bridge                                   | `ovs-vsctl del-br ovs-br` # If it does not exist, there will be error log `ovs-vsctl --if-exists del-br ovs-br` |
| Change the ofport (openflow port number) to 100 | `ovs-vsctl add-port ovs-br eth0 -- set Interface eth0 ofport_request=100` |
| Set the port to internal                        | `ovs-vsctl set Interface eth0 type=internal`                 |

## About the Controller

| OpenVswitch                                                  | About the Controller                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Set the Controller                                           | `ovs-vsctl set-controller ovs-br tcp:1.2.3.4:6633`           |
| Set the multi controller                                     | `ovs-vsctl set-controller ovs-br tcp:1.2.3.4:6633 tcp:5.6.7.8:6633` |
| Query the Controller settings                                | `ovs-vsctl show`                                             |
| If you have successfully connected to the controller appears is_connected:true , otherwise not connected | `ovs-vsctl get-controller ovs-br`                            |
| Remove the Controller                                        | `ovs-vsctl del-controller ovs-br`                            |

## About STP (Spanning Tree Protocol)

| OpenVswitch             | About STP                                                   |
| :---------------------- | :---------------------------------------------------------- |
| Enable STP              | `ovs-vsctl set bridge ovs-br stp_enable=true`               |
| Turn off STP            | `ovs-vsctl set bridge ovs-br stp_enable=false`              |
| Query STP settings      | `ovs-vsctl get bridge ovs-br stp_enable`                    |
| Set Priority            | `ovs−vsctl set bridge br0 other_config:stp-priority=0x7800` |
| Set Cost                | `ovs−vsctl set port eth0 other_config:stp-path-cost=10`     |
| Remove the STP settings | `ovs−vsctl clear bridge ovs-br other_config`                |

## About Openflow Version

| OpenVswitch                          | About Openflow Version                                       |
| :----------------------------------- | :----------------------------------------------------------- |
| OpenFlow Version 1.3 is supported    | `ovs-vsctl set bridge ovs-br protocols=OpenFlow13`           |
| Support OpenFlow Version 1.3 1.2     | `ovs-vsctl set bridge ovs-br protocols=OpenFlow12,OpenFlow13` |
| Remove the OpenFlow support settings | `ovs-vsctl clear bridge ovs-br protocols`                    |

## VLAN

| OpenVswitch                                | About VLAN                                                   |
| :----------------------------------------- | :----------------------------------------------------------- |
| Set the VLAN tag                           | `ovs-vsctl add-port ovs-br vlan3 tag=3 -- set interface vlan3 type=internal` |
| Remove the VLAN                            | `ovs-vsctl del-port ovs-br vlan3`                            |
| Query the VLAN                             | `ovs-vsctl show` `ifconfig vlan3`                            |
| Set the Vlan trunk                         | `ovs-vsctl add-port ovs-br eth0 trunk=3,4,5,6`               |
| Set the add port to access port, vlan id 9 | `ovs-vsctl set port eth0 tag=9`                              |
| Ovs-ofctl add-flow Set vlan 100            | `ovs-ofctl add-flow ovs-br in_port=1,dl_vlan=0xffff,actions=mod_vlan_vid:100,output:3` `ovs-ofctl add-flow ovs-br in_port=1,dl_vlan=0xffff,actions=push_vlan:0x8100,set_field:100-\>vlan_vid,output:3` |

| Ovs-ofctl add-flow Remove the vlan tag | `ovs-ofctl add-flow ovs1 in_port=3,dl_vlan=100,actions=strip_vlan,output:1` |
| -------------------------------------- | ------------------------------------------------------------ |
| Two_vlan example                       | `ovs-ofctl add-flow pop-vlan` `ovs-ofctl add-flow ovs-br in_port=3,dl_vlan=0xffff,actions=pop_vlan,output:1` |

## About GRE tunnels

| OpenVswitch          | About GRE                                                    |
| :------------------- | :----------------------------------------------------------- |
| Set the GRE tunnel   | `ovs−vsctl add−port ovs-br ovs-gre -- set interface ovs-gre type=gre options:remote_ip=1.2.3.4` |
| Check the GRE tunnel | `ovs-vsctl show`                                             |

## About Dump flows

| OpenVswitch                                                  | About Dump flows                      |
| :----------------------------------------------------------- | :------------------------------------ |
| Dumps OpenFlow flows do not contain hidden flows (common)    | `ovs-ofctl dump-flows ovs-br`         |
| Dumps OpenFlow flows contain hidden flows                    | `ovs-appctl bridge/dump-flows ovs-br` |
| Dump specific bridge of the datapath flows regardless of any type | `ovs-appctl dpif/dump-flows ovs-br`   |
| Dump in the Linux kernel in the datapath flow table (commonly used) | `ovs-dpctl dump-flows [dp]`           |
| Top like behavior for ovs-dpctl dump-flows                   | `ovs-dpctl-top`                       |

## XenServer starts OpenvSwitch mode

| OpenVswitch                   | XenServer                               |
| :---------------------------- | :-------------------------------------- |
| Check whether it is on or not | `service openvswitch status`            |
| Openv                         | `xe-switch-network-backend openvswitch` |
| shut down                     | `xe-switch-network-backend bridge`      |

## About Log

| OpenVswitch                                                 | About Log                                                    |
| :---------------------------------------------------------- | :----------------------------------------------------------- |
| Query log level list                                        | `ovs-appctl vlog/list`                                       |
| Set the log level (to stp set dbg level file as an example) | `ovs-appctl vlog/set stp:file:dbg` `ovs-appctl vlog/set {module name}:{console, syslog, file}:{off, emer, err, warn, info, dbg}` |

## About Fallback:???

| OpenVswitch                                                  | About Fallback                              |
| :----------------------------------------------------------- | :------------------------------------------ |
| Controller connection: false, will be automatically transferred into the legacy switch mode | `ovs-vsctl set-fail-mode ovs-br standalone` |
| Regardless of the Controller connection status why, must be carried out through OpenFlow network behavior (default) | `ovs-vsctl set-fail-mode ovs-br secure`     |
| Remove                                                       | `ovs-vsctl del-fail-mode ovs-br`            |
| Inquire                                                      | `ovs-vsctl get-fail-mode ovs-br`            |

```bash
$ ovs-vsctl add-br OVN -- set Bridge OVN fail-mode=secure
$ ovs-vsctl add-br ovb-t2
$ ovs-vsctl add-br ovn-t1 -- set Bridge ovn-t1 fail-mode=secure
$ ovs-vsctl list bridge|grep -E "name|fail_mode"
fail_mode           : []
name                : ovn-t2
fail_mode           : secure
name                : ovn-t1
fail_mode           : secure
name                : OVN

```



## About sFlow

| OpenVswitch | About sFlow                              |
| :---------- | :--------------------------------------- |
| Inquire     | `ovs-vsctl list sflow`                   |
| New         | `set sFlow`                              |
| delete      | `ovs-vsctl -- clear Bridge ovs-br sflow` |

## About NetFlow

| OpenVswitch | About NetFlow                              |
| :---------- | :----------------------------------------- |
| Inquire     | `ovs-vsctl list netflow`                   |
| New         | `Set NetFlow`                              |
| Delete      | `ovs-vsctl -- clear Bridge ovs-br netflow` |

## Set the Out-of-band and in-band

| OpenVswitch            | Set the Out-of-band and in-band                              |
| :--------------------- | :----------------------------------------------------------- |
| Inquire                | `ovs-vsctl get controller ovs-br connection-mode`            |
| Out-of-band            | `ovs-vsctl set controller ovs-br connection-mode=out-of-band` |
| In-band (default)      | `ovs-vsctl set controller ovs-br connection-mode=in-band`    |
| Remove the hidden flow | `ovs-vsctl set bridge br0 other-config:disable-in-band=true` |

## About ssl

| OpenVswitch | About SSL                                                 |
| :---------- | :-------------------------------------------------------- |
| Inquire     | `ovs-vsctl get-ssl`                                       |
| set up      | `ovs-vsctl set-ssl sc-privkey.pem sc-cert.pem cacert.pem` |
| delete      | `ovs-vsctl del-ssl`                                       |

## About SPAN

| OpenVswitch       | About SPAN                                                   |
| :---------------- | :----------------------------------------------------------- |
| Detailed settings | `ovs-vsctl add-br ovs-br` `ovs-vsctl add-port ovs-br eth0` `ovs-vsctl add-port ovs-br eth1` `ovs-vsctl add-port ovs-br tap0 \` `- --id = @ p get port tap0 \` `- - id = @m create mirror name = m0 select-all = true output-port = @ p \` `- set bridge ovs-br mirrors = @ m` |
| Add               | `ovs-br on add-port {eth0, eth1} mirror to tap0`             |
| delete            | `ovs-vsctl clear bridge ovs-br mirrors # About Table`        |
| Check the Table   | `ovs-ofctl dump-tables ovs-br`                               |

## About VXLAN

[Reference rascov - Bridge Remote Mininets using VXLAN](http://blog.mcchan.io/bridge-remote-networks-using-vxlan)

| OpenVswitch                                                  | About VxLAN                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Establish the VXLAN Network ID (VNI) and the specified OpenFlow port number, eg: VNI = 5566, OF_PORT = 9 | `ovs-vsctl set interface vxlan type=vxlan option:remote_ip=xxxx option:key=5566 ofport_request=9` |
| VNI flow by flow                                             | `ovs-vsctl set interface vxlan type=vxlan option:remote_ip=140.113.215.200 option:key=flow ofport_request=9` |
| Set the VXLAN tunnel id                                      | `ovs-ofctl add-flow ovs-br in_port=1,actions=set_field:5566->tun_id,output:2` `ovs-ofctl add-flow s1 in_port=2,tun_id=5566,actions=output:1` |

## About OVSDB Manager

[Reference OVSDB Integration: Mininet OVSDB Tutorial](https://wiki.opendaylight.org/view/OVSDB_Integration:Mininet_OVSDB_Tutorial)

| OpenVswitch               | About OVSDB                              |
| :------------------------ | :--------------------------------------- |
| Active Listener settings  | `ovs-vsctl set-manager tcp:1.2.3.4:6640` |
| Passive Listener settings | `ovs-vsctl set-manager ptcp:6640`        |

## OpenFlow Trace

| OpenVswitch           | About OpenFlow Trace                                         |
| :-------------------- | :----------------------------------------------------------- |
| Generate pakcet trace | `ovs-appctl ofproto/trace ovs-br in_port=1,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02 -generate` |

## Other

| OpenVswitch                               | Others                       |
| :---------------------------------------- | :--------------------------- |
| Query the OpenvSwitch version             | `ovs-ofctl -V`               |
| Query the history of the next instruction | `ovsdb-tool show-log [-mmm]` |

## Reference

### **DB**

```bash
# Show OVS basic info (version, dpdk enabled, PMD cores, lcore, ODL bridge mapping, balancing, auto-balancing etc)
ovs-vsctl list open_vswitch
ovs-vsctl list bride
ovs-vsctl list interface
ovs-vsctl list interface vxlan-ac000344
ovs-vsctl --columns=options list interface vxlan-ac000344
ovs-vsctl --columns=ofport,name list Interface
ovs-vsctl --columns=ofport,name --format=table list Interface
ovs-vsctl -f csv --no-heading --columns=_uuid list controller
ovs-vsctl -f csv --no-heading -d bare --columns=other_config list port
ovs-vsctl --format=table --columns=name,mac_in_use find Interface name=br-dpdk1
ovs-vsctl get interface vhub656c3cb-23 name

ovs-vsctl set port vlan1729 tag=1729
ovs-vsctl get port vlan1729 tag
ovs-vsctl remove port vlan1729 tag 1729

# not sure this is best
ovs-vsctl set interface vlan1729 mac='5c\:b9\:01\:8d\:3e\:9d'

ovs-vsctl clear Bridge br0 stp_enable

ovs-vsctl --may-exist add-br br0 -- set bridge br0 datapath_type=netdev
ovs-vsctl --if-exists del-br br0
```

### **Flows**

```bash
ovs-ofctl dump-flows br-int

# include hidden flows
ovs-appctl bridge/dump-flows br0

# remove stats on older versions that don't have --no-stats
ovs-ofctl dump-flows br-int | cut -d',' -f3,6,7-
ovs-ofctl -O OpenFlow13 dump-flows br-int | cut -d',' -f3,6,7-

ovs-appctl dpif/show
ovs-ofctl show br-int | egrep "^ [0-9]"

ovs-ofctl add-flow brbm priority=1,in_port=11,dl_src=00:05:95:41:ec:8c/ff:ff:ff:ff:ff:ff,actions=drop
ovs-ofctl --strict del-flows brbm priority=0,in_port=11,dl_src=00:05:95:41:ec:8c

# kernel datapath
ovs-dpctl dump-flows
ovs-appctl dpctl/dump-flows
ovs-appctl dpctl/dump-flows system@ovs-system
ovs-appctl dpctl/dump-flows netdev@ovs-netdev
```

### **DPDK**

```bash
ovs-appctl dpif/show
ovs-ofctl dump-ports br-int
ovs-appctl dpctl/dump-flows
ovs-appctl dpctl/show --statistics
ovs-appctl dpif-netdev/pmd-stats-show
ovs-appctl dpif-netdev/pmd-stats-clear
ovs-appctl dpif-netdev/pmd-rxq-show
```

### **Debug log**

```
ovs-appctl vlog/list | grep dpdk
ovs-appctl vlog/set dpdk:file:dbg

# log openflow
ovs-appctl vlog/set vconn:file:dbg
```

**Misc**

```bash
ovs-appctl list-commands
ovs-appctl fdb/show brbm

ovs-appctl ofproto/trace br-int in_port=6

ovs-appctl ofproto/trace br-int tcp,in_port=3,vlan_tci=0x0000,dl_src=fa:16:3e:8d:26:61,dl_dst=fa:16:3e:0d:f5:e6,nw_src=10.0.0.26,nw_dst=10.0.0.9,nw_tos=0,nw_ecn=0,nw_ttl=0,tp_src=0,tp_dst=22,tcp_flags=0

# history
ovsdb-tool -mm show-log /etc/openvswitch/conf.db

top -p `pidof ovs-vswitchd` -H -d1

# port and dp cache stats
ovs-appctl dpctl/show -s
ovs-appctl memory/show
ovs-appctl upcall/show
ovs-appctl coverage/show
```

### **neutron ml2/ovs tracing**

```bash
PORT=tapfdd73231-29
tag=$(ovs-vsctl get port $PORT tag)
ofport=$(ovs-vsctl get interface $PORT ofport)
mac=$(ovs-vsctl get interface $PORT external_ids:attached-mac | sed -e 's/"//g')

# will flood to all tunnels
ovs-appctl ofproto/trace br-int in_port=${ofport},dl_src=${mac}

# unicast
dhcp_mac=fa:16:3e:46:07:82
ovs-appctl ofproto/trace br-int in_port=${ofport},dl_src=${mac},dl_dst=${dhcp_mac}

# Inbound from tunnel
ovs-ofctl show br-tun | grep -E "^ [0-9]"
tun_ofport=2
tun_id=$(ovs-vsctl get port $PORT other_config:segmentation_id | sed -e 's/"//g')
ovs-appctl ofproto/trace br-tun in_port=${tun_ofport},dl_src=${dhcp_mac},dl_dst=${mac},tun_id=$tun_id
```