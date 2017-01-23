---
permalink: /:categories/:year/:month/:title.html
layout: post
title: Retrieving a remote SSL cert and  importing it in the JRE's keystore
date: '2015-05-07T15:32:00.000-07:00'
author: Andres Olarte
tags:
- security
- tips
- java
modified_time: '2015-05-07T15:48:15.806-07:00'
thumbnail: http://2.bp.blogspot.com/-tZ8vT_hF3sc/VUvraf1IjWI/AAAAAAAAFME/6YuFvbNl2nc/s72-c/AD76394B17.jpg
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-6895112003371107263
blogger_orig_url: http://www.javaprocess.com/2015/05/retrieving-remote-ssl-cert-and.html
---

<img src="/images/2015-05-07-retrieving-remote-ssl-cert/lock.jpg" width="150" align="left" hspace="10"> 

In the daily work of a developer, it's often necessary to interact with external servers, for example to access web services. Many times, these are test or development servers, some of which might be secured with a self signed SSL certificate. Since these certificates are not trusted by default in the JRE, the best option is to import said certificate in your JRE. While the information to achieve this is definitely out there, I always find myself having look at a quite a few pages to get all of the pieces together to get the certificate imported. This quick post is an attempt to have full step by step guide to getting your certificate installed. While there are ways of retrieving a certificate using a web browser, that is sometimes not possible, for example when working in a remote server that might lack a GUI. This approach can be used with only a terminal.

First retrieve the server's certificate, and format it in a format suitable to import into the JRE's keystore format. Just replace the hostname and port that you want to connect to:

~~~
echo | openssl s_client -connect hostname:port 2>/dev/null | openssl x509 > test_self_signed.cer
~~~

This will create file that will look roughly like this:

~~~
-----BEGIN CERTIFICATE-----
MIIDezCCAmOgAwIBAgIEaejl6zANBgkqhkiG9w0BAQsFADBuMRAwDgYDVQQGEwdV
bmtub3duMRAwDgYDVQQIEwdVbmtub3duMRAwDgYDVQQHEwdVbmtub3duMRAwDgYD
... quite a few more lines here ...
/FHwsiau3ntmBn358GhaD4exNPkf346eDcYnHii/nvfEJivC5vEDnQsFBcxSWEU6
P/GspsPqjjuEwxh6HDGOYBNg7a7jwk66uwPJ/QoRNg==
-----END CERTIFICATE-----
~~~

Second, import the file into the JRE's keystore.  If using a JDK, the JRE is in a directory called jre inside your JDK. Make sure that you have the proper permissions, if the JDK is installed system wide, you might require root access for the next step. Ensure that you are importing the cert into the JDK version that you're using, since you might have several versions installed in your computer.

~~~
keytool -import -alias test_self_signed -file self_signed.cer -keystore lib/security/cacerts -storepass changeit
~~~

Finally, you can list the certs in the keystore to ensure your cert was imported:

~~~
keytool -list -keystore lib/security/cacerts -storepass changeit
~~~

Or you can search for only your particular cert by passing the alias name:

~~~
keytool -list -alias test_self_signed -keystore lib/security/cacerts -storepass changeit
~~~

You will see an output similar to this: 

~~~
test_self_signed, May 5, 2015, trustedCertEntry,
Certificate fingerprint (SHA1): 9C:A6:FC:74:05:02:B1:11:F9:02:BB:3C:14:DA:7A:5B:84:F2:F0:A8
~~~

The commands were put together from examples I found [here](https://docs.oracle.com/javase/tutorial/security/toolsign/rstep2.html) and [here](http://stackoverflow.com/questions/7885785/using-openssl-to-get-the-certificate-from-a-server).