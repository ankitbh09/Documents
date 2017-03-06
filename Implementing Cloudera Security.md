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
The first section [logging] is fine for us with default values. Second section [libdefaults] is also fine except first value. We want to change default_realm to our domain name in upper case. Below line shows new value for my domain.
```bash
default_realm = LOCAL
```

Third section [realms] also require changes. Both of the entries should specify the host name of the machine where we installed KDC server. Changed entry for my system is shown below.
```bash
LOCAL = {
kdc = host1.local
admin_server = host1.local
}
```
The last section also requires changes as shown below.
```bash
.local = LOCAL
local = LOCAL
```
The Updated krb5.conf file will be as follows

```bash
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = LOCAL
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 LOCAL = {
   kdc = host1.local
   admin_server = host1.local
  }

[domain_realm]
  .local = LOCAL
  local = LOCAL
```
The kdc.conf file is used to control the listening ports of the KDC as well as realm-specific defaults, the database type and location, and logging. The default values of this file are shown below.

```bash
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88
[realms]
 EXAMPLE.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal }
```

The first section [kdcdefaults] doesn’t require any changes. Second section [realms] need to be modified. The only change we want to make is to change EXAMPLE.COM to LOCAL

```bash
[kdcdefaults]
kdc_ports = 88
 kdc_tcp_ports = 88
[realms]
LOCAL = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal }
```
This completes configuring our realm correctly

### Create KDC Database

We will use the kdb5_util command on the master KDC to create the Kerberos database and the stash file. The stash file is used to store the master key for the KDC database. In order to make KDC database more secure, Kerberos implementation encrypts the database using the master key. Even any database dumps, used as backups are encrypted using this key. It is necessary to know the master key in order to reload them. If we don’t create stash file, the KDC will prompt you for the master key each time it starts up. This means that the KDC will not be able to start automatically, such as after a system reboot.

Lets create a database for our KDC installation.

```bash
kdb5_util create -r LOCAL -s
```
![alt text](https://github.com/ankitbh09/Documents/blob/master/images/Kerberos%20Database.png)

As you can see above, it will ask you for the master key which is associated with the principal K/M@LOCAL. You should remember this key for future use.

### Add administrators to ACL file

Next, we need to add the Kerberos principal of administrators into Kerberos ACL file. This file is used by the kadmind daemon to determine which principals have administrative access to the Kerberos database. The default ACL file is already created as /var/kerberos/krb5kdc/kadm5.acl with a single line as shown below.

```bash
*/admin@EXAMPLE.COM     *
```
This line is good enough for us, we just need to modify realm name to LOCAL. This line gives all privileges to all users with principal instance  admin.

```bash
*/admin@LOCAL     *
```

### Add administrators to the Kerberos database

Now, we need to add administrative principals to the Kerberos database. To do this, we will use the kadmin.local utility. The kadmin.local is designed to be run on the master KDC host without using Kerberos authentication. Notice that kadmin.local is being used, rather than kadmin. This is due to kadmin requiring principals to authenticate before issuing commands and since we do not yet have any principals, the kadmin utility cannot be used at this time. You will be prompted to create a password for this principal.

Let us add an administrator

```bash
[root@host1 ~]# kadmin.local
Authenticating as principal root/admin@LOCAL with password.
kadmin.local:
```
When you enter kadmin.local command, it gives you a prompt as shown above. Enter addprinc command as shown below with a principal name.

```bash
kadmin.local:  addprinc root/admin@LOCAL
```
I am adding root user as an administrator to Kerberos database. It will ask a password for this principal which you must remember for future use.

### Start the Kerberos daemons
There are two Kerberos services which we need to start.
```bash
 service krb5kdc start
 service kadmin start
```
execute below command to make sure that both of the above services are automatically started after reboot.
```bash
 chkconfig krb5kdc on
 chkconfig kadmin on
```
### Test KDC Server

We are finished with the installation. Now we need to test if KDC is correctly issuing tickets.
The klist command shows the list of credentials in the cache. If I issue this command at this stage, it should show an empty list.

```bash
klist
```
![alt text](https://github.com/ankitbh09/Documents/blob/master/images/klist.png)

The kinit command is used to obtain a ticket and store it in credential cache. Let’s try that and recheck klist command.

```bash
[root@host1 ~]# kinit root/admin@LOCAL
Password for root/admin@LOCAL:

[root@host1 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: root/admin@LOCAL

Valid starting     Expires            Service principal
11/25/16 01:00:32  11/26/16 01:00:32  krbtgt/LOCAL@LOCAL
        renew until 11/25/16 01:00:32
```

Great, kinit will ask me for my password and get the ticket from KDC. the klist command shows information about the ticket that I received.

## Kerberos on Cloudera Using Cloudera Manager
Login to Cloudera Manager and navigate to the Administration tab on the top, Select security from the admin tab.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI1.png)

The Security page for Cloudera Manager provides you with the option to Enable Kerberos on the any cluster that is configured with Cloudera Manger.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI2.png)

Step 1: While setting up Kerberos on the cluster, Cloudera recommends some prerequisites which need to be fulfilled and few checklists which are to be completed.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI3.png)

Step 2: Provide Cloudera Manager with KDC information and the KDC Server Host.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI4.png)

Step 3: Check, if you want to manage the krb5.conf through Cloudera Manager.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI4a.png)

Make changes to the krb5.conf as needed 

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI5.png)

Step 4 : Enter the credentials for KDC Account manager.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI6.png)

Step 5: If everything checks outm than CM will successfully import the KDC Account manager credentials.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/Ki7.png)

Step 6: Specify the Kerberos principals used by each service in the cluster.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI8.png)

Step 7: Configure Ports as required.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI9.png)

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI9a.png)

Step 8: The Cluster will be restarted for Kerberos to take effect.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/KI10.png)

Step 9: Kerberos has been successfully installed on the cluster.

## Using Hadoop Services after Kerberos Installation

