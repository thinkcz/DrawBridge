# DrawBridge
Research on DrawBridge Library OS, which is base building block for MSSQL on Linux

## Introduction
Microsoft introduced library OS research called DrawBridge long time ago. Now it seems there is an implementation. MSSQL server on Linux is, in fact LibraryOS of windows 8  and on top of it runs unmodified MSSQL engine. Windows as an application.

## Hacking
There is a first tool to unpack .sfp files which are library os archives, can be found after installation of mssql on Ubuntu in **/opt/mssql/lib**
Tool is here: https://github.com/nta/sfpack, which can be used to unpack .sfp files 

## Files inside .sfp packages
It seems there are unmodified binaries of **Windows 8**. However, there are some special ones:
### .dbpatch files
are binary patch files to their respective siblings with the same name. This makes perfect sense as you need to remove any syscalls instructions from binaries like **ntdll.dll wi32k.sys** etc. So instead of store them patched, original files are presented and they are patched on the go.
### ntoskrnl.dll.bin
this strange file makes perfect sense too, it is with help of **ntoskrnl.dll.bin.ini** user-mode  kernel version in a format ready to run. My guess is that is for reason  that you don't need PE loader to load such prepared image to memory so it is more OS independent way of loading it. Just map it into memory and from .ini file (which contains sections with their addresses, sizes, and protections) set respective protection on that memory. You load a kernel that way without the need of any PE loader.

### Extracted interesting parts
When you look on sqlservr /opt/mssql/bin you can find interestig things inside, binary is probably pre-configured **palrun** tool for running arbitrary app inside DarwBridge package, as a proof there are part of command line help interface:

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

