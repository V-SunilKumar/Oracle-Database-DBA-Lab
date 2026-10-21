1. Decommissioning sequence

  Phase 1 — Pre-decommission checks
    Confirm the VM, database name, Oracle Home, services, storage, and backup requirements.

  Phase 2 — Decommission the database
    Capture database details, stop services, and remove the database using the appropriate supported method.

  Phase 3 — Decommission Oracle software
    Remove the Oracle Database Home using Oracle's deinstallation tooling and verify the inventory.

  Phase 4 — OS and storage cleanup
    Review Oracle users, groups, kernel settings, directories, disks, and mounts before removing anything.

  Phase 5 — Final verification and documentation
    Confirm the database and software are removed, record evidence, and prepare the VM for reuse.

===================================================================================================================================

Step 1.1 — Confirm the operating system and hostname
Log in as root or use a privileged shell on orion9.
echo "===== HOSTNAME ====="
hostname -f

echo "===== OPERATING SYSTEM ====="
cat /etc/os-release

echo "===== KERNEL ====="
uname -r

echo "===== CURRENT FILESYSTEM USAGE ====="
df -hT

OutPut:
===== HOSTNAME =====
orion9.sunilit.org
===== OPERATING SYSTEM =====
NAME="Oracle Linux Server"
VERSION="9.8"
ID="ol"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="9.8"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Oracle Linux Server 9.8"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:oracle:linux:9:8:server"
HOME_URL="https://linux.oracle.com/"
BUG_REPORT_URL="https://github.com/oracle/oracle-linux"

ORACLE_BUGZILLA_PRODUCT="Oracle Linux 9"
ORACLE_BUGZILLA_PRODUCT_VERSION=9.8
ORACLE_SUPPORT_PRODUCT="Oracle Linux"
ORACLE_SUPPORT_PRODUCT_VERSION=9.8
===== KERNEL =====
6.12.0-203.76.7.3.el9uek.x86_64
===== CURRENT FILESYSTEM USAGE =====
Filesystem          Type      Size  Used Avail Use% Mounted on
devtmpfs            devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs               tmpfs     3.9G     0  3.9G   0% /dev/shm
tmpfs               tmpfs     1.6G   17M  1.6G   2% /run
efivarfs            efivarfs  256K   34K  218K  14% /sys/firmware/efi/efivars
/dev/mapper/ol-root xfs        20G   11G  9.5G  53% /
/dev/mapper/ol-u03  xfs        10G  104M  9.9G   2% /u03
/dev/mapper/ol-u04  xfs       5.0G   68M  4.9G   2% /u04
/dev/sda2           xfs       960M  361M  600M  38% /boot
/dev/sda1           vfat      599M  7.7M  592M   2% /boot/efi
/dev/mapper/ol-u01  xfs        20G   12G  9.0G  56% /u01
/dev/mapper/ol-u02  xfs        50G   11G   40G  21% /u02
Sunil-Tech-HyperLab vboxsf    1.1T  679G  404G  63% /media/sf_Sunil-Tech-HyperLab
tmpfs               tmpfs     792M  4.0K  792M   1% /run/user/1000
tmpfs               tmpfs     792M  4.0K  792M   1% /run/user/0
[svanapal@orion9 ~]$




Identify the Oracle installation owner and database processes:

echo "===== ORACLE PROCESSES ====="
ps -ef | grep -E '[o]ra_pmon_|[o]ra_smon_|[t]nslsnr|[d]bca'

echo "===== ORACLE ACCOUNTS ====="
getent passwd oracle
getent passwd svanapal

echo "===== ORATAB CONFIGURATION ====="
cat /etc/oratab 2>/dev/null

echo "===== ORACLE HOME CONTENTS ====="
ls -ld /u01/app/oracle/product/19.0.0/dbhome_1

==========================================================================================================

===== ORACLE PROCESSES =====
svanapal    1506       1  0 07:33 ?        00:00:00 /u01/app/oracle/product/19.0.0/dbhome_1/bin/tnslsnr LISTENER -inherit
svanapal    6505       1  0 09:22 ?        00:00:00 ora_pmon_orcl
svanapal    6569       1  0 09:22 ?        00:00:00 ora_smon_orcl
===== ORACLE ACCOUNTS =====
svanapal:x:1000:1001:V Sunil Kumar:/home/svanapal:/bin/bash
===== ORATAB CONFIGURATION =====
#



# This file is used by ORACLE utilities.  It is created by root.sh
# and updated by either Database Configuration Assistant while creating
# a database or ASM Configuration Assistant while creating ASM instance.

# A colon, ':', is used as the field terminator.  A new line terminates
# the entry.  Lines beginning with a pound sign, '#', are comments.
#
# Entries are of the form:
#   $ORACLE_SID:$ORACLE_HOME:<N|Y>:
#
# The first and second fields are the system identifier and home
# directory of the database respectively.  The third field indicates
# to the dbstart utility that the database should , "Y", or should not,
# "N", be brought up at system boot time.
#
# Multiple entries with the same $ORACLE_SID are not allowed.
#
#
orcl:/u01/app/oracle/product/19.0.0/dbhome_1:N
===== ORACLE HOME CONTENTS =====
drwxrwxr-x. 73 svanapal oinstall 4096 Oct 20 07:59 /u01/app/oracle/product/19.0.0/dbhome_1

Identify the database name and storage locations :
[svanapal@orion9 ~]$ su - svanapal
Password:
Last login: Wed Oct 21 08:37:09 IST 2026 on pts/1
[svanapal@orion9 ~]$

[svanapal@orion9 ~]$ export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH

echo "===== ORACLE ENVIRONMENT ====="
echo "ORACLE_HOME=$ORACLE_HOME"
echo "ORACLE_SID=$ORACLE_SID"

echo "===== DATABASE CONFIGURATION FILES ====="
ls -l /etc/oratab

echo "===== LISTENER STATUS ====="
lsnrctl status

echo "===== DATABASE INSTANCES ====="
ps -ef | grep '[o]ra_pmon_'

echo "===== ORACLE INVENTORY ====="
cat /u01/app/oraInventory/ContentsXML/inventory.xml
===== ORACLE ENVIRONMENT =====
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_SID=
===== DATABASE CONFIGURATION FILES =====
-rw-rw-r--. 1 svanapal oinstall 788 Oct 21 08:38 /etc/oratab
===== LISTENER STATUS =====

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 21-OCT-2026 10:31:20

Copyright (c) 1991, 2024, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=orion9.sunilit.org)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                20-OCT-2026 07:59:59
Uptime                    1 days 2 hr. 31 min. 22 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19.0.0/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/orion9/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=orion9.sunilit.org)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "5e52ac062ec719ede0635101a8c062f3" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s) for this service...
Service "ORCL" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s) for this service...
Service "orclXDB" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s) for this service...
Service "orclpdb1" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s) for this service...
The command completed successfully
===== DATABASE INSTANCES =====
svanapal    6505       1  0 09:22 ?        00:00:00 ora_pmon_orcl
===== ORACLE INVENTORY =====
<?xml version="1.0" standalone="yes" ?>
<!-- Copyright (c) 1999, 2026, Oracle and/or its affiliates.
All rights reserved. -->
<!-- Do not modify the contents of this file by hand. -->
<INVENTORY>
<VERSION_INFO>
   <SAVED_WITH>12.2.0.7.0</SAVED_WITH>
   <MINIMUM_VER>2.1.0.6.0</MINIMUM_VER>
</VERSION_INFO>
<HOME_LIST>
<HOME NAME="OraDB19Home1" LOC="/u01/app/oracle/product/19.0.0/dbhome_1" TYPE="O" IDX="1"/>
</HOME_LIST>
<COMPOSITEHOME_LIST>
</COMPOSITEHOME_LIST>
</INVENTORY>
[svanapal@orion9 ~]$

[svanapal@orion9 ~]$ . oraenv
ORACLE_SID = [svanapal] ? orcl
The Oracle base has been set to /u01/app/oracle
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Oct 21 10:32:20 2026
Version 19.26.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

SQL> SET LINESIZE 200
SQL> SET PAGESIZE 100
SQL>
SELECT name, dbid, open_mode, database_role
FROM v$database;

SELECT instance_name, status, version
FROM v$instance;

SELECT name, cdb
FROM v$database;

SELECT name
FROM v$datafile;

SELECT member
FROM v$logfile;

SELECT name
FROM v$controlfile;

EXIT;SQL>   2
NAME            DBID OPEN_MODE            DATABASE_ROLE
--------- ---------- -------------------- ----------------
ORCL      1774995996 READ WRITE           PRIMARY

SQL> SQL>   2
INSTANCE_NAME    STATUS       VERSION
---------------- ------------ -----------------
orcl             OPEN         19.0.0.0.0

SQL> SQL>   2
NAME      CDB
--------- ---
ORCL      YES

SQL> SQL>   2
NAME
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
/u02/oradata/ORCL/datafile/o1_mf_system_ofjc6463_.dbf
/u02/oradata/ORCL/datafile/o1_mf_sysaux_ofjc7w2k_.dbf
/u02/oradata/ORCL/datafile/o1_mf_undotbs1_ofjc8tvo_.dbf
/u02/oradata/ORCL/datafile/o1_mf_system_ofjdk1oy_.dbf
/u02/oradata/ORCL/datafile/o1_mf_sysaux_ofjdk1r4_.dbf
/u02/oradata/ORCL/datafile/o1_mf_users_ofjc8w8c_.dbf
/u02/oradata/ORCL/datafile/o1_mf_undotbs1_ofjdk1rg_.dbf
/u02/oradata/ORCL/5E52AC062EC719EDE0635101A8C062F3/datafile/o1_mf_system_ofjfqcyb_.dbf
/u02/oradata/ORCL/5E52AC062EC719EDE0635101A8C062F3/datafile/o1_mf_sysaux_ofjfqcyr_.dbf
/u02/oradata/ORCL/5E52AC062EC719EDE0635101A8C062F3/datafile/o1_mf_undotbs1_ofjfqcyr_.dbf
/u02/oradata/ORCL/5E52AC062EC719EDE0635101A8C062F3/datafile/o1_mf_users_ofjfr49h_.dbf

11 rows selected.

SQL> SQL>   2
MEMBER
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
/u02/oradata/ORCL/onlinelog/o1_mf_3_ofjcb71z_.log
/u02/oradata/ORCL/onlinelog/o1_mf_2_ofjcb5wn_.log
/u02/oradata/ORCL/onlinelog/o1_mf_1_ofjcb5t4_.log

SQL> SQL>   2
NAME
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
/u02/oradata/ORCL/controlfile/o1_mf_ofjcb4tq_.ctl

SQL> SQL>
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$


Remove the database:
Oracle provides DBCA (Database Configuration Assistant) to delete a database and its associated files and configuration. The deletion process is destructive.

Review the database deletion command
dbca -deleteDatabase -sourceDB orcl

DISPLAY not set.
Set DISPLAY environment variable, then re-run.


Verify the Oracle environment
As svanapal, run these commands:
