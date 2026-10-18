# Oracle Database 19c Silent Installation Runbook — Oracle Linux 9.8

> **Document type:** Reconstructed execution runbook and technical audit  
> **Source:** `1.1-silent-installation-binaries-final (1).md`  
> **Source coverage:** Complete source file read from first to last line; 1,119 newline-delimited lines (the source's final line is line 1119; its file index reported 1120 file lines).  
> **Target system evidenced in source:** `orion9`, Oracle Linux Server 9.8, x86_64, UEK kernel.  
> **Runbook end state:** Oracle Database 19c software-only installation with RU 19.26 applied. **Database creation remains blocked/pending.**

## Purpose and use

This runbook preserves the commands, observed outputs, decisions, warnings, failed attempts, and pending work in the source while organizing them as a repeatable procedure. Historical evidence is kept distinct from reconstructed guidance. Do not interpret a command included for future execution as proof that it was executed on `orion9`.

## Scope and target architecture

- Oracle Database 19c Database Home only; silent response-file installation.
- Base image: `V982063-01_Db.zip` (19.3 image).
- RU staged/applied: patch `37260974`, RU `19.26.0.0.250121`.
- OPatch target version: `12.2.0.1.45`.
- Oracle Base: `/u01/app/oracle`.
- Oracle Home: `/u01/app/oracle/product/19.0.0/dbhome_1`.
- Central inventory: `/u01/app/oraInventory`.
- Software owner: `svanapal`; primary group `oinstall`; administration group `dba`.
- Planned database: single instance with filesystem storage; no Grid Infrastructure, ASM, or Oracle Restart in this plan.
- Database, listener, SQL connectivity, and open-mode checks are not established as complete.

## Safety and execution conventions

1. Inspect existing state before creating users, groups, files, swap, or configuration entries.
2. Commands explicitly described as root/`sudo` actions must not be run as the Oracle software owner. Installer, OPatch, and Oracle Home extraction actions are run as `svanapal`, unless the Oracle instructions explicitly require otherwise.
3. Keep a backup before replacing Oracle Home components. Do not overwrite an existing backup without inspecting it.
4. The source used shared-folder paths containing spaces. Quote paths and verify the mount point on the target VM.
5. Do not commit passwords, credentials, Oracle installation ZIPs, RU archives, patch stages, backups, or sensitive logs to Git unless specifically approved.
6. The source records RU 19.26 for this lab and also notes that Oracle Linux 9 guidance recommended RU 19.32 or later. Verify current certification, RU guidance, and patch README for the exact OS/kernel before reusing this procedure.
7. The DBCA prerequisite failure (`DBT-09101`, `PRVF-5301`, `PRVF-9905`) is unresolved. Do not blindly use `-ignorePrereqFailure` or treat database creation as successful.

## Activity inventory and dependency order

| Activity | Activity name | Depends on | Source evidence / status |
|---|---|---|---|
| 01 | [Activity 01](#activity-01--confirm-platform-and-installation-route) | None | Recorded OS/kernel/architecture; partially verified |
| 02 | [Activity 02](#activity-02--inventory-software-patches-users-groups-and-paths) | 01 | ZIPs and account/group details recorded; partially verified |
| 03 | [Activity 03](#activity-03--ensure-oracle-groups-user-and-directories-exist) | 02 | User, groups, directories verified on `orion9` |
| 04 | [Activity 04](#activity-04--extract-the-oracle-19c-base-image) | 03; base ZIP available; sufficient `/u01` space | Installer and extracted Home observed |
| 05 | [Activity 05](#activity-05--audit-and-install-os-prerequisites) | 01 | Initial missing packages and later install command recorded; final broad package audit incomplete |
| 06 | [Activity 06](#activity-06--configure-oracle-user-resource-limits) | 03 | Effective values recorded after re-login |
| 07 | [Activity 07](#activity-07--configure-persistent-kernelnetwork-parameters) | 01 | Effective sysctl values recorded |
| 08 | [Activity 08](#activity-08--configure-transparent-hugepages) | 01 | `madvise` recorded after reboot; confirm current Oracle guidance |
| 09 | [Activity 09](#activity-09--review-and-stage-ru-1926) | 02; RU ZIP available | Patch metadata and stage files verified; extraction command is not fully recorded |
| 10 | [Activity 10](#activity-10--back-up-oracle-home-before-changing-opatch) | 04 | Backup archive and gzip integrity recorded; archive-creation command is absent |
| 11 | [Activity 11](#activity-11--upgrade-opatch-to-1220145) | 10; OPatch ZIP available; RU README minimum checked | Version 12.2.0.1.45 expected/target; old directory preserved |
| 12 | [Activity 12](#activity-12--create-and-validate-the-silent-response-file) | 03, 04 | Twelve selected parameters each appeared once |
| 13 | [Activity 13](#activity-13--run-installer-prerequisite-checks) | 05–08, 09, 11, 12 | Swap about 8 GiB recorded; optional warning remained |
| 14 | [Activity 14](#activity-14--install-oracle-19c-software-and-apply-ru-1926) | 04–13 | Installer completed with warnings; RU application reported successful |
| 15 | [Activity 15](#activity-15--run-root-scripts) | 14 | Both root scripts run; log review found no explicit error-word matches |
| 16 | [Activity 16](#activity-16--verify-oracle-home-and-patch-inventory) | 11, 14, 15 | SQL*Plus 19.26 and patch IDs recorded |
| 17 | [Activity 17](#activity-17--check-dbca-availability-and-silent-options) | 16 | DBCA exists; silent single/FS options listed |
| 18 | [Activity 18](#activity-18--dbca-prerequisite-check-unresolved-clusterware-errors) | 17 | **Blocked** by Clusterware checks; cause and supported resolution unverified |
| 19 | [Activity 19](#activity-19--database-creation-pending) | 18 resolved; capacity and configuration choices recorded | **Pending; not started** |

## Prerequisite and dependency summary

- **Before any changes:** identify OS, architecture, hostname, current user/groups, software paths, filesystem capacity, memory/swap, and whether the target directories already contain an Oracle Home.
- **Before installing:** verify the 19c image, RU archive and README, OPatch archive/version, Oracle Home and inventory paths, resource limits, kernel settings, THP mode, swap, and response-file values.
- **Before RU application:** inspect RU README prerequisites and ensure the installed OPatch meets the documented minimum. The source records a minimum of `12.2.0.1.44`; the chosen OPatch was `12.2.0.1.45`.
- **Before DBCA:** verify the installed Home and patch inventory, decide database identity/storage/memory/character set/password handling, re-check free space, and resolve the critical DBCA prerequisite failure through a supported procedure.
- **Evidence rule:** historical output is labelled “Recorded result.” Results shown as examples or targets are labelled “Expected result.” Any step added to close a source gap is labelled “Reconstructed / proposed; not historically verified.”

---

> **Purpose:** A consolidated, GitHub-friendly runbook for the work
> completed on `orion9`. The activity sequence is listed first. The
> detailed sections then follow that same sequence and include commands,
> observed results, and explanations.
>
> **Scope:** Oracle Database 19c software-only installation with Release
> Update (RU) 19.26 applied during installation. The database is **not
> yet created**. The intended next architecture is a single-instance
> database using filesystem storage, without Grid Infrastructure, ASM,
> or Oracle Restart.
>
> **Important:** This is a lab record, not a claim that Oracle 19.26 is
> the latest or recommended RU for every Oracle Linux 9 deployment. The
> original notes mention Oracle Linux 9 guidance recommending RU 19.32
> or later. Verify current Oracle certification and patch guidance
> before using this procedure outside the lab.

## 1. Complete activity sequence

Use this as the master index. The detailed activity records below use
the same numbering.

  -----------------------------------------------------------------------------------------------
                   Seq. Activity         Main purpose                      Recorded status
  --------------------- ---------------- --------------------------------- ----------------------
                     01 Confirm platform Identify OS, kernel,              Complete
                        and installation architecture, and selected silent 
                        route            method                            

                     02 Inventory        Confirm source ZIPs and intended  Complete
                        software, patch  Oracle directories                
                        files, users,                                      
                        groups, and                                        
                        paths                                              

                     03 Create Oracle    Establish `svanapal`, `oinstall`, Complete
                        groups and       `dba`, Oracle Base, Oracle Home,  
                        directories      and inventory                     

                     04 Extract Database Populate the Database Home from   Complete
                        19c base image   the 19c ZIP                       

                     05 Audit and        Check packages, build tools,      Completed with
                        install OS       memory, swap, disk, hostname, and recorded
                        prerequisites    limits                            warnings/limitations

                     06 Configure Oracle Persist `nofile`, `nproc`,        Complete and verified
                        user limits      `stack`, and `memlock` settings   

                     07 Configure        Set persistent sysctl values and  Complete and verified
                        kernel/network   verify effective values           
                        parameters                                         

                     08 Configure        Set                               Complete and verified
                        Transparent      `transparent_hugepage=madvise`,   
                        HugePages        reboot, and verify                

                     09 Review RU 19.26  Inspect patch archive,            Complete
                        instructions and prerequisites, metadata, and      
                        stage files      extraction                        

                     10 Back up Oracle   Create and validate a recoverable Complete; backup
                        Home before      backup                            recorded
                        changing OPatch                                    

                     11 Upgrade OPatch   Replace old OPatch with           Complete
                                         12.2.0.1.45                       

                     12 Create and       Configure software-only           Complete
                        validate the     installation and privilege groups 
                        silent response                                    
                        file                                               

                     13 Run installer    Inspect CVU results and resolve   Completed; optional
                        prerequisite     the swap-related requirement      warning remained
                        checks                                             

                     14 Install Oracle   Run                               Complete with warnings
                        19c and apply RU `runInstaller -silent -applyRU`   
                        19.26                                              

                     15 Run root scripts Configure central inventory and   Complete; log review
                                         finish Oracle Home root actions   did not show explicit
                                                                           errors

                     16 Verify Oracle    Check OPatch, SQL\*Plus,          Complete
                        Home and patch   executable, and applied patch IDs 
                        inventory                                          

                     17 Check DBCA       Confirm DBCA exists and supports  Complete
                        availability and silent creation                   
                        silent options                                     

                     18 Run DBCA         Identify why single-instance      **Blocked: Clusterware
                        creation         prerequisite check failed         checks reported
                        prerequisites                                      critical failures**

                     19 Create           Choose database name, storage,    **Not started**
                        standalone       memory, CDB/PDB, and credentials  
                        filesystem                                         
                        database                                           
  -----------------------------------------------------------------------------------------------

### Current end state

-   OS: Oracle Linux Server 9.8, x86_64, UEK kernel.
-   Oracle Home: `/u01/app/oracle/product/19.0.0/dbhome_1`
-   Oracle Base: `/u01/app/oracle`
-   Central inventory: `/u01/app/oraInventory`
-   Software owner: `svanapal`; primary group `oinstall`; administration
    group `dba`.
-   OPatch: `12.2.0.1.45`
-   SQL\*Plus: `19.26.0.0.0`
-   Applied patch inventory includes RU `37260974` (19.26) and OCW
    `29585399` (19.3.0.0.0).
-   The software installation is complete. A database and listener have
    not been confirmed as created.
-   Grid Infrastructure is not installed and will **not** be installed
    as part of the current plan.
-   DBCA prerequisite command failed with `DBT-09101`, citing
    `PRVF-5301` and `PRVF-9905` Clusterware checks. This is unresolved;
    do not mark database creation complete or blindly bypass the
    failure.

------------------------------------------------------------------------

# Detailed activity records

## Activity 01 — Confirm platform and installation route

**Run as:** `svanapal`

``` bash
cat /etc/os-release
uname -r
uname -m
```

**Observed result**

-   `Oracle Linux Server 9.8`
-   Kernel: `6.12.0-203.76.7.3.el9uek.x86_64`
-   Architecture: `x86_64`

**Explanation**

These facts identify the OS and architecture for the installation and
patch compatibility review. The chosen installation method is a silent
response-file installation. The original notes state that Oracle Linux 9
guidance recommends RU 19.32 or later; RU 19.26 was nevertheless
selected for this lab exercise. Confirm current certification and patch
recommendations before repeating this route in another environment.



**Audited status:** Partially verified — OS/kernel/architecture outputs are recorded; current certification/RU recommendation must be rechecked.

## Activity 02 — Inventory software, patches, users, groups, and paths

**Run as:** `svanapal`

``` bash
echo "===== CURRENT USER ====="
id

echo
echo "===== ORACLE USER ====="
getent passwd svanapal

echo
echo "===== ORACLE GROUPS ====="
getent group oinstall
getent group dba
getent group oper

echo
echo "===== DATABASE ZIP ====="
ls -lh /media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software/V982063-01_Db.zip

echo
echo "===== RU ZIP ====="
ls -lh /media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch/p37260974_19.26_JAN_2025.zip

echo
echo "===== OPATCH ZIP ====="
ls -lh /media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch/p6880880_12.2.0.1.45_19cDB.zip
```

**Observed result**

-   Database image: `V982063-01_Db.zip`, approximately 2.9 GB.
-   RU archive: `p37260974_19.26_JAN_2025.zip`, approximately 1.8 GB.
-   OPatch archive: `p6880880_12.2.0.1.45_19cDB.zip`.
-   The Oracle user and groups were established as `svanapal`,
    `oinstall`, and `dba`.

**Explanation**

The ZIP files are staged on the VirtualBox shared folder. Keep paths
quoted when using shell variables or paths containing spaces. Use the
actual paths verified on the VM; do not assume another VM has the same
mount point.



**Audited status:** Partially verified — archive paths and approximate sizes are recorded; repeat checks on any new VM.

## Activity 03 — Ensure Oracle groups, user, and directories exist

**Run only the required creation/ownership commands with `sudo`.** This
section is safe to follow on either a new VM or an already configured VM:
existing groups and the `svanapal` user are retained, and missing items
are created. Do not blindly rerun `groupadd` on an already configured VM.

### 3.1 Inspect the current configuration

```bash
echo "===== ORACLE GROUPS ====="
getent group oinstall || true
getent group dba || true

echo "===== ORACLE USER ====="
getent passwd svanapal || true
id svanapal 2>/dev/null || true

echo "===== ORACLE DIRECTORIES ====="
ls -ld /u01/app/oracle /u01/app/oracle/product/19.0.0/dbhome_1 /u01/app/oraInventory 2>/dev/null || true
```

### 3.2 Create only missing groups and the user

```bash
# Create each group only if it does not already exist.
if ! getent group oinstall >/dev/null; then
    sudo groupadd oinstall
else
    echo "Group oinstall already exists; no action required."
fi

if ! getent group dba >/dev/null; then
    sudo groupadd dba
else
    echo "Group dba already exists; no action required."
fi

# Create the Oracle software owner only if it does not already exist.
if ! getent passwd svanapal >/dev/null; then
    sudo useradd -m -g oinstall -G dba -s /bin/bash svanapal
else
    echo "User svanapal already exists; no useradd required."
fi

# Ensure the intended primary and supplementary group assignments.
sudo usermod -g oinstall -aG dba svanapal
```

If your environment uses a different Oracle software-owner name, replace
`svanapal` consistently before running these commands. If `useradd` creates
a new account, set its password according to your lab's access policy before
logging in as that user.

### 3.3 Create directories and set ownership/permissions

```bash
# mkdir -p is safe when the directories already exist.
sudo mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
sudo mkdir -p /u01/app/oraInventory

# Apply the intended owner and group to the Oracle Base and inventory.
sudo chown -R svanapal:oinstall /u01/app/oracle
sudo chown -R svanapal:oinstall /u01/app/oraInventory
sudo chmod -R 775 /u01/app/oracle
sudo chmod -R 775 /u01/app/oraInventory
```

### 3.4 Verify the final state

```bash
echo "===== USER AND GROUP MEMBERSHIP ====="
id svanapal
getent group oinstall
getent group dba

echo "===== DIRECTORY OWNERSHIP ====="
ls -ld /u01/app/oracle
ls -ld /u01/app/oracle/product/19.0.0/dbhome_1
ls -ld /u01/app/oraInventory

echo "===== FILESYSTEM SPACE ====="
df -h /u01
```

**Observed final state on `orion9`**

-   User: `svanapal`, UID 1000.
-   Primary group: `oinstall`, GID 1001.
-   Supplementary `dba` group: GID 1002.
-   Oracle Home: `/u01/app/oracle/product/19.0.0/dbhome_1`
-   Inventory: `/u01/app/oraInventory`

**Explanation**

The central inventory is outside the Database Home so it can track Oracle
installations independently. The conditional checks make this activity
repeatable: existing groups and users are not recreated, and `mkdir -p`
works when the directories already exist. Ownership and permissions are
then explicitly set to the values expected by this runbook. Run the
verification commands after the changes; if you are changing membership for
the currently logged-in user, start a new login session before relying on
the updated group membership.



**Audited status:** Verified on the recorded orion9 state — user/group IDs and target paths were observed after configuration.

## Activity 04 — Extract the Oracle 19c base image

**Run as:** `svanapal`; do not use `sudo` for extraction.

``` bash
command -v unzip

unzip /media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software/V982063-01_Db.zip \
  -d /u01/app/oracle/product/19.0.0/dbhome_1

export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1

echo "===== INSTALLER ====="
ls -l "$ORACLE_HOME/runInstaller"

echo
echo "===== ORACLE HOME SIZE ====="
du -sh "$ORACLE_HOME"

echo
echo "===== OWNERSHIP ====="
ls -ld "$ORACLE_HOME" "$ORACLE_HOME/bin" "$ORACLE_HOME/oui"

echo
echo "===== SPACE ====="
df -h /u01
```

**Observed result**

-   `unzip` was available at `/usr/bin/unzip`.
-   `runInstaller` was present.
-   Extracted Oracle Home size was approximately 6.5 GB.
-   Some extracted directories initially had group `svanapal`; ownership
    was standardized to `svanapal:oinstall`.

**Explanation**

The base ZIP supplies the 19.3 Database Home files. The RU is applied
during installation with `-applyRU`, rather than being treated as a
separate database-creation step.



**Audited status:** Partially verified — extraction and installer presence are recorded; archive checksum/signature validation is not recorded.

## Activity 05 — Audit and install OS prerequisites

### 05.1 Read-only OS audit

``` bash
echo "===== OS ====="
cat /etc/os-release
uname -r
uname -m

echo
echo "===== MEMORY AND SWAP ====="
free -h
swapon --show

echo
echo "===== FILESYSTEMS ====="
df -hT

echo
echo "===== PACKAGE SCREEN ====="
rpm -q bc binutils elfutils-libelf glibc glibc-devel \
  ksh libaio libaio-devel libgcc libstdc++ make sysstat

echo
echo "===== BUILD TOOLS ====="
for tool in make ar ld nm; do
  command -v "$tool" || true
done

echo
echo "===== HOSTNAME ====="
hostname
hostname -f
getent hosts orion9
getent hosts orion9.sunilit.org
```

**Recorded findings and results**

-   Initial package screen reported missing `bc`, `ksh`, `libaio-devel`,
    and `sysstat`; these were installed later.
-   The broader package audit reported some packages missing from the
    selected query. A package marked missing by `rpm -q` is not
    automatically proof that the installer requires it on this OS;
    compare with the exact RU and OS documentation before installing.
-   Required build tools `make`, `ar`, `ld`, and `nm` were found under
    `/usr/bin`.
-   Hostname shown: `orion9`; FQDN: `orion9.sunilit.org`.
-   An optional `compat-libcap1-1.10` prerequisite was later reported as
    **IGNORABLE**. It was not found in the enabled repositories; do not
    install an arbitrary unrelated RPM to satisfy that message.

### 05.2 Install packages recorded as missing in the initial screen

The file records this package installation command:

``` bash
sudo dnf install -y bc ksh libaio-devel sysstat
rpm -q bc ksh libaio-devel sysstat
```

Additional packages were considered after a broader package audit.
Verify package availability and the applicable Oracle Linux 9 guidance
before installing any optional package set on another VM.

**Explanation**

The initial package screen was captured before the installation work was
complete. Do not use that early output as the final package state. The
later DBCA log still recorded `compat-libcap1-1.10` as an ignorable
check, while the fatal DBCA issue was a separate Clusterware check.



**Audited status:** Partially verified — some package findings and install command are recorded; a complete final package audit is not shown.

## Activity 06 — Configure Oracle user resource limits

**Run configuration changes with `sudo`; verify from a new login session
as `svanapal`.**

``` bash
sudo tee /etc/security/limits.d/99-oracle-database.conf > /dev/null <<'EOF'
svanapal soft nofile 1024
svanapal hard nofile 65536
svanapal soft nproc 16384
svanapal hard nproc 16384
svanapal soft stack 10240
svanapal hard stack 32768
svanapal soft memlock 3145728
svanapal hard memlock 3145728
EOF

cat /etc/security/limits.d/99-oracle-database.conf
```

Verify effective limits in a fresh login session:

``` bash
ulimit -Sn
ulimit -Hn
ulimit -Su
ulimit -Hu
ulimit -Ss
ulimit -Hs
ulimit -Sl
ulimit -Hl
```

**Observed result after configuration/re-login**

  Limit              Effective value
  ---------------- -----------------
  Soft `nofile`                 1024
  Hard `nofile`                65536
  Soft `nproc`                 16384
  Hard `nproc`                 16384
  Soft `stack`              10240 KB
  Hard `stack`              32768 KB
  Soft `memlock`          3145728 KB
  Hard `memlock`          3145728 KB

**Explanation**

The file makes the Oracle user's resource-limit configuration
persistent. The effective values should be checked after a new login;
editing a limits file does not retroactively change limits in an
existing shell.



**Audited status:** Verified on the recorded orion9 state — effective resource limits were recorded after re-login.

## Activity 07 — Configure persistent kernel/network parameters

**Run with `sudo`; then verify the effective values.**

``` bash
sudo tee /etc/sysctl.d/98-oracle19c.conf > /dev/null <<'EOF'
fs.aio-max-nr = 1048576
kernel.panic_on_oops = 1
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF

sudo sysctl -p /etc/sysctl.d/98-oracle19c.conf

echo "===== VERIFY EFFECTIVE VALUES ====="
sysctl fs.aio-max-nr
sysctl kernel.panic_on_oops
sysctl net.ipv4.ip_local_port_range
sysctl net.core.rmem_default
sysctl net.core.rmem_max
sysctl net.core.wmem_default
sysctl net.core.wmem_max
```

**Observed result**

-   `fs.aio-max-nr = 1048576`
-   `kernel.panic_on_oops = 1`
-   `net.ipv4.ip_local_port_range = 9000 65500`
-   `net.core.rmem_default = 262144`
-   `net.core.rmem_max = 4194304`
-   `net.core.wmem_default = 262144`
-   `net.core.wmem_max = 1048576`

**Explanation**

The file under `/etc/sysctl.d/` persists the values across reboots.
These are the values recorded in this lab; confirm requirements and
compatibility for the exact Oracle RU and operating system before
applying the same settings elsewhere.


**To find Oracle-related configuration files in the relevant directories:**
sudo find /etc/security/limits.d /etc/sysctl.d -maxdepth 1 -type f -iname '*oracle*' -print




**Audited status:** Verified on the recorded orion9 state — effective values are recorded after applying the sysctl file.

## Activity 08 — Configure Transparent HugePages

The lab configuration uses `transparent_hugepage=madvise` for the UEK 8
kernel. The source record shows a backup of `/etc/default/grub`,
followed by persistent boot-argument configuration.

``` bash
# Backup was created during the lab:
sudo cp -a /etc/default/grub /etc/default/grub.pre-oracle19c

# The recorded boot argument:
sudo grubby --update-kernel=ALL --args="transparent_hugepage=madvise"

# Inspect the configured arguments:
sudo grubby --info=ALL | grep -E '^(kernel|args)='
```

After reboot, verify:

``` bash
echo "===== ACTIVE THP MODE ====="
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag

echo
echo "===== ACTIVE KERNEL COMMAND LINE ====="
cat /proc/cmdline
```

**Observed result after reboot**

-   THP enabled file: `always [madvise] never`
-   Kernel command line contained `transparent_hugepage=madvise`.

**How to interpret the results:**
- always [madvise] never means madvise is the current THP mode.
- [always] madvise never means THP is set to always.
- always madvise [never] means THP is disabled.
- HugePages_Total: 0 means no standard HugePages pool is currently reserved. It does not mean THP is disabled.

**Explanation**

The bracketed value is the active mode. The recorded result confirms
`madvise` was selected after reboot.

## HugePages and Transparent HugePages (THP)

- **HugePages:** Uses large memory pages to reduce page-table overhead and improve memory-management efficiency for Oracle SGA.
- **Transparent HugePages (THP):** Kernel manages large pages automatically. Oracle Database 19c guidance recommends disabling THP.
- **Important:** Standard HugePages require explicit sizing and configuration. Do not confuse them with THP.
- **AMM restriction:** Standard HugePages are incompatible with Oracle Automatic Memory Management (`MEMORY_TARGET` / `MEMORY_MAX_TARGET`).
- **Check THP status:**
  ```bash
  cat /sys/kernel/mm/transparent_hugepage/enabled
  ```
- **Check HugePages allocation:**
  ```bash
  grep -E 'HugePages|Hugepagesize|Hugetlb' /proc/meminfo
  ```
- **Implementation:** Verify THP status first. Configure standard HugePages only after determining the planned SGA size and available server memory. Follow the applicable Oracle Linux and Oracle Database 19c documentation.

What does “kernel manages large pages automatically” mean?
- Instead of reserving a fixed HugePages pool in advance, Linux can dynamically convert eligible regular memory pages into larger pages while the system is running. This process can involve the khugepaged kernel thread and may introduce delays.
- The current Oracle AI Database installation guide (May 2026) recommends setting THP to madvise on UEK7 and later kernels, while recommending disabling THP on earlier UEK kernels.





**Audited status:** Partially verified — active THP mode and kernel argument were recorded after reboot; suitability must be checked against current guidance.

## Activity 09 — Review and stage RU 19.26

**Recorded patch details**

-   RU patch ID: `37260974`
-   Version: `19.26.0.0.250121`
-   Archive: `p37260974_19.26_JAN_2025.zip`
-   Extracted stage directory: `/home/svanapal/patch_stage/37260974`
-   Staged directory size: approximately 4.3 GB.
-   RU README redirected to `README.html`.
-   RU instructions identified OPatch minimum `12.2.0.1.44` or later.
-   Database Home-only installation was planned; the RU must not be
    applied to a Grid Infrastructure Home.

Inspect archive metadata:

``` bash
PATCH_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch"
RU="$PATCH_DIR/p37260974_19.26_JAN_2025.zip"

ls -lh "$RU"
unzip -Z1 "$RU" | awk -F/ 'NF>0 {print $1}' | sort -u | head -30
unzip -Z1 "$RU" | grep -iE '(^|/)README[^/]*\.html?$'

unzip -p "$RU" 37260974/README.html |
  perl -0777 -pe '
    s/<(script|style)\b[^>]*>.*?<\/\1>//gis;
    s/<\/(h[1-6]|p|li|tr|div|table)>/\n/gi;
    s/<[^>]+>/ /g;
    s/&nbsp;/ /gi;
    s/&amp;/\&/gi;
    s/&lt;/</gi;
    s/&gt;/>/gi;
    s/&quot;/"/gi;
  ' |
  tr -s '[:space:]' ' ' |
  sed -n '/2 Prerequisites/,/3 Installation/p' |
  fold -sw 120 |
  head -80
```

The lab then extracted the RU to the staging directory and verified the
metadata:

``` bash
RU_STAGE="$HOME/patch_stage/37260974"

ls -l "$RU_STAGE/etc/config/inventory.xml" \
      "$RU_STAGE/etc/config/actions.xml"
grep -oE '19\.26\.0\.0\.250121|37260974' "$RU_STAGE/README.html" | sort -u
```

**Observed result**

The stage contained `etc/config/inventory.xml`,
`etc/config/actions.xml`, and `README.html`; the README contained
`19.26.0.0.250121` and `37260974`.

**Explanation**

This is a verification of the patch archive and extracted stage, not yet
proof that the patch is applied. Application was performed in Activity
14.



**Audited status:** Partially verified — patch metadata and extracted files were checked; the exact extraction command is not present in the source.

## Activity 10 — Back up Oracle Home before changing OPatch

The source record shows a backup of the Oracle Home to the external
shared-folder storage before replacing OPatch.

**Recorded backup**

`/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software_Backup/oracle19c_home_pre_opatch_20261016_134941.tar.gz`

-   Compressed size: approximately 2.9 GB.
-   `gzip -t` passed.

A representative verification command is:

``` bash
BACKUP="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software_Backup/oracle19c_home_pre_opatch_20261016_134941.tar.gz"

ls -lh "$BACKUP"
gzip -t "$BACKUP" && echo "Backup gzip integrity check passed"
```

**Explanation**

The backup provides a recovery point before changing the Oracle Home's
OPatch utility. A successful gzip integrity check confirms the archive
can be decompressed without a gzip-format error; it does not by itself
prove a full Oracle restore has been tested.



**Audited status:** Partially verified — archive existence/size and gzip integrity are recorded; archive-creation command and restore test are not recorded.

## Activity 11 — Upgrade OPatch to 12.2.0.1.45

The original OPatch version was `12.2.0.1.17`. The staged OPatch archive
was extracted separately and checked before replacing the Oracle Home's
OPatch directory. The previous OPatch directory was preserved as a
backup.

Run as the Oracle software owner `svanapal`.

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
cd "$ORACLE_HOME"

# 1. Preserve the existing OPatch directory
mv OPatch OPatch_12.2.0.1.17

# 2. Extract the new OPatch utility directly into ORACLE_HOME
unzip -o /home/svanapal/opatch_stage/p6880880_12.2.0.1.45_19cDB.zip \
  -d "$ORACLE_HOME"

# 3. Verify the installed version
"$ORACLE_HOME/OPatch/opatch" version
```

**Expected result:**

```text
OPatch Version: 12.2.0.1.45
OPatch succeeded.
```

**Explanation:** The original OPatch directory is retained as `OPatch_12.2.0.1.17` for recovery. The new utility is extracted into `$ORACLE_HOME`, creating the active `OPatch` directory.

**Important:** Run the backup command only if `OPatch_12.2.0.1.17` does not already exist. Verify the extracted ZIP contains an `OPatch/` directory before proceeding. Confirm the required OPatch version in the selected RU's README.



**Audited status:** Partially verified — target OPatch version and replacement procedure are documented; the source presents the version output as expected rather than captured historical output.

## Activity 12 — Create and validate the silent response file

The original template was copied so the Oracle-supplied response file
remains unchanged.

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

cp "$ORACLE_HOME/install/response/db_install.rsp" \
   "$HOME/db_install_lab.rsp"
chmod 600 "$HOME/db_install_lab.rsp"
```

The recorded response-file values were:

``` properties
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=dba
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba
```

Validate the key values and duplicates:

``` bash
RSP="$HOME/db_install_lab.rsp"

grep -nE '^(oracle.install.option|UNIX_GROUP_NAME|INVENTORY_LOCATION|ORACLE_HOME|ORACLE_BASE|oracle.install.db.InstallEdition|oracle.install.db.OSDBA_GROUP|oracle.install.db.OSOPER_GROUP|oracle.install.db.OSBACKUPDBA_GROUP|oracle.install.db.OSDGDBA_GROUP|oracle.install.db.OSKMDBA_GROUP|oracle.install.db.OSRACDBA_GROUP)=' "$RSP"

grep -E '^(oracle.install.option|UNIX_GROUP_NAME|INVENTORY_LOCATION|ORACLE_HOME|ORACLE_BASE|oracle.install.db.InstallEdition|oracle.install.db.OSDBA_GROUP|oracle.install.db.OSOPER_GROUP|oracle.install.db.OSBACKUPDBA_GROUP|oracle.install.db.OSDGDBA_GROUP|oracle.install.db.OSKMDBA_GROUP|oracle.install.db.OSRACDBA_GROUP)=' "$RSP" |
  cut -d= -f1 | sort | uniq -c
```

**Observed result**

Each of the 12 selected parameters appeared once. The installation type
was `INSTALL_DB_SWONLY`, so this installation did not create a starter
database.

**Explanation**

The response file configures software-only installation. Database
creation is a separate DBCA operation after the binaries are installed.



**Audited status:** Verified for the selected response-file values — each of the 12 selected parameters appeared once; full response-file validation is not shown.

## Activity 13 — Run installer prerequisite checks

The installer prerequisite check was run using the Oracle 19c home and a
temporary CVU distribution override:

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export CV_ASSUME_DISTID=OL7
cd "$ORACLE_HOME"

./runInstaller -silent -executePrereqs \
  -responseFile "$HOME/db_install_lab.rsp"
```

**Recorded result**

-   CVU required approximately `7.7277 GiB` of swap.
-   The original 2 GiB swap was insufficient.
-   A 6 GiB `/swapfile` was created, activated, and added to
    `/etc/fstab`; combined active swap was approximately 8 GiB.
-   The subsequent installer prerequisite check reported `INS-13014`,
    indicating optional requirements remained unmet.
-   `compat-libcap1-1.10` was listed as an ignorable prerequisite in the
    captured log.
-   `net-tools` was installed later in the sequence.

**Explanation**

`CV_ASSUME_DISTID=OL7` was a temporary compatibility workaround for the
19.3 CVU running on Oracle Linux 9. It does not change the operating
system release. Do not set it globally without understanding the
implications.

### Swap setup, persistence, and verification

The lab had an existing 2 GiB LVM swap area. A separate 6 GiB `/swapfile`
was added to bring the combined active swap to approximately 8 GiB.
Follow the checks first; do not recreate an existing swap file or append
a duplicate `/etc/fstab` entry.

**1. Check the current state before creating anything**

``` bash
sudo grep -nE '^[[:space:]]*([^#].*[[:space:]]swap[[:space:]]|/swapfile[[:space:]])' /etc/fstab
sudo ls -lh /swapfile
swapon --show
free -h
```

**2. Create and activate the swap file only if `/swapfile` does not exist**

``` bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=6144 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

If `/swapfile` already exists and is active, do not rerun these creation
commands.

**3. Add persistence to `/etc/fstab` only if the entry is missing**

First check for an existing entry:

``` bash
grep -nE '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab
```

If this command returns no output, add the persistent entry:

``` bash
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
```

Expected `/etc/fstab` entry:

``` fstab
/swapfile none swap defaults 0 0
```

If the entry already exists, **do not run the `tee -a` command again**.

**4. Verify active swap and persistence**

``` bash
swapon --show
free -h
grep -nE '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab
sudo findmnt --verify --verbose
```

**Recorded result and caution:** The lab had approximately 8 GiB total
active swap. A previous `findmnt --verify` run reported warnings about
duplicate `none` targets and the swap file being a regular file. Review
the actual `/etc/fstab` entries and resolve duplicates before treating
validation as clean. Do not append another entry simply to repeat the
setup sequence.



**Audited status:** Partially verified — prerequisite findings and swap state are recorded; fstab validation reported warnings that need review.

## Activity 14 — Install Oracle 19c software and apply RU 19.26

**Run as:** `svanapal`, not root.

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export CV_ASSUME_DISTID=OL7

cd "$ORACLE_HOME"

./runInstaller \
  -silent \
  -applyRU /home/svanapal/patch_stage/37260974 \
  -responseFile "$HOME/db_install_lab.rsp" \
  -waitforcompletion
```

**Recorded installer output**

-   `Preparing the home to patch...`
-   `Applying the patch ...`
-   `Successfully applied the patch.`
-   Warning `INS-32047`: central inventory location was not empty.
-   Warning `INS-13014`: some optional prerequisites were not met.
-   `Successfully Setup Software with warning(s).`
-   Installer instructed root to run:
    1.  `/u01/app/oraInventory/orainstRoot.sh`
    2.  `/u01/app/oracle/product/19.0.0/dbhome_1/root.sh`

**Explanation**

The installation succeeded with warnings. `-applyRU` applied RU 19.26
during the software installation. The central inventory warning was
recorded; it should not be silently removed from the history. The
installation command did not create a database or configure a listener.



**Audited status:** Verified with warnings — installer reported successful RU application and software setup with warnings.

## Activity 15 — Run root scripts

**Run as root, in the order shown by the installer.**

``` bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

**Observed result**

-   `orainstRoot.sh` completed and changed central inventory
    permissions/group.
-   `root.sh` created `/etc/oratab` and copied `dbhome`, `oraenv`, and
    `coraenv` to `/usr/local/bin`.
-   Root-script log:
    `/u01/app/oracle/product/19.0.0/dbhome_1/install/root_orion9_2026-10-16_20-51-43-302697673.log`
-   The captured log did not contain explicit `error`, `failed`, or
    `fatal` matches. It ended after "Now product-specific root actions
    will be performed."

**Explanation**

The root scripts are part of finishing the software installation. The
recorded log review found no explicit error matches, but absence of
those words is not a complete functional test of every Oracle Home
operation.



**Audited status:** Partially verified — scripts were run and log reviewed; functional verification is not exhaustive.

## Activity 16 — Verify Oracle Home and patch inventory

Run as `svanapal`:

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_BASE=/u01/app/oracle
export PATH="$ORACLE_HOME/bin:$PATH"

echo "===== SQL*PLUS VERSION ====="
"$ORACLE_HOME/bin/sqlplus" -V

echo
echo "===== INSTALLED PATCHES ====="
"$ORACLE_HOME/OPatch/opatch" lspatches

echo
echo "===== ORACLE EXECUTABLE ====="
ls -lh "$ORACLE_HOME/bin/oracle"

echo
echo "===== INVENTORY SUMMARY ====="
"$ORACLE_HOME/OPatch/opatch" lsinventory -oh "$ORACLE_HOME" | tail -20
```

**Observed result**

``` text
SQL*Plus: Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

37260974;Database Release Update : 19.26.0.0.250121 (37260974)
29585399;OCW RELEASE UPDATE 19.3.0.0.0 (29585399)

OPatch succeeded.
```

-   Oracle executable:
    `/u01/app/oracle/product/19.0.0/dbhome_1/bin/oracle`, approximately
    441 MB.
-   OPatch version: `12.2.0.1.45`.
-   Oracle Home inventory listed Oracle Database 19c and patch
    `37260974`.

**Explanation**

These checks confirm the patched software Home and its inventory. They
do not prove a database exists or can be opened, because the response
file was for software-only installation.



**Audited status:** Verified for software version and patch inventory — database existence/open state is not implied.

## Activity 17 — Check DBCA availability and silent options

Run as `svanapal`:

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -help
"$ORACLE_HOME/bin/dbca" -silent -createDatabase -help
```

**Observed result**

-   DBCA executable existed at `$ORACLE_HOME/bin/dbca`.
-   Calling `dbca -version` returned `DISPLAY not set`; this was a
    graphical-display message, not proof that DBCA was missing.
-   `dbca -help` listed `-silent` and `-createDatabase`.
-   The creation help listed `-databaseConfigType SINGLE` and
    `-storageType FS`.

**Explanation**

Silent DBCA is available. For the desired standalone architecture, use
filesystem storage and a single-instance configuration. Do not use ASM
or RAC options for this plan.



**Audited status:** Verified for DBCA executable/help options — database creation itself is not established.

## Activity 18 — DBCA prerequisite check: unresolved Clusterware errors

The following check was attempted:

``` bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -silent -executePrereqs \
  -databaseConfigType SINGLE
```

**Observed result**

``` text
[FATAL] [DBT-09101] Target environment does not meet some mandatory requirements.
PRVF-5301 : Failed to locate CRS home
PRVF-9905 : Clusterware is not installed on the following nodes : "orion9"
```

The trace file was:

`/u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-16_09-14-13PM`

The same log listed `compat-libcap1-1.10` with severity `IGNORABLE`; the
critical task failures were CRS Integrity and Node Application
Existence.

**Explanation**

Grid Infrastructure is not installed and the current plan is not to
install it. The error therefore needs to be investigated as a DBCA
prerequisite-check/configuration issue for the intended single-instance
filesystem database. The current record does **not** establish why DBCA
invoked these Clusterware checks, and it does not establish that
bypassing them is safe.

**Do not do this as an unexamined workaround:**

``` bash
# Do not blindly add this to the creation command:
-ignorePrereqFailure
```

Instead, retain the trace log, inspect the exact DBCA command and
response options, and verify the supported Oracle 19c/RU 19.26 DBCA
procedure for standalone filesystem storage. Only proceed when the
critical check is understood and a supported path is established.



**Audited status:** Blocked — mandatory DBCA prerequisite checks failed and no supported resolution is recorded.

## Activity 19 — Database creation: pending

The database has **not** been confirmed as created in this runbook.
Before a creation command is written, make and record these choices:

-   Global database name and SID.
-   CDB name and PDB name.
-   Datafile location on a filesystem with adequate free space.
-   Recovery area location and size, or explicit decision not to
    configure one.
-   Memory allocation appropriate for the VM's 8 GiB RAM.
-   Character set and national character set.
-   Listener creation or use of an existing listener.
-   SYS, SYSTEM, and PDB administrator passwords (do not commit
    passwords to Git).
-   Whether sample schemas are required.

The last recorded filesystem check showed:

  Filesystem      Size     Used   Available
  ------------ ------- -------- -----------
  `/`            20 GB    11 GB      9.5 GB
  `/u01`         20 GB    11 GB      9.4 GB
  `/u02`         10 GB   4.4 GB      5.7 GB
  `/u03`         10 GB   104 MB      9.9 GB
  `/u04`          5 GB    68 MB      4.9 GB

These are the values recorded at that point in the session; re-run
`df -h / /u01 /u02 /u03 /u04` before database creation.



**Audited status:** Pending — database creation has not started; required design choices and prerequisite resolution remain open.

## Reusable final verification checklist

-   [x] OS and architecture recorded.
-   [x] Oracle groups and directories configured.
-   [x] Oracle 19c base image extracted.
-   [x] OS settings reviewed and configured as recorded.
-   [x] RU 19.26 staged and README inspected.
-   [x] OPatch upgraded to 12.2.0.1.45.
-   [x] Response file configured for `INSTALL_DB_SWONLY`.
-   [x] Oracle 19c software installed with `-applyRU`.
-   [x] Root scripts run.
-   [x] SQL\*Plus reports version 19.26.0.0.0.
-   [x] OPatch inventory lists RU 37260974.
-   [x] DBCA silent creation options inspected.
-   [ ] DBCA critical Clusterware prerequisite failure resolved by a
    supported method.
-   [ ] Standalone filesystem database created.
-   [ ] Database open mode, CDB/PDB status, listener, and SQL
    connectivity verified.

## Git hygiene

-   Keep this runbook free of actual database passwords, wallet
    passwords, and private credentials.
-   Store the response file template with placeholder values or generate
    it from a protected local file.
-   Do not commit Oracle installation ZIPs, RU archives, extracted patch
    stages, backups, or installation logs unless there is a specific
    approved reason.
-   Keep the recorded result separate from future changes so another run
    can distinguish observed facts from planned steps.

---

## Reconstruction audit: gaps, additions, and cautions

This section identifies source gaps explicitly. Commands in this section are **proposed reconstruction**, not proof that they were historically executed. Validate them against the exact archive layout, Oracle patch README, and current Oracle documentation before using them on a production system.

### A. Archive integrity checks — proposed additions

The source records ZIP names and approximate sizes but does not record checksum/signature validation. If Oracle provides a checksum for the exact download, compare it before extraction. Example for a locally staged archive (use the checksum supplied by the authoritative download source):

```bash
sha256sum /path/to/archive.zip
```

Do not invent or reuse a checksum. The source does not provide one.

### B. RU extraction command — source gap

The source shows the RU stage directory and checks files under it, but does not preserve the exact extraction command. Inspect the destination first and avoid extracting over an existing stage without review.

```bash
RU="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch/p37260974_19.26_JAN_2025.zip"
RU_STAGE="$HOME/patch_stage/37260974"

ls -lh "$RU"
test -d "$RU_STAGE" && find "$RU_STAGE" -mindepth 1 -maxdepth 2 -print | head -50
```

If the stage directory is absent or has been reviewed and confirmed safe to populate, a reconstructed extraction is:

```bash
mkdir -p "$RU_STAGE"
unzip "$RU" -d "$RU_STAGE"
```

**Not historically verified:** this exact extraction command is a proposed completion of the documented sequence. Confirm whether the archive itself creates a top-level `37260974/` directory; the later source commands expect `README.html` directly under `$RU_STAGE`.

### C. OPatch archive preflight — source gap

The source records the OPatch ZIP path and the later replacement command, but does not show the initial archive listing or extraction test. Before moving the active OPatch directory, inspect the archive and destination:

```bash
OPATCH_ZIP="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch/p6880880_12.2.0.1.45_19cDB.zip"
ORACLE_HOME="/u01/app/oracle/product/19.0.0/dbhome_1"

ls -lh "$OPATCH_ZIP"
unzip -Z1 "$OPATCH_ZIP" | head -30
unzip -Z1 "$OPATCH_ZIP" | grep -E '^OPatch/(opatch|version.txt)$'
test -x "$ORACLE_HOME/OPatch/opatch" && "$ORACLE_HOME/OPatch/opatch" version
test ! -e "$ORACLE_HOME/OPatch_12.2.0.1.17" || {
  echo "STOP: OPatch backup directory already exists; inspect before proceeding."
  exit 1
}
```

The replacement command in Activity 11 should only be run after confirming the archive contains the expected `OPatch/` directory and the backup destination does not already exist. The source's `unzip -o` can overwrite files in the destination; do not use it casually against an unknown or already-populated directory.

### D. Oracle Home backup creation — source gap

The source records the resulting tarball and a passing `gzip -t`, but not the command that created the archive. A candidate command, **not historically verified**, is:

```bash
ORACLE_HOME="/u01/app/oracle/product/19.0.0/dbhome_1"
BACKUP_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software_Backup"
BACKUP="$BACKUP_DIR/oracle19c_home_pre_opatch_$(date +%Y%m%d_%H%M%S).tar.gz"

mkdir -p "$BACKUP_DIR"
test ! -e "$BACKUP" || { echo "STOP: backup target already exists"; exit 1; }
tar -C "$(dirname "$ORACLE_HOME")" -czf "$BACKUP" "$(basename "$ORACLE_HOME")"
gzip -t "$BACKUP"
tar -tzf "$BACKUP" | head -30
```

Check free space on the backup filesystem before creating the archive. A successful gzip test validates archive format integrity, not a tested Oracle Home restore. The historical backup filename and timestamp remain recorded in Activity 10.

### E. Oracle Home ownership — source gap

Activity 03 sets ownership before extraction, while Activity 04 notes that some extracted directories initially had group `svanapal` and were standardized to `svanapal:oinstall`. The source does not show the exact ownership-correction command. A targeted reconstruction after extraction is:

```bash
sudo chown -R svanapal:oinstall /u01/app/oracle/product/19.0.0/dbhome_1
```

Run only after confirming this is the intended Oracle Home and no unrelated files are present beneath it. Do not recursively change ownership of a different or shared path.

### F. Swap-file recovery and fstab warnings

The source records about 8 GiB total swap but also `findmnt --verify` warnings about duplicate `none` targets and a regular-file swap target. Inspect `/etc/fstab` and `swapon --show` before any edits. Keep one valid `/swapfile` entry and retain the pre-existing LVM swap entry. Do not delete or rewrite unrelated mount entries. After a controlled correction, verify with:

```bash
sudo findmnt --verify --verbose
swapon --show
free -h
grep -nE '^[[:space:]]*([^#].*[[:space:]]swap[[:space:]]|/swapfile[[:space:]])' /etc/fstab
```

The source does not establish that all `findmnt` warnings were resolved; treat clean persistence validation as pending until the output is reviewed.

### G. Installer warnings and platform compatibility

- `INS-32047` reported that the central inventory location was not empty. This warning is part of the historical record. Inspect inventory ownership and contents on a new VM; do not delete an existing inventory to suppress the warning.
- `INS-13014` reported unmet optional prerequisites. The source notes `compat-libcap1-1.10` as ignorable in captured checks, but does not provide a comprehensive final prerequisite report.
- `CV_ASSUME_DISTID=OL7` was used as a temporary CVU compatibility override. It does not change the OS identity. Do not make it global without documented justification.
- The source states RU 19.26 was selected for the lab while noting Oracle Linux 9 guidance recommending RU 19.32 or later. Re-check current Oracle certification and RU guidance before repeating this installation elsewhere.
- The source records `transparent_hugepage=madvise` and states guidance for UEK7 and later differs from earlier UEK kernels. Confirm the current guidance for the exact kernel and Oracle release before adopting this setting on another system.

### H. DBCA Clusterware prerequisite failure — unresolved

Recorded error:

```text
[FATAL] [DBT-09101] Target environment does not meet some mandatory requirements.
PRVF-5301 : Failed to locate CRS home
PRVF-9905 : Clusterware is not installed on the following nodes : "orion9"
```

The source does not explain why DBCA's prerequisite invocation performed Clusterware checks for the intended standalone filesystem configuration. Preserve the trace file, inspect the exact command/options, and validate a supported standalone DBCA workflow for the installed Oracle 19c/RU level. Do not silently bypass the check or install Grid Infrastructure contrary to the stated plan.

### I. Final state and work still required

The following are not complete in the source and must remain open:

- Resolve the critical DBCA prerequisite failure using a supported method.
- Re-check current free space on `/`, `/u01`, `/u02`, `/u03`, and `/u04`.
- Decide database name/SID, CDB/PDB names, datafile path, recovery-area strategy, memory allocation for the 8 GiB VM, character sets, listener plan, credentials, and sample-schema requirement.
- Create the database only after the prerequisite failure is resolved.
- Verify database open mode, CDB/PDB status, listener, and SQL connectivity.

## Consolidated final verification checklist

### Operating system and prerequisites

- [ ] Reconfirm OS release, kernel, architecture, hostname/FQDN, memory, swap, filesystem space, and package state.
- [ ] Verify current Oracle certification and RU guidance for this OS/kernel.
- [ ] Confirm `svanapal`, `oinstall`, `dba`, directory ownership, and permissions.
- [ ] Verify effective limits from a fresh `svanapal` login.
- [ ] Verify effective sysctl values and active THP mode.
- [ ] Review `/etc/fstab` warnings and confirm swap persistence without duplicate entries.

### Software Home and patching

- [ ] Confirm base ZIP and patch ZIPs are the intended files; verify checksums where authoritative checksums are available.
- [ ] Confirm Oracle Home path and sufficient space.
- [ ] Confirm OPatch archive contents before replacement; retain and validate the old OPatch backup.
- [ ] Confirm RU README prerequisites and patch stage layout.
- [ ] Confirm response file has the intended software-only values and no secrets.
- [ ] Confirm installer completed and both root scripts ran.
- [ ] Confirm SQL*Plus version, OPatch version, `opatch lspatches`, and `opatch lsinventory`.

### Database creation (not completed)

- [ ] Resolve DBCA `DBT-09101` / `PRVF-5301` / `PRVF-9905` through a supported standalone path.
- [ ] Record database configuration choices and recheck filesystem capacity.
- [ ] Create the standalone filesystem database.
- [ ] Verify database open mode, CDB/PDB state, listener status, and SQL connectivity.

## Source coverage and reconstruction notes

- **Coverage:** The complete uploaded source was read from its first line through its final line. The source contains 1,119 newline-delimited lines (the file retrieval metadata described it as 1,120 file lines); no source range was intentionally skipped.
- **Activity count:** 19 distinct numbered activities, preserved in the original execution order and given an explicit dependency inventory.
- **Duplicate handling:** The source's repeated checks and setup notes were consolidated into the relevant activity or audit section while retaining meaningful historical warnings, corrections, and outputs.
- **Missing/reconstructed steps identified:** checksum verification (where an authoritative checksum exists), exact RU extraction command, OPatch archive preflight, Oracle Home backup-creation command, and exact post-extraction ownership-correction command. These are explicitly labelled proposed and not historically verified.
- **Unresolved issues:** DBCA mandatory Clusterware prerequisite failure; `findmnt --verify` warnings associated with swap persistence; optional installer prerequisite warning; current certification/RU guidance needs revalidation before reuse.
- **Historical-output integrity:** Actual output is retained where present in the source. OPatch version output in Activity 11 is labelled expected in the source and is not upgraded to historical evidence. No database-creation success is claimed.
- **Source ranges not processed:** None identified.
- **Complete source reviewed:** Yes.
