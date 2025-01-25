# Drivers

Linux 驱动分为字符设备驱动、块设备驱动、网络设备驱动

- 字符设备驱动

  字符设备指必须以串行顺序依次访问的设备，如 led、触摸屏、鼠标等

  通过 open、close、read、write 等系统调用访问

- 块设备驱动

  块设备可以按任意顺序访问，以块为单位进行操作，如硬盘、EMMC 等

  块设备和字符设备的驱动设计有很大的差异，但是也可以通过 open、close、read、write 等系统调用进行访问，不过一般都是使用文件系统来进行管理

- 网络设备驱动

  网络设备是面向数据包的接收和发送设计的，在文件系统中并没有对应的设备结点，通过 socket 接口进行访问

### Misc Device

Linux 内核中的杂项设备（Miscellaneous Devices）是一种通用的设备类型，用于表示那些不适合其他设备类型的设备。这些设备通常是不规则的，没有标准的通信协议或接口。杂项设备提供了一种灵活的机制，允许我们将不同类型的设备注册为杂项设备，并通过统一的接口在用户空间访问它们。所有的misc类设备都是字符设备，也就是misc类设备(主设备号为10）其实是字符设备中分出来的一个小类。

misc类设备在应用层的操作接口：/dev/xxxx，设备类对应在 /sys/class/misc。misc类设备有自己的一套驱动框架，所以写一个misc设备的驱动直接利用的是内核中提供的驱动框架来实现的（linux/drivers/char/misc.c）。misc驱动框架是对内核提供的原始的字符设备。在内核中，misc驱动框架的源码实现在： driver/char/misc.c  相应的头文件在：include/linux/miscdevice.h

### Device Tree

简单的说，如果要使用 Device Tree，首先用户要了解自己的硬件配置和系统运行参数，并把这些信息组织成 Device Tree source file。通过 DTC（Device Tree Compiler），可以将这些适合人类阅读的 Device Tree source file 变成适合机器处理的 Device Tree binary file（有一个更好听的名字，DTB，device tree blob）。在系统启动的时候，boot program（例如：firmware、bootloader）可以将保存在 flash 中的 DTB copy 到内存（当然也可以通过其他方式，例如可以通过 bootloader 的交互式命令加载 DTB，或者 firmware 可以探测到 device 的信息，组织成 DTB 保存在内存中），并把 DTB 的起始地址传递给 client program（例如 OS kernel，bootloader 或者其他特殊功能的程序）。对于计算机系统（computer system），一般是 firmware->bootloader->OS，对于嵌入式系统，一般是 bootloader->OS。

##### Device Tree的结构

在描述 Device Tree 的结构之前，先考虑一个基础问题：是否 Device Tree 要描述系统中的所有硬件信息？答案是否定的。基本上，那些可以动态探测到的设备是不需要描述的，例如 USB device。不过对于 SOC 上的 usb host controller，它是无法动态识别的，需要在 Device Tree 中描述。同样的道理，在 computer system中，PCI device 可以被动态探测到，不需要在 Device Tree 中描述，但是 PCI bridge 如果不能被探测，那么就需要描述之。

```c
/ o device-tree
      |- name = "device-tree"
      |- model = "MyBoardName"
      |- compatible = "MyBoardFamilyName"
      |- #address-cells = <2>
      |- #size-cells = <2>
      |- linux,phandle = <0>
      |
      o cpus
      | | - name = "cpus"
      | | - linux,phandle = <1>
      | | - #address-cells = <1>
      | | - #size-cells = <0>
      | |
      | o PowerPC,970@0
      |   |- name = "PowerPC,970"
      |   |- device_type = "cpu"
      |   |- reg = <0>
      |   |- clock-frequency = <0x5f5e1000>
      |   |- 64-bit
      |   |- linux,phandle = <2>
      |
      o memory@0
      | |- name = "memory"
      | |- device_type = "memory"
      | |- reg = <0x00000000 0x00000000 0x00000000 0x20000000>
      | |- linux,phandle = <3>
      |
      o chosen
        |- name = "chosen"
        |- bootargs = "root=/dev/sda2"
        |- linux,phandle = <4>
```

Device Tree 的基本单元是 node。这些 node 被组织成树状结构，除了 root node，每个 node 都只有一个 parent。一个 device tree 文件中只能有一个 root node。每个 node 中包含了若干的 property/value 来描述该node的一些特性。每个 node 用节点名字（node name）标识，节点名字的格式是 node-name@unit-address。如果该 node 没有 reg 属性，那么该节点名字中不能包括 @ 和 unit-address。unit-address 的具体格式是和设备挂在哪个 bus 上相关。例如对于 CPU，其 unit-address 就是从0开始编址，以此加一。而具体的设备，例如以太网控制器，其 unit-address 就是寄存器地址。root node 的 node name 是确定的，必须是“/”。

在一个树状结构的 device tree 中，如何引用一个 node 呢？要想唯一指定一个 node 必须使用 full path，例如 /node-name-1/node-name-2/node-name-N。在上面的例子中，cpu node 我们可以通过 /cpus/PowerPC,970@0 访问。

属性（property）值标识了设备的特性，它的值（value）是多种多样的：

1. 可能是空，也就是没有值的定义。例如上图中的64-bit ，这个属性没有赋值。
2. 可能是一个u32、u64的数值（值得一提的是cell这个术语，在Device Tree表示32bit的信息单位）。例如#address-cells = <1> 。当然，可能是一个数组。例如<0x00000000 0x00000000 0x00000000 0x20000000>
3. 可能是一个字符串。例如 device_type = "memory" ，当然也可能是一个 string list。例如 "PowerPC,970"



#### Device Tree source file语法介绍

在 linux kernel 中，扩展名是 dts 的文件就是描述硬件信息的 device tree source file，在 dts 文件中，一个 node 被定义成：

```c
[label:] node-name[@unit-address] {
   [properties definitions]
   [child nodes]
}
```

“[]” 表示 option，因此可以定义一个只有 node name 的空节点。label 方便在 dts 文件中引用，具体后面会描述。child node 的格式和 node 是完全一样的，因此，一个 dts 文件中就是若干嵌套组成的 node，property 以及 child note、child note property 描述。

用实例来讲解 Device Tree Source file 的数据格式。假设要制作了一个 S3C2416 的开发板，把该 development board 命名为 snail，那么需要撰写一个 s3c2416-snail.dts 的文件。如果把所有的开发板的硬件信息（SOC 以及外设）都描述在一个文件中是不合理的，因此有可能其他公司也使用 S3C2416 搭建自己的开发板并命令 pig、cow 什么的，如果大家都用自己的 dts 文件描述硬件，那么其中大部分是重复的，因此可以把和 S3C2416 相关的硬件描述保存成一个单独的 dts 文件可以供使用 S3C2416 的 target board 来引用并将文件的扩展名变成 dtsi（i表示include）。同理，三星公司的 S3C24xx 系列是一个 SOC family，这些 SOCs（2410、2416、2450等）也有相同的内容，因此同样的道理，可以将公共部分抽取出来，变成 s3c24xx.dtsi，方便大家 include。同样的道理，各家 ARM vendor 也会共用一些硬件定义信息，这个文件就是 skeleton.dtsi。下面自下而上（类似C＋＋中的从基类到顶层的派生类）逐个进行分析。

##### skeleton.dtsi

位于 linux-3.14\arch\arm\boot\dts 目录下，具体该文件的内容如下：

```c
/ {
    #address-cells = <1>;
    #size-cells = <1>;
    chosen { };
    aliases { };
    memory { device_type = "memory"; reg = <0 0>; };
};
```

device tree 顾名思义是一个树状的结构，既然是树，必然有根。“/”是根节点的 node name。“{” 和 “}” 之间的内容是该节点的具体的定义，其内容包括各种属性的定义以及 child node 的定义。chosen、aliases 和 memory 都是 sub node，sub node 的结构和 root node 是完全一样的，因此，sub node 也有自己的属性和它自己的 sub node，最终形成了一个树状的 device tree。属性的定义采用 property ＝ value 的形式。例如 #address-cells 和 #size-cells 就是 property，而 <1> 就是 value。value 有三种情况：

1. 属性值是 text string 或者 string list，用双引号表示。例如 device_type = "memory"
2. 属性值是 32bit unsigned integers，用尖括号表示。例如 #size-cells = <1>
3. 属性值是 binary data，用方括号表示。例如`binary-property = [0x01 0x23 0x45 0x67]`

如果一个 device node 中包含了有寻址需求（要定义 reg property）的 sub node（后文也许会用 child node，和 sub node 是一样的意思），那么就必须要定义这两个属性。“#” 是 number 的意思，#address-cells 这个属性是用来描述 sub node 中的 reg 属性的地址域特性的，也就是说需要用多少个 u32 的 cell 来描述该地址域。同理可以推断 #size-cells 的含义，下面对 reg 的描述中会给出更详细的信息。

chosen node 主要用来描述由系统 firmware 指定的 runtime parameter。如果存在 chosen 这个 node，其 parent node 必须是名字是 “/” 的根节点。原来通过tag list 传递的一些 linux kernel 的运行时参数可以通过 Device Tree 传递。例如 command line 可以通过 bootargs 这个 property 这个属性传递；initrd 的开始地址也可以通过 linux,initrd-start 这个 property 这个属性传递。在本例中，chosen 节点是空的，在实际中，建议增加一个 bootargs 的属性，例如：

```c
"root=/dev/nfs nfsroot=1.1.1.1:/nfsboot ip=1.1.1.2:1.1.1.1:1.1.1.1:255.255.255.0::usbd0:off console=ttyS0,115200 mem=64M@0x30000000"
```

通过该 command line 可以控制内核从 usbnet 启动，当然，具体项目要相应修改 command line 以便适应不同的需求。device tree 用于 HW platform 识别，runtime parameter 传递以及硬件设备描述。chosen 节点并没有描述任何硬件设备节点的信息，它只是传递了 runtime parameter。

aliases 节点定义了一些别名。为何要定义这个 node 呢？因为 device tree 是树状结构，当要引用一个 node 的时候要指明相对于 root node 的 full path，例如 /node-name-1/node-name-2/node-name-N。如果多次引用，每次都要写这么复杂的字符串多少是有些麻烦，因此可以在 aliases 节点定义一些设备节点 full path 的缩写。

memory device node 是所有设备树文件的必备节点，它定义了系统物理内存的 layout。device_type 属性定义了该 node 的设备类型，例如 cpu、serial 等。对于 memory node，其 device_type 必须等于 memory。reg 属性定义了访问该 device node 的地址信息，该属性的值被解析成任意长度的（address，size）数组，具体用多长的数据来表示 address 和 size 是在其 parent node 中定义（#address-cells 和 #size-cells）。对于 device node，reg 描述了 memory-mapped IO register 的 offset 和 length。对于 memory node，定义了该 memory 的起始地址和长度。

本例中的物理内存的布局并没有通过 memory node 传递，其实可以使用 command line 传递，command line 中的参数 `mem=64M@0x30000000` 已经给出了具体的信息。再用另外一个例子来加深对本节描述的各个属性以及 memory node 的理解。假设系统是 64bit 的，physical memory 分成两段，定义如下：

RAM: starting address 0x0, length 0x80000000 (2GB)
RAM: starting address 0x100000000, length 0x100000000 (4GB)

对于这样的系统，可以将 root node 中的 #address-cells 和 #size-cells 这两个属性值设定为 2，可以用下面两种方法来描述物理内存：

```c
// 方法1：
memory@0 {
    device_type = "memory";
    reg = <0x000000000 0x00000000 0x00000000 0x80000000
           0x000000001 0x00000000 0x00000001 0x00000000>;
};

// 方法2：
memory@0 {
    device_type = "memory";
    reg = <0x000000000 0x00000000 0x00000000 0x80000000>;
};

memory@100000000 {
    device_type = "memory";
    reg = <0x000000001 0x00000000 0x00000001 0x00000000>;
};
```

##### s3c24xx.dtsi

位于 linux-3.14\arch\arm\boot\dts 目录下，具体该文件的内容如下：

```c
#include "skeleton.dtsi"

/ {
    compatible = "samsung,s3c24xx"; －－－－－－－－－－－－－－－－－－－(A)
    interrupt-parent = <&intc>; －－－－－－－－－－－－－－－－－－－－－－(B)

    aliases {
        pinctrl0 = &pinctrl_0; －－－－－－－－－－－－－－－－－－－－－－－－(C)
    };

    intc:interrupt-controller@4a000000 { －－－－－－－－－－－－－－－－－－(D)
        compatible = "samsung,s3c2410-irq";
        reg = <0x4a000000 0x100>;
        interrupt-controller;
        #interrupt-cells = <4>;
    };

    serial@50000000 { －－－－－－－－－－－－－－－－－－－－－－(E)
        compatible = "samsung,s3c2410-uart";
        reg = <0x50000000 0x4000>;
        interrupts = <1 0 4 28>, <1 1 4 28>;
        status = "disabled";
    };

    pinctrl_0: pinctrl@56000000 {－－－－－－－－－－－－－－－－－－(F)
        reg = <0x56000000 0x1000>;

        wakeup-interrupt-controller {
            compatible = "samsung,s3c2410-wakeup-eint";
            interrupts = <0 0 0 3>,
              	         <0 0 1 3>,
                  	     <0 0 2 3>,
                         <0 0 3 3>,
                         <0 0 4 4>,
                         <0 0 5 4>;
        };
    };

……
};
```

这个文件描述了三星公司的 S3C24xx 系列 SOC family 共同的硬件 block 信息。首先提出的问题就是：为何定义了两个根节点？按理说 Device Tree 只能有一个根节点，所有其他的节点都是派生于根节点的。猜测是这样的：Device Tree Compiler 会对 DTS 的 node进行合并，最终生成的 DTB 只有一个 root node。

（A）在描述 compatible 属性之前要先描述 model 属性。model 属性指明了该设备属于哪个设备生产商的哪一个 model。一般而言，我们会给 model 赋值“manufacturer,model”。例如 `model = "samsung,s3c24xx"`。`samsung` 是生产商，`s3c24xx` 是 model 类型，指明了具体的是哪一个系列的 SOC。compatible 属性的值是 string list，定义了一系列的 model（每个string是一个model）。这些字符串列表被操作系统用来选择用哪一个 driver 来驱动该设备。假设定义该属性：`compatible = “aaaaaa”, “bbbbb"`。那么操作操作系统可能首先使用 `aaaaaa` 来匹配适合的 driver，如果没有匹配到，那么使用字符串 `bbbbb` 来继续寻找适合的 driver，对于本例，`compatible = "samsung,s3c24xx"`，这里只定义了一个 model 而不是一个 list。对于 root node，compatible 属性是用来匹配 machine type 的。对于普通的 HW block 的节点，例如 interrupt-controller，compatible 属性是用来匹配适合的 driver 的。

（B）具体各个 HW block 的 interrupt source 是如何物理的连接到 interrupt controller 的呢？在 dts 文件中是用 interrupt-parent 这个属性来标识的。这里定义 interrupt-parent 属性的是 root node，难道 root node 会产生中断到 interrupt controller吗？当然不会，只不过如果一个能够产生中断的 device node 没有定义 interrupt-parent 的话，其 interrupt-parent 属性就是跟随 parent node。因此，与其在所有的下游设备中定义 interrupt-parent，不如统一在 root node 中定义了。

`intc` 是一个 label，标识了一个 device node（在本例中是标识了 `interrupt-controller@4a000000` 这个 device node）。实际上，interrupt-parent 属性值应该是是一个 u32 的整数值（这个整数值在 Device Tree 的范围内唯一识别了一个 device node，也就是 `phandle`），不过在 dts 文件中，可以使用类似 c 语言的 Labels and References 机制。定义一个 label，唯一标识一个 node 或者 property，后续可以使用&来引用这个label。DTC 会将 label 转换成 u32 的整数值放入到 DTB 中，用户层面就不再关心具体转换的整数值了。

关于 interrupt 值得进一步描述。在 Device Tree 中，有一个概念叫做 interrupt tree，也就是说 interrupt 也是一个树状结构。以下图为例

![drivers](./images/drivers1.gif)

系统中有一个 interrupt tree 的根节点，`device1`、`device2` 以及 PCI host bridge 的 interrupt line 都是连接到 root interrupt controller 的。PCI host bridge设备中有一些下游的设备，也会产生中断，但是他们的中断都是连接到 PCI host bridge 上的 interrupt controller（术语叫做 interrupt nexus），然后报告到root interrupt controller 的。每个能产生中断的设备都可以产生一个或者多个 interrupt，每个 interrupt source（另外一个术语叫做 interrupt specifier，描述了 interrupt source 的信息）都是限定在其所属的 interrupt domain 中。

在了解了上述的概念后，回头再看看 interrupt-parent 这个属性。其实这个属性是建立 interrupt tree 的关键属性。它指明了设备树中的各个 device node 如何路由 interrupt event。另外，需要提醒的是 interrupt controller 也是可以级联的，上图中没有表示出来。那么在这种情况下如何定义 interrupt tree 的 root 呢？那个没有定义 interrupt-parent 的 interrupt controller 就是 root。

（C）`pinctrl0` 是一个缩写，他是 `/pinctrl@56000000` 的别名。这里同样也是使用了 Labels and References 机制。

（D）`intc`（node name 是 `interrupt-controller@4a000000` ，这里直接使用 label）是描述 interrupt controller 的 device node。根据 S3C24xx 的 data sheet，interrupt controller 的寄存器地址从 `0x4a000000` 开始，长度为 `0x100`（实际 S3C2451 的 interrupt 的寄存器地址空间没有那么长，`0x4a000074` 是最后一个寄存器），也就是 reg 属性定义的内容。interrupt-controller 属性为空，只是用来标识该 node 是一个 interrupt controller 而不是 interrupt nexus（interrupt nexus 需要在不同的 interrupt domains 之间进行翻译，需要定义 interrupt-map 的属性）。#interrupt-cells 和 #address-cells 概念是类似的，也就是说，用多少个u32来标识一个 interrupt source。可以看到，在具体 HW block 的 interrupt 定义中都是用了4个u32来表示，例如串口的中断是这样定义的：

```c
interrupts = <1 0 4 28>, <1 1 4 28>;
```

（E） 从 reg 属性可以 serial controller 寄存器地址从 `0x50000000` 开始，长度为 `0x4000`。对于一个能产生中断的设备，必须定义 interrupts 这个属性。也可以定义 interrupt-parent 这个属性，如果不定义，则继承其 parent node 的 interrupt-parent 属性。 对于 interrupt 属性值，各个 interrupt controller 定义是不一样的，有的用3个u32表示，有的用4个。具体上面的各个数字的解释权归相关的 interrupt controller 所有。

（F）这个 node 是描述 GPIO 控制的。这个节点定义了一个 wakeup-interrupt-controller 的子节点，用来描述有唤醒功能的中断源。

##### s3c2416.dtsi

位于linux-3.14\arch\arm\boot\dts目录下，具体该文件的内容如下：

```c
#include "s3c24xx.dtsi"
#include "s3c2416-pinctrl.dtsi"

/ {
    model = "Samsung S3C2416 SoC"; 
    compatible = "samsung,s3c2416"; －－－－－－－－－－－－－－－(A)

    cpus { －－－－－－－－－－－－－－－－－－－－－－－－－－－－(B)
        #address-cells = <1>;
        #size-cells = <0>;

        cpu {
            compatible = "arm,arm926ejs";
        };
    };

    interrupt-controller@4a000000 { －－－－－－－－－－－－－－－－－(C)
        compatible = "samsung,s3c2416-irq";
    };

……

};
```

（A）在 s3c24xx.dtsi 文件中已经定义了 compatible 这个属性，在 s3c2416.dtsi 中重复定义了这个属性，一个 node 不可能有相同名字的属性，具体如何处理就交给 DTC 了。经过反编译，可以看出，DTC 是丢弃掉了前一个定义。因此，到目前为止，`compatible ＝ samsung,s3c2416`。在 s3c24xx.dtsi 文件中定义了compatible 的属性值被覆盖了。

（B）对于根节点，必须有一个 `cpus` 的 child node 来描述系统中的 CPU 信息。对于 CPU 的编址用一个u32整数就可以描述了，因此，对于 `cpus` node，#address-cells 是1，而 #size-cells 是0。其实 CPU 的 node 可以定义很多属性，例如 TLB，cache、频率信息什么的，不过对于ARM，这里只是定义了compatible 属性就OK了，`arm926ejs` 包括了所有的 processor 相关的信息。

（C）s3c24xx.dtsi 文件和 s3c2416.dtsi 中都有 `interrupt-controller@4a000000` 这个node，DTC 会对这两个 node进行合并，最终编译的结果如下：

```c
interrupt-controller@4a000000 {
        compatible = "samsung,s3c2416-irq";
        reg = <0x4a000000 0x100>;
        interrupt-controller;
        #interrupt-cells = <0x4>;
        linux,phandle = <0x1>;
        phandle = <0x1>;
    };
```

##### s3c2416-pinctrl.dtsi

 这个文件定义了 `pinctrl@56000000` 这个节点的若干 child node，主要用来描述 GPIO 的 bank 信息。

##### s3c2416-snail.dts

 这个文件应该定义一些 SOC 之外的 peripherals 的定义。



#### Device Tree binary格式

##### DTB整体结构

经过 Device Tree Compiler 编译，Device Tree source file 变成了 Device Tree Blob（又称作 flattened device tree）的格式。Device Tree Blob 的数据组织如下图所示：

![drivers](./images/drivers2.gif)

##### DTB header

| header field name | description                                                  |
| ----------------- | ------------------------------------------------------------ |
| magic             | 用来识别 DTB 的。通过这个 magic，kernel 可以确定 bootloader 传递的参数 block 是一个 DTB 还是 tag list。 |
| totalsize         | DTB 的 total size                                            |
| off_dt_struct     | device tree structure block 的 offset                        |
| off_dt_strings    | device tree strings block 的 offset                          |
| off_mem_rsvmap    | offset to memory reserve map。有些系统，也许会保留一些 memory 有特殊用途（例如 DTB 或者 initrd image），或者在有些 DSP + ARM 的 SOC platform 上，有写 memory 被保留用于 ARM 和 DSP 进行信息交互。这些保留内存不会进入内存管理系统。 |
| version           | 该 DTB 的版本。                                              |
| last_comp_version | 兼容版本信息                                                 |
| boot_cpuid_phys   | 在哪一个 CPU（用ID标识）上 booting                           |
| dt_strings_size   | device tree strings block 的 size。和 off_dt_strings 一起确定了 strings block 在内存中的位置 |
| dt_struct_size    | device tree structure block 的 size。和 off_dt_struct 一起确定了 device tree structure block 在内存中的位置 |

##### memory reserve map 的格式描述

这个区域包括了若干的 reserve memory 描述符。每个 reserve memory 描述符是由 address 和 size 组成。其中 address 和 size 都是用U64来描述。

##### device tree structure block 的格式描述

device tree structure block 区域是由若干的分片组成，每个分片开始位置都是保存了 token，以此来描述该分片的属性和内容。共计有5种 token：

（1）FDT_BEGIN_NODE (0x00000001)。该 token 描述了一个 node 的开始位置，紧挨着该 token 的就是 node name（包括 unit address）

（2）FDT_END_NODE (0x00000002)。该 token 描述了一个 node 的结束位置。

（3）FDT_PROP (0x00000003)。该 token 描述了一个 property 的开始位置，该 token 之后是两个u32的数据，分别是 length 和 name offset。length 表示该property value data 的 size。name offset 表示该属性字符串在 device tree strings block 的偏移值。length 和 name offset 之后就是长度为 length 具体的属性值数据。

（4）FDT_NOP (0x00000004)。

（5）FDT_END (0x00000009)。该 token 标识了一个 DTB 的结束位置。

一个可能的 DTB 的结构如下：

（1）若干个 FDT_NOP（可选）

（2）FDT_BEGIN_NODE

​          node name

​          paddings

（3）若干属性定义。

（4）若干子节点定义。（被 FDT_BEGIN_NODE 和 FDT_END_NODE 包围）

（5）若干个 FDT_NOP（可选）

（6）FDT_END_NODE

（7）FDT_END

##### device tree strings block 的格式描述

device tree strings bloc 定义了各个 node 中使用的属性的字符串表。由于很多属性会出现在多个 node 中，因此，所有的属性字符串组成了一个 string block。这样可以压缩 DTB 的 size。



#### Device Tree 代码分析

##### 运行时参数传递以及 platform 的识别

`linux/arch/arm/kernel/head.S` 文件定义了 bootloader 和 kernel 的参数传递要求：

```c
MMU = off, D-cache = off, I-cache = dont care, r0 = 0, r1 = machine nr, r2 = atags or dtb pointer.
```

目前的 kernel 支持旧的 tag list 的方式，同时也支持 device tree 的方式。`r2` 可能是 device tree binary file 的指针（bootloader 要传递给内核之前要 copy 到memory 中），也可以能是 tag list 的指针。在 ARM 的汇编部分的启动代码中（主要是 `head.S` 和 `head-common.S`），machine type ID 和指向 DTB 或者 `atags` 的指针被保存在变量 `__machine_arch_type` 和 `__atags_pointer` 中，这么做是为了后续 c 代码进行处理。

具体的 c 代码都是在 `setup_arch` 中处理，这个函数是一个总的入口点。具体代码如下（删除了部分无关代码）：

```c
void __init setup_arch(char **cmdline_p)
{
    const struct machine_desc *mdesc;

……

    mdesc = setup_machine_fdt(__atags_pointer);
    if (!mdesc)
        mdesc = setup_machine_tags(__atags_pointer, __machine_arch_type);
    machine_desc = mdesc;
    machine_name = mdesc->name;

……
}
```

对于如何确定 HW platform 这个问题，旧的方法是静态定义若干的 machine 描述符（`struct machine_desc` ），在启动过程中，通过 machine type ID 作为索引，在这些静态定义的 machine 描述符中扫描，找到那个 ID 匹配的描述符。在新的内核中，首先使用 `setup_machine_fdt` 来 setup machine 描述符，如果返回 NULL，才使用传统的方法 `setup_machine_tags` 来 setup machine 描述符。传统的方法需要给出 `__machine_arch_type`（bootloader 通过 `r1` 寄存器传递给 kernel 的）和 tag list 的地址（用来进行 tag parse）。`__machine_arch_type` 用来寻找 machine 描述符；tag list 用于运行时参数的传递。随着内核的不断发展，相信有一天 linux kernel 会完全抛弃 tag list 的机制。

`setup_machine_fdt` 函数的功能就是根据 Device Tree 的信息，找到最适合的 machine 描述符。具体代码如下：

```c
const struct machine_desc * __init setup_machine_fdt(unsigned int dt_phys)
{
    const struct machine_desc *mdesc, *mdesc_best = NULL;

    if (!dt_phys || !early_init_dt_scan(phys_to_virt(dt_phys)))
        return NULL;

    mdesc = of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);

    if (!mdesc) { 
        /* 出错处理 */
    }

    /* Change machine number to match the mdesc we're using */
    __machine_arch_type = mdesc->nr;

    return mdesc;
}
```

`early_init_dt_scan` 函数有两个功能，一个是为后续的 DTB scan 进行准备工作，另外一个是运行时参数传递。`of_flat_dt_match_machine` 是在 machine 描述符的列表中 scan，找到最合适的那个 machine 描述符。首先看如何组成 machine 描述符的列表。和传统的方法类似，也是静态定义的。`DT_MACHINE_START` 和 `MACHINE_END` 用来定义一个 machine 描述符。编译的时候，compiler 会把这些 machine descriptor 放到一个特殊的段中（`.arch.info.init`），形成 machine 描述符的列表。machine 描述符用下面的数据结构来标识：

```c
struct machine_desc {
    unsigned int        nr;            /* architecture number */
    const char *const   *dt_compat;    /* array of device tree 'compatible' strings */
……
   };
```

`nr` 成员就是过去使用的 machine type ID。内核 machine 描述符的 table 有若干个 entry，每个都有自己的 ID。bootloader 传递了 machine type ID，指明使用哪一个 machine 描述符。目前匹配 machine 描述符使用 compatible strings，也就是 `dt_compat` 成员，这是一个 string list，定义了这个 machine 所支持的列表。在扫描 machine 描述符列表的时候需要不断的获取下一个 machine 描述符的 compatible 字符串的信息，具体的代码如下：

```c
static const void * __init arch_get_next_mach(const char *const **match)
{
    static const struct machine_desc *mdesc = __arch_info_begin;
    const struct machine_desc *m = mdesc;

    if (m >= __arch_info_end)
        return NULL;

    mdesc++;
    *match = m->dt_compat;
    return m;
}
```

`__arch_info_begin` 指向 machine 描述符列表第一个 entry。通过 `mdesc++` 不断的移动 machine 描述符指针（Note：`mdesc` 是 static 的）。`match` 返回了该machine 描述符的 compatible string list。具体匹配的算法倒是很简单，就是比较字符串而已，一个是 root node 的 `compatible` 字符串列表，一个是 machine 描述符的 `compatible` 字符串列表，得分最低的（最匹配的）就是最终选定的 machine type。

运行时参数是在扫描 DTB 的 chosen node 时候完成的，具体的动作就是获取 chosen node 的 bootargs、initrd 等属性的 value，并将其保存在全局变量（boot_command_line，initrd_start、initrd_end）中。使用 tag list 方法是类似的，通过分析 tag list，获取相关信息，保存在同样的全局变量中。具体代码位于`early_init_dt_scan` 函数中：

```c
bool __init early_init_dt_scan(void *params)
{
    if (!params)
        return false;

    /* 全局变量initial_boot_params指向了DTB的header*/
    initial_boot_params = params;

    /* 检查DTB的magic，确认是一个有效的DTB */
    if (be32_to_cpu(initial_boot_params->magic) != OF_DT_HEADER) {
        initial_boot_params = NULL;
        return false;
    }

    /* 扫描 /chosen node，保存运行时参数（bootargs）到boot_command_line，此外，还处理initrd相关的property，并保存在initrd_start和initrd_end这两个全局变量中 */
    of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);

    /* 扫描根节点，获取 {size,address}-cells信息，并保存在dt_root_size_cells和dt_root_addr_cells全局变量中 */
    of_scan_flat_dt(early_init_dt_scan_root, NULL);

    /* 扫描DTB中的memory node，并把相关信息保存在meminfo中，全局变量meminfo保存了系统内存相关的信息。*/
    of_scan_flat_dt(early_init_dt_scan_memory, NULL);

    return true;
}
```

设定 `meminfo`（该全局变量确定了物理内存的布局）有若干种途径：

1. 通过 tag list（tag 是 `ATAG_MEM`）传递 memory bank 的信息。
2. 通过 command line（可以用 tag list，也可以通过 DTB）传递 memory bank 的信息。
3. 通过 DTB 的 memory node 传递 memory bank 的信息。

目前当然是推荐使用 Device Tree 的方式来传递物理内存布局信息。

##### 初始化流程

在系统初始化的过程中，将 DTB 转换成节点是 device_node 的树状结构，以便后续方便操作。具体的代码位于 `setup_arch->unflatten_device_tree` 中。

```c
void __init unflatten_device_tree(void)
{
    __unflatten_device_tree(initial_boot_params, &of_allnodes,
                early_init_dt_alloc_memory_arch);

    /* Get pointer to "/chosen" and "/aliases" nodes for use everywhere */
    of_alias_scan(early_init_dt_alloc_memory_arch);
}

struct device_node {
    const char *name; // device node name
    const char *type; // 对应device_type的属性
    phandle phandle; // 对应该节点的phandle属性
    const char *full_name; // 从“/”开始的，表示该node的full path

    struct    property *properties; // 该节点的属性列表
    struct    property *deadprops;  // 如果需要删除某些属性，kernel并非真的删除，而是挂入到deadprops的列表
    struct    device_node *parent; // parent、child以及sibling将所有的device node连接起来
    struct    device_node *child;
    struct    device_node *sibling;
    struct    device_node *next; // 通过该指针可以获取相同类型的下一个node
    struct    device_node *allnext; // 通过该指针可以获取node global list下一个node
    struct    proc_dir_entry *pde; // 开放到userspace的proc接口信息
    struct    kref kref; // 该node的reference count
    unsigned long _flags;
    void    *data;
};
```

`unflatten_device_tree` 函数的主要功能就是扫描 DTB，将 device node 被组织成：

1、global list。全局变量 `struct device_node *of_allnodes` 就是指向设备树的 global list

2、tree。

这些功能主要是在 `__unflatten_device_tree` 函数中实现，具体代码如下：

```c
static void __unflatten_device_tree(struct boot_param_header *blob, // 需要扫描的DTB
                 					struct device_node **mynodes, // global list指针
                 					void * (*dt_alloc)(u64 size, u64 align)) // 内存分配函数
{
    unsigned long size;
    void *start, *mem;
    struct device_node **allnextp = mynodes;

    /* 此处删除了health check代码，例如检查DTB header的magic，确认blob的确指向一个DTB。 */

    /* scan过程分成两轮，第一轮主要是确定device-tree structure的长度，保存在size变量中 */
    start = ((void *)blob) + be32_to_cpu(blob->off_dt_struct);
    size = (unsigned long)unflatten_dt_node(blob, 0, &start, NULL, NULL, 0);
    size = ALIGN(size, 4);

    /* 初始化的时候，并不是扫描到一个node或者property就分配相应的内存，实际上内核是一次性的分配了一大片内存，这些内存包括了所有的struct device_node、node name、struct property所需要的内存。*/
    mem = dt_alloc(size + 4, __alignof__(struct device_node));
    memset(mem, 0, size);

    *(__be32 *)(mem + size) = cpu_to_be32(0xdeadbeef);   //用来检验后面unflattening是否溢出

    /* 这是第二轮的scan，第一次scan是为了得到保存所有node和property所需要的内存size，第二次就是实打实的要构建device node tree了 */
    start = ((void *)blob) + be32_to_cpu(blob->off_dt_struct);
    unflatten_dt_node(blob, mem, &start, NULL, &allnextp, 0); 
   

    /* 此处略去校验溢出和校验OF_DT_END。 */
}
```

##### 并入 linux kernel 的设备驱动模型

在 linux kernel 引入统一设备模型之后，bus、driver 和 device 形成了设备模型中的铁三角。在驱动初始化的时候会将代表该 driver 的一个数据结构（一般是`xxx_driver`）挂入 bus 上的 driver 链表。device 挂入链表分成两种情况，一种是即插即用类型的 bus，在插入一个设备后，总线可以检测到这个行为并动态分配一个 device 数据结构（一般是 `xxx_device`，例如 `usb_device`），之后，将该数据结构挂入 bus 上的 device 链表。bus 上挂满了 driver 和 device，那么如何让 device 遇到“对”的那个 driver 呢？那么就要靠 bus 的 match 函数。

回到 Device Tree。导致 Device Tree 的引入 ARM 体系结构的代码其中一个最重要的原因的太多的静态定义的表格。例如：一般代码中会定义一个 `static struct platform_device *xxx_devices` 的静态数组，在初始化的时候调用 `platform_add_devices`。这些静态定义的 `platform_device` 往往又需要静态定义各种 resource，这导致静态表格进一步增大。如果 ARM linux 中不再定义这些表格，那么一定需要一个转换的过程，也就是说，系统应该会根据 device tree 来动态的增加系统中的 `platform_device`。当然，这个过程并非只是发生在 platform bus 上，也可能发生在其他的非即插即用的 bus 上，例如 AMBA 总线、PCI 总线。一言以蔽之，如果要并入 linux kernel 的设备驱动模型，那么就需要根据 device_node 的树状结构（root 是 `of_allnodes`）将一个个的 device node 挂入到相应的总线 device 链表中。只要做到这一点，总线机制就会安排 device 和 driver 的约会。当然，也不是所有的 device node 都会挂入 bus 上的设备链表，比如 cpus node，memory node，chosen node 等。

```c
/* cpus node 的处理 */
void __init arm_dt_init_cpu_maps(void)
{
    /* scan device node global list，寻找 full path 是 “/cpus” 的那个 device node。cpus 这个 device node 只是一个容器，
     * 其中包括了各个 cpu node 的定义以及所有 cpu node 共享的 property。
     */
    cpus = of_find_node_by_path("/cpus");

    for_each_child_of_node(cpus, cpu) {           // 遍历 cpus 的所有的 child node
        u32 hwid;

        if (of_node_cmp(cpu->type, "cpu"))        // 只关心那些 device_type 是 cpu 的 node
            continue;

        if (of_property_read_u32(cpu, "reg", &hwid)) {    // 读取 reg 属性的值并赋值给 hwid
            return;
        }

        /* reg 的属性值的 8 MSBs 必须设置为0，这是 ARM CPU binding 定义的。 */
        if (hwid & ~MPIDR_HWID_BITMASK)  
            return;

        /* 不允许重复的 CPU id，那是一个灾难性的设定 */
        for (j = 0; j < cpuidx; j++)
            if (WARN(tmp_map[j] == hwid, "Duplicate /cpu reg "
                             "properties in the DT\n"))
                return;

	/* 数组 tmp_map 保存了系统中所有 CPU 的 MPIDR 值（CPU ID值），具体的 index 的编码规则是： tmp_map[0] 保存了 booting CPU 的 id 值，
	 * 其余的 CPU 的 ID 值保存在 1～NR_CPUS 的位置。
	 */
        if (hwid == mpidr) {
            i = 0;
            bootcpu_valid = true;
        } else {
            i = cpuidx++;
        }

        tmp_map[i] = hwid;
    }

	/* 根据 DTB 中的信息设定 cpu logical map 数组。*/
    for (i = 0; i < cpuidx; i++) {
        set_cpu_possible(i, true);
        cpu_logical_map(i) = tmp_map[i];
    }
}

/* memory 的处理 */
int __init early_init_dt_scan_memory(unsigned long node, const char *uname, int depth, void *data)
{
    char *type = of_get_flat_dt_prop(node, "device_type", NULL); 获取device_type属性值
    __be32 *reg, *endp;
    unsigned long l;

    /* 在初始化的时候，会对每一个 device node 都要调用该 callback 函数，因此，要过滤掉那些和 memory block 定义无关的 node。
     * 和 memory block 定义有的节点有两种，一种是 node name 是 memory@ 形态的，另外一种是 node 中定义了 device_type 属性并且其值是 memory。
     */
    if (type == NULL) {
        if (depth != 1 || strcmp(uname, "memory@0") != 0)
            return 0;
    } else if (strcmp(type, "memory") != 0)
        return 0;

    /* 获取 memory 的起始地址和 length 的信息。有两种属性和该信息有关，一个是 linux,usable-memory，不过最新的方式还是使用 reg 属性。*/
	reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
    if (reg == NULL)
        reg = of_get_flat_dt_prop(node, "reg", &l);
    if (reg == NULL)
        return 0;

    endp = reg + (l / sizeof(__be32));

	/* reg 属性的值是 address，size 数组，那么如何来取出一个个的 address/size 呢？由于 memory node 一定是 root node 的 child，
	 * 因此 dt_root_addr_cells（root node 的 #address-cells 属性值）和 dt_root_size_cells（root node 的 #size-cells 属性值）
	 * 之和就是 address，size 数组的 entry size。*/

    while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
        u64 base, size;

        base = dt_mem_next_cell(dt_root_addr_cells, ®);
        size = dt_mem_next_cell(dt_root_size_cells, ®);

        early_init_dt_add_memory_arch(base, size);  // 将具体的memory block信息加入到内核中。
    }

    return 0;
}

/* interrupt controller 的处理 */
/* 初始化是通过 start_kernel->init_IRQ->machine_desc->init_irq() 实现的。用 S3C2416 为例来描述 interrupt controller 的处理过程。
 * 下面是machine描述符的定义。
 */
DT_MACHINE_START(S3C2416_DT, "Samsung S3C2416 (Flattened Device Tree)")
    ……
    .init_irq    = irqchip_init,
……
    MACHINE_END

/* 在 driver/irqchip/irq-s3c24xx.c 文件中定义了两个 interrupt controller */
IRQCHIP_DECLARE(s3c2416_irq, "samsung,s3c2416-irq", s3c2416_init_intc_of);
IRQCHIP_DECLARE(s3c2410_irq, "samsung,s3c2410-irq", s3c2410_init_intc_of);

/* 系统中可以定义更多的 irqchip，不过具体用哪一个是根据 DTB 中的 interrupt controller node 中的 compatible 属性确定的。
 * 在 driver/irqchip/irqchip.c 文件中定义了 irqchip_init 函数 */
void __init irqchip_init(void)
{
    of_irq_init(__irqchip_begin);
}
	
/* __irqchip_begin 就是所有的 irqchip的一个列表，of_irq_init 函数是遍历 Device Tree，找到匹配的 irqchip。 */
void __init of_irq_init(const struct of_device_id *matches)
{
    struct device_node *np, *parent = NULL;
    struct intc_desc *desc, *temp_desc;
    struct list_head intc_desc_list, intc_parent_list;

    INIT_LIST_HEAD(&intc_desc_list);
    INIT_LIST_HEAD(&intc_parent_list);

    /* 遍历所有的 node，寻找定义了 interrupt-controller 属性的 node，如果定义了 interrupt-controller 属性则说明该 node 就是一个中断控制器。 */
    for_each_matching_node(np, matches) {
        if (!of_find_property(np, "interrupt-controller", NULL) ||
                !of_device_is_available(np))
            continue;
       

		/* 分配内存并挂入链表，当然还有根据 interrupt-parent 建立 controller 之间的父子关系。
		 * 对于 interrupt controller，它也可能是一个树状的结构。
		 */
        desc = kzalloc(sizeof(*desc), GFP_KERNEL);
        if (WARN_ON(!desc))
            goto err;

        desc->dev = np;
        desc->interrupt_parent = of_irq_find_parent(np);
        if (desc->interrupt_parent == np)
            desc->interrupt_parent = NULL;
        list_add_tail(&desc->list, &intc_desc_list);
    }

    /* 正因为 interrupt controller 被组织成树状的结构，因此初始化的顺序就需要控制，应该从根节点开始，
     * 依次递进到下一个 level 的 interrupt controller。intc_desc_list 链表中的节点会被一个个的处理，
     * 每处理完一个节点就会将该节点删除，当所有的节点被删除，整个处理过程也就是结束了。
     */
    while (!list_empty(&intc_desc_list)) {  
        list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
            const struct of_device_id *match;
            int ret;
            of_irq_init_cb_t irq_init_cb;

            /* 最开始的时候 parent 变量是 NULL，确保第一个被处理的是 root interrupt controller。在处理完 root node 之后，
             * parent 变量被设定为 root interrupt controller，因此，第二个循环中处理的是所有 parent 是 root interrupt controller 的
             * child interrupt controller。也就是 level 1（如果 root 是 level 0 的话）的节点。
             */
            if (desc->interrupt_parent != parent)
                continue;

            list_del(&desc->list); // 从链表中删除
            match = of_match_node(matches, desc->dev); // 匹配并初始化
            if (WARN(!match->data, // match->data 是初始化函数
                "of_irq_init: no init function for %s\n",
                match->compatible)) {
                kfree(desc);
                continue;
            }

            irq_init_cb = (of_irq_init_cb_t)match->data;
            ret = irq_init_cb(desc->dev, desc->interrupt_parent); // 执行初始化函数
            if (ret) {
                kfree(desc);
                continue;
            }

           	/* 处理完的节点放入 intc_parent_list 链表，后面会用到 */
            list_add_tail(&desc->list, &intc_parent_list);
        }

        /* 对于 level 0，只有一个 root interrupt controller，对于 level 1，可能有若干个 interrupt controller，
         * 因此要遍历这些 parent interrupt controller，以便处理下一个 level 的 child node。*/
        desc = list_first_entry_or_null(&intc_parent_list,
                        typeof(*desc), list);
        if (!desc) {
            pr_err("of_irq_init: children remain, but no parents\n");
            break;
        }
        list_del(&desc->list);
        parent = desc->dev;
        kfree(desc);
    }

    list_for_each_entry_safe(desc, temp_desc, &intc_parent_list, list) {
        list_del(&desc->list);
        kfree(desc);
    }
err:
    list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
        list_del(&desc->list);
        kfree(desc);
    }
}
```

只有该 node 中有 interrupt-controller 这个属性定义，那么 linux kernel 就会分配一个 interrupt controller 的描述符（`struct intc_desc`）并挂入队列。通过interrupt-parent 属性，可以确定各个 interrupt controller 的层次关系。在 scan 了所有的 Device Tree 中的 interrupt controller 的定义之后，系统开始匹配过程。一旦匹配到了 interrupt chip 列表中的项次后，就会调用相应的初始化函数。如果 CPU 是 S3C2416 的话，匹配到的是 `irqchip` 的初始化函数是`s3c2416_init_intc_of`。

已经通过 compatible 属性找到了适合的 interrupt controller，那么如何解析 reg 属性呢？对于 S3c2416 的 interrupt controller 而言，其 #interrupt-cells 的属性值是4，定义为。每个域的解释如下：

（1）`ctrl_num` 表示使用哪一种类型的 interrupt controller，其值的解释如下：

   \- 0 ... main controller
   \- 1 ... sub controller
   \- 2 ... second main controller

（2）`parent_irq`。对于 sub controller，`parent_irq` 标识了其在 main controller 的 bit position。

（3）`ctrl_irq` 标识了在 controller 中的 bit 位置。

（4）type 标识了该中断的 trigger type，例如：上升沿触发还是电平触发。

简单的介绍2416的中断控制器，其block diagram如下：

![drivers](./images/drivers3.gif)

53个 `Samsung2416` 的中断源被分成两种类型，一种是需要 sub 寄存器进行控制的，例如 DMA，系统中的8个 DMA 中断是通过两级识别的，先在 `SRCPND` 寄存器中得到是 DMA 中断的信息，具体是哪一个 channel 的 DMA 中断需要继续查询 `SUBSRC` 寄存器。那些不需要 sub 寄存器进行控制的，例如 timer，5个timer 的中断可以直接从 `SRCPND` 中得到。

中断 MASK 寄存器可以控制产生的中断是否要报告给 CPU，当一个中断被 mask 的时候，虽然 `SRCPND` 寄存器中，硬件会 set 该 bit，但是不会影响到 `INTPND` 寄存器，从而不会向 CPU 报告该中断。对于 `SUBMASK` 寄存器，如果该 bit 被 set，也就是该 sub 中断被 mask 了，那么即便产生了对应的 sub 中断，也不会修改 `SRCPND` 寄存器的内容，只是修改 `SUBSRCPND` 中寄存器的内容。

不过随着硬件的演化，更多的 HW block 加入到 SOC 中，这使得中断源不够用了，因此中断寄存器又被分成两个 group，一个是 group 1（开始地址是`0X4A000000`，也就是 main controller 了），另外一个是 group 2（开始地址是 `0X4A000040`，叫做 second main controller）。group 1中的 sub 寄存器的起始地址是 `0X4A000018`（也就是 sub controller）。

```c
static struct s3c24xx_irq_of_ctrl s3c2416_ctrl[] = {
    {
        .name = "intc", // main controller
        .offset = 0,
    }, {
        .name = "subintc", // sub controller
        .offset = 0x18,
        .parent = &s3c_intc[0],
    }, {
        .name = "intc2", // second main controller
        .offset = 0x40,
    }
};
```

对于 s3c2416 而言，`irqchip` 的初始化函数是 `s3c2416_init_intc_of`，`s3c2416_ctrl` 作为参数传递给了 `s3c_init_intc_of`，大部分的处理都是在`s3c_init_intc_of` 函数中完成的。

```c
/* machine 初始化 */
static int __init customize_machine(void)
{

    if (machine_desc->init_machine)
        machine_desc->init_machine();
    else
        of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL);

    return 0;
}
arch_initcall(customize_machine);
```

在这个函数中，一般会调用 machine 描述符中的 `init_machine callback` 函数来把各种 Device Tree 中定义的 platform device 设备节点加入到系统（即platform bus 的所有的子节点，对于 device tree 中其他的设备节点，需要在各自 bus controller 初始化的时候自行处理）。如果 machine 描述符中没有定义`init_machine` ，那么直接调用 `of_platform_populate` 把所有的 platform device 加入到 kernel 中。对于 s3c2416，其 machine 描述符中的 `init_machine callback` 函数就是 `s3c2416_dt_machine_init`。

```c
static void __init s3c2416_dt_machine_init(void)
{
    of_platform_populate(NULL, of_default_bus_match_table, s3c2416_auxdata_lookup, NULL);  // 传入NULL参数表示从root node开始scan
    s3c_pm_init(); // power management相关的初始化
}
```

由此可见，最终生成 platform device 的代码来自 `of_platform_populate` 函数。该函数的逻辑比较简单，遍历 device node global list 中所有的 node，并调用`of_platform_bus_create` 处理。

```c
static int of_platform_bus_create(struct device_node *bus, // 要创建的那个 device node
                  				  const struct of_device_id *matches, // 要匹配的 list
                				  const struct of_dev_auxdata *lookup, // 附属数据
        				          struct device *parent, bool strict) // parent 指向父节点。strict 是否要求完全匹配
{
    const struct of_dev_auxdata *auxdata;
    struct device_node *child;
    struct platform_device *dev;
    const char *bus_id = NULL;
    void *platform_data = NULL;
    int rc = 0;

	// 省略确保 device node 有 compatible 属性的代码。

    auxdata = of_dev_lookup(lookup, bus);  // 在传入的 lookup table 寻找和该 device node 匹配的附加数据
    if (auxdata) {
        bus_id = auxdata->name; // 如果找到，那么就用附加数据中的静态定义的内容
        platform_data = auxdata->platform_data;
    }

	/* ARM 公司提供了 CPU core，除此之外，它设计了 AMBA 的总线来连接 SOC 内的各个 block。
	 * 符合这个总线标准的 SOC 上的外设叫做 ARM Primecell Peripherals。如果一个 device node 的 compatible 属性值是 arm,primecell 的话，
	 * 可以调用 of_amba_device_create 来向 amba 总线上增加一个 amba device。
	 */
    if (of_device_is_compatible(bus, "arm,primecell")) {
        of_amba_device_create(bus, bus_id, platform_data, parent);
        return 0;
    }

    /* 如果不是 ARM Primecell Peripherals，那么就需要向 platform bus 上增加一个 platform device。 */
    dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
    if (!dev || !of_match_node(matches, bus))
        return 0;

    /* 一个 device node 可能是一个桥设备，因此要重复调用 of_platform_bus_create 来把所有的 device node 处理掉。 */
    for_each_child_of_node(bus, child) {
        pr_debug("   create child: %s\n", child->full_name);
        rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
        if (rc) {
            of_node_put(child);
            break;
        }
    }
    return rc;
}

static struct platform_device *of_platform_device_create_pdata(struct device_node *np,
                    										   const char *bus_id,
                    										   void *platform_data,
                    										   struct device *parent)
{
    struct platform_device *dev;

    if (!of_device_is_available(np)) // check status 属性，确保是 enable 或者 OK 的。
        return NULL;

    /* of_device_alloc 除了分配 struct platform_device 的内存，还分配了该 platform device 需要的 resource 的内存。
     * 当然，这就需要解析该 device node 的 interrupt 资源以及 memory address 资源。 */
    dev = of_device_alloc(np, bus_id, parent);
    if (!dev)
        return NULL;

	/* 设定 platform_device 中的其他成员 */
    dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);
    if (!dev->dev.dma_mask)
        dev->dev.dma_mask = &dev->dev.coherent_dma_mask;
    dev->dev.bus = &platform_bus_type;
    dev->dev.platform_data = platform_data;

    if (of_device_add(dev) != 0) { // 把这个 platform device 加入统一设备模型系统中
        platform_device_put(dev);
        return NULL;
    }

    return dev;
}
```

### SMMU

SMMU（System Memory Management Unit）的作用和 MMU 类似，MMU 作用是替 CPU 翻译页表将进程的虚拟地址转换成 CPU 可以识别的物理地址。同理，SMMU 的作用就是替设备将 DMA 请求的地址，翻译成设备真正能用的物理地址，但是当 SMMU bypass 的时候，设备也可以直接使用物理地址来进行 DMA 。

![smmu](D:/Develop/Operating_System/images/smmu.webp)

SMMU 地址翻译的数据结构都是放在内存中的，由SMMU的寄存器保存着这些表在内存中的基地址，即Stream Table Entry(STE)。 STE 既包含 stage1 的翻译结构也包含 stage2 的翻译结构，所谓 stage1 负责 VA 到 PA 的转换，stage2 负责 IPA(Intermediate Physical Address) 到 PA 的转换。在 ARM 体系结构中，如果没有虚拟机参与的话，无论是 CPU 还是 SMMU 地址翻译都是从 VA->PA/IOVA->PA，被称之为 stage1，也就是不涉及虚拟，只是一阶段翻译而已。

相比MMU仅为一个 CPU 服务，一个 SMMU 可以为很多个设备服务。故为了区分，SMMU 给其所管理的每个设备一个唯一的 StreamID，通过 StreamID 可以查到 STE。 对于设备比较少的情况下，STE 只需要是 1 维数组就可以了，如下图：

![smmu](D:/Develop/Operating_System/images/ste1.webp)

STE 采用线性表并不是真是由设备的数量来决定的，而是写在 SMMU 的 ID0 寄存器中的，也就是配置好了的。对于设备数量较多的情况下，可以采用两层 STE 的结构，如下图：

![smmu](D:/Develop/Operating_System/images/ste2.webp)

ARM SMMU v3 第一层的目录 desc 的目录结构，大小采用 8（STRTAB_SPLIT）位，也就是 SteamID 的高 8 位，SteamID  剩下的低位全部用来寻址第二层真正的 STE。

![smmu](D:/Develop/Operating_System/images/ste3.webp)

如上如所示，红框中就是 SMMU 中一个 STE 的全貌。一个 STE 同时管理了 stage1 和 stage2 的数据结构，其中 Config 是表示 STE 有关的配置项，VMID 是指虚拟机 ID，S1ContextPtr 指向一个 Context Descriptor 的目录结构。使用 SMMU 的设备上可以跑多个任务，这些任务可以使用不同的 page table，而 SMMU 采用了 CD 来管理每个 page table，并新增一个 SubstreamID(pasid)，用于查找 CD。CD 在 SMMU 中也是可以是线性的或者两级的，可以在 SMMU 寄存器中配置，由 SMMU 驱动来读去，进行按对应的位进行分级，和 STE 表一样的原理。

![smmu](D:/Develop/Operating_System/images/ste4.webp)

在虚拟机的 Guest 中启用 SMMU 的时候，是需要同时开启 stage1 和 stage2 的，当然了，SMMU 也是可以进行 bypass 的，如下图所示：

![smmu](D:/Develop/Operating_System/images/stage1_stage2.webp)

下图是一个外设请求 SMMU 的地址翻译的基本流程，当一个外设需要 DMA 的物理地址的时候，开始请求 SMMU 的地址翻译。这时候外设给 SMMU 3 个比较重要的信息，分别是：StreamID：协助 SMMU 找到管理外设的 STE，SubsreamID：当找到 STE 后，协助 SMMU 找到对应的 CD，通过这两个 ID SMMU 就可以找到对应的 io page table 了。SMMU  找到 page table 后结合外设提交过来的最后一个信息 iova，即可开始进行地址翻译； SMMU 也有 tlb 的缓存，SMMU 首先会根据当前 CD 中存放的 ASID 来查查 tlb 缓存中有没有对应 page table 的缓存，其实和 mmu 找页表的原理是一样的。

![smmu](D:/Develop/Operating_System/images/smmu_translation.webp)

#### SMMU 驱动代码分析

```c
static int arm_smmu_device_probe(struct platform_device *pdev)
{
	int irq, ret;
	struct resource *res;
	resource_size_t ioaddr;
	struct arm_smmu_device *smmu; /* 内核中每个 smmu 都有一个结构体 struct arm_smmu_device 来管理，实际上初始化的流程就是在填充着个结构。 */
	struct device *dev = &pdev->dev;
	bool bypass;

	smmu = devm_kzalloc(dev, sizeof(*smmu), GFP_KERNEL); /* 从 slub/slab 中分配一个对象空间 */
	if (!smmu) {
		dev_err(dev, "failed to allocate arm_smmu_device\n");
		return -ENOMEM;
	}
	smmu->dev = dev;

	if (dev->of_node) {
		ret = arm_smmu_device_dt_probe(pdev, smmu); /* 从 dts 中的 smmu 节点中读取一些 smmu 中断等属性 */
	} else {
		ret = arm_smmu_device_acpi_probe(pdev, smmu); /* 从 acpi 的 smmu 配置表中读取一些 smmu 中断等属性 */
		if (ret == -ENODEV)
			return ret;
	}

	/* Set bypass mode according to firmware probing result */
	bypass = !!ret;

	/* Base address */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0); /* 从 dts 或者 apci 表中读取 smmu 的寄存器的基地址 */
	if (resource_size(res) < arm_smmu_resource_size(smmu)) {
		dev_err(dev, "MMIO region too small (%pr)\n", res);
		return -EINVAL;
	}
	ioaddr = res->start;
    
	/*
	 * Don't map the IMPLEMENTATION DEFINED regions, since they may contain
	 * the PMCG registers which are reserved by the PMU driver.
	 */
    /* smmu 寄存器映射 */
	smmu->base = arm_smmu_ioremap(dev, ioaddr, ARM_SMMU_REG_SZ);
	if (IS_ERR(smmu->base))
		return PTR_ERR(smmu->base);

	if (arm_smmu_resource_size(smmu) > SZ_64K) {
		smmu->page1 = arm_smmu_ioremap(dev, ioaddr + SZ_64K,
					       ARM_SMMU_REG_SZ);
		if (IS_ERR(smmu->page1))
			return PTR_ERR(smmu->page1);
	} else {
		smmu->page1 = smmu->base;
	}

	/* Interrupt lines */
    /* 读取 smmu 的几个中断号，smmu 硬件给软件消息有队列 buffer，smmu 硬件通过中断的方式让 smmu 驱动从队列 buffer 中取消息。 */
	irq = platform_get_irq_byname_optional(pdev, "combined");
	if (irq > 0)
		smmu->combined_irq = irq;
	else {
    /* smmu 的一个队列叫 event 队列，这个队列是给挂在 smmu 上的 platform 设备用的，当 platform 设备使用 smmu 翻译 dma 的 iova 的时候，
     * 如果发生了异常 smmu 会首先将异常的消息填到 event 队列中，随后上报一个 eventq 的中断给 smmu 驱动，smmu 驱动接到这个中断后，
     * 开始执行中断处理程序，从 event 队列中将异常的消息读出来，显示异常。
     */
		irq = platform_get_irq_byname_optional(pdev, "eventq");
		if (irq > 0)
			smmu->evtq.q.irq = irq;
    /* priq 中断时给 pri 队列用的，这个队列是专门给挂在 smmu 上的 pcie 类型的设备用的，具体的流程其实是和 event 队列是一样的。 */
		irq = platform_get_irq_byname_optional(pdev, "priq");
		if (irq > 0)
			smmu->priq.q.irq = irq;
    /* 如果 smmu 在执行过程中，发生了不可恢复的严重错误，smmu 会报告一个 gerror 中断给 smmu 驱动 */
		irq = platform_get_irq_byname_optional(pdev, "gerror");
		if (irq > 0)
			smmu->gerr_irq = irq;
	}
    
    /* Probe the h/w */
    /* 读取提前写死在 smmu 硬件寄存器中的各种配置，将配置 bit 位读取出来放到 struct arm_smm_device 的数据结构中。
     * 信息包括 smmu 支持二级 ste 还是一级的 ste，二级的 cd 还有 一 级的 cd，smmu 支持的物理也大小，iova 和 pa 的地址位数等等
     */
	ret = arm_smmu_device_hw_probe(smmu);
	if (ret)
		return ret;

	/* Initialise in-memory data structures */
    /* 初始化两种数据结构，一个 strtab(stream table 的简写)，根据硬件寄存器读出来的信息，决定初始化一级或者二级 ste，
     * 对于二级 ste 仅仅只会初始化第一级的 ste 目录项，分配的大小是 STRTAB_SPLIT，这个宏目前在 smmu 驱动中是 8 位，也就是预先会分配 2^8 个目录项，
     * 每个目录项的大小是固定的。另外一个种是队列(cmdq，evtq，priq)的内存分配和初始化。
     */
	ret = arm_smmu_init_structures(smmu);
	if (ret)
		return ret;

	/* Record our private device structure */
	platform_set_drvdata(pdev, smmu);

	/* Reset the device */
    /* 将先前初始化好的队列和 stream table 的目录项的基地址设置 smmu 的控制寄存器中 */
	ret = arm_smmu_device_reset(smmu, bypass);
	if (ret)
		return ret;

	/* And we're up. Go go go! */
	ret = iommu_device_sysfs_add(&smmu->iommu, dev, NULL,
				     "smmu3.%pa", &ioaddr);
	if (ret)
		return ret;

	iommu_device_set_ops(&smmu->iommu, &arm_smmu_ops);
	iommu_device_set_fwnode(&smmu->iommu, dev->fwnode);

    /* 将 smmu 的基本数据结构注册到上层的 iommu 抽象框架里，简单说明下此框架：能够有能力完成设备 iova 到 pa 转换的有很多，
     * 例如有 intel iommu, amd 的 iommu ,arm 的 smmu 等等。这些不同的硬件架构不会都作为一个独立的子系统，
     * 所以，在 linux 内核中 抽象了一层 iommu 层，由 iommu 层给各个外部设备驱动提供结构，隐藏底层的不同的架构
     */
	ret = iommu_device_register(&smmu->iommu);
	if (ret) {
		dev_err(dev, "Failed to register iommu\n");
		return ret;
	}
    /* 根据 smmu 所挂载的是 pcie 外设，还是 platform 外设，将 smmu 绑定到不同的总线类型上 */
	return arm_smmu_set_bus_ops(&arm_smmu_ops);
}
```



### DMA

DMA 是 Direct Memory Access 的缩写，顾名思义，就是绕开 CPU 直接访问 memory 的意思。在计算机中，相比 CPU，memory 和外设的速度是非常慢的，因而在 memory 和 memory（或者 memory 和设备）之间搬运数据，非常浪费 CPU 的时间，造成 CPU 无法及时处理一些实时事件。DMA 控制器是被设计专门用来搬运数据的器件，协助CPU进行数据搬运，如下图所示：

![dma](D:/Develop/Operating_System/images/dma.gif)

思路很简单，因而大多数的 DMA controller 都有类似的设计原则。得益于类似的设计原则，Linux kernel才有机会使用一套 framework 去抽象 DMA engine 有关的功能。下文的两个概念在此解释：Provider（DMA controller 驱动）和 Consumer（其它驱动怎么使用DMA传输数据）。

##### DMA channels

一个 DMA controller 可以“同时”进行的 DMA 传输的个数是有限的，这称作 DMA channels。当然，这里的 channel，只是一个逻辑概念，因为：鉴于总线访问的冲突，以及内存一致性的考量，从物理的角度看，不大可能会同时进行两个（及以上）的 DMA 传输。因而 DMA channel 不太可能是物理上独立的通道；很多时候，DMA channels 是 DMA controller 为了方便，抽象出来的概念，让 consumer 以为独占了一个 channel，实际上所有 channel 的 DMA 传输请求都会在 DMA controller 中进行仲裁，进而串行传输；因此，软件也可以基于 controller 提供的 channel（称为“物理”channel），自行抽象更多的“逻辑” channel，软件会管理这些逻辑 channel 上的传输请求。实际上很多平台都这样做了，在DMA Engine framework中，不会区分这两种 channel（本质上没区别）。

##### DMA request lines

DMA 传输是由 CPU 发起的：CPU 会告诉 DMA 控制器，帮忙将xxx地方的数据搬到xxx地方。CPU 发完指令之后，就当甩手掌柜了。而 DMA 控制器，除了负责怎么搬之外，还要决定一件非常重要的事情（特别是有外部设备参与的数据传输）：**何时可以开始数据搬运**

因为，CPU 发起 DMA 传输的时候，并不知道当前是否具备传输条件，例如 source 设备是否有数据、dest 设备的 FIFO 是否空闲等等。那谁知道是否可以传输呢？设备！因此，需要 DMA 传输的设备和 DMA 控制器之间，会有几条物理的连接线（称作DMA request，DRQ），用于通知 DMA 控制器可以开始传输了。

这就是 DMA request lines 的由来，通常来说，每一个数据收发的节点（称作endpoint），和 DMA controller 之间，就有一条 DMA request line（memory 设备除外）。

最后总结：DMA channel 是 Provider（提供传输服务），DMA request line 是 Consumer（消费传输服务）。在一个系统中 DMA request line 的数量通常比DMA channel 的数量多，因为并不是每个 request line 在每一时刻都需要传输。

##### 传输参数

在最简单的 DMA 传输中，只需为 DMA controller 提供一个参数 **transfer size**，它就可以欢快的工作了：在每一个时钟周期，DMA controller 将 1byte 的数据从一个 buffer 搬到另一个 buffer，直到搬完 transfer size 个 bytes 即可停止。

不过这在现实世界中往往不能满足需求，因为有些设备可能需要在一个时钟周期中，传输指定 bit 的数据，例如：memory 之间传输数据的时候，希望能以总线的最大宽度为单位（32-bit、64-bit等），以提升数据传输的效率；而在音频设备中，需要每次写入精确的16-bit或者24-bit的数据；等等。因此，为了满足这些多样的需求，需要为 DMA controller 提供一个额外的参数 **transfer width**。

另外，当传输的源或者目的地是 memory 的时候，为了提高效率，DMA controller 不愿意每一次传输都访问 memory，而是在内部开一个 buffer，将数据缓存在自己 buffer 中：memory 是源的时候，一次从 memory 读出一批数据，保存在自己的buffer中，然后再一点点（以时钟为节拍），传输到目的地；memory 是目的地的时候，先将源的数据传输到自己的 buffer 中，当累计一定量的数据之后，再一次性的写入 memory。这种场景下，DMA 控制器内部可缓存的数据量的大小，称作 **burst size**。

##### scatter-gather

一般情况下，DMA 传输一般只能处理在物理上连续的 buffer。但在有些场景下，我们需要将一些非连续的 buffer 拷贝到一个连续 buffer 中（这样的操作称作scatter gather）。对于这种非连续的传输，大多时候都是通过软件，将传输分成多个连续的小块（chunk）。但为了提高传输效率（特别是在图像、视频等场景中），有些DMA controller从硬件上支持了这种操作，具体实现与硬件相关。

##### Slave-DMA API和Async TX API

DMA 传输可以分为4类：memory 到 memory、memory 到 device、device 到 memory 以及 device 到 device。Linux kernel 作为 CPU 的代理人，从它的视角看，外设都是 slave，因此称这些有 device 参与的传输（MEM2DEV、DEV2MEM、DEV2DEV）为 Slave-DMA 传输。而另一种 memory 到 memory 的传输，被称为 Async TX。（Slave-DMA 中的 slave，指的是参与 DMA 传输的设备。而对应的，master 就是指 DMA controller 自身。）

为什么强调这种差别呢？因为Linux为了方便基于 DMA 的 memcpy、memset 等操作，在 dma engine 之上，封装了一层更为简洁的 API（如下面图所示），这种 API 就是 Async TX API（以 async_ 开头，例如 async_memcpy、async_memset、async_xor 等）。

![dma](D:/Develop/Operating_System/images/dma2.gif)

最后，因为 memory 到 memory 的 DMA 传输有了比较简洁的 API，没必要直接使用 dma engine 提供的 API，最后就导致 dma engine 所提供的 API 就特指为 Slave-DMA API（把 mem2mem 剔除了）。

##### dma engine的使用步骤

对设备驱动的编写者来说，要基于dma engine提供的Slave-DMA API进行DMA传输的话，需要如下的操作步骤：

1. 申请一个DMA channel。
2. 根据设备（slave）的特性，配置 DMA channel 的参数。
3. 要进行 DMA 传输的时候，获取一个用于识别本次传输（transaction）的描述符（descriptor）。
4. 将本次传输（transaction）提交给dma engine并启动传输。
5. 等待传输（transaction）结束。
6. 然后，重复3~5即可。

##### 申请DMA channel

任何 consumer 在开始 DMA 传输之前，都要申请一个 DMA channel。DMA channel（在kernel中由 `struct dma_chan`  数据结构表示）由 provider（或者是DMA controller）提供，被 consumer 使用。consumer可以通过如下的API申请DMA channel：

```c
/* 该接口会返回绑定在指定设备（dev）上名称为 name 的 dma channel。
 * dma engine 的 provider 和 consumer 可以使用 device tree、ACPI 或者 struct dma_slave_map 类型的 match table 提供这种绑定关系
 */
struct dma_chan *dma_request_chan(struct device *dev, const char *name);

/* 申请得到的dma channel可以在不需要使用的时候通过下面的API释放掉 */
void dma_release_channel(struct dma_chan *chan);
```

##### 配置DMA channel的参数

driver 申请到一个为自己使用的 DMA channel 之后，需要根据自身的实际情况，以及 DMA controller 的能力，对该 channel 进行一些配置。可配置的内容由`struct dma_slave_config` 数据结构表示。driver 将它们填充到一个 `struct dma_slave_config` 变量中后，可以调用如下 API 将这些信息告诉给 DMA controller：

```c
/* include/linux/dmaengine.h */
int dmaengine_slave_config(struct dma_chan *chan, struct dma_slave_config *config);

/* 包含了完成一次DMA传输所需要的所有可能的参数 */
struct dma_slave_config {
	/* 指明传输的方向，包括（具体可参考enum dma_transfer_direction的定义和注释）：
     * DMA_MEM_TO_MEM，memory 到 memory 的传输；
     * DMA_MEM_TO_DEV，memory 到设备的传输；
     * DMA_DEV_TO_MEM，设备到 memory 的传输；
     * DMA_DEV_TO_DEV，设备到设备的传输。
     * controller不一定支持所有的 DMA 传输方向，具体要看 provider 的实现。
     * MEM to MEM 的传输，一般不会直接使用 dma engine 提供的API。
     */
	enum dma_transfer_direction direction;
    /* 传输方向是 dev2mem 或者 dev2dev 时，读取数据的位置（通常是固定的FIFO地址）。
     * 对 mem2dev 类型的 channel，不需配置该参数（每次传输的时候会指定）。
     */
	phys_addr_t src_addr;
    /* 传输方向是 mem2dev 或者 dev2dev 时，写入数据的位置（通常是固定的 FIFO 地址）。
     * 对 dev2mem 类型的 channel，不需配置该参数（每次传输的时候会指定）。
     */
	phys_addr_t dst_addr;
    /* src地址的宽度，包括1、2、3、4、8、16、32、64（bytes）等（具体可参考enum dma_slave_buswidth 的定义）。 */
    enum dma_slave_buswidth src_addr_width;
    /* dst地址的宽度，包括1、2、3、4、8、16、32、64（bytes）等（具体可参考enum dma_slave_buswidth 的定义）。 */
    enum dma_slave_buswidth dst_addr_width;
    /* src 最大可传输的 burst size，单位是 src_addr_width（注意，不是byte）。 */
    u32 src_maxburst;
    /* dst 最大可传输的 burst size，单位是 dst_addr_width（注意，不是byte）。 */
    u32 dst_maxburst;
    /* 当外设是 Flow Controller（流控制器）的时候，需要将该字段设置为 true。CPU 中有关 DMA 和外部设备之间连接方式的设计中，
     * 决定 DMA 传输是否结束的模块，称作 flow controller，DMA controller 或者外部设备，都可以作为 flow controller，
     * 具体要看外设和 DMA controller 的设计原理、信号连接方式等。
     */
    bool device_fc;
    /* 外部设备通过 slave_id 告诉 dma controller 自己是谁（一般和某个 request line 对应）。很多 dma controller 并不区分 slave，
     * 只要给它 src、dst、len 等信息，它就可以进行传输，因此 slave_id 可以忽略。而有些 controller，必须清晰地知道此次传输的对象是哪个外设，
     * 就必须要提供 slave_id了（至于怎么提供，可dma controller的硬件以及驱动有关，要具体场景具体对待）。
     */
    unsigned int slave_id;
};
```

##### 获取传输描述（tx descriptor）

DMA 传输属于异步传输，在启动传输之前，slave driver 需要将此次传输的一些信息（例如 src/dst 的 buffer、传输的方向等）提交给 dma engine（本质上是dma controller driver），dma engine 确认 okay 后，返回一个描述符（由 `struct dma_async_tx_descriptor` 抽象）。此后，slave driver 就可以以该描述符为单位，控制并跟踪此次传输。根据传输模式的不同，slave driver 可以使用下面三个 API 获取传输描述符：

```c
/* 用于在“scatter gather buffers”列表和总线设备之间进行DMA传输
 * chan，本次传输所使用的 dma channel。
 * sgl，要传输的“scatter gather buffers”数组的地址；
 * sg_len，“scatter gather buffers”数组的长度。
 * direction，数据传输的方向，具体可参考 enum dma_data_direction （include/linux/dma-direction.h）的定义。
 * flags，可用于向 dma controller driver 传递一些额外的信息，包括（具体可参考enum dma_ctrl_flags中以DMA_PREP_开头的定义）：
 * DMA_PREP_INTERRUPT，告诉 DMA controller driver，本次传输完成后，产生一个中断，并调用 client 提供的回调函数（可在该函数返回后，
 * 通过设置 struct dma_async_tx_descriptor 指针中的相关字段，提供回调函数）；
 * DMA_PREP_FENCE，告诉 DMA controller driver，后续的传输，依赖本次传输的结果（这样 controller driver 就会小心的组织多个 dma 传输之间的顺序）；
 * DMA_PREP_PQ_DISABLE_P、DMA_PREP_PQ_DISABLE_Q、DMA_PREP_CONTINUE，PQ有关的操作。
 */
struct dma_async_tx_descriptor *dmaengine_prep_slave_sg(struct dma_chan *chan, struct scatterlist *sgl,
        												unsigned int sg_len, enum dma_data_direction direction,
        												unsigned long flags);

/* 常用于音频等场景中，在进行一定长度的dma传输（buf_addr&buf_len）的过程中，每传输一定的 byte（period_len），就会调用一次传输完成的回调函数
 * chan，本次传输所使用的 dma channel。
 * buf_addr、buf_len，传输的 buffer 地址和长度。
 * period_len，每隔多久（单位为byte）调用一次回调函数。需要注意的是，buf_len 应该是 period_len 的整数倍。
 * direction，数据传输的方向。
 */
struct dma_async_tx_descriptor *dmaengine_prep_dma_cyclic(struct dma_chan *chan, dma_addr_t buf_addr, size_t buf_len,
        												  size_t period_len, enum dma_data_direction direction);


/* 可进行不连续的、交叉的DMA传输，通常用在图像处理、显示等场景中 */
struct dma_async_tx_descriptor *dmaengine_prep_interleaved_dma(struct dma_chan *chan, struct dma_interleaved_template *xt,
        													   unsigned long flags);

/* 传输描述符用于描述一次DMA传输（类似于一个文件句柄）。client driver 将自己的传输请求提交给 dma controller driver 后，
 * controller driver 会返回给 client driver 一个描述符。client driver 获取描述符后，可以以它为单位，
 * 进行后续的操作（启动传输、等待传输完成、等等）。也可以将自己的回调函数通过描述符提供给 controller driver。
 */
struct dma_async_tx_descriptor {
    /* 一个整型数，用于追踪本次传输。一般情况下，dma controller driver 会在内部维护一个递增的 number，
     * 每当client获取传输描述的时候，都会将该number赋予cookie，然后加一。
     */
	dma_cookie_t cookie;
    /* flags， DMA_CTRL_开头的标记，包括：
	 * DMA_CTRL_REUSE，表明这个描述符可以被重复使用，直到它被清除或者释放；
	 * DMA_CTRL_ACK，如果该 flag 为0，表明暂时不能被重复使用。
	 */
    enum dma_ctrl_flags flags; /* not a 'long' to pack with cookie */
    /* 该描述符的物理地址 */
    dma_addr_t phys;
    /* 对应的dma channel */
    struct dma_chan *chan;
    /* controller driver 提供的回调函数，用于把改描述符提交到待传输列表。通常由 dma engine 调用，client driver 不会直接和该接口打交道。 */
    dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *tx);
    /* 用于释放该描述符的回调函数，由 controller driver 提供，dma engine 调用，client driver 不会直接和该接口打交道。 */
    int (*desc_free)(struct dma_async_tx_descriptor *tx);
    /* 传输完成的回调函数，由 client driver 提供。 */
    dma_async_tx_callback callback;
    /* 传输完成的回调函数参数，由 client driver 提供。 */
    void *callback_param;
    struct dmaengine_unmap_data *unmap;
#ifdef CONFIG_ASYNC_TX_ENABLE_CHANNEL_SWITCH
    struct dma_async_tx_descriptor *next;
    struct dma_async_tx_descriptor *parent;
    spinlock_t lock;
#endif
};
```

##### 启动传输

client driver 可以通过 `dmaengine_submit` 接口将该描述符放到传输队列上，然后调用 `dma_async_issue_pending` 接口，启动传输。kernel dma engine 鼓励client driver 一次提交多个传输，然后由 kernel（或者dma controller driver）统一完成这些传输。

```c
/* 参数为传输描述符指针，返回一个唯一识别该描述符的 cookie，用于后续的跟踪、监控。 */
dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc);

/* 参数为dma channel,无返回值 */
void dma_async_issue_pending(struct dma_chan *chan);
```

##### 等待传输结束

传输请求被提交之后，client driver 可以通过回调函数获取传输完成的消息，当然，也可以通过 `dma_async_is_tx_complete` 等 API，测试传输是否完成。最后，如果等不及了，也可以使用 `dmaengine_pause`、`dmaengine_resume`、`dmaengine_terminate_xxx` 等 API，暂停、终止传输。

##### dma controller驱动的软件框架

设备驱动的本质是描述并抽象硬件，然后为 consumer 提供操作硬件的友好接口。dma controller 驱动也不例外，它要做的事情无外乎是：

1. 抽象并控制 DMA 控制器。
2. 管理 DMA channel（可以是物理channel，也可以是虚拟channel），并向 client driver 提供友好、易用的接口。
3. 以 DMA channel 为操作对象，响应 client driver（consumer）的传输请求，并控制 DMA controller，执行传输。

![dma](D:/Develop/Operating_System/images/dma3.gif)

为了统一提供给 consumer 的 API，并减少 DMA controller driver 的开发难度，dmaengine framework 提供了一套 controller driver 的开发框架：

1. 使用 `struct dma_device` 抽象 DMA controller，controller driver 只要填充该结构中必要的字段，就可以完成 dma controller 的驱动开发。
2. 使用 `struct dma_chan`（上图中的DCn）抽象物理的 DMA channel（上图中的CHn），物理 channel 和 controller 所能提供的通道数一一对应。
3. 基于物理的 DMA channel，使用 `struct virt_dma_chan` 抽象出虚拟的 dma channel（上图中的VCx）。多个虚拟 channel 可以共享一个物理 channel，并在这个物理 channel 上进行分时传输。
4. 基于这些数据结构，提供一些便于 controller driver 开发的API，供 driver 使用。

```c
/* 用于抽象 dma controller */
struct dma_device {
	...
    /* 一个链表头，用于保存该 controller 支持的所有 dma channel（struct dma_chan）。
     * 在初始化的时候，dma controller driver 首先要调用 INIT_LIST_HEAD 初始化它，然后调用 list_add_tail 将所有的 channel 添加到该链表头中。
     */
	struct list_head channels;
    /* cap_mask，一个bitmap，用于指示该dma controller所具备的能力（可以进行什么样的DMA传输），例如：
     * DMA_MEMCPY，可进行memory copy；
     * DMA_MEMSET，可进行memory set；
     * DMA_SG，可进行 scatter list 传输；
     * DMA_CYCLIC，可进行 cyclic 的传输；
     * DMA_INTERLEAVE，可进行交叉传输；
     * 另外，该bitmap的定义，需要和后面 device_prep_dma_xxx 形式的回调函数对应（bitmap中支持某个传输类型，就必须提供该类型对应的回调函数）。
	*/
	dma_cap_mask_t  cap_mask;
    ...
    /* 一个bitmap，表示该 controller 支持哪些宽度的 src 类型，包括1、2、3、4、8、16、32、64（bytes）等
     *（具体可参考enum dma_slave_buswidth 的定义）。
     */
	u32 src_addr_widths;
    /* 一个bitmap，表示该 controller 支持哪些宽度的 dst 类型，包括1、2、3、4、8、16、32、64（bytes）等
     *（具体可参考enum dma_slave_buswidth 的定义）。
     */
	u32 dst_addr_widths;
    /* 一个bitmap，表示该c ontroller 支持哪些传输方向，包括 DMA_MEM_TO_MEM、DMA_MEM_TO_DEV、DMA_DEV_TO_MEM、DMA_DEV_TO_DEV，
     * 具体可参考 enum dma_transfer_direction 的定义和注释。
     */
	u32 directions;
	...
    /* 支持的最大的burst传输的size。 */
	u32 max_burst;
	...
    /* 指示该 controller 的传输描述符是否可重复使用（client driver 可只获取一次传输描述符，然后进行多次传输）。 */
	bool descriptor_reuse;
	...
    /* client driver 申请/释放 dma channel 的时候，dmaengine 会调用 dma controller driver 相应的 alloc/free 回调函数，以准备相应的资源。
     * 具体要准备哪些资源，则需要 dma controller driver 根据硬件的实际情况，自行决定。 */
	int (*device_alloc_chan_resources)(struct dma_chan *chan);
	void (*device_free_chan_resources)(struct dma_chan *chan);
    /* client driver 通过 dmaengine_prep_xxx API 获取传输描述符的时候，
     * damengine 则会直接回调 dma controller driver 相应的device_prep_dma_xxx 接口。
     * 至于要在这些回调函数中做什么事情，dma controller driver自己决定。 */
	struct dma_async_tx_descriptor *(*device_prep_dma_memcpy)(
		struct dma_chan *chan, dma_addr_t dst, dma_addr_t src,
		size_t len, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_xor)(
		struct dma_chan *chan, dma_addr_t dst, dma_addr_t *src,
		unsigned int src_cnt, size_t len, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_xor_val)(
		struct dma_chan *chan, dma_addr_t *src,	unsigned int src_cnt,
		size_t len, enum sum_check_flags *result, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_pq)(
		struct dma_chan *chan, dma_addr_t *dst, dma_addr_t *src,
		unsigned int src_cnt, const unsigned char *scf,
		size_t len, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_pq_val)(
		struct dma_chan *chan, dma_addr_t *pq, dma_addr_t *src,
		unsigned int src_cnt, const unsigned char *scf, size_t len,
		enum sum_check_flags *pqres, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_memset)(
		struct dma_chan *chan, dma_addr_t dest, int value, size_t len,
		unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_memset_sg)(
		struct dma_chan *chan, struct scatterlist *sg,
		unsigned int nents, int value, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_interrupt)(
		struct dma_chan *chan, unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_slave_sg)(
		struct dma_chan *chan, struct scatterlist *sgl,
		unsigned int sg_len, enum dma_transfer_direction direction,
		unsigned long flags, void *context);
	struct dma_async_tx_descriptor *(*device_prep_dma_cyclic)(
		struct dma_chan *chan, dma_addr_t buf_addr, size_t buf_len,
		size_t period_len, enum dma_transfer_direction direction,
		unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_interleaved_dma)(
		struct dma_chan *chan, struct dma_interleaved_template *xt,
		unsigned long flags);
	struct dma_async_tx_descriptor *(*device_prep_dma_imm_data)(
		struct dma_chan *chan, dma_addr_t dst, u64 data,
		unsigned long flags);

	...
    /* client driver 调用 dmaengine_slave_config 配置 dma channel 的时候，
     * dmaengine 会调用该回调函数，交给 dma controller driver 处理。
     */
	int (*device_config)(struct dma_chan *chan,
			     struct dma_slave_config *config);
    /* client driver 调用 dmaengine_pause 的时候，dmaengine 会调用相应的回调函数。 */
	int (*device_pause)(struct dma_chan *chan);
    /* client driver 调用 dmaengine_resume 的时候，dmaengine 会调用相应的回调函数。 */
	int (*device_resume)(struct dma_chan *chan);
    /* client driver 调用 dmaengine_terminate_xxx 的时候，dmaengine 会调用相应的回调函数。 */
	int (*device_terminate_all)(struct dma_chan *chan);
 	...
    /* client driver调用 dma_async_issue_pending 启动传输的时候，会调用调用该回调函数。 */
	void (*device_issue_pending)(struct dma_chan *chan);
	...
};

/* 用于抽象 dma channel */
struct dma_chan {
    /* 指向该 channel 所在的 dma controller。 */
	struct dma_device *device;
    /* client driver 以该 channel 为操作对象获取传输描述符时，dma controller driver 返回给 client 的最后一个cookie。 */
    dma_cookie_t cookie;
    /* 在这个 channel 上最后一次完成的传输的 cookie。
     * dma controller driver 可以在传输完成时调用辅助函数 dma_cookie_complete 设置它的 value。
     */
    dma_cookie_t completed_cookie;
	...
	/* 用于将该 channel 添加到 dma_device 的 channel 列表中。 */
    struct list_head device_node;
    ...
};

/* 用于抽象一个虚拟的 dma channel，多个虚拟channel可以共用一个物理channel，
 * 并由软件调度多个传输请求，将多个虚拟channel的传输串行地在物理channel上完成。
 */
struct virt_dma_chan {
    /* 用于和 client driver 打交道（屏蔽物理 channel 和虚拟 channel 的差异）。 */
    struct dma_chan chan;
    /* 用于等待该虚拟 channel 上传输的完成（由于是虚拟channel，传输完成与否只能由软件判断）。 */
    struct tasklet_struct task;
    ....
    /* 四个链表头，用于保存不同状态的虚拟channel描述符（struct virt_dma_desc，仅仅对 struct dma_async_tx_descriptor做了一个简单的封装）。 */
    struct list_head desc_allocated;
    struct list_head desc_submitted;
    struct list_head desc_issued;
    struct list_head desc_completed;
	....
};

struct virt_dma_desc {
    struct dma_async_tx_descriptor tx;
    /* protected by vc.lock */
    struct list_head node;
};

/* dma controller driver 准备好 struct dma_device 变量后，可以调用 dma_async_device_register 将它（controller）注册到 kernel 中。
 * 该接口会对 device 指针进行一系列的检查，然后对其做进一步的初始化，最后会放在一个名称为 dma_device_list 的全局链表上，以便后面使用。
 */
int dma_async_device_register(struct dma_device *device);
/* dma_async_device_unregister，注销接口。 */
void dma_async_device_unregister(struct dma_device *device);

/* 初始化dma channel中的cookie、completed_cookie字段。 */
static inline void dma_cookie_init(struct dma_chan *chan);
/* 为指针的传输描述（tx）分配一个cookie。 */
static inline dma_cookie_t dma_cookie_assign(struct dma_async_tx_descriptor *tx);
/* 当某一个传输（tx）完成的时候，可以调用该接口，更新该传输所对应 channel 的 completed_cookie 字段。 */
static inline void dma_cookie_complete(struct dma_async_tx_descriptor *tx);
/* 获取指定 channel（chan）上指定cookie的传输状态。 */
static inline enum dma_status dma_cookie_status(struct dma_chan *chan, dma_cookie_t cookie, struct dma_tx_state *state);

/* client 可以同时提交多个具有依赖关系的 dma 传输。因此当某个传输结束的时候，dma controller driver 需要检查是否有依赖该传输的传输，
 * 如果有，则传输之。这个检查并传输的过程，可以借助该接口进行（dma controller driver只需调用即可，省很多事）。
 */
void dma_run_dependencies(struct dma_async_tx_descriptor *tx);

/* 用于将client device node 中有关 dma 的字段解析出来，并获取对应的 dma channel。 */
extern struct dma_chan *of_dma_simple_xlate(struct of_phandle_args *dma_spec, struct of_dma *ofdma);
extern struct dma_chan *of_dma_xlate_by_chan_id(struct of_phandle_args *dma_spec, struct of_dma *ofdma);
```

##### 编写一个dma controller driver的方法和步骤

1. 定义一个 `struct dma_device` 变量，并根据实际的硬件情况，填充其中的关键字段。
2. 根据 controller 支持的 channel 个数，为每个channel定义一个 `struct dma_chan` 变量，进行必要的初始化后，将每个 channel 都添加到 `struct dma_device` 变量的 channels 链表中。
3. 根据硬件特性，实现 `struct dma_device` 变量中必要的回调函数（`device_alloc_chan_resources`/`device_free_chan_resources`、`device_prep_dma_xxx`、`device_config`、`device_issue_pending` 等等）。
4. 调用 `dma_async_device_register` 将 `struct dma_device` 变量注册到 kernel 中。
5. 当 client driver 申请 dma channel 时（例如通过 device tree 中的 dma 节点获取），dmaengine 会调用 dma controller driver 的`device_alloc_chan_resources` 函数，controller driver 需要在这个接口中将该 channel 的资源准备好。
6. 当 client driver 配置某个 dma channel 时，dmaengine 会调用 dma controller driver 的 `device_config` 函数，controller driver 需要在这个函数中将 client 想配置的内容准备好，以便进行后续的传输。
7. client driver 开始一个传输之前，会把传输的信息通过 `dmaengine_prep_slave_xxx` 接口交给 controller driver，controller driver 需要在对应的`device_prep_dma_xxx` 回调中，将这些要传输的内容准备好，并返回给 client driver 一个传输描述符。
8. 然后，client driver 会调用 `dmaengine_submit` 将该传输提交给 controller driver，此时 dmaengine 会调用 controller driver 为每个传输描述符所提供的`tx_submit` 回调函数，controller driver 需要在这个函数中将描述符挂到该 channel 对应的传输队列中。
9. client driver 开始传输时，会调用 `dma_async_issue_pending`，controller driver 需要在对应的回调函数（`device_issue_pending`）中，依次将队列上所有的传输请求提交给硬件。
10. 等待dma传输完成。

##### PMIO

PMIO(Port-Mapped I/O)，CPU使用专门的I/O指令对设备进行访问，并把设备的地址称作端口号。在执行其中的一条指令时，CPU使用地址总线选择所请求的I/O端口，使用数据总线在CPU寄存器和端口之间传送数据。系统设计者的主要目的是提供对I/O编程的统一方法，但又不牺牲性能。为了达到这个目的，每个设备的I/O 端口都被组织成一组专用寄存器。CPU把要发给设备的命令写入控制寄存器（control register），并从状态寄存器（status register）中读出表示设备内部状态的值。CPU还可以通过读取输入寄存器（input register）的内容从设备取得数据，也可以通过向输出寄存器（output register）中写入字节而把数据输出到设备。

##### MMIO

MMIO(Memory-Mapped I/O)，即将 I/O 设备中的内部存储和寄存器都映射到统一的存储地址空间（Memory Address Space）中。内存映射I/O这种编址方式非常巧妙，它是通过不同的物理内存地址给设备编址。这种编址方式将一部分物理内存的访问"重定向"到I/O地址空间中，CPU 尝试访问这部分物理内存的时候，实际上最终是访问了相应的I/O设备，CPU却浑然不知。这样以后, CPU就可以通过普通的访存指令来访问设备。这也是内存映射I/O得天独厚的好处：物理内存的地址空间和 CPU 的位宽都会不断增长，内存映射 I/O 从来不需要担心 I/O 地址空间耗尽的问题。从原理上来说，内存映射I/O唯一的缺点就是，CPU无法通过正常渠道直接访问那些被映射到I/O地址空间的物理内存了。但随着计算机的发展，内存映射I/O的唯一缺点已经越来越不明显了：现代计算机都已经是64位计算机，物理地址线都有48根，这意味着物理地址空间有256TB这么大，从里面划出3MB的地址空间给I/O设备，根本就是不痛不痒。

##### DMA Mapping

对于一个硬件设备上的寄存器等设备资源，内核是按照物理地址来管理的。通过 `/proc/iomem` ，你可以看到这些和设备 IO 相关的物理地址。当然，驱动并不能直接使用这些物理地址，必须首先通过 `ioremap()` 接口将这些物理地址映射到内核虚拟地址空间上去。

I/O设备使用第三种地址：**总线地址**。如果设备在 MMIO 地址空间中有若干的寄存器，或者该设备足够的智能，它可以通过 DMA 执行读写系统内存的操作，这些情况下，设备使用的地址就是总线地址。在某些系统中，总线地址与 CPU 物理地址相同，但一般来说它们不是。IOMMU 和 host bridge 可以在物理地址和总线地址之间进行映射。

从设备的角度来看，DMA 控制器使用总线地址空间，不过可能仅限于总线空间的一个子集。例如：即便是一个系统支持64位地址内存和64 位地址的PCI bar，但是DMA可以不使用全部的64 bit地址，通过 IOMMU 的映射，PCI 设备上的 DMA 可以只使用32位 DMA 地址。

![dma](D:/Develop/Operating_System/images/dma4.gif)

在 PCI 设备枚举（初始化）过程中，内核了解了所有的 IO device 及其对应的 MMIO 地址空间（MMIO 是物理地址空间的子集），并且也了解了是 PCI 主桥设备将这些 PCI device 和系统连接在一起。PCI 设备会有 BAR（base address register），表示自己在 PCI 总线上的地址，CPU 并不能通过总线地址 A（位于BAR范围内）直接访问总线上的 PCI 设备，PCI host bridge 会在 MMIO（即物理地址）和总线地址之间进行 mapping。因此，对于 CPU，它实际上是可以通过 B 地址（位于 MMIO 地址空间）访问 PCI 设备（反正 PCI host bridge 会进行翻译）。地址 B 的信息保存在 `struct resource` 变量中，并可以通过 `/proc/iomem` 开放给用户空间。对于驱动程序，它往往是通过 `ioremap()` 把物理地址 B 映射成虚拟地址 C，这时候，驱动程序就可以通过 `ioread32(C)` 来访问 PCI 总线上的地址 A了。

如果 PCI 设备支持 DMA，那么在驱动中我们可以通过 `kmalloc` 或者其他类似接口分配一个 DMA buffer，并且返回了虚拟地址 X，MMU 将 X 地址映射成了物理地址 Y，从而定位了 DMA buffer 在系统内存中的位置。因此，驱动可以通过访问地址 X 来操作 DMA buffer，但是 PCI 设备并不能通过 X 地址来访问 DMA buffer，因为 MMU 对设备不可见，而且系统内存所在的系统总线和 PCI 总线属于不同的地址空间。

在一些简单的系统中，设备可以通过 DMA 直接访问物理地址 Y，但是在大多数的系统中，有一个 IOMMU 的硬件用来将 DMA 可访问的总线地址翻译成物理地址，也就是把上图中的地址 Z 翻译成 Y 。理解了这些底层硬件，你也就知道类似 `dma_map_single` 这样的 DMA API 是在做什么了。驱动在调用 `dma_map_single` 这样的接口函数的时候会传递一个虚拟地址 X，在这个函数中会设定 IOMMU 的页表，将地址 X 映射到 Z，并且将返回 Z 这个总线地址。驱动可以把 Z 这个总线地址设定到设备上的 DMA 相关的寄存器中。这样，当设备发起对地址 Z 开始的 DMA 操作的时候，IOMMU 可以进行地址映射，并将 DMA 操作定位到 Y 地址开始的 DMA buffer。

根据上面的描述可以得出这样的结论：Linux 可以使用动态 DMA 映射（dynamic DMA mapping）的方法，当然，这需要一些来自驱动的协助。所谓动态 DMA 映射是指只有在使用的时候，才建立 DMA buffer 虚拟地址到总线地址的映射，一旦 DMA 传输完毕，就将之前建立的映射关系销毁。

DMA API 适用于各种 CPU arch，各种总线类型，DMA mapping framework 已经屏蔽了底层硬件的细节。对于驱动工程师而言，应该使用通用的 DMA API（例如 `dma_map()` 接口函数），而不是和特定总线相关的 API（例如 `pci_map()` 接口函数）。

##### 可被DMA控制器访问到的系统内存

可访问的内存：

- 伙伴系统的接口（例如 `__get_free_page*()`）或者类似 `kmalloc()` or `kmem_cache_alloc()` 
- 在驱动中定义的全局变量。如果编译到内核，那么全局变量位于内核的数据段或者bss段。在内核初始化的时候，会建立 kernel image mapping，因此全局变量所占据的内存都是连续的，并且 VA 和 PA 是有固定偏移的线性关系，因此可以用于 DMA 操作。不过，在定义这些全局变量的 DMA buffer 的时候，要进行cacheline 的对齐，并且要处理 CPU 和 DMA controller 之间的操作同步，以避免 cache coherence 问题。
- 块设备I/O子系统和网络子系统在分配 buffer 的时候会确保可以使用 DMA 访问。

不可访问或不建议使用的内存：

- `vmalloc()` 分配的DMA buffer。`vmalloc` 分配的 page frame 是不连续的，如果底层硬件需要物理内存连续，那么 `vmalloc` 分配的内存不能满足硬件要求。即便是底层 DMA 硬件支持 scatter-gather，`vmalloc` 分配出来的内存虚拟地址和对应的物理地址没有线性关系（`kmalloc` 或者 `__get_free_page*` 这样的接口，其返回的虚拟地址和物理地址有一个固定偏移的关系），而在做 DMA mapping 的时候，需要知道物理地址，有线性关系的虚拟地址很容易可以获取其物理地址，但是对于 `vmalloc` 分配的虚拟地址，需要遍历页表才可以找到其物理地址。
- 驱动编译成模块。驱动中的全局定义的 DMA buffer 不在内核的线性映射区域，其虚拟地址是在模块加载的时候，通过vmalloc分配。因此这时候如果 DMA buffer 如果大于一个 page frame，那么也是无法保证其底层物理地址的连续性，也无法保证 VA 和 PA 的线性关系，这一点和编译到内核是不同的。
- `kmap()` 接口返回的内存。其原理类似vmalloc。

##### DMA寻址限制

不同的硬件平台有不同的配置方式，有的平台没有限制，外设可以访问系统内存的每一个 Byte，有些则不可以。例如：系统总线有32个bit，设备通过DMA只能驱动低24位地址，在这种情况下，外设在发起 DMA 操作的时候，只能访问16M以下的系统内存。如果设备有 DMA 寻址的限制，那么驱动需要将这个限制通知到内核。如果驱动不通知内核，那么内核缺省情况下认为外设的 DMA 可以访问所有的系统总线的32 bit地址线。64 bit平台同理。是否有DMA寻址限制是和硬件设计相关，有时候标准总线协议也会规定这一点。例如：PCI-X 规范规定，所有的 PCI-X 设备必须要支持64 bit的寻址。如果有寻址限制，那么在该外设驱动的 probe 函数中，驱动需要询问内核，看看是否有 DMA controller 可以支持这个外设的寻址限制。虽然有缺省的寻址限制的设定，不过最好还是在 probe 函数中进行相关处理。

```c
int dma_set_mask_and_coherent(struct device *dev, u64 mask);
```

DMA 操作有两种：一种是 streaming，DMA buffer 是一次性的，用完就算。这种 DMA buffer 需要自己考虑 cache 一致性。另外一种是 DMA buffer 是 cache coherent 的，软件实现上比较简单，更重要的是这种 DMA buffer 往往是静态的、长时间存在的。不同类型的 DMA 操作可能有有不同的寻址限制，也可能相同。如果相同，可以用上面这个接口设定 streaming 和 coherent 两种DMA 操作的地址掩码。如果不同，可以下面的接口进行设定：

```c
/* 设定streaming类型的DMA地址掩码 */
int dma_set_mask(struct device *dev, u64 mask);
/* 设定coherent类型的DMA地址掩码 */
int dma_set_coherent_mask(struct device *dev, u64 mask);
```

`dev` 指向该设备的 `struct device` 对象，一般来说，这个 `struct device` 对象应该是嵌入在 bus-specific 的实例中。例如对于PCI设备，有一个 `struct pci_dev` 的实例与之对应，而在这里需要传入的 `dev` 参数则可以通过 `&pdev->dev` 得到（`pdev` 指向 `struct pci_dev` 的实例）。`mask` 表示设备支持的地址线信息。如果调用这些接口返回0，则说明一切OK，从该设备到指定 mask 的内存的 DMA 操作是可以被系统支持的（包括 DMA controller、bus layer 等）。如果返回值非0，那么说明这样的 DMA 寻址是不能正确完成的，如果强行这么做将会产生不可预知的后果。驱动必须检测返回值，如果不行，那么建议修改 mask 或者不使用 DMA。也就是说，对上面接口调用失败后，有三个选择：

1. 用另外的mask
2. 不使用DMA模式，采用普通I/O模式
3. 忽略这个设备的存在，不对其进行初始化

```c
/* 一个可以寻址32 bit的设备，其初始化的示例代码如下 */
if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32))) {
    dev_warn(dev, "mydev: No suitable DMA available\n");
    goto ignore_this_device;
}
```

另一个常见的场景是有64位寻址能力的设备。一般来说会首先尝试设定64位的地址掩码，但是这时候有可能会失败，从而将掩码降低为32位。内核之所以会在设定64位掩码的时候失败，这并不是因为平台不能进行64位寻址，而仅仅是因为32位寻址比64位寻址效率更高。例如，SPARC64 平台上，PCI SAC寻址比DAC寻址性能更好。

```c
int using_dac;

if (!dma_set_mask(dev, DMA_BIT_MASK(64))) {
    using_dac = 1;
} else if (!dma_set_mask(dev, DMA_BIT_MASK(32))) {
    using_dac = 0;
} else {
    dev_warn(dev, "mydev: No suitable DMA available\n");
    goto ignore_this_device;
}
```

设定 coherent 类型的DMA地址掩码也是类似的。需要说明的是：coherent 地址掩码总是等于或者小于 streaming 地址掩码，因此，一般来说，只要设定了streaming 地址掩码成功了，那么使用同样的掩码或者小一些的掩码来设定 coherent 地址掩码总是会成功，因此这时候一般就不检查 `dma_set_coherent_mask `的返回值了，当然，有些设备很奇怪，只能使用 coherent DMA，那么这种情况下，驱动需要检查 `dma_set_coherent_mask` 的返回值。

##### DMA操作方向

```c
DMA_BIDIRECTIONAL
DMA_TO_DEVICE
DMA_FROM_DEVICE
DMA_NONE
```

DMA_TO_DEVICE 表示“从内存（dma buffer）到设备”，而 DMA_FROM_DEVICE 表示“从设备到内存（dma buffer）”，上面的这些字符定义了数据在 DMA 操作中的移动方向。然而，如果不知道具体的操作方向，那么设定为DMA_BIDIRECTIONAL也是可以的，表示DMA操作可以执行任何一个方向的的数据搬移。平台需要保证这一点可以让 DMA 正常工作，当然，这也有可能会引入一些性能上的额外开销。DMA_NONE 主要是用于调试。在驱动知道精确的 DMA 方向之前，可以把它保存在 DMA 控制数据结构中，在 DMA 方向设定有问题的时候，你可以跟踪 DMA 方向的设置情况，以便定位问题所在。

除了潜在的平台相关的性能优化之外，精确地指定 DMA 操作方向还有另外一个优点就是方便调试。有些平台实际上在创建 DMA mapping 的时候，页表（指将bus 地址映射到物理地址的页表）中有一个写权限布尔值，这个值非常类似于用户程序地址空间中的页保护。当 DMA 控制器硬件检测到违反权限设置时（这时候dma buffer 设定的是 DMA_TO_DEVICE 类型，实际上 DMA controller 只能是读 dma buffer），这样的平台可以将错误写入内核日志，从而方便了debug。

只有 streaming mappings 才会指明 DMA 操作方向，一致性 DMA 映射隐含的 DMA 操作方向是 DMA_BIDIRECTIONAL。举一个 streaming mappings 的例子：在网卡驱动中，如果要发送数据，那么在 map/umap 的时候需要指明 DMA_TO_DEVICE 的操作方向，而在接受数据包的时候，map/umap 需要指明 DMA 操作方向是 DMA_FROM_DEVICE。

##### 一致性DMA映射（Consistent DMA mappings ）

Consistent DMA mapping 有下面两种特点：

1. 持续使用该 DMA buffer（不是一次性的），因此 Consistent DMA 总是在初始化的时候进行 map，在 shutdown 的时候 unmap。
2. CPU 和 DMA controller 在发起对 DMA buffer 的并行访问的时候不需要考虑 cache 的影响，也就是说不需要软件进行 cache 操作，CPU 和 DMA controller都可以看到对方对 DMA buffer 的更新。实际上一致性 DMA 映射中的那个 Consistent 实际上可以称为 coherent，即 cache coherent。

缺省情况下，coherent mask 被设定为低 32 bit（0xFFFFFFFF），即便缺省值是OK了，也建议通过接口在驱动中设定 coherent mask。

一般使用 Consistent DMA mapping 的场景包括：

1. 网卡驱动和网卡 DMA 控制器往往是通过一些内存中的描述符（形成环或者链）进行交互，这些保存描述符的 memory 一般采用 Consistent DMA mapping。
2. SCSI(Small Computer System Interface) 硬件适配器上的 DMA 可以主存中的一些数据结构（mailbox command）进行交互，这些保存 mailbox command 的 memory 一般采用 Consistent DMA mapping。
3. 有些外设有能力执行主存上的固件代码（microcode），这些保存 microcode 的主存一般采用 Consistent DMA mapping。

上面的这些例子有同样的特性：CPU 对 memory 的修改可以立刻被 device 感知到，反之亦然。一致性映射可以保证这一点。

需要注意的是：一致性的 DMA 映射并不意味着不需要 memory barrier 这样的工具来保证 memory order，CPU 有可能为了性能而重排对 consistent memory 上内存访问指令。例如：如果在 DMA consistent memory 上有两个 word，分别是 word0 和 word1，对于 device 一侧，必须保证 word0 先更新，然后才有对 word1 的更新，那么你需要这样写代码才能保证在所有的平台上，给设备驱动可以正常的工作：

```c
desc->word0 = address;
wmb();
desc->word1 = DESC_VALID;
```

此外，在有些平台上，修改了 DMA Consistent buffer 后，驱动可能需要 flush write buffer，以便让 device 侧感知到 memory 的变化。这个动作类似在 PCI 桥中的 flush write buffer 的动作。

```c
/* 分配并映射dma buffer */
dma_addr_t dma_handle;
cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
```

DMA 操作总是会涉及具体设备上的 DMA controller，而 `dev` 参数就是执行该设备的 `struct device` 对象的。`size` 参数指明了你想要分配的 DMA Buffer 的大小，byte 为单位。`dma_alloc_coherent` 这个接口也可以在中断上下文调用，当然，`gfp` 参数要传递 `GFP_ATOMIC` 标记。`gfp` 是内存分配的 flag，`dma_alloc_coherent` 仅仅是透传该flag到内存管理模块。

需要注意的是 `dma_alloc_coherent` 分配的内存的起始地址和 size 都是对齐在 page 上（类似 `__get_free_pages` 的感觉，当然 `__get_free_pages` 接受的size 参数是 page order），如果不需要那么大的 DMA buffer，那么可以选择 `dma_pool` 接口。

如果传入非空的 `dev` 参数，即使驱动调用了掩码设置接口函数设定了 DMA mask，说明该设备可以访问大于32-bit地址空间的地址，一致性 DMA 映射的接口函数也一般会默认的返回一个32-bit可寻址的 DMA buffer 地址。要知道 dma mask 和 coherent dma mask 是不同的，除非驱动显示的调用`dma_set_coherent_mask()` 接口来修改 coherent dma mask，例如大小大于32-bit地址，`dma_alloc_coherent` 接口函数才会返回大于32-bit地址空间的地址。`dma pool` 接口也是如此。

`dma_alloc_coherent` 函数返回两个值，一个是从 CPU 角度访问 DMA buffer 的虚拟地址，另外一个是从设备（DMA controller）角度看到的 bus address：dma_handle，驱动可以将这个 bus address 传递给硬件。

即便是请求的 DMA buffer 的大小小于 PAGE_SIZE，`dma_alloc_coherent` 返回的 CPU 虚拟地址和 DMA 总线地址都保证对齐在最小的 PAGE_SIZE 上，这个特性确保了分配的 DMA buffer 有这样的特性：如果 PAGE_SIZE  是64K，即便是驱动分配一个小于或者等于64K的 dma buffer，那么 DMA buffer 不会越过64K的边界。

```c
/* umap并释放dma buffer */
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

`dev`、`size`与分配接口相同，而 `cpu_addr` 和 `dma_handle` 这两个参数就是 `dma_alloc_coherent()` 接口的那两个地址返回值。需要强调的一点就是：和`dma_alloc_coherent` 不同，`dma_free_coherent` 不能在中断上下文中调用。（因为在有些平台上，free DMA的操作会引发TLB维护的操作（从而引发cpu core之间的通信），如果关闭了IRQ会锁死在SMP IPI 的代码中）。

如果驱动需要非常多的小的 dma buffer，那么 `dma_pool` 是最适合你的机制。这个概念类似 `kmem_cache`，`__get_free_pages` 往往获取的是连续的 page frame，而 `kmem_cache` 是批发了一大批 page frame，然后自己“零售”。`dma_pool` 就是通过 `dma_alloc_coherent` 接口获取大块一致性的 DMA 内存，然后驱动可以调用 `dma_pool_alloc` 从那个大块 DMA 内存中分一个小块的 dma buffer 供自己使用。

##### 流式DMA映射（streaming DMA mapping） 

流式 DMA 映射是一次性的，一般是需要进行 DMA 传输的时候才进行 mapping，一旦 DMA 传输完成，就立刻 ummap（除非使用 `dma_sync_*` 的接口）。并且硬件可以为顺序化访问进行优化，设计 streaming DMA mapping 这样的接口是为了充分优化硬件的性能。这里的 streaming 可以被认为是 asynchronous，或者是不属于 coherent memory 范围的。

一般使用streaming DMA mapping的场景包括：

1. 网卡进行数据传输使用的 DMA buffer
2. 文件系统中的各种数据 buffer，这些 buffer 中的数据最终到读写到 SCSI 设备上去，一般而言，驱动会接受这些 buffer，然后进行 streaming DMA mapping，之后和 SCSI 设备上的 DMA 进行交互。

无论哪种类型的 DMA 映射都有对齐的限制，这些限制来自底层的总线，当然也有可能是某些总线上的设备有这样的限制。此外，如果系统中的 cache 并不是 DMA coherent 的，而且底层的 DMA buffer 不合其他数据共享 cacheline，这样的系统将工作的更好。

```c
/* map单个的dma buffer */
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
void *addr = buffer->ptr;
size_t size = buffer->len;

dma_handle = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
    goto map_error_handling;
}

/* umap单个的dma buffer */
dma_unmap_single(dev, dma_handle, size, direction);
```

当调用 `dma_map_single()` 返回错误的时候，应当调用 `dma_mapping_error()` 来处理错误。虽然并不是所有的 DMA mapping 实现都支持 `dma_mapping_error` 这个接口（调用 `dma_mapping_error` 函数实际上会调用底层 `dma_map_ops` 操作函数集中的 `mapping_error` 成员函数），但是调用它来进行出错处理仍然是一个好的做法。这样做的好处是可以确保 DMA mapping 代码在所有 DMA 实现中都能正常工作，而不需要依赖底层实现的细节。没有检查错误就使用返回的地址可能会导致程序失败，可能会产生 kernel panic 或者悄悄的损坏有用的数据。上述同样适用于`dma_map_page()`。

当 DMA 传输完成的时候，程序应该调用 `dma_unmap_single()` 函数 umap dma buffer。例如：在 DMA 完成传输后会通过中断通知 CPU，而在 interrupt handler 中可以调用 `dma_unmap_single()` 函数。`dma_map_single` 函数在进行 DMA mapping 的时候使用的是 CPU 指针（虚拟地址），这样就导致该函数有一个弊端：不能使用 HIGHMEM memory 进行mapping。鉴于此，map/unmap 接口提供了另外一个类似的接口，这个接口不使用 CPU 指针，而是使用 page 和 page offset 来进行 DMA mapping：

```c
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
struct page *page = buffer->page;
unsigned long offset = buffer->offset;
size_t size = buffer->len;

dma_handle = dma_map_page(dev, page, offset, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
    goto map_error_handling;
}
...
dma_unmap_page(dev, dma_handle, size, direction);
```

在上面的代码中，`offset` 表示一个指定 page 内的页内偏移（以 Byte 为单位）。和 `dma_map_single` 接口函数一样，调用 `dma_map_page()` 返回错误后需要调用 `dma_mapping_error()` 来进行错误处理。当 DMA 传输完成的时候，程序应该调用 `dma_unmap_page()` 函数 umap dma buffer。例如：在 DMA 完成传输后会通过中断通知 CPU，而在 interrupt handler 中可以调用 `dma_unmap_page()` 函数。

```c
/* 在 scatterlist 的情况下，映射的对象是分散的若干段 DMA buffer */
int i, count = dma_map_sg(dev, sglist, nents, direction);
struct scatterlist *sg;

for_each_sg(sglist, sg, count, i) {
    hw_address[i] = sg_dma_address(sg);
    hw_len[i] = sg_dma_len(sg);
}
```

`nents` 说明了 `sglist` 中条目的数量（即 map 多少段 dma buffer）。具体 DMA 映射的实现是自由的，它可以把 scatterlist 中的若干段连续的 DMA buffer 映射成一个大块的，连续的 bus address region。例如：如果 DMA mapping 是以 PAGE_SIZE 为粒度进行映射，那么那些分散的一块块的 dma buffer 可以被映射到一个对齐在 PAGE_SIZE，然后各个 dma buffer 依次首尾相接的一个大的总线地址区域上。这样做的好处就是对于那些不支持（或者支持有限）scatter-gather 的 DMA controller，仍然可以通过 mapping 来实现。`dma_map_sg` 调用识别的时候返回0，当调用成功的时候，返回成功 mapping 的数目。一旦调用成功，你需要调用 `for_each_sg` 来遍历所有成功映射的 mappings（这个数目可能会小于 nents）并且使用 `sg_dma_address()` 和 `sg_dma_len()` 这两个宏来得到mapping 后的 dma 地址和长度。

```c
/* umap 多个形成 scatterlist 的 dma buffer */
dma_unmap_sg(dev, sglist, nents, direction);
```

调用 `dma_unmap_sg` 的时候要确保 DMA 操作已经完成。另外，传递给 `dma_unmap_sg`  的 `nents` 参数需要等于传递给 `dma_map_sg` 的 `nents` 参数，而不是该函数返回的count。由于 DMA 地址空间是共享资源，每一次 `dma_map_{single,sg}()` 的调用都需要有其对应的 `dma_unmap_{single,sg}()`，如果你总是分配 dma 地址资源而不回收，那么系统将会由于 DMA address 被用尽而陷入不可用的状态。

如果需要多次访问同一个 streaming DMA buffer，并且在 DMA 传输之间读写 DMA Buffer 上的数据，要小心进行 DMA buffer 的 sync 操作，以便 CPU 和设备（DMA controller）可以看到最新的、正确的数据。

```c
/* dma_map_{single,sg}()映射完，在完成DMA传输之后，用以下接口来完成sync的操作，以便CPU可以看到最新的数据 */
dma_sync_single_for_cpu(dev, dma_handle, size, direction);
dma_sync_sg_for_cpu(dev, sglist, nents, direction);
```

如果 CPU 操作了 DMA buffer 的数据，又想把控制权交给设备上的 DMA 控制器，让 DMA controller 访问 DMA buffer，这时候，在真正让 HW（指 DMA 控制器）去访问 DMA buffer 之前，需要调用以下接口，以便device（也就是设备上的 DMA 控制器）可以看到 CPU 更新后的数据。

```c
dma_sync_single_for_device(dev, dma_handle, size, direction);
dma_sync_sg_for_device(dev, sglist, nents, direction);
```

需要强调的是：传递给 `dma_sync_sg_for_cpu()` 和 `dma_sync_sg_for_device()` 的 `nents` 参数需要等于传递给 `dma_map_sg` 的 `nents` 参数，而不是该函数返回的 `count`。在完成最后一次 DMA 传输之后，需要调用 DMA unmap 函数 `dma_unmap_{single,sg}()`。如果在第一次 `dma_map_*()` 调用和 `dma_unmap_*()` 之间， `DMA buffer` 中的数据没有修改，则不需要调用 `dma_sync_*()` 这样的 sync 操作。

##### DMA_ZONE

DMA ZONE 产生的本质原因是：不一定所有的 DMA 都可以访问到所有的内存，这本质上是硬件的设计限制。在32位X86计算机的条件下，ISA实际只可以访问16MB以下的内存。那么ISA上面假设有个网卡，要 DMA，超过16MB以上的内存，它根本就访问不到。所以 Linux 内核干脆简单一点，把16MB砍一刀，这一刀以下的内存单独管理。如果ISA的驱动要申请DMA buffer，带一个GFP_DMA标记来表明想从这个区域申请，Linux 内核保证申请的内存是可以访问的。所以DMA_ZONE的大小，以及DMA_ZONE要不要存在，都取决于实际的硬件是什么。

![dma](D:/Develop/Operating_System/images/dma5.jpeg)

DMA_ZONE的内存做什么都可以。DMA_ZONE的作用是让有缺陷的 DMA 对应的外设驱动申请 DMA buffer 的时候从这个区域申请而已，但是它不是专有的。其他所有人的内存（包括应用程序和内核）也可以来自这个区域。

```c
static void *__dma_alloc(struct device *dev, size_t size, dma_addr_t *handle,
			 			 gfp_t gfp, pgprot_t prot, bool is_coherent,
			 			 unsigned long attrs, const void *caller)
{
	u64 mask = min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit);
	...
    /* mask < 4G，内存会从 DAM_ZONE 里分配 */
	if (mask < 0xffffffffULL)
		gfp |= GFP_DMA;
	...
}
```

缺省情况下，`dma_alloc_coherent()` 申请的内存缺省是进行 uncache 配置的。但是现代 SoC 特别强，这样有一些 SoC 里面可以用硬件做 CPU 和外设的 cache coherence，如下图中的 cache coherent interconnect。这些 SoC 的厂商就可以把内核的通用实现 overwrite 掉，变成 `dma_alloc_coherent()` 申请的内存也是可以带 cache 的。

![dma](D:/Develop/Operating_System/images/dma6.jpeg)

绝大多数的 SoC 目前都支持和使用 CMA 技术，并且多数情况下，DMA coherent APIs 以 CMA 区域为申请的后端，这个时候，`dma_alloc_coherent()` 本质上用 `__alloc_from_contiguous()` 从 CMA 区域获取内存，申请出来的内存显然是物理连续的。但是，如果 IOMMU 存在（ARM里面叫SMMU）的话，DMA 完全可以访问非连续的内存，并且把物理上不连续的内存，用 IOMMU 进行重新映射为 I/O virtual address (IOVA)：

![dma](D:/Develop/Operating_System/images/dma7.jpeg)

在支持 SVA（Shared Virtual Addressing）的场景下，外设可以和 CPU 共享相同的虚拟地址，这样外设就可以直接共享进程的地址空间，即可以直接在进程的虚拟地址空间进行 DMA 操作。



### DMA_BUF

DMA_BUF 可以实现 buffer 在多个设备的共享，应用可以把一片底层驱动 A 的 buffer 导出到用户空间成为一个 fd，也可以把 fd 导入到底层驱动 B。当然，如果进行 `mmap()` 得到虚拟地址，CPU 也是可以在用户空间访问到已经获得用户空间虚拟地址的底层 buffer 的。

![dmabuf](D:/Develop/Operating_System/images/dmabuf1.png)

上图中，进程 A 访问设备 A 并获得其使用的 buffer 的 fd，之后通过 socket 把 fd 发送给进程 B，而后进程 B 导入 fd 到设备 B，B 获得对设备 A 中的 buffer 的共享访问。如果 CPU 也需要在用户态访问这片 buffer，则进行了 `mmap()` 动作。

为什么要共享DMA buffer？想象一个场景：当前有一个诉求需要把屏幕 framebuffer 的内容透过 gstreamer 多媒体组件的服务，变成 h264 的视频码流，广播到网络上面，变成流媒体播放。在这个场景中，为了提升性能，想尽一切可能的避免**内存拷贝**。

管理 framebuffer 的驱动可以把这片 buffer 在底层实现为 dma_buf，然后 graphics compositor 给这片 buffer 映射出来一个 fd1，之后透过 socket 发送 fd1 把这块内存交给 gstreamer 相关的进程，如果 gstreamer 相关的 "color space 硬件转换组件"、"H264编码硬件组件"可以透过收到的 fd1 还原出这些 dma_buf 的地址，则可以进行直接的加速操作了。比如 color space 透过接收到的 fd1 还原出 framebuffer 的地址，然后把转化的结果放到另外一片 dma_buf，之后 fd2 对应这片 YUV buffer 被共享给 h264 编码器，h264 编码器又透过 fd2 还原出 YUV buffer 的地址。

![dmabuf](D:/Develop/Operating_System/images/dmabuf2.png)

这里面的核心点就是 fd 只是充当了一个"句柄"，用户进程和设备驱动透过 fd 最终寻找到底层的 dma_buf，实现 buffer 在进程和硬件加速组件之间的 zero-copy，这里面唯一进行了 exchange 的就是fd。

再比如，如果把方向反过来，gstreamer 从网络上收到了视频流，把它透过一系列动作转换为一片 RGB 的 buffer，那么这片 RGB 的 buffer 最终还要在 graphics compositor 里面渲染到屏幕上，我们也需要透过 dma_buf 实现内存在 video 的 decoder 相关组件与 GPU 组件的共享。

![dmabuf](D:/Develop/Operating_System/images/dmabuf3.png)

通常，将分配 buffer 的模块成为 exporter，将使用该 buffer 的模块称为 importer 或 user。

- exporter 驱动申请或者引用导入的待共享访问的内存。
- exporter 驱动调用 `dma_buf_export()` 创建 dma_buf 对象，同时将自定义的 `struct dma_buf_ops` 方法集和步骤1中的内存挂载到 dma_buf 对象中。
- exporter 驱动调用 `dma_buf_fd()` 将步骤2中创建的 dma_buf 对象关联到全局可见的文件描述符fd，同时通过 ioctl 方法将 fd 传递给应用层。
- 用户程序将 fd 传递给 importer 驱动程序。
- importer 驱动通过调用 `dma_buf_get(fd)` 获取 dma_buf 对象。
- importer 驱动调用 `dma_buf_attach()` 和 `dma_buf_map_attachment()` 获取共享缓存的信息。

##### sg_table

sg_table 本质上是由一块块单个物理连续的 buffer 所组成的链表，但是这个链表整体上看却是离散的，因此它可以很好的描述从高端内存上分配出的离散 buffer。当然，它同样可以用来描述从低端内存理连续 buffer。

![dmabuf](D:/Develop/Operating_System/images/dmabuf4.png)

sg_table 代表着整个链表，而它的每一个链表项则由 scatterlist 来表示。因此，1个 scatterlist 也就对应着一块物理连续的 buffer。可以通过如下接口来获取一个 scatterlist 对应 buffer 的物理地址和长度：

```c
sg_dma_address(sgl)
sg_dma_len(sgl)
```

有了 buffer 的物理地址和长度，就可以将这两个参数配置到 DMA 硬件寄存器中，从而实现 DMA 硬件对这一小块 buffer 的访问。在没有 IOMMU 的情况下，访问整块离散 buffer 需要用 for 循环，不断的解析 scatterlist，不断的配置 DMA 硬件寄存器，且每次配置完 DMA 硬件寄存器后，都需要等待本次 DMA 传输完毕，效率很低。而 IOMMU 就是用来解析 sg_table 的，它会将 sg_table 内部一个个离散的小 buffer 映射到自己内部的设备地址空间，使得这整块 buffer 在自己内部的设备地址空间上是连续的。这样，在访问离散 buffer 的时候，只需要将 IOMMU 映射后的设备地址（与 MMU 映射后的 CPU 虚拟地址不是同一概念）和整块 buffer 的 size 配置到 DMA 硬件寄存器中即可，中途无需再多次配置，便完成了 DMA 硬件对整块离散 buffer 的访问，大大的提高了软件的效率。

##### Attach与物理内存分配

同一个 dma_buf 可能会被多个 DMA 硬件访问，而每个 DMA 硬件可能会因为自身硬件能力的限制，对这块 buffer 有自己特殊的要求。比如硬件 A 的寻址能力只有 0x0 ~ 0x10000000，而硬件 B 的寻址能力为 0x0 ~ 0x80000000，那么在分配 dma_buf 的物理内存时，就必须以硬件 A 的能力为标准进行分配，这样硬件 A 和 B 都可以访问这段内存。否则，如果只满足 B 的需求，那么 A 可能就无法访问超出 0x10000000 地址以外的内存空间，道理其实类似于木桶理论。
因此，attach 操作可以让 exporter 驱动根据不同的 device 硬件能力，来分配最合适的物理内存。通过设置 `device->dma_parms` 参数，来告知 exporter 驱动该 DMA 硬件的能力限制。

但如果dma_buf 的物理内存都是在 dma_buf_export() 的时候就分配好了的，而 attach 操作只能在 export 之后才能执行，如何确保已经分配好的内存是符合硬件能力要求的呢？其实内存既可以在 export 阶段分配，也可以在 map attachment 阶段分配，甚至可以在两个阶段都分配，这通常由 DMA 硬件能力来决定。

通常的策略如下（假设只有 A、B 两个硬件需要访问 dma_buf ）：

- 如果硬件 A 和 B 的寻址空间有交集，则在 export 阶段进行内存分配，分配时以 A / B 的交集为准；
- 如果硬件 A 和 B 的寻址空间没有交集，则只能在 map attachment 阶段分配内存。

对于第二种策略，因为 A 和 B 的寻址空间没有交集（即完全独立），所以它们实际上是无法实现内存共享的。此时的解决办法是： A 和 B 在 map attachment 阶段，都分配各自的物理内存，然后通过 CPU 或 通用DMA 硬件，将 A 的 buffer 内容拷贝到 B 的 buffer 中去，以此来间接的实现 buffer “共享”。

另外还有一种策略，先在 export 阶段分配好内存，然后在首次 map attachment 阶段通过 `dma_buf->attachments` 链表，与所有 device 的能力进行一一比对，如果满足条件则直接返回 sg_table；如果不满足条件，则重新分配符合所有 device 要求的物理内存，再返回新的 sg_table。

![dmabuf](D:/Develop/Operating_System/images/dmabuf5.png)

##### Cache一致性

CPU 在访问内存时是要经过 Cache 的，而 DMA 外设则是直接和 DDR 打交道，因此这就存在 Cache 一致性的问题了，即 Cache 里面的数据是否和 DDR 里面的数据保持一致。比如 DMA 外设早已将 DDR 中的数据改写了，而 CPU 却浑然不知，仍然在访问 Cache 里面暂存的旧数据。所以 Cache 一致性问题，只有在 CPU 参与访问的情况下才会发生。如果一个 dma_buf 自始自终都只被一个硬件访问（要么CPU，要么DMA），那么 Cache 一致性问题就不会存在。当然，如果一个 dma_buf 所对应的物理内存本身就是 Uncache 的（也叫一致性内存），或者说该 buffer 在被分配时是以 coherent 方式分配的，那么这种情况下，CPU 是不经过 cache 而直接访问 DDR 的，自然 Cache 一致性问题也就不存在了。

`dma_buf_map_attachment()` 函数的一个重要功能，那就是同步 Cache 操作。但是该函数通常使用的是 `dma_map_{single,sg}` 这种流式 DMA 映射接口来实现 Cache 同步操作，这类接口的特点就是 Cache 同步只是一次性的，即在 dma map 的时候执行一次 Cache Flush 操作，在 dma unmap 的时候执行一次 Cache Invalidate 操作，而这中间的过程是不保证 Cache 和 DDR 上数据的一致性的。因此如果 CPU 在 dma map 和 unmap 之间又去访问了这块内存，那么有可能 CPU 访问到的数据就只是暂存在 Cache 中的旧数据，这就带来了问题。

举个实际的例子：网卡驱动首先通过 `dma_map_single()` 将接收缓冲区映射给了网卡 DMA 硬件，此后便发起了 DMA 传输请求，等待网卡接收数据完成。当网卡接收完数据后，会触发中断，此时网卡驱动需要在中断里检查本次传输数据的有效性。如果是有效数据，则调用 `dma_unmap_single()` 结束本次 DMA 映射；如果不是，则丢弃本次数据，继续等待下一次 DMA 接收的数据。在这个过程中，检查数据有效性是通过 CPU 读取接收缓冲区中的包头来实现的，也只有在数据检查完成后，才能决定是否执行 `dma_unmap_single()` 操作。因此这里出现了 dma map 和 unmap 期间 CPU 要访问这段内存的需求。

针对这种情况，就需要在 CPU 访问内存前，先将 DDR 数据同步到 Cache 中（Invalidate）；在 CPU 访问结束后，将 Cache 中的数据回写到 DDR 上（Flush），以便 DMA 能获取到 CPU 更新后的数据。这也就是 dma-buf 给我们预留 `{begin,end}_cpu_access` 的原因。

dma_buf 为我们提供了如下内核 API，用来在 dma map 期间发起 CPU 访问操作：

- `dma_buf_begin_cpu_access()`
- `dma_buf_end_cpu_access()`

它们分别对应 dma_buf_ops 中的 begin_cpu_access 和 end_cpu_access 回调接口。

通常在驱动设计时， begin_cpu_access / end_cpu_access 使用如下流式 DMA 接口来实现 Cache 同步：

- `dma_sync_single_for_cpu() / dma_sync_single_for_device()`
- `dma_sync_sg_for_cpu() / dma_sync_sg_for_device()`

CPU 访问内存之前，通过调用 `dma_sync_{single,sg}_for_cpu()` 来 Invalidate Cache，这样 CPU 在后续访问时才能重新从 DDR 上加载最新的数据到 Cache 上。
CPU 访问内存结束之后，通过调用 `dma_sync_{single,sg}_for_device()` 来 Flush Cache，将 Cache 中的数据全部回写到 DDR 上，这样后续 DMA 才能访问到正确的有效数据。

##### 驱动代码示例

```c
/* exporter */
#include <linux/device.h>
#include <linux/dma-buf.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/miscdevice.h>
#include <linux/uaccess.h>

static struct dma_buf *dmabuf_exported;

static int exporter_attach(struct dma_buf *dmabuf, struct device *dev, struct dma_buf_attachment *attachment)
{
	pr_info("dmabuf attach device: %s\n", dev_name(dev));
	return 0;
}

static void exporter_detach(struct dma_buf *dmabuf, struct dma_buf_attachment *attachment)
{
	pr_info("dmabuf detach device: %s\n", dev_name(attachment->dev));
}

static struct sg_table *exporter_map_dma_buf(struct dma_buf_attachment *attachment, enum dma_data_direction dir)
{
	void *vaddr = attachment->dmabuf->priv;
	struct sg_table *table;

	table = kmalloc(sizeof(*table), GFP_KERNEL);

	sg_alloc_table(table, 1, GFP_KERNEL);
	sg_dma_len(table->sgl) = PAGE_SIZE;
    /* kzalloc */
	sg_dma_address(table->sgl) = dma_map_single(NULL, vaddr, PAGE_SIZE, dir);
    
    /* alloc_page */
    sg_dma_address(table->sgl) = dma_map_page(NULL, page, 0, PAGE_SIZE, dir);
    
	return table;
}

static void exporter_unmap_dma_buf(struct dma_buf_attachment *attachment, struct sg_table *table, enum dma_data_direction dir)
{
    /* kzalloc */
    dma_unmap_single(NULL, sg_dma_address(table->sgl), PAGE_SIZE, dir);
    
    /* alloc_page */
    dma_unmap_page(NULL, sg_dma_address(table->sgl), PAGE_SIZE, dir);
	sg_free_table(table);
	kfree(table);
}

static void exporter_release(struct dma_buf *dmabuf)
{
    /* kzalloc */
    kfree(dmabuf->priv);
    
    /* alloc_page */
    struct page *page = dma_buf->priv;
	put_page(page)
}

static void *exporter_vmap(struct dma_buf *dmabuf)
{
    /* kzalloc */
	return dmabuf->priv;
    
	/* alloc_page */
    struct page *page = dma_buf->priv;
	return vmap(&page, 1, 0, PAGE_KERNEL);
}

static void exporter_vunmap(struct dma_buf *dmabuf, void *vaddr)
{
    /* alloc_page */
    vunmap(vaddr);
}

static int exporter_mmap(struct dma_buf *dmabuf, struct vm_area_struct *vma)
{
    /* kzalloc */
	void *vaddr = dmabuf->priv;

	return remap_pfn_range(vma, vma->vm_start, virt_to_pfn(vaddr), PAGE_SIZE, vma->vm_page_prot);
    
    /* alloc_page */
    struct page *page = dma_buf->priv;

	return remap_pfn_range(vma, vma->vm_start, page_to_pfn(page), PAGE_SIZE, vma->vm_page_prot);
}

static int exporter_begin_cpu_access(struct dma_buf *dmabuf, enum dma_data_direction dir)
{
    /* kzalloc */
	dma_addr_t dma_addr = virt_to_phys(dmabuf->priv);

	dma_sync_single_for_cpu(NULL, dma_addr, PAGE_SIZE, dir);
    
    /* alloc_page */
	struct dma_buf_attachment *attachment;
	struct sg_table *table;

	if (list_empty(&dmabuf->attachments))
		return 0;

	attachment = list_first_entry(&dmabuf->attachments, struct dma_buf_attachment, node);
	table = attachment->priv;
	dma_sync_sg_for_cpu(NULL, table->sgl, 1, dir);

	return 0;
}

static int exporter_end_cpu_access(struct dma_buf *dmabuf, enum dma_data_direction dir)
{
    /* kzalloc */
	dma_addr_t dma_addr = virt_to_phys(dmabuf->priv);

	dma_sync_single_for_device(NULL, dma_addr, PAGE_SIZE, dir);
    
    /* alloc_page */
    struct dma_buf_attachment *attachment;
	struct sg_table *table;

	if (list_empty(&dmabuf->attachments))
		return 0;

	attachment = list_first_entry(&dmabuf->attachments, struct dma_buf_attachment, node);
	table = attachment->priv;
	dma_sync_sg_for_device(NULL, table->sgl, 1, dir);

	return 0;
}

static const struct dma_buf_ops exp_dmabuf_ops = {
	.attach = exporter_attach,
	.detach = exporter_detach,
	.map_dma_buf = exporter_map_dma_buf,
	.unmap_dma_buf = exporter_unmap_dma_buf,
	.release = exporter_release,
	.vmap = exporter_vmap,
	.vunmap = exporter_vunmap,
    .mmap = exporter_mmap,
	.begin_cpu_access = exporter_begin_cpu_access,
	.end_cpu_access = exporter_end_cpu_access,
};

static struct dma_buf *exporter_alloc_page(void)
{
	DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
	struct dma_buf *dmabuf;
	void *vaddr;
    struct page *page;

    /* kzalloc */
	vaddr = kzalloc(PAGE_SIZE, GFP_KERNEL);
    /* alloc_page */
    page = alloc_page(GFP_KERNEL);

	exp_info.ops = &exp_dmabuf_ops;
	exp_info.size = PAGE_SIZE;
	exp_info.flags = O_CLOEXEC;
    /* kzalloc */
	exp_info.priv = vaddr;
    /* alloc_page */
    exp_info.priv = page;

	dmabuf = dma_buf_export(&exp_info);

    /* kzalloc */
	sprintf(vaddr, "hello world!");
    /* alloc_page */
    sprintf(page_address(page), "hello world!");

	return dmabuf;
}

static int exporter_misc_mmap(struct file *file, struct vm_area_struct *vma)
{
	return dma_buf_mmap(dmabuf_exported, vma, 0);
}

static long exporter_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int fd = dma_buf_fd(dmabuf_exported, O_CLOEXEC);
	copy_to_user((int __user *)arg, &fd, sizeof(fd));

	return 0;
}
 
static struct file_operations exporter_fops = {
	.owner		    = THIS_MODULE,
	.unlocked_ioctl	= exporter_ioctl,
    .mmap	        = exporter_misc_mmap,
};
 
static struct miscdevice mdev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name  = "exporter",
	.fops  = &exporter_fops,
};
 
static int __init exporter_init(void)
{
	dmabuf_exported = exporter_alloc_page();
	return misc_register(&mdev);
}

static void __exit exporter_exit(void)
{
	misc_deregister(&mdev);
}

module_init(exporter_init);
module_exit(exporter_exit);

/* importer */
#include <linux/device.h>
#include <linux/dma-buf.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/miscdevice.h>
#include <linux/uaccess.h>

extern struct dma_buf *dmabuf_exported;

static int importer_test(struct dma_buf *dmabuf)
{   
    void *vaddr;
    /* 将 vmap 操作放到了 begin / end 操作中间，以确保读取数据的正确性，仅为示例，实际不需要 */
    dma_buf_begin_cpu_access(dmabuf, DMA_FROM_DEVICE);

    /* 把实际的物理内存，映射到 kernel 空间，并转化成 CPU 可以连续访问的虚拟地址，方便后续软件直接读写这块物理内存。
     * 因此，无论这块 buffer 在物理上是否连续，在经过 vmap 映射后的虚拟地址一定是连续的。
     * 该函数对应 dma_buf_ops 中的 vmap 回调接口
     */
	vaddr = dma_buf_vmap(dmabuf);
	pr_info("read from dmabuf vmap: %s\n", (char *)vaddr);
	dma_buf_vunmap(dmabuf, vaddr);
    
    dma_buf_end_cpu_access(dmabuf, DMA_FROM_DEVICE);

	struct dma_buf_attachment *attachment;
	struct sg_table *table;
	struct device *dev;
	unsigned int reg_addr, reg_size;
    
	dev = kzalloc(sizeof(*dev), GFP_KERNEL);
	dev_set_name(dev, "importer");

    /* 该函数实际是 “dma-buf attach device” 的缩写，用于建立一个 dma_buf 与 device 的连接关系，
     * 这个连接关系被存放在新创建的 dma_buf_attachment 对象中，供后续调用 dma_buf_map_attachment() 使用。
     * 该函数对应 dma_buf_ops 中的 attach 回调接口，如果 device 对后续的 map attachment 操作没有什么特殊要求，可以不实现。
     */
	attachment = dma_buf_attach(dmabuf, dev);
    /* 该函数实际是 “dma-buf map attachment into sg_table” 的缩写，主要完成2件事情：1) 生成 sg_table; 2) 同步 Cache
     * 选择返回 sg_table 而不是物理地址，是为了兼容所有 DMA 硬件（带或不带 IOMMU），因为 sg_table 既可以表示连续物理内存，也可以表示非连续物理内存。
     * 同步 Cache 是为了防止该 buffer 事先被 CPU 填充过，数据暂存在 Cache 中而非 DDR 上，导致 DMA 访问的不是最新的有效数据。
     * 通过将 Cache 中的数据回写到 DDR 上可以避免此类问题的发生。同样的，在 DMA 访问内存结束后，需要将 Cache 设置为无效，
     * 以便后续 CPU 直接从 DDR 上（而非 Cache 中）读取该内存数据。通常我们使用如下流式 DMA 映射接口来完成 Cache 的同步：
     * dma_map_single() / dma_unmap_single()
	 * dma_map_page() / dma_unmap_page()
	 * dma_map_sg() / dma_unmap_sg()
     * dma_buf_map_attachment() 对应 dma_buf_ops 中的 map_dma_buf 回调接口
	 */
	table = dma_buf_map_attachment(attachment, DMA_BIDIRECTIONAL);

	reg_addr = sg_dma_address(table->sgl);
	reg_size = sg_dma_len(table->sgl);
	pr_info("reg_addr = 0x%08x, reg_size = 0x%08x\n", reg_addr, reg_size);

	dma_buf_unmap_attachment(attachment, table, DMA_BIDIRECTIONAL);
	dma_buf_detach(dmabuf, attachment);

	return 0;
}

static long importer_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int fd;
	struct dma_buf *dmabuf;

	copy_from_user(&fd, (void __user *)arg, sizeof(int));

	dmabuf = dma_buf_get(fd);
	importer_test(dmabuf);
	dma_buf_put(dmabuf);

	return 0;
}
 
static struct file_operations importer_fops = {
	.owner	= THIS_MODULE,
	.unlocked_ioctl	= importer_ioctl,
};
 
static struct miscdevice mdev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = "importer",
	.fops = &importer_fops,
};
 
static int __init importer_init(void)
{
	return misc_register(&mdev);
}

static void __exit importer_exit(void)
{
	misc_deregister(&mdev);
}

module_init(importer_init);
module_exit(importer_exit);

/* userspace */
int main(int argc, char *argv[])
{
	int fd;
	int dmabuf_fd = 0;
    struct dma_buf_sync sync = { 0 };
    /* 首先通过 exporter 驱动的 ioctl() 获取到 dma_buf 的 fd，然后直接使用该 fd 做 mmap() 映射，
     * 最后使用 printf() 来输出映射后的 buffer 内容。
     */
	fd = open("/dev/exporter", O_RDONLY);
	ioctl(fd, 0, &dmabuf_fd);
	close(fd);
    
    /* 将 mmap() 操作放到了 ioctl SYNC_START / SYNC_END 之间，以确保读取数据的正确性，仅为示例，实际不需要 */
	sync.flags = DMA_BUF_SYNC_READ | DMA_BUF_SYNC_START;
	ioctl(dmabuf_fd, DMA_BUF_IOCTL_SYNC, &sync);

	char *str = mmap(NULL, 4096, PROT_READ, MAP_SHARED, dmabuf_fd, 0);
	printf("read from dmabuf mmap: %s\n", str);
    
    sync.flags = DMA_BUF_SYNC_READ | DMA_BUF_SYNC_END;
	ioctl(dmabuf_fd, DMA_BUF_IOCTL_SYNC, &sync);
    
	/* 不再通过 ioctl() 方式获取 dma-buf 的 fd，而是直接使用 exporter misc device 的 fd 进行 mmap() 操作，
	 * 此时执行的则是 misc driver 的 mmap 文件操作接口。
	 */
	fd = open("/dev/exporter", O_RDONLY);

	char *str = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
	printf("read from /dev/exporter mmap: %s\n", str);

	close(fd);
    
    /* 将 dma-buf 的 fd 从 exporter 传递给 importer 驱动 */
	fd = open("/dev/exporter", O_RDONLY);
	ioctl(fd, 0, &dmabuf_fd);
	close(fd);

	fd = open("/dev/importer", O_RDONLY);
	ioctl(fd, 0, &dmabuf_fd);
	close(fd);

	return 0;
}
```
