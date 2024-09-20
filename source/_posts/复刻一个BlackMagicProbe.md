---
title: 复刻一个BlackMagicProbe
author: WangGaojie
---

## Black Magic
Black Magic Probe是一个内建GDB server的嵌入式芯片调试器，可以用来调试ARM、riscv等单片机。关于该调试器的详细信息可以参考官[网网](https://black-magic.org/)。

## 克隆版本
我基于官方的固件，设计了一个电路板，做了一个自己的版本，主要的功能同官方版本一样，只是多了目标设备电源控制和电压检测功能。

<center>

![BMP](bmp.png)
*图1. 自己设计制作的BMP*

</center>

## 常用命令
这里记录一些该调试器常用的一些命令，方便以后查阅。
```
# 查询BlackMagic设备的串口号
# powershell
Get-CimInstance -ClassName Win32_SerialPort -Filter "PNPDeviceID like 'USB\\VID_1D50&PID_6018&MI_00%'" | Select -ExpandProperty DeviceID | Set-Variable -Name BMP_GDB_PORT

# 固件一键下载
arm-none-eabi-gdb -nx --batch -ex 'target extended-remote \\.\\COM21' -ex 'monitor swdp_scan' -ex 'attach 1' -ex 'load' -ex 'compare-sections' -ex 'kill' binary.elf

# GDB命令
# 连接调试器
target extended-remote \\.\COM21

# 使用SWD模式扫描设备
monitor swdp_scan

# 使用JTAG模式扫描设备
monitor jtag_scan

# 连接目标设备
attach 1

# 设置下载线频率
monitor freq 900k

# 目标设备供电控制
monitor tpwr enable
monitor tpwr disable

# 下载固件
load binary.elf

# 比较固件
compare-sections

# 使用RTT功能
monitor rtt

# 查看寄存器
info registers

# 运行目标设备
start
run

# 断点调试
break <function>
break <file>:<line>
watch <var>

# 退出调试器
detach
kill

```

## 在STM32CubeIDE中调试
新建一个调试配置，类型选择GDB Hardware Debug。

在Debugger选项框中GDB命令输入arm-none-eabi-gdb。勾选Use remote target，Debug server选择Generic Serial, 协议选择extended-remote，端口选择BMP的串口号(例如\\\\.\COM21 or COM6)。

在Startup选项框中初始命令输入：
mon tpwr enable
mon freq 900k
monitor swdp_scan
attach 1

如果仅调试不下载程序则不勾选load image。可以为下载单独建立一个配置，否则每次下载都要执行一遍下载动作(影响芯片Flash寿命和调试的速度)。

勾选Set breakpoint at, 后面输入main。表示在main函数建立一个默认断点。
在下面的命令框中再输入run指令，这样可以每次启动调试都会复位一次程序，如果不想复位则不输入。

保存该配置即可开始调试。

## 在VSCODE中调试
我最开始的想法是使用VSCODE中原生的调试组件，但是在进行了一些尝试后发现Native Debug存在一些问题。这似乎是cppdbg不太兼容串口远程调试，在执行附加进程时会出现一些故障，可能是GDB server交互存在一些兼容性问题。我在检索相关资料时也发现其它网友存在该问题。

这个无法使用的调试配置也可以参考：
```
    {
        "name": "Black Magic Probe(invalid)",
        "type": "cppdbg",
        "request": "launch",
        "cwd": "${workspaceRoot}",
        "MIMode": "gdb",
        "targetArchitecture": "arm",
        "logging": {"engineLogging": true},
        "program": "${workspaceRoot}/Debug/${workspaceFolderBasename}.elf",
        "miDebuggerPath": "arm-none-eabi-gdb",
        "customLaunchSetupCommands": [
            {"text": "cd ${workspaceRoot}/Debug"},
            {"text": "file ${workspaceFolderBasename}.elf"},
            {"text": "target extended-remote \\\\.\\COM23"},
            {"text": "monitor tpwr enable"},
            {"text": "mon freq 500k"},
            {"text": "monitor swdp_scan"},
            {"text": "attach 1"},
            {"text": "load"},
            {"text": "cd ${workspaceRoot}"},
            {"text": "set mem inaccessible-by-default off"},
            {"text": "break main"}
        ],
        "serverLaunchTimeout": 10000,
        "windows": {
            "miDebuggerPath": "arm-none-eabi-gdb.exe"
        }
    }
```

无法使用原生调试，那就只有依赖于VSCODE插件，很流行的一个调试插件cortex-debug支持Black Magic Probe，这里列出下载脚本和调试脚本。
```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    "version": "0.2.0",
    "configurations": [
        {
            "name": "BMP Download",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceRoot}/Debug/${workspaceFolderBasename}.elf",
            "request": "launch",
            "type": "cortex-debug",
            "device": "",
            "runToEntryPoint": "main",
            "showDevDebugOutput": "raw",
            "servertype": "bmp",
            "interface": "swd",
            "BMPGDBSerialPort": "\\\\.\\COM23",
            "powerOverBMP": "enable",
            "preLaunchCommands": ["mon freq 500k"]
        },
        {
            "name": "BMP Debug",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceRoot}/Debug/${workspaceFolderBasename}.elf",
            "request": "attach",
            "type": "cortex-debug",
            "device": "",
            "runToEntryPoint": "main",
            "showDevDebugOutput": "raw",
            "servertype": "bmp",
            "interface": "swd",
            "BMPGDBSerialPort": "\\\\.\\COM23",
            "powerOverBMP": "enable",
            "preAttachCommands": ["mon freq 500k"]
        }
    ]
}
```

Black Magic还支持RTT功能，可以使用postAttachCommands选项去手动开启，我简单测试了一下，似乎会影响调试功能。

PS:[cortex-debug参数说明](https://github.com/Marus/cortex-debug/blob/master/debug_attributes.md)

## 总结
Black Magic Probe非常的灵活好用，不依赖特别的驱动，支持主流的三大操作系统，且不需要类似openocd的中间适配层就能实现完整的嵌入式调试。

相较于Jlink等调试器来说，它的性能不算特别强大，但它可以和GDB调试器直接交互，使得它的灵活性更强。

目前该调试器的缺陷是速度问题，USB2.0FS在某些情况下可能会出现一些意外的情况。后续我想制作一个高速版本的BMP来改进使用体验。
