---
title: Develop and Deploy Mesos Hook
author: bluezd
layout: post
permalink: /archives/783
categories:
  - mesos
tags:
  - mesos
---

## Introduction

Here is the detail introduction regarding mesos module [http://mesos.apache.org/documentation/latest/modules/](http://mesos.apache.org/documentation/latest/modules/), it supports a vast variety of modules, this article aims to introduce Hook.

## Developing Hook

#### Usecase

User is able to write hooks for fulfilling the following functionalities:

  - docker mount path checking *eg: for an illegal mount path hook will reject the task running* 
  - docker `privileged` checking *eg: hook will check the flag and perform corresponding actions based on certain circumstances*
  - etc.

#### Scaffold Codes

Below are the example hook codes, user can write their own logic in `slavePreLaunchDockerTaskExecutorDecorator` method:

```c++
#include <limits.h>
#include <stdlib.h>
#include <mesos/hook.hpp>
#include <mesos/mesos.hpp>
#include <mesos/module.hpp>

#include <mesos/module/hook.hpp>

#include <process/future.hpp>
#include <process/id.hpp>
#include <process/process.hpp>
#include <process/protobuf.hpp>

#include <stout/foreach.hpp>
#include <stout/os.hpp>
#include <stout/try.hpp>

#include "messages/messages.hpp"

#include <string>
#include <iostream>
#include <vector>

using namespace mesos;
using std::map;
using std::string;

using process::Failure;
using process::Future;

class MesosHook : public Hook {
public:
    MesosHook();

    /**
     * @param taskInfo
     * @param executorInfo
     * @param containerName
     * @param containerWorkDirectory
     * @param mappedSandboxDirectory
     * @param env
     * @return
     */
    virtual Future<Option<DockerTaskExecutorPrepareInfo>>
    slavePreLaunchDockerTaskExecutorDecorator(
            const Option<TaskInfo> &taskInfo,
            const ExecutorInfo &executorInfo,
            const string &containerName,
            const string &containerWorkDirectory,
            const string &mappedSandboxDirectory,
            const Option<map<string, string>> &env) {
        LOG(INFO) << "containerName: " + containerName;

        const Resource& resource = executorInfo.resources(0);
        LOG(INFO) << "resource role: " +resource.allocation_info().role();

        if (!taskInfo->has_container()) {
            return None();
        }

        if (taskInfo->container().type() == ContainerInfo::DOCKER) {
            LOG(INFO) << "Got a Docker Container";
        } else if (taskInfo->container().type() == ContainerInfo::MESOS) {
            LOG(INFO) << "Got a Mesos Container";
            // For now we ingore mesos container
            return None();
        } else {
            LOG(WARNING) << "Got a Unknown Type Container";
            return None();
        }

        if (!taskInfo->container().has_docker()) {
            return Failure("Check Docker Container failed");
        }
        ....
};

static Hook *createHook(const Parameters &parameters) {
    return new MesosHook();
}

mesos::modules::Module<Hook> xxx_Mesoshook(
        MESOS_MODULE_API_VERSION,
        MESOS_VERSION,
        "Company",
        "mail",
        "Despription",
        nullptr,
        createHook);
```

## Compile Mesos Hook

Previously we installed the mesos environment successfully [https://bluezd.github.io/archives/782](https://bluezd.github.io/archives/782), Now let's head to compile mesos hook by taking advantage of existing mesos environment. 

Replace variable `MESOS_SOURCE_PATH` and `MESOS_INSTALL_PATH` accordingly then execute the following shell script:

```sh
#!/bin/bash

set -e

MESOS_SOURCE_PATH="/root/mesos-1.9.0"
MESOS_BUILD_PATH="${MESOS_SOURCE_PATH}/build"
MESOS_INSTALL_PATH="/usr/local/mesos"

function compile_gcc () {
    if [[ -d build-gcc ]]; then
        rm -f  build-gcc/*.so
    else
        mkdir build-gcc
    fi

    cd build-gcc

    c++ -std=c++11 \
        -I $MESOS_INSTALL_PATH/lib/mesos/3rdparty/include \
        -I $MESOS_INSTALL_PATH/include/ \
        -I $MESOS_BUILD_PATH/src \
        -I $MESOS_SOURCE_PATH/src \
        -lmesos \
        -fpic \
        -o mesos_hook.o \
        -c ../mesos_hook.cpp

    gcc -shared -o libmesos_hook.so mesos_hook.o -lboost_filesystem -lboost_system
}

compile_gcc
```

The shared lib `libmesos_hook.so` will be placed in `build-gcc` dir.

## Deploy Mesos Hook

#### prepare hook configuration file

Add following configurations to `/etc/mesos/agent-modules.json`:

```sh
     {
       "file": "/root/mesos-hook/build-gcc/libmesos_hook.so",
       "modules": [
         {
           "name": "xxx_Mesoshook"
         }
       ]
     }
```

The `name` must exactly same with `mesos::modules::Module<Hook> xxx_Mesoshook` in cpp file.

If you hope to pass parameters to your hook, add parameters defination as below:

```sh
     {
       "file": "/root/mesos-hook/build-gcc/libmesos_hook.so",
       "modules": [
         {
           "name": "xxx_Mesoshook",
           "parameters" :
              [
                {
                  "key": "xx",
                  "value" : "xxx"
                }
              ]
         }
       ]
     }
```

This the final file looks like:

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
     },
     {
       "file": "/root/mesos-hook/build-gcc/libmesos_hook.so",
       "modules": [
         {
           "name": "xxx_Mesoshook",
            "parameters" :
              [
                {
                  "key": "xx",
                  "value" : "xxx"
                }
              ]
         }
       ]
     }
  ]
}
```

#### starting mesos agent:

Add `--hooks=xxx_Mesoshook` to mesos agent startup options and start the agent:

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
    --modules=file:///etc/mesos/agent-modules.json \
    --hooks=xxx_Mesoshook
```

## What happend behind the scenes ?

#### mesos agent loads modules and initialize on startup

  1. Load modules `Try<Nothing> result = ModuleManager::load(flags.modules.get());` in *`src/slave/main.cpp`*
    1. `Try<Nothing> ModuleManager::loadManifest(const Modules& modules)` in *`src/module/manager.cpp`*
    2. `moduleBases[moduleName] = moduleBase;`
  2. Initialize hooks `Try<Nothing> result = HookManager::initialize(flags.hooks.get());` in *`src/slave/main.cpp`*
     1. `Try<Hook*> module = ModuleManager::create<Hook>(hook);` in *`src/hook/manager.cpp`*
         1.  `Module<T>* module = (Module<T>*) moduleBases[moduleName];` in *`src/module/manager.hpp`*
         2.  `T* instance = module->create(params.isSome() ? params.get() : moduleParameters[moduleName]);` invoke `createHook` function
     2. `availableHooks[hook] = module.get();` in *`src/hook/manager.cpp`*

#### call mesos hook logic on task startup

  - `Future<Containerizer::LaunchResult> DockerContainerizerProcess::launch()` in *`src/slave/containerizer/docker.cpp`* 
     - `f = HookManager::slavePreLaunchDockerTaskExecutorDecorator()`
        - `hook->slavePreLaunchDockerTaskExecutorDecorator()` in `src/hook/manager.cpp` call user logic

*The end.*