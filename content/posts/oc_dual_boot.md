---
title: "关于OpenCore引导双系统的一些总结和讨论"
date: 2019-10-25T04:14:45-04:00
draft: false
toc: true
tags:
 - hackintosh
 - macOS
 - OpenCore
 - OC
 - Windows
---

最近OpenCore越发火热，越来越多人想要换用OC引导来得到更为原生的macOS体验。而使用OC引导，其对ACPI表的修改将会传递给操作系统，有些人就会发现OC引导的Windows会出现无法启动、容易崩溃、激活失效等等问题。其实，在配置OpenCore的时候，我们只要做好以下几个注意事项，这些问题都可以避免，在享受更原生macOS的同时，也能比较优雅地切换Windows系统。

#### 一. 谨慎使用ACPI Patch的改名
1. 不要改动_PRW _DSM和_OSI，前两个可能使硬件在Windows下工作不正常，而对_OSI的改动会使得Windows启动时蓝屏。另外，如KBD、GFX0等等，只要不是必须的，都尽量不去改名。[^1][^2]
2. 在ACPI Patch中所有改动了DSDT表中原始名称的，都应有相应的SSDT-xxx.aml重新实现原有名字的设备或函数，具体的就是一下第二部分的内容了。

#### 二.自定义SSDT中应灵活应用`If (_OSI= (“Darwin”))`判断语句，使得改变后的功能只对macOS生效
1. 改名后重新实现的设备或函数，都要在内容里添加系统判断：如果是macOS，则使用改过的内容；如果不是macOS，则调用改名后的方法。
举个例子，很多笔记本会通过修改_Qxx函数的内容来实现在macOS下的亮度调节热键。这里以改动_Q15为例，首先在config.plist中会有一项改名：

	Comment             | Find     | Replace
	:-------------------|:---------|:-------
	Change _Q15 to XQ15 | 5f513135 | 588513135

	同时会有一个SSDT-key.aml重新写了一个_Q15函数，用以实现调节亮度的功能：
	```aml
	Method (_Q15, 0, NotSerialized)  // _Qxx: EC Query
	{
		Notify (\_SB.PCI0.LPCB.KBD, 0x0405)
	}
	```
	这就是说，原本的_Q15函数被改名为XQ15了，所有调用_Q15函数的都执行SSDT-key中定义的的新命令。
	但此时通过OC引导重启到Windows的话，会发现这个热键失效了，因为Windows中原本要执行的命令现在是在XQ15函数中了。所以，我们修改SSDT-key.aml，判断系统为macOS时，才执行新的命令，否则调用XQ15，使得这个热键在macOS和Windows下都按我们预想的方式工作：
	```aml
	Method (_Q15, 0, NotSerialized)  // _Qxx: EC Query
	{
		If (_OSI ("Darwin"))
		{
			Notify (\_SB.PCI0.LPCB.KBD, 0x0405)
		}
		\_SB.PCI0.LPCB.EC.XQ15 ()
	}
	```

2. SSDT中新添加的设备，应定义其_STA函数，用于判断它是否工作。
比如，很多笔记本会添加一个ALS0虚拟环境光传感器，用于保存屏幕亮度，SSDT-ALS0.aml：
```aml
Device (_SB.ALS0)
{
        Name (_HID, "ACPI0008")  // _HID: Hardware ID
        Name (_CID, "smc-als")  // _CID: Compatible ID
        Name (_ALI, 0x012C)  // _ALI: Ambient Light Illuminance
        Name (_ALR, Package (0x01)  // _ALR: Ambient Light Response
        {
            Package (0x02)
            {
                0x64, 
                0x012C
            }
        })
}
```
此时重启到Windows，你会发现设备管理器中也多出了一个传感器设备。这东西本来不应该出现在Windows中的，所以我们给ALS0设备定义_STA函数：
```aml
Device (_SB.ALS0)
{
        Name (_HID, "ACPI0008")  // _HID: Hardware ID
        Name (_CID, "smc-als")  // _CID: Compatible ID
        Name (_ALI, 0x012C)  // _ALI: Ambient Light Illuminance
        Name (_ALR, Package (0x01)  // _ALR: Ambient Light Response
        {
            Package (0x02)
            {
                0x64, 
                0x012C
            }
        })
        Method (_STA, 0, NotSerialized)  // _STA: Status
        {
            If (_OSI ("Darwin"))
            {
                Return (0x0F)
            }
            Else
            {
                Return (Zero)
            }
        }
}
```
这样，ACPI只会告诉macOS系统有这么一个设备，而其他操作系统中完全找不到它的存在。

#### 三. 在OpenCore配置使用主板的UUID作为SystemUUID，以免破坏Windows的激活环境。[^1]
一般主板的UUID可以在BIOS中查看，如果BIOS中看不到，我们可以通过传统方式启动Windows（不能是OC引导），在命令行中查看。
打开cmd，输入wmic命令并回车，会看到提示符变成了`wmic:root\cli>`，再输入命令`csproduct list full`，得到内容如下：
```cmd
C:\Users\Aten>wmic
wmic:root\cli>csproduct list full

Description=Computer System Product
IdentifyingNumber=Default string
Name=MS-7B23
SKUNumber=
UUID=00000000-0000-0000-0000-309C23AE5B3E
Vendor=Micro-Star International Co., Ltd.
Version=1.0
```
将这个UUID填入config.plist中的 Platforminfo->Generic->SystemUUID 即可。以上Windows中的查看方式也可以在配置好OpenCore引导后作为验证。

#### 四. 如果Windows安装在另外一块硬盘，确保该硬盘的EFI分区是第一个分区。[^1]
有些电脑出厂时硬盘的第一个分区是恢复分区，或者用别的引导工具安装Windows后在该硬盘没有建立ESP分区，这会导致OpenCore找不到Windows系统。此时只要调整硬盘分区，新建或移动ESP到该硬盘的第一分区即可，推荐在winPE下用DiskGenius操作。

#### 总结
就我看来，只要做到以上几点，OpenCore引导的双系统就比较有舒服的体验了。但是这些东西可能对于已经配置好了clover，想要转用OpenCore的人比较有帮助，而那些从头开始配置OpenCore的同志们，直接从[小兵的oc-little](https://github.com/daliansky/OC-little)开始入手可能会省事一点。

#### 扩展内容：
1. 关于_OSI函数：
  _OSI函数是用于询问操作系统名称的，因为硬件厂商可能会在ACPI表中规定一些功能只在某些操作系统实现，以回避bug之类，而同时，操作系统也可以伪造回答，以规避bug或实现更多功能。

2. 关于_STA函数：
  _STA用于描述设备的状态。ACPI表中的每个设备，在初始化之前，都会先执行_STA函数，去检测这个设备的状态，如果设备存在才初始化。如果没有显式定义_STA函数，这个设备则被默认是存在且能正常工作的。描述设备的状态主要靠_STA的返回值，它的返回值有32位，但目前只有末5位有定义[^3]：

	|位数 |31:5 | 4 | 3 | 2 | 1 | 0 |
	|:--|:--:|:--:|:--:|:--:|:--:|:--:|
	|定义|未使用|是否有电池|是否正常工作|是否在UI中显示|是否激活并解码资源|是否存在|

	对以上定义，“是”则置该位为1，“否”则置为0。如果一个设备，没有电池，可以正常工作并在系统中显示，那它的_STA返回值就是二进制的01111，换成16进制就是0x0F，如果一个设备不存在，那返回值就是00000，即16进制0x00。

知道了_OSI和_STA的意义，我们便可以理解上面的自定义的_STA函数了。ACPI首先询问操作系统，你是不是Darwin内核，系统回答是，那么ACPI就得到_STA为0x0F，开始正常初始化设备；如果系统回答不是，那么_STA返回0x00，ACPI就会忽略这个设备，之后系统中也看不到这个设备了。
```aml
Method (_STA, 0, NotSerialized)  // _STA: Status
{
            If (_OSI ("Darwin"))
            {
                Return (0x0F)
            }
            Else
            {
                Return (Zero)
            }
}
```

[^1]: OpenCore-0.5.2 Configuration.pdf
[^2]: [精解OpenCore-黑果小兵](https://blog.daliansky.net/OpenCore-BootLoader.html)
[^3]: ACPI规范

