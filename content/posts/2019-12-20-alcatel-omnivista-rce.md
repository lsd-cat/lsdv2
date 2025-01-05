+++
date = "2019-12-20T00:00:00+00:00"
title = "Alcatel-Lucent OmniVista 4760/8770 Remote Code Execution"
note = "These were my first public CVE, while I did CTF and had some work experience, it was fun to start pursuing targets on my own."
toc = false
+++

### TL;DR
- **4760** suffers from an unauthenticated remote code execution vulnerability as SYSTEM. No special configuration is required.
- **8770** and **4760** both suffer from a remote administrative password disclosure vulnerability. No special configuration is required.
- **8770** suffers from an authenticated remote code execution vulnerability. When chained with the disclosure vulnerability, it becomes an unauthenticated RCE. In this case, access to port 389 and a directory license are required.
- [Link to the exploit](https://git.lsd.cat/g/omnivista-rce/src/branch/master/omnivista.py).

The following CVEs were assigned on 27/12/2019:
- 4760 pre-auth RCE: [CVE-2019-20049](https://nvd.nist.gov/vuln/detail/CVE-2019-20049)
- 4760 and 8770 Directory Manager credentials leak: [CVE-2019-20047](https://nvd.nist.gov/vuln/detail/CVE-2019-20047)
- 8770 post-auth RCE: [CVE-2019-20048](https://nvd.nist.gov/vuln/detail/CVE-2019-20048)

Furthermore, [Alcatel-Lucent has published a statement acknowledging the issues](https://www.al-enterprise.com/en/-/media/assets/internet/documents/sa-c0065-ov8770-rce-vulnerability-en.pdf).

As per Alcatel-Lucent's statement, **4760** will remain unpatched as it is a discontinued product. **8770** should be fixed in versions `4.1.2` and `4.2`.

## Introduction
Alcatel OmniVista is a graphical interface for Alcatel OmniPCX, a common VoIP solution. This software is used to manage VoIP accounts and serve as a public directory. [Official product page](https://www.al-enterprise.com/en/products/communications-management-security/omnivista-8770-network-management-system).

I noticed this software while performing a penetration test. At the time, there were no known vulnerabilities in this component. Without the source code, I was unable to identify any exploitable vulnerabilities but found a few hints:

- Exposed error log at `/log/error.log`
- LDAP errors shown in the error log when special characters were used in login and search forms
- Failed `unserialize()` calls in the error log while decoding the `bookmarks`, `themes`, or `cfilter` cookies

The LDAP injection seemed unlikely to yield significant results, as extracting hashed passwords wasn't my focus. I was more intrigued by the `unserialize()` calls but lacked enough insight into the PHP code to proceed further. Therefore, I decided to acquire a full copy of the software for further analysis.

## The hardest part... Getting a copy
Obtaining the OmniVista 4760/8770 software as a researcher is challenging. It is distributed exclusively on CD/DVD to legitimate companies via local Alcatel-Lucent partners. Here’s how I approached the problem:

1. **Identify file names** (e.g., CD name or component name). These can often be found in online documentation, manufacturer websites, or manuals uploaded by random users (e.g., Scribd). Forums like [Alcatel Unleashed](https://www.alcatelunleashed.com/) are also useful.
2. **Search for downloads** by networking on forums, using Google dorking for specific components (e.g., `intext:"ClientSetup.exe"`), or searching full filenames. Create a list of known versions and filenames to dork effectively.
3. **Use file-sharing sites**, such as [4shared](https://www.4shared.com). For example: [https://www.4shared.com/rar/HsteugXy/A4760_R500702b.html](https://www.4shared.com/rar/HsteugXy/A4760_R500702b.html). Sorting by file size can help locate relevant files.

## Unpacking It
Since **4760** is an outdated product, I set up a Windows XP virtual machine and attempted installation. As is common with enterprise software, the installation failed—neither the main services were set up properly, nor were any PHP files extracted. Part of the files were packed using the old ACE file format, likely with custom modifications. Even tools like binwalk proved ineffective.

After significant frustration with this outdated software, I eventually discovered the PHP files inside a CAB archive.

## Vulnerabilities

### 4760 pre auth remote code execution

The file `info.php`, given the parameter `void=phDPhd` display the `phpinfo()` useful for debugging and vulnerable target identification.

{{< highlight php >}}
$MyG["void"] = varform("void");
if ($MyG["void"] == "phDPhd") {
    phpinfo();
}
{{< / highlight>}}

The following two functions in `utils.php` are used to extract `$_POST` and `$_GET` variables as well as to manage `$_SESSION`.

{{< highlight php >}}
function varform($nom)
{
    if (!isset($_GET)) {
        global $MyG;
        if (isset($MyG[$nom])) {
            $var = $MyG[$nom];
            return $var;
        }
        $var = false;
        return $var;
    }
    if (isset($_GET[$nom])) {
        $var = $_GET[$nom];
        return $var;
    }
    if (isset($_POST[$nom])) {
        $var = $_POST[$nom];
        return $var;
    }
    $var = false;
    return $var;
}
function sessionform($nom)
{
    global $MyG;
    $toEncodeList = array("password", "ldappwd");
    if (!isset($MyG[$nom]) || $MyG[$nom] === false) {
        if (isset($_SESSION[$nom])) {
            if (in_array($nom, $toEncodeList)) {
                $var = decodepwd($_SESSION[$nom]);
                return $var;
            }
            $var = $_SESSION[$nom];
            return $var;
        }
        $var = false;
        return $var;
    }
    $var = $MyG[$nom];
    if (in_array($nom, $toEncodeList)) {
        $_SESSION[$nom] = encodepwd($var);
        return $var;
    }
    $_SESSION[$nom] = $var;
    return $var;
}
{{< / highlight>}}

The admin interface allows authenticated users to edit the we interface themes. The following code from the file `EditThemeAction.php` checks for the users permissions before showing the page used to edit a theme.

{{< highlight php >}}
class EditThemeAction
{
    public function Invoke()
    {
        global $MyG;
        $MyG["themeId"] = varform("themeId");
        $MyG["themeId"] = sessionform("themeId");
        self::getaccessparameters();
        self::getuserlogin();
        $access = new CustomAccess($MyG["ldapHost"], $MyG["ldapPort"], "o=nmc", $MyG["userDn"], $MyG["userPass"]);
        if ($access->Connect() == true) {
            if ($access->TestAdminRights()) {
                $skin = new SkinAccess($MyG["themeId"]);
                $MyG["themeDate"] = $skin->GetLastMDate();
                sessionform("themeDate");
                $view = new EditThemeView($skin, false);
                $view->Display();
                $skin = null;
            } else {
                $view = new ErrorPopupView($access);
                $view->Display();
            }
            $access->Disconnect();
        } else {
            logerror(E_ERROR, $access->GetErrorMessage());
            $view = new ErrorPopupView($access);
            $view->Display();
        }
    }
}
{{< / highlight>}}

The default themes are numbered from 1 to 4 and each one has its files stored in a folder `/theme/<id>`. Each theme folder contains a `params.st` file which contains a serialized PHP Object with the theme configuration.
However, as seen below, the authentication and permission check is not performed when actually saving an edit. The only condition the is required to bypass is the `CompareThemeDate()`, which compares the last edit time of the `params.st` file with the value saved in session in the code above (`$MyG["themeDate"] = $skin->GetLastMDate()`). This check, intended or not, prevents an unauthenticated user to do the save, unless in the destination folder a `params.st` file is not yet present.

From `SaveThemeAction.php`:

{{< highlight php >}}
class SaveThemeAction
{
    public function Invoke()
    {
        global $MyG;
        logerror(E_NOTICE, "SaveThemeAction::Invoke()");
        $MyG["p"] = varform("p");
        $MyG["p"] = sessionform("p");
        $MyG["themeId"] = varform("themeId");
        $MyG["themeId"] = sessionform("themeId");
        $MyG["themeDate"] = sessionform("themeDate");
        $skin = new SkinAccess($MyG["themeId"]);
        if ($skin->CompareThemeDate($MyG["themeDate"])) {
            $skin->SetSkinParams($MyG["p"]);
            $skin->SetSkinImages($_FILES);
            $skin->BuildSkin();
            if ($skin->GetErrorNumber() == E_SUCCESS) {
                $MyG["themeDate"] = $skin->GetLastMDate();
                sessionform("themeDate");
                $themeAccess = new ThemeAccess();
                $themeAccess->SetThemeName($MyG["themeId"], $MyG["p"]["Name"]);
                $themeAccess = NULL;
            }
        }
        $view = new EditThemeView($skin, true);
        $view->Display();
        $skin = null;
    }
}
{{< / highlight>}}

The final pieces of vulnerable code are in `SkinAccess.php`:

{{< highlight php >}}
class SkinAccess
{
	...
    public function __construct($idSkin)
    {
        $this->errno = E_SUCCESS;
        $this->idSkin = $idSkin;
        $this->path = "../Themes/Theme" . $this->idSkin . "/";
        $this->filename = $this->path . "params.st";
        $this->params["Id"] = $idSkin;
        $this->params["Name"] .= $idSkin;
        $this->LoadSkinParams();
    }
    ...

    public function CompareThemeDate($beforeDate)
    {
        $bEquals = true;
        $afterDate = $this->GetLastMDate();
        if ($beforeDate != $afterDate) {
            $bEquals = false;
            $this->errno = E_MOD_BYANOTHER;
        }
        return $bEquals;
    }

    public function SetSkinImages($files)
    {
        foreach ($files as $key => $file) {
            if ($file["error"] == 0) {
                if (strncmp($file["type"], "image", 5) != 0) {
                    $this->errno = E_BAD_IMAGETYPE;
                } else {
                    $this->params[$key] = $file["name"];
                    $outputFile = $this->path . $file["name"];
                    move_uploaded_file($file["tmp_name"], $outputFile);
                }
            }
        }
    }
}
{{< / highlight>}}

So here it is! A path traversal in the `__construct()` and an unsecure file upload in  `SetSkinImages()`. Combining them we can set the upload folder to a folder which doesn't have yet a `params.st` file thus overcoming the `if ($skin->CompareThemeDate($MyG["themeDate"]))` condition because both values are empty!

The aforementioned exploit does not work in the 8770 because the path traversal is fixed, probably by checking that the `themeId` is between 1 and 4. The authentication issue seems to be still present, but the `CompareThemeDate()` check seems not possible to bypass.

### 4760/8770 LDAP admin credentials disclosure
The following PHP code is used to initialize some settings when a user starts a new session:

{{< highlight php >}}
abstract class Action
{
	...
    public function GetAccessParameters()
    {
        global $MyG;
        $sess2reg = array("ldapHost" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Host", "ldapPort" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Port", "ldapSuDn" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Login", "ldapSuPass" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Password");
        $MyG["ldapHost"] = sessionform("ldapHost");
        if ($MyG["ldapHost"] === false) {
            $shell = new COM("WScript.Shell");
            foreach ($sess2reg as $sess => $reg) {
                $MyG[$sess] = $shell->RegRead($reg);
                sessionform($sess);
                logerror(E_NOTICE, "Action::GetAccessParameters() [from registry] : " . $sess . "=" . $MyG[$sess]);
            }
            if (self::testledouxtruc() == false) {
                sessionunset("ldapHost");
                exit;
            }
        } else {
            foreach ($sess2reg as $sess => $reg) {
                $MyG[$sess] = sessionform($sess);
                logerror(E_NOTICE, "Action::GetAccessParameters() [from session] : " . $sess . "=" . $MyG[$sess]);
            }
        }
        self::getcompanysuffix();
    }
    ...
}
{{< / highlight>}}

So the code use the `COM` native PHP module to run some shell commands in order to get the LDAP bind credentials, which in this case are of "cn=directory manager" that is the administrator user of the instance. The password is encoded with a simple reversible algorithm we'll see below. Then the data is saved in the user session, which is stored server side.
While this code is quite terrible, and the whole idea of putting the cleartext credentials in the registry doesn't make sense to me, the real problem is the webserver configuration: all user session files are stored in a public directory! So by just starting a session and reading the respective session file it is possible to get the credentials.
Sessions are stored in `<url>/sessions/sess_<sessionid>`, simple as that.

The following function is used internally to decode the passwords stored in the session variables:

The decode function:

{{< highlight php >}}
function DecodePwd($data)
{
    $decryptData = "";
    if (strncasecmp("{NMC}", $data, 5) == 0) {
        $src = substr($data, 5);
    } else {
        $src = $data;
    }
    $a = 16;
    $len = strlen($src);
    for ($i = 0; $i < $len; $i++) {
        $c = substr($src, $i, 1);
        if (32 <= ord($c)) {
            $dst = ord($c) ^ $a;
            $b = chr($dst);
        } else {
            $dst = $c;
            $b = $c;
        }
        $decryptData .= $b;
        if (ord($b) != 0) {
            $a = $i * ord($b) % 255 >> 3;
        } else {
            $a = 16;
        }
    }
    return $decryptData;
}
{{< / highlight>}}
