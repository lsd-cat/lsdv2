+++
date = "2020-01-10T00:00:00+00:00"
title = "Nokia-Alcatel FTTH CPE reverse engineering and WPA/admin keygen"
role = "Co-Author"
note = "Almost a decade ago a group of unknown Italian researchers broke the algorithms for two of the local main ISP, Telecom and Fastweb. At the time I was a kid and their work was a great source of inspiration: [Pirelli Fastweb](https://wifiresearchers.wordpress.com/2010/03/25/pirelli-fastweb-free-access/), [Alice AGPF](https://wifiresearchers.wordpress.com/2010/06/02/alice-agpf-lalgoritmo/), [Telsey Fastweb](https://wifiresearchers.wordpress.com/2010/09/09/telsey-fastweb-full-disclosure/). There is also some interesting [work on the topic](https://haxx.in/upc_keys.c) by [blasty](https://twitter.com/@bl4sty). Written in collaboration with [ndaprela](https://twitter.com/ndaprela)."
toc = false
+++

In recent years, the development of Fiber to the Home (FTTH) technology has finally gained momentum in Italy, driven largely by OpenFiber, a state subsidized company responsible for much of the infrastructure work. OpenFiber's network is then made available to numerous partner providers, who resell connectivity to end-users.

This trend mirrors developments in other countries, where the dominant standard for residential optical connections is GPON (Gigabit Passive Optical Network). [Pierre Kim's research provides valuable insights into the GPON standard, its network topology, and key security considerations](https://pierrekim.github.io/blog/2016-11-01-gpon-ftth-networks-insecurity.html).

FTTH requires specialized equipment, leading to the widespread availability of GPON gateways on the market. These devices come in various forms, such as standalone converters (GPON to SFP or GPON to RJ45) or fully integrated Customer Premises Equipment (CPE) units.

Manufacturers like Technicolor, Alcatel-Lucent, Nokia, and Huawei dominate the market for these devices, supplying ISPs worldwide. Notably, many devices from different manufacturers share common components and software stacks. Below is a non-exhaustive list of confirmed and suspected devices with the common vulnerabilities described later in this post. Additionally, some companies, like Zhone, appear to customize these CPEs for specific ISPs, while Eltex might also be involved in similar practices.

| CODE | Country | ISP | Model | SSID Format  |
|---|---|---|---|---|
| MXXT | Mexico | Telmex | [G-240W-B](https://www.nokia.com/networks/products/7368-isam-ont-g-240w-b/) | `INFINITUM%4s_2.4` |
| PTXX | Indonesia | Speedy | Unknown | `speedy@%02x%02x` |
| ALCL | Generic | Unknown | Unknown | `ALHN-%s` |
| SRBJ | Serbia | Unknown | Unknown | `ALHN-%s` |
| ORGS | Generic | Unknown | Unknown | `ALHN-%s` |
| VFSP | Spain | Vodafone | Unknown | `vodafone%4m` |
| HGXX | Generic | Unknown | Unknown | `ALHN-%s` |
| ENTB | Colombia | ETB | [I-240W-A](https://fccid.io/2ADZRI240WA/User-Manual/User-Manual-2679109) | `FibraETB%s` |
| AXTE | Mexico | Axtel | Unknown | `XTEL-XTREMO-%s` |
| IUSA | Mexico | Iusacell | Unknown | `ALU-I240WA-%s` |
| STXX | Saudi Arabia | STC | [G-240W-B](https://www.nokia.com/networks/products/7368-isam-ont-g-240w-b/) | `STC_WiFi_%4s` |
| DUXX | United Arab Emirates | Du Telecom | Unknown | `ALHN-%s` |
| ITPC | Iraq | Unknown | Unknown | `ALHN-%s` |
| KAZA | Kazakhstan | Kazakhtelecom | Unknown | `ALHN-%s` |
| TKRG | Turkey | Turk Telecom | Unknown | `ALHN-%s` |
| ITAX | Italy | Wind | [G-240W-B](https://www.nokia.com/networks/products/7368-isam-ont-g-240w-b/)/[I-240W-Q](https://www.infostrada.it/fileadmin/Materiale/guidemodem/Manuale_modem_fibra_ALU_I-240W-Q_rev_1.2_-_Italiano.pdf) | `ALHN-%s` |
| VIET | Vietnam | Viettel | [G241W-A](https://fccid.io/2ADZRG241W-A/User-Manual/Users-Manual-2688980) | `VIETTEL-%4s` |
| ETIS | United Arab Emirates | Etisalat | [G-240G-C](https://static1.squarespace.com/static/519f78cae4b0cc7b837ad9d2/t/5b339694f950b7fadb98098c/1530107541390/Nokia_7368_ISAM_ONT_G-240G-C_for_Optical_LAN_Data_Sheet_EN.pdf) | `ONT%fs2.4G` |
| ATEB | Saudi Arabia | GO Telecom | Unknown | `GO_WiFi_%4s` |
| TRUE | Thailand | True Internet | Unknown | `true_home2G_%01x%02x` |
| SIGH | Singapore | Singtel | Unknown | `SINGTEL-%4s` |
| VFTK | Turkey | Vodafone Turkey | Unknown | `VodafoneNet` |
| MXXV | Mexico | Telmex | Unknown | `ALHN-%s` |
| SAIB | Mexico | Telmex | Unknown | `ALHN-%s` |
| LATT | Latvia | Lattelekom | Unknown | `ALHN-%s` |
| JPNX | Japan | Unknown | Unknown | `ALHN-%s` |
| LAOS | Laos | Sky Telecom | Unknown | `SKYTEL-%4s` |
| VIVA | Bulgaria | Vivacom | Unknown | `VIVACOM_FiberNet-%4s` |
| PXSF | Belgium | Belgacom | Unknown | Unspecified |
| OCIT | Ivory Coast | Orange Ivory Coast | G-240W-A | `ORANGEFIBER-%4s` |
| Unknown | Caraibes | Canalbox Caraibes | Unknown | `CANALBOX-%4s` |
| Unknown | Poland | Inea | Unknown | `INEA-%4s` |

Different enclosures or slight variants of the same board can be identified by reviewing the [documents published by Nokia for the FCC](https://fccid.io/2ADZR). Here are some notable examples:

- [G-240W-G](https://fccid.io/2ADZRG240WG)
- [G-240W-E](https://fccid.io/2ADZRG240WE)
- [G-241W-A](https://fccid.io/2ADZRG241W-A)
- [G-240W-B](https://fccid.io/2ADZRG240WB)
- [G-240W-A](https://fccid.io/2ADZRG240WA)
- [I-240W-A](https://fccid.io/2ADZRI240WA)
- [G-240W-ZA](https://fccid.io/2ADZRG240WZA)

The models which share the same firmware, or at least part of it, according to some reversed code, are:

| **Category**          | **Models**                                                                                                       |
|------------------------|-----------------------------------------------------------------------------------------------------------------|
| **B-series**           | B0404G, B-0404G-B                                                                                              |
| **F-series**           | F-010G-A, F010GA, F-010G-B, F010GB, F-240W-A, F240WA                                                           |
| **G-series**           | G-010F-A, G-010F-B, G-010G-A, G-040F-A, G-040F-B, G-040G-A, G040GA, G-110G-A, G110GA, G-211M-A                  |
| **G-240-series**       | G-240G-A, G240GA, G-240G-B, G-240G-C, G240GC, G-240G-D, G240GD                                                  |
| **G-240W-series**      | G-240W-A, G-240W-B, G-240W-D, G-240WZ-A, G-241W-A                                                              |
| **G-440-series**       | G-440G-A, G440GA                                                                                               |
| **G-821-series**       | G-821G-B, G821GB, G-821M-A                                                                                     |
| **G-881-series**       | G-881G-A, G881GA, G-881G-B, G881GB                                                                             |
| **GP-series**          | GP-240G-A, GP240GA                                                                                            |
| **HN-series**          | HN-5224-XG                                                                                                    |
| **I-series**           | I-010G, I-010G-C, I-040E, I-120E, I-120E-Q, I-120E-QT, I-120G-B, I-240E, I-240E-Q, I240GB, I-240G-D, I240GD    |
| **I-240-series**       | I-240W, I-240W-A, I240WA, I-240W-B, I240WB, I-240W-Q, I-240W-QT                                                |
| **MCMB-series**        | MCMB-A                                                                                                         |
| **RG-series**          | RG200O-CA, RG201O-CA, RG221O-CA                                                                               |
| **SRNT-series**        | SRNT-A                                                                                                        |
| **TW-series**          | TW-080GX-A, TW-240GX-A                                                                                        |


## Hardware
The board has 128MB flash and 256MB RAM. An additional SPI Flash of 16MB is soldered on the back of the board.

`/proc/mtd`
```
mtd0: 00080000 00020000 "cferom"
mtd1: 00080000 00020000 "nvram"
mtd2: 00c00000 00020000 "cfg"
mtd3: 00680000 00020000 "jff2_0"
mtd4: 00680000 00020000 "jff2_1"
mtd5: 02600000 00020000 "rootfs0"
mtd6: 02600000 00020000 "rootfs1"
mtd7: 01a00000 00020000 "log"
mtd8: 0237b000 0001f000 "0"
mtd9: 0000000c 0001f000 "0"
mtd10: 00a0d000 0001f000 "1"
mtd11: 01702000 0001f000 "0"
mtd12: 0237b000 0001f000 "0"
```

It boots with the CFE bootloader, which is [described here](https://raw.githubusercontent.com/Noltari/cfe_bcm63xx/master/bcm963xx_bootloader_appnote.pdf).

`/proc/cpuinfo`

```
system type             : 968380GERG
processor               : 0
cpu model               : Broadcom BMIPS4350 V8.0
BogoMIPS                : 598.01
wait instruction        : yes
microsecond timers      : yes
tlb_entries             : 32
extra interrupt vector  : no
hardware watchpoint     : no
ASEs implemented        :
shadow register sets    : 1
kscratch registers      : 0
core                    : 0
VCED exceptions         : not available
VCEI exceptions         : not available
```

The main SoC and the GPON module are made by Broadcom. Both 2.4GHz and 5GHz Wi-Fi are supported on this device, and the hardware appears to be made by Quantenna. The Wi-Fi functionality seems to be powered by the Topaz chipset, [which was recently added to the Linux Kernel](https://patchwork.kernel.org/patch/10630331/). However, there is a lack of precompiled binaries in `linux-firmware`, as noted in [this interesting discussion](https://patchwork.kernel.org/patch/10643357/). Additional relevant code might also be available in the [Google Fiber repository](https://gfiber.googlesource.com/kernel/skids/+/master/drivers/topaz).

## Software
There are already known backdoors in similar devices, as [detailed in this article](https://www.websec.ca/publication/Blog/backdoors-in-Zhone-GPON-2520-and-Alcatel-Lucent-I240Q)). While my device does not have the SSH service exposed, it does have a Telnet service running on port 23. Gaining root access is trivial—simply log in with the credentials ONTUSER:SUGAR2A041.

Once logged in, it becomes clear that the manufacturer has committed all device-specific customizations directly onto the firmware. This results in numerous files that are irrelevant to this specific model, including a dozen different web interfaces and configuration files for ISPs listed in the table above. Additionally, the firmware contains a significant amount of debugging artifacts, such as this one:

{{< highlight bash >}}
#!/bin/sh

#add debug ONTUSER and config ONTUSER env, add by xufuguo, 20120808
#Note: don't chang PS1, bob calibration software need terminal prefix just "#"
#update for R30 read-only rootfs, modify by xufuguo, 20130816

exist=no
preconfig=no
home_dir=/configs/home

while read LINE
do
        temp=$(echo $LINE | awk -F : '{print $1}')
        if [ "${temp}" != "ONTUSER" ]; then
                continue
        fi
        exist=yes
        break
done < /etc/passwd

# BRCM use confignew.cfg
if [ ! -f /configs/config.cfg -a  ! -f /configs/confignew.cfg ]; then
        preconfig=yes
fi

if [ "${exist}" = "no" -o "${preconfig}" = "yes" ]; then
        rm -f /configs/etc/*+ #clean up temp files
        [ -d ${home_dir} ] || mkdir -p ${home_dir}
        sed -i -e '/ONTUSER/'d /configs/etc/passwd #del username,passwd,config file
        sed -i -e '/ONTUSER/'d /configs/etc/shadow
        rm -rf ${home_dir}/ONTUSER
        op_id=$(ritool get OperatorID | awk -F : '{print $2}')
        if [ "${op_id}" = "0000" -o ! -f /usr/sbin/vtysh ];then #factory mode
                adduser -D -h ${home_dir}/ONTUSER -G wheel ONTUSER
        else
                adduser -D -h ${home_dir}/ONTUSER -G wheel ONTUSER -s /usr/sbin/vtysh
        fi
        echo "ONTUSER:SUGAR2A041" | chpasswd -m > /dev/null 2>&1
        echo "export PS1=\"[\\u@\\h: \\W]\\\\\$ \"" >> ${home_dir}/ONTUSER/.bashrc

        #chang uid to 0 for get root authorization
        source /bcm/script/rootmod.sh ONTUSER

        sync
fi
{{< / highlight >}}

That's how bad it is, and [Wind experienced it firsthand in 2017](https://www.macitynet.it/la-rete-fibra-wind-bloccata-un-malware-mette-uso-modem/) when a faulty update exposed the Telnet port to the internet. This allowed the infamous BrickerBot malware to render almost 20,000 CPEs into total electronic garbage (or at least, more so than they were before). Wind had to freely ship a new modem to every affected customer, causing multiple days of total downtime.

### Configuration Tools

There are two useful configuration tools available: `cfgcli` and `ritool`.

`cfgcli` is used to configure various properties of the device in an XML-like structure. It can retrieve information such as VOIP settings, PPP credentials, TR69 configurations, SLID, and much more.

A simple `cfgcli dump` command will output all properties.

Apparently, all devices and boards share the same flash content, except for certain specific data, such as the serial number, operator ID, and similar identifiers. These OEM binaries manage this data through a library called `/lib/libri.so`, which interacts with a device at `/dev/ri_drv`. 

This device is controlled by a proprietary Broadcom kernel module located at `/bcm/bin/ri.ko`. Essentially, this module handles reading and writing values to an I2C EEPROM.

The command `ritool dump` returns all the configured variables, especially useful to conigure a different CPE with the same FTTH line:

{{< highlight text >}}
the Format:01
the MfrID:ALCL
the Factorycode:02
the HardwareVersion:3FE56756AABB
the ICS:01
the YPSerialNum:        B133F2B0
the CleiCode:0000000000
the Mnemonic:G-240W-B
the ProgDate:170720
the MACAddress:f8:44:e3:d6:63:f0
the DeviceIDPref:3030
the SWImage:3030
the OnuMode:0003
the Mnemonic2:
the Password:30303030303030303030
the G984Serial:b133f2b0
the HWConfiguration:3030303030303030
the PartNumber:3FE56756BAAA
the Variant:AA
the Spare4:303030303030303030303030
the Checksum:c9e2
the InserviceReg:3030
the UserName:       userAdmin
the UserPassword:00000000
the MgntUserName:      adminadmin
the MgntUserPassword: ALC#FGU
the SSID-1Name:0000000000000000
the SSID-1Password:00000000
the SSID-2Name:0000000000000000
the SSID-2Password:00000000
the OperatorID:ITAX
the SLID:30303030303030303030303030303030
the CountryID:eu
the Spare5:303030303030
the Checksum1:01e4
the Spare6:3030
the RollbackFlag:ffff
ProductClass[G-240W-B] Platform[29] Uplink[GPON] Type[HGU] EthPorts[4] EnetPortType[GE] POTS[2] USB[2] WIFI[2] SIM[0] IsBosa[4] SlicType[LE9540] WifiType[BCM4331QTN11AC] JDM[T&W] SOC[BRCM_68380]
{{< / highlight >}}

The most important data fields are:
- **G984Serial**: The serial value used across the system.
- **OperatorID**: Defines the branding of the modem.
- **PartNumber**: Specifies the actual board model.

For example, using the command:

{{< highlight bash >}}
ritool set OperatorID AXTEL
{{< / highlight >}}

It’s possible to brand the modem for Mexico’s Axtel. To apply the changes, a factory reset is necessary. This can be done by pressing the shortest button on the back of the modem for at least 10 seconds while it is powered on.

### Password generation

![Back of the CPE including labels with serials and passwords](/posts/2020-06-01-nokia-alcatel-ftth-gpon-g-240-w-reverse-engineering-wpa-keygen-01.jpg)

While it is impossible to know in advance, most of the time, this type of device generates a unique password using a hardcoded algorithm. It is extremely rare for manufacturers to manually insert secrets into every device. The automatic generation solution can be secure if the algorithm is seeded with data unknown to potential attackers.

However, there are bad examples, such as passwords generated based on the MAC address. Passwords derived from serial numbers are often better, as an attacker typically cannot know the serial and it isn’t derived from other public data. In this case, however, the serial is partially predictable and partially exposed to attackers due to its inclusion in the SSID.

Additionally, note that a WPA key comprised only of ten digits can be cracked in under 10 hours using [Hashcat](https://hashcat.net/hashcat/) and a GTX 1070 GPU.

[An interesting file can be found at `/etc/preconfiguration_global.xml`](https://git.lsd.cat/g/nokia-keygen/raw/master/xml/preconfiguration_global.xml). This file contains the automatic configuration instructions for every ISP supported.

{{< highlight xml >}}
<WLAN_1. n="WLAN_1" a="static">
        <Enable t="string" ml="12" v="true" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.Enable"/>
        <MACAddressControlEnabled t="string" ml="12" v="false" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.MACAddressControlEnabled"/>
        <Standard t="string" ml="12" v="b,g,n" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.Standard"/>
        <ssidName t="string" ml="64" v="" fri="alhn_genssid1" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.SSID"/>
        <Powerlevel t="string" ml="4" v="100" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.TransmitPower"/>
        <BeaconType t="string" ml="4" v="WPAand11i" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.BeaconType"/>
        <WPSEnable t="string" ml="12" v="1" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.WPSEnable"/>
        <PreSharedKey t="string" ml="64" v="" fri="genpwd_longpasswd" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.PreSharedKey.1.PreSharedKey"/>
</WLAN_1.>
{{< / highlight >}}

By examining other files and binaries, it becomes clear that parsing the XML performs the following actions:

- For every ISP, there is a root element with its corresponding code. For example, `ITAX` represents Wind Italy.
- Each nested element within the ISP root is a configuration section. Useful information includes:
  - TR-69 URLs and credentials
  - PPP credentials
  - Wi-Fi SSID and Key
- A single line in the XML can either contain a static value or a string that maps to a specific function in a configuration binary:
  - `fri` specifies the function.
  - `v` provides an optional argument.

[The binary responsible for processing this file](https://git.lsd.cat/g/nokia-keygen/raw/master/bin/cfgmgr) is `/sbin/cfgmgr`, which starts at boot and remains active during normal device operation.

### Reversing
By reversing this binary and using `ritool` for dynamic testing, it becomes relatively straightforward to understand how the system operates.

Essentially, the various fields in the XML file can either be hardcoded or dynamically generated using runtime code and configurations. In many cases, there is a format string that defines a template, along with another attribute specifying how to process or populate the string. Here’s an example snippet illustrating this scheme:
{{< highlight xml >}}
<ssidName t="string" ml="64" v="ALHN-%4s-11ac-4" fri="genssidfmt" dburi="SSID"/>
{{< / highlight >}}

This mechanism is not limited to WLAN passwords—it can be applied more broadly across the system. By inspecting the XML, we can identify a limited set of _format strings_ and only a few values for the attribute `fri`. Here’s another example:
{{< highlight xml >}}
<ssidName t="string" ml="64" v="" fri="alhn_genssid1" dburi="InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.SSID"/>
{{< / highlight >}}

In this case, by examining the corresponding C code within `cfgmgr`, we can identify how the _format string_ is actually constructed and shaped.
{{< highlight c >}}
snprintf(ssidName,0x40,"ALHN-%s",__n + 0x74c4ee);
{{< / highlight >}}

We leave it to the reader to fully reverse the XML and the binary for a complete understanding of how everything works. (BONUS: during this process, you’ll likely uncover numerous hardcoded credentials in cleartext!)

For our purposes, there are just a few key points to clarify. First, these are not traditional format strings. By inspecting the code and performing tests with `ritool`, it becomes clear that the most commonly used "format strings" have the following meanings:

- **`%4s`**: The lower 2 bytes of the serial code, represented in hex.
- **`%4m`**: The lower 2 bytes of the MAC address, represented in hex.
- **`%8m`**: The lower 4 bytes of the MAC address, represented in hex.

Our focus is on identifying cases where WLAN passwords can be efficiently cracked using publicly available information, such as the SSID and MAC address. An immediate example that requires no further reversing is when the WLAN configuration in the XML directly uses the lower bytes of the MAC address as the password:

{{< highlight xml >}}
<PreSharedKey t="string" ml="64" v="%8m" fri="genssidfmt" dburi="PreSharedKey.1.PreSharedKey"/>
{{< / highlight >}}

This behavior is observed in 3 WLANs described within the XML.

Another case that we will publicly present occurs when the SSID contains a portion of the serial code, and the password is generated using the `gen_mgntlongpw_and_wanpw` function. Specifically, this applies to WLANs that:

1. Use a format string containing `%4s` combined with `genssidfmt` for SSID generation and `gen_mgntlongpw_and_wanpw` for password generation.
2. Have `fri=alhn_genssid1` for the SSID and use `gen_mgntlongpw_and_wanpw` for password generation.

Below is the decompiled code retrieved using Ghidra, with added manual analysis hints and variable renaming for clarity:

{{< highlight c >}}
undefined4 gen_mgntlongpw_and_wanpw(undefined4 oui,char *serialnum_string,char *longpasswd_dst,char *wanpasswd_dst) {
  undefined8 longpasswd;
  int i;
  uint n2;
  uint n1;
  char secret1 [32];
  char secret2 [32];
  undefined4 serialnum_int;
  char md5_secret1 [16];
  char md5_secret2 [16];
  char strToBase32Encode [19];
  undefined wanpasswd [32];
  
  if (enter != 1) {
    log_log(2,"gen_mgntlongpw_and_wanpw",0x112,"enter\n");
    sscanf(serialnum_string,"%x",&serialnum_int);
    sprintf(secret1,"%s-ALCL%s",oui,serialnum_string);
    sprintf(secret2,"%s-01%u",oui,serialnum_int);
    md5(secret1,md5_secret1);
    md5(secret2,md5_secret2);
    strToBase32Encode._0_4_ = md5_secret1._0_4_;
    strToBase32Encode._4_4_ = md5_secret1._4_4_;
    strToBase32Encode._8_4_ = md5_secret1._8_4_;
    strToBase32Encode._12_4_ = md5_secret1._12_4_;
    strToBase32Encode[16] = md5_secret2[0];
    strToBase32Encode[17] = md5_secret2[1];
    strToBase32Encode[18] = md5_secret2[2];
    base32_probably(strToBase32Encode,0x13,wanpasswd,0x20);
    snprintf(wanpasswd_dst,0x1f,"%s",wanpasswd);
    n1 = 0;
    n2 = 0;
    i = 0;
    while (i < 8) {
      n2 = n1 >> 0x18 | n2 << 8;
      n1 = n1 << 8 | (uint)(byte)md5_secret2[i + 8];
      i = i + 1;
    }
    longpasswd = __umoddi3(n2,n1,2,0x540be400);
    snprintf(longpasswd_dst,0xb,"%010llu");
    log_log(2,"gen_mgntlongpw_and_wanpw",0x134,"succeed!\n",longpasswd);
    enter = 1;
  }
  return 0;
}
{{< / highlight >}}

The actual inputs to the function are only the OUI (available in the XML and also easily brute-forceable) and the serial number. The other two parameters are destinations for storing the generated passwords.

The code follows a straightforward sequence:
1. **Hash Generation**: 
   - Two strings are created using the inputs.
   - MD5 hashes are computed for each string.
2. **First Password**:
   - The first MD5 hash is concatenated with the first 3 bytes of the second hash.
   - The concatenated result is encoded in base32.
   - The base32-encoded result is truncated to create the first password.
3. **Second Password**:
   - Two numbers are derived from the bytes of the second MD5 hash.
   - These numbers are used as the lower and upper bits of a 64-bit integer.
   - The second password is generated by computing this integer modulus `0x20x540be400`.

This process shows the simplicity and predictability of the password generation, especially given that key inputs like the OUI and serial number are either public or easily derived.

### Keygen
With this information, we can create a proof-of-concept (PoC) script to generate a wordlist for efficiently cracking a vulnerable SSID. [It is available here](https://git.lsd.cat/g/nokia-keygen/src/branch/master/poc/keygen.py).

We can quickly check this script against a test network. SSID=INFINITUMF2B0_2.4-3 and PASSWORD=3345321247.

{{< highlight bash >}}
user@host#: ./keygen.py INFINITUMF2B0_2.4-3 | grep 3345321247
3345321247
{{< / highlight >}}

Extending this keygen to handle all the possible vulnerable SSIDs, and the WLANs which use hardcoded password is pretty trivial.

### Unlocking
Before making any changes, make sure to back up your configuration to avoid breaking your internet connection or losing important credentials such as SLID or VOIP information. Use the following commands:
- `cfgcli dump`
- `ritool dump`

If you’re looking to unlock and use a branded CPE with a different GPON ISP, follow these steps:

1. **Enable Telnet or SSH**:
   - If neither Telnet nor SSH is available, try enabling one of them via the web interface under the `Access Control` page.
   - Admin credentials can be found in [preconfiguration_global.xml](https://git.lsd.cat/g/nokia-keygen/raw/master/xml/preconfiguration_global.xml).

2. **Login**:
   - Use Telnet or SSH to log in with the credentials `ONTUSER:SUGAR2A041`.

3. **Set Operator ID**:
   - Run the following command to set the operator ID:
     ```bash
     ritool set OperatorID ALCL
     ```

4. **Factory reset**:
   - Press the reset button on the device for 10 seconds and wait for it to reboot.

5. **Reconnect**:
   - After the reset, connect to the router either via cable or using the preconfigured Wi-Fi password.
   - The SSID will be `ALHN-%4s`.

6. **Router access**:
   - The router’s IP address will be `192.168.1.254`.

7. **Web panel and backdoor**:
   - The `ONTUSER` backdoor will still be active.
   - Web panel credentials:
     - Admin user: `AdminGPON:ALC#FGU`
     - Lower-privileged user: `userAdmin` (password matches the Wi-Fi password).