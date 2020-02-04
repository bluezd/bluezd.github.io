---
title: Deploy mesos environment with overlay net from scratch
author: bluezd
layout: post
permalink: /archives/782
categories:
  - mesos
tags:
  - mesos
---

## Introduction

DC/OS is the perfect platform for running both docker and **mesos** workloads, it builds on top of apache mesos  and supports additional advanced features (eg: network, service catalog, etc.) It provides comprehensive installers for both Cloud(aws, azure and google) and On-Prem. If users are interested in the principle under the hood they can also provision their own mesos env from scratch with advanced overlay network support(dcos-net powered by D2IQ).

This article aims to give you a guidence of deploying a pure mesos developing/testing environment with the following capabilities:

  1. **overlay network** by taking advantage of dcos-net module
  2. **DNS resolution**
     1. **mesos-dns** by taking advantage of mesos-dns module
     2. **dcos-dns** by taking advantage of dcos-dns(in dcos-net) module
  3. **container orchestration platform** by using marathon

Now Let's get started.

## Environment preparation

At least two machines:

  1. CentOS 7.6 (for mesos master) ip **172.16.9.101**
  2. CentOS 7.6 (for mesos agent) ip **172.16.9.59**

## Install Depedencies on both master and slave

Execute the following commands:
```sh
#!/bin/bash

yum groupinstall -y "Development Tools"
yum install -y maven python-devel python-six python-virtualenv java-1.8.0-openjdk-devel \
               zlib-devel libcurl-devel openssl-devel cyrus-sasl-devel cyrus-sasl-md5 \
               apr-devel subversion-devel apr-util-devel vim wget

yum install -y gcc-c++ \
               ncurses-devel libevent-devel systemd-devel libsodium-devel

yum install epel-release -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
```

## Compile and Install dcos-net on both master and slave

#### Install `kerl`

  1. `curl -o /usr/local/bin/kerl https://raw.githubusercontent.com/kerl/kerl/master/kerl`
  2. `chmod a+x /usr/local/bin/kerl`

#### Compiling Erlang

```sh
KERL_CONFIGURE_OPTIONS=
KERL_CONFIGURE_OPTIONS+="--enable-dirty-schedulers "
KERL_CONFIGURE_OPTIONS+="--enable-kernel-poll "
KERL_CONFIGURE_OPTIONS+="--with-ssl"
KERL_CONFIGURE_OPTIONS+="--without-javac "
export KERL_CONFIGURE_OPTIONS

kerl build 22.0 22.0
kerl install 22.0 $HOME/erl

# kerl status
Available builds:
22.0,22.0
----------
Available installations:
22.0 /root/erl
----------
The current active installation is:
/root/erl
There is no Dialyzer PLT for the active installation
The build options for the active installation are:

--enable-dirty-schedulers --enable-kernel-poll --with-ssl--without-javac
```

## Compile and Install Mesos on both master and slave

  1. `wget http://archive.apache.org/dist/mesos/1.9.0/mesos-1.9.0.tar.gz`
  2. `tar -zxvf mesos-1.9.0.tar.gz`
  3. `cd mesos-1.9.0`
  4. `mkdir build; cd build`
  5. `../configure --enable-libevent --enable-ssl --enable-install-module-dependencies --enable-launcher-sealing --disable-libtool-wrappers --prefix=/usr/local/mesos/`
  6. `make -j $(getconf _NPROCESSORS_ONLN)`
  7. `make install`

## Compile and Install dc/os net overlay module on both master and slave

1. `yum install libsodium-devel -y`
2. `git clone https://github.com/dcos/dcos-mesos-modules.git`
3. `./bootstrap`
4. `mkdir build && cd build` 
5. `../configure --with-mesos=/usr/local/mesos --prefix=/usr/local/dcos-mesos-modules --with-protobuf=/usr/local/mesos/lib/mesos/3rdparty`
6. `make -j $(getconf _NPROCESSORS_ONLN)`
7. `make install`

## Running a batch of services on master

#### starting exhibitor with zookeeper

```sh
#!/bin/bash

NODE_1_IP=172.16.9.101

docker run --rm -it -d \
     --name exhibitor \
     --net bridge \
     -p 2181:2181/tcp \
     -p 2888:2888/tcp \
     -p 3888:3888/tcp \
     -p 8181:8080/tcp \
     netflixoss/exhibitor:1.5.2 \
     --hostname $NODE_1_IP
```
#### starting mesos master

```sh
mkdir -p /etc/mesos/cni
```

```sh
#!/bin/bash

NODE_1_IP=172.16.9.101
export MESOS_MASTER=zk://localhost:2181/mesos

export PATH=$PATH:/usr/local/mesos/sbin/
mesos-master \
     --cluster=dcos-net-cluster \
     --hostname=$NODE_1_IP \
     --ip=0.0.0.0 \
     --log_dir=/var/log/mesos \
     --port=5050 \
     --quorum=1 \
     --zk=zk://$NODE_1_IP:2181/mesos \
     --work_dir=/var/lib/mesos/master \
     --modules=file:///etc/mesos/master-modules.json
```

```sh
# cat /etc/mesos/master-modules.json
{
  "libraries":
  [
    {
      "file": "/usr/local/dcos-mesos-modules/lib/mesos/libmesos_network_overlay.so",
      "modules":
        [
          {
            "name": "com_mesosphere_mesos_OverlayMasterManager",
            "parameters" :
              [
                {
                  "key": "master_config",
                  "value" : "/etc/mesos/overlay/master-config.json"
                }
              ]
          },
          {
            "name": "com_mesosphere_mesos_OverlayAgentManager",
            "parameters" :
              [
                {
                  "key": "agent_config",
                  "value" : "/etc/mesos/overlay/master-agent-config.json"
                }
              ]
          }
        ]
     }
  ]
}
```

```sh
# cat /etc/mesos/overlay/master-config.json
{
    "replicated_log_dir":"/var/lib/mesos/master",
    "network": {
        "vtep_mac_oui": "70:B3:D5:00:00:00",
        "overlays": [{
            "prefix": 24,
            "name": "dcos",
            "subnet": "9.0.0.0/8"
        }],
        "vtep_subnet": "44.128.0.0/20"
    }
}
```

```sh
# cat /etc/mesos/overlay/master-agent-config.json
{
    "cni_dir": "/etc/mesos/cni",
    "cni_data_dir": "/var/run/mesos/cni/networks",
    "network_config" : {
        "allocate_subnet": true,
        "mesos_bridge": false,
        "docker_bridge": false,
        "overlay_mtu": 1420,
        "enable_ipv6": false
    },
    "max_configuration_attempts": 5
}
```

#### starting mesos dns

  1. downlaod mesos dns from [https://github.com/mesosphere/mesos-dns/releases](https://github.com/mesosphere/mesos-dns/releases)
  2. `/root/mesos-dns-v0.7.0-rc2-linux-amd64 -config /etc/mesos-dns.json`

```sh
# cat /etc/mesos-dns.json
{
    "zk": "zk://172.16.9.101:2181/mesos",
    "refreshSeconds": 30,
    "ttl": 60,
    "domain": "mesos",
    "port": 61053,
    "resolvers": ["8.8.8.8"],
    "timeout": 5,
    "listener": "0.0.0.0",
    "email": "root.mesos-dns.mesos",
    "IPSources": ["host", "netinfo"],
    "SetTruncateBit": true
}
```

#### configuring and running dcos-net

Add the following entries to `/etc/resolv.conf`

```
nameserver 198.51.100.1
nameserver 198.51.100.2
nameserver 198.51.100.3
```

```sh
#!/bin/bash

modprobe ip_vs_wlc
modprobe dummy

iptables --wait -A FORWARD -j ACCEPT
iptables --wait -t nat -I POSTROUTING -m ipvs --ipvs --vdir ORIGINAL \
         --vmethod MASQ -m comment \
         --comment Minuteman-IPVS-IPTables-masquerade-rule \
         -j MASQUERADE
ip link add minuteman type dummy
ip link set minuteman up
ip link add spartan type dummy
ip link set spartan up
ip addr add 198.51.100.1/32 dev spartan
ip addr add 198.51.100.2/32 dev spartan
ip addr add 198.51.100.3/32 dev spartan
```

starting dcos-net

```sh
git clone https://github.com/dcos/dcos-net.git
```

```sh
#!/bin/bash

source $HOME/erl/activate

cd dcos-net
export MASTER_SOURCE=exhibitor
export EXHIBITOR_ADDRESS=172.16.9.101

./rebar3 shell \
        --config /etc/dcos-net/master.config \
        --name dcos-net@172.16.9.101 \
        --setcookie dcos-net \
        --apps dcos_dns,dcos_l4lb,dcos_overlay,dcos_rest,dcos_net
```

```erlang
# cat /etc/dcos-net/master.config
{% raw %}[
{dcos_net,
  [{dist_port, 62501},
   {is_master, true}]},

 {dcos_dns,
  [{udp_port, 53},
   {tcp_port, 53},
   {zookeeper_servers, [{"172.16.9.101", 2181}]},
   {upstream_resolvers,
    [{{8,8,8,8}, 53}]},
   {zones,
    #{"component.thisdcos.directory" =>
          [#{
             type => cname,
             name => "registry",
             value => "master.dcos.thisdcos.directory"
            }]}}]},

 {dcos_l4lb,
  [{master_uri, "http://$172.16.9.101:5050/master/state"},
   {enable_lb, true},
   {enable_ipv6, false},
   {min_named_ip, {11,0,0,0}},
   {max_named_ip, {11,255,255,255}},
   {min_named_ip6, {16#fd01,16#c,16#0,16#0,16#0,16#0,16#0,16#0}},
   {max_named_ip6, {16#fd01,16#c,16#0,16#0,16#ffff,16#ffff,16#ffff,16#ffff}},
   {agent_polling_enabled, false}]},

 {dcos_overlay,
  [{enable_overlay, true},
   {enable_ipv6, false}]},

 {dcos_rest,
  [{enable_rest, true},
   {ip, {0,0,0,0}},
   {port, 62080}]},

 {erldns,
  [{servers, []},
   {pools, []}]},

 {telemetry,
  [{forward_metrics, false}]}
].{% endraw %}
```

#### running marathon framework

download marathon 1.8
  
`wget https://downloads.mesosphere.io/marathon/builds/1.8.222-86475ddac/marathon-1.8.222-86475ddac.tgz` 

run marathon
```sh
#!/bin/bash

export MESOS_NATIVE_JAVA_LIBRARY=/usr/local/mesos/lib/libmesos.so
/root/marathon-1.8.222-86475ddac/bin/marathon --master zk://localhost:2181/mesos --zk=zk://localhost:2181/marathon
```

## Running service on agent

#### running mesos agent

install container networking plugins:

  1. install golang env
     1. `wget https://dl.google.com/go/go1.13.6.linux-amd64.tar.gz` 
     2. `tar -C /usr/local -xzf go1.13.6.linux-amd64.tar.gz`
     3. `export PATH=$PATH:/usr/local/go/bin`
  2. build container networking plugins  
     1. `git clone https://github.com/containernetworking/plugins.git`
     2. `cd plugins && ./build_linux.sh`
  3. install container networking plugins 
     1. `mkdir -p /etc/mesos/active/cni/`
     2. `ln -s /root/plugins/bin/bridge /etc/mesos/active/cni/`
     3. `ln -s /root/plugins/bin/host-local /etc/mesos/active/cni/`
     4. `ln -s /usr/local/mesos/libexec/mesos/mesos-cni-port-mapper /etc/mesos/active/cni/`

prepare conf file:

```sh
# cat /etc/mesos/agent-modules.json
{
  "libraries":
  [
    {
      "file": "/usr/local/dcos-mesos-modules/lib/mesos/libmesos_network_overlay.so",
      "modules":
        [
          {
            "name": "com_mesosphere_mesos_OverlayAgentManager",
            "parameters" :
              [
                {
                  "key": "agent_config",
                  "value" : "/etc/mesos/overlay/agent-config.json"
                }
              ]
          }
        ]
     }
  ]
}
```

```sh
# cat /etc/mesos/overlay/agent-config.json
{
    "master": "172.16.9.101:5050",
    "cni_dir":"/etc/mesos/cni",
    "cni_data_dir": "/var/run/mesos/cni/networks",
    "network_config": {
        "allocate_subnet": true,
        "mesos_bridge": true,
        "docker_bridge": false,
        "overlay_mtu": 1420,
        "enable_ipv6": false
    },
    "max_configuration_attempts": 5
}
```

```sh
# cat /etc/mesos/cni/dcos.conf
{
  "name": "dcos",
  "type": "mesos-cni-port-mapper",
  "excludeDevices": ["m-dcos"],
  "chain": "M-DCOS",
  "delegate": {
    "type": "bridge",
    "bridge": "m-dcos",
    "isGateway": true,
    "ipMasq": false,
    "hairpinMode": true,
    "mtu": 1420,
    "ipam": {
      "type": "host-local",
      "dataDir": "/var/run/mesos/cni/networks",
      "subnet": "9.0.0.0/25",
      "routes": [{
        "dst": "0.0.0.0/0"
      }]
    }
  }
}
```

starting mesos agent:

```sh
yum install ipset -y
systemctl enable docker;systemctl start docker
mkdir -p /var/run/mesos/cni/networks
```

```sh
#!/bin/bash

NODE_1_IP=172.16.9.101
NODE_2_IP=172.16.9.59

export PATH=$PATH:/usr/local/mesos/sbin/

mesos-agent \
    --containerizers=mesos,docker \
    --hostname=$NODE_2_IP \
    --image_providers=docker \
    --ip=0.0.0.0 \
    --isolation=docker/runtime,network/cni,filesystem/linux,volume/sandbox_path \
    --network_cni_config_dir=/etc/mesos/cni \
    --network_cni_plugins_dir=/etc/mesos/active/cni \
    --launcher_dir=/usr/local/mesos/libexec/mesos \
    --log_dir=/var/log/mesos \
    --master=zk://$NODE_1_IP:2181/mesos \
    --port=5051 \
    --work_dir=/var/lib/mesos/agent \
    --no-systemd_enable_support \
    --modules=file:///etc/mesos/agent-modules.json
```

#### configuring and running dcos-net

Add the following entries to `/etc/resolv.conf`

```
nameserver 198.51.100.1
nameserver 198.51.100.2
nameserver 198.51.100.3
```

```sh
#!/bin/bash

modprobe ip_vs_wlc
modprobe dummy

iptables --wait -A FORWARD -j ACCEPT
iptables --wait -t nat -I POSTROUTING -m ipvs --ipvs --vdir ORIGINAL \
         --vmethod MASQ -m comment \
         --comment Minuteman-IPVS-IPTables-masquerade-rule \
         -j MASQUERADE
ip link add minuteman type dummy
ip link set minuteman up
ip link add spartan type dummy
ip link set spartan up
ip addr add 198.51.100.1/32 dev spartan
ip addr add 198.51.100.2/32 dev spartan
ip addr add 198.51.100.3/32 dev spartan
```

starting dcos-net

```sh
git clone https://github.com/dcos/dcos-net.git
```

```sh
#!/bin/bash

source $HOME/erl/activate

cd dcos-net
export MASTER_SOURCE=exhibitor
export EXHIBITOR_ADDRESS=172.16.9.101

/root/dcos-net/rebar3 shell \
        --config /etc/dcos-net/agent.config \
        --name dcos-net@172.16.9.59 \
        --setcookie dcos-net \
        --apps dcos_dns,dcos_l4lb,dcos_overlay,dcos_rest,dcos_net
```

```erlang
# cat /etc/dcos-net/agent.config
{% raw %}[
 {dcos_net,
  [{dist_port, 62501},
   {is_master, false}]},

 {dcos_dns,
  [{udp_port, 53},
   {tcp_port, 53},
   {mesos_resolvers, []},
   {zookeeper_servers, [{"172.16.9.101", 2181}]},
   {upstream_resolvers,
    [{{8,8,8,8}, 53}]}]},

 {dcos_l4lb,
  [{master_uri, "http://172.16.9.101:5050/master/state"},
   {enable_networking, true},
   {enable_lb, true},
   {enable_ipv6, false},
   {min_named_ip, {11,0,0,0}},
   {max_named_ip, {11,255,255,255}},
   {min_named_ip6, {16#fd01,16#c,16#0,16#0,16#0,16#0,16#0,16#0}},
   {max_named_ip6, {16#fd01,16#c,16#0,16#0,16#ffff,16#ffff,16#ffff,16#ffff}},
   {agent_polling_enabled, true}]},

 {dcos_overlay,
  [{enable_overlay, true},
   {enable_ipv6, false}]},

 {dcos_rest,
  [{enable_rest, true},
   {ip, {0,0,0,0}},
   {port, 62080}]},

 {erldns,
  [{servers, []},
   {pools, []}]}
].{% endraw %}
```

## Running tasks

#### accessing ui

  1. mesos ui [http://master-ip:5050](http://master-ip:5050)
  2. marathon ui [http://master-ip:8080](http://master-ip:8080)

#### running tasks in marathon

```json
{
  "id": "/nginx-mesos",
  "cmd": null,
  "cpus": 1,
  "mem": 128,
  "disk": 0,
  "instances": 1,
  "container": {
    "type": "MESOS",
    "docker": {
      "forcePullImage": false,
      "image": "nginx:latest",
      "parameters": [],
      "privileged": false
    },
    "volumes": [],
    "portMappings": [
      {
        "containerPort": 0,
        "labels": {},
        "name": "default",
        "protocol": "tcp",
        "servicePort": 10000
      }
    ]
  },
  "networks": [
    {
      "name": "dcos",
      "mode": "container"
    }
  ],
  "portDefinitions": [],
  "maxLaunchDelaySeconds": 300
}
```

#### verifying results

```sh
# ping nginx-mesos.marathon.containerip.dcos.thisdcos.directory
PING nginx-mesos.marathon.containerip.dcos.thisdcos.directory (9.0.2.2) 56(84) bytes of data.
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=2 ttl=64 time=0.039 ms
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=3 ttl=64 time=0.041 ms
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=4 ttl=64 time=0.042 ms
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=5 ttl=64 time=0.043 ms
64 bytes from 9.0.2.2 (9.0.2.2): icmp_seq=6 ttl=64 time=0.040 ms
```

```html
# curl nginx-mesos.marathon.containerip.dcos.thisdcos.directory
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

*The end.*