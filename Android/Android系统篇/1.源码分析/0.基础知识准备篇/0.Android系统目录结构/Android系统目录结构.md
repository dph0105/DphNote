

|  一级目录   |                             作用                             |
| :---------: | :----------------------------------------------------------: |
|    /bin     |                   软链接，指向 /system/bin                   |
|    /data    |                      用户软件和各种数据                      |
|    /dev     |                       设备文件保存位置                       |
|    /etc     |                   软链接，指向/system/etc                    |
|    /lib     |                   软链接，指向/system/lib                    |
| /lost+found | 当系统意外崩溃或意外关机时，产生的一些文件碎片会存放在这里。在系统启动的过程中，fsck 工具会检查这里，并修复已经损坏的文件系统。这个目录只在每个分区中出现，例如，/lost+found 就是根分区的备份恢复目录 |
|    /mnt     | 挂载目录。用来挂载额外的设备，如 U 盘、移动硬盘和其他操作系统的分区 |
|    /proc    | 虚拟文件系统。该目录中的数据并不保存在硬盘上，而是保存到内存中。主要保存系统的内核、进程、外部设备状态和网络状态等。如 /proc/cpuinfo 是保存 CPU 信息的，/proc/devices 是保存设备驱动的列表的，/proc/filesystems 是保存文件系统列表的，/proc/net 是保存网络协议信息的...... |
|    /sbin    | 保存与系统环境设置相关的命令，只有 root 可以使用这些命令进行系统环境设置，但也有些命令可以允许普通用户查看 |
|   /sdcard   |                                                              |
|  /storage   |              软链接，指向/storage/self/primary               |
|    /sys     | 虚拟文件系统。和 /proc 目录相似，该目录中的数据都保存在内存中，主要保存与内核相关的信息 |
|   /system   |                                                              |
|   /vendor   |                  软链接，指向/system/vendor                  |



** /init           系统启动文件**

- /system
  - **app**        系统应用安装目录
  - **bin**         常用的系统本地命令（二进制），大部分是toolbox的链接（类似于嵌入式Linux中的busybox）
  - **etc**         系统配置文件，如hosts
  - **font**        字体目录
  - **framework**   Java平台架构核心库，jar包和odex优化的文件
  - **lib**         系统底层共享库，.so库文件
  - **xbin**        不常用的系统管理工具，相当于linux的/sbin
  - media
    - audio　　铃声，提示音等音频文件， .ogg
      - notifications   通知
      - ui          界面
      - alarms       警告
      - ringtones     铃声
  - usr         用户配置文件，如键盘布局、共享、时区文件等
    - keychars
    - keylayout
    - share
    - srec     配置
    - ......
  - **vendor**  厂商定制代码目录
  - build.prop    系统设置和变更属性
  
- /etc  -->  /system/etc 

- /vendor --> /system/vendor

- /dev            存放设备节点文件

- **/proc**          全局系统信息

- /data  用户软件和各种数据
  - **local/tmp**　　临时目录，无权限要求
  - **app**        普通程序安装目录
  - system
    - location   其中的location.gps记录最后的坐标，LocationManager.getLastKnownLocation()数据来自此处
  - data
    - <package_name>
      - **files**           Context.getFilesDir() ，Context.openFileOutput() 获取的目录，应用安装目录下
      - **cache**         Context.getCacheDir()  获取的目录，应用安装目录下，系统会自动在内存不足或目录大小达到特定数值时自动清理。
      - **shared_pref**     Context.getSharedPreferences() 建立的preferences文件（xml）存放目录
  - **anr**         应用发生ANR（Applicaiton is Not Responding）时，Android将问题点的堆栈写入到traces.txt文件中
  - location
    - gps    GPS location provider配置
  - property     其中persist.sys.timezone记录系统时区

- /sdcard 

   -->/storage/emulated/legacy     SD卡的FAT32文件系统挂载到此目录

  - Android

    - data

      - <package_name>

         应用的额外数据，应用卸载时自动删除。

        - **files**   Context.getExternalFilesDir()获取的目录。设置->应用->具体应用详情-> 清除数据  操作对象就是这个目录。
        - **cache**   Context.getExternalCacheDir()获取的缓存目录。设置->应用->具体应用详情-> 清除缓存  操作对象就是这个目录。