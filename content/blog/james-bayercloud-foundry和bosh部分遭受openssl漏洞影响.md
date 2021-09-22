+++
id = "33"

title = "[James Bayer]Cloud Foundry和BOSH部分遭受OpenSSL漏洞影响"
description = "James Bayer [jbayer@gopivotal.com](mailto:jbayer@gopivotal.com) greg oehmen (BOSH PM) has put together an excellent explanation on how Cloud Foundry and BOSH stemcells are affected by the OpenSSL heartbleed the CVE."
tags = ["bosh","cloudfoundry","news"]
date = "2014-04-11 10:49:01"
author = "丁轶群"
banner = "img/blogs/33/openssl.jpg"
categories = ["Bosh","Cloudfoundry"]

+++



James Bayer [jbayer@gopivotal.com](mailto:jbayer@gopivotal.com) 

greg oehmen (BOSH PM) has put together an excellent explanation on how Cloud Foundry and BOSH stemcells are affected by the OpenSSL heartbleed the CVE. 

<!--more-->

the short summary is:

*   Ubuntu stemcells based on 10.04 LTS are **NOT VULNERABLE**
*   CentOS stemcells based on CentOS 6.x are **VULNERABLE**

see below for more detail from greg. thanks for putting this together so quickly greg! 

*On Tue, Apr 8, 2014 at 12:15 PM, Greg Oehmen [goehmen@pivotallabs.com](mailto:goehmen@pivotallabs.com) wrote:*

The vulnerability issue related to OpenSSL has recently been exposed. See a full description here: https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-0160 and here: http://heartbleed.com/ 

In terms of the exposure that BOSH 

stemcells are experiencing, here are the OpenSSL versions and their specific exposure: OpenSSL 1.0.1 through 1.0.1f (inclusive) are vulnerable OpenSSL 1.0.1g is NOT vulnerable OpenSSL 1.0.0 branch is NOT vulnerable OpenSSL 0.9.8 branch is NOT vulnerable **Bug was introduced to OpenSSL in December 2011 and has been out in the wild since OpenSSL release 1.0.1 on 14th of March 2012. OpenSSL 1.0.1g released on 7th of April 2014 fixes the bug.** 

BOSH UBUNTU STEMCELLS 

The BOSH ubuntu stemcell **uses OpenSSL 0.9.8k and thus is not vulnerable.** 

BOSH team currently has stories in the backlog (https://www.pivotaltracker.com/story/show/69022850 and https://www.pivotaltracker.com/story/show/62015812) to migrate to ubuntu 14.04 for both AWS and Vsphere for the new Go agent. The team will be verifying that the new 14.04 stemcells will have OpenSSL 1.0.1g or higher. But again note that current ubuntu stemcells are NOT vulnerable. 

BOSH CENTOS STEMCELLS 

The BOSH centos stemcell **uses OpenSSL 1.0.1e and thus is vulnerable.** 

THe BOSH team currently has a story in the backlog (https://www.pivotaltracker.com/story/show/69106298) to upgrade the centos stemcell as soon as centos.org announces a patched version of centos. I will send out communication when we confirm that a patched version of centos is available to the BOSH team and when a fixed stemcell is GA. 

Best, Greg 

-- Greg Oehmen Cloud Foundry Product Manager - Bosh Pivotal