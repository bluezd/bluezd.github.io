---
id: 700
title: OVS GRE Tunnel Configuration
date: 2013-11-10T17:54:49+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=700
permalink: /archives/700
views:
  - "5438"
categories:
  - kernel
  - Linux
  - networking
  - 系统管理
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
    <code>make install</code>
  </li>
  <li>
    <code>make modules_install</code>
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

## GRE tunnel Configuration {#gre-tunnel-configuration}

  * Host1 
      * `ovs-vsctl add-br br1`
      * `ovs-vsctl add-br br2`
      * `ovs-vsctl add-port br1 em1`
      * `ifconfig em1 0;dhclient br1`
      * `ifconfig br2 10.1.1.1 netmask 255.255.255.0`
      * `ovs-vsctl add-port br2 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.1.252`
  * VM1 
      * `virsh dumpxml kvm1 > domain.xml`
      * search the XML `<interface type='network'>` section and change it to something like this(see below):
      * `virsh create domain.xml`
      * `ifconfig eth3 up`
      * `ifconfig eth3 10.1.1.20 netmask 255.255.255.0`

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
            Port "br2"
                Interface "br2"
                type: internal
            Port "gre0"
                Interface "gre0"
                type: gre
                options: {remote_ip="192.168.1.252"}
            Bridge "br1"
            Port "br1"
                Interface "br1"
                type: internal
            Port "em1"
                Interface "em1"

  * Host2 
      * `ovs-vsctl add-br br1`
      * `ovs-vsctl add-br br2`
      * `ovs-vsctl add-port br1 eth0`
      * `ifconfig em1 0;dhclient br1`
      * `ifconfig br2 10.1.1.2 netmask 255.255.255.0`
      * `ovs-vsctl add-port br2 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.1.249`
  * VM2 
      * redid the configuration as VM1
      * `ifconfig eth5 up`
      * `ifconfig eth5 10.1.1.30 netmask 255.255.255.0`

        # ovs-vsctl show
        66f0d773-a7c9-451e-980b-c2def85eda23
            Bridge "br1"
            Port "eth0"
                Interface "eth0"
            Port "br1"
                Interface "br1"
                type: internal
            Bridge "br2"
            Port "gre0"
                Interface "gre0"
                type: gre
                options: {remote_ip="192.168.1.249"}
            Port "br2"
                Interface "br2"
                type: internal
            Port "vnet0"
                Interface "vnet0"

#### Test {#test}

ping VM2 on VM1:

        # ping 10.1.1.30
        PING 10.1.1.30 (10.1.1.30) 56(84) bytes of data.
        64 bytes from 10.1.1.30: icmp_seq=1 ttl=64 time=3.66 ms
        64 bytes from 10.1.1.30: icmp_seq=2 ttl=64 time=1.82 ms
        64 bytes from 10.1.1.30: icmp_seq=3 ttl=64 time=1.47 ms
        64 bytes from 10.1.1.30: icmp_seq=4 ttl=64 time=1.64 ms
        64 bytes from 10.1.1.30: icmp_seq=5 ttl=64 time=1.64 ms

capture the packets on VM2 at the same time:

        # tcpdump -i eth5 -xxx    
        08:24:38.425182 IP 10.1.1.20 > 10.1.1.30: ICMP echo request, id 53255, seq 1, length 64
            0x0000:  5254 001b ead5 5254 00b8 2775 0800 4500
            0x0010:  0054 0000 4000 4001 2476 0a01 0114 0a01
            0x0020:  011e 0800 263f d007 0001 96f1 7852 0000
            0x0030:  0000 2aa1 0900 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
        08:24:38.425262 IP 10.1.1.30 > 10.1.1.20: ICMP echo reply, id 53255, seq 1, length 64
            0x0000:  5254 00b8 2775 5254 001b ead5 0800 4500
            0x0010:  0054 8ef0 0000 4001 d585 0a01 011e 0a01
            0x0020:  0114 0000 2e3f d007 0001 96f1 7852 0000
            0x0030:  0000 2aa1 0900 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
        08:24:39.426066 IP 10.1.1.20 > 10.1.1.30: ICMP echo request, id 53255, seq 2, length 64
            0x0000:  5254 001b ead5 5254 00b8 2775 0800 4500
            0x0010:  0054 0000 4000 4001 2476 0a01 0114 0a01
            0x0020:  011e 0800 d835 d007 0002 97f1 7852 0000
            0x0030:  0000 77a9 0900 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
        08:24:39.426135 IP 10.1.1.30 > 10.1.1.20: ICMP echo reply, id 53255, seq 2, length 64
            0x0000:  5254 00b8 2775 5254 001b ead5 0800 4500
            0x0010:  0054 8ef1 0000 4001 d584 0a01 011e 0a01
            0x0020:  0114 0000 e035 d007 0002 97f1 7852 0000
            0x0030:  0000 77a9 0900 0000 0000 1011 1213 1415
            0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425
            0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435
            0x0060:  3637
    

ftrace debug:

  * `cd /sys/kernel/debug/tracing`
  * `echo "iptunnel_xmit" > set_ftrace_filter`
  * `echo "function_graph" > current_tracer`
  * `echo "1" > tracing_on`

        # cat trace
        # tracer: function_graph
        #
        # CPU  DURATION                  FUNCTION CALLS
        # |     |   |                     |   |   |   |
         0) + 63.180 us   |  iptunnel_xmit [openvswitch]();
         0) + 29.963 us   |  iptunnel_xmit [openvswitch]();
         1) + 51.486 us   |  iptunnel_xmit [openvswitch]();
         2) + 49.981 us   |  iptunnel_xmit [openvswitch]();
         ------------------------------------------
         0)  upcall_-1891  =>  vhost-2-2339 
         ------------------------------------------
    
         0) + 11.770 us   |  iptunnel_xmit [openvswitch]();
         2) + 45.533 us   |  iptunnel_xmit [openvswitch]();
         2) + 48.497 us   |  iptunnel_xmit [openvswitch]();
         2) + 48.216 us   |  iptunnel_xmit [openvswitch]();
         2) + 48.348 us   |  iptunnel_xmit [openvswitch]();

Gre Tunnel 就介绍到这里，之后会简单的介绍下 LISP Tunnel 的配置过程

#### References {#references}

<ol style="list-style-type: decimal">
  <li>
    <a href="http://wangcong.org/blog/archives/2163">http://wangcong.org/blog/archives/2163</a>
  </li>
  <li>
    <a href="http://networkstatic.net/open-vswitch-gre-tunnel-configuration/">http://networkstatic.net/open-vswitch-gre-tunnel-configuration/</a>
  </li>
</ol>