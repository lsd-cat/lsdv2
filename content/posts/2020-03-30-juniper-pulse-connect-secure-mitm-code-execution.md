+++
date = "2020-03-30T00:00:00+00:00"
title = "Juniper Pulse Connect Secure MITM Code Execution"
toc = false
+++

 - No certificate Validation - [CVE-2020-11580](https://nvd.nist.gov/vuln/detail/CVE-2020-11580)
 - Command Injection - [CVE-2020-11581](https://nvd.nist.gov/vuln/detail/CVE-2020-11581)
 - DNS Rebinding - [CVE-2020-11582](https://nvd.nist.gov/vuln/detail/CVE-2020-11582)

Link to Juniper official advisory [SA44426](https://kb.pulsesecure.net/articles/Pulse_Security_Advisories/SA44426), and EU-CERT [2020-023](https://cert.europa.eu/publications/security-advisories/2020-023/).

The **Host Checker** is a client-side component of the [Pulse Connect Secure](https://www.pulsesecure.net/products/pulse-connect-secure/) appliance, which may be required for connecting to the VPN. The Host Checker retrieves a policy from the server and performs basic checks on the client system. These checks can include verifying MAC addresses, inspecting running processes (e.g., ensuring antivirus software is active), and others.

On Windows, the plugin operates as an ActiveX component, while on Linux, Solaris, and macOS, it functions as a Java applet.

As with most client-side checks, these can be bypassed. An open-source, albeit not well-documented, implementation of such bypassing [exists here](https://raw.githubusercontent.com/russdill/juniper-vpn-py/master/tncc.py). 

## Attack
Probably to maintain compatibility with misconfigured instances, the **Host Checker** neither verifies the validity of the server certificate nor checks its hostname. This vulnerability allows a malicious server to set a harmful cookie (or exploit this via DNS Rebinding), which can lead to a command injection when the user is deemed non-compliant. It is worth noting that a malicious server can intentionally force a user to be marked as non-compliant.

The client implements a custom protocol to communicate with the server. For further reference, the [open-source client](https://raw.githubusercontent.com/russdill/juniper-vpn-py/master/tncc.py) has reverse-engineered and implemented this protocol. The file `tncc.jar` is not obfuscated, and the original source code can be extracted using almost any Java decompiler.

### Certificate
Below are excerpts from the code of the classes that handle connections with the Pulse Connect Secure appliance. For example, in `net.juniper.tnc.client.HttpNAR.HttpNAR`:

{{< highlight java >}}
private void trustAllCerts() throws Exception {
    final TrustManager[] tm = { new X509TrustManager() {
            @Override
            public X509Certificate[] getAcceptedIssuers() {
                return null;
            }
            
            @Override
            public void checkClientTrusted(final X509Certificate[] array, final String s) {
            }
            
            @Override
            public void checkServerTrusted(final X509Certificate[] array, final String s) {
            }
        } };
    final SSLContext instance = SSLContext.getInstance(NARUtil.getSSLProtocol());
    instance.init(null, tm, new SecureRandom());
    HttpsURLConnection.setDefaultSSLSocketFactory(instance.getSocketFactory());
}

private void allowHostnameMismatch() {
    HttpsURLConnection.setDefaultHostnameVerifier(new HostnameVerifier() {
        @Override
        public boolean verify(final String s, final SSLSession sslSession) {
            return true;
        }
    });
}

{{< / highlight >}}

Both function gets executed when initializing the connection to a server. From the same class as above:

{{< highlight java >}}
public void initialize(final String[] array) throws Exception {
        [..parametr parsing..]
    if (this.mHomeDir.length() == 0) {
        this.mHomeDir = System.getProperty("user.home");
    }
    if (HttpNAR.gLoggingEnabled) {
        [..omitted log instructions..]
    }
    if (!this.isPlatformSupported()) {
        NARUtil.logError("HttpNAR: unsupported operating system " + NARUtil.getOSName() + "; stopping...");
        throw new Exception("Unsupported operating system");
    }
    this.mAppSupportDir = this.getPlatformSupportDir(this.mHomeDir);
    [..proxy usage..]
    
    (this.mTncClient = new TNCClient()).initialize(this);
    this.loadbundledIMC();
    this.mHandshakeRequestor.start();
    this.trustAllCerts();
    this.allowHostnameMismatch();
}
{{< / highlight >}}

### Cookie
In order for the Host Checker to work two cookies are needed, `DSPREAUTH` and `DSSIGNIN`.
They can be either set by the server or from sending commands to a socket listening to all interfaces (but accepting connections only from localhost).
The following code updates the `DSPREAUTH` cookie when sending periodic updates to the server. Periodic updates may or may not be required depending on the policy configuration.

From `net.juniper.tnc.client.HttpNAR.HttpConnection`:

{{<highlight java >}}
public int sendUpdate(final byte[] array, final ByteArrayOutputStream byteArrayOutputStream, final boolean b) throws Exception {
    NARUtil.logInfo("TNCoHTTP: sending update = ");
    NARUtil.logData(array);
    final String byteArrayToBase64 = Base64.byteArrayToBase64(array);
    final String string = "https://" + this.mIveHost + "/dana-na/hc/tnchcupdate.cgi";
    NARUtil.logInfo("TNCoHTTP: opening connection to " + string);
    final HttpsURLConnection httpsURLConnection = (HttpsURLConnection)new URL(string).openConnection();
    httpsURLConnection.setDoOutput(true);
    httpsURLConnection.setDoInput(true);
    httpsURLConnection.setRequestProperty("Cookie", "DSPREAUTH=" + this.mPreauthCookie + ";DSSIGNIN=" + this.mSignIn);
    if (this.mUserAgent.length() > 0) {
        httpsURLConnection.setRequestProperty("User-Agent", this.mUserAgent);
    }
    NARUtil.setConnectTimeout(httpsURLConnection, 2000);
    httpsURLConnection.connect();
    final PrintWriter printWriter = new PrintWriter(httpsURLConnection.getOutputStream());
    [..adding parameters..]
    if (b) {
        printWriter.print("firsttime=1;");
    }
    printWriter.close();
    String headerFieldKey;
    for (int n = 1; (headerFieldKey = httpsURLConnection.getHeaderFieldKey(n)) != null; ++n) {
        final String headerField = httpsURLConnection.getHeaderField(n);
        if (headerFieldKey.equalsIgnoreCase("Set-Cookie") && headerField.startsWith("DSPREAUTH=")) {
            final int index = headerField.indexOf(59);
            this.mPreauthCookie = headerField.substring(10, index);
            this.mPreauthOpts = headerField.substring(index);
            NARUtil.logInfo("TNCoHTTP: received response DSPREAUTH = " + this.mPreauthCookie + this.mPreauthOpts);
        }
    }
{{< / highlight >}}

### Command injection
When a client is found to be non-compliant, remediation instructions have to be shown to the user in order to give him a chance to fix his problems.
In `net.juniper.tnc.client.HttpNAR.TNCHandshake`:
{{< highlight java >}}
public void doCustomRemediateInstructions() {
    final StringBuffer sb = new StringBuffer();
    sb.append("https://").append(this.mIcURL);
    sb.append("/dana-na/auth/rdpreauth.cgi?DSPREAUTH=" + this.mCookie);
    final String string = sb.toString();
    NARUtil.logInfo("TncHandshake: launching browser for " + string);
    NARUtil.execCommand((NARUtil.isMacOSX() ? "open -a Safari " : ((NARUtil.isLinux() || NARUtil.isSolaris()) ? "firefox " : "")) + string);
    NARUtil.logInfo("TncHandshake: browser launched");
}
{{< / highlight >}}

From `net.juniper.tnc.client.HttpNAR.NARUtil`:
{{{< highlight java >}}}
public static String execCommand(final String s) {
    String execCommand = null;
    if (NARUtil.PlatformUtil != null) {
        execCommand = NARUtil.PlatformUtil.execCommand(s);
    }
    return execCommand;
}
{{< / highlight >}}

From `net.juniper.tnc.client.NARPlatform.linux.LinuxNARPlatform`:
{{< highlight java >}}
@Override
public String execCommand(final String s) {
    String output = null;
    try {
        final Process exec = Runtime.getRuntime().exec(s);
        final CommandOutputThread commandOutputThread = new CommandOutputThread(exec.getInputStream());
        final CommandOutputThread commandOutputThread2 = new CommandOutputThread(exec.getErrorStream());
        commandOutputThread.start();
        commandOutputThread2.start();
        final int wait = exec.waitFor();
        if (wait != 0) {
            this.logError("Command " + s + " failed; return = " + wait + "; error output = " + commandOutputThread2.getOutput());
        }
        output = commandOutputThread.getOutput();
    }
    catch (Exception ex) {
        this.logException(ex);
    }
    return output;
}
{{< / highlight >}}

As we can see, the `NARUtil.execCommand()` function is just a wrapper around `Runtime.getRuntime().exec()`.

### Full Chain
An attacker capable of performing a Man-in-the-Middle (MitM) attack can spoof the server and send a malicious cookie along with an impossible-to-comply policy. For example, the attacker could inject a cookie containing a command injection payload on Linux, such as:

- `; sleep 20;`
- `$(sleep 20)`
- `\nsleep 20`

When the client attempts to comply with the policy, it will fail and subsequently execute the injected command while displaying remediation instructions. This exploitation leverages the fact that the client processes the appended malicious value as part of the command.

### DNS Rebinding (Extra)
The Host Checker is controlled using a command socket. By default, when the Host Checker is started, it opens a socket using `ServerSocket(0)` which will automatically choose a port to listen on all interfaces. The selected port will then be writte to `~/.pulse_secure/narport.txt`.
The code prevents sending commands from a non-local host but apart from that doesn't have any other authentication mechanism. An attacker may [brute force the port using JavaScript](https://portswigger.net/research/exposing-intranets-with-reliable-browser-based-port-scanning) or if in the same network directly ports can since it is listening on all interfaces. Once the ports is known, a DNS Rebinding attack can be done and commands can be sent to the socket. While this does not imply a command execution per se, one of the supported commands is `setcookie` which sets the cookie used for the command injection descripted in the paragraph above.
Note: the command socket expect a command to start at the beginnig of the first line but will try to parse up to 25 invalid commands before exiting, so a GET or a POST requests should work.
