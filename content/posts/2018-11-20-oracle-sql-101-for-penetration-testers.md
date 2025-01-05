+++
date = "2018-11-20T00:00:00+00:00"
title = "Oracle SQL 101 for penetration testers"
note = "When I wrote this many years ago, there were already Oracle SQL injection cheatsheets. However, achieving remote code execution was still challenging and there were many other undocumented features. Linked databases provided a huge additional attack surface back then and it felt relevant to document them a bit."
toc = false
+++

### Why

There are plenty of cheatsheets and documentation for Oracle SQL Injections. There have also been countless presentations and research on the subject. However, the material is very sparse, mostly outdated, and far from exhaustive.  
As a penetration tester, I have to deal daily with Oracle DBMS, both via SQL Injections, direct connections, or access to the hosting machine.

### Basic Information

Most of the time, Oracle RDMS runs on Linux, specifically RedHat or Oracle Linux. The most common versions found in the wild are 9 (extremely old), 10, and 11, with 12 being seen in the best cases.  
Oracle has an official client called `sqlplus`. Sometimes it is extremely useful to have `sqlplus` and the import/export utilities ready in standalone packages. [Look here for a standalone copy](https://git.lsd.cat/g/oracle-101/src/branch/master/stuff).
Oracle instances are defined in a file called `tnsnames.ora`, where an instance name is associated with a connection string.

Example `tnsnames.ora`:

{{< highlight text >}}
PROD_DB = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP) (HOST = 10.50.50.10) (PORT = 1521)) (CONNECT_DATA = (SID = PROD)))
PREPROD_DB = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP) (HOST = 10.50.50.10) (PORT = 1522)) (CONNECT_DATA = (SID = PREPROD)))
DEV_DB = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP) (HOST = 10.20.20.10) (PORT = 1521)) (CONNECT_DATA = (SID = DEVEL)))
{{< /highlight >}}

This file supports failover, load balancing, and many more options. For further information, refer to [Oracle Documentation](https://docs.oracle.com/cd/B28359_01/network.111/b28317/tnsnames.htm#NETRF007).  
Information specified in this file is extremely useful to know the database in use and its specifics. All information in the connection string is required to connect to an Oracle instance.

### Sqlplus

Sqlplus examples:

{{< highlight bash >}}
# Automatic login
sqlplus my_user/my_password@PROD_DB

# Prompt for password
sqlplus myuser@PROD_DB

# Connect to an instance not present in tnsnames.ora
sqlplus my_user@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(Host=192.168.1.10)(Port=1521))(CONNECT_DATA=(SID=REMOTE_SID)))

# In case there are problems with the shell, double quotes can be used
sqlplus my_user@"(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(Host=192.168.1.10)(Port=1521))(CONNECT_DATA=(SID=REMOTE_SID)))"

# For a local instance if logged in with the oracle user
sqlplus "/ as sysdba"
sqlplus "sys as sysdba"
sqlplus my_user/my_password@PROD_DB as sysdba
{{< /highlight >}}

Better formatting:

{{< highlight sql >}}
set pagesize 0;
set linesize 5000;
{{< /highlight >}}

## Import/Export

Oracle uses a proprietary format for both storing data on disk and for the export/import process. While it is technically possible to dump a table through sqlplus, it is often very CPU-intensive and inefficient for large tables. In such cases, Oracle provides two sets of utilities:  
- `imp/exp`  
- `impdp/expdp`  

Both require special privileges, which means that even if you have select privileges on a table, you might not have the privilege to bulk export it. Refer to [Oracle Import/Export Utilities Documentation](https://docs.oracle.com/database/121/SUTIL/GUID-8B6975D3-3BEC-4584-B416-280125EEC57E.htm).  

Now, the old import/export format has been reverse-engineered, and there's a Python script available.

## Password Hashes

Oracle password hashes are stored both inside the database (selectable by privileged users) and on disk.  

{{< highlight text >}}
$ORACLE_HOME/dbs/orapw<sid>             # Unix
%ORACLE_HOME%\database\PWD<sid>.ora     # Windows
{{< /highlight >}}

Example query:

{{< highlight sql >}}
SELECT NAME, PASSWORD FROM SYS.USER$;
{{< /highlight >}}

For more info:  
- [Oracle Password Hashes by Marcel van de Waters](http://marcel.vandewaters.nl/oracle/security/password-hashes)  
- [Changes in Oracle Database 12c Password Hashes](https://www.trustwave.com/en-us/resources/blogs/spiderlabs-blog/changes-in-oracle-database-12c-password-hashes/)

## Recon

Oracle has plenty of system tables and views to keep track of its properties. Some of them have either `USER`, `ALL`, or `DBA` prefixes. What do they mean?

From [SQL Jana Blog](https://sqljana.wordpress.com/2016/12/21/oracle-data-dictionary-views-user-vs-all-vs-dba-views-100-level-basics/):  

- `USER_*`: Lists only the objects owned by the currently logged-in user.  
- `ALL_*`: Lists objects the currently logged-in user has permissions to access.  
- `DBA_*`: Lists all objects unless restricted by a `WHERE` clause.  

Most of the time, the most useful views are those with the `ALL_` prefix, unless the user is already DBA. Queries to `SYS` tables and `DBA_*` views require high privileges.

Oracle default databases to exclude for cleaner results (from [SQLMap Project](https://github.com/sqlmapproject/sqlmap/blob/master/lib/core/settings.py)):

{{< highlight text >}}
('ANONYMOUS', 'APEX_030200', 'APEX_PUBLIC_USER', 'APPQOSSYS', 'BI', 'CTXSYS', 'DBSNMP', 'DIP', 'EXFSYS', 'FLOWS_%', 'FLOWS_FILES', 'HR', 'IX', 'LBACSYS', 'MDDATA', 'MDSYS', 'MGMT_VIEW', 'OC', 'OE', 'OLAPSYS', 'ORACLE_OCM', 'ORDDATA', 'ORDPLUGINS', 'ORDSYS', 'OUTLN', 'OWBSYS', 'PM', 'SCOTT', 'SH', 'SI_INFORMTN_SCHEMA', 'SPATIAL_CSW_ADMIN_USR', 'SPATIAL_WFS_ADMIN_USR', 'SYS', 'SYSMAN', 'SYSTEM', 'WKPROXY', 'WKSYS', 'WK_TEST', 'WMSYS', 'XDB', 'XS$NULL')
{{< /highlight >}}

Check Oracle version:

{{< highlight sql >}}
SELECT BANNER FROM V$VERSION;
SELECT INSTANCE_NAME, HOST_NAME, EDITION, VERSION FROM V$INSTANCE;
SELECT PRODUCT, VERSION FROM PRODUCT_COMPONENT_VERSION;
{{< /highlight >}}

Get the current user:

{{< highlight sql >}}
SELECT USER FROM DUAL;
SELECT USERNAME FROM USER_USERS;
{{< /highlight >}}

Get current user privileges:

{{< highlight sql >}}
SELECT * FROM SESSION_PRIVS;
{{< /highlight >}}

Check if Java is available:

{{< highlight sql >}}
SELECT DBMS_JAVA.GET_OJVM_PROPERTY(PROPSTRING=>'java.version') FROM DUAL;
{{< /highlight >}}

List all users:

{{< highlight sql >}}
SELECT USER, PASSWORD FROM SYS.USER$;
SELECT USERNAME, PASSWORD FROM DBA_USERS;
SELECT USERNAME FROM ALL_USERS;
{{< /highlight >}}

List all tables:

{{< highlight sql >}}
SELECT OWNER, TABLE_NAME FROM ALL_TABLES;
{{< /highlight >}}

List all columns:

{{< highlight sql >}}
SELECT OWNER, TABLE_NAME, COLUMN_NAME FROM ALL_TAB_COLUMNS;
{{< /highlight >}}

Linked instances:

{{< highlight sql >}}
SELECT * FROM SYS.LINK$;
SELECT * FROM DBA_DB_LINKS;
SELECT OWNER, USERNAME, HOST, DB_LINK FROM ALL_DB_LINKS;
{{< /highlight >}}

Password for DB Links may be cleartext, encrypted, or nonexistent depending on the version.

Past queries:

{{< highlight sql >}}
SELECT SQL_TEXT FROM V$SQL;
SELECT SQL_TEXT FROM WRH$_SQLTEXT;
{{< /highlight >}}

List all procedures:

{{< highlight sql >}}
SELECT OWNER, OBJECT_NAME, OBJECT_TYPE FROM ALL_OBJECTS WHERE OBJECT_TYPE IN ('PROCEDURE', 'FUNCTION', 'PACKAGE', 'PACKAGE BODY');
{{< /highlight >}}

Non-system procedures:

{{< highlight sql >}}
SELECT OWNER, OBJECT_NAME, OBJECT_TYPE
FROM ALL_OBJECTS
WHERE
    OBJECT_TYPE IN ('PROCEDURE', 'FUNCTION', 'PACKAGE', 'PACKAGE BODY')
    AND WHERE OWNER NOT IN ('ANONYMOUS', 'APEX_030200', 'APEX_PUBLIC_USER', 'APPQOSSYS', 'BI', 'CTXSYS', 'DBSNMP', 'DIP', 'EXFSYS', 'FLOWS_%', 'FLOWS_FILES', 'HR', 'IX', 'LBACSYS', 'MDDATA', 'MDSYS', 'MGMT_VIEW', 'OC', 'OE', 'OLAPSYS', 'ORACLE_OCM', 'ORDDATA', 'ORDPLUGINS', 'ORDSYS', 'OUTLN', 'OWBSYS', 'PM', 'SCOTT', 'SH', 'SI_INFORMTN_SCHEMA', 'SPATIAL_CSW_ADMIN_USR', 'SPATIAL_WFS_ADMIN_USR', 'SYS', 'SYSMAN', 'SYSTEM', 'WKPROXY', 'WKSYS', 'WK_TEST', 'WMSYS', 'XDB', 'XS$NULL')
{{< /highlight >}}

Get the source code of an object:

{{< highlight sql >}}
SELECT TEXT FROM ALL_SOURCE WHERE NAME = 'MY_PROCEDURE' ORDER BY LINE;
{{< /highlight >}}

For procedures with obfuscated source code, see:  
- [Unwrapper PLSQL](https://github.com/DarkAngelStrike/UnwrapperPLSQL)  
- [Unwrap It](https://www.codecrete.net/UnwrapIt)


## Linked Instances

Linked instances are used very often and are useful to attackers as they extend the attack surface and may allow lateral movement to different networks.

Basic syntax:

{{< highlight sql >}}
SELECT USER FROM DUAL@MY_LINKED_DB;
{{< /highlight >}}

Passwords are stored in cleartext or a reversible format:

{{< highlight sql >}}
SELECT PASSWORDX FROM SYS.LINK$;
{{< /highlight >}}

It is important to note that the whole recon process might be done in every linked databases. The user in the linked database might have higher privileges, a different Oracle version and different procedures and packages. It might be possibile to gain RCE in a linked database and not in the one used as entry point.

Queries to linked database may also be nested which means that it is possible to run queries on a database linked to a database linked to the main instance thus making a 'jump' to a third database. While the following syntax will not work:

{{< highlight sql >}}
SELECT USER FROM DUAL@DB1@DB2; error
{{< /highlight >}}

It is possible to create a view or a synomin to overcome the syntax problem as described here in [this StackOverflow answer](https://stackoverflow.com/questions/2094843/possible-to-link-to-another-database-link).

DB1:
{{< highlight sql >}}
CREATE SYNONYM X FOR MyTable@sqlServerDB
{{< /highlight >}}

DB2:
{{< highlight sql >}}
# (assumes db link to DB1 connects as owner of synonym)
SELECT * from X@DB1
{{< /highlight >}}
