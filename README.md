# DrawBridge
Research on DrawBridge Library OS, which is base building block for MSSQL on Linux

## Introduction
Microsoft introduced library OS research called DrawBridge long time ago. Now it seems there is an implementation. MSSQL server on Linux is, in fact LibraryOS of windows 8  and on top of it runs unmodified MSSQL engine. Windows as an application.

## Papers

DrawBridge research page at microsoft.com:
https://www.microsoft.com/en-us/research/project/drawbridge/
Using DrawBridge and SGX to protect applications both ways:
https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-baumann.pdf


## Hacking
There is a first tool to unpack .sfp files which are library os archives, can be found after installation of mssql on Ubuntu in **/opt/mssql/lib**
Tool is here: https://github.com/nta/sfpack, which can be used to unpack .sfp files 

## Files inside .sfp packages
It seems there are unmodified binaries of **Windows 8**. However, there are some special ones:
### .dbpatch files
are binary patch files to their respective siblings with the same name. This makes perfect sense as you need to remove any syscalls instructions from binaries like **ntdll.dll**, **win32k.sys** etc. So instead of store them patched, original files are presented and they are patched on the go.
### ntoskrnl.dll.bin
this strange file makes perfect sense too, it is with help of **ntoskrnl.dll.bin.ini** user-mode  kernel version in a format ready to run. My guess is that is for reason  that you don't need PE loader to load such prepared image to memory so it is more OS independent way of loading it. Just map it into memory and from .ini file (which contains sections with their addresses, sizes, and protections) set respective protection on that memory. You load a kernel that way without the need of any PE loader.

### Extracted interesting parts
When you look on sqlservr in /opt/mssql/bin you can find interesting things inside, binary is probably pre-configured **palrun** tool for running arbitrary app inside DarwBridge package, as a proof there is a part of command line help interface:

```
Usage: palrun [OPTION]... -- [PROGRAM ARGUMENTS]
Options that take an additional argument.
  -a, --application         Guest application executable.
  -p, --package             Package directory path.  Multiple packages supported.
  -w, --working-directory   Application working directory path.
  -d, --debug-level         Debug levels: silent, libos, error, warning, debug, trace_low, trace and noisy
  -l, --log-location        Log location path.
  -t, --debug-trace-channel Debug trace channel filter.
  -m, --memory-size         Available memory size in MiB.
  
Options that don't take additional arguments.
  -v, --version             Display palrun version information.
  -f, --log-file-load       Log to stdout every file load operation.
  -u, --log-unimplemented   Log unimplemented APIs, needs a debug level of libos or more.
  -h, --help                Display help.
      --allow-attach        Allow attach non-root debugger. 
```

## Environmental Variables

* `PAL_LOCALE_INFO`
* `PAL_SQLDK_XPLAT`
* `PAL_MEMORY_SIZE`
* `PAL_PROGRAM_INFO`: demonstrated here, https://dba.stackexchange.com/a/194582/2639


## Learned facts so far
* it is based on DrawBridge research as Linux on Windows is
* in this case it is all user-mode implementation, ntdll.dll and few others are patched on the go to remove syscalls
* it is all done to make more abstract (and with less "calls") interface called PAL (Platform Abstraction Layer?)
* to run DrawBridged system you really don't need much to implement which is OS dependant.

### Bleed through

DrawBridge's Library OS may not abstract away the host opperating system. For instance, SQL Server 2017 ships with `./sqlservr.exe.lnx.hiv` which has numerious file locations on the host system,

```
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/140/VerSpecificRootDir
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/MSSQL/CPE/ErrorDumpDir
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/MSSQL/MSSQLServer/Parameters/SQLArg0
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/MSSQL/MSSQLServer/Parameters/SQLArg1
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/MSSQL/MSSQLServer/Parameters/SQLArg2
Registry/Machine/SOFTWARE/Microsoft/Microsoft SQL Server/MSSQL/MSSQLServer/BackupDirectory
Registry/User/.Default/Software/Microsoft/Windows/CurrentVersion/Explorer/Shell Folders/AppData
```

Further, the supplied `/opt/mssql/lib/system/Content/Windows/windows.hiv` has a whole structure under `Registry/Machine/SYSTEM/LibraryOS`.

### Subjects of more research

There are two binaries distributed that with SQLServer that are suspicious

* `PalProcessLauncher.exe`
* `launchpad.exe`

Launchpad.exe seems to be how utilities get launched inside the container,


```
> .\launchpad.exe
Invalid arguments. The supported arguments are:
        -launcher: Define the launcher dll's full path.
        -logPath: Define the launchpad's base log path.
        -satelliteDllPath: Define the sql satellite dll path for the satellites.
        -workingDir: Define the launchpad and satellite process base working directory.
        -cleanupLog: Whether to cleanup the log directory after every execution [0|1].
        -cleanupWorkingDir: Whether to cleanup the working directory after every execution [0|1].
        -pipeName: Define the launchpad's name pipe's name.
        -timeout: Define the default timeout in ms.
        -SqlInstanceName: Define the SqlInstanceName as in MSSQLSERVER or blank for default or an instance name.
        -logFilesCount - Tells the max number of launchpad error log file to keep across launchpad restarts.
Example:
Launchpad.exe  -launcher commonlauncher.dll -logPath C:\sqlServerBinaries\log -satelliteDllPath C:\sqlServerBinaries\bin
n\sqlsatellite.dll      -workingDir C:\sqlServerBinaries\data\ExtensibilityData -cleanupLog 0 -cleanupWorkingDir 0 -pipe
Name sqlsatellitelaunch -timeout 600000 -SqlInstanceName MSSQLSERVER    PS Microsoft.PowerShell.Core\FileSystem::\\VBOXS
VR\mssql\lib\sqlservr\Content\binn>
```
