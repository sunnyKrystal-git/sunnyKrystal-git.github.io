---
layout: post
title: VS Code搭建EPICS开发环境
date: 2025-01-06 15:30 +0800
categories: [Tech]
tags: [VS Code, EPICS]
description: 记录利用VS Code搭建EPICS开发环境的步骤和过程中遇到的问题。
---



- MacBook Pro 2020
- Mac OS 15.2

## 开始前准备

### testIOC

创建一个IOC用于测试开发环境搭建是否成功。

```shell
# testIOC目录
├── Makefile
├── README.md
├── configure
│   ├── CONFIG
│   ├── CONFIG_SITE
│   ├── Makefile
│   ├── RELEASE
│   ├── RULES
│   ├── RULES.ioc
│   ├── RULES_DIRS
│   └── RULES_TOP
├── iocBoot
│   ├── Makefile
│   └── ioctest
│       ├── Makefile
│       ├── st.cmd
└── testApp
    ├── Db
    │   ├── Makefile
    ├── Makefile
    └── src
        ├── Makefile
        └── testMain.cpp
```

### VS Code插件

- [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
- [Makefile](https://marketplace.visualstudio.com/items?itemName=ms-vscode.makefile-tools) EPICS is Makefile Project

## 添加编译选项

### testIOC

在`testIOC/configure/`添加一个文件`RULES.debug`

```makefile
# @filename: RULES.debug
# RULES for debugging

USR_CFLGAS += -g
USR_CXXFLAGS += -g
```

在`testIOC/configure/Makefile`文件中引用RULES.debug：

```makefile
include $(TOP)/configure/RULES       # 在此行后面，
include $(TOP)/configure/RULES.debug # 添加这行
```

### EPICS Base

参照前面testIOC添加编译选项的方法。完成后，才可在debug时定位到Base的代码。

## 配置 VS Code

### 创建launch.json

点击左侧导航栏`Run and Debug` --> `create a launch.json file` --> `C++(GDB/LLDB)`



![](/assets/img/250106/launch_1.png){: w="600"}

选择`C/C+: (lldb) Launch`。根据实际使用的工具，也可能是gdb launch。


![](/assets/img/250106/launch_2.png){: w="500" h="400"}

**完成创建后，在`.vscode`目录下会出现`launch.json`**

### 配置launch.json

**`launch.json`全文如下**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "LaunchIOC",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/testIOC/bin/darwin-x86/test",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}/testIOC/iocBoot/ioctest/",
            "environment": [],
            "externalConsole": true,
            "MIMode": "lldb"
        }
    ]
}
```

**`name`**

给这组debug configuration起个名字。

**`program`**

`program`指向可执行文件。
`${workspaceFolder}`是 VS Code的工作目录，这里我在testIOC的上一级目录中打开VS Code。
testIOC的可执行文件地址查询`testIOC/iocBoot/ioctest/st.cmd`。

```shell
# Example
#!../../bin/darwin-x86/test
```

**`cwd`**

启动后，会进入`cwd`指定的目录。这里先将它设置为`testIOC/iocBoot/ioctest/`，作用在后面说明。

**`externalConsole`**

设置启动后是否开启新的console。
理论上，可以设置为`false`，在VS Code内嵌的console交互，在之前的环境中我采用的是这样的方式。
但这次重新搭建环境莫名无法使用内嵌console，暂未解决😓，先将就用。

## 开始Debug！

#### 编译

确认EPICS Base和test IOC都编译好。可在编译时看到，编译命令中已有`-g`选项。

#### 设置断点
在目标位置设置断点（下图是没有实际意义的演示😳）

![](/assets/img/250106/debug_3.png){: w="600"}

### 开始Debug（真的开始

点击左侧导航栏`Run and Debug` -> `LaunchIOC`（对应`launch.json`中`name`的配置）

![](/assets/img/250106/debug_1.png){: w="300"  h="400" }

此时会启动一个新的console：
![](/assets/img/250106/debug_2.png){: w="600"}

这时候只是启动了iocshell，并没有加载任何dbd、records等对象。
还需要执行st.cmd中的命令，完成环境变量设置、IOC初始化、加载DB等工作。

**方法一**

由于我们前面在`launch.json`将`cwd`设置为`st.cmd`所在目录，所以此时可以在启动的console中逐一输入`st.cmd`中的命令。

```shell
## #!../../bin/darwin-x86/test
## 不要执行st.cmd第一行⬆

< envPaths

cd "${TOP}"

## Register all support components
dbLoadDatabase "dbd/test.dbd"
test_registerRecordDeviceDriver pdbbase

## Load record instances
dbLoadRecords("db/test.db","user=krystal")

cd "${TOP}/iocBoot/${IOC}"
iocInit

## Start any sequence programs
#seq sncxxx,"user=krystal"
```

**方法二（推荐）**

在`st.cmd`所在目录创建一个新的文件`start_debug`，将`envPaths`和`st.cmd`的命令复制到`start_debug`。
在iocshell中执行`< start_debug`即可。

```shell
## @filename: start_debug

epicsEnvSet("IOC","ioctest")
epicsEnvSet("TOP","/Users/krystal/EPICS/testIOC")
epicsEnvSet("EPICS_BASE","/Users/krystal/EPICS/base")

cd "${TOP}"

## Register all support components
dbLoadDatabase "dbd/test.dbd"
test_registerRecordDeviceDriver pdbbase

## Load record instances
dbLoadRecords("db/test.db","user=krystal")

cd "${TOP}/iocBoot/${IOC}"
iocInit
```

![](/assets/img/250106/debug_4.png){: w="600"}

**到此，~~愉快的~~Debug之旅就开始了**