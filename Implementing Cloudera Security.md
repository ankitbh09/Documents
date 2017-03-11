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

2 Kerberos on Cloudera Using Cloudera Manager Wizard

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

## Kerberos on Cloudera Using Cloudera Manager Wizard
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

Once Kerberos is deployed Hadoop-wide using the Cloudera Manager wizard, All Hadoop services can be accessed by Authenticated users only. Any user trying to use a Hadoop service should have a valid ticket from the KDC. To generate a ticket for a user you need to run the following command and provide the password for the principal.

```bash
kinit root/admin@LOCAL
```

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/Using%20Hadoop%20After%20KI.png)

After the above command is run and a valid ticket is issued, root user can access HDFS, Hive and other Hadoop services on the cluster.

For Hadoop user’s such as ‘hdfs’ and ‘hive’ to work, we need to use the keytab files. The keytab files can be found at the path /var/run/cloudera-scm-agent/process/  followed by the directory of the service which you want to access. These keytab files are generated by Cloudera Manager when the principals for the services are being made. The “-kt” flag specifies the kinit command to use the keytab file which is stored at path following it.

```bash
 kinit –kt /var/run/cloudera-scm-agent/process/pid-serverice-role/role.keytab hive@wrkr.local.com@LOCAL.COM
```

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/Using%20Hadoop%20After%20KI_a.png)


## Sentry Service

The Sentry service is a RPC server that stores the authorization metadata in an underlying relational database and provides RPC interfaces to retrieve and manipulate privileges. It supports secure access to services using Kerberos. The service serves authorization metadata from the database backed storage; it does not handle actual privilege validation. The Hive and Impala services are clients of this service and will enforce Sentry privileges when configured to use Sentry.

The motivation behind introducing a new Sentry service is to make it easier to handle user privileges than the existing policy file approach. Providing a database instead, allows you to use the more traditional GRANT/REVOKE statements to modify privileges.

CDH 5.5 introduces column-level access control for tables in Hive and Impala. Previously, Sentry supported privilege granularity only down to a table. Hence, if you wanted to restrict access to a column of sensitive data, the workaround would be to first create view for a subset of columns, and then grant privileges on that view. To reduce the administrative overhead associated with such an approach, Sentry now allows you to assign the SELECT privilege on a subset of columns in a table.

### Terminologies

* An object is an entity protected by Sentry's authorization rules. The objects supported in the current release are server, database, table, and URI.
* A role is a collection of rules for accessing a given Hive object.
* A privilege is granted to a role to govern access to an object. With CDH 5.5, Sentry allows you to assign the SELECT privilege to columns (only for Hive and Impala). Supported privileges are:

Table: Valid privilege types and objects that apply to 

Privilege | Object
---|---
INSERT | DB, TABLE
SELECT | DB, TABLE, COLOUMN
ALL | SERVER, TABLE, DB, URI

* A user is an entity that is permitted by the authentication subsystem to access the Hive service. This entity can be a Kerberos principal, an LDAP userid, or an artifact of some other pluggable authentication system supported by HiveServer2.

* A group connects the authentication system with the authorization system. It is a collection of one or more users who have been granted one or more authorization roles. Sentry allows a set of roles to be configured for a group.

* A configured group provider determines a user’s affiliation with a group. The current release supports HDFS-backed groups and locally configured groups.

### Privilege Model

Sentry uses a role-based privilege model with the following characteristics.

* Allows any user to execute show function, desc function, and show locks.

* Allows the user to see only those tables and databases for which this user has privileges.

* Requires a user to have the necessary privileges on the URI to execute HiveQL operations that take in a location. Examples of such operations include LOAD, IMPORT, and EXPORT.

* Privileges granted on URIs are recursively applied to all subdirectories. That is, privileges only need to be granted on the parent directory.

* CDH 5.5 introduces column-level access control for tables in Hive and Impala. Previously, Sentry supported privilege granularity only down to a table. Hence, if you wanted to restrict access to a column of sensitive data, the workaround would be to first create view for a subset of columns, and then grant privileges on that view. To reduce the administrative overhead associated with such an approach, Sentry now allows you to assign the SELECT privilege on a subset of columns in a table.

### Prerequisites for Installing Sentry

* CDH 5.1.x (or higher) managed by Cloudera Manager 5.1.x (or higher). See the Cloudera Manager Administration Guide and Cloudera Installation and Upgrade for instructions.

* HiveServer2 and the Hive Metastore running with strong authentication. For HiveServer2, strong authentication is either Kerberos or LDAP. For the Hive Metastore, only Kerberos is considered strong authentication.

* Impala 1.4.0 (or higher) running with strong authentication. With Impala, either Kerberos or LDAP can be configured to achieve strong authentication.

* Implement Kerberos authentication on your cluster. For instructions, see Enabling Kerberos Authentication Using the Wizard as described above.

In addition to the Prerequisites above, make sure that the following are true:
*	The Hive warehouse directory (/user/hive/warehouse or any path you specify as hive.metastore.warehouse.dir in your hive-site.xml) must be owned by the Hive user and group.
Permissions on the warehouse directory must be set as follows (see following Note for caveats):
771 on the directory itself (for example, /user/hive/warehouse)
771 on all subdirectories (for example, /user/hive/warehouse/mysubdir)
All files and subdirectories should be owned by hive:hive

For Example:

```bash
$ sudo -u hdfs hdfs dfs -chmod -R 771 /user/hive/warehouse
$ sudo -u hdfs hdfs dfs -chown -R hive:hive /user/hive/warehouse
```

*	HiveServer2 impersonation must be turned off.
*	The Hive user must be able to submit MapReduce jobs. You can ensure that this is true by setting the minimum user ID for job submission to 0. Edit the taskcontroller.cfg file and set min.user.id=0.To enable the Hive user to submit YARN jobs, add the user hive to the allowed.system.users configuration property. Edit the container-executor.cfg file and add hive to the allowed.system.users property. For example,

```bash
allowed.system.users = nobody,impala,hive
```
