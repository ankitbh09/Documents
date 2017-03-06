#   Implementing Cloudera Security
##   (Using Kerberos & Sentry )

1 Kerberos Installation
* Prerequisites
* Kerberos Installation
* Create KDC Database
* Add administrators to ACL file
* Add administrators to Kerberos Database
* Start the Kerebros Daemons
* Test KDC Server

2 Kerberos on Cloudera Using Wizard

3 Using Hadoop Services after Kerberos Installation

4 Sentry Service
* Terminologies
* Privilege Model
* Prerequisites for Installing Sentry

5 Sentry Installation

6 Sentry Architecture
* Sentry Components
* Key Concepts
* User Identity and Group Mapping
* Role-Based Access Control
* Unified Authorization

7 Hive SQL Syntax for Use with Sentry
* Column-Level Authorization
* CREATE ROLE Statement
* DROP ROLE Statement
* GRANT ROLE Statement
* REVOKE ROLE Statement
* GRANT <PRIVILEGE> Statement
* REVOKE <PRIVILEGE> Statement
* GRANT <PRIVILEGE> ...WITH GRANT OPTION
* SET ROLE Statement
* SHOW Statement

8 Using GRANT/REVOKE Statements to Match an Existing Policy File

## Kerberos Installation

### Prerequisites

### Kerberos Installation

Please check the prerequisites  for any dependencies before installing Kerberos
Assuming We already have krb5-libs and krb5-workstation packages installed. Let’s install server because we want to make a KDC server. 

```bash
 yum install krb5-server
```
![alt text](https://github.com/ankitbh09/Documents/blob/master/images/Install%20Kerberos.png)

Note:- Yum is smart enough to update any existing dependent packages also.

Configure Kerberos

Once Kerberos is installed, we need to make changes to the following configuration files in Kerberos.
*	/etc/krb5.conf
*	/var/kerberos/krb5kdc/kdc.conf

We begin with the krb5.conf file The krb5 is used to specify your realm details.

```bash
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = EXAMPLE.COM
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 EXAMPLE.COM = {
   kdc = kerberos.example.com
   admin_server = kerberos.example.com
  }

[domain_realm]
  .example.com = EXAMPLE.COM
  example.com = EXAMPLE.COM

```

