---
id: 710
title: OVS LISP Tunnel Configuration
date: 2013-11-19T19:32:16+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=710
permalink: /archives/710
views:
  - "6881"
categories:
  - kernel
  - Linux
  - networking
  - 系统管理
tags:
  - networking
  - ovs
  - tunnel
---
## Install {#install}

<ol style="list-style-type: decimal">
  <li>
    <code>git clone git://git.openvswitch.org/openvswitch</code>
  </li>
  <li>
    <code>cd openvswitch</code>
  </li>
  <li>
    <code>./boot.sh</code>
  </li>
  <li>
    <code>./configure --with-linux=/lib/modules/${uname -r}/build</code>
  </li>
  <li>
    <code>make</code>
  </li>
  <li>
    <code>make install;make modules_install</code>
  </li>
  <li>
    <code>modprobe openvswitch</code>
  </li>
  <li>
    <code>mkdir -p /usr/local/etc/openvswitch</code>
  </li>
  <li>
    <code>ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema</code>
  </li>
</ol>

## Startup {#startup}

<ol style="list-style-type: decimal">
  <li>
    ovsdb-server &#8211;remote=punix:/usr/local/var/run/openvswitch/db.sock <br /> &#8211;remote=db:Open_vSwitch,Open_vSwitch,manager_options <br /> &#8211;private-key=db:Open_vSwitch,SSL,private_key <br /> &#8211;certificate=db:Open_vSwitch,SSL,certificate <br /> &#8211;bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert <br /> &#8211;pidfile &#8211;detach
  </li>
  <li>
    <code>ovs-vsctl --no-wait init</code>
  </li>
  <li>
    <code>ovs-vswitchd --pidfile --detach</code>
  </li>
</ol>

## LISP Tunnel Configuration {#lisp-tunnel-configuration}

#### Topo {#topo}

                   +---+                               +---+
                   |VM1|                               |VM2|
                   +---+                               +---+
                     |                                   |
               +-- vnet0 --+                       +-- vnet0 --+
               |           |                       |           |
            lisp0   OVS1  eth0--------------------eth0  OVS2  lisp0
               |           |                       |           |
               +-----------+                       +-----------+
                   Host1                               Host2

#### Configuration {#configuration}

  * Host1 
      * `ovs-vsctl add-br br1`
      * `ovs-vsctl add-br br2`
      * `ovs-vsctl add-port br1 em1`
      * `ifconfig em1 0;dhclient br1`
      * `ifconfig br2 10.1.1.1 netmask 255.255.255.0`
      * `ovs-vsctl add-port br2 lisp0 -- set Interface lisp0 type=lisp options:remote_ip=192.168.1.252`  
        192.168.1.252 is Host2 eth0&#8217;s ip address
      * `ovs-ofctl add-flow br2 "priority=0,action=NORMAL"`
      * `ovs-ofctl add-flow br2 "priority=2,in_port=1,dl_type=0x0806,action=NORMAL"`
      * `ovs-ofctl add-flow br2 "priority=3,dl_dst=02:00:00:00:00:00,action=mod_dl_dst:52:54:00:B8:27:75,output:1"`  
        52:54:00:B8:27:75 is VM1&#8217;s mac address
      * `ovs-ofctl add-flow br2 "priority=1,in_port=1,dl_type=0x0800,vlan_tci=0,nw_src=10.1.1.0/24,action=output:2"`
  * VM1 
      * `virsh dumpxml kvm1 > domain.xml`
      * search the XML <interface type='network'> section and change it to something like this:
      * `virsh create domain.xml`
      * `ifconfig eth3 up`
      * `ifconfig eth3 10.1.1.20 netmask 255.255.255.0`
      * `echo "10.1.1.30 52:54:00:1B:EA:D5" > /etc/ethers`
      * `arp -f /etc/ethers`

             <interface type=&#39;bridge&#39;>
               <mac address=&#39;52:54:00:b8:27:75&#39;/>
               <source bridge=&#39;br2&#39;/>
               <virtualport type=&#39;openvswitch&#39;>
               </virtualport>
               <model type=&#39;virtio&#39;/>
               <address type=&#39;pci&#39; domain=&#39;0x0000&#39; bus=&#39;0x00&#39; slot=&#39;0x03&#39; function=&#39;0x0&#39;/>
             </interface>

        # ovs-vsctl show
        390982aa-7bf6-4dfd-84eb-76fe7654e73d
            Bridge "br2"
            Port "vnet0"
                Interface "vnet0"
            Port "lisp0"
                Interface "lisp0"
                type: lisp
                options: {remote_ip="192.168.1.252"}
            Port "br2"
                Interface "br2"
                type: internal
            Bridge "br1"
            Port "br1"
                Interface "br1"
                type: internal
            Port "em1"
                Interface "em1"

        # ovs-ofctl show br2
        OFPT_FEATURES_REPLY (xid=0x2): dpid:00008ad828ce7949
        n_tables:254, n_buffers:256
        capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
        actions: OUTPUT SET_VLAN_VID SET_VLAN_PCP STRIP_VLAN SET_DL_SRC SET_DL_DST SET_NW_SRC SET_NW_DST SET_NW_TOS SET_TP_SRC SET_T        P_DST ENQUEUE
         1(vnet0): addr:fe:54:00:b8:27:75
             config:     0
             state:      0
             current:    10MB-FD COPPER
             speed: 10 Mbps now, 0 Mbps max
         2(lisp0): addr:36:d5:4f:15:c0:7a
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         LOCAL(br2): addr:8a:d8:28:ce:79:49
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
        OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0

        # ovs-ofctl dump-ports br2
        OFPST_PORT reply (xid=0x2): 3 ports
          port  1: rx pkts=609, bytes=58706, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=341, bytes=33418, drop=0, errs=0, coll=0
          port  2: rx pkts=429, bytes=42042, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=582, bytes=56532, drop=0, errs=0, coll=0
          port LOCAL: rx pkts=6, bytes=468, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=296, bytes=28032, drop=0, errs=0, coll=0

        # ovs-ofctl dump-flows br2
        NXST_FLOW reply (xid=0x4):
         cookie=0x0, duration=1615.828s, table=0, n_packets=195, n_bytes=19110, idle_age=1378, priority=3,dl_dst=02:00:00:00:00:00 a         ctions=mod_dl_dst:52:54:00:b8:27:75,output:1
         cookie=0x0, duration=1717.963s, table=0, n_packets=326, n_bytes=31948, idle_age=1539, priority=0 actions=NORMAL
         cookie=0x0, duration=1519.394s, table=0, n_packets=101, n_bytes=9898, idle_age=1378, priority=1,ip,in_port=1,vlan_tci=0x000         0,nw_src=10.1.1.0/24 actions=output:2
         cookie=0x0, duration=1667.873s, table=0, n_packets=0, n_bytes=0, idle_age=1667, priority=2,arp,in_port=1 actions=NORMAL

  * Host2 
      * `ovs-vsctl add-br br1`
      * `ovs-vsctl add-br br2`
      * `ovs-vsctl add-port br1 eth0`
      * `ifconfig em1 0;dhclient br1`
      * `ifconfig br2 10.1.1.2 netmask 255.255.255.0`
      * `ovs-vsctl add-port br2 lisp0 -- set Interface lisp0 type=lisp options:remote_ip=192.168.1.249`
      * `ovs-ofctl add-flow br2 "priority=0,action=NORMAL"`
      * `ovs-ofctl add-flow br2 "priority=2,in_port=1,dl_type=0x0806,action=NORMAL"`
      * **`ovs-ofctl add-flow br2 "priority=3,dl_dst=02:00:00:00:00:00,action=mod_dl_dst:52:54:00:1B:EA:D5,output:1"`** 52:54:00:1B:EA:D5 is VM2&#8217;s mac address
      * `ovs-ofctl add-flow br2 "priority=1,in_port=1,dl_type=0x0800,vlan_tci=0,nw_src=10.1.1.0/24,action=output:2"`
  * VM2 
      * redid the configuration as VM1
      * `ifconfig eth5 up`
      * `ifconfig eth5 10.1.1.30 netmask 255.255.255.0`
      * `echo "10.1.1.20 52:54:00:B8:27:75" > /etc/ethers`
      * `arp -f /etc/ethers`

        # ovs-vsctl show
        66f0d773-a7c9-451e-980b-c2def85eda23
            Bridge "br2"
            Port "br2"
                Interface "br2"
                type: internal
            Port "lisp0"
                Interface "lisp0"
                type: lisp
                options: {remote_ip="192.168.1.249"}
            Port "vnet0"
                Interface "vnet0"
            Bridge "br1"
            Port "eth0"
                Interface "eth0"
            Port "br1"
                Interface "br1"
                type: internal

        # ovs-ofctl show br2
        OFPT_FEATURES_REPLY (xid=0x2): dpid:00001e2e0e0cf040
        n_tables:254, n_buffers:256
        capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
        actions: OUTPUT SET_VLAN_VID SET_VLAN_PCP STRIP_VLAN SET_DL_SRC SET_DL_DST SET_NW_SRC SET_NW_DST SET_NW_TOS SET_TP_SRC SET_T        P_DST ENQUEUE
         1(vnet0): addr:fe:54:00:1b:ea:d5
             config:     0
             state:      0
             current:    10MB-FD COPPER
             speed: 10 Mbps now, 0 Mbps max
         2(lisp0): addr:96:fa:62:5b:6f:09
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         LOCAL(br2): addr:1e:2e:0e:0c:f0:40
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
        OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0

        # ovs-ofctl dump-ports br2
        OFPST_PORT reply (xid=0x2): 3 ports
          port  1: rx pkts=657, bytes=60526, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=401, bytes=39094, drop=0, errs=0, coll=0
          port  2: rx pkts=617, bytes=60466, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=593, bytes=54754, drop=0, errs=0, coll=0
          port LOCAL: rx pkts=11, bytes=790, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=294, bytes=24952, drop=0, errs=0, coll=0

        # ovs-ofctl dump-flows br2
        NXST_FLOW reply (xid=0x4):
         cookie=0x0, duration=246.526s, table=0, n_packets=38, n_bytes=3724, idle_age=170, priority=3,dl_dst=02:00:00:00:00:00 actio         ns=mod_dl_dst:52:54:00:1b:ea:d5,output:1
         cookie=0x0, duration=293.084s, table=0, n_packets=98, n_bytes=9604, idle_age=234, priority=0 actions=NORMAL
         cookie=0x0, duration=194.739s, table=0, n_packets=23, n_bytes=2254, idle_age=170, priority=1,ip,in_port=1,vlan_tci=0x0000,n         w_src=10.1.1.0/24 actions=output:2
         cookie=0x0, duration=285.264s, table=0, n_packets=0, n_bytes=0, idle_age=285, priority=2,arp,in_port=1 actions=NORMAL

#### Test {#test}

ping test:

        # ping 10.1.1.30
        PING 10.1.1.30 (10.1.1.30) 56(84) bytes of data.
        64 bytes from 10.1.1.30: icmp_seq=1 ttl=64 time=3.51 ms
        64 bytes from 10.1.1.30: icmp_seq=2 ttl=64 time=1.21 ms
        64 bytes from 10.1.1.30: icmp_seq=3 ttl=64 time=1.62 ms
        64 bytes from 10.1.1.30: icmp_seq=4 ttl=64 time=1.80 ms
        64 bytes from 10.1.1.30: icmp_seq=5 ttl=64 time=1.33 ms
        64 bytes from 10.1.1.30: icmp_seq=6 ttl=64 time=1.49 ms
        64 bytes from 10.1.1.30: icmp_seq=7 ttl=64 time=1.66 ms

        Capture the packets on VM2 at the same time:   

        # tcpdump -i eth5 -xxx
        tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
        listening on eth5, link-type EN10MB (Ethernet), capture size 65535 bytes
        03:01:22.900663 IP 10.1.1.20 > 10.1.1.30: ICMP echo request, id 59656, seq 1, length 64
            0x0000:  5254 001b ead5 0200 0000 0000 0800 4500
            0x0010:  0054 0000 4000 4001 2476 0a01 0114 0a01
            0x0020:  011e 0800 cb2f e908 0001 533d 7f52 0000
            0x0030:  0000 ae63 0400 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
        03:01:22.900735 IP 10.1.1.30 > 10.1.1.20: ICMP echo reply, id 59656, seq 1, length 64
            0x0000:  5254 00b8 2775 5254 001b ead5 0800 4500
            0x0010:  0054 fff3 0000 4001 6482 0a01 011e 0a01
            0x0020:  0114 0000 d32f e908 0001 533d 7f52 0000
            0x0030:  0000 ae63 0400 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
    

ftrace debug(send):

  * `cd /sys/kernel/debug/tracing`
  * `echo "iptunnel_xmit" > set_ftrace_filter`
  * `echo "function" > current_tracer`
  * `echo "1" > tracing_on`

        # cat trace
        # tracer: function
        #
        # entries-in-buffer/entries-written: 6/6   #P:4
        #
        #                              _-----=> irqs-off
        #                             / _----=> need-resched
        #                            | / _---=> hardirq/softirq
        #                            || / _--=> preempt-depth
        #                            ||| /     delay
        #           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
        #              | |       |   ||||       |         |
            upcall_6-2529  001 ..s. 78269.070472: iptunnel_xmit -lisp_send
             vhost-31615-31617 003 ..s. 78270.070891: iptunnel_xmit -lisp_send
             vhost-31615-31617 000 ..s. 78271.072337: iptunnel_xmit -lisp_send
             vhost-31615-31617 000 ..s. 78272.073472: iptunnel_xmit -lisp_send
             vhost-31615-31617 000 ..s. 78273.075184: iptunnel_xmit -lisp_send
             vhost-31615-31617 002 ..s. 78274.076124: iptunnel_xmit -lisp_send

ftrace debug(reveive):

  * `cd /sys/kernel/debug/tracing`
  * `echo "ovs_vport_receive" > set_ftrace_filter`
  * `echo "function" > current_tracer`
  * `echo "1" > tracing_on`

        # cat trace | grep lisp_rcv
            upcall_5-2530  003 ..s1 78668.265341: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78669.266634: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78670.268925: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78671.269857: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78672.271818: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78673.273648: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78674.275753: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78675.277631: ovs_vport_receive -lisp_rcv
                Xorg-743   002 ..s. 78676.279053: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78677.281010: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78678.282785: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78679.284470: ovs_vport_receive -lisp_rcv
              <idle>-0     002 ..s. 78680.285205: ovs_vport_receive -lisp_rcv

All done.