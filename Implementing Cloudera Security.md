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
## Sentry Installation

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI1.png)
Step 1: Click on the Actions tab and click Add Service.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI2.png)
Step 2: Select the Sentry Service and Click Continue.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI3.png)
Step 3: Configure the Sentry Server and the Gateway nodes for the Cluster and Click Continue.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI4.png)
Step 4: Create a database for sentry to store its metadata and Policy information.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI5.png)
Step 5: Cloudera Manager will run the Start command and complete the perquisites for Sentry.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI6.png)
Step 6: Sentry Service has been successfully installed on the Cluster.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI7.png)
Step 7: Enable Sentry for HDFS, by changing the values of ‘Enable Access Control Lists’ and ‘Enable Sentry Synchronization’ to true. 

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI8.png)
Step 8: Enbale Sentry for Hive by setting the value of ‘Sentry Service = Sentry’ and disabling HiveServer2 Impersonation.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI9.png)
Step 9: Deploy the Client Configurations which have been changed after the above values were modified.

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SI10.png)
Step 10: Restart the Cluster.

## Sentry Architecture

### Sentry Components

![alt text](https://github.com/ankitbh09/Documents/blob/master/images/SA1.png)

There are three components involved in the authorization process:

**Sentry Server**
The Sentry RPC server manages the authorization metadata. It supports interfaces to securely retrieve and manipulate the metadata.

**Data Engine**
This is a data processing application such as Hive or Impala that needs to authorize access to data or metadata resources. The data engine loads the Sentry plugin and all client requests for accessing resources are intercepted and routed to the Sentry plugin for validation.

**Sentry Plugin**
The Sentry plugin runs in the data engine. It offers interfaces to manipulate authorization metadata stored in the Sentry server, and includes the authorization policy engine that evaluates access requests using the authorization metadata retrieved from the server.

### Key Concepts

* Authentication - Verifying credentials to reliably identify a user
* Authorization - Limiting the user’s access to a given resource
* User - Individual identified by underlying authentication system
* Group - A set of users, maintained by the authentication system
* Privilege - An instruction or rule that allows access to an object
* Role - A set of privileges; a template to combine multiple access rules
* Authorization models - Defines the objects to be subject to authorization rules and the granularity of actions allowed. For example, in the SQL model, the objects can be databases or tables, and the actions are SELECT, INSERT, CREATE and so on. For the Search model, the objects are indexes, collections and documents; the access modes are query, update and so on.

### User Identity Group Mapping

Sentry relies on underlying authentication systems, such as Kerberos or LDAP, to identify the user. It also uses the Group mapping mechanism configured in Hadoop to ensure that Sentry sees the same group mapping as other components of the Hadoop ecosystem.

Consider a sample organization with users Alice and Bob who belong to an Active Directory (AD) group called finance-department. Bob also belongs to a group called finance-managers. In Sentry, you first create roles and then grant privileges to these roles. For example, you can create a role called Analyst and grant SELECT on tables Customer and Sales to this role.

The next step is to join these authentication entities (users and groups) to authorization entities (roles). This can be done by granting the Analyst role to the finance-department group. Now Bob and Alice who are members of the finance-department group get SELECT privilege to the Customer and Sales tables.

### Role-Based Access Control

Role-based access control (RBAC) is a powerful mechanism to manage authorization for a large set of users and data objects in a typical enterprise. New data objects get added or removed, users join, move, or leave organisations all the time. RBAC makes managing this a lot easier. Hence, as an extension of the sample organization discussed previously, if a new employee Carol joins the Finance Department, all you need to do is add her to the finance-department group in AD. This will give Carol access to data from the Sales and Customer tables.

### Unified Authorization

Another important aspect of Sentry is the unified authorization. The access control rules once defined, work across multiple data access tools. For example, being granted the Analyst role in the previous example will allow Bob, Alice, and others in the finance-department group to access table data from SQL engines such as Hive and Impala, as well as using MapReduce, Pig applications or metadata access using HCatalog.

## Hive SQL Syntax for Use with Sentry

Sentry permissions can be configured through Grant and Revoke statements issued either interactively or programmatically through the HiveServer2 SQL command line interface, Beeline. The syntax described below is very similar to the GRANT/REVOKE commands available in well-established relational database systems.

### Column-Level Authorization

CDH 5.5 introduces column-level access control for tables in Hive and Impala. Previously, Sentry supported privilege granularity only down to a table. Hence, if you wanted to restrict access to a column of sensitive data, the workaround would be to first create view for a subset of columns, and then grant privileges on that view. To reduce the administrative overhead associated with such an approach, Sentry now allows you to assign the SELECT privilege on a subset of
columns in a table. 

The following command grants a role the SELECT privilege on a column:
```sql
GRANT SELECT(column_name) ON TABLE table_name TO ROLE role_name;
```

The following command can be used to revoke the SELECT privilege on a column:
```sql
REVOKE SELECT(column_name) ON TABLE table_name FROM ROLE role_name;
```

Any new columns added to a table will be inaccessible by default, until explicitly granted access.

**Actions allowed for users with SELECT privilege on a column:**

Users whose roles have been granted the SELECT privilege on columns only, can perform operations which explicitly refer to those columns. Some examples are:
```sql
SELECT column_name FROM TABLE table_name;
```

In this case, Sentry will first check to see if the user has the required privileges to access the table. It will then further check to see whether the user has the SELECT privilege to access the column(s).
```sql
SELECT COUNT(column_name) FROM TABLE table_name;
```

Users are also allowed to use the COUNT function to return the number of values in the column.
```sql
SELECT column_name FROM TABLE table_name WHERE column_name <operator> GROUP BY column_name;
```

The above command will work as long as you refer only to columns to which you already have access.

To list the column(s) to which the current user has SELECT access: 
```sql
SHOW COLUMNS;
```

**Exceptions:**

* If a user has SELECT access to all columns in a table, the following command will work. Note that this is an exception, not the norm. In all other cases, SELECT on all columns does not allow you to perform table-level operations.
```sql
SELECT * FROM TABLE table_name;
```

* The DESCRIBE table command differs from the others, in that it does not filter out columns for which the user does not have SELECT access.

```sql
DESCRIBE (table_name);
```

**Limitations:**

* Column-level privileges can only be applied to tables and partitions, not views.

* HDFS-Sentry Sync: With HDFS-Sentry sync enabled, even if a user has been granted access to all columns of a table, they will not have access to the corresponding HDFS data files. This is because Sentry does not consider SELECT on all columns equivalent to explicitly being granted SELECT on the table.

* Column-level access control for access from Spark SQL is not supported by the HDFS-Sentry plug-in.

### CREATE ROLE Statement

The CREATE ROLE statement creates a role to which privileges can be granted. Privileges can be granted to roles, which can then be assigned to users. A user that has been assigned a role will only be able to exercise the privileges of that role.

Only users that have administrative privileges can create/drop roles. By default, the hive, impala and hue users have admin privileges in Sentry.

```sql
CREATE ROLE [role_name];
```

### DROP ROLE Statement

The DROP ROLE statement can be used to remove a role from the database. Once dropped, the role will be revoked for all users to whom it was previously assigned. Queries that are already executing will not be affected. However, since Hive checks user privileges before executing each query, active user sessions in which the role has already been enabled will be affected.

```sql
DROP ROLE [role_name];
```

### GRANT ROLE Statement

The GRANT ROLE statement can be used to grant roles to groups. Only Sentry admin users can

```sql
GRANT ROLE role_name [, role_name]
TO GROUP <groupName> [,GROUP <groupName>]
```

### REVOKE ROLE Statement

The REVOKE ROLE statement can be used to revoke roles from groups. Only Sentry admin users can revoke the role from a group.

```sql
REVOKE ROLE role_name [, role_name]
FROM GROUP <groupName> [,GROUP <groupName>]
```

### GRANT <PRIVILEGE> Statement

In order to grant privileges on an object to a role, the user must be a Sentry admin user.

```sql
GRANT
<PRIVILEGE> [, <PRIVILEGE> ]
ON <OBJECT> <object_name>
TO ROLE <roleName> [,ROLE <roleName>]
```

With CDH 5.5, you can grant the SELECT privilege on specific columns of a table. For example:

```sql
GRANT SELECT(column_name) ON TABLE table_name TO ROLE role_name;
```

### REVOKE <PRIVILEGE> Statement

Since only authorized admin users can create roles, consequently only Sentry admin users can revoke privileges from a group.

```sql
REVOKE
<PRIVILEGE> [, <PRIVILEGE> ]
ON <OBJECT> <object_name>
FROM ROLE <roleName> [,ROLE <roleName>]
```

You can also revoke any previously-granted SELECT privileges on specific columns of a table. For example:

```sql
REVOKE SELECT(column_name) ON TABLE table_name FROM ROLE role_name;
```

### GRANT <PRIVILEGE> ... WITH GRANT OPTION

With CDH 5.2, you can delegate granting and revoking privileges to other roles. For example, a role that is granted a privilege WITH GRANT OPTION can GRANT/REVOKE the same privilege to/from other roles. Hence, if a role has the ALL privilege on a database and the WITH GRANT OPTION set, users granted that role can execute GRANT/REVOKE statements only for that database or child tables of the database.

```sql
GRANT
<PRIVILEGE>
ON <OBJECT> <object_name>
TO ROLE <roleName>
WITH GRANT OPTION
```

Only a role with GRANT option on a specific privilege or its parent privilege can revoke that privilege from other roles.

Once the following statement is executed, all privileges with and without grant option are revoked.

```sql
REVOKE
<PRIVILEGE>
ON <OBJECT> <object_name>
FROM ROLE <roleName>
```

Hive does not currently support revoking only the WITH GRANT OPTION from a privilege previously granted to a role. To remove the WITH GRANT OPTION, revoke the privilege and grant it again without the WITH GRANT OPTION flag.

### SET ROLE Statement

The SET ROLE statement can be used to specify a role to be enabled for the current session. A user can only enable a role that has been granted to them. Any roles not listed and not already enabled are disabled for the current session.
If no roles are enabled, the user will have the privileges granted by any of the roles that (s)he belongs to.

* To enable a specific role:
```sql
SET ROLE <roleName>;
```

* To enable all roles:
```sql
SET ROLE ALL;
```

* No roles enabled:
```sql
SET ROLE NONE;
```

### SHOW STATMENT

*  To list the database(s) for which the current user has database, table, or column-level access:
```sql
SHOW DATABASES;
```

* To list the table(s) for which the current user has table or column-level access:
```sql
SHOW TABLES;
```

*  To list the column(s) to which the current user has SELECT access:
```sql
SHOW COLUMNS;
```

* To list all the roles in the system (only for sentry admin users):
```sql
SHOW ROLES;
```

* To list all the roles in effect for the current user session:
```sql
SHOW CURRENT ROLES;
```

*  To list all the roles assigned to the given <groupName> (only allowed for Sentry admin users and others users that are part of the group specified by <groupName>):
```sql
SHOW ROLE GRANT GROUP <groupName>;
```

* The SHOW statement can also be used to list the privileges that have been granted to a role or all the grants given to a role for a particular object.
To list all the grants for the given <roleName> (only allowed for Sentry admin users and other users that have
been granted the role specified by <roleName>). The following command will also list any column-level privileges:
```sql
SHOW GRANT ROLE <roleName>;
```

* To list all the grants for a role on the given <objectName> (only allowed for Sentry admin users and other users that have been granted the role specified by <roleName>). The following command will also list any column-level privileges:
```sql
SHOW GRANT ROLE <roleName> on OBJECT <objectName>;
```

## Using Grant/Revoke Statements to Match an Existing Policy File

**Note: In the following example(s), server1 refers to an alias Sentry uses for the associated Hive service. It does not refer to any physical server. This alias can be modified using the hive.sentry.server property in hive-site.xml. If you are using Cloudera Manager, modify the Hive property, Server Name for Sentry Authorization, in the Service-Wide > Advanced category.**

Here is a sample policy file:

```bash

[groups]
# Assigns each Hadoop group to its set of roles
manager = analyst_role, junior_analyst_role
analyst = analyst_role
jranalyst = junior_analyst_role
customers_admin = customers_admin_role
admin = admin_role

[roles] # The uris below define a define a landing skid which
# the user can use to import or export data from the system.
# Since the server runs as the user "hive" files in that directory
# must either have the group hive and read/write set or
# be world read/write.
analyst_role = server=server1->db=analyst1, \
server=server1->db=jranalyst1->table=*->action=select
server=server1->uri=hdfs://ha-nn-uri/landing/analyst1 
junior_analyst_role = server=server1->db=jranalyst1, \
server=server1->uri=hdfs://ha-nn-uri/landing/jranalyst1

# Implies everything on server1.
admin_role = server=server1
```

The following sections show how you can use the new GRANT statements to assign privileges to roles (and assign roles to groups) to match the sample policy file above.

**Grant privileges to admin_role:**

```sql
CREATE ROLE admin_role
GRANT ALL ON SERVER server1 TO ROLE admin_role;
```

**Grant privileges to analyst_role:**
```sql
CREATE ROLE analyst_role;
GRANT ALL ON DATABASE analyst1 TO ROLE analyst_role;
GRANT SELECT ON DATABASE jranalyst1 TO ROLE analyst_role;
GRANT ALL ON URI 'hdfs://ha-nn-uri/landing/analyst1' \
TO ROLE analyst_role;
```

**Grant privileges to junior_analyst_role:**
```sql 
CREATE ROLE junior_analyst_role;
GRANT ALL ON DATABASE jranalyst1 TO ROLE junior_analyst_role;
GRANT ALL ON URI 'hdfs://ha-nn-uri/landing/jranalyst1' \
TO ROLE junior_analyst_role;
```

**GRANT roles to group:**
```sql
GRANT ROLE admin_role TO GROUP admin;
GRANT ROLE analyst_role TO GROUP analyst;
GRANT ROLE jranalyst_role TO GROUP jranalyst;
```
