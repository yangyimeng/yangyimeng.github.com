---
layout: post
title: "Netgear路由器移植android系统"
description: ""
category: "work android linux"
tags: []
sumarry: |
  此篇文章目的在于简单说明移植android系统到Netgear路由器上的一些具体的步骤.其中工作量主要在于linux内核的移植和文件系统的移植.而linux内核的移植部分主要的因为涉及到了驱动问题,还有专有平台的问题,都需要根据具体的平台进行修改.而文件系统部分则是要只保留下核心的部分,上层的框架全部剥离系统.其次,一个额外的工作量起源于大小端的问题,由于路由器是大端系统,但是android默认的是小端的,且由于所选的android源码对大端系统的支持不够,因此需要进行大量的修改.
---
{% include JB/setup %}

####概述

此篇文章目的在于简单说明移植android系统到Netgear路由器上的一些具体的步骤.其中工作量主要在于linux内核的移植和文件系统的移植.而linux内核的移植部分主要的因为涉及到了驱动问题,还有专有平台的问题,都需要根据具体的平台进行修改.而文件系统部分则是要只保留下核心的部分,上层的框架全部剥离系统.其次,一个额外的工作量起源于大小端的问题,由于路由器是大端系统,但是android默认的是小端的,且由于所选的android源码对大端系统的支持不够,因此需要进行大量的修改.


####目录
1. [linux内核移植](#a1)
2. [文件系统裁剪](#a2)
3. [大小端系统兼容处理](#a3)

<br/>
<br/>






<a name="a1"> </a>

####1. linux内核移植

因为要运行的是android系统,而android系统所使用的linux内核与原版的linux内核有些修改.因此,为了方便,直接使用官网提供的linux内核.但是因为使用了官网的内核,这个内核并没有Netgear路由器开发板的平台相关配置以及一系列的驱动.

<div class="ymy_div">
<p class="triangle-isosceles left">这部分的相关参考笔记为2012-7-23 到 2012-8-20期间的笔记(android移植笔记.chm)</p>
</div>

###1.0 调试方法

为了方便进行测试内核状况, 在移植过程里,使用了tftp刷内核和串口调试方案,其中由于Netgear路由器内置的u-boot有支持tftp的功能,因此只要在主机上开tftp服务器,在路由器上的u-boot命令里,直接用

<pre class="brush: js;"> 
tftp 82000000 uImage
</pre>

<p>其中, 82000000表示要将主机上的uImage载入到内存的82000000地址上,记住82000000这个地址要慎重选举,一般要参照内核的load address. 最主要的是不要覆盖u-boot的地址,否者u-boot将会损坏.</p>

<p>
至于串口调试,就是简单的用串口线将路由器和主机连线,而后通过minicom来进行调试.内核的调试信息都会输出到串口终端.
</p>



###1.1 获取内核

<pre class="brush: js;">
git clone git://git.linux-mips.org/pub/scm/linux-mti.git
</pre>

获取了内核之后,就开始我们的工作了.

###1.2 编译

编译这个源码只要修改一下Makefile就可以了,为了方便,直接也把平台选好.
~/Makefile

<pre class="brush: js;">
diff --git a/Makefile b/Makefile
index 7f30a22..24f4e49 100644
--- a/Makefile
+++ b/Makefile
@@ -193,7 +193,7 @@ SUBARCH := $(shell uname -m | sed -e s/i.86/i386/ -e s/sun4u/sparc64/ \
 # Note: Some architectures assign CROSS_COMPILE in their arch/*/Makefile
 export KBUILD_BUILDHOST := $(SUBARCH)
 ARCH           ?= mips
-CROSS_COMPILE  ?= $(CONFIG_CROSS_COMPILE:"%"=%)
+CROSS_COMPILE  ?= mips-linux-gnu-
 
 # Architecture as present in compile.h
 UTS_MACHINE    := $(ARCH)
</pre>


###1.3 添加平台

由于Netgear路由器的开发板是"NETGEAR WNDR3700 board", 但是官网的内核没有支持,因此,我们需要自己添加对这个board的支持.


##Kconfig文件与Makefile

~/arch/mips/ath79/Kconfig

<pre class="brush: js;">
diff --git a/arch/mips/ath79/Kconfig b/arch/mips/ath79/Kconfig
index 4770741..04b4b81 100644
--- a/arch/mips/ath79/Kconfig
+++ b/arch/mips/ath79/Kconfig
@@ -23,6 +23,16 @@ config ATH79_MACH_PB44
          Say 'Y' here if you want your kernel to support the
          Atheros PB44 reference board.
 
+config ATH79_MACH_WNDR3700
+       bool "NETGEAR WNDR3700 board support"
+       select SOC_AR71XX
+       select ATH79_DEV_AP9X_PCI if PCI
+       select ATH79_DEV_ETH
+       select ATH79_DEV_GPIO_BUTTONS
+       select ATH79_DEV_LEDS_GPIO
+       select ATH79_DEV_M25P80
+       select ATH79_DEV_USB
+
 endmenu
 
 config SOC_AR71XX
@@ -52,4 +62,7 @@ config ATH79_DEV_LEDS_GPIO
 config ATH79_DEV_SPI
        def_bool n
 
+config ATH79_DEV_ETH
+       def_bool n
+
 endif
</pre>

~/arch/mips/ath79/Makefile

<pre class="brush: js;">
diff --git a/arch/mips/ath79/Makefile b/arch/mips/ath79/Makefile
index c33d465..3a83437 100644
--- a/arch/mips/ath79/Makefile
+++ b/arch/mips/ath79/Makefile
@@ -20,9 +20,11 @@ obj-$(CONFIG_ATH79_DEV_AR913X_WMAC)  += dev-ar913x-wmac.o
 obj-$(CONFIG_ATH79_DEV_GPIO_BUTTONS)   += dev-gpio-buttons.o
 obj-$(CONFIG_ATH79_DEV_LEDS_GPIO)      += dev-leds-gpio.o
 obj-$(CONFIG_ATH79_DEV_SPI)            += dev-spi.o
+obj-$(CONFIG_ATH79_DEV_ETH)            += dev-eth.o
 
 #
 # Machines
 #
 obj-$(CONFIG_ATH79_MACH_AP81)          += mach-ap81.o
 obj-$(CONFIG_ATH79_MACH_PB44)          += mach-pb44.o
+obj-$(CONFIG_ATH79_MACH_WNDR3700)      += mach-wndr3700.o
</pre>



###1.4 添加网卡驱动

由于在移植的过程里, 为了方便调试, 我使用的是nfs网络文件系统进行挂载android系统.因此, 第一步就是需要把网卡打通,也就是需要移植网卡驱动到内核,且使其要在文件系统挂载前网卡就可以使用.这就需要在机器启动的某个地方将网卡启动起来.

<pre class="brush: js;">
diff --git a/drivers/net/ethernet/Kconfig b/drivers/net/ethernet/Kconfig
new file mode 100644
index 0000000..eca6dc4
--- /dev/null
+++ b/drivers/net/ethernet/Kconfig
@@ -0,0 +1,17 @@
+#
+# Ethernet LAN device configuration
+#
+
+menuconfig ETHERNET
+       bool "Ethernet driver support"
+       depends on NET
+       default y
+       ---help---
+         This section contains all the Ethernet device drivers.
+
+if ETHERNET
+
+source "drivers/net/ethernet/atheros/Kconfig"
+
+endif # ETHERNET
+
</pre>

<br/>
<br/>

<pre class="brush: js;">
diff --git a/drivers/net/ethernet/Makefile b/drivers/net/ethernet/Makefile
new file mode 100644
index 0000000..8c4bac8
--- /dev/null
+++ b/drivers/net/ethernet/Makefile
@@ -0,0 +1,5 @@
+#
+# Makefile for the Linux network Ethernet device drivers.
+#
+
+obj-$(CONFIG_NET_VENDOR_ATHEROS) += atheros/
</pre>


##添加网卡启动

因为使用的是nfs文件系统,因此,需要再挂载文件系统之前就必须将网卡打通,就是启动起来.ath79_generic_init函数是开发板初始化的操作,我就在这里面把网卡的一系列的操作写在这里了.

~/arch/mips/ath79/setup.c

<pre class="brush: js;">
static void __init ath79_generic_init(void)
{
	/* Nothing to do */
	u8 *art = (u8 *) KSEG1ADDR(0x1fff0000);
	printk(KERN_ERR "###############ath79_generic_init arch-mips-ath79-setup.c\n");
	/*
	 * The eth0 and wmac0 interfaces share the same MAC address which
	 * can lead to problems if operated unbridged. Set the locally
	 * administered bit on the eth0 MAC to make it unique.
	 */
	ath79_init_local_mac(ath79_eth0_data.mac_addr,
			     art + WNDR3700_ETH0_MAC_OFFSET);
	ath79_eth0_pll_data.pll_1000 = 0x11110000;
	ath79_eth0_data.mii_bus_dev = &amp;wndr3700_rtl8366s_device.dev;
	ath79_eth0_data.phy_if_mode = PHY_INTERFACE_MODE_RGMII;
	ath79_eth0_data.speed = SPEED_1000;
	ath79_eth0_data.duplex = DUPLEX_FULL;

	ath79_init_mac(ath79_eth1_data.mac_addr,
			art + WNDR3700_ETH1_MAC_OFFSET, 0);
	ath79_eth1_pll_data.pll_1000 = 0x11110000;
	ath79_eth1_data.mii_bus_dev = &amp;wndr3700_rtl8366s_device.dev;
	ath79_eth1_data.phy_if_mode = PHY_INTERFACE_MODE_RGMII;
	ath79_eth1_data.phy_mask = 0x10;

	ath79_register_eth(0);
	ath79_register_eth(1);

	//ath79_register_usb();

	//ath79_register_m25p80(NULL);

	ath79_register_leds_gpio(-1, ARRAY_SIZE(wndr3700_leds_gpio),
				 wndr3700_leds_gpio);

	ath79_register_gpio_keys_polled(-1, WNDR3700_KEYS_POLL_INTERVAL,
					ARRAY_SIZE(wndr3700_gpio_keys),
					wndr3700_gpio_keys);

	platform_device_register(&amp;wndr3700_rtl8366s_device);
	platform_device_register_simple("wndr3700-led-usb", -1, NULL, 0);

	ap9x_pci_setup_wmac_led_pin(0, 5);
	ap9x_pci_setup_wmac_led_pin(1, 5);

	/* 2.4 GHz uses the first fixed antenna group (1, 0, 1, 0) */
	ap9x_pci_setup_wmac_gpio(0, (0xf &lt;&lt; 6), (0xa &lt;&lt; 6));

	/* 5 GHz uses the second fixed antenna group (0, 1, 1, 0) */
	ap9x_pci_setup_wmac_gpio(1, (0xf &lt;&lt; 6), (0x6 &lt;&lt; 6));

	ap94_pci_init(art + WNDR3700_CALDATA0_OFFSET,
		      art + WNDR3700_WMAC0_MAC_OFFSET,
		      art + WNDR3700_CALDATA1_OFFSET,
		      art + WNDR3700_WMAC1_MAC_OFFSET);
	printk(KERN_ERR "###############ath79_generic_init arch-mips-ath79-setup.c over \n");

}
</pre>


<a name="a2"> </a>

####2. 文件系统裁剪


###2.0 调试方法

同样的,这里的调试跟前面的一样,也是通过串口调试的,不过由于不想每次修改文件系统后,都要重新将文件系统刷入flash里面,很耗时间,因此,采用的是nfs文件系统挂载的方式.而若要使用nfs文件系统的挂载方式的话,就有几个步骤需要保证做到.


<div class="ymy_div">
<p class="triangle-isosceles left">这部分的相关参考笔记为2012-7-27 到 2012-9-24期间的笔记</p>
</div>

1. [网卡驱动](#p1)
2. [内核的启动参数](#p2)
3. [内核的nfs配置](#p3)


其中,关于网卡驱动的内容下面将会着重讲,这里主要的还是说说启动参数和nfs配置吧.

##内核启动参数

<pre class="brush: js;"> 
noinitrd console=ttyS0,115200  root=/dev/nfs rw rootpath=/home/ymy/nfsdir/rootfs 
init=/init nfsroot=192.168.1.144:/home/ymy/nfsdir/rootfs  
ip=192.168.1.2:192.168.1.144:192.168.1.1:255.255.255.0:linux:eth0:off Mem=128M
</pre>


下面来分析下上面的启动参数,其中root=/dev/nfs,表明了内核的启动方式是nfs的方式,而rootpath指定了root的挂载目录,init指定了挂载文件系统要执行的入口,其中192.168.1.144是nfs的服务器的ip地址,192.168.1.2是路由器的ip地址.

##nfs配置

要保证内核支持nfs启动,最主要的是要开启如下的配置.

<pre class="brush: js;"> 
Depends on: NETWORK_FILESYSTEMS [=y] &amp;&amp; NFS_FS [=y]=y &amp;&amp; IP_PNP [=y] 
[*]   Root file system on NFS   
</pre>

Root file system on NFS配置启动,将保证内核支持nfs启动.


###2.1 编译

这里的编译跟一般的android的编译差不多,不过有几个地方需要注意.

1. [指定目标系统为大端系统.](http://)
2. [修改交叉编译工具,要指定为大端系统的工具.](http://)



##指定目标系统为大端系统


<pre class="brush: js;">
mipsandroid$ source build/envsetup.sh 
including device/moto/stingray/vendorsetup.sh
including device/moto/wingray/vendorsetup.sh
including device/samsung/crespo4g/vendorsetup.sh
including device/samsung/crespo/vendorsetup.sh
including device/samsung/maguro/vendorsetup.sh
including device/samsung/torospr/vendorsetup.sh
including device/samsung/toro/vendorsetup.sh
including device/samsung/tuna/vendorsetup.sh
including device/ti/panda/vendorsetup.sh
including sdk/bash_completion/adb.bash
</pre>

通过环境变量指定目标的系统为大端系统,其中eb意思是endian big
的意思.

<pre class="brush: js;">
lauch full_mips_eng
export TARGET_ARCH_VARIANT=mips32r2-fp-eb
</pre>

最终的系统参数如下

<pre class="brush: js;">
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=4.0.4
TARGET_PRODUCT=full_mips
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=mips
TARGET_ARCH_VARIANT=mips32r2-fp-eb
HOST_ARCH=x86
HOST_OS=linux
HOST_BUILD_TYPE=release
BUILD_ID=IMM76D
</pre>



##交叉编译工具

交叉编译与一般的android编译不一样的地方在于要指定为大端的工具,否者编译过程会出错.

<pre class="brush: js;">
diff --git a/build/core/combo/TARGET_linux-mips.mk b/build/core/combo/TARGET_linux-mips.mk
index cb68b1a..85ad486 100644
--- a/build/core/combo/TARGET_linux-mips.mk
+++ b/build/core/combo/TARGET_linux-mips.mk
@@ -43,19 +43,26 @@ include $(TARGET_ARCH_SPECIFIC_MAKEFILE)
 
 # You can set TARGET_TOOLS_PREFIX to get gcc from somewhere else
 ifeq ($(strip $(TARGET_TOOLS_PREFIX)),)
+#$(warning ******************************************************  )
 TARGET_TOOLS_PREFIX := \
-       prebuilt/$(HOST_PREBUILT_TAG)/toolchain/mips-linux-android-4.5.2/bin/mips-linux-android-
+       prebuilt/$(HOST_PREBUILT_TAG)/toolchain/mips-4.4.6/bin/mips-linux-gnu-
+#$(warning TARGET_TOOLS_PREFIX is $(TARGET_TOOLS_PREFIX) )
+#$(warning ******************************************************  )
 endif
 
 # Only define these if there's actually a gcc in there.
 # The gcc toolchain does not exists for windows/cygwin. In this case, do not reference it.
 ifneq ($(wildcard $(TARGET_TOOLS_PREFIX)gcc$(HOST_EXECUTABLE_SUFFIX)),)
+    #$(warning @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ )
     TARGET_CC := $(TARGET_TOOLS_PREFIX)gcc$(HOST_EXECUTABLE_SUFFIX)
     TARGET_CXX := $(TARGET_TOOLS_PREFIX)g++$(HOST_EXECUTABLE_SUFFIX)
     TARGET_AR := $(TARGET_TOOLS_PREFIX)ar$(HOST_EXECUTABLE_SUFFIX)
     TARGET_OBJCOPY := $(TARGET_TOOLS_PREFIX)objcopy$(HOST_EXECUTABLE_SUFFIX)
     TARGET_LD := $(TARGET_TOOLS_PREFIX)ld$(HOST_EXECUTABLE_SUFFIX)
     TARGET_STRIP := $(TARGET_TOOLS_PREFIX)strip$(HOST_EXECUTABLE_SUFFIX)
+    $(warning TARGET_CC is $(TARGET_CC) )
+
+    #$(warning @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ )
     ifeq ($(TARGET_BUILD_VARIANT),user)
         TARGET_STRIP_COMMAND = $(TARGET_STRIP) --strip-all $&lt; -o $@
     else
</pre>




###2.2 裁剪概括

由于运行在路由器上的android系统不需要太多的东西,因此,我将android的所有的ui以及应用程序框架的部分全部的都删掉了,为了方便,我是在tiny android的基础上进行工作的.tiny android是android源码提供的一个模式,这个模式使得编译出来的系统,只有linux内核,而跟android有关的dalvik虚拟机或者上层的都不存在.其实也就是说是一个简单的shell,不过有关android的东西都不存在.那我下面的工作就是在这个模式下进行添加dalvik虚拟机进来.从全局的来讲,可以初略用下面的patch进行概括.

<pre class="brush: js;">
--- build/core/main.mk	2012-08-30 07:56:17.000000000 -0700
+++ /home/ymy/work/mipsandroid/build/core/main.mk	2012-09-28 22:54:37.000000000 -0700
@@ -463,10 +463,8 @@
 
 else	# !SDK_ONLY
 ifeq ($(BUILD_TINY_ANDROID), true)
-
 # TINY_ANDROID is a super-minimal build configuration, handy for board
 # bringup and very low level debugging
-
 subdirs := \
 	bionic \
 	system/core \
@@ -474,13 +472,30 @@
 	system/extras/su \
 	build/libs \
 	build/target \
+	frameworks/base/libs/utils \
 	build/tools/acp \
+	external/gtest \
 	external/mksh \
 	external/yaffs2 \
-	external/zlib \
 	external/stlport \
-	external/gtest \
-	frameworks/base/libs/utils
+	external/zlib \
+	external/expat \
+	external/openssl/crypto\
+	external/openssl/ssl \
+	external/icu4c/common \
+	external/icu4c/stubdata \
+	external/icu4c/i18n \
+	dalvik/vm \
+	dalvik/libdex \
+	dalvik/libnativehelper \
+	libcore \
+	external/fdlibm \
+	external/libffi \
+	abi/cpp \
+	dalvik/dexopt \
+	dalvik/dalvikvm
+
+	
 else	# !BUILD_TINY_ANDROID
</pre>





<a name="a3"> </a>

####3. 大小端系统兼容处理

在移植系统的时候, 大小端的问题是一定要考虑到的.由于路由器是大端的,但是呢,android的文件系统一般的都是小端系统.因此在编译的过程里出现了很多的问题.有需要的地方都需要修改.


<div class="ymy_div">
<p class="triangle-isosceles left">这部分的相关参考笔记为2012-7-23 到 2012-8-20期间的笔记</p>
</div>

###3.1 编译相关的修改

在android源码里,没有为大端系统__fswab16做出相关的定义,因此这部分需要我们自己定义好.宏定义参考linux源码.

<pre class="brush: js;">
diff --git a/bionic/libc/kernel/common/linux/byteorder/swab.h b/bionic/libc/kernel/common/linux/byteorder/swab.h
index 37336b5..48a20d5 100644
--- a/bionic/libc/kernel/common/linux/byteorder/swab.h
+++ b/bionic/libc/kernel/common/linux/byteorder/swab.h
@@ -24,6 +24,22 @@
 #define ___constant_swab32(x)   ((__u32)(   (((__u32)(x) &amp; (__u32)0x000000ffUL) &lt;&lt; 24) |   (((__u32)(x) &amp; (__u32)0x0000ff00UL) &lt;&lt; 8) |   (((__u32)(x) &amp; (__u32)0x00ff0000UL) >> 8) |   (((__u32)(x) &amp; (__u32)0xff000000UL) >> 24) ))
 #define ___constant_swab64(x)   ((__u64)(   (__u64)(((__u64)(x) &amp; (__u64)0x00000000000000ffULL) &lt;&lt; 56) |   (__u64)(((__u64)(x) &amp; (__u64)0x000000000000ff00ULL) &lt;&lt; 40) |   (__u64)(((__u64)(x) &amp; (__u64)0x0000000000ff0000ULL) &lt;&lt; 24) |   (__u64)(((__u64)(x) &amp; (__u64)0x00000000ff000000ULL) &lt;&lt; 8) |   (__u64)(((__u64)(x) &amp; (__u64)0x000000ff00000000ULL) >> 8) |   (__u64)(((__u64)(x) &amp; (__u64)0x0000ff0000000000ULL) >> 24) |   (__u64)(((__u64)(x) &amp; (__u64)0x00ff000000000000ULL) >> 40) |   (__u64)(((__u64)(x) &amp; (__u64)0xff00000000000000ULL) >> 56) ))
 
+
+/*
+ * Implement the following as inlines, but define the interface using
+ * macros to allow constant folding when possible:
+ * ___swab16, ___swab32, ___swab64, ___swahw32, ___swahb32
+ */
+
+static inline __attribute_const__ __u16 __fswab16(__u16 val)
+{
+#ifdef __arch_swab16
+	return __arch_swab16(val);
+#else
+	return ___constant_swab16(val);
+#endif
+}
+
 #ifndef __arch__swab16
 #define __arch__swab16(x) ({ __u16 __tmp = (x) ; ___swab16(__tmp); })
 #endif
</pre>


在system/core的源码里, 同样的没有对大端系统的convertAbgr8888ToRgb565_hi16做出定义.因此也需要自己定义,定义参考小端系统的定义进行修改.

<pre class="brush: js;">
diff --git a/system/core/libpixelflinger/scanline.cpp b/system/core/libpixelflinger/scanline.cpp
index be85df6..14ce226 100644
--- a/system/core/libpixelflinger/scanline.cpp
+++ b/system/core/libpixelflinger/scanline.cpp
@@ -131,6 +131,14 @@ static inline uint16_t  convertAbgr8888ToRgb565(uint32_t  pix)
                       ((pix >> 19) &amp; 0x001f) );
 }
 
+static inline uint16_t  convertAbgr8888ToRgb565_hi16(uint32_t  pix)
+{
+    fprintf(stderr, "convertAbgr8888ToRgb565_hi16 \n");
+    return uint16_t( ((pix &lt;&lt; 8) &amp; 0xf800) |
+                      ((pix >> 5) &amp; 0x07e0) |
+                      ((pix >> 19) &amp; 0x001f) );
+}
+
 struct shortcut_t {
     needs_filter_t  filter;
     const char*     desc;
</pre>


android源码对于大端系统的JValue联合体数据类型的l定义为Object,但是使用这个数据结构的函数,依旧识别为void *类型,因此,需要改为void *类型.

<pre class="brush: js;">
--- a/dalvik/vm/Common.h
+++ b/dalvik/vm/Common.h
@@ -134,7 +134,7 @@ union JValue {
     s8      j;
     float   f;
     double  d;
-    Object*   l;
+    void*   l;
 #endif
 };
 </pre>


android源码external/v8里没有支持大端系统,检测到大端系统时直接退出,因此,这部分需要修改为支持大端系统.

<pre class="brush: js;">
 diff --git a/external/v8/src/globals.h b/external/v8/src/globals.h
index 4ab0950..d0c78d6 100644
--- a/external/v8/src/globals.h
+++ b/external/v8/src/globals.h
@@ -86,9 +86,6 @@ namespace internal {
 #elif defined(__MIPSEL__)
 #define V8_HOST_ARCH_MIPS 1
 #define V8_HOST_ARCH_32_BIT 1
-#elif defined(__MIPSEB__)
-#define V8_HOST_ARCH_MIPS 1
-#define V8_HOST_ARCH_32_BIT 1
 #else
 #error Host architecture was not detected as supported by v8
 #endif
diff --git a/system/extras/tests/memset/memset_omips.S b/system/extras/tests/memset/memset_omips.S
index 2697264..788c644 100644
--- a/system/extras/tests/memset/memset_omips.S
+++ b/system/extras/tests/memset/memset_omips.S
@@ -26,7 +26,7 @@
 #endif
 
 #if defined(__MIPSEB__)
-//#error big-endian is not supported in Broadcom MIPS Android platform
+#error big-endian is not supported in Broadcom MIPS Android platform
 # define SWHI	swl		/* high part is left in big-endian	*/
 #else
 # define SWHI	swr		/* high part is right in little-endian	*/
</pre>



###3.2 dalvikvm相关的修改


与dalvikvm相关的修改到的文件如下

<pre class="brush: js;"> 
/dalvik/dalvikvm/Main.cpp
/dalvik/libdex/DexFile.cpp
/dalvik/libdex/DexOptData.cpp
/dalvik/libdex/DexSwapVerify.cpp
/dalvik/libnativehelper/JNIHelp.cpp
/dalvik/libnativehelper/include/nativehelper/jni.h
/dalvik/vm/CheckJni.cpp
/dalvik/vm/Debugger.cpp
/dalvik/vm/DvmDex.cpp
/dalvik/vm/Exception.cpp
/dalvik/vm/Exception.h
/dalvik/vm/Init.cpp
/dalvik/vm/InitRefs.cpp
/dalvik/vm/JarFile.cpp
/dalvik/vm/Jni.cpp
/dalvik/vm/Misc.cpp
/dalvik/vm/Thread.cpp
/dalvik/vm/alloc/Alloc.cpp
/dalvik/vm/alloc/Heap.cpp
/dalvik/vm/analysis/DexPrepare.cpp
/dalvik/vm/analysis/DexVerify.cpp
/dalvik/vm/interp/Interp.cpp
/dalvik/vm/interp/Stack.cpp
/dalvik/vm/mterp/Mterp.cpp
/dalvik/vm/mterp/c/OP_THROW.cpp
/dalvik/vm/mterp/c/gotoTargets.cpp
/dalvik/vm/mterp/out/InterpC-allstubs.cpp
/dalvik/vm/mterp/out/InterpC-mips.cpp
/dalvik/vm/mterp/out/InterpC-portable.cpp
/dalvik/vm/oo/Array.cpp
/dalvik/vm/oo/Class.cpp
/dalvik/vm/oo/Class.h
/external/icu4c/common/propname.cpp
/external/icu4c/common/putil.c
/external/icu4c/common/ucmndata.c
/external/icu4c/common/ucnv_io.c
/external/icu4c/common/udata.cpp
/external/icu4c/common/uinit.c
/external/icu4c/common/unicode/platform.h
/external/icu4c/common/uniset.cpp
/external/icu4c/common/uniset_props.cpp
/external/icu4c/common/unistr.cpp
/external/icu4c/i18n/regexcmp.cpp
/external/icu4c/i18n/regexst.cpp
/external/icu4c/i18n/regexst.h
/external/icu4c/i18n/repattrn.cpp
/external/icu4c/stubdata/Android.mk
/external/icu4c/stubdata/root.mk
/external/v8/src/json-parser.h
/libcore/include/ScopedJavaUnicodeString.h
/libcore/luni/src/main/native/Register.cpp
/libcore/luni/src/main/native/java_lang_ProcessManager.cpp
/libcore/luni/src/main/native/java_util_regex_Pattern.cpp
/libcore/luni/src/main/native/libcore_icu_ICU.cpp
/libcore/luni/src/main/native/libcore_io_Posix.cpp
/system/core/libcutils/ashmem-dev.c
/system/extras/su/su.c
</pre>


##odex文件大小端的转换
android源码编译的时候,会对core.jar等核心的jar进行优化得到对应的odex文件(core.odex). 这时候我们需要注意android系统是如何得到这些优化的odex的,通过查看源码,我们可以知道android是用host上的dexopt对jar进行优化的,而host是小端系统,我们的target路由器却是大端系统,因此会造成大小端不兼容,这个问题比较严重.


解决方案就是在路由器上,用大端系统上dexopt命令对jar进行优化,得到大端系统的odex.
但是,由于android源码对大端系统的支持不够,有很多的地方没有考虑到大端系统.

举例, 如下面的, 在转换的时候,忽略掉了一个成员的转换.

<pre class="brush: js;"> 
diff --git a/dalvik/libdex/DexSwapVerify.cpp b/dalvik/libdex/DexSwapVerify.cpp
index 0d028da..e28a52b 100644
--- a/dalvik/libdex/DexSwapVerify.cpp
+++ b/dalvik/libdex/DexSwapVerify.cpp
@@ -910,7 +910,7 @@ static void* swapClassDefItem(const CheckState* state, void* ptr) {
     SWAP_INDEX4_OR_NOINDEX(item->sourceFileIdx, state->pHeader->stringIdsSize);
     SWAP_OFFSET4(item->annotationsOff);
     SWAP_OFFSET4(item->classDataOff);
-
+       SWAP_OFFSET4(item->staticValuesOff);
     return item + 1;
 }
</pre>


##虚拟机指令修改


java虚拟机在执行java程序,在执行前,会对指令进行验证,在验证的时候,有一个地方会出问题,导致了java类验证无法通过.这个问题也是由于大小端的问题引起的.这个问题起源于前面的odex优化.因为在优化dex的时候,对dex文件的指令(一般的都是属于method)的转换过程是这样的.

<pre class="brush: js;"> 
/* Perform byte-swapping and intra-item verification on code_item. */
static void* swapCodeItem(const CheckState* state, void* ptr) {
    DexCode* item = (DexCode*) ptr;
    u2* insns;
    u4 count;

    if ((item->outsSize > 5) &amp;&amp; (item->outsSize > item->registersSize)) {
        /*
         * It's okay for outsSize to be up to five, even if registersSize
         * is smaller, since the short forms of method invocation allow
         * repetition of a register multiple times within a single parameter
         * list. Longer parameter lists, though, need to be represented
         * in-order in the register file.
         */
        LOGE("outsSize (%u) > registersSize (%u)", item->outsSize,
                item->registersSize);
        return NULL;
    }

    count = item->insnsSize;
    insns = item->insns;
    CHECK_LIST_SIZE(insns, count, sizeof(u2));

    while (count--) {
        *insns = SWAP2(*insns);
        insns++;
    }
</pre>

从指令code部分的转换可以看到,对于insns的指令,都是按照两个字节的转换的.但是呢?如果有些指令的解析需要到四个字节的呢,这个时候就会出现问题了,来,我们来看看问题到底在哪里.

我们关注的是fill-array-data指令,这个指令比较特殊,如下

<pre class="brush: js;"> 
fill-array-data vAA, +BBBBBBBB 
(with supplemental data as specified below in "fill-array-data-payload Format")

A: array reference (8 bits)
B: signed "branch" offset to table data pseudo-instruction (32 bits)
</pre>

很明确的,B是四个字节的,但是却被分解为两次进行转化,因此,会导致转化过后出问题.
因此,如下,对这个问题进行了修正.

<pre class="brush: js;"> 
@@ -371,20 +419,36 @@ static bool checkArrayData(const Method* meth, u4 curOffset)
     if ((((u4) arrayData) &amp; 0x03) != 0) {
         LOG_VFY("VFY: unaligned array data table: at %d, data offset %d",
             curOffset, offsetToArrayData);
+		ymy_display(stderr, "DexVerify.cpp checkArrayData %s VFY: unaligned array data table: at %d, data offset %d\n",
+            meth->name, curOffset, offsetToArrayData);
         return false;
     }
-
+	ymy_display(stderr, "content is \n");
+	ptr = (char *) arrayData;
+	for (i = 0; i &lt; 8; i++) {
+		ymy_display(stderr, "%02X   ", ptr[i]);
+	}
+	ymy_display(stderr, "\n");
     valueWidth = arrayData[1];
-    valueCount = *(u4*)(&amp;arrayData[2]);
-
+    //valueCount = *(u4*)(&amp;arrayData[2]);
+    valueCount = arrayData[2];
</pre>



##icudt46b.dat与icudt46l.dat

ICU(International Component for Unicode/Unicode国际化组件) 是 Unicode 支持、软件国际化、全球化的一个成熟的、广泛应用的库.然后android系统也是用了这个库.小端系统上用的是icudt46l.dat,而大端系统上用的是icudt46b.dat.但是android源码在大端系统上没有支持够,因此这里我们又需要自己来对源码进行修改了.


首先,要改一个宏,就是U_IS_BIG_ENDIAN,因为是大端系统,这个值应该为1,这样子,系统就找icudt46b.dat文件了.

<pre class="brush: js;"> 
diff --git a/external/icu4c/common/unicode/platform.h b/external/icu4c/common/unicode/platform.h
index 211de1e..d7a269f 100644
--- a/external/icu4c/common/unicode/platform.h
+++ b/external/icu4c/common/unicode/platform.h
@@ -147,7 +147,7 @@
 #if defined(BYTE_ORDER) &amp;&amp; defined(BIG_ENDIAN)
 #define U_IS_BIG_ENDIAN (BYTE_ORDER == BIG_ENDIAN)
 #else
-#define U_IS_BIG_ENDIAN 0
+#define U_IS_BIG_ENDIAN 1
 #endif
 
 /* 1 or 0 to enable or disable threads.  If undefined, default is: enable threads. */
</pre>


其次,我们需要对icudt46l.dat源文件进行转化.目的在于把小端系统的icudt46l.dat转化为大端系统的icudt46b.dat.转化的命令如下:

<pre class="brush: js;"> 
icupkg -tb icudt46l.dat icudt46b.dat
</pre>










