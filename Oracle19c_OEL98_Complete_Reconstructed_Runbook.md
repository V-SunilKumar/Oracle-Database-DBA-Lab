# Oracle Database 19c Silent Installation Runbook — Oracle Linux 9.8

> **Document type:** Reconstructed execution runbook, source audit, and consolidation report  
> **Source file:** `1.1-silent installation-binaries(1).md`  
> **Source coverage:** 6,024 newline-delimited lines read from first to last line. The uploaded source file is preserved unchanged.  
> **Target recorded in source:** `orion9`, Oracle Linux Server 9.8, x86_64, UEK kernel.  
> **Recorded end state:** Oracle Database 19c software-only installation completed with RU 19.26 applied. **Database creation is blocked/pending** because DBCA prerequisite checks failed.

## 1. Purpose

This runbook consolidates repeated discovery, package, configuration, staging, installation, and verification operations into one execution sequence. It preserves meaningful historical findings, successful results, warnings, errors, corrections, and decisions while distinguishing recorded actions from reconstructed guidance.

It is not a claim that every proposed command was historically executed. Use it as an audited lab procedure, and revalidate OS/RU certification and patch instructions before using it on another system or in production.

## 2. Scope and target configuration

| Item | Target / recorded value |
|---|---|
| Host | `orion9` |
| OS | Oracle Linux Server 9.8 |
| Kernel | `6.12.0-203.76.7.3.el9uek.x86_64` |
| Architecture | `x86_64` |
| Installation method | Silent response-file install |
| Installation type | `INSTALL_DB_SWONLY` |
| Base image | `V982063-01_Db.zip` (19.3 Database Home image) |
| RU archive | `p37260974_19.26_JAN_2025.zip` |
| RU patch ID/version | `37260974` / `19.26.0.0.250121` |
| OPatch target | `12.2.0.1.45` (initially `12.2.0.1.17`) |
| Oracle owner | `svanapal` |
| Primary group | `oinstall` |
| Administration group | `dba` |
| Oracle Base | `/u01/app/oracle` |
| Oracle Home | `/u01/app/oracle/product/19.0.0/dbhome_1` |
| Central inventory | `/u01/app/oraInventory` |
| RU stage used for install | `/u02/patch_stage/37260974` |
| Response file | `/home/svanapal/db_install_lab.rsp` |
| Intended database architecture | Single instance, filesystem storage; no Grid Infrastructure, ASM, or Oracle Restart |
| Database status | Not created/verified |

**RU caution:** The source says RU 19.26 was chosen for the lab and also states Oracle Linux 9 guidance recommends RU 19.32 or later. Treat 19.26 as the historical lab choice, not a current universal recommendation. Confirm current Oracle certification and RU guidance for the exact OS, kernel, and intended deployment before reuse.

## 3. Safety and execution conventions

1. Follow **inspect existing state → identify missing/incorrect items → make only necessary changes → verify**.
2. Commands are marked as run by `svanapal`, root, or `sudo`. Do not run Oracle installer/OPatch commands as root unless Oracle's instructions explicitly require it.
3. Do not blindly rerun `groupadd`, `useradd`, `mv`, recursive ownership changes, swap creation, or `/etc/fstab` append commands.
4. Inspect destination directories and existing backups before replacement. Preserve the old OPatch directory and the Oracle Home backup.
5. The shared-folder path contains spaces. Quote variables and confirm the mount exists on the VM.
6. Keep passwords, private credentials, installation ZIPs, RU archives, patch stages, backups, and sensitive logs out of Git unless specifically approved.
7. `CV_ASSUME_DISTID=OL7` was used as a temporary CVU compatibility override for the 19.3 installer on OL9. It does not change the OS release. Do not make it global without justification.
8. `compat-libcap1-1.10` was unavailable in the enabled repositories and was treated as an ignorable prerequisite by the recorded installer. Do not install an arbitrary unrelated RPM just to silence the check.
9. Do not use DBCA's `-ignorePrereqFailure` as an unexamined workaround. The critical Clusterware checks remain unresolved, and Grid Infrastructure is not part of the intended architecture.
10. Historical filesystem sizes and package states are point-in-time results. Re-run them before each new installation.

## 4. Activity index

| Activity | Name | Depends on | Original source lines | Status |
|---|---|---|---:|---|
| [01](#activity-01-confirm-platform-and-installation-route) | Confirm platform and installation route | — | 1–41 | Partially verified |
| [02](#activity-02-inventory-archives-and-existing-installation-state) | Inventory archives and existing installation state | 01 | 42–70, 1062–1142, 1346–1376 | Partially verified |
| [03](#activity-03-ensure-oracle-user-groups-and-directories) | Ensure Oracle user, groups, and directories | 02 | 71–157, 1062–1106 | Verified for recorded state |
| [04](#activity-04-extract-and-verify-the-base-image) | Extract and verify the base image | 03 | 158–232, 1107–1142 | Partially verified |
| [05](#activity-05-consolidated-os-package-and-tool-readiness) | Consolidated OS package and tool readiness | 01 | 233–468, 1586–1630, 3392–3417 | Partially verified |
| [06](#activity-06-configure-oracle-user-resource-limits) | Configure Oracle user resource limits | 03 | 469–740, 926–1035 | Verified for recorded values |
| [07](#activity-07-configure-persistent-kernel-and-network-settings) | Configure persistent kernel and network settings | 01 | 469–865, 926–1035 | Verified for recorded values |
| [08](#activity-08-configure-transparent-hugepages) | Configure Transparent HugePages | 01 | 549–620, 866–1035 | Verified for recorded post-reboot state |
| [09](#activity-09-create-and-validate-the-silent-response-file) | Create and validate the silent response file | 03–04 | 1143–1345 | Verified for selected parameters |
| [10](#activity-10-validate-the-ru-and-opatch-prerequisites) | Validate RU and OPatch prerequisites | 02, 05 | 1346–1630 | Verified for recorded metadata/tool checks |
| [11](#activity-11-back-up-oracle-home-and-preserve-original-opatch) | Back up Oracle Home and preserve original OPatch | 04 | 1631–1846 | Partially verified |
| [12](#activity-12-replace-opatch-and-stage-ru) | Replace OPatch and stage RU | 10–11 | 1847–1972 | Verified for OPatch version and RU metadata |
| [13](#activity-13-run-installer-prerequisite-checks-and-correct-swap) | Run installer prerequisite checks and correct swap | 05–12 | 2034–3545 | Partially verified; optional warning remains |
| [14](#activity-14-install-oracle-19c-and-apply-ru-1926) | Install Oracle 19c and apply RU 19.26 | 01–13 | 3530–3617 | Verified with warnings |
| [15](#activity-15-run-root-scripts-and-review-logs) | Run root scripts and review logs | 14 | 3618–3747 | Partially verified |
| [16](#activity-16-verify-oracle-home-and-patch-inventory) | Verify Oracle Home and patch inventory | 12, 14–15 | 3748–5639, 5641–5728 | Verified for software inventory |
| [17](#activity-17-verify-dbca-executable-and-silent-options) | Verify DBCA executable and silent options | 16 | 5729–5809 | Verified for executable/help |
| [18](#activity-18-investigate-the-dbca-prerequisite-failure) | Investigate DBCA prerequisite failure | 17 | 5810–6021 | Blocked |
| [19](#activity-19-database-creation-and-end-to-end-validation) | Database creation and end-to-end validation | 18 resolved | 6023–6024 and choices not yet recorded | Pending |

## 5. Dependency summary

- **Discovery and identity:** Activities 01–03.
- **OS readiness:** Activities 04–08. Package validation, limits, sysctl, THP, swap, and disk space must be reviewed before installation.
- **Installation inputs:** Activities 09–12. Response file, RU README, OPatch, backup, and RU staging must be validated before installer execution.
- **Install and validate:** Activities 13–16. Run prerequisite checks, apply RU during installation, run root scripts, then verify software inventory.
- **Database creation:** Activities 17–19. DBCA is present, but the prerequisite check failed on critical Clusterware checks. Resolve this with a supported standalone procedure before creating a database.

---

## Activity 01 — Confirm platform and installation route

### Objective

Identify the OS, kernel, architecture, and selected installation method before choosing Oracle 19c installation media and RU.

### Preconditions

Access to `orion9` and a shell. The commands are read-only and may run as `svanapal`.

### Commands

```bash
cat /etc/os-release
uname -r
uname -m
```

### Recorded results

- `PRETTY_NAME="Oracle Linux Server 9.8"`
- Kernel: `6.12.0-203.76.7.3.el9uek.x86_64`
- Architecture: `x86_64`
- The selected method is Method 2: silent installation using a response file.
- The source states Oracle Linux 9 guidance recommends Oracle Database 19c RU 19.32 or later, while this lab uses RU 19.26.

### Explanation

The OS/kernel/architecture identify the target platform for certification and patch compatibility checks. A response-file installation is non-graphical and repeatable, but the response file and installation commands must still match the chosen Oracle Home and RU.

### Validation

Repeat the three commands above on the target VM. Check the current Oracle certification and RU guidance for that exact OS and kernel before reusing the lab RU.

### Recovery or caution

Do not infer Oracle certification solely from `os-release`. Stop if the target OS/architecture differs from the intended environment or current Oracle documentation does not support the planned combination.

### Status

**Partially verified** — historical OS/kernel/architecture output is present; current certification and RU recommendation require revalidation.

---

## Activity 02 — Inventory archives and existing installation state

### Objective

Confirm the Oracle 19c base image, RU ZIP, OPatch ZIP, current Oracle Home state, inventory pointer, and filesystem capacity before changing files.

### Preconditions

Activity 01 completed. The shared-folder mount and exact archive paths must exist on the VM. Run read-only checks as `svanapal`.

### Commands

```bash
echo "===== CURRENT USER ====="
id

echo
echo "===== ORACLE USER AND GROUPS ====="
getent passwd svanapal || true
getent group oinstall || true
getent group dba || true
getent group oper || true

echo
echo "===== BASE IMAGE ====="
ls -lh "/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software/V982063-01_Db.zip"

echo
echo "===== PATCH ARCHIVES ====="
PATCH_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch"
find "$PATCH_DIR" -maxdepth 1 -type f -printf '%f | %s bytes\n' | sort

echo
echo "===== ORACLE HOME AND INVENTORY ====="
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ls -ld /u01 /u01/app /u01/app/oracle "$ORACLE_HOME" /u01/app/oraInventory 2>&1
if [ -f /etc/oraInst.loc ]; then
  cat /etc/oraInst.loc
else
  echo "/etc/oraInst.loc does not exist"
fi

echo
echo "===== FILESYSTEM SPACE ====="
df -hT / /u01 /u02 /u03 /u04 /tmp \
  /media/sf_Sunil-Tech-HyperLab
```

### Recorded results

- Base image: `V982063-01_Db.zip`, approximately 2.9 GiB.
- RU: `p37260974_19.26_JAN_2025.zip`, 1,873,338,568 bytes (about 1.8 GiB).
- OPatch: `p6880880_12.2.0.1.45_19cDB.zip`, 131,979,850 bytes.
- Other files present in the patch folder included `p36912597_19.25_OCT_2024.zip` and `RU_19.24_19c_DB_Linux-x86-64.zip`; they were not the selected RU for this installation.
- Before the first installer run, `/etc/oraInst.loc` did not exist and `/u01/app/oraInventory` was empty. OPatch `lsinventory` returned error 104 because it could not locate a valid `oraInst.loc`; the source correctly associated this with the pre-install state.
- Initial `/u01` space was about 20 GiB total. It was about 14 GiB free after extracting the base image, and about 13 GiB free before installation.
- The shared folder was mounted at `/media/sf_Sunil-Tech-HyperLab`.

### Explanation

Inventory checks prevent operating on the wrong ZIP or an existing Oracle Home. A pre-install OPatch inventory error caused by the absence of the inventory pointer is not, by itself, proof that the OPatch executable is corrupt.

### Validation

Confirm each archive exists, is readable, and has the expected name/size. Check available disk space immediately before extraction and installation. If an Oracle Home already exists, inspect it instead of extracting over it.

### Recovery or caution

- The source does not record a vendor checksum for the base ZIP, RU ZIP, or OPatch ZIP. If an authoritative checksum is available, verify it before extraction; never invent one.
- Do not delete or empty a non-empty central inventory to suppress a warning.
- Quote paths that contain spaces.

### Status

**Partially verified** — archive paths/sizes and pre-install inventory state were recorded. Checksums/signatures were not recorded.

---

## Activity 03 — Ensure Oracle user, groups, and directories

### Objective

Ensure `svanapal`, `oinstall`, and `dba` exist, then establish Oracle Base, Oracle Home, and central inventory with the intended owner/group.

### Preconditions

Activity 02 reviewed. `sudo` access is available. Inspect existing accounts, groups, and paths before making changes.

### Commands

#### 3.1 Inspect existing state

```bash
getent passwd svanapal || true
getent group oinstall || true
getent group dba || true
id svanapal 2>/dev/null || true

ls -ld /u01/app/oracle \
  /u01/app/oracle/product/19.0.0/dbhome_1 \
  /u01/app/oraInventory 2>/dev/null || true

df -hT /u01
```

#### 3.2 Create only missing groups/user

```bash
if ! getent group oinstall >/dev/null; then
  sudo groupadd oinstall
fi

if ! getent group dba >/dev/null; then
  sudo groupadd dba
fi

if ! getent passwd svanapal >/dev/null; then
  sudo useradd -m -g oinstall -G dba -s /bin/bash svanapal
fi

sudo usermod -g oinstall -aG dba svanapal
```

#### 3.3 Create directories and set intended ownership

```bash
sudo mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
sudo mkdir -p /u01/app/oraInventory

sudo chown -R svanapal:oinstall /u01/app/oracle
sudo chown -R svanapal:oinstall /u01/app/oraInventory
sudo chmod -R 775 /u01/app/oracle
sudo chmod -R 775 /u01/app/oraInventory
```

### Recorded results

- `svanapal`: UID 1000.
- `oinstall`: GID 1001.
- `dba`: GID 1002.
- Membership output: `uid=1000(svanapal) gid=1001(oinstall) groups=1001(oinstall),10(wheel),995(vboxsf),1002(dba)`.
- Oracle Home: `/u01/app/oracle/product/19.0.0/dbhome_1`.
- Central inventory: `/u01/app/oraInventory`.
- Before extraction, source output said `oinstall` and `dba` were absent; after creation, membership was verified.

### Explanation

The inventory is outside Oracle Home so it can track Oracle installations independently. Conditional account/group creation supports both a new VM and a VM where the identities already exist.

### Validation

```bash
id svanapal
getent group oinstall
getent group dba
ls -ld /u01/app/oracle \
  /u01/app/oracle/product/19.0.0/dbhome_1 \
  /u01/app/oraInventory
df -h /u01
```

### Recovery or caution

The recursive `chown`/`chmod` commands affect all content under their target paths. Confirm the paths are exactly the intended Oracle directories before running them. If group membership changes, use a new login session before relying on the updated membership.

### Status

**Verified for recorded state** — account/group IDs and directory ownership were observed after configuration.

---

## Activity 04 — Extract and verify the base image

### Objective

Extract the 19.3 Database Home image into the target Oracle Home and verify the installer exists and the extracted tree is owned by `svanapal:oinstall`.

### Preconditions

Activity 03 completed. The base ZIP is readable, `unzip` is available, and `/u01` has sufficient space. Run extraction as `svanapal`, not root.

### Commands

```bash
command -v unzip

ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
BASE_ZIP="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software/V982063-01_Db.zip"

# STOP if the Oracle Home already contains a populated installation.
ls -la "$ORACLE_HOME" | head -30
df -h /u01

# Extract only if the Home is empty or the existing contents have been
# reviewed and intentionally backed up.
unzip "$BASE_ZIP" -d "$ORACLE_HOME"

export ORACLE_BASE ORACLE_HOME

echo "===== INSTALLER ====="
ls -l "$ORACLE_HOME/runInstaller"

echo
echo "===== ORACLE HOME SIZE ====="
du -sh "$ORACLE_HOME"

echo
echo "===== ORACLE HOME OWNERSHIP ====="
ls -ld "$ORACLE_HOME" "$ORACLE_HOME/bin" "$ORACLE_HOME/oui"

echo
echo "===== FILESYSTEM SPACE ====="
df -h /u01
```

If extraction leaves subdirectories owned by `svanapal:svanapal`, the source records this correction:

```bash
sudo chown -R svanapal:oinstall /u01/app/oracle/product/19.0.0/dbhome_1

ls -ld /u01/app/oracle/product/19.0.0/dbhome_1 \
  /u01/app/oracle/product/19.0.0/dbhome_1/bin \
  /u01/app/oracle/product/19.0.0/dbhome_1/oui
```

### Recorded results

- `unzip` was `/usr/bin/unzip`.
- `runInstaller` existed and was executable.
- Extracted Home size was about 6.5 GiB.
- Initial post-extraction output showed Home root owned by `svanapal:oinstall`, while `bin` and `oui` were `svanapal:svanapal`.
- After recursive ownership correction, Home root, `bin`, and `oui` were all shown as `svanapal:oinstall`.
- `/u01` showed 20 GiB total and 14 GiB available after extraction.

### Explanation

The base image populates the software Home; it does not create a database. Ownership was standardized before running the installer.

### Validation

`test -x "$ORACLE_HOME/runInstaller"`, `du -sh "$ORACLE_HOME"`, `ls -ld` on Home and important subdirectories, and `df -h /u01`.

### Recovery or caution

Do not extract over a populated Home without inspecting and backing it up. Archive integrity verification was not recorded. If a vendor checksum is available, validate it before extraction.

### Status

**Partially verified** — extraction, installer presence, Home size, and ownership correction are recorded; checksum verification is absent.

---

## Activity 05 — Consolidated OS package and tool readiness

### Objective

Consolidate the source's repeated package checks into one inspect-first audit, install only packages actually identified as missing and required by the selected guidance, and verify the result.

### Preconditions

Activity 01 completed. Use `svanapal` for read-only checks and `sudo`/root for DNF package changes.

### Commands

#### 5.1 Inspect OS, resources, hostname, tools, and packages

```bash
echo "===== OS / KERNEL / ARCHITECTURE ====="
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
echo "===== HOSTNAME ====="
hostname
hostname -f
getent hosts orion9
getent hosts orion9.sunilit.org

echo
echo "===== BUILD TOOLS ====="
for tool in make ar ld nm; do
  if command -v "$tool" >/dev/null 2>&1; then
    printf "%-8s %s\n" "$tool" "$(command -v "$tool")"
  else
    printf "%-8s MISSING\n" "$tool"
  fi
done

echo
echo "===== CONSOLIDATED PACKAGE SCREEN ====="
for p in \
  bc binutils compat-openssl11 elfutils-libelf fontconfig \
  glibc glibc-devel ksh libaio libaio-devel libX11 libXau libXi \
  libXrender libXtst libxcrypt-compat libgcc libibverbs libnsl \
  libstdc++ libstdc++-devel libxcb libvirt-libs make \
  policycoreutils policycoreutils-python-utils smartmontools \
  sysstat xz-devel net-tools
do
  rpm -q "$p" >/dev/null 2>&1 \
    && printf "INSTALLED  %s\n" "$p" \
    || printf "MISSING    %s\n" "$p"
done

echo
echo "===== PREINSTALL RPM INFORMATION (READ-ONLY) ====="
rpm -q oracle-database-preinstall-19c || true
sudo dnf repolist --enabled
dnf info oracle-database-preinstall-19c
```

#### 5.2 Install packages recorded as initially missing

The source explicitly installed these packages:

```bash
sudo dnf install -y bc ksh libaio-devel sysstat
rpm -q bc ksh libaio-devel sysstat
```

The source later installed `net-tools`:

```bash
sudo dnf list --available net-tools compat-libcap1
sudo dnf install -y net-tools
rpm -q net-tools
command -v netstat
```

The source also proposed installing this additional candidate set after a broader package audit:

```bash
sudo dnf install \
  compat-openssl11 fontconfig \
  libX11 libXau libXi libXrender libXtst \
  libibverbs libnsl libxcb libvirt-libs \
  smartmontools xz-devel
```

**Do not execute that optional batch blindly.** First compare each package against the current Oracle Linux 9 installation guide and the selected RU's prerequisites, then install only packages required and available for the platform.

### Recorded results

Initial package check:

- Missing: `bc`, `ksh`, `libaio-devel`, `sysstat`.
- Present: `binutils`, `elfutils-libelf`, `glibc`, `glibc-devel`, `libaio`, `libgcc`, `libstdc++`, `libstdc++-devel`, `make`.
- In the broader candidate list, the source reported missing: `compat-openssl11`, `fontconfig`, `libX11`, `libXau`, `libXi`, `libXrender`, `libXtst`, `libibverbs`, `libnsl`, `libxcb`, `libvirt-libs`, `smartmontools`, and `xz-devel`.
- The recorded later package install explicitly covered `bc`, `ksh`, `libaio-devel`, `sysstat`, and `net-tools`.
- `net-tools-2.0-0.64.20160912git.el9.x86_64` was installed and `netstat` was found at `/usr/bin/netstat`.
- `compat-libcap1` returned no available package results from enabled repositories.
- `make`, `ar`, `ld`, and `nm` were found under `/usr/bin`.
- Hostname resolution returned `orion9`, FQDN `orion9.sunilit.org`, and IPv4 `192.168.1.81`; IPv6 results were also returned.

Preinstall RPM decision:

- `oracle-database-preinstall-19c` was not installed.
- Source reported a candidate package version `1.0-3.el9` was available.
- The chosen route was manual prerequisite configuration to keep the already-configured `svanapal` account rather than introduce a separate standard `oracle` user.

### Explanation

This is the single authoritative package audit. The source repeated package queries before and after installation; the runbook retains the before/after distinction where it demonstrates a change but avoids repeating the same install commands later.

### Validation

Re-run the consolidated package loop after package changes. Confirm the four initially missing packages and `net-tools` are installed if still required. Recheck `make`, `ar`, `ld`, `nm`, hostname resolution, memory, swap, and disk space.

### Recovery or caution

- A package reported missing by `rpm -q` is not automatically a mandatory dependency for this OS/RU.
- The installer marked `compat-libcap1-1.10` as `IGNORABLE`, but this does not mean the package requirement was satisfied. The source did not locate it in enabled OL9 repositories.
- Do not download an arbitrary old RPM from an untrusted source.
- Package availability can vary by enabled repository and OS point release.

### Status

**Partially verified** — initial missing packages, installation commands, tool locations, and `net-tools` result are recorded; a complete final audit of every candidate package is not shown.

---

## Activity 06 — Configure Oracle user resource limits

### Objective

Persist the source's selected `nofile`, `nproc`, `stack`, and `memlock` limits for `svanapal`, then verify effective values from a new login session.

### Preconditions

Activity 03 completed. Use `sudo` to modify the configuration; log in afresh as `svanapal` to test effective limits.

### Commands

Inspect existing settings first:

```bash
sudo grep -R -nE 'svanapal|oracle|nofile|nproc|stack|memlock' \
  /etc/security/limits.conf /etc/security/limits.d 2>/dev/null
grep -R -n pam_limits.so /etc/pam.d 2>/dev/null
```

The recorded configuration was written to `/etc/security/limits.d/99-oracle-database.conf`. Create or edit it only after reviewing the existing file:

```bash
sudo tee /etc/security/limits.d/99-oracle-database.conf >/dev/null <<'EOF'
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

In a fresh login as `svanapal`:

```bash
ulimit -Sn
ulimit -Hn
ulimit -Su
ulimit -Hu
ulimit -Ss
ulimit -Hs
ulimit -Sl
ulimit -Hl
```

### Recorded results

Initial shell values included:

- Soft open files: `1024`.
- Hard open files: `524288`.
- Soft/hard processes: `31336`.
- Soft stack: `8192 KB`; hard stack: unlimited.
- Soft/hard locked memory: `8192 KB`.

After configuration and re-login, the source recorded:

| Limit | Effective value |
|---|---:|
| Soft `nofile` | 1024 |
| Hard `nofile` | 65536 |
| Soft `nproc` | 16384 |
| Hard `nproc` | 16384 |
| Soft `stack` | 10240 KB |
| Hard `stack` | 32768 KB |
| Soft `memlock` | 3145728 KB |
| Hard `memlock` | 3145728 KB |

The source references Oracle's 19c resource-limit guidance and identifies the initial soft stack as below its cited minimum.

### Explanation

`limits.d` provides persistent per-user limits for PAM-managed login sessions. Changes do not retroactively alter an existing shell; a new login is needed to verify the effective values.

### Validation

Check the file and all eight `ulimit` values from a fresh `svanapal` login. Compare the values with the current Oracle 19c installation guide and the intended HugePages/memory configuration.

### Recovery or caution

The `memlock` value depends on the chosen memory-management/HugePages plan. Do not blindly copy these values to a different host without validating the applicable Oracle requirements. Confirm `pam_limits.so` is active for the login path used by the Oracle owner.

### Status

**Verified for recorded values** — the source captured the persistent file and effective values after re-login.

---

## Activity 07 — Configure persistent kernel and network settings

### Objective

Consolidate repeated sysctl inspection and configuration into one persistent file and verify the active values.

### Preconditions

Activity 01 completed. Use `sudo` for configuration changes. Inspect existing `/etc/sysctl.conf` and `/etc/sysctl.d` entries before modifying the selected file.

### Commands

#### 7.1 Inspect existing persistent settings and current values

```bash
sudo grep -R -nE \
'fs\.aio-max-nr|kernel\.panic_on_oops|net\.ipv4\.ip_local_port_range|net\.core\.(rmem|wmem)_(default|max)|net\.ipv4\.conf\.(all|default)\.rp_filter' \
/etc/sysctl.conf /etc/sysctl.d 2>/dev/null

sysctl fs.aio-max-nr
sysctl kernel.panic_on_oops
sysctl net.ipv4.ip_local_port_range
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
sysctl net.core.rmem_default
sysctl net.core.rmem_max
sysctl net.core.wmem_default
sysctl net.core.wmem_max
```

#### 7.2 Configure the recorded values

The source used `/etc/sysctl.d/98-oracle19c.conf`. If the file exists, review and back it up before replacing its contents.

```bash
sudo tee /etc/sysctl.d/98-oracle19c.conf >/dev/null <<'EOF'
fs.aio-max-nr = 1048576
kernel.panic_on_oops = 1
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF

sudo sysctl -p /etc/sysctl.d/98-oracle19c.conf
```

The source also inspected `kernel.shmmax`, `kernel.shmall`, `kernel.shmmni`, `kernel.sem`, `fs.file-max`, and `kernel.pid_max`. Those values were recorded as:

- `kernel.shmmni = 4096`
- `kernel.sem = 32000 1024000000 500 32000`
- `fs.file-max = 9223372036854775807`
- `kernel.pid_max = 4194304`
- `kernel.shmmax` and `kernel.shmall` were very large values (`18446744073692774399`) on the recorded kernel.

These were assessed in the source against the cited installation minimums; do not override them merely because they were inspected.

### Recorded results

Before configuration:

- `fs.aio-max-nr = 65536`
- `net.ipv4.ip_local_port_range = 32768 60999`
- `net.core.rmem_default = 229376`
- `net.core.rmem_max = 229376`
- `net.core.wmem_default = 229376`
- `net.core.wmem_max = 229376`
- `kernel.panic_on_oops = 1`
- `net.ipv4.conf.all.rp_filter = 0`
- `net.ipv4.conf.default.rp_filter = 1`

After applying the file, the source recorded:

- `fs.aio-max-nr = 1048576`
- `kernel.panic_on_oops = 1`
- `net.ipv4.ip_local_port_range = 9000 65500`
- `net.core.rmem_default = 262144`
- `net.core.rmem_max = 4194304`
- `net.core.wmem_default = 262144`
- `net.core.wmem_max = 1048576`

The source did not include `rp_filter` in the final `98-oracle19c.conf`; its prior values were recorded as `all=0` and `default=1`.

### Explanation

The file under `/etc/sysctl.d` persists selected values and `sysctl -p` applies them immediately. The before/after values are preserved because they demonstrate the configuration change.

### Validation

```bash
sysctl fs.aio-max-nr kernel.panic_on_oops net.ipv4.ip_local_port_range
sysctl net.core.rmem_default net.core.rmem_max
sysctl net.core.wmem_default net.core.wmem_max
cat /etc/sysctl.d/98-oracle19c.conf
```

### Recovery or caution

Values are historical lab settings, not universal defaults. Confirm the exact RU and platform guidance before applying them elsewhere. Do not create a second configuration file with conflicting keys without understanding sysctl load order.

### Status

**Verified for recorded values** — the persistent file was applied and active values were captured.

---

## Activity 08 — Configure Transparent HugePages

### Objective

Change the recorded THP mode from `always` to `madvise` for the UEK 8 lab kernel, persist the boot argument, reboot, and verify the active mode.

### Preconditions

Review current Oracle guidance for the exact Oracle release and UEK kernel. Use `sudo`; schedule a reboot window. Inspect the backup path before copying `/etc/default/grub`.

### Commands

Inspect the current configuration:

```bash
sudo grep -nE 'GRUB_CMDLINE_LINUX|GRUB_ENABLE_BLSCFG' /etc/default/grub
sudo ls -l /boot/loader/entries/
cat /proc/cmdline
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

The source recorded BLS enabled and no THP argument in the original kernel command line. It backed up the GRUB defaults file and used `grubby`:

```bash
# Inspect whether the backup already exists before copying.
sudo test ! -e /etc/default/grub.pre-oracle19c \
  && sudo cp -a /etc/default/grub /etc/default/grub.pre-oracle19c

sudo grubby --update-kernel=ALL --args="transparent_hugepage=madvise"
sudo grubby --info=ALL | grep -E '^(kernel|args)='
```

The source also edited `GRUB_CMDLINE_LINUX` to append `transparent_hugepage=madvise`. Avoid adding the same parameter multiple times; inspect the current line and use one authoritative boot-configuration method consistent with the applicable Oracle Linux 9 guidance.

After the planned reboot:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
cat /proc/cmdline
```

### Recorded results

Before the change:

- THP: `[always] madvise never`
- `defrag`: `always defer defer+madvise [madvise] never`
- BLS was enabled in `/etc/default/grub`.
- Kernel command line did not contain a THP parameter.

After reboot:

- THP: `always [madvise] never`
- Kernel command line included `transparent_hugepage=madvise`.
- `fs.aio-max-nr` and the configured network buffers remained at the values from Activity 07.
- The recorded Oracle user limits remained `1024`, `65536`, `10240`, `32768`, `3145728`, `3145728` for the values checked.

### Explanation

The bracketed value in `/sys/kernel/mm/transparent_hugepage/enabled` is the active THP mode. Standard HugePages are separate from THP; `HugePages_Total: 0` does not mean THP is disabled. The source's explanation notes that standard HugePages need explicit sizing and are incompatible with AMM settings such as `MEMORY_TARGET`/`MEMORY_MAX_TARGET`.

### Validation

Verify both the active THP mode and the boot command line after reboot. Review `/sys/kernel/mm/transparent_hugepage/defrag` separately. If planning standard HugePages, size them from the intended SGA and memory strategy rather than inferring from THP output.

### Recovery or caution

The source states Oracle guidance differs by kernel generation and records `madvise` as appropriate for UEK 7 and later in the referenced guidance. Verify current Oracle guidance for the precise 19c RU and UEK release before adopting this setting elsewhere. Do not overwrite an existing backup.

### Status

**Verified for recorded post-reboot state** — `madvise` and the boot argument were captured after reboot; appropriateness for a different environment needs revalidation.

---

## Activity 09 — Create and validate the silent response file

### Objective

Create a protected copy of the Oracle response template and configure it for software-only Enterprise Edition installation with the established groups and paths.

### Preconditions

Activities 03–04 completed. Run as `svanapal`. The Oracle-supplied response template must exist and remain unchanged.

### Commands

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
RSP="$HOME/db_install_lab.rsp"

# Inspect destination first; do not overwrite a response file with local edits.
if [ -e "$RSP" ]; then
  echo "STOP: $RSP already exists; review and back it up before replacing."
else
  cp "$ORACLE_HOME/install/response/db_install.rsp" "$RSP"
  chmod 600 "$RSP"
fi

ls -l "$RSP"
```

The source configured the following values using `sed`:

```bash
sed -i \
  -e 's|^oracle.install.option=.*|oracle.install.option=INSTALL_DB_SWONLY|' \
  -e 's|^UNIX_GROUP_NAME=.*|UNIX_GROUP_NAME=oinstall|' \
  -e 's|^INVENTORY_LOCATION=.*|INVENTORY_LOCATION=/u01/app/oraInventory|' \
  -e "s|^ORACLE_HOME=.*|ORACLE_HOME=$ORACLE_HOME|" \
  -e 's|^ORACLE_BASE=.*|ORACLE_BASE=/u01/app/oracle|' \
  -e 's|^oracle.install.db.InstallEdition=.*|oracle.install.db.InstallEdition=EE|' \
  -e 's|^oracle.install.db.OSDBA_GROUP=.*|oracle.install.db.OSDBA_GROUP=dba|' \
  -e 's|^oracle.install.db.OSOPER_GROUP=.*|oracle.install.db.OSOPER_GROUP=dba|' \
  -e 's|^oracle.install.db.OSBACKUPDBA_GROUP=.*|oracle.install.db.OSBACKUPDBA_GROUP=dba|' \
  -e 's|^oracle.install.db.OSDGDBA_GROUP=.*|oracle.install.db.OSDGDBA_GROUP=dba|' \
  -e 's|^oracle.install.db.OSKMDBA_GROUP=.*|oracle.install.db.OSKMDBA_GROUP=dba|' \
  -e 's|^oracle.install.db.OSRACDBA_GROUP=.*|oracle.install.db.OSRACDBA_GROUP=dba|' \
  "$RSP"
```

Validate selected parameters and duplicates:

```bash
grep -nE '^(oracle.install.option|UNIX_GROUP_NAME|INVENTORY_LOCATION|ORACLE_HOME|ORACLE_BASE|oracle.install.db.InstallEdition|oracle.install.db.OSDBA_GROUP|oracle.install.db.OSOPER_GROUP|oracle.install.db.OSBACKUPDBA_GROUP|oracle.install.db.OSDGDBA_GROUP|oracle.install.db.OSKMDBA_GROUP|oracle.install.db.OSRACDBA_GROUP)=' "$RSP"

grep -E '^(oracle.install.option|UNIX_GROUP_NAME|INVENTORY_LOCATION|ORACLE_HOME|ORACLE_BASE|oracle.install.db.InstallEdition|oracle.install.db.OSDBA_GROUP|oracle.install.db.OSOPER_GROUP|oracle.install.db.OSBACKUPDBA_GROUP|oracle.install.db.OSDGDBA_GROUP|oracle.install.db.OSKMDBA_GROUP|oracle.install.db.OSRACDBA_GROUP)=' "$RSP" \
  | cut -d= -f1 | sort | uniq -c

echo "===== DATABASE-CREATION AND ROOT OPTIONS ====="
grep -niE 'createAsContainerDatabase|GDBNAME|SID|PDBNAME|storageType|dataLocation|recoveryLocation|rootconfig|executeRootScript|configMethod' "$RSP"
```

### Recorded results

The 12 selected parameters appeared once each:

```properties
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

The source inspected but did not set the starter database name/SID, PDB name, storage type, data location, recovery location, password, or root-script configuration fields. The selected `INSTALL_DB_SWONLY` option means the installer should not create a starter database.

### Explanation

A separate response file protects the original Oracle template and makes the installation repeatable. The file is restricted to mode `600` because Oracle response files can contain sensitive settings.

### Validation

Confirm all 12 selected values and their occurrence counts. Review active response-file settings and ensure there are no contradictory database-creation options. Verify the file is readable by `svanapal` and has restrictive permissions.

### Recovery or caution

Do not commit actual credentials. Do not assume blank starter-database fields are configured; database creation is a separate DBCA step. `sed` substitutions expect the Oracle template's parameter lines to exist in the format shown.

### Status

**Verified for selected parameters** — the source shows each of the 12 parameters once; full response-file correctness is not established.

---

## Activity 10 — Validate the RU and OPatch prerequisites

### Objective

Inspect the selected RU archive, read its README, check the OPatch minimum, and verify required build tools before modifying OPatch or installing the RU.

### Preconditions

Activities 02 and 05 completed. Run checks as `svanapal`. The exact selected RU ZIP must be readable.

### Commands

```bash
PATCH_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch"
RU="$PATCH_DIR/p37260974_19.26_JAN_2025.zip"
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

ls -lh "$RU"
df -h "$PATCH_DIR"

echo "===== RU ARCHIVE TOP LEVEL ====="
unzip -Z1 "$RU" | awk -F/ 'NF>0 {print $1}' | sort -u | head -30

echo
echo "===== README PATHS ====="
unzip -Z1 "$RU" | grep -iE '(^|/)README[^/]*\.html?$'

echo
echo "===== RU README.TXT ====="
unzip -p "$RU" 37260974/README.txt | head -100

echo
echo "===== BUILD TOOLS ====="
for tool in make ar ld nm; do
  command -v "$tool" || true
done

echo
echo "===== CURRENT OPATCH ====="
"$ORACLE_HOME/OPatch/opatch" version
```

The archive's `README.txt` redirects to `README.html`. The source used the following command to read the relevant prerequisite section from the HTML without extracting it:

```bash
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

### Recorded results

- RU ZIP: 1.8 GiB, 1,873,338,568 bytes.
- Top-level archive entries included `37260974` and `PatchSearch.xml`.
- Metadata files included `37260974/etc/config/inventory.xml` and `37260974/etc/config/actions.xml`.
- README files included `37260974/README.txt` and `37260974/README.html`; the text README says `Refer to README.html`.
- The RU README identified OPatch `12.2.0.1.44` or later as the minimum.
- `make`, `ar`, `ld`, and `nm` were available under `/usr/bin`.
- The initial Oracle Home OPatch version was `12.2.0.1.17`, below the RU-specific minimum.
- The selected OPatch ZIP was `p6880880_12.2.0.1.45_19cDB.zip`.

### Explanation

The RU must be applied to the Database Home, not a Grid Infrastructure Home. Patch-specific instructions and OPatch minimum must be checked before staging or applying the patch.

### Validation

Check the selected archive, README path, patch ID/version, build tools, and the OPatch version that will actually be used. An archive listing is not proof that the patch has been applied.

### Recovery or caution

Do not proceed if the patch README cannot be read, the patch ID/version does not match the intended RU, or OPatch is below the documented minimum. The source did not record authoritative archive checksums.

### Status

**Verified for recorded metadata/tool checks** — patch ID/version and required tools were identified; certification/current RU recommendations must be rechecked for reuse.

---

## Activity 11 — Back up Oracle Home and preserve original OPatch

### Objective

Create and validate an Oracle Home backup before replacing OPatch, and preserve the original OPatch directory separately.

### Preconditions

Activity 04 completed; Oracle Home exists. The shared-folder backup destination is mounted and writable, with sufficient free space. Run as `svanapal`.

### Commands

#### 11.1 Inspect Oracle Home, inventory, and backup destination

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
BACKUP_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Software_Backup"

ls -ld "$ORACLE_HOME" "$ORACLE_HOME/OPatch" /u01/app/oraInventory
find /u01/app/oraInventory -maxdepth 2 -type f -printf '%p\n'
df -h "$BACKUP_DIR" 2>/dev/null || df -h /media/sf_Sunil-Tech-HyperLab

mkdir -p "$BACKUP_DIR"
touch "$BACKUP_DIR/.write_test" &&
rm "$BACKUP_DIR/.write_test" &&
echo "Write test passed"
```

#### 11.2 Create a unique backup

```bash
BACKUP_FILE="$BACKUP_DIR/oracle19c_home_pre_opatch_$(date +%Y%m%d_%H%M%S).tar.gz"

test ! -e "$BACKUP_FILE" || {
  echo "STOP: backup file already exists: $BACKUP_FILE"
  exit 1
}

tar -czf "$BACKUP_FILE" \
  -C "$(dirname "$ORACLE_HOME")" \
  "$(basename "$ORACLE_HOME")"

if [ $? -eq 0 ]; then
  ls -lh "$BACKUP_FILE"
  gzip -t "$BACKUP_FILE" && echo "Archive integrity check passed"
else
  echo "BACKUP FAILED — do not proceed with OPatch replacement"
  exit 1
fi
```

#### 11.3 Preserve original OPatch

```bash
# Stop if the backup destination already exists; inspect it before proceeding.
if [ -e "$HOME/OPatch_19.3_original" ]; then
  echo "STOP: $HOME/OPatch_19.3_original already exists; inspect it."
else
  cp -a "$ORACLE_HOME/OPatch" "$HOME/OPatch_19.3_original"
fi

"$HOME/OPatch_19.3_original/opatch" version
```

### Recorded results

- Backup directory on the shared folder passed a write test.
- Oracle Home size before backup was about 6.5 GiB.
- Oracle Home backup created: `oracle19c_home_pre_opatch_20261016_134941.tar.gz`, about 2.9 GiB.
- `gzip -t` passed and output said `Archive integrity check passed`.
- Original OPatch directory was copied to `/home/svanapal/OPatch_19.3_original`.
- Original OPatch version was `12.2.0.1.17`.
- Before installation, `/etc/oraInst.loc` did not exist and the central inventory directory was empty. OPatch `lsinventory -detail` failed with code 104 because it could not find a valid `oraInst.loc`; the source recognized this as the pre-install state.

### Explanation

The compressed archive is a rollback artifact before the OPatch replacement. The separate OPatch copy enables a more targeted recovery. The pre-install `lsinventory` error is retained as historical evidence and is not treated as a failure of the base ZIP extraction.

### Validation

Run `gzip -t "$BACKUP_FILE"` and inspect `tar -tzf "$BACKUP_FILE" | head -30`. Confirm the old OPatch copy reports `12.2.0.1.17`. A gzip integrity test is not a full restore test.

### Recovery or caution

Do not overwrite an existing backup. Check free space first. The source recorded an archive of Oracle Home but did not record a tested restoration. The central inventory was not separately included in the recorded tar command; if following Oracle patch guidance that calls for inventory backup too, design and validate that backup separately.

### Status

**Partially verified** — backup creation, archive integrity, and original OPatch copy are recorded; a restore test is absent.

---

## Activity 12 — Replace OPatch and stage RU

### Objective

Replace OPatch with version `12.2.0.1.45`, verify the active version, and extract the RU to a staging directory suitable for `-applyRU`.

### Preconditions

Activities 10–11 completed. Oracle Home backup and original OPatch copy are validated. Run as `svanapal`. Inspect destinations before moving/copying.

### Commands

#### 12.1 Stage and verify OPatch

```bash
PATCH_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch"
OPATCH_ZIP="$PATCH_DIR/p6880880_12.2.0.1.45_19cDB.zip"
STAGE="$HOME/opatch_stage"

unzip -l "$OPATCH_ZIP" | head -25
mkdir -p "$STAGE"
unzip -q "$OPATCH_ZIP" -d "$STAGE"
"$STAGE/OPatch/opatch" version
ls -ld "$STAGE/OPatch"
```

#### 12.2 Replace OPatch only after checking destination state

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
STAGED_OPATCH="$HOME/opatch_stage/OPatch"

"$STAGED_OPATCH/opatch" version

# Inspect these paths first. Do not overwrite a prior backup directory.
ls -ld "$ORACLE_HOME/OPatch" "$ORACLE_HOME/OPatch.pre_12.2.0.1.45" 2>/dev/null || true

# Execute only if active OPatch exists and the destination does not.
if [ -d "$ORACLE_HOME/OPatch" ] &&
   [ ! -e "$ORACLE_HOME/OPatch.pre_12.2.0.1.45" ]; then
  mv "$ORACLE_HOME/OPatch" "$ORACLE_HOME/OPatch.pre_12.2.0.1.45"
  cp -a "$STAGED_OPATCH" "$ORACLE_HOME/OPatch"
else
  echo "STOP: inspect OPatch and backup paths before replacement."
fi

"$ORACLE_HOME/OPatch/opatch" version
```

#### 12.3 Extract RU to a staging directory

The source first staged the RU under the home directory and later moved it to `/u02` because root filesystem free space fell to about 5.3 GiB.

```bash
PATCH_DIR="/media/sf_Sunil-Tech-HyperLab/ISO/Oracle-Softwares/Oracle_Patches/19300_Patches/DB_Patch"
RU_ZIP="$PATCH_DIR/p37260974_19.26_JAN_2025.zip"
HOME_STAGE="$HOME/patch_stage"

mkdir -p "$HOME_STAGE"
# Only extract if the destination is absent or its contents have been reviewed.
unzip -q "$RU_ZIP" -d "$HOME_STAGE"

ls -ld "$HOME_STAGE/37260974"
ls -lh "$HOME_STAGE/37260974/README.html"
du -sh "$HOME_STAGE/37260974"
df -h "$HOME"
```

#### 12.4 Move/copy stage to `/u02` only after checking capacity

The source recorded this copy-and-compare approach:

```bash
sudo mkdir -p /u02/patch_stage
sudo chown svanapal:oinstall /u02/patch_stage
sudo chmod 775 /u02/patch_stage

# Run as svanapal; stop if destination already exists and inspect it first.
cp -a "$HOME/patch_stage/37260974" /u02/patch_stage/

du -sh "$HOME/patch_stage/37260974" /u02/patch_stage/37260974
diff -qr "$HOME/patch_stage/37260974" /u02/patch_stage/37260974
```

A zero exit status from `diff -qr` means no differences were reported; nonzero means investigate before deleting the original. The source subsequently removed the home-directory copy after comparing and used `/u02/patch_stage/37260974` for installation. **Do not run `rm -rf` until the destination has been verified and the path is exactly correct.**

### Recorded results

- Staged OPatch reported `OPatch Version: 12.2.0.1.45`; `OPatch succeeded.`
- Active Oracle Home OPatch then reported `12.2.0.1.45`.
- The original directory was preserved as `OPatch.pre_12.2.0.1.45`; another copy existed at `$HOME/OPatch_19.3_original`.
- RU extracted to `/home/svanapal/patch_stage/37260974`; size about 4.3 GiB.
- The stage contained `README.html`, `etc/config/inventory.xml`, and `etc/config/actions.xml`; README metadata contained `19.26.0.0.250121` and `37260974`.
- Due to root filesystem space pressure (about 5.3 GiB free while RU stage occupied 4.3 GiB), the patch stage was moved/copied to `/u02/patch_stage/37260974`.
- Before install, `/u01` had about 13 GiB free and `/` about 12 GiB free after the stage move.

### Explanation

The RU staging path used by `runInstaller -applyRU` must point to the extracted patch directory, not the ZIP. Staging under `/u02` avoids consuming root filesystem space. The active OPatch version meets the RU README's recorded minimum of `12.2.0.1.44`.

### Validation

Confirm active OPatch version `12.2.0.1.45`, patch metadata files, README version/patch ID, stage size, ownership, and free space. Confirm `/u02/patch_stage/37260974` is the intended install argument.

### Recovery or caution

- Never move an existing OPatch directory if the backup destination already exists without investigating.
- Do not run the `cp`/`rm` sequence against an unknown stage path.
- The source's first RU extraction command is recorded; no archive checksum is recorded.
- Do not apply the RU to a Grid Infrastructure Home.

### Status

**Verified for OPatch version and RU metadata** — the active OPatch version and staged patch metadata are recorded. Verify the stage destination on any new run.

---

## Activity 13 — Run installer prerequisite checks and correct swap

### Objective

Run the Oracle installer prerequisite check, investigate warnings, raise active swap to the recorded approximately 8 GiB target, persist the swap file, and verify it before installation.

### Preconditions

Activities 05–12 reviewed. Run the installer as `svanapal`; run swap/LVM changes as root. Review `/etc/fstab`, active swap, and disk space before creating a file.

### Commands

#### 13.1 Run installer prerequisite check

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export CV_ASSUME_DISTID=OL7
cd "$ORACLE_HOME"

./runInstaller -silent -executePrereqs \
  -responseFile "$HOME/db_install_lab.rsp"
```

#### 13.2 Inspect prerequisite logs

Use the actual log path printed by the installer. The source captured these log locations at different attempts:

- `/tmp/InstallActions2026-10-16_02-56-08PM/`
- `/tmp/InstallActions2026-10-16_07-23-46PM/`
- `/tmp/InstallActions2026-10-16_07-47-12PM/`
- `/tmp/InstallActions2026-10-16_07-58-34PM/`

Example:

```bash
LOG=/tmp/InstallActions2026-10-16_07-47-12PM/installActions2026-10-16_07-47-12PM.log

grep -n -B 3 -A 5 -E \
'PRVF-|INS-13014|failed Tasks|OverallStatus:VERIFICATION_FAILED|Verification Result' \
"$LOG"

grep -n -E \
'compat-libcap1|net-tools|Swap Size|SUCCESS_WITH_WARNINGS|Exit Status' \
"$LOG"
```

#### 13.3 Inspect swap, filesystems, and LVM before changing swap

```bash
free -h
swapon --show
df -h / /u01 /u02 /u03 /u04 /tmp
grep -vE '^[[:space:]]*(#|$)' /etc/fstab

# Run as root:
lvs -o lv_name,vg_name,lv_size,lv_path
vgs -o vg_name,vg_size,vg_free
pvs -o pv_name,vg_name,pv_size,pv_free
findmnt -no FSTYPE /
```

#### 13.4 Create and activate a 6 GiB swap file only if absent

The source found an existing 2 GiB swap LV (`/dev/mapper/ol-swap`) and only 4 MiB free in VG `ol`; it chose a swap file rather than resizing LVM.

```bash
# Run as root. Inspect /swapfile first; do not recreate an existing file.
if [ -e /swapfile ]; then
  echo "STOP: /swapfile already exists; inspect it and swapon --show."
else
  dd if=/dev/zero of=/swapfile bs=1M count=6144 status=progress
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
fi

free -h
swapon --show
df -h /
```

#### 13.5 Add persistence only if the entry is absent

```bash
# Run as root.
grep -nE '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab

# Only if the previous command returns no line:
echo '/swapfile none swap defaults 0 0' >> /etc/fstab

grep -nE '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab
findmnt --verify --verbose
swapon --show
free -h
```

#### 13.6 Re-run the prerequisite check after swap and package corrections

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export CV_ASSUME_DISTID=OL7
cd "$ORACLE_HOME"

free -h
swapon --show

./runInstaller -silent -executePrereqs \
  -responseFile "$HOME/db_install_lab.rsp"
```

### Recorded results

Initial installer prerequisite findings:

- Swap: 2 GiB active; installer reported approximately `7.7277 GiB` required.
- `compat-libcap1-1.10`: missing/ignorable in the captured installer checks.
- `net-tools`: missing in the initial check.
- Installer result: `INS-13014`, `SUCCESS_WITH_WARNINGS`; all three findings were classified as ignorable by the captured prerequisite workflow.

Swap investigation and correction:

- LVM swap: `/dev/mapper/ol-swap`, 2 GiB.
- VG `ol`: 67 GiB total, 4 MiB free.
- Root filesystem was XFS, with about 12 GiB available before creating the swap file.
- Created `/swapfile` with `dd ... bs=1M count=6144`, `chmod 600`, `mkswap`, and `swapon`.
- Active swap afterward: 2 GiB partition + 6 GiB file = 8 GiB total.
- `/etc/fstab` entry recorded: `/swapfile none swap defaults 0 0`.
- `findmnt --verify --verbose` reported 0 parse errors, 0 errors, 2 warnings:
  - target `none` specified more than once for the existing LVM swap and the swap file;
  - `/swapfile` is a regular file, so the ordinary mount validation warning is not a complete swap-file validation.
- `swapon --show` confirmed both swap devices active.
- After `net-tools` was installed, the subsequent prerequisite log still reported `compat-libcap1` as the remaining failed/ignorable package check. The source did not install an arbitrary RPM for it.

### Explanation

The additional swap was created because the installer reported a swap requirement higher than the existing 2 GiB. The LVM volume group had virtually no free extents, so the lab used a swap file on `/`. The `/etc/fstab` entry makes the file persistent across reboot.

### Validation

Confirm `free -h`, `swapon --show`, the single `/swapfile` entry, root filesystem space, and installer log result. A warning-only prerequisite result must be reviewed; it is not equivalent to every prerequisite being satisfied.

### Recovery or caution

- Do not recreate an existing swap file or append duplicate `/etc/fstab` entries.
- Inspect the existing LVM swap entry and preserve it.
- Do not treat `findmnt` warnings as a fatal error without interpretation, but do confirm swap activation and persistence.
- `CV_ASSUME_DISTID=OL7` is temporary and should be scoped to the shell running the installer.
- `compat-libcap1` remained unavailable in enabled repositories; do not download an arbitrary old package.

### Status

**Partially verified** — active swap and persistence were recorded; installer still reported optional prerequisite warnings.

---

## Activity 14 — Install Oracle 19c and apply RU 19.26

### Objective

Run the silent software-only installer and apply RU 19.26 during the installation using the extracted RU stage.

### Preconditions

Activities 01–13 completed or reviewed. Verify current disk space, installer executable, response file, RU metadata, OPatch version, and stage path. Run as `svanapal`, not root.

### Commands

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_BASE=/u01/app/oracle
export CV_ASSUME_DISTID=OL7
RU_STAGE=/u02/patch_stage/37260974
RSP="$HOME/db_install_lab.rsp"

echo "===== PRE-INSTALL SPACE ====="
df -h / /u01 /u02 /tmp

echo
echo "===== INPUT VALIDATION ====="
test -x "$ORACLE_HOME/runInstaller" && echo "runInstaller executable"
test -r "$RSP" && echo "response file readable"
test -r "$RU_STAGE/etc/config/inventory.xml" && echo "RU metadata readable"
"$ORACLE_HOME/OPatch/opatch" version

echo
echo "===== INSTALLER HELP ====="
cd "$ORACLE_HOME"
./runInstaller -help 2>&1 |
  grep -iE -A2 -B2 'applyRU|responseFile|silent|waitforcompletion|ignorePrereq'
```

The source's installation command:

```bash
./runInstaller \
  -silent \
  -applyRU /u02/patch_stage/37260974 \
  -responseFile "$HOME/db_install_lab.rsp" \
  -waitforcompletion
```

### Recorded results

The source captured:

- `Applying the patch /u02/patch_stage/37260974...`
- `Successfully applied the patch.`
- Warning `INS-32047`: the central inventory location `/u01/app/oraInventory` was not empty.
- Warning `INS-13014`: some optional prerequisites were not met.
- Installer reported `Successfully Setup Software with warning(s).`
- Installer instructed root to run:
  1. `/u01/app/oraInventory/orainstRoot.sh`
  2. `/u01/app/oracle/product/19.0.0/dbhome_1/root.sh`

### Explanation

`-applyRU` applies RU 19.26 during the software installation. This remains a software-only installation because the response file sets `INSTALL_DB_SWONLY`; it does not create a database or configure a listener.

### Validation

Review the full installer result and log path printed by the installer. After Activity 15, run `sqlplus -V`, `opatch version`, `opatch lspatches`, and `opatch lsinventory`.

### Recovery or caution

- Preserve `INS-32047` and `INS-13014` in the audit record.
- Do not delete a non-empty central inventory to suppress `INS-32047`.
- Do not claim a clean prerequisite pass: installation completed with warnings.
- Do not run root scripts until the installer has completed and listed them.

### Status

**Verified with warnings** — the installer reported successful RU application and software setup with warnings.

---

## Activity 15 — Run root scripts and review logs

### Objective

Complete the required root actions after installation and inspect the root-script log without overstating what log greps prove.

### Preconditions

Activity 14 completed and named the two root scripts. Run as root, in the installer-specified order.

### Commands

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

Review the recorded root log:

```bash
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
LOG="$ORACLE_HOME/install/root_orion9_2026-10-16_20-51-43-302697673.log"

ls -lh "$LOG"
grep -niE 'error|failed|fatal|exception|successfully completed' "$LOG" \
  || echo "No matching error or completion messages found in this log."

cat /etc/oratab 2>/dev/null
ls -l /usr/local/bin/dbhome /usr/local/bin/oraenv /usr/local/bin/coraenv 2>/dev/null
```

### Recorded results

- `orainstRoot.sh` ran and changed central inventory permissions/group.
- `root.sh` created `/etc/oratab` and copied `dbhome`, `oraenv`, and `coraenv` to `/usr/local/bin`.
- Root log path: `/u01/app/oracle/product/19.0.0/dbhome_1/install/root_orion9_2026-10-16_20-51-43-302697673.log`.
- A grep for `error|failed|fatal|exception|successfully completed` returned: `No matching error or completion messages found in this log.`

### Explanation

The root scripts are part of completing the software installation. The log grep was the check recorded in the source, but a lack of matching words does not prove every operation was functionally successful.

### Validation

Check the exit result of both scripts, inspect the root log and inventory permissions, and verify `/etc/oratab` and the environment helper scripts exist.

### Recovery or caution

If either script exits nonzero, stop and review its full output and log before retrying. Do not infer success from a process no longer running or from a keyword grep alone.

### Status

**Partially verified** — both root scripts were run and log/output checks were recorded; functional validation was not exhaustive.

---

## Activity 16 — Verify Oracle Home and patch inventory

### Objective

Confirm the installed SQL*Plus version, OPatch version, RU patch ID, inventory pointer, and Oracle executable.

### Preconditions

Activities 12, 14, and 15 completed. Run as `svanapal`.

### Commands

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_BASE=/u01/app/oracle
export PATH="$ORACLE_HOME/bin:$PATH"

echo "===== SQL*PLUS VERSION ====="
"$ORACLE_HOME/bin/sqlplus" -V

echo
echo "===== OPATCH VERSION ====="
"$ORACLE_HOME/OPatch/opatch" version

echo
echo "===== INSTALLED PATCHES ====="
"$ORACLE_HOME/OPatch/opatch" lspatches

echo
echo "===== INVENTORY SUMMARY ====="
"$ORACLE_HOME/OPatch/opatch" lsinventory -oh "$ORACLE_HOME" | tail -30

echo
echo "===== ORACLE EXECUTABLE ====="
ls -lh "$ORACLE_HOME/bin/oracle"

echo
echo "===== FILESYSTEM SPACE ====="
df -h "$ORACLE_HOME"
```

### Recorded results

```text
SQL*Plus: Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

37260974;Database Release Update : 19.26.0.0.250121 (37260974)
29585399;OCW RELEASE UPDATE 19.3.0.0.0 (29585399)

OPatch Version: 12.2.0.1.45
OPatch succeeded.
```

Additional recorded inventory details:

- Oracle Home: `/u01/app/oracle/product/19.0.0/dbhome_1`.
- Central inventory: `/u01/app/oraInventory`, located from `$ORACLE_HOME/oraInst.loc`.
- OPatch: `12.2.0.1.45`.
- OUI version: `12.2.0.7.0`.
- Oracle executable: `/u01/app/oracle/product/19.0.0/dbhome_1/bin/oracle`, about 441 MiB.
- RU patch `37260974` was shown as applied on Fri Oct 16 20:30:01 IST 2026.
- The source includes a very large `lsinventory` output listing base component patch IDs. The complete repetitive patch-ID dump is not copied into this runbook; the verification command and the decisive RU/OCW entries are preserved here, with source traceability to lines 3748–5639.

### Explanation

These checks confirm the patched software Home and inventory. They do not prove that a database exists or can be opened, because the installation was software-only.

### Validation

Confirm SQL*Plus reports `19.26.0.0.0`, OPatch reports `12.2.0.1.45`, `lspatches` lists RU `37260974`, and inventory points to the intended Home and central inventory.

### Recovery or caution

If inventory reports a different Home, missing RU, or errors, stop before DBCA. Do not interpret the base component patch list as evidence that database creation has succeeded.

### Status

**Verified for software version and patch inventory** — SQL*Plus, OPatch, and patch IDs were recorded.

---

## Activity 17 — Verify DBCA executable and silent options

### Objective

Confirm DBCA exists and exposes silent database-creation options for a single-instance filesystem database.

### Preconditions

Activity 16 completed. Run as `svanapal`.

### Commands

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_BASE=/u01/app/oracle
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -version
ls -lh "$ORACLE_HOME/bin/dbca"
"$ORACLE_HOME/bin/dbca" -help
```

### Recorded results

- DBCA executable exists at `$ORACLE_HOME/bin/dbca`.
- File size recorded: about 7.6 KiB.
- `dbca -version` printed `DISPLAY not set. Set DISPLAY environment variable, then re-run.` This is not evidence that DBCA is missing.
- `dbca -help` listed `-silent`, `-createDatabase`, and `-executePrereqs`, as well as `-ignorePreReqs` and `-ignorePrereqFailure`.
- The source earlier notes that the create-database help exposed `-databaseConfigType SINGLE` and `-storageType FS`; the final source ends with a heading to check DBCA's supported database-creation options, but has no captured result under that final heading.

### Explanation

DBCA is available, and silent mode is supported. However, presence of the executable/help flags does not prove that the prerequisite check passes or that database creation is successful.

### Validation

Use the silent help output to confirm supported options for the exact installed RU. Review the complete `-createDatabase` help before preparing a creation command.

### Recovery or caution

Do not treat `DISPLAY not set` as an installation failure. Do not assume that the `-ignorePrereqFailure` option is a safe workaround for a critical prerequisite error.

### Status

**Verified for executable/help options** — DBCA exists and help output is recorded; a successful database-creation run is not established.

---

## Activity 18 — Investigate the DBCA prerequisite failure

### Objective

Preserve and diagnose the critical failure from the single-instance DBCA prerequisite check without introducing Grid Infrastructure or bypassing the check blindly.

### Preconditions

Activity 17 completed. Keep the exact trace log. Intended architecture remains single instance with filesystem storage and no Grid Infrastructure.

### Commands

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_BASE=/u01/app/oracle
export PATH="$ORACLE_HOME/bin:$PATH"

"$ORACLE_HOME/bin/dbca" -silent -executePrereqs \
  -databaseConfigType SINGLE
```

The source then inspected the trace:

```bash
LOG=/u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-16_09-14-13PM

grep -n -i -E 'FATAL|ERROR|FAILED|mandatory|DBT-|PRVF-|PRVG-' "$LOG" | tail -80
tail -100 "$LOG"
```

### Recorded results

DBCA returned:

```text
[FATAL] [DBT-09101] Target environment does not meet some mandatory requirements.
CAUSE: Some of the mandatory prerequisites are not met.
Trace: /u01/app/oracle/cfgtoollogs/dbca/trace.log_2026-10-16_09-14-13PM
```

The trace records:

```text
PRVF-5301 : Failed to locate CRS home
PRVF-9905 : Clusterware is not installed on the following nodes : "orion9"
TaskCRSIntegrity: ... FAILED: CRITICAL: OPERATION_FAILED
TaskNodeAppCreation: ... FAILED: CRITICAL: OPERATION_FAILED
OverallStatus: VERIFICATION_FAILED
OverallStatus: OPERATION_FAILED
```

The trace also reports `compat-libcap1-1.10` as `Severity: IGNORABLE`; the critical tasks are CRS Integrity and Node Application Existence.

### Explanation

The current plan does not include Grid Infrastructure. The trace does not establish why DBCA's single-instance prerequisite check invoked critical Clusterware checks, nor does it establish a supported workaround. The cause remains unresolved.

### Validation

Keep the full trace. Confirm the exact DBCA command and options, review the response/help for the installed Oracle 19c RU, and identify a supported procedure for standalone filesystem database creation. Re-run the prerequisite check only after the issue is understood and the intended configuration is unchanged.

### Recovery or caution

- **Blocked:** do not create the database until a supported resolution is identified.
- Do not blindly append `-ignorePrereqFailure` or `-ignorePreReqs`.
- Do not install Grid Infrastructure just to silence the check; that would change the intended architecture.
- Do not assume `compat-libcap1` caused the fatal DBCA failure; the trace labels it ignorable and separately identifies two critical Clusterware checks.

### Status

**Blocked** — mandatory DBCA checks failed with `DBT-09101`, `PRVF-5301`, and `PRVF-9905`; no supported resolution is recorded.

---

## Activity 19 — Database creation and end-to-end validation

### Objective

Complete a standalone filesystem database only after the DBCA blocker is resolved and database design choices are documented.

### Preconditions

Activity 18 must be resolved through a supported procedure. Re-check disk space and record database configuration choices. Do not execute a guessed DBCA command.

### Commands

**No database-creation command is supplied here because the source does not establish a successful or supported path past Activity 18.** Before drafting that command, record:

- Global database name and SID.
- CDB name and PDB name.
- Filesystem datafile location and required capacity.
- Recovery area location/size, or an explicit decision not to configure one.
- Memory allocation appropriate for the VM's 7.7 GiB RAM.
- Character set and national character set.
- Listener creation or use of an existing listener.
- Secure handling of SYS, SYSTEM, and PDB administrator passwords.
- Whether sample schemas are needed.

Re-check the current capacity:

```bash
df -h / /u01 /u02 /u03 /u04
free -h
swapon --show
```

### Recorded results

Database creation was not started or confirmed in the source. The last recorded space assessment before installation showed approximately:

| Filesystem | Size | Available at that point |
|---|---:|---:|
| `/` | 20 GiB | 5.3–5.4 GiB before patch stage was moved; later about 12 GiB after moving it |
| `/u01` | 20 GiB | about 13 GiB |
| `/u02` | 10 GiB | about 9.9 GiB |
| `/u03` | 10 GiB | about 9.9 GiB |
| `/u04` | 5 GiB | about 4.9 GiB |

These are historical point-in-time readings, not current free-space guarantees.

### Explanation

The response file was configured for software-only installation. Database creation, listener configuration, open-mode checks, PDB status, and SQL connectivity are separate activities and are not proven complete.

### Validation

After an approved DBCA run, verify the database exists, open mode, CDB/PDB state, listener status, and SQL connectivity. Record command outputs and log paths.

### Recovery or caution

Do not proceed while Activity 18 is blocked. Avoid putting passwords in the shell history or committing them to Git. Recheck filesystem capacity before allocating database files.

### Status

**Pending** — database creation and end-to-end database validation are not established.

---

## 6. Consolidated final verification checklist

Use this checklist on the target system immediately before considering the software-only installation complete.

- [x] OS version, kernel, and architecture recorded.
- [x] Oracle user/group IDs and directory paths verified.
- [x] Oracle 19c base image extracted and `runInstaller` found.
- [x] Extracted Oracle Home ownership corrected to `svanapal:oinstall`.
- [x] Initial missing packages identified and the recorded package-install commands run.
- [x] `make`, `ar`, `ld`, and `nm` found in `PATH`.
- [x] Oracle user limits configured and effective values recorded after re-login.
- [x] Persistent sysctl settings applied and effective values recorded.
- [x] THP `madvise` boot argument applied and active mode verified after reboot.
- [x] RU README inspected; RU patch ID/version and OPatch minimum identified.
- [x] Oracle Home backup created and gzip integrity test passed.
- [x] Original OPatch preserved; active OPatch reports `12.2.0.1.45`.
- [x] RU stage verified with README and metadata files.
- [x] Swap increased from 2 GiB to about 8 GiB total and `/swapfile` added to `/etc/fstab`.
- [x] Installer prerequisite check rerun; optional `compat-libcap1` warning remains.
- [x] Oracle 19c software installation completed with RU 19.26 applied, with warnings.
- [x] Both root scripts run.
- [x] SQL*Plus reports `19.26.0.0.0`.
- [x] `opatch lspatches` reports RU `37260974` and OCW `29585399`.
- [x] DBCA executable/help options inspected.
- [ ] `findmnt --verify` warnings reviewed in the context of the swap file; persistent swap confirmed after a reboot.
- [ ] Current certification/RU recommendation revalidated before reuse.
- [ ] DBCA critical Clusterware prerequisite failure resolved by a supported method.
- [ ] Standalone filesystem database created.
- [ ] Database open mode, CDB/PDB status, listener, and SQL connectivity verified.

## 7. Outstanding issues and unresolved errors

1. **DBCA blocker:** `DBT-09101`, `PRVF-5301`, and `PRVF-9905` are critical and unresolved.
2. **Optional installer prerequisite:** `compat-libcap1-1.10` remained unavailable in enabled OL9 repositories and was marked ignorable by the installer.
3. **Installer warnings:** `INS-32047` (central inventory not empty) and `INS-13014` (optional prerequisites unmet) were recorded; do not erase these from the audit trail.
4. **Swap validation:** active swap was recorded at 8 GiB, and the `/etc/fstab` entry was added. `findmnt --verify` showed two warnings; the source explains why these may be benign for swap but a reboot-persistence check is still prudent.
5. **RU/platform currency:** RU 19.26 was applied in this lab even though the source notes Oracle Linux 9 guidance recommending RU 19.32 or later. Confirm current support guidance.
6. **Archive authenticity:** no authoritative checksums/signatures were recorded for the base image, RU, or OPatch ZIP.
7. **Restore testing:** the Oracle Home backup passed `gzip -t`, but no full restore test was recorded.
8. **Database configuration:** database identity, CDB/PDB, data location, recovery area, character set, memory, listener, and credentials are not finalized.

## 8. Source coverage and reconstruction notes

### Coverage

- Uploaded source: `1.1-silent installation-binaries(1).md`.
- Source lines processed: **6,024** (first through last line).
- Source line ranges are mapped to each activity in the index and in the detailed sections.
- The source contains 143 Markdown headings and large command outputs, including a multi-thousand-line OPatch inventory and a DBCA trace.
- The large repetitive OPatch inventory dump (source lines 3748–5639) is not reproduced line-for-line. The inventory command, Home/inventory metadata, OPatch version, applied RU/OCW entries, and relevant verification result are retained. The many base component patch IDs were not considered useful to duplicate into the execution flow.
- The final source heading at line 6023 (“Check DBCA's supported database-creation options”) has no following captured output before the file ends at line 6024. This is an explicit source boundary, not evidence that the options check succeeded.

### Reconstructed/proposed steps

The following are procedural safety additions or sequencing improvements, not claims of historical execution:

- Conditional account/group creation and destination checks for a rerunnable procedure.
- Preflight checks before extracting over a populated Oracle Home.
- Guard against overwriting an existing backup or OPatch backup directory.
- Conditional swap-file creation and `/etc/fstab` append.
- Pre-install revalidation of stage path, disk space, response file, and OPatch.
- Post-install validation commands consolidated into Activity 16.
- Database-creation validation checklist, because the source did not create a database.

The source does explicitly contain the Oracle Home backup command, OPatch temporary extraction and replacement, RU extraction, `/u02` stage copy/compare, swap persistence, installer invocation, root scripts, inventory checks, and DBCA diagnostics. These were incorporated into their proper activities rather than treated as missing.

## 9. Consolidation report

### Activities and commands consolidated

- OS release, kernel, architecture, hostname, memory, swap, filesystem, and storage checks merged into Activities 01, 02, 05, and 13.
- User/group and Oracle directory checks merged into Activity 03.
- Initial and post-extraction ownership checks merged into Activity 04.
- Repeated RPM checks and package installations merged into Activity 05, retaining the initial missing state, later installation commands, and `net-tools` outcome.
- Repeated kernel parameter inspections and sysctl configuration merged into Activity 07, retaining before/after values.
- THP discovery, GRUB/BLS inspection, configuration, reboot, and verification merged into Activity 08.
- Response-file copy, parameter edits, duplicate detection, and software-only-option checks merged into Activity 09.
- RU archive listing, README inspection, OPatch minimum, and build-tool validation merged into Activity 10.
- Oracle Home backup, archive integrity check, original OPatch preservation, and pre-install inventory-pointer error merged into Activity 11.
- OPatch staging/replacement and RU extraction/move to `/u02` merged into Activity 12.
- Repeated installer prerequisite checks, log extraction, swap setup/persistence, `net-tools` install, and `compat-libcap1` investigation merged into Activity 13.
- Installer invocation, root scripts, OPatch inventory, SQL*Plus version, DBCA help, and DBCA failure consolidated into Activities 14–18.

### Repetitions removed from the execution flow

- Duplicate `rpm -q` checks are represented by one pre-change and one post-change validation rather than repeated installation instructions.
- Repeated user/group and directory checks are grouped around the change that they validate.
- Repeated sysctl and `ulimit` checks are kept where they demonstrate before/after changes; final verification is listed once in the checklist.
- Repeated RU metadata and OPatch version checks are retained only at meaningful checkpoints: before replacement, after replacement, before install, and after install.
- The massive repeated/base patch inventory is summarized, with the relevant RU and OCW entries preserved.

### Historical failures and corrections retained

- Initial missing package results and subsequent package installation commands.
- Initial kernel/network settings below the assessment values and final effective sysctl values.
- Initial THP `always` mode and post-reboot `madvise` mode.
- Initial OPatch `12.2.0.1.17`, pre-install `lsinventory` error 104 due to missing inventory pointer, and successful OPatch replacement.
- Root filesystem pressure from the 4.3 GiB RU stage, move/copy to `/u02`, and comparison before removal of the original stage.
- Installer swap shortfall, 6 GiB swap-file addition, persistence entry, and `findmnt` warnings.
- `INS-13014` and `INS-32047` warnings.
- Successful software installation and applied RU inventory.
- DBCA `DBT-09101`, `PRVF-5301`, `PRVF-9905`, and critical Clusterware tasks.

### Repetitions not fully merged

- Different installer prerequisite logs were kept as separate attempts because they show state changes after swap and `net-tools` installation.
- Pre- and post-change outputs for kernel settings, THP, swap, OPatch, and patch inventory were retained because they establish corrective changes.
- The OPatch inventory's very long list of base component patch IDs was not reproduced line-for-line; it is redundant inventory detail for the purpose of the runbook, while the decisive RU/OCW entries and exact verification command remain.

## 10. Final audit summary

| Audit item | Result |
|---|---|
| Source lines processed | 6,024 of 6,024 |
| Source fully reviewed | Yes, first through last line |
| Activity count | 19 |
| Consolidated activity groups | OS discovery; identity/directories; packages/tools; limits; sysctl; THP; response file; RU/OPatch; backup; swap/prerequisites; installation; root scripts; inventory; DBCA |
| Missing/reconstructed steps identified | Rerun-safety guards, archive checksum procedure if authoritative checksums exist, safe destination checks, current support/certification revalidation, reboot-persistence check, DBCA resolution, final database validation |
| Unresolved technical blocker | DBCA mandatory Clusterware prerequisite failure (`DBT-09101`, `PRVF-5301`, `PRVF-9905`) |
| Other unresolved items | Optional `compat-libcap1`; installer warnings; swap validation warning review; current RU/certification check; no archive checksum/restore test |
| Source ranges not processed | None |
| Historical outputs invented | No; recorded outputs are identified as recorded, and reconstructed safety commands are not presented as historical results |

---

**End state:** Oracle Database 19c software-only Home with RU 19.26 is recorded as installed. Database creation is **not complete**. Do not mark the lab fully complete until the DBCA blocker is resolved through a supported procedure and the resulting database is verified end-to-end.
