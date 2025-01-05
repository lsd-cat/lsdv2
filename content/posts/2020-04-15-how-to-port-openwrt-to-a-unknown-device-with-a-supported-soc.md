+++
date = "2020-04-15T00:00:00+00:00"
title = "How to port Openwrt to an unknown device with a supported SoC"
note = "When I wrote this, I had almost zero hardware experience. It was fun to learn via open source development!"
toc = false
+++

Recently, I found some interesting outdoor wireless equipment [offered at a very low price](https://www.alibaba.com/product-detail/CPE46B-300Mbs-2-4Ghz-12dBi-Antenna_1600320162579.html), typically between $10 and $20 on both Aliexpress and Alibaba, often referred to as CPE46B.

These devices are based on 2.4 GHz AR9330 boards, powered via PoE and feature two gigabit ethernet ports, an integrated antenna, and a waterproof enclosure.

[Here’s a YouTube video](https://www.youtube.com/watch?v=i3WUmMOqit0) of someone unboxing and reviewing the device. The video also showcases the OEM web interface.

![Photo of the front view of the device](/posts/2020-04-15-how-to-port-openwrt-to-a-unknown-device-with-a-supported-soc-01.jpg)
![Photo of the internal antenna metal structure](/posts/2020-04-15-how-to-port-openwrt-to-a-unknown-device-with-a-supported-soc-02.jpg)

## Analysis
### PCB

![Photo of the main board from the top](/posts/2020-04-15-how-to-port-openwrt-to-a-unknown-device-with-a-supported-soc-03.jpg)

From the picture, it is clear that the board has an easily accessible serial header and a SOIC8 packaged NAND flash (Winbond 25Q64). Given this information, there are two approaches to start learning about the board via hardware:  
1. Connecting to the serial console to observe whatever the OEM firmware prints out.  
2. Creating a direct hardware image of the flash chip.

### Dumping the firmware

#### Via software

Before attempting to use the SOIC clip or serial port, I decided to investigate the stock firmware. It appears that the device lacks a DHCP server but uses a fixed `192.168.0.1` IP address with default `admin:admin` credentials.  

By default, the device provides only a web interface and a Telnet server on the public interface. The credentials for the Telnet interface are `root` with no password.

{{< highlight bash >}}
CPE46B mips #1 Thu Sep 5 18:02:48 CST 2019 (none)
CPE46B login: root
Ziking logintalk start ...................

Interactive mode

> help
 help                          :Show this usage help
 art.sh                        :Run art server
 get_log                       :Download log from ap to remote.  Usage: get_log [remote ip]
 ifconfig                      :Network configuration commands
 ip                            :Network configuration commands
 iwconfig                      :Wlan configuration commands
 iwpriv                        :Wlan configuration commands
 iwlist                        :Wlan configuration commands
 oem                           :Change/Show MAC address & sn; Usage: oem get/set
 ping                          :Command ping
 ps                            :Command ps
 route                         :Network configuration commands
 sendAT                        :Send AT command for lte device
 show_oem                      :Show OEM infomation
 show_ver                      :Show AP software version
 tc                            :Qos configuration commands
 top                           :Command top
 wlanconfig                    :Athreos wlan configuration commands
 T1                            :Test 5G RF with 20M bandwidth
 T2                            :Test 5G RF with 40M bandwidth
 T3                            :Test 2.4G RF with 20M bandwidth
 T4                            :Test 2.4G RF with 40M bandwidth
 T5                            :Test upload.Usage: T5 [remote ip]
 T6                            :Test download.Usage: T6
>
{{< / highlight >}}

While interacting with the device, the user is initially dropped into a restricted prompt with only a few commands available. However, it is possible to inject commands into almost any of these using common shell command delimiters such as `|`, `;`, and `&`.  

With command injection, it becomes apparent that the device is running a heavily customized OpenWrt fork, based on `Linux 2.6.31`.

Catting `/proc/mtd` return more info about the flash layout.


{{< highlight bash >}}
> iwconfig|cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00010000 00010000 "u-boot"
mtd1: 00010000 00010000 "u-boot-env"
mtd2: 00360000 00010000 "rootfs"
mtd3: 00100000 00010000 "uImage"
mtd4: 00360000 00010000 "rootfs1"
mtd5: 00010000 00010000 "NVRAM"
mtd6: 00010000 00010000 "ART"
{{< / highlight >}}

And `/proc/cpuinfo` about the SoC and the CPU.

{{< highlight bash >}}
> iwconfig|cat /proc/cpuinfo
system type             : Atheros AR9330 (Hornet)
processor               : 0
cpu model               : MIPS 24Kc V7.4
BogoMIPS                : 266.24
wait instruction        : yes
microsecond timers      : yes
tlb_entries             : 16
extra interrupt vector  : yes
hardware watchpoint     : yes, count: 4, address/irw mask: [0x0000, 0x0020, 0x0020, 0x0588]
ASEs implemented        : mips16
shadow register sets    : 1
core                    : 0
VCED exceptions         : not available
VCEI exceptions         : not available
{{< / highlight >}}

By knowing the size of each MTD partition, we can determine that the device has an 8 MiB flash chip. This is consistent with the marking on the chip, `25Q64`, where `64` represents the size in Megabits.

Using `dd`, it is possible to dump each partition, download them, and reassemble the full firmware image with `cat`:

{{< highlight bash >}}
# On the device
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd0
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd1
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd2
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd3
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd4
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd5
> iwconfig|dd if=/dev/mtd0 of=/var/tmp/web/mtd6

# Locally
user@host:# for i in $(seq 0 6); do wget http://192.168.0.1/mtd$i; done

# cat mtd0 mtd1 mtd2 mtd3 mtd4 mtd5 mtd6 > flash.bin
{{< / highlight >}}


Let's now check the size of the resulting dump:

{{< highlight bash >}}
user@host:# ls -lart flash.bin
-rwxrwxrwx 1 user user 8388608 Apr 12 12:40 flash.bin
{{< / highlight >}}

Also, compute a `sha1sum` of the final file. It should match the image extracted later using the SOIC clip.

The size of the flash file is `8388608` bytes, which equals `8192 KiB` (`8388608 / 1024 = 8192`).

#### Quick analysis

When the device boots up, numerous custom scripts and services are executed. The most customized parts of the firmware—such as the web interface and proprietary binaries—are encrypted or obfuscated and loaded into RAM at runtime. At rest, these obfuscated files are named:

- `/usr/web.bin`
- `/usr/sbin.bin`
- `/usr/apps.bin`

The executable responsible for decrypting these files into simpler `.tgz` archives is called `ap_monitor`. Ghidra successfully decompiles this binary, revealing that the obfuscation mechanism is not particularly complex and could be reversed with moderate effort. However, there may not be a compelling reason to do so for most purposes.

Since I was unable to identify the manufacturer from the packaging or elsewhere, I’ll refer to it as `ZiKing`, as this name appears in its proprietary configuration file. On AliExpress, the same device is often attributed to `ANDDEAR`. Neither `ZiKing` nor `ANDDEAR` seems to have any significant presence on the English-speaking internet.

From an internal configuration file:

```
FID="OEM"
FLASH_ID="SPI"
PCB="v1.0"
PN="CPE46B"
PT="AP"
VER="4.3.7"
VER1="4.3.7"
RF_MODE="1T1R"
WAN="0"
EXT_PA="1"
TRSW="1"
SERVER_DOMAIN="www.ziking.net"
DHCPD_EVER="0"
IANA="37260"
MAXNUM=4

####language
CSS_STYLE="SHX46B"
LANG="en"
SUPPORT_LANG="en,zh"
COUNTRYCODE="76"
SUPPORT_COUNTRYCODE="76,156,276,392"
####radio & vaps
MAX_VAPS="8"
MAX_RFS="1"
#0: auto, 1:2.4G, 2-5.8G
RF0_SUPPORT_FREQ="1"
#RF1_SUPPORT_FREQ="0"

SUPPORT_AUTO_ACTIVE="0"
#### product Type
###0: FIT AP mode
###1: WIFI CPE mode
###2: LTE/3G CPE mode
###3: Route mode
####for UPNP
MANUFACTURER="XIAN ZIKING NETWORK COMMUNICATIONS CO.,LTD."
MANUFACTURERURL="http://www.ziking.net"
MODELDESCRIPTION="Wireless Broadband Access Point / CPE"
####
PRODUCT_TYPE="1"
SUPPORT_PRODUCT_TYPE="2,3"
SUPPORT_WAN_MODE="251"
SUPPORT_AUTH_MODE="63"
SUPPORT_WLAN_MODE="7"
SUPPORT_MAC_MAP="0"
PRODUCT_ID="0"
APSYSNEID="SYSNEIDatleast16chars1234567890123456"
AP_NASID="NASIDatleast16chars1234567890123456"
APSYSHOSTNAME="APNAMEatleast40chars1234567890123456789012345678901234567890"
AP_LOCATION="shenzhen"
AP_COVERAGETYPE="2"
AP_DESCRIPRION="Customer Premise Equipment"
AP_SOFT_VERDOR="ZiKing"
AP_ORIG_VENDOR="ZiKing"
AP_CPU="ar9331"
CPU_SPEED="400000000"
#it must be xxMB(type)
AP_MEMORY="64MB(S29GL064M)"
AP_FLASH="8MiBB(HY57V561620TP-H)"
#max power, dbm
AP_MAX_POWER="15"
PCB0="SX933146B"
BUS="AHB"
SUPPORT_AC_CURL_MGR="0"
AP_SERIALNUMBER=001122334455

```

1. The `64 MiB` RAM value specified in the configuration is incorrect. The device actually has `32 MiB`, but this likely varies based on the production batch.   
2. As of now, the website (`www.ziking.net`) displays only a default page, offering no substantial information about the manufacturer or product.
3. The only verifiable piece of information from the configuration file is the [IANA assignment number](https://oidref.com/1.3.6.1.4.1.37260), which is registered to `ZiKing`.

#### Via hardware with a SOIC8 clip
![Photo of a Raspberry PI connected via a SOIC8n clip to the device flash chip](/posts/2020-04-15-how-to-port-openwrt-to-a-unknown-device-with-a-supported-soc-04.jpg)

_WARNING: The Raspberry VCC will fire some power also to the SoC. This means that there might be interference given the unknown state of the SoC while working on the flash chip. In my case, that lead to some verifying errors after a write, but the write itself never failed._

```
    ______
1--| O    |--8
2--|      |--7
3--|      |--6
4--|______|--5

```

Remember to research your chip model and manufacturer and double-check the pin layout using the official datasheet.

| 1  | 2  | 3   | 4   | 5  | 6   | 7     | 8   | Flash pin number |
|----|----|-----|-----|----|-----|-------|-----|------------------|
| CS | DO | /WP | GND | DI | CLK | /HOLD | VCC | Pin name         |
| 24 | 21 | GND | 25  | 19 | 23  | GND   | 17  | Rpi GPIO number  |

Please refer to the multiple flashing guides available
 * https://www.flashrom.org/RaspberryPi
 * https://libreboot.org/docs/install/rpi_setup.html
 * https://karlcordes.com/coreboot-x220/
 * https://tylercipriani.com/blog/2016/11/13/coreboot-on-the-thinkpad-x220-with-a-raspberry-pi/
 * https://github.com/bibanon/Coreboot-ThinkPads/wiki/Hardware-Flashing-with-Raspberry-Pi

From a root prompt on the Raspberry Pi:
```
# flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r flash1.bin
# flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r flash2.bin
# flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r flash3.bin
# sha1sum flash*.bin
```

Check that all the checksums match, including those obtained via `dd` and `cat`. If they don't match, there is likely an issue with the clip positioning or the wiring. Ensure that no pin is left floating, even if it is not required for the operation. Pins such as `/WP` and `/HOLD` should always be connected to either GND or VCC.

A reliable alternative is to use a CH341 USB programmer. In this case, an external VCC source is required.

For more information about In-System Programming, [visit the flashrom wiki](https://www.flashrom.org/ISP).

### Serial Interface
![Photo of the Raspberry Pi connected via GPIO to the device's serial header](/posts/2020-04-15-how-to-port-openwrt-to-a-unknown-device-with-a-supported-soc-05.jpg)

The serial header is easy to work with and has the pinout clearly labeled. Any cheap USB-to-serial adapter should work. In my case, the baud rate is `115200`. However, if you're unsure of the baud rate, a script like [baudrate.py](https://github.com/somu1795/baudrate) can help determine it.

Popular software for serial communication includes `minicom` and `screen`.

Example command to connect:
{{< highlight bash >}}
user@host:# screen /dev/ttyUSB0 115200
{{< / highlight >}}

## Porting
### Partition Layout

The information from `/proc/mtd` is extremely useful for understanding the partition layout:

- **`mtd0 u-boot`**: A 64 KiB partition containing the U-Boot bootloader.  
- **`mtd1 u-boot-env`**: A 64 KiB partition containing the U-Boot configuration.  
- **`mtd2 rootfs`**: A JFFS2 partition containing the actual image.  
- **`mtd3 uImage`**: A SquashFS kernel image.  
- **`mtd4 rootfs1`**: A JFFS2 partition containing a secondary image, probably used for recovery.  
- **`mtd5 NVRAM`**: A 64 KiB partition containing a `tgz` file with OEM system configuration files.  
- **`mtd6 ART`**: A 64 KiB partition [containing calibration data for the radio chip](https://github.com/pepe2k/ar9300_eeprom).  

The total size is 8192 KiB. These partitions are not traditional partitions like those in EXT or NTFS filesystems. Instead, the data is laid out contiguously on the flash chip, with the bootloader and kernel treating different regions as separate partitions. This simplicity is why working with these partitions using `cat` is so straightforward.

For vanilla OpenWrt, a custom configuration partition is not necessary, and two rootfs partitions are redundant. By consolidating everything into a single partition, more space becomes available for packages and user data. The target partition layout could be:

- **`mtd0 u-boot`**: Original image.  
- **`mtd1 u-boot-env`**: Some values here may need modification.  
- **`mtd2 firmware`**: An 8000 KiB OpenWrt partition (`firmware` is the standard OpenWrt naming).  
- **`mtd3 ART`**: Original image.  

On some devices, such changes are unnecessary because their partition layout already makes sense—for example, a single partition containing both the kernel and data. These devices are easier to work with, as they often don't require boot environment modifications. Additionally, building an image for flashing through the OEM web interface might not be feasible in this case.

### U-Boot

U-Boot is an open-source bootloader primarily used for embedded devices. While it is actively developed, the version used in a device depends on the SDK provided by the SoC vendor.

For the Atheros `ar9330`, it seems that version 1.4 was used as the base, which is almost a decade old.

Here's the u-boot log from the serial interface:

{{< highlight bash >}}
U-Boot 1.1.413 (Aug 29 2012 - 10:36:47)

AP121-2MB (ar9330) U-boot
DRAM:  32 MB
flash size 8388608, sector count = 128
Flash:  8 MB
In:    serial
Out:   serial
Err:   serial
Net:
eth0: c8:ee:a6:3f:62:ad
eth0 up
eth1: 00:0a:0b:0c:0d:0e
eth1 up
eth0, eth1
Hit any key to stop autoboot:  0
## Booting image at 9f380000 ...
   Image Name:   Linux Kernel Image
   Created:      2019-09-05  10:02:56 UTC
   Image Type:   MIPS Linux Kernel Image (lzma compressed)
   Data Size:    864262 Bytes = 844 kB
   Load Address: 80002000
   Entry Point:  801d0de0
   Verifying Checksum at 0x9f380040 ...OK
   Uncompressing Kernel Image ... OK
No initrd
## Transferring control to Linux (at address 801d0de0) ...
## Giving linux memsize in bytes, 33554432

Starting kernel ...

Booting AR9330(Hornet)...
init started: BusyBox v1.15.0 (2019-09-05 18:04:35 CST)
starting pid 19, tty '': '/etc/rc.d/rcS'
Isking Copyright 2016....
Not support pin simple config
Not found /etc/rc.d/rc.getethdev
Not found /etc/rc.d/rc.fs
load driver
  adf loaded sucessfully
  asf loaded sucessfully
  ath_hal loaded sucessfully
  ath_rate_atheros loaded sucessfully
  ath_dev loaded sucessfully
  umac loaded sucessfully
2
starting pid 301, tty '': '/bin/getty ttyS0 115200'

CPE46B mips #1 Thu Sep 5 18:02:48 CST 2019 (none)
CPE46B login:
{{< / highlight >}}

This confirms most of what we have learned from the OEM firmware: 8 MiB flash, 32 MiB RAM, and the AR9330 SoC. "Hornet" is the codename for a [specific ALFA board](https://openwrt.org/toh/alfa_network/hornet-ub), and its target configuration was likely recycled for this U-Boot build.

U-Boot provides the ability to drop the user into an interactive command prompt if a key is pressed during the early boot phase:

{{< highlight bash >}}
U-Boot 1.1.413 (Aug 29 2012 - 10:36:47)

AP121-2MB (ar9330) U-boot
DRAM:  32 MB
flash size 8388608, sector count = 128
Flash:  8 MB
In:    serial
Out:   serial
Err:   serial
Net:
eth0: c8:ee:a6:3f:62:ad
eth0 up
eth1: 00:0a:0b:0c:0d:0e
eth1 up
eth0, eth1
Hit any key to stop autoboot:  0
ar7240> help
padfix  - fixed uboot bug
?       - alias for 'help'
bdinfo  - print Board Info structure
boot    - boot default, i.e., run 'bootcmd'
bootd   - boot default, i.e., run 'bootcmd'
bootm   - boot application image from memory
cp      - memory copy
erase   - erase FLASH memory
help    - print online help
loadb   - load binary file over serial line (kermit mode)
loads   - load S-Record file over serial line
loady   - load binary file over serial line (ymodem mode)
md      - memory display
mm      - memory modify (auto-incrementing)
mw      - memory write (fill)
ping    - send ICMP ECHO_REQUEST to network host
printenv- print environment variables
progmac - Set ethernet MAC addresses
protect - enable or disable FLASH write protection
reset   - Perform RESET of the CPU
run     - run commands in an environment variable
saveenv - save environment variables to persistent storage
setenv  - set environment variables
tftpboot- boot image via network using TFTP protocol
version - print monitor version
wd      - check and set watchdog
ar7240>
{{< / highlight >}}

_Note: `ar7240>` is probably there just because someone forgot to edit the prompt to the correct chipset name. It's just a static name and it is irrelevant._

The `printenv` prints out the current u-boot environment:

{{< highlight bash >}}
bootargs0=console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),3456k(rootfs),1024K(uImage),3456k(rootfs1),64k(NVRAM),64k(ART)
bootcmd0=bootm 0x9f380000
bootargs1=console=ttyS0,115200 root=31:04 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),3456k(rootfs),1024K(uImage),3456k(rootfs1),64k(NVRAM),64k(ART)
bootcmd1=bootm 0x9f380000
baudrate=115200
ethaddr=0x00:0xaa:0xbb:0xcc:0xdd:0xee
ethact=eth0
filesize=27d000
fileaddr=80060000
ipaddr=192.168.0.144
serverip=192.168.0.141
bootparam=0
bootdelay=4
runver1=AWS-SX9331027-4.3.3
runver0=OEM-SX933146B-4.3.7
LANG=en
stdin=serial
stdout=serial
stderr=serial
ver=U-Boot 1.1.413 (Aug 29 2012 - 10:36:47)
bootargs=console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),3456k(rootfs),1024K(uImage),3456k(rootfs1),64k(NVRAM),64k(ART)
bootcmd=bootm 0x9f380000

Environment size: 978/65532 bytes
{{< / highlight >}}

The same information can be obtained by running `strings` on the `mtd1` partition image.

As seen above, the `bootargs` variable contains details about the serial console, the partition scheme, and the init process. Therefore, any changes to the partition scheme must also be reflected in this variable.  

Another crucial variable is `bootcmd`, which contains the actual boot command. The address specified in this variable represents the starting address of the partition that contains the kernel image, with the `0x9f`, which is the memory mapped address of the flash chip.  

For example, given the value `0x0038000`, we calculate:  
`3670016 / 1024 = 3584 KiB`.  

By summing up the sizes of `mtd0`, `mtd1`, and `mtd2`, the total should equal the starting address of `mtd3`:  
`64 + 64 + 3456 = 3584 KiB`.  

While we don't have our OpenWrt image yet, we can proceed to write the new variables for the updated partition scheme described in the previous section.

{{< highlight bash >}}
bootargs=console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),8000k(firmware),64k(ART)
bootcmd=bootm 0x9f020000
{{< / highlight >}}

Where, `root=31:02` stands for the `mtd2` partition which is labelled `firmware`. The `bootcmd` address is `0x0002000` which is 128KiB. `8000k` is the OpenWrt partition size.

However, it looks like now the `root` and `mtdparts` parameters are no longer needed since the DTS is built into the image (more on this later). Also, the default init for OpenWrt is `/sbin/init` so that may be omitted too.

The following environment should be sufficient:
{{< highlight bash >}}
bootargs=console=ttyS0,115200 rootfstype=squashfs,jffs2 
bootcmd=bootm 0x9f020000
{{< / highlight >}}

### Modifying the u-boot environment

#### From the u-boot shell
U-Boot includes the commands `setenv` and `saveenv` for managing environment variables:

- **`setenv`**: Changes or adds a variable in the current session.  
- **`saveenv`**: Writes the current environment state to the `u-boot-env` partition.

{{< highlight bash >}}
ar7240> setenv bootargs0
ar7240> setenv bootcmd0
ar7240> setenv bootargs1
ar7240> setenv bootcmd1
ar7240> setenv bootargs console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),8000k(firmware),64k(ART)
ar7240> setenv bootcmd bootm 0x9f020000
ar7240> saveenv
Saving Environment to Flash...
Protect off 9F010000 ... 9F01FFFF
Un-Protecting sectors 1..1 in bank 1
Un-Protected 1 sectors
Erasing Flash...Erase Flash from 0x9f010000 to 0x9f01ffff in Bank # 1
First 0x1 last 0x1 sector size 0x10000                                                                                                                                                                                                      1
Erased 1 sectors
Writing to Flash... write addr: 9f010000
done
Protecting sectors 1..1 in bank 1
Protected 1 sectors
ar7240> printenv
baudrate=115200
ethaddr=0x00:0xaa:0xbb:0xcc:0xdd:0xee
ethact=eth0
filesize=27d000
fileaddr=80060000
ipaddr=192.168.0.144
serverip=192.168.0.141
bootparam=0
bootdelay=4
runver1=AWS-SX9331027-4.3.3
runver0=OEM-SX933146B-4.3.7
LANG=en
stdin=serial
stdout=serial
stderr=serial
ver=U-Boot 1.1.413 (Aug 29 2012 - 10:36:47)
bootargs=console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),8000k(firmware),64k(ART)
bootcmd=bootm 0x9f020000

Environment size: 498/65532 bytes
{{< / highlight >}}

#### Manually building a u-boot-env partition
_Warning: this method did not work for me, albeit it should have. It is eiuther a human error on my side, or something nonstandard the platform u-boot build._

As for the u-boot environment, it is possible to create an entirely new `u-boot-env` data using an utility called `mkenvimage` which is shipped within the u-boot source code distribution.
Just write all the parameters in a file called `env.txt`

{{< highlight bash >}}
baudrate=115200
ethaddr=0x00:0xaa:0xbb:0xcc:0xdd:0xee
stdin=serial
stdout=serial
stderr=serial
ver=U-Boot 1.1.413 (Aug 29 2012 - 10:36:47)
bootargs=console=ttyS0,115200 root=31:02 rootfstype=squashfs,jffs2 init=/bin/init mtdparts=ar7240-nor0:64k(u-boot),64k(u-boot-env),8000k(firmware),64k(ART)
bootcmd=bootm 0x9f020000
ethact=eth0
filesize=27d000
fileaddr=80060000
ipaddr=192.168.0.144
serverip=192.168.0.141
bootparam=0
bootdelay=4
runver1=AWS-SX9331027-4.3.3
runver0=OEM-SX933146B-4.3.7
LANG=en
{{< / highlight >}}

And run `./mkenvimage  -s 0x01000 -o env.bin env.txt`. The output should be a 64KiB env.bin.

### Building OpenWrt

Adding a device target for an already supported SoC shouldn't be a difficult task. The OpenWrt wiki provides helpful documentation for understanding the process:

- [Guide: Adding a New Device](https://openwrt.org/docs/guide-developer/add.new.device)  
- [Guide: Defining Firmware Partitions](https://openwrt.org/docs/guide-developer/defining-firmware-partitions)  

High-level steps to follow:

1. **Find the Target Partition Scheme**: Determine how the flash memory is divided.  
2. **Find LED and Button GPIO**: Identify the GPIOs used for LEDs and buttons.  
3. **Add Board-Specific Files**: Include any additional files required for the board (e.g., network configuration, calibration data, etc.).  

One of the best ways to understand the process is by reviewing recent commits that add support for other devices, particularly those using the same SoC. For reference, I used the [Comfast CF-EW72 target](https://git.openwrt.org/?p=openwrt/openwrt.git;a=commit;h=7daab6286110b95167e291409395494f18edc9dc).

I started by copying a DTS file for the same SoC, specifically `./target/linux/ath79/dts/ar9330_glinet_gl-ar150.dts` from the OpenWrt source tree. I then:

1. Adjusted the partition scheme to match the target device.  
2. Removed GPIO information for LEDs and buttons that I don't have yet.  
3. Verified that the ART partition offsets matched my device.  

Here's the result:


{{< highlight c >}}
// SPDX-License-Identifier: GPL-2.0-or-later OR MIT
/dts-v1/;

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/input/input.h>

#include "ar9330.dtsi"

/ {
        model = "ZiKing CPE46B";
        compatible = "ziking,cpe46b", "qca,ar9330";

        aliases {
                serial0 = &uart;
                label-mac-device = &eth0;
        };

};

&gpio {
        status = "okay";
};

&uart {
        status = "okay";
};


&spi {
        status = "okay";

        num-chipselects = <1>;

        flash@0 {
                compatible = "jedec,spi-nor";
                spi-max-frequency = <50000000>;
                reg = <0>;

                partitions {
                        compatible = "fixed-partitions";
                        #address-cells = <1>;
                        #size-cells = <1>;

                        partition@0 {
                                label = "u-boot";
                                reg = <0x000000 0x010000>;
                                read-only;
                        };

                        partition@1 {
                                label = "u-boot-env";
                                reg = <0x010000 0x010000>;
                        };

                        partition@2 {
                                compatible = "denx,uimage";
                                label = "firmware";
                                reg = <0x020000 0x7d0000>;
                        };

                        art: partition@3 {
                                label = "art";
                                reg = <0x7f0000 0x010000>;
                                read-only;
                        };
                };
        };
};

&eth0 {
        status = "okay";

        mtd-mac-address = <&art 0x0>;
};

&eth1 {
        status = "okay";

        mtd-mac-address = <&art 0x0>;

};

&wmac {
        status = "okay";

        mtd-cal-data = <&art 0x1000>;
        mtd-mac-address = <&art 0x0>;
};
{{< / highlight >}}

Furthermore, I added the `ZiKing CPE46B` target to enable selecting the target image within the OpenWrt build system.

In the file `./target/linux/ath79/image/generic.mk`, add the following entry:

{{< highlight makefile >}}
define Device/ziking_cpe46b
  SOC := ar9330
  DEVICE_VENDOR := ZiKing
  DEVICE_MODEL := CPE46B
  IMAGE_SIZE := 8000k
  SUPPORTED_DEVICES += cpe46b
endef
TARGET_DEVICES += ziking_cpe46b
{{< / highlight >}}

`8000k` is the size of the partition chosen earlier. With this configuration, it should now be possible to select the new target using the standard `make menuconfig` tool and follow the normal OpenWrt build instructions.

After completing the build process, the resulting firmware file should be:
`./bin/targets/ath79/generic/openwrt-ath79-generic-ziking_cpe46b-squashfs-sysupgrade.bin`

### Flashing the new Openwrt image

Technically, there are at least three methods to flash the newly obtained images:

- **Flash from a running system**: Use the `mtd` utility.  
- **Flash from U-Boot**: Transfer the image via TFTP and flash it.  
- **Flash using the SOIC clip**: Directly write the image to the flash chip.  

As of now, I have yet to try the first two methods. This document will be updated as soon as they are tested.

---

### Flash via SOIC

**Warning**: The `du` command does not report exact file sizes; instead, it rounds them based on the sector size in use.  

To flash using `flashrom`, the target image must be exactly **8192 KiB**. Therefore, a full image must be assembled with `NULL` padding (`0x00`) added where necessary.

1. **Dump the new `u-boot-env` partition**: Write the new parameters using `setenv` as described in the U-Boot section, then dump the partition.  
2. **Assemble the final image layout**:  
   - `u-boot`: 64 KiB, the exact same file as dumped (`mtd0`).  
   - `u-boot-env`: 64 KiB, newly dumped with the changes.  
   - `firmware`: 8000 KiB, the OpenWrt image padded with `0x00`.  
   - `ART`: 64 KiB, the exact same file as dumped (`mtd6`).  

{{< highlight bash >}}
user@host:# cat u-boot > flash
user@host:# cat u-boot-env >> flash
user@host:# ls -l openwrt-ath79-generic-ziking_cpe46b-squashfs-sysupgrade.bin
-rwxrwxrwx 1 user user 3998478 Apr 14 01:11 openwrt-ath79-generic-ziking_cpe46b-squashfs-sysupgrade.bin
user@host:# cat openwrt-ath79-generic-ziking_cpe46b-squashfs-sysupgrade.bin >> flash
user@host:# ls -l flash
-rwxrwxrwx 1 user user 4129550 Apr 14 20:53 flash
user@host:# dd if=/dev/zero of=padding bs=1 count=4193522
user@host:# cat padding >> flash
user@host:# cat art >> flash
user@host:# ls -l flash
-rwxrwxrwx 1 user user 8388608 Apr 14 20:54 flash
{{< / highlight >}}

The `dd` command `count` is the result of the following operation: `8000*1024-3998478=4193522`.

And now flash the obtained `flash` binary.

{{< highlight bash >}}
user@host:# flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w flash
flashrom  on Linux 4.19.97+ (armv6l)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
No EEPROM/flash device found.
Note: flashrom can never write if the flash chip isn't found automatically.
root@raspberrypi:/home/pi# flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w flash_new
flashrom  on Linux 4.19.97+ (armv6l)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) on linux_spi.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
{{< / highlight >}}

### Reboot!
Now it is possible to reboot the device and hope everything has worked as expected. While the serial console is still useful in order to see the boot process, the debug messages and eventual problems, OpenWrt should now work normally and it is possible to connect via ethernet and get an IP Address assigned (by default there's `odhcpd` running).

```
BusyBox v1.31.1 () built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt SNAPSHOT, r12948-97c5fb4709
 -----------------------------------------------------
=== WARNING! =====================================
There is no root password defined on this device!
Use the "passwd" command to set up a new password
in order to prevent unauthorized SSH logins.
--------------------------------------------------
root@OpenWrt:/# uname -a
Linux OpenWrt 4.19.108 #0 Mon Apr 13 08:14:48 2020 mips GNU/Linux
```

## Acknowledgements

`#openwrt-devel` on `irc.freenode.net` has been very helpful. The channel is well populated and people try to do their best to answer technical questions.

A special thanks to _Paul Fertser_ (`PaulFertser` on `irc.freenode.net`).