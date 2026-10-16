# The main ways to install Oracle 19c binaries

```
Method 1 - GUI installation (interactive)
Method 2 - Silent installation using a response file
Method 3 - Silent installation with command-line overrides
Method 4 - Install from a prepared Gold Image
Method 5 - RPM-based installation from oracle 26ai.

Method 1 — GUI installation (interactive)
Extract the Oracle 19c software image into the Oracle Home directory and launch runInstaller. Complete the installation wizard using the graphical interface.
- Easy to understand each installation option.
- Requires a graphical desktop or suitable X11 forwarding.
- Good for your first hands-on installation.

Method 2 — Silent installation using a response file
Supply configuration values in a .rsp file and run runInstaller -silent.
- No graphical interface required.
- Repeatable and suitable for automation.
- Excellent DBA practice for building multiple lab.

Method 3 — Silent installation with command-line overrides
Use the silent installer with a response file and specify selected settings directly in the command.
- Useful for scripts and repeatable builds.
- Easier to automate selected configuration values.
- Still uses Oracle Universal Installer; it is not a separate installation engine.

Method 4 — Install from a prepared Gold Image
Prepare a reusable Oracle Home image and use it to deploy the software to another compatible environment.
- Useful when building multiple Oracle VMs.
- Reduces repeated preparation work.
- Requires careful control of patches, configuration, and compatibility.

Method 5 - RPM-based installation from oracle 26ai.
Oracle's 26ai preinstallation RPM is a separate package that configures operating-system prerequisites. It is not the database software itself.
```
