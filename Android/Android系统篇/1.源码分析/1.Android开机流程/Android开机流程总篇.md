### Android启动流程

Android 系统是基于 Linux 内核的，所以启动过程与 Linux 系统有很多相似的地方。

由于 Android 属于嵌入式设备，并没有像计算机上那样的 BIOS 程序， 取而代之的是 Bootloader —— 系统启动加载器。 它类似于 BIOS，在系统加载前，用以初始化硬件设备，建立内存空间的映像图，为最终调用系统内核准备好环境。

在 Android 里没有硬盘，而是 ROM，它类似于硬盘存放操作系统，用户程序等。 那么 Bootloader 是如何被加载的呢？

#### 1.启动电源以及系统启动

​		与计算机启动过程类似，当按下电源按键后，引导芯片代码开始从预定义的地方（固化在 ROM 中的预设代码）开始执行，芯片上的 ROM 会寻找 Bootloader 代码，并加载到内存（RAM）中。

#### 2.引导程序BootLoader

​	引导程序BootLoader是在Android操作系统开始运行前的一个小程序Bootloader 会读取 ROM 找到操作系统并将 Linux 内核加载到 RAM 中。

#### 3.Linux内核启动

​	当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置时，首先启动Init进程，它主要的任务为创建和挂在一些文件目录，属性服务的初始化以及运行init.rc脚本文件。

引导加载程序流程示例：

1. 引导加载程序会先加载并初始化内存。
2. 如果使用 [A/B 更新](https://source.android.google.cn/devices/bootloader/updating#a-b-updates)，引导加载程序会确定要启动的当前槽位。
3. 引导加载程序确定是否应启动恢复模式（请参阅[支持更新](https://source.android.google.cn/devices/bootloader/updating)）。
4. 引导加载程序加载启动映像，其中包含内核和 ramdisk 映像。
5. 引导加载程序将内核作为可自行执行的压缩二进制文件加载到内存中。然后，内核将自身解压缩并开始执行到内存中。
6. 引导加载程序从 `ramdisk` 分区（在旧款设备上）或 system 分区（在新款设备上）加载 `init`。
7. `init` 从 system 分区中启动并装载其他所有分区（如 `vendor`、`oem` 和 `odm`），然后开始执行代码以启动设备