firmadyne论文

the first automated dynamic analysis system that specifically targets Linuxbased firmware on network-connected COTS devices in a scalable manner.

为什么要仿真路由器系统：

1、应用层面：最直接的方法是把网页文件拷贝出来，用一个常用的webserver启动，比如apache，但是这种方法与目标不符（建立一个通用的动态分析平台）
另外有些网页文件依赖于服务器端脚本语言的非标准性扩展，来获得特定硬件的功能，比如nvram的数值
比如，数以百计的固件利用自定义的get_conf的php函数和nvram_get的asp.net函数来获得硬件配置信息
其他的固件将网页嵌入到自定义的webserver二进制程序中
专注于应用数据的方法 只能检测特定应用的数据漏洞，比如php的命令注入

2、进程层面：仿真单个进程的行为，利用qemu的user mode，结合chroot，改变根目录到路由器的目录下

然而特定的硬件外设，例如nvram仍然获取不到，当程序试图获取nvram的外设，通过/dev/nvram，就很可能会因错误而终止

另外，在执行的环境的细微差别会对程序的行为有较大影响，比如alphafs webserver会校验产品和厂商的id，在获取nvram之前。如果这些数值，在预定义的物理内存地址没有获取到，则webserver会终止，为此，werserver利用mmap的系统调用函数，通过/dev/mem来获取内存，检查特定偏移来获取productid和vendorid，利用非易失性的芯片

仿真httpd行为，利用user-mode的模式会很复杂，模拟器需要跟踪映射内存的文件句柄和系统调用来确定程序行为，仿真器将需要识别各种存储器地址的语义定义，并且适当地替换这些值

此外，由于主存储设备上的写入周期有限，因此许多固件映像会在启动时为临时数据装载临时内存支持的文件系统，这个文件系统是动态生成和挂载的，结果是，/dev/和/etc/目录可能会被符号连接到临时文件系统的子目录下，所以在静态检查时会出现被破坏的情况，比如dir-865路由器利用了一个启动脚本，去生成程序的配置文件，包括lighttpd，配置文件被程序的-c参数传入程序中，所以单一的动态仿真lighttpd程序就会失败，即使在原路由器的位置

环境差异会对漏洞出现有较大影响，例如，对于信息泄露漏洞，一个适当的控制策略就可以修复，另外，系统配置也会影响目录遍历攻击

这个方法比上一个方法要精确很多，但是受到低的仿真保真度影响，没有准确的路由器系统运行时的相关信息，主机环境会不经意间影响对独立进程的动态分析（改变了程序执行）

3、系统层面：相比之下，系统级别的仿真方法足以克服之前提到的挑战，会提供到硬件外设的接口，精确的系统环境的仿真允许动态生成的数据被创建，就如同在实体机上一样，所有被系统启动的进程都可以被分析，包括负责协议的各种守护进程，如http，ftp和telnet，通过利用内核提供的内置硬件抽象，使用修改过的内核来代替固件中的内核文件，固件中的内核文件是针对于特定硬件平台来编译的，使用系统启动的序列，由init和rcS的二进制程序提供，能够初始化用户空间，与实体机保持一致。

不适用于内核模块，缺乏对位于文件系统上的的一些内核模块的支持，该内核模块不在预准备的内核文件中，所以内核版本的差异可能导致系统不稳定，不过绝大部分的那些缺少的内核模块对于系统都是无用的，因为我们构建的内核提供了功能等价的模块



# IV. IMPLEMENTATION

## A. Acquisition
## B. Extraction
## C. Emulation

#### NVRAM

nvram：52.6%占比

通过共享库libnvram.so来访问nvram，获取配置参数
此外设可以被抽象为键值库，通过截断nvram相关的函数请求get、set等，开发自定义的库libnvram.so，添加内核传递给init程序LD_PRELOAD的环境变量，保证所有的程序都继承了相同的环境变量

一个普通的挑战是不同固件是由不同的c 工具链编译的，一些获取不到，对于共享库有问题，因为必须制定动态加载的路径，但是这个取决于系统，采用惰性链接的方法

另一个挑战：nvram配置在没有系统特定的默认信息的情况下不好使，这些信息一般保存在硬件的nvram外设中，单纯返回null或者空字符串是不管用的，在检查一些仿真失败的固件之后，发现大多数固件将nvram的默认数值保存在一些常见的位置，比如 /etc/nvram.default /etc/nvram.conf /var/etc/nvram.default，有些会引入一个符号router_defaults  nvram  的字符串类型，在libnvram.so或libshared.so中，可以声明这些变量为弱引用

nvram配置不适用于所有固件：一些固件会调用没有仿真的nvram相关函数等

#### Kernel

2、内核：90.8%占比  armel  mipsel  mipseb

不使用原来的内核
用自己的定制内核

使用 kernel dynamic probes （kprobes）hook 20个系统调用  
截断改变执行环境的调用，包括设置mac地址，创建网桥，重启系统，执行程序

一些固件会需要特定的文件系统 在 boot 时 mount 
例如 /dev /proc

使用 rdinit 内核参数 在init执行之前运行自定义脚本来初始化这些文件系统
另外，加载 nandsim 内核模块，用于仿真mtd分区，通过/dev/mtdX访问
禁止用户重启，通过运行init来模拟此行为

针对mips架构，给大端和小端都编译了内核2.6.32.68
针对arm架构，只支持小端系统，目标瞄准arm versatile express development platform，使用的是Cortex-A9(ARMv7-A)处理器，该平台只支持最多一个仿真的以太网硬件，由于pci总线的缺失

增加架构支持是非自动化的
PCI / VirtIO 都要写相应架构的

3）内核配置

通过启动 学习正确的网卡配置

内核模块：58.8%的模块配置网络相关的功能，12.7%用于提供不同外设的支持（驱动）（无线，平台芯片，其他硬件），许多剩下的内核模块似乎包括在编译的内核中，被编译为可加载的，包括usb接口配置，文件系统加载，加密功能

不能仿真的情况：缺少init程序（缺少，或者名称不对），解压失败，只能解压类似于unix的文件系统
arm只能支持一个网卡，可能导致一些固件不能正确infer networking