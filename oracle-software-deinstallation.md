Oracle software deinstallation

ls -l "$ORACLE_HOME/deinstall/deinstall"

[svanapal@orion9 ~]$ ls -l "$ORACLE_HOME/deinstall/deinstall"
-rwxr-x---. 1 svanapal oinstall 11243 Oct 16 20:39 /u01/app/oracle/product/19.0.0/dbhome_1/deinstall/deinstall


Run the deinstallation check
echo "ORACLE_HOME=$ORACLE_HOME"

[svanapal@orion9 ~]$ echo "ORACLE_HOME=$ORACLE_HOME"
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1


Now run the deinstallation utility in check-only mode:
"$ORACLE_HOME/deinstall/deinstall" -checkonly

[svanapal@orion9 ~]$ "$ORACLE_HOME/deinstall/deinstall" -checkonly
Checking for required files and bootstrapping ...
Please wait ...
Location of logs /tmp/deinstall2026-10-21_10-57-53AM/logs/

############ ORACLE DECONFIG TOOL START ############


######################### DECONFIG CHECK OPERATION START #########################
## [START] Install check configuration ##


Checking for existence of the Oracle home location /u01/app/oracle/product/19.0.0/dbhome_1
Oracle Home type selected for deinstall is: Oracle Single Instance Database
Oracle Base selected for deinstall is: /u01/app/oracle
Checking for existence of central inventory location /u01/app/oraInventory
ERROR: null
[svanapal@orion9 ~]$


The check-only run has started correctly and identified the expected Oracle configuration:
- Oracle Home: /u01/app/oracle/product/19.0.0/dbhome_1
- Installation type: Oracle Single Instance Database
- Oracle Base: /u01/app/oracle
- Central inventory: /u01/app/oraInventory
However, it stopped at the inventory check with:
Checking for existence of central inventory location /u01/app/oraInventory
ERROR: null


Do not run the actual deinstallation yet. This error does not establish that the inventory is corrupt; we need to inspect the inventory pointer, permissions, and detailed deinstallation logs before deciding what to fix.
Step 3.2 — Diagnose the inventory error
Run these commands on orion9 as svanapal. They are read-only and will not modify the installation.
A. Check the Oracle inventory pointer

[svanapal@orion9 ~]$ echo "===== INVENTORY POINTER ====="
cat /etc/oraInst.loc

echo "===== INVENTORY DIRECTORY ====="
ls -ld /u01/app/oraInventory

echo "===== INVENTORY XML DIRECTORY ====="
ls -ld /u01/app/oraInventory/ContentsXML

echo "===== INVENTORY FILES ====="
ls -l /u01/app/oraInventory/ContentsXML/
===== INVENTORY POINTER =====
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
===== INVENTORY DIRECTORY =====
drwxrwx---. 4 svanapal oinstall 78 Oct 21 10:58 /u01/app/oraInventory
===== INVENTORY XML DIRECTORY =====
drwxrwx---. 2 svanapal oinstall 81 Oct 16 20:39 /u01/app/oraInventory/ContentsXML
===== INVENTORY FILES =====
total 16
-rw-rw----. 1 svanapal oinstall 300 Oct 16 20:39 comps.xml
-rw-rw----. 1 svanapal oinstall 478 Oct 16 20:39 inventory.xml
-rw-rw----. 1 svanapal oinstall 292 Oct 16 20:39 libs.xml
-rw-rw----. 1 svanapal oinstall 174 Oct 16 20:39 oui-patch.xml
[svanapal@orion9 ~]$


Inspect the deinstallation logs:
echo "===== DEINSTALLATION LOG FILES ====="
find /tmp/deinstall2026-10-21_10-57-53AM/logs \
  -maxdepth 2 -type f -print

echo "===== ERROR MESSAGES ====="
grep -RniE 'ERROR|Exception|inventory|null|failed' \
  /tmp/deinstall2026-10-21_10-57-53AM/logs

[svanapal@orion9 ~]$ echo "===== DEINSTALLATION LOG FILES ====="
find /tmp/deinstall2026-10-21_10-57-53AM/logs \
  -maxdepth 2 -type f -print

echo "===== ERROR MESSAGES ====="
grep -RniE 'ERROR|Exception|inventory|null|failed' \
  /tmp/deinstall2026-10-21_10-57-53AM/logs
===== DEINSTALLATION LOG FILES =====
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.out
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.err
===== ERROR MESSAGES =====
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.out:20:Checking for existence of central inventory location /u01/app/oraInventory
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.out:21:ERROR: null
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.err:1:java.lang.reflect.InvocationTargetException
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.err:8:Caused by: java.lang.ExceptionInInitializerError
/tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.err:13:Caused by: java.lang.NullPointerException
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$ echo "===== INVENTORY REGISTRATION ====="
grep -nE 'HOME NAME|LOC=' \
  /u01/app/oraInventory/ContentsXML/inventory.xml
===== INVENTORY REGISTRATION =====
11:<HOME NAME="OraDB19Home1" LOC="/u01/app/oracle/product/19.0.0/dbhome_1" TYPE="O" IDX="1"/>
[svanapal@orion9 ~]$

The inventory XML confirms that your Oracle Home is registered as OraDB19Home1 at the expected path. That is useful evidence, but it does not yet prove the inventory pointer and permissions are correct.

The deinstaller's error log also shows a Java NullPointerException during initialization. We need the full stack trace to identify the failing component rather than changing inventory files speculatively.
Step 3.3 — Capture the complete error and verify the pointer
Please run the following commands as svanapal.

[svanapal@orion9 ~]$ echo "===== INVENTORY REGISTRATION ====="
grep -nE 'HOME NAME|LOC=' \
  /u01/app/oraInventory/ContentsXML/inventory.xml
===== INVENTORY REGISTRATION =====
11:<HOME NAME="OraDB19Home1" LOC="/u01/app/oracle/product/19.0.0/dbhome_1" TYPE="O" IDX="1"/>
[svanapal@orion9 ~]$ echo "===== FULL DEINSTALL ERROR ====="
cat /tmp/deinstall2026-10-21_10-57-53AM/logs/deinstall_deconfig2026-10-21_10-58-59-AM.err

echo "===== INVENTORY POINTER ====="
cat /etc/oraInst.loc

echo "===== INVENTORY PERMISSIONS ====="
ls -ld /u01/app/oraInventory
ls -ld /u01/app/oraInventory/ContentsXML
ls -l /u01/app/oraInventory/ContentsXML/inventory.xml

echo "===== INVENTORY REGISTRATION ====="
cat /u01/app/oraInventory/ContentsXML/inventory.xml
===== FULL DEINSTALL ERROR =====
java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at oracle.install.db.deinstall.wrapper.Deinstall.callCheckImpl(Deinstall.java:1449)
        at oracle.install.db.deinstall.wrapper.Deinstall.main(Deinstall.java:732)
Caused by: java.lang.ExceptionInInitializerError
        at oracle.ops.verification.framework.util.Assert.assertPath(Assert.java:601)
        at oracle.cluster.verification.ClusterVerification.setPreReqXmlFile(ClusterVerification.java:9059)
        at oracle.install.db.deinstall.core.CheckInstall.checkConfig(CheckInstall.java:1083)
        ... 6 more
Caused by: java.lang.NullPointerException
        at oracle.ops.verification.framework.storage.StorageUtil.<clinit>(StorageUtil.java:194)
        ... 9 more
===== INVENTORY POINTER =====
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
===== INVENTORY PERMISSIONS =====
drwxrwx---. 4 svanapal oinstall 78 Oct 21 10:58 /u01/app/oraInventory
drwxrwx---. 2 svanapal oinstall 81 Oct 16 20:39 /u01/app/oraInventory/ContentsXML
-rw-rw----. 1 svanapal oinstall 478 Oct 16 20:39 /u01/app/oraInventory/ContentsXML/inventory.xml
===== INVENTORY REGISTRATION =====
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



Thanks. The latest output gives us a clearer picture.
What we have established
Check	Result
Oracle Home registration	OraDB19Home1 is present in inventory.xml.
Inventory pointer	/etc/oraInst.loc points to /u01/app/oraInventory.
Inventory ownership	Owned by svanapal:oinstall.
Deinstaller failure	A Java NullPointerException occurs inside oracle.ops.verification.framework.storage.StorageUtil.
The pointer and inventory registration are internally consistent. The error occurs inside Oracle's verification framework, before the deinstallation check completes.

I cannot confirm the root cause from this stack trace alone. In particular, we should not conclude that the inventory is corrupt or that a permission change will fix it.


Check the installed Oracle version and patch level:

[svanapal@orion9 ~]$ echo "===== ORACLE DATABASE VERSION ====="
"$ORACLE_HOME/bin/sqlplus" -V

echo "===== OPATCH VERSION ====="
"$ORACLE_HOME/OPatch/opatch" version

echo "===== INSTALLED PATCHES ====="
"$ORACLE_HOME/OPatch/opatch" lsinventory

echo "===== DEINSTALLER HELP ====="
"$ORACLE_HOME/deinstall/deinstall" -help
===== ORACLE DATABASE VERSION =====

SQL*Plus: Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

===== OPATCH VERSION =====

OPatch Version: 12.2.0.1.45

OPatch succeeded.
===== INSTALLED PATCHES =====
Oracle Interim Patch Installer version 12.2.0.1.45
Copyright (c) 2026, Oracle Corporation.  All rights reserved.


Oracle Home       : /u01/app/oracle/product/19.0.0/dbhome_1
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/19.0.0/dbhome_1/oraInst.loc
OPatch version    : 12.2.0.1.45
OUI version       : 12.2.0.7.0
Log file location : /u01/app/oracle/product/19.0.0/dbhome_1/cfgtoollogs/opatch/opatch2026-10-21_11-04-15AM_1.log

Lsinventory Output file location : /u01/app/oracle/product/19.0.0/dbhome_1/cfgtoollogs/opatch/lsinv/lsinventory2026-10-21_11-04-15AM.txt
--------------------------------------------------------------------------------
Local Machine Information::
Hostname: orion9.sunilit.org
ARU platform id: 226
ARU platform description:: Linux x86-64

Installed Top-level Products (1):

Oracle Database 19c                                                  19.0.0.0.0
There are 1 products installed in this Oracle Home.


Interim patches (2) :

Patch  37260974     : applied on Fri Oct 16 20:30:01 IST 2026
Unique Patch ID:  26040769
Patch description:  "Database Release Update : 19.26.0.0.250121 (37260974)"
   Created on 20 Jan 2025, 04:30:30 hrs UTC
   Bugs fixed:
     10121473, 10123661, 1297945, 14570574, 14735102, 15878434, 15931756
     15959416, 16662822, 16664572, 16750494, 17275499, 17395507, 17428816
     17468475, 17562236, 17777718, 18534283, 18697534, 18943905, 19080742
     19138896, 19147630, 19226753, 19630878, 19697993, 19884953, 19958239
     20007421, 20044584, 20083476, 20289608, 20313356, 20319830, 20479545
     20661314, 20721183, 20867658, 20922160, 21119541, 21217621, 21232786
......................................................................
......................................................................
Patch  29585399     : applied on Thu Apr 18 12:51:33 IST 2019
Unique Patch ID:  22840393
Patch description:  "OCW RELEASE UPDATE 19.3.0.0.0 (29585399)"
   Created on 9 Apr 2019, 19:12:47 hrs PST8PDT
   Bugs fixed:
     27222128, 27572040, 27604329, 27760043, 27877830, 28302580, 28470673
     28621543, 28642469, 28699321, 28710663, 28755846, 28772816, 28785321
     28800508, 28808652, 28815557, 28847541, 28847572, 28870496, 28871040
     28874416, 28877252, 28881191, 28881848, 28888083, 28911140, 28925250
     28925460, 28935956, 28940472, 28942694, 28951332, 28963036, 28968779
     28980448, 28995287, 29003207, 29003617, 29016294, 29018680, 29024876
     29026154, 29027933, 29047127, 29052850, 29058476, 29111631, 29112455
     29117337, 29123444, 29125708, 29125786, 29129476, 29131772, 29132456
     29139727, 29146157, 29147849, 29149170, 29152603, 29152752, 29154631
     29154636, 29154829, 29159216, 29159661, 29160462, 29161923, 29169540
     29169739, 29170717, 29173618, 29181568, 29182920, 29183298, 29186091
     29191827, 29201143, 29201695, 29209545, 29210577, 29210610, 29210624
     29210683, 29213641, 29219627, 29224294, 29225861, 29229839, 29235934
     29242906, 29243749, 29244495, 29244766, 29244968, 29248723, 29249583
     29251564, 29255616, 29260224, 29261695, 29271019, 29273360, 29282090
     29282666, 29285453, 29285621, 29290235, 29292232, 29293806, 29294753
     29299830, 29307090, 29307109, 29311336, 29329675, 29330791, 29339299
     29357821, 29360467, 29360775, 29367971, 29368725, 29379299, 29379381
     29380527, 29381000, 29382296, 29391301, 29393649, 29402110, 29411931
     29413360, 29457319, 29465047, 3



--------------------------------------------------------------------------------

OPatch succeeded.
===== DEINSTALLER HELP =====

deinstall
               [ -silent <-checkonly | -paramfile <complete path of input parameter properties file>> ]
               [ -checkonly ]
               [ -local ]
               [ -paramfile <complete path of input parameter properties file> ]
               [ -params <name1=value[ name2=value name3=value ...]> ]
               [ -o <complete path of directory for saving files> ]
               [ -tmpdir <complete path of temporary directory to use> ]
               [ -logdir <complete path of log directory to use> ]
               [ -skipLocalHomeDeletion : To retain the oracle home on local node. Supported only for GI or SIHA Home. ]
               [ -skipRemoteHomeDeletion : To retain the oracle home on remote node(s). Supported only for GI or SIHA Home. ]
               [ -help : Display this help message. ]
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$


Check the operating-system and deinstaller environment:
[svanapal@orion9 ~]$ echo "===== OS RELEASE ====="
cat /etc/os-release

echo "===== JAVA ENVIRONMENT ====="
env | grep -E '^(JAVA_HOME|JAVA_TOOL_OPTIONS|CLASSPATH|PATH|ORACLE_HOME|ORACLE_BASE|ORACLE_SID)='

echo "===== DEINSTALL RESPONSE DIRECTORY ====="
ls -la "$ORACLE_HOME/deinstall/response"

echo "===== TEMP SPACE ====="
df -h /tmp /u01

echo "===== INVENTORY POINTER ====="
cat /etc/oraInst.loc
===== OS RELEASE =====
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
===== JAVA ENVIRONMENT =====
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_SID=orcl
PATH=/u01/app/oracle/product/19.0.0/dbhome_1/bin:/u01/app/oracle/product/19.0.0/dbhome_1/bin:/home/svanapal/.local/bin:/home/svanapal/bin:/usr/share/Modules/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
===== DEINSTALL RESPONSE DIRECTORY =====
total 8
drwxr-xr-x. 2 svanapal oinstall   32 Apr 17  2019 .
drwxr-xr-x. 5 svanapal oinstall  191 Oct 16 20:39 ..
-rw-r-----. 1 svanapal oinstall 5793 Apr 17  2019 deinstall.rsp.tmpl
===== TEMP SPACE =====
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/ol-root   20G   11G  9.5G  53% /
/dev/mapper/ol-u01    20G   11G  9.1G  55% /u01
===== INVENTORY POINTER =====
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
[svanapal@orion9 ~]$






Generate a separate diagnostic log
Oracle's Linux documentation supports specifying a separate output directory and temporary directory for deinstallation. 

Let's try one more non-destructive check, using a fresh log and temporary directory. This will not delete the Oracle Home or database files.
mkdir -p /home/svanapal/deinstall_check
mkdir -p /home/svanapal/deinstall_tmp

"$ORACLE_HOME/deinstall/deinstall" \
  -checkonly \
  -o /home/svanapal/deinstall_check \
  -tmpdir /home/svanapal/deinstall_tmp \
  -logdir /home/svanapal/deinstall_check

  [svanapal@orion9 ~]$ "$ORACLE_HOME/deinstall/deinstall" \
  -checkonly \
  -o /home/svanapal/deinstall_check \
  -tmpdir /home/svanapal/deinstall_tmp \
  -logdir /home/svanapal/deinstall_check
Checking for required files and bootstrapping ...
Please wait ...
Location of logs /home/svanapal/deinstall_check/

############ ORACLE DECONFIG TOOL START ############


######################### DECONFIG CHECK OPERATION START #########################
## [START] Install check configuration ##


Checking for existence of the Oracle home location /u01/app/oracle/product/19.0.0/dbhome_1
Oracle Home type selected for deinstall is: Oracle Single Instance Database
Oracle Base selected for deinstall is: /u01/app/oracle
Checking for existence of central inventory location /u01/app/oraInventory
ERROR: null
[svanapal@orion9 ~]$


[svanapal@orion9 ~]$ echo "===== NEW DEINSTALLATION LOGS ====="
find /home/svanapal/deinstall_check \
  -maxdepth 2 -type f -print

echo "===== ERRORS ====="
grep -RniE 'ERROR|Exception|NullPointerException|failed' \
  /home/svanapal/deinstall_check
===== NEW DEINSTALLATION LOGS =====
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.out
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.err
===== ERRORS =====
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.out:21:ERROR: null
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.err:1:java.lang.reflect.InvocationTargetException
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.err:8:Caused by: java.lang.ExceptionInInitializerError
/home/svanapal/deinstall_check/deinstall_deconfig2026-10-21_11-08-13-AM.err:13:Caused by: java.lang.NullPointerException
[svanapal@orion9 ~]$





