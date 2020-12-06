# xCP-SAML2-OKTA
The instructions posted here are solely to help developers to understand the integration of Documentum xCP with any SAML IdaaS (Identity as a Service). Currently, there are limited articles on this, and it is quite challenging for new implementers. I have considered to use OKTA IdaaS provider as it is free for Developers and quite simple to learn advance features. Please refer OpenText xCP deployment guide before using this article. OpenText guides shows how to use ADFS, PingIdentity to implement SAML2, and have been used to signoff the releases.

## Before you begin...
Please note that you have satisfied below points before proceeding.

* I have Documentum xCP 16.4 Patched stack up and running
* I have OKTA account to register SAML2 app
* I have test OKTA, Documentum users where OKTA's login_name corresponds to Documentum's user_login_name
* I have open firewall where communications between xCP App server and OKTA app is open
* I have DNS for xCP Application (Optioinally, Windows hosts file can be used for this purpose)

## Enable SSL on xCP App Server
I'm assuming that your xCP App Server is running on Windows to make it easy for Windows lovers. If not, you can still use this guide either as is by cloning your xCP App instance on Windows or refactor the path etc. per Linux system. 

TBC
