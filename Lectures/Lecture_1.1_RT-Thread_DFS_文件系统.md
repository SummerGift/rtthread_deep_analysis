# RT-Thread DFS 文件系统

## 1. 简介

### 1.2 文件系统功能

- 文件系统是一套实现了数据的存储、分级组织、访问和获取等操作的抽象数据类型，是一种用于向用户提供底层数据访问的机制。文件系统通常存储的基本单位是文件，即数据是按照一个个文件的方式进行组织。在这里介绍的 DFS 即在 RT-Thread 操作系统中的文件系统，对用户提供的接口主要以 POSIX 标准接口为主，这样也能够保证程序可以在PC 上编写、调试，然后再移植到 RT-Thread 操作系统上。

### 1.2 文件系统结构

RT-Thread 的文件系统采用了三层结构:

- 最顶层：最顶层的是一套面向嵌入式系统，专门优化过的虚拟文件系统（接口），设备虚拟文件系统 POSIX 文件接口，这一层有点像是 Linux 中的虚拟文件系统 VFS。
- 中间层：各种文件系统的实现,如 fatfs, romfs, devfs等。
- 最底层：各类存储驱动，例如SD 卡驱动，IDE 硬盘驱动等，使得存储器可以支持相应的文件系统。  

## 2. Menuconfig 中 DFS 配置项介绍
- Using device virtual file system ： 使用设备虚拟文件系统，是一种轻量级的虚拟文件系统。
- Using working directory : 关闭这个选项，那么在使用文件、目录接口进行操作时应该使用绝对目录进行（因为此时系统中不存在当前工作的目录）。如果需要使用当前工作目录以及相对目录，就要打开这个选项。
- The maximal number of mounted file system ：最大挂载文件系统的数量
- The maximal number of file system type ： 最多支持的文件系统类型
- The maximal number of opened files ： 打开文件的最大数量
- Enable elm-chan fatfs ： 使用 elm-chan FatFs
- elm-chan's FatFs, Generic FAT Filesystem Module : elm-chan 文件系统的配置项
- Using devfs for device objects ： 使用设备文件系统作为设备对象
- Enable BSD socket operated by file system API ： 使 BSD socket 可以使用文件系统的 API 来管理，比如读写操作和被 select/poll 的 POSIX API 调用。 
- Enable ReadOnly file system on flash ： 在 Flash 上使用只读文件系统
- Enable RAM file system ： 使用 RAM 文件系统
- Enable UFFS file system: Ultra-low-cost Flash File System ：使用 UFFS
- Enable JFFS2 file system : 使用 JFFS2 文件系统
- Using NFS v3 client file system ：使用 NFS 文件系统

## 3. 虚拟文件系统的初始化（在 RT-Thread 中简称 DFS）
- 和 DFS 相关的宏如下：
```
#define RT_USING_DFS                   //使用设备虚拟文件系统           
#define DFS_USING_WORKDIR              //使用相对路径
#define DFS_FILESYSTEMS_MAX 2          //最多支持挂载两个文件系统
#define DFS_FILESYSTEM_TYPES_MAX 2     //最多支持两种文件系统类型，这里平时开启elmfs和devfs两种类型
#define DFS_FD_MAX 64                  //文件最大打开数量为 64 个  
#define RT_USING_DFS_ELMFAT            //使用 elm FS，下面进行配置
#define RT_DFS_ELM_CODE_PAGE 437
#define RT_DFS_ELM_WORD_ACCESS
#define RT_DFS_ELM_USE_LFN_3
#define RT_DFS_ELM_USE_LFN 3
#define RT_DFS_ELM_MAX_LFN 255
#define RT_DFS_ELM_DRIVES 2
#define RT_DFS_ELM_MAX_SECTOR_SIZE 512
#define RT_DFS_ELM_REENTRANT
#define RT_USING_DFS_DEVFS             //开启 devfs，这里是将对各种设备(外设)的操作抽象为对文件的操作
#define RT_USING_DFS_NET               //BSD socket也可以被DFS管理，可以使用DFS的API对socket进行 read/write 或者 select/poll 操作
```