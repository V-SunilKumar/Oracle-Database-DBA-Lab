Oracle Database Configuration Assistant (DBCA) — Create the Oracle Database 19c database on your orion9 VM and with filesystem storage, and the silent response-file method.

Step 1 — Inspect the current database state
Log in as the Oracle software owner, svanapal, and run

export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH"

echo "===== ORACLE ENVIRONMENT ====="
echo "ORACLE_BASE=$ORACLE_BASE"
echo "ORACLE_HOME=$ORACLE_HOME"
which dbca
dbca -version

echo "===== DATABASE PROCESSES ====="
ps -ef | grep '[o]ra_pmon'

echo "===== ORATAB ENTRIES ====="
if [ -f /etc/oratab ]; then
    cat /etc/oratab
else
    echo "/etc/oratab does not exist"
fi

echo "===== EXISTING DATABASE CONFIGURATION FILES ====="
ls -l "$ORACLE_HOME/dbs"/spfile*.ora \
      "$ORACLE_HOME/dbs"/init*.ora \
      "$ORACLE_HOME/dbs"/orapw* 2>/dev/null

echo "===== EXISTING DATABASE DIRECTORIES ====="
ls -ld /u02 /u02/oradata /u03 /u03/backup 2>/dev/null

echo "===== FILESYSTEM CAPACITY ====="
df -hT /u01 /u02 /u03 2>/dev/null

What we'll establish
- Whether DBCA is available from the intended Oracle Home.
- Whether a database instance is already running.
- Whether the previous attempt left configuration files or an /etc/oratab entry.
- Whether the intended filesystem locations exist and have sufficient space.

===============================================================================================================
OutPut:
 ===== ORACLE ENVIRONMENT =====
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
/u01/app/oracle/product/19.0.0/dbhome_1/bin/dbca
DISPLAY not set.
Set DISPLAY environment variable, then re-run.
===== DATABASE PROCESSES =====
===== ORATAB ENTRIES =====
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
===== EXISTING DATABASE CONFIGURATION FILES =====
-rw-r--r--. 1 svanapal oinstall 3079 May 14  2015  /u01/app/oracle/product/19.0.0/dbhome_1/dbs/init.ora
===== EXISTING DATABASE DIRECTORIES =====
drwxr-xr-x. 3 root root 25 Oct 16 20:07 /u02
drwxr-xr-x. 2 root root  6 Oct 15 20:44 /u03
===== FILESYSTEM CAPACITY =====
Filesystem         Type  Size  Used Avail Use% Mounted on
/dev/mapper/ol-u01 xfs    20G   11G  9.4G  53% /u01
/dev/mapper/ol-u02 xfs    10G  4.4G  5.7G  44% /u02
/dev/mapper/ol-u03 xfs    10G  104M  9.9G   2% /u03
===============================================================================================================
Explanation:

Good. The output confirms that no database instance is currently running, /etc/oratab has no database entries, and the previous DBCA attempt has not left an obvious database configuration behind.

There are two items to address before creating the database:
- dbca -version reports DISPLAY not set. This is because DBCA is attempting to initialize its graphical environment. We will use silent mode, so a graphical display should not be required.
- /u02 and /u03 are owned by root:root. We must check and prepare the intended database directories for svanapal before DBCA writes database files.
===============================================================================================================
Step 2 — Prepare and validate filesystem directories

Run the following commands as root or with sudo:

echo "===== CREATE DATABASE DIRECTORIES ====="
sudo mkdir -p /u02/oradata
sudo mkdir -p /u03/backup

echo "===== SET ORACLE OWNERSHIP ====="
sudo chown -R svanapal:oinstall /u02/oradata /u03/backup

echo "===== SET DIRECTORY PERMISSIONS ====="
sudo chmod 750 /u02/oradata /u03/backup

echo "===== VERIFY ====="
ls -ld /u02/oradata /u03/backup
df -hT /u02/oradata /u03/backup

================================================================================================================
output:
[svanapal@orion9 ~]$ echo "===== CREATE DATABASE DIRECTORIES ====="
sudo mkdir -p /u02/oradata
sudo mkdir -p /u03/backup

echo "===== SET ORACLE OWNERSHIP ====="
sudo chown -R svanapal:oinstall /u02/oradata /u03/backup

echo "===== SET DIRECTORY PERMISSIONS ====="
sudo chmod 750 /u02/oradata /u03/backup

echo "===== VERIFY ====="
ls -ld /u02/oradata /u03/backup
df -hT /u02/oradata /u03/backup
===== CREATE DATABASE DIRECTORIES =====
[sudo] password for svanapal:

Sorry, try again.
[sudo] password for svanapal:
Sorry, try again.
[sudo] password for svanapal:
===== SET ORACLE OWNERSHIP =====
===== SET DIRECTORY PERMISSIONS =====
===== VERIFY =====
drwxr-x---. 2 svanapal oinstall 6 Oct 20 07:18 /u02/oradata
drwxr-x---. 2 svanapal oinstall 6 Oct 20 07:18 /u03/backup
Filesystem         Type  Size  Used Avail Use% Mounted on
/dev/mapper/ol-u02 xfs    10G  4.4G  5.7G  44% /u02
/dev/mapper/ol-u03 xfs    10G  104M  9.9G   2% /u03

===============================================================================================================
Explanation:
Good. The important result is that both database directories were created successfully and have the correct ownership and permissions.
Check	Result
/u02/oradata	svanapal:oinstall, permissions 750
/u03/backup	svanapal:oinstall, permissions 750
======================================================================================================================
Confirm the Oracle user's environment
Run as svanapal:
echo "===== ORACLE ENVIRONMENT ====="
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH"

echo "ORACLE_BASE=$ORACLE_BASE"
echo "ORACLE_HOME=$ORACLE_HOME"
id

Then check the DBCA version without requesting the GUI:
"$ORACLE_HOME/bin/dbca" -help | head -40
====================================================================================================
[svanapal@orion9 ~]$ "$ORACLE_HOME/bin/dbca" -help | head -40
Usage:  dbca [<flag>] [<command> <option>]
Following are the possible flags:
        -help
                -contextId <RAC | SI>
        -ignorePreReqs - Ignore prerequisite checks for current operation.
        -ignorePrereqFailure  - Ignore all prerequisites failures.
        -silent - This flag allows you to carry on configuration in silent mode.
                -<command>

Following are the possible commands:
        -addInstance - Command to Add an instance to an admin managed cluster database.
        -configureDatabase - Command to Configure a database.
        -configurePluggableDatabase - Command to Configure a pluggable database.
        -createCloneTemplate - Command to Create a clone template from an existing database.
        -createDatabase - Command to Create a database.
        -createDuplicateDB - Command to Duplicate a database.
        -createPluggableDatabase - Command to Create a pluggable database.
        -createTemplateFromDB - Command to Create a template from an existing database.
        -createTemplateFromTemplate - Command to Create a template from an existing template.
        -deleteDatabase - Command to Delete a database.
        -deleteInstance - Command to Delete an instance from an admin managed cluster database.
        -deletePluggableDatabase - Command to Delete a pluggable database.
        -deleteTemplate - Command to delete a template.
        -executePrereqs - Command to execute prerequisite checks.
        -generateScripts - Command to Generate scripts which can be used to create database.
        -relocatePDB - Command to Relocate a pluggable database.
        -unplugDatabase - Command to Unplug a pluggable database.
====================================================================================================

Explanation :
The most important lines
Line 1
Usage: dbca [<escape><flag></escape>] [<escape><command> <option></escape>]
DBCA accepts flags, commands, and command-specific options.
Line 2
-help
Requests help information.
Line 3
-silent
This is the key option: DBCA can perform configuration operations in silent mode, without requiring the graphical interface.
Line 4
-createDatabase
DBCA has a command specifically for creating a database.
Line 5
-executePrereqs
DBCA can run prerequisite checks for a specified operation.
====================================================================================================

Step 3 — Choose the initial database configuration
Parameter	Value
Database name	ORCL
SID	orcl
Architecture	Single-instance
Storage	Filesystem
Data files	/u02/oradata
Backup directory	/u03/backup
Character set	AL32UTF8
Memory approach	ASMM
Initial SGA target	2 GiB
Archive mode	NOARCHIVELOG

Note: we proposed ORCL as the database name and orcl as the SID. The capitalization is a convention, not a requirement.
Database Name — ORCL : Identifies the database itself. It is associated with the Oracle initialization parameter DB_NAME and is stored in the database control file.
SID — orcl : Identifies the local Oracle instance: the running memory structures (SGA) and background processes that manage the database. On Linux, it is commonly used in ORACLE_SID to select the local instance.

if you use inconsistent values in commands or configuration. For example, if the instance SID is orcl, then:
export ORACLE_SID=orcl
sqlplus / as sysdba

If you mistakenly set:
export ORACLE_SID=ORCL
Linux and Oracle utilities may look for a different local instance, potentially producing errors such as ORA-01034: ORACLE not available.

Oracle's database naming rules and configuration must still be followed. The SID is case-sensitive in Linux process and environment contexts, so you should consistently use the exact SID you created.
***important***
DB_NAME identifies the database, DB_UNIQUE_NAME uniquely identifies a database within a Data Guard configuration or across an enterprise, and ORACLE_SID identifies the local Oracle instance on a server.
==============================================================================================
Step 4 — Validate DBCA creation options
Before creating the database, run this command as svanapal:
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH="$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH"

"$ORACLE_HOME/bin/dbca" -help -createDatabase
==============================================================================================

Step 5 — Validate the DBCA template and filesystem
Run these commands as svanapal. They do not create the database.

export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

echo "===== AVAILABLE DBCA TEMPLATES ====="
ls -lh "$ORACLE_HOME/assistants/dbca/templates/"

echo "===== TEMPLATE DATABASE FILES ====="
ls -lh "$ORACLE_HOME/assistants/dbca/templates/"*.dbt

echo "===== DIRECTORY OWNERSHIP ====="
ls -ld /u02/oradata /u03/backup

echo "===== AVAILABLE SPACE ====="
df -h /u01 /u02 /u03

echo "===== MEMORY ====="
free -h

=========================================================
output:
===== AVAILABLE DBCA TEMPLATES =====
total 363M
-rw-r-----. 1 svanapal oinstall 4.8K Apr 17  2019 Data_Warehouse.dbc
-rw-r-----. 1 svanapal oinstall 4.7K Apr 17  2019 General_Purpose.dbc
-rw-r-----. 1 svanapal oinstall  11K Apr  6  2019 New_Database.dbt
-rw-r-----. 1 svanapal oinstall  83M Apr 17  2019 pdbseed.dfb
-rw-r-----. 1 svanapal oinstall 6.5K Apr 17  2019 pdbseed.xml
-rw-r-----. 1 svanapal oinstall  18M Apr 17  2019 Seed_Database.ctl
-rw-r-----. 1 svanapal oinstall 262M Apr 17  2019 Seed_Database.dfb
===== TEMPLATE DATABASE FILES =====
-rw-r-----. 1 svanapal oinstall 11K Apr  6  2019 /u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/templates/New_Database.dbt
===== DIRECTORY OWNERSHIP =====
drwxr-x---. 2 svanapal oinstall 6 Oct 20 07:18 /u02/oradata
drwxr-x---. 2 svanapal oinstall 6 Oct 20 07:18 /u03/backup
===== AVAILABLE SPACE =====
Filesystem          Size  Used Avail Use% Mounted on
/dev/mapper/ol-u01   20G   11G  9.4G  53% /u01
/dev/mapper/ol-u02   10G  4.4G  5.7G  44% /u02
/dev/mapper/ol-u03   10G  104M  9.9G   2% /u03
===== MEMORY =====
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       463Mi       7.3Gi        16Mi       186Mi       7.3Gi
Swap:          8.0Gi          0B       8.0Gi

===================================================

Step 6 — Validate the selected template
Run this as svanapal:
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

echo "===== GENERAL PURPOSE TEMPLATE ====="
ls -l "$ORACLE_HOME/assistants/dbca/templates/General_Purpose.dbc"

echo "===== EXISTING LISTENERS ====="
lsnrctl status

output :
=====================================================================================
===== GENERAL PURPOSE TEMPLATE =====
-rw-r-----. 1 svanapal oinstall 4768 Apr 17  2019 /u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/templates/General_Purpose.dbc
===== EXISTING LISTENERS =====

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 20-OCT-2026 07:48:53

Copyright (c) 1991, 2024, Oracle.  All rights reserved.

Connecting to (ADDRESS=(PROTOCOL=tcp)(HOST=)(PORT=1521))
TNS-12541: TNS:no listener
 TNS-12560: TNS:protocol adapter error
  TNS-00511: No listener
   Linux Error: 111: Connection refused
=====================================================================================

Step 7 — Configure a listener

First, check whether a listener configuration file already exists:
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

echo "===== LISTENER CONFIGURATION ====="
ls -l "$ORACLE_HOME/network/admin/listener.ora" \
      "$ORACLE_HOME/network/admin/tnsnames.ora" 2>/dev/null

echo "===== NETWORK ADMIN DIRECTORY ====="
ls -ld "$ORACLE_HOME/network/admin"

output:
===== LISTENER CONFIGURATION =====
===== NETWORK ADMIN DIRECTORY =====
drwxr-xr-x. 3 svanapal oinstall 39 Apr 17  2019 /u01/app/oracle/product/19.0.0/dbhome_1/network/admin
==========================================================================================================
Explanataion:
The output confirms that the Oracle Network Admin directory exists, but neither listener.ora nor tnsnames.ora exists. We can create a listener configuration using Oracle Net Configuration Assistant (NETCA).
=======================================================================================================================
Step 8 — Create the listener in silent mode

First inspect the NETCA response file or configuration

[svanapal@orion9 ~]$ grep -n -E 'LISTENER|PORT|PROTOCOL|CREATE' \
  "$ORACLE_HOME/assistants/netca/netca.rsp" | head -80
32:CREATE_TYPE="CUSTOM"
73:#LISTENER_NUMBER;Number;Number of Listeners
75:LISTENER_NUMBER=1
77:#LISTENER_NAMES;StringList;list of listener names
79:# "LISTENER","LISTENER1","LISTENER2","LISTENER3", ...
80:# A typical install sets only "LISTENER"
81:LISTENER_NAMES={"LISTENER"}
83:#LISTENER_PROTOCOLS;StringList;list of listener addresses (protocols and parameters separated by semicolons)
89:LISTENER_PROTOCOLS={"TCP;1521"}
91:#LISTENER_START;String;name of the listener to start, in double quotes
92:LISTENER_START=""LISTENER""
125:#NSN_PROTOCOLS;StringList;list of coma separated strings of Net Service Name protocol parameters
129:NSN_PROTOCOLS={"TCP;HOSTNAME;1521"}

[svanapal@orion9 ~]$ export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/netca" -silent \
  -responseFile "$ORACLE_HOME/assistants/netca/netca.rsp"

Parsing command line arguments:
    Parameter "silent" = true
    Parameter "responsefile" = /u01/app/oracle/product/19.0.0/dbhome_1/assistants/netca/netca.rsp
Done parsing command line arguments.
Oracle Net Services Configuration:
Profile configuration complete.
Oracle Net Listener Startup:
    Running Listener Control:
      /u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl start LISTENER
    Listener Control complete.
    Listener started successfully.
Listener configuration complete.
Oracle Net Services configuration successful. The exit code is 0
[svanapal@orion9 ~]$
=======================================================================================================================
output:
[svanapal@orion9 ~]$ lsnrctl status LISTENER

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 20-OCT-2026 08:00:49

Copyright (c) 1991, 2024, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=orion9.sunilit.org)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                20-OCT-2026 07:59:59
Uptime                    0 days 0 hr. 0 min. 52 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19.0.0/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/orion9/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=orion9.sunilit.org)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
The listener supports no services
The command completed successfully
[svanapal@orion9 ~]$
=======================================================================================================================

step-9 Run DBCA prerequisite checks
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -silent \
  -executePrereqs \
  -databaseConfigType SINGLE \
  -databaseType MULTIPURPOSE \
  -sid orcl \
  -gdbName ORCL \
  -storageType FS \
  -datafileDestination /u02/oradata

Error : [FATAL] java.lang.NullPointerException

  Locate the DBCA diagnostic logs:
  export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

echo "===== DBCA LOG DIRECTORY ====="
ls -ltr "$ORACLE_BASE/cfgtoollogs/dbca/"

echo
echo "===== FIND DBCA TRACE FILES ====="
find "$ORACLE_BASE/cfgtoollogs/dbca" \
  -type f \( -name 'trace.log*' -o -name '*.log' \) \
  -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | sort | tail -15

===== DBCA LOG DIRECTORY =====
total 3088
-rw-r-----. 1 svanapal oinstall 2034668 Oct 16 21:14 trace.log_2026-10-16_09-14-13PM
-rw-r-----. 1 svanapal oinstall     114 Oct 16 21:14 silent.log_2026-10-16_09-14-13PM
-rw-r-----. 1 svanapal oinstall      71 Oct 20 08:03 silent.log_2026-10-20_08-03-16AM
-rw-r-----. 1 svanapal oinstall  555286 Oct 20 08:03 trace.log_2026-10-20_08-03-16AM
-rw-r-----. 1 svanapal oinstall  553266 Oct 20 08:05 trace.log_2026-10-20_08-05-09AM
-rw-r-----. 1 svanapal oinstall      71 Oct 20 08:05 silent.log_2026-10-20_08-05-09AM

===== FIND DBCA TRACE FILES =====
2026-10-16 21:14 /u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-16_09-14-13PM
2026-10-20 08:03 /u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-20_08-03-16AM
2026-10-20 08:05 /u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-20_08-05-09AM

Extract the Java exception:

TRACE="$ORACLE_BASE/cfgtoollogs/dbca/trace.log_2026-10-20_08-05-09AM"

echo "===== EXCEPTION AND STACK TRACE ====="
grep -n -B 15 -A 35 -E \
  'NullPointerException|Exception|ERROR|CRS home undefined' \
  "$TRACE" | tail -160

  [svanapal@orion9 ~]$ tail -100 "$TRACE"
Host key verification failed.
</CV_VAL><CV_VRES>255</CV_VRES><CV_ERES>0</CV_ERES><INITCMD>/tmp/CVU_19.0.0.0.0_svanapal/exectask -runexe /usr/bin/ssh -o FallBackToRsh=no -o PasswordAuthentication=no -o StrictHostKeyChecking=yes -o NumberOfPasswordPrompts=0 orion9 -n /tmp/CVU_19.0.0.0.0_svanapal/exectask.sh -getrlimit svanapal soft stack </INITCMD><HOST>orion9</HOST><ERROR_LOG><ERR_DESC><ENTITY>GLOBAL</ENTITY><ERR_CMD><CMD>/usr/bin/ssh -o FallBackToRsh=no -o PasswordAuthentication=no -o StrictHostKeyChecking=yes -o NumberOfPasswordPrompts=0 orion9 -n /tmp/CVU_19.0.0.0.0_svanapal/exectask.sh -getrlimit svanapal soft stack </CMD><OUT>No ED25519 host key is known for orion9 and you have requested strict checking.
Host key verification failed.
</OUT><STAT>255</STAT><ERR_LOC>CVU00367</ERR_LOC></ERR_CMD></ERR_DESC></ERROR_LOG>)
[main] [ 2026-10-20 08:05:17.834 IST ] [GlobalExecution.getLocalNodeResourceLimit:2157]  Local node output is (No ED25519 host key is known for orion9 and you have requested strict checking.
Host key verification failed.
 )
[main] [ 2026-10-20 08:05:17.834 IST ] [GlobalExecution.getLocalNodeResourceLimit:2160]  CV_VAL for local node is (null)
[main] [ 2026-10-20 08:05:17.834 IST ] [GlobalExecution.getLocalNodeResourceLimit:2195]  Failed to parse the output for local node limit value. Output is (No ED25519 host key is known for orion9 and you have requested strict checking.
Host key verification failed.
 )
[main] [ 2026-10-20 08:05:17.835 IST ] [ResultSet.traceResultTable:1065]

ResultTable Info ===>

        contents of resultTable

Dumping Result data.
  Status     : SUCCESSFUL
  Name       : orion9
  Type       : Node
  Has Results: No
  Exp. value : null
  Act. value : null

  Errors  :



[main] [ 2026-10-20 08:05:17.835 IST ] [TaskShellLimits.isTaskApplicable:291]  Actual resource limit could not be retrieved for the local node. Skipping the check...
[main] [ 2026-10-20 08:05:17.835 IST ] [XmlTaskFactory.createTaskShellLimits:1542]  Not adding Task shell Limit for Soft Limit - Stack as there is no default limit available for the login shell(s) on nodes.
[main] [ 2026-10-20 08:05:17.836 IST ] [CVUVariables.secureVariableValueTrace:776]  getting CVUVariableConstant : VAR = RELEASE VAL = 19.0
SEVERE: Oct 20, 2026 8:05:17 AM oracle.install.commons.util.exception.AbstractErrorAdvisor getDetailedMessage
SEVERE: [FATAL] java.lang.NullPointerException
java.lang.NullPointerException
        at oracle.cluster.verification.constraints.XmlTaskFactory.getTasks(XmlTaskFactory.java:415)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskListSysReq(TaskFactory.java:1217)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getSystemRequirementTasks(TaskFactory.java:986)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskList(TaskFactory.java:760)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskList(TaskFactory.java:648)
        at oracle.cluster.verification.ClusterVerification.getPreReqTasksForDBConfig(ClusterVerification.java:1514)
        at oracle.assistants.dbca.action.PrereqAction.getProductVerificationTasks(PrereqAction.java:112)
        at oracle.install.commons.base.interview.common.action.AbstractPrereqAction.execute(AbstractPrereqAction.java:106)
        at oracle.install.commons.base.interview.common.util.PrereqExecutor.execute(PrereqExecutor.java:110)
        at oracle.install.commons.flow.FlowApplication.executeFlow(FlowApplication.java:161)
        at oracle.assistants.common.base.driver.AssistantApplication.executeFlow(AssistantApplication.java:178)
        at oracle.install.commons.flow.FlowApplication.run(FlowApplication.java:167)
        at oracle.assistants.common.base.driver.AssistantApplication.run(AssistantApplication.java:281)
        at oracle.install.commons.util.Application.startup(Application.java:1131)
        at oracle.install.commons.flow.FlowApplication.startup(FlowApplication.java:181)
        at oracle.install.commons.flow.FlowApplication.startup(FlowApplication.java:198)
        at oracle.assistants.common.base.driver.AssistantApplication.startup(AssistantApplication.java:336)
        at oracle.assistants.dbca.driver.DBConfigurator.startup(DBConfigurator.java:378)
        at oracle.assistants.dbca.driver.DBConfigurator.main(DBConfigurator.java:513)




Refer associated stacktrace #oracle.install.commons.util.exception.AbstractErrorAdvisor:3932
[main] [ 2026-10-20 08:05:17.853 IST ] [AssistantLogHandler.publish:465]  ---# Begin Stacktrace #---------------------------
ID: oracle.install.commons.util.exception.AbstractErrorAdvisor:3932
java.lang.NullPointerException
        at oracle.cluster.verification.constraints.XmlTaskFactory.getTasks(XmlTaskFactory.java:415)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskListSysReq(TaskFactory.java:1217)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getSystemRequirementTasks(TaskFactory.java:986)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskList(TaskFactory.java:760)
        at oracle.ops.verification.framework.engine.task.TaskFactory.getTaskList(TaskFactory.java:648)
        at oracle.cluster.verification.ClusterVerification.getPreReqTasksForDBConfig(ClusterVerification.java:1514)
        at oracle.assistants.dbca.action.PrereqAction.getProductVerificationTasks(PrereqAction.java:112)
        at oracle.install.commons.base.interview.common.action.AbstractPrereqAction.execute(AbstractPrereqAction.java:106)
        at oracle.install.commons.base.interview.common.util.PrereqExecutor.execute(PrereqExecutor.java:110)
        at oracle.install.commons.flow.FlowApplication.executeFlow(FlowApplication.java:161)
        at oracle.assistants.common.base.driver.AssistantApplication.executeFlow(AssistantApplication.java:178)
        at oracle.install.commons.flow.FlowApplication.run(FlowApplication.java:167)
        at oracle.assistants.common.base.driver.AssistantApplication.run(AssistantApplication.java:281)
        at oracle.install.commons.util.Application.startup(Application.java:1131)
        at oracle.install.commons.flow.FlowApplication.startup(FlowApplication.java:181)
        at oracle.install.commons.flow.FlowApplication.startup(FlowApplication.java:198)
        at oracle.assistants.common.base.driver.AssistantApplication.startup(AssistantApplication.java:336)
        at oracle.assistants.dbca.driver.DBConfigurator.startup(DBConfigurator.java:378)
        at oracle.assistants.dbca.driver.DBConfigurator.main(DBConfigurator.java:513)

---# End Stacktrace #-----------------------------

SEVERE: Oct 20, 2026 8:05:17 AM oracle.assistants.common.base.util.AssistantAdvisor logMessage
SEVERE: [ 2026-10-20 08:05:17.853 IST ] [FATAL] java.lang.NullPointerException


INFO: Oct 20, 2026 8:05:17 AM oracle.install.commons.util.exception.AbstractErrorAdvisor advise
INFO: Advice is ABORT


SEVERE: Oct 20, 2026 8:05:17 AM oracle.install.commons.util.exception.DefaultExceptionHandler handleException
SEVERE: Unconditional Exit


INFO: Oct 20, 2026 8:05:17 AM oracle.install.commons.util.ExitStatusSet add
INFO: Adding ExitStatus FAILURE to the exit status set


[svanapal@orion9 ~]$

Verify the SSH host key on orion9:
Run these commands as svanapal:
echo "===== SSH SERVICE ====="
sudo systemctl is-active sshd

echo
echo "===== SSH HOST KEY ====="
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

echo
echo "===== HOSTNAME RESOLUTION ====="
getent hosts orion9

[svanapal@orion9 ~]$ echo "===== SSH SERVICE ====="
sudo systemctl is-active sshd

echo
echo "===== SSH HOST KEY ====="
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

echo
echo "===== HOSTNAME RESOLUTION ====="
getent hosts orion9
===== SSH SERVICE =====
[sudo] password for svanapal:
active

===== SSH HOST KEY =====
256 SHA256:lcBgq8JCyyOhsrH70dDGwyUt9N1wPOKJ99EbRGFLQMg root@orion9 (ED25519)

===== HOSTNAME RESOLUTION =====
fd17:625c:f037:2:a00:27ff:fe08:931b orion9
2401:4900:8fd0:7e75:ef6f:8801:aef2:bcac orion9
fe80::a00:27ff:fe08:931b orion9
fe80::f962:b7c3:be89:e00a orion9


Good. We have confirmed that sshd is active and the VM has an ED25519 host key. The hostname orion9 resolves to IPv6 addresses, which may be relevant to the SSH connection failure.

The DBCA trace specifically says that SSH does not know the host key for orion9. Let's verify the connection without changing any configuration yet.
Step 2 — Test SSH to the local hostname
Run this as svanapal:

[svanapal@orion9 ~]$ echo "===== SSH CONNECTION TEST ====="
ssh -o StrictHostKeyChecking=yes \
    -o BatchMode=yes \
    orion9 'hostname; id'

echo
echo "===== SSH KNOWN-HOSTS ENTRIES ====="
ssh-keygen -F orion9
===== SSH CONNECTION TEST =====
No ED25519 host key is known for orion9 and you have requested strict checking.
Host key verification failed.

===== SSH KNOWN-HOSTS ENTRIES =====
Cannot stat /home/svanapal/.ssh/known_hosts: No such file or directory
[svanapal@orion9 ~]$

That confirms the immediate problem: svanapal has no ~/.ssh/known_hosts file, so SSH cannot verify the identity of orion9 under strict host-key checking. This matches the error recorded in your DBCA trace.

The VM's ED25519 host-key fingerprint that we verified is:
SHA256:lcBgq8JCyyOhsrH70dDGwyUt9N1wPOKJ99EbRGFLQMg

We'll add trust for the local hostname only after confirming the key presented by SSH matches this fingerprint. We will not disable host-key verification.  

Retrieve and verify the SSH host key
Run this as svanapal. This retrieves the ED25519 public host key offered by orion9 without adding it to your trusted keys yet.
ssh-keyscan -t ed25519 orion9 2>/dev/null \
  > /tmp/orion9_hostkey

echo "===== KEY OFFERED BY SSH ====="
ssh-keygen -lf /tmp/orion9_hostkey

[svanapal@orion9 ~]$ ssh-keyscan -t ed25519 orion9 2>/dev/null \
  > /tmp/orion9_hostkey

echo "===== KEY OFFERED BY SSH ====="
ssh-keygen -lf /tmp/orion9_hostkey
===== KEY OFFERED BY SSH =====
256 SHA256:lcBgq8JCyyOhsrH70dDGwyUt9N1wPOKJ99EbRGFLQMg orion9 (ED25519)
[svanapal@orion9 ~]$

Excellent! The SSH host-key fingerprints match exactly. We can now safely add the verified key to svanapal's trusted host list.

Add the verified host key:
Run as svanapal:
mkdir -p ~/.ssh
chmod 700 ~/.ssh

touch ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts

ssh-keygen -F orion9 >/dev/null 2>&1 || \
  cat /tmp/orion9_hostkey >> ~/.ssh/known_hosts
  
Test SSH again:
ssh -o StrictHostKeyChecking=yes \
    -o BatchMode=yes \
    orion9 'hostname; id'
    
svanapal@orion9: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
[svanapal@orion9 ~]$

Explanation:
These are two different stages:
- Host-key verification: Resolved — the server identity matches the fingerprint we verified.
- SSH authentication: Still failing — the client cannot authenticate as svanapal using the available methods without prompting.
For DBCA, we need to determine whether passwordless SSH is required by this particular prerequisite check and whether a usable SSH key already exists.

Check for an existing SSH key:

Run as svanapal:
echo "===== SSH CLIENT KEYS ====="
ls -la ~/.ssh

echo
echo "===== PUBLIC KEY FINGERPRINTS ====="
for key in ~/.ssh/id_ed25519.pub ~/.ssh/id_rsa.pub; do
    if [ -f "$key" ]; then
        ssh-keygen -lf "$key"
    fi
done

echo
echo "===== SSH AUTHENTICATION CONFIGURATION ====="
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|authorizedkeysfile) '

Output:
===== PUBLIC KEY FINGERPRINTS =====

===== SSH AUTHENTICATION CONFIGURATION =====
[sudo] password for svanapal:
pubkeyauthentication yes
passwordauthentication yes
authorizedkeysfile .ssh/authorized_keys

Explanation:: output confirms that SSH authentication is enabled on the server, but no existing client public-key fingerprints were displayed. We need to establish passwordless SSH for svanapal to the local hostname orion9.

For this single-instance lab, we'll use a dedicated ED25519 client key and authorize it for the same account.

Generate an SSH client key: 
Run as svanapal:
ssh-keygen -t ed25519 \
  -f ~/.ssh/id_ed25519 \
  -N "" \
  -C "svanapal@orion9"

Authorize the public key for the local account
If key generation succeeds, run:

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

Then test:
ssh -o BatchMode=yes \
    -o StrictHostKeyChecking=yes \
    orion9 'hostname; id'

uid=1000(svanapal) gid=1001(oinstall) groups=1001(oinstall),10(wheel),995(vboxsf),1002(dba) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023


Rerun DBCA prerequisite validation:
[svanapal@orion9 ~]$ export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -silent \
  -executePrereqs \
  -databaseConfigType SINGLE
[FATAL] java.lang.NullPointerException


The SSH issue is resolved, but DBCA still returns the same NullPointerException. This tells us SSH host-key trust was one problem, but it was not the only cause of the prerequisite-check failure.
Let's stop repeating the same command and investigate the new trace. We should also verify whether DBCA's prerequisite framework is encountering a configuration or compatibility issue on your Oracle Linux 9.8 system.

Verify the Oracle RU and kernel compatibility:
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

echo "===== OPERATING SYSTEM ====="
cat /etc/os-release

echo
echo "===== RUNNING KERNEL ====="
uname -r

echo
echo "===== ORACLE DATABASE VERSION ====="
"$ORACLE_HOME/bin/sqlplus" -V

echo
echo "===== ORACLE PATCH INVENTORY ====="
"$ORACLE_HOME/OPatch/opatch" lsinventory |
  grep -i -E 'Oracle Database 19c|Release Update|19\.26|19\.32'


  [svanapal@orion9 ~]$ read -rsp "Enter SYS password: " SYS_PWD; echo
read -rsp "Enter SYSTEM password: " SYSTEM_PWD; echo
read -rsp "Enter PDB admin password: " PDB_PWD; echo

export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -silent -ignorePreReqs -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbName ORCL \
  -sid orcl \
  -databaseConfigType SINGLE \
  -storageType FS \
  -datafileDestination /u02/oradata \
  -useOMF true \
  -recoveryAreaDestination NONE \
  -createAsContainerDatabase true \
  -pdbName ORCLPDB1 \
  -characterSet AL32UTF8 \
  -nationalCharacterSet AL16UTF16 \
  -totalMemory 1536 \
  -enableArchive false \
  -databaseType OLTP \
  -sampleSchema false \
  -listeners LISTENER \
  -sysPassword "$SYS_PWD" \
  -systemPassword "$SYSTEM_PWD" \
  -pdbAdminPassword "$PDB_PWD"

RC=$?
unset SYS_PWD SYSTEM_PWD PDB_PWD
echo "DBCA exit code: $RC"
Enter SYS password:
Enter SYSTEM password:
Enter PDB admin password:
Prepare for db operation
8% complete
Copying database files
31% complete
Creating and starting Oracle instance
32% complete
36% complete
40% complete
43% complete
46% complete
Completing Database Creation
51% complete
53% complete
54% complete
Creating Pluggable Databases
58% complete
77% complete
Executing Post Configuration Actions
100% complete
Database creation complete. For details check the logfiles at:
 /u01/app/oracle/cfgtoollogs/dbca/ORCL.
Database Information:
Global Database Name:ORCL
System Identifier(SID):orcl
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/ORCL/ORCL.log" for further details.
DBCA exit code: 0
[svanapal@orion9 ~]$


Verify the database and PDB:
[svanapal@orion9 ~]$ export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH="$ORACLE_HOME/bin:$PATH"

sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Oct 21 10:16:31 2026
Version 19.26.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

SQL> SELECT instance_name, status, database_status
FROM v$instance;

SELECT name, open_mode, cdb
FROM v$database;

SHOW PDBS;  2
INSTANCE_NAME    STATUS       DATABASE_STATUS
---------------- ------------ -----------------
orcl             OPEN         ACTIVE

SQL> SQL>   2
NAME      OPEN_MODE            CDB
--------- -------------------- ---
ORCL      READ WRITE           YES

SQL> SQL>

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB1                       READ WRITE NO
SQL>
SQL>
SQL> EXIT;
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$ export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH="$ORACLE_HOME/bin:$PATH"

sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Oct 21 10:16:31 2026
Version 19.26.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

SQL> SELECT instance_name, status, database_status
FROM v$instance;

SELECT name, open_mode, cdb
FROM v$database;

SHOW PDBS;  2
INSTANCE_NAME    STATUS       DATABASE_STATUS
---------------- ------------ -----------------
orcl             OPEN         ACTIVE

SQL> SQL>   2
NAME      OPEN_MODE            CDB
--------- -------------------- ---
ORCL      READ WRITE           YES

SQL> SQL>

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB1                       READ WRITE NO
SQL>
SQL>
SQL> EXIT;
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0
[svanapal@orion9 ~]$
[svanapal@orion9 ~]$ export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH="$ORACLE_HOME/bin:$PATH"

sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Oct 21 10:16:31 2026
Version 19.26.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

SQL> SELECT instance_name, status, database_status
FROM v$instance;

SELECT name, open_mode, cdb
FROM v$database;

SHOW PDBS;  2
INSTANCE_NAME    STATUS       DATABASE_STATUS
---------------- ------------ -----------------
orcl             OPEN         ACTIVE

SQL> SQL>   2
NAME      OPEN_MODE            CDB
--------- -------------------- ---
ORCL      READ WRITE           YES

SQL> SQL>

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB1                       READ WRITE NO
SQL>
SQL>
SQL> EXIT;
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0
[svanapal@orion9 ~]$


