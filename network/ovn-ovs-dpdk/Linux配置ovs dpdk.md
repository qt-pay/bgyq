## Linux配置ovs dpdk

### 管理命令

从ovs-v2.7.0开始，开启dpdk功能已不是vswitchd进程启动时指定–dpdk等参数了，而是通过设置ovsdb来开启dpdk功能

```bash
cat /etc/init.d/openvswitch
#!/bin/sh
#
# openvswitch
#
# chkconfig: 2345 09 91
# description: Manage Open vSwitch kernel modules and user-space daemons

# Copyright (C) 2009, 2010, 2011, 2013 Nicira, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
### BEGIN INIT INFO
# Provides:          openvswitch
# Required-Start:
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Open vSwitch switch
### END INIT INFO

SYSTEMCTL_SKIP_REDIRECT=yes
SYSTEMD_NO_WRAP=yes

. /usr/share/openvswitch/scripts/ovs-lib || exit 1
test -e /etc/sysconfig/openvswitch && . /etc/sysconfig/openvswitch

start () {
    set ovs_ctl ${1-start}
    set "$@" --system-id="local_chassis"
    if test X"$FORCE_COREFILES" != X; then
        set "$@" --force-corefiles="$FORCE_COREFILES"
    fi
    if test X"$OVSDB_SERVER_PRIORITY" != X; then
        set "$@" --ovsdb-server-priority="$OVSDB_SERVER_PRIORITY"
    fi
    if test X"$VSWITCHD_PRIORITY" != X; then
        set "$@" --ovs-vswitchd-priority="$VSWITCHD_PRIORITY"
    fi
    if test X"$VSWITCHD_MLOCKALL" != X; then
        set "$@" --mlockall="$VSWITCHD_MLOCKALL"
    fi
    set "$@" $OVS_CTL_OPTS
    "$@"

    touch /var/lock/subsys/openvswitch
    ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
    ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x3c
    ovs-vsctl --no-wait set Open_vSwitch . external_ids:ovn-bridge-datapath-type="netdev"
    ovs-vsctl --no-wait set Open_vSwitch . external-ids:ovn-bridge=business
    ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="16384,16384"
    ovs-vsctl --no-wait set open_vSwitch . external-ids:ovn-remote=tcp:127.0.0.1:6642
    ovs-vsctl --no-wait set open_vSwitch . external-ids:ovn-encap-type=geneve
    ovs-vsctl --no-wait set open_vSwitch . external-ids:ovn-encap-ip=127.0.0.1
    ovs-vsctl --no-wait set Open_vSwitch . other_config:smc-enable=true
}

stop () {
    ovs_ctl stop
    rm -f /var/lock/subsys/openvswitch
}

restart () {
    if [ "$1" = "--save-flows=yes" ]; then
        start restart
    else
        stop
        start
    fi
}

case $1 in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        shift
        restart "$@"
        ;;
    reload|force-reload)
        # Nothing to do.
        ;;
    status)
        ovs_ctl status
        exit $?
        ;;
    version)
        ovs_ctl version
        ;;
    force-reload-kmod)
        start force-reload-kmod
        ;;
    help)
        printf "$0 [start|stop|restart|reload|force-reload|status|version|force-reload-kmod]\n"
        ;;
    *)
        printf "Unknown command: $1\n"
        exit 1
        ;;
esac

```

