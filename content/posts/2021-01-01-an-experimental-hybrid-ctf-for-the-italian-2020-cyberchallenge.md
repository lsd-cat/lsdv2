+++
date = "2021-01-01T00:00:00+00:00"
title = "An experimental hybrid Capture-the-Flag challenge for an A/D competition"
note = "The work described here was commissioned and paid for by [Consorzio CINI](https://www.consorzio-cini.it/) to be used as the final challenge for their 2020 [Cyberchallenge](https://cyberchallenge.it) finals. For reasons incomprehensible to the author, I had to keep it confidential until my NDA expired, which happened finally in 2024. The project was developed primarily in September 2020, and the competition was held on October 1, 2020. The post was written in 2021, and finaly published in 2025."
toc = false
+++

CyberChallenge.IT is the free, Italian, state-sponsored cybersecurity training program aimed at high school and university students. As of 2020, it has been organized annually since 2017. Following a selection process based primarily on logical and programming skills, students attend classes on specific cybersecurity topics.

After completing the course, competitions are held in local universities, and the best-ranked teams gain access to the final event, which usually takes place in the summer. Since no prior cybersecurity skills are required—although they are, of course, an advantage—the challenges should not be excessively complex or deviate significantly from the program's content.

The teams competing in the final event are composed of a limited number of players, typically four, and have a fixed time of 8 hours to solve as many challenges as possible.

## Hardware

The hardware was provided by a technical sponsor, [Tiesse Spa](https://tiesse.com), a company that manufactures embedded devices for networking. Specifically, the hardware supplied, named **Tiesse TGR**, has the following specifications:

- _NXP i.MX8M SoC_
- _2 GB DDR3 RAM_
- _8 GB eMMC_
- _2 10/100/1000 RJ45 ports_
- _Qualcomm Atheros QCA6174 Wi-Fi Chipset_

The _Qualcomm wireless chipset can operate in both **Station** and **Access Point** modes simultaneously. This allows it to expose a wireless network while connecting to a second one at the same time. 

![Tiesse TGR rear with stickers.](/posts/2025-01-01-an-experimental-hybrid-ctf-for-the-italian-2020-cyberchallenge-01.jpg)

## Planning

### Requirements

Due to the structure of the CyberChallenge.IT final competition, every challenge must be structured in an **Attack/Defense** format. 

Each team is assigned what are defined as **services**. Each service contains one or more challenges that other teams must exploit to collect flags. The team that owns a service, by understanding the vulnerabilities present in it, should be able to fix them by writing and applying a patch. Additionally, each service is monitored for **uptime**, meaning that it must always be operational; otherwise, a penalty is applied.

The following requirements should be considered when developing an Attack/Defense challenge:

- **Patching a service must preserve its core functionalities.**
- **Patching a service must require at least a partial understanding of the vulnerabilities involved.**
- **A team must be able to patch only its own service.**

The last requirement is both extremely important and the hardest to achieve: if, through an attack chain, a team can interfere with the proper functioning of another team's services, it may completely disrupt the game. While the challenge must inherently contain vulnerabilities, the introduction of unintended vulnerabilities or unforeseen attack chains could result in an unfair game, potentially making it impossible to detect or resolve issues.

Additionally, even without the constraints imposed by the ongoing pandemic, no **hardware tinkering** must be required. Participating players are not trained in hardware operations and lack the tools necessary for tasks like flash dumping or UART extraction.

### Challenge Outline

The primary goal is to implement simple vulnerabilities that mimic real-life scenarios, providing participants with a positive learning experience. The challenge should help players train real-world skills, including the ability to chain minor vulnerabilities into a more serious attack chain.

The four main steps to develop the attack chain are:

1. **Predictable password generation**: To gain an initial point of entry.
2. **Command injection**: To achieve command execution.
3. **Elevation of Privilege via a race condition**: To patch and gain higher-level access.
4. **Lateral movement**: To attack other targets.


## Implementation
Originally, the competition was intended to be held on-site. However, due to the pandemic, a remote solution was required. 

In the new setup, each team is assigned a device that is accessible on **port 80** (HTTP) and **port 22** (SSH) through the game VPN. Any network traffic between devices and teams is strictly prohibited by firewall rules.

![Table setup with the switch and all the devices.](/posts/2025-01-01-an-experimental-hybrid-ctf-for-the-italian-2020-cyberchallenge-02.jpg)

### Environment
Since the product is based on **U-Boot** and **Linux**, which are both GPL-licensed, the supplier has provided patches to the projects and building instructions. For the purposes of this project, everything has been integrated into **Buildroot** and the entire firmware building process has been automated.

The only configuration required on a per-device basis has been automated via **SSH access**. This configuration involves modifying the DHCP settings of the Access Point (AP) to ensure that devices acting as both Clients and APs do not have overlapping addresses. 

During the competition, SSH access also proved invaluable for **rotating flags**, **monitoring**, **debugging**, and rolling out minor fixes to challenges as needed.

## Vulnerabilities
### Predictable password generation
Hardcoded or predictable passwords are a common pattern across low-end embedded devices. These passwords are often based on parameters that vary even between identical devices, such as MAC addresses, serial numbers, software revisions, and production batches. These inputs serve as seeds for key generation algorithms.

For this challenge, a similar behavior is emulated with a custom **C binary**, integrated into the platform to run at boot. The binary performs the following tasks:

1. **Verify the target device**: Checks if the binary is running on the target device by performing a basic checksum on the `kernel.release` property.
2. **Generate `/etc/serial`**: If `/etc/serial` does not exist, generates a random 32-character hexadecimal string and creates the file.
3. **Generate `/etc/update_key`**: If `/etc/update_key` does not exist, generates a random 64-character hexadecimal string and creates the file.
4. **Read the MAC address**: Reads the access point interface's MAC address from `/sys/class/net/ap0/address`.
5. **Configure the SSID**: If a placeholder is found, writes the SSID to `hostapd.conf` in the format `CyberChallenge-%4s`, using the serial number as a parameter.
6. **Generate WPA key**: If `/etc/wpa` does not exist, runs the `gen_key` function with the serial number and MAC address as inputs.

The following Python pseudocode summarizes the `gen_key` function, or see the [original C code](https://git.lsd.cat/g/cyberchallenge-modem/src/branch/master/keygen/keygen.c):

{{< highlight python >}}
def keygen(mac, serial):
    l = 20
    password = ""
    md5a = hashlib.md5(mac).hexdigest()
    md5b = hashlib.md5(serial).hexdigest()
    w = int(md5a[0:8], 16)
    x = int(md5a[8:16], 16)
    y = int(md5a[16:24], 16)
    z = int(md5a[24:32], 16)

    for i in range(0, l):
    x, y, z, w, t = xorshift(x, y, z, w)
    password += md5b[t % 20]

    return password
{{< / highlight >}}

The purpose of the algorithm is to be straightforward to understand, easy to reverse-engineer, and quick to implement. The compiled static binary containing the generation functions is the same for all teams and is distributed at the start of the competition. From this point onward, there are two possible approaches to solving the challenge:

1. **Reverse-engineer the C code**: Rewrite the `gen_key` function with custom inputs.
2. **Patch or hook the binary**: Bypass the platform check, modify the hardcoded paths, and run the binary with `qemu-user` to emulate the `arm64` architecture.

Once an attacker successfully completes this phase, they can compute both the **WPA2 key** for wireless connectivity and the **web interface password** for all devices by leveraging the list of MAC addresses and serial numbers provided as part of the challenge materials. 

Before an attacker can begin targeting other devices, they must first complete the exploit chain on their own device. This involves connecting to the web interface of the device, accessible via the game VPN tunnel, and authenticating using the computed password. The serial number is conveniently echoed in the basic authentication request prompt, simplifying this initial step.

### Command Injection
![Screenshot of utils.php rendered in a browser.](/posts/2025-01-01-an-experimental-hybrid-ctf-for-the-italian-2020-cyberchallenge-03.png)

Post-authentication command injections are among the most common web vulnerabilities found in embedded devices, particularly in ISP-supplied CPEs. 

Once the attacker logs in to the web interface, they are greeted with a basic PHP web page displaying minimal information, such as the device's IP address and basic network statistics. 

The web application menu contains only two additional pages:

1. **Utils Page**: This is where the vulnerability to be exploited in this phase resides.
2. **Update Page**: While present, this page is merely a hint for the next phase of the challenge.

As is common in many embedded devices with network functionalities, the **Utils Page** implements basic network diagnostic tools such as **ping** and **traceroute**. These functions are typically provided using standard Linux utilities, regardless of the programming language in use, and involve a call to a `system()`-like function. If the input parameters are not properly sanitized and enclosed in quotes, command injection vulnerabilities are frequently possible, even when some characters are blacklisted.

The following excerpt of code demonstrates this mechanism, or see the [original](https://git.lsd.cat/g/cyberchallenge-modem/src/branch/master/webpanel/utils.php):

{{< highlight php >}}

<?php

$blacklist = array(';', '#', '(', ')', '|', '&', ' ', "\t", '<', '>');

if ($_SERVER['REQUEST_METHOD'] === 'POST' && ($_POST['action'] === 'ping' ||
    $_POST['action'] === 'traceroute') && isset($_POST['host']) && !empty($_POST['host'])) {

    switch ($_POST['action']) {
        case 'ping':
            $cmd = '/bin/'.$_POST['action']. ' -c 2';
            break;
        case 'traceroute':
            $cmd = '/usr/bin/'.$_POST['action'];
            break;
    }
    $action = $_POST['action'];
    $host = str_replace($blacklist, '', $_POST['host']);
    exec($cmd.' '.$host, $result);
}
?>
{{< / highlight >}}

The blacklist is sufficient to prevent some common patterns of command injection, such as concatenating commands using shell operators (`;`, `|`, `&`) or performing variable substitution using `$()`. However, there are still multiple ways to bypass this filtering:

- **`\n` (newline)**: A command separator.
- **`` ` `` (backtick)**: A variable substitution character.
- **`${IFS}` (Input Field Separator)**: The default field separator.

To summarize, one could gain execution by either `\n` or `` ` `` and then add input to any command by separating the keywords and values using `${IFS}`, for instance, sending the following POST body: `` action=ping&host=127.0.0.1`uname${IFS}-a` ``.

### Elevation of privilege via race condition

To ensure a fair game for all participants, every attack, if possible, should happen in a controlled environment, meaning that root privileges must never be granted. In this context, privilege escalation occurs from a low-privileged service user, `www-data`, to a standard user, `upgrade`, who has a shell defined in `/etc/passwd` and can eventually log in via SSH. 

Additionally a team must only be able to patch their own device. To achieve this, a random password called `update_key` is generated upon first boot and is unique per device. This password is distributed to each team and stored on disk with restricted read permissions.

To guide players in the right direction and save time, the **Update** feature contains hints in the form of the following code:

{{< highlight php >}}
<?php

if ($_SERVER['REQUEST_METHOD'] === POST
    && isset($_FILES['update'])
    && isset($_POST['password'])
    && !empty($_POST['password'])) {
	move_uploaded_file($_FILES['file']['tmp_name'], '/tmp/update.tgz.cc');
	exec("/usr/bin/sudo key='".escapeshellarg($_POST['password'])."' /update.sh", $result);
}
?>
{{< / highlight >}}

The user `www-data` has read permissions for the `update.sh` script. Furthermore, the following lines are placed in `/etc/sudoers` and can be verified by running `sudo -l` without a password:

{{< highlight bash >}}
Defaults  env_reset
Defaults  env_keep = "key"

www-data ALL=(upgrade) NOPASSWD: /bin/sh /update.sh
{{< / highlight >}}

The key referenced in the snippet above corresponds to the `update_key`. Since this key was communicated with the challenge material, its purpose is solely to prevent exploitation of this step on a device not owned by the attacker.

The content of `update.sh` is as follows. The script uses SH syntax for the busybox shell and utilities:

{{< highlight bash >}}
#!/bin/sh

if [[ -z $key ]]; then
    /bin/echo "Usage: sudo key=<update_key> -E update.sh"
    exit 1
fi

password=`/bin/cat /etc/update_key | /usr/bin/sha512sum | /usr/bin/cut -d' ' -f 1`
auth=`/bin/echo -n $key | /usr/bin/sha512sum | /usr/bin/cut -d' ' -f 1`

if [[ "$auth" != "$password" ]]; then
        /bin/echo "Wrong password"
        exit 1
fi

pubkey="/pub.pem"
file="/tmp/update.tar.cc"
/bin/chmod 777 $file
/bin/echo "### ccOS Update Script ###"
/bin/echo "[+] Starting"
/bin/sleep 1
/bin/echo "[+] Extracting Signature"
skip=$(expr $(stat -c '%s' $file) - 256)
if [[ -L $file ]]; then
    exit 0
fi 
/bin/dd if=/tmp/update.tar.cc of=sig bs=1 count=256 skip=$skip
/usr/bin/truncate -s $skip $file
check=`/usr/bin/openssl dgst -sha256 -verify $pubkey -signature /tmp/sig $file`
if [ "$check" == "Verified OK" ]; then
    /bin/echo "[+] Signature is valid!"
    /bin/echo "[+] Upgrading..."
    /bin/tar -xvf $file -C /
    /bin/rm /tmp/sig
    /bin/echo "[+] Done"
else
    /bin/echo "[-] Signature error, exiting..."
    /bin/rm /tmp/sig
fi
{{< / highlight >}}

The public certificate stored in `/pub.pem` is generated during firmware building, and the corresponding private key is securely held by the competition organizers and never revealed. Furthermore, both `/update.sh` and `/pub.pem` are root-owned and cannot be modified. A sample update archive signed with a valid signature is placed in the root directory of every device.

The attacker can now successfully execute the `/update.sh` script with the permissions of the `upgrade` user. However, the attacker can only extract the signed sample archive since it passes the signature check. The `upgrade` user is a standard system user with a home directory and a defined shell. This user also has write permissions in the directory containing the PHP files for the web panel.

The vulnerability here is a classic TOCTOU race condition: between the signature verification and the actual extraction, there is a small time window during which the archive file can be swapped with a custom one that does not have a valid signature. Since the original file is owned by the `www-data` user who moved it to the `/tmp` folder, the attacker is allowed to make changes. Additionally, the cryptographic operations involving RSA are computationally expensive and, therefore, slower on embedded devices, including the one used in this challenge.

Sample exploit script:

{{< highlight bash >}}
#!/bin/sh

# pwn is the crafted package
# orig is the sample signed package

cd /tmp;
cp orig update.tar.cc;
chmod 777 update.tar.cc pwn;
sudo -u upgrade key=<key> /bin/sh /update.sh > log &
while true; do
    if [[ "$(cat log | grep valid)" ]]; then
        cp pwn update.tar.cc;
        exit 0
    fi
done;
{{< /highlight >}}

The easiest way to elevate privileges, by having a write primitive in the `upgrade` user's home folder, is to write the file `~/.ssh/authorized_keys` with proper permissions and log in via SSH. Once this is done, it is possible to edit the PHP files and patch the command injection, thereby allowing a team to fix the main entry point on their device.

### Lateral Movement

The last step is to attack other teams' devices and finally start capturing flags and scoring points. The players know that the devices are all in the same room and simulate consumer routers. They can calculate all the WPA2 keys of all devices using the solution for the first step and the list of all serial numbers and MAC addresses provided with the challenge material. They are also able to log in via SSH as the `upgrade` user on their assigned device.

Since operating on the Wi-Fi module requires higher privileges, and since users should not interfere with the configured `hostapd` in Access Point mode, some basic scripts are provided and can again be discovered by running `sudo -l`.

In `/etc/sudoers`:

{{< highlight bash >}}
upgrade  ALL=(root) NOPASSWD: /usr/sbin/iw client0 scan, /usr/sbin/iw client0 link, /bin/cat /etc/wpa_supplicant.conf, /bin/sh /wifi/connect.sh *, /bin/sh /wifi/disconnect.sh 
{{< /highlight >}}

Outline of the commands enabled:

- `/usr/sbin/iw client0 scan` → for wireless scanning
- `/usr/sbin/iw client0 link` → to check if it is currently associated with a wireless network
- `/bin/cat /etc/wpa_supplicant.conf` → to check for current settings
- `/bin/sh /wifi/connect.sh *` → to connect to a wireless network, supplying SSID and key on the command-line interface (CLI)
- `/bin/sh /wifi/disconnect.sh *` → to disconnect

All devices have a DHCP server installed, so the `connect.sh` script takes just an SSID and a WPA2 key as input and automatically connects. It also offers the possibility to run a wireless scan to get all the SSIDs of the opponent teams. Each SSID is in the form `CyberChallenge-%s`, where the format specifier is replaced by the last four characters of the serial number.

Once connected, it is possible to perform the command injection on every device and capture flags written in `/flag`. The whole process can be scripted and fully automated via SSH.

### Exploitation Flow

The full exploitation flow is displayed in the diagram below.

![Diagram of the full exploitation flow for the challenge.](/posts/2025-01-01-an-experimental-hybrid-ctf-for-the-italian-2020-cyberchallenge-04.png)

## Integration and Monitoring
### Local
All the devices are connected to a local 48-port switch with a connection trunk to a Debian-based server. Each device has its own VLAN trunked on the same terminal cable. The server is configured to provide a different DHCP address range to every VLAN. All the VLANs are also bridged to the interface connected to the internet. However, there are strict firewall rules forbidding all network traffic between VLANs and any outgoing connection except those to the IP address of the Cloud Gameserver.

### Remote
The architecture of the Cloud Gameserver is out of scope for this document, but it can be summarized as a **VPN concentrator**. Each team and device is assigned a Wireguard peer, which is then bridged together. For competition security, it is critical that teams are perfectly isolated.

## Overview

![Infrastructure overview.](/posts/2025-01-01-an-experimental-hybrid-ctf-for-the-italian-2020-cyberchallenge-05.png)

## Final Thoughts
Only four of the 24 participating teams managed to complete the challenge, all within the last hour. Many teams struggled to score points due to issues caused by not calling the `disconnect.sh` script. [One team produced a valid writeup of the challenge](https://molteniluca.github.io/posts/homerooter/).

The source code and the documentation (in Italian) [is available at this git repository](https://git.lsd.cat/g/cyberchallenge-modem).