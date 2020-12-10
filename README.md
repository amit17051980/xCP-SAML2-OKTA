# OpenText Documentum xCP SAML2 Integration with OKTA
The instructions posted here are solely to help developers to understand the integration of Documentum xCP with any SAML IdaaS (Identity as a Service). Currently, there are limited articles on this, and it is quite challenging for new implementers. I have considered to use OKTA IdaaS provider as it is free for Developers and quite simple to learn advance features. Please refer OpenText xCP deployment guide before using this article. OpenText guides shows how to use ADFS, PingIdentity to implement SAML2, and have been used to signoff the releases.

## Before you begin...
Please note that you have satisfied below points before proceeding.

* I have Documentum xCP 16.4 Patched stack up and running
* I have OKTA account to register SAML2 app
* I have test OKTA, Documentum users where OKTA's login_name corresponds to Documentum's user_login_name
* I have open firewall where communications between xCP App server and OKTA app is open
* I have DNS for xCP Application (Optionally, Windows hosts file can be used for this purpose)

## 1. Enable SSL on xCP App Server (Tomcat-8.5)
It is being assumed that your xCP App Server is running on Windows to make it easy for Windows lovers. If not, you can still use this guide either as-is by cloning your xCP App instance on Windows or by re-factoring the path etc. per Linux system. 

### 1.a Create Keystore and Certificate files for xCP App Server

Lets assume that the dns name is '_xcpapp_' for your application. Create the '_xcpapp.keystore_' by executing the command below. Make sure PATH environment is pointing to your JRE. 

_**Command Prompt**_
```
cd C:\apache-tomcat-8.5.60\conf

keytool -genkey ^
-alias xcpapp ^
-keyalg RSA ^
-keystore xcpapp.keystore ^
-storepass changeit ^
-keypass changeit ^
-dname "CN=xcpapp, OU=xcpapp, O=xcpapp, L=London, ST=London, C=GB"
```
_**Shell Script**_
```
cd /usr/local/tomcat/conf

keytool -genkey \
-alias xcpapp \
-keyalg RSA \
-keystore xcpapp.keystore \
-storepass changeit \
-keypass changeit \
-dname "CN=xcpapp, OU=xcpapp, O=xcpapp, L=London, ST=London, C=GB"
```


Now, export the '_xcpapp.cer_' certificate by executing the command below:

_**Command Prompt**_
```
cd C:\apache-tomcat-8.5.60\conf

keytool -export ^
-alias xcpapp ^
-keystore xcpapp.keystore ^
-file xcpapp.cer ^
-storepass changeit
```
_**Shell Script**_
```
cd /usr/local/tomcat/conf

keytool -export \
-alias xcpapp \
-keystore xcpapp.keystore \
-file xcpapp.cer \
-storepass changeit
```

> **Note**: Place **xcpapp.keystore** and **xcpapp.cer** in _**C:\apache-tomcat-8.5.60\conf**_


### 1.b Import '_xcpapp.cer_' certificate to xCP App Server's java truststore

_**Command Prompt**_
```
cd C:\apache-tomcat-8.5.60\conf

keytool -import -trustcacerts ^
-alias xcpapp ^
-keystore "%JAVA_HOME%\jre\lib\security\cacerts" ^
-file xcpapp.cer -storepass changeit
```
_**Shell Script**_
```
cd /usr/local/tomcat/conf

keytool -import -trustcacerts \
-alias xcpapp \
-keystore "%JAVA_HOME%\jre\lib\security\cacerts" \
-file xcpapp.cer -storepass changeit
```

### 1.c (NOT REQUIRED if Client=App Server) Import '_xcpapp.cer_' certificate to Client's (Browser) java truststore

_**Command Prompt**_
```
keytool -import -trustcacerts ^
-alias xcpapp ^
-keystore "%JAVA_HOME%\jre\lib\security\cacerts" ^
-file xcpapp.cer -storepass changeit
```

### 1.d Update '_C:\apache-tomcat-8.5.60\conf\server.xml_' for SSL

```
<Connector port="8443"
protocol="org.apache.coyote.http11.Http11NioProtocol"
ciphers="TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
TLS_ECDHE_RSA_WITH_RC4_128_SHA,TLS_RSA_WITH_AES_128_CBC_SHA256,
TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA256,
TLS_RSA_WITH_AES_256_CBC_SHA,SSL_RSA_WITH_RC4_128_SHA"
sslEnabledProtocols="TLSv1,TLSv1.1,TLSv1.2"
enableLookups="true"
sslProtocol="TLS"
keystorePass="changeit"
keystoreFile="C:\apache-tomcat-8.5.60\conf\xcpapp.keystore"
#LINUX
#keystoreFile=/usr/local/tomcat/conf/xcpapp.keystore
clientAuth="want"
secure="true"
scheme="https"
SSLEnabled="true"
maxThreads="150"/>
```

Restart the services and verify that the xCP App Server is accessible over SSL.

URL : [https://xcpapp:8443/{xCP-App-Name}](https://xcpapp:8443/sample)

## Register xCP App on OKTA IdaaS

Follow the screenshots under OKTA Admin --> Applications

![Create New App](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/CreateApp-1.PNG)

![Select Web and SAML2.0](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/CreateApp-2.PNG)

![App Settings](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/App-Settings.PNG)

![General Settings](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/General-Settings.PNG)

![Attribute Settings](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/Attribute-Settings.PNG)

> **Note**: _View Setup Instructions to download the metadata.xml and certificate that will be use in xCP App and Method Server respectively._
 
![Metadata and Certificate](https://github.com/amit17051980/xCP-SAML2-OKTA/blob/main/Get-MetaData-Certificate.PNG)

## Configure xCP App and Method Server for SAML2 AuthN/AuthZ and Documentum Login Session

There are 2 main components within xCP stack those need to be configured for E2E SAML based login session. Please follow the steps in sequence and restart full stack if there are any issues.

### Download and place metadata.xml in xCP App Server location

Get the **metadata.xml** (review previous instructions) and save it to an appropriate place in xCP App Server. In this case I used **C:** as it search in this location for some reason. The next step is based on some predefined locations I used earlier in this post. Please review and use accordingly.

### Modify xCP App 'rest-api-runtime.properties' file in xCP App Server

_Here is the sample file for you to use. Mind that the trailing spaces are causing mystery and you might give-up very soon. So triple make sure that the file is having no trailing spaces._

> C:\apache-tomcat-8.5.60\webapps\sample\WEB-INF\classes\rest-api-runtime.properties
```
#
# Copyright (c) 2017. Open Text Corporation. All Rights Reserved.
#

# This file holds configurable parameters for the xCP REST server side deployment.
# Settings in this file override the default ones defined in specific libraries.


###################################################
##        xCP Rest Service Configuration         ##
###################################################

# The default number of results per page. The value MUST be a non-negative integer. The default value is 100.
rest.paging.default.size=100

# Specifies the max number of results per page.
rest.paging.max.size=1000

####################################################
##       Security Configuration                   ##
####################################################

# Authentication scheme
#rest.security.auth.mode=basic
# SAML authentication schema
#rest.security.auth.mode=saml
# For fallback support, change the mode to saml-basic
rest.security.auth.mode=saml-basic
#specify the java key store file
rest.security.saml2.ks.file=C:\\apache-tomcat-8.5.60/conf/xcpapp.keystore
#specify the password of the java key store
rest.security.saml2.ks.password=changeit
#specify the alias of key entry used by the SAML Service Provider to sign the SAML message
rest.security.saml2.ks.entry.alias=xcpapp
#specify the password of the key entry used by the SAML Service Provider to sign the SAML message
rest.security.saml2.ks.entry.password=changeit
#specify the HTTP method used to send SAML request to the Identity Provider
rest.security.saml2.request.binding=HTTP-Redirect
#specify the metadata files of the Identity Providers
rest.security.saml2.idp.metadata.files=metadata.xml
#specify the attributes used to extract principal names from the SAML response
rest.security.saml2.user.attributes=UserName
#specify the cookie timeout of SAML request token
rest.security.client.saml2.timeout=300
#specify the documnetum ticket timeout , our recommendation is to set the value twice as the http session timeout for the xCP application and provide value is in minutes
xcp.signon.ticket.timeout=480
#xcp.signon.logout.url=https://dev-354610.oktapreview.com/login/login.htm
```

### Download and place saml.crt in Java Method Server

Get the **saml.crt** (review previous instructions) and save it to an appropriate place in **dmadmin** user's area. In this case I used **/home/dmadmin/certs/**. I created file **saml.crt** (.cert is not recognised) in this folder. The next step is based on some predefined locations I used earlier in this post. Please review and use accordingly.

### Modify 'saml.properties' file in Java Method Server

_Here is the sample file for you to use. Mind that the trailing spaces are causing mystery and you might give-up very soon. So triple make sure that the file is having no trailing spaces._

> /opt/dctm/wildfly9.0.1/server/DctmServer_MethodServer/deployments/ServerApps.ear/SAMLAuthentication.war/WEB-INF/classes/saml.properties
```
stackTrace = True
certPath = /home/dmadmin/certs
responseSkew = 60
```

Now is the time to restart all the components and give it a try.

Hope this helps you understanding OKTA with the OpenText Documentum xCP, and gives you step forward to use Azure SAML registry without ADFS, and some other IdaaS like auth0.com.

> **Note**: Don't forget to assign the users in OKTA App and the Documentum. Make sure the login names are matching between OKTA and Documentum for SAMLAuthentication.war to get the valid user's login ticket.

## Appendix

### Troubleshooting SAML Issues on xCP App Server

To enable logging of error messages, configure these packages in the log4j.properties configuration file:

```
log4j.category.com.emc.xcp.rest.security=DEBUG
log4j.category.com.emc.documentum.rest=DEBUG
log4j.category.org.springframework=DEBUG
```

### Method Server SAML Log Location

`/opt/dctm/wildfly9.0.1/server/DctmServer_MethodServer/logs/SAMLAuthentication.log`

> E.g. Output
```
2020-12-05 11:15:12.673 INFO  - Thread[default task-12,5,main] ====START SAML AUTHENTICATION====
2020-12-05 11:15:12.674 INFO  - Thread[default task-12,5,main] User: tuser1@okta.com
2020-12-05 11:15:12.674 INFO  - Thread[default task-12,5,main] responseMessage: ......
......................................................................................
......................................................................................
2020-12-05 11:15:12.926 INFO  - Thread[default task-12,5,main] Response status: urn:oasis:names:tc:SAML:2.0:status:Success
2020-12-05 11:15:12.926 INFO  - Thread[default task-12,5,main] Retrieving Certificates in folder: /home/dmadmin/certs
2020-12-05 11:15:12.926 INFO  - Thread[default task-12,5,main] Validating signature with certificate: /home/dmadmin/certs/saml.crt
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Signature verification: Success
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Verifying token validity
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] notBefore: 2020-12-05T11:10:13.263Z
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] notOnOrAfter: 2020-12-05T11:20:13.263Z
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Token valid
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Validating NameID from Subject
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] NameID: tuser1@okta.com
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] UserName matches with the NameID in Assertion
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Validating subject confirmation
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Ticket expiry: 2020-12-05T11:20:13.263Z
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Subject confirmation verified and ticket is valid
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] Authentication success
2020-12-05 11:15:12.938 INFO  - Thread[default task-12,5,main] ====END SAML AUTHENTICATION====
```

### Date/Time Sync Issue on VMs

If your content server is not in sync with the app server date/time, you are likely to see the below message on _SAMLAuthentication.log_

```
2020-12-08 10:34:56.902 INFO  - Thread[default task-3,5,main] Verifying token validity
2020-12-08 10:34:56.903 INFO  - Thread[default task-3,5,main] notBefore: 2020-12-08T21:42:06.275Z
2020-12-08 10:34:56.903 INFO  - Thread[default task-3,5,main] Assertion is not yet valid, invalidated by condition notBefore
```
