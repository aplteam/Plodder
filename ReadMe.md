# Plodder

## Overview

Plodder...

* is a general server for HTTP(S) requests of any kind.
* runs on Windows, Linux, and MacOS. Currently, there are no plans for Raspberry Pi support, and it is not compatible with AIX.
* uses Dyalog's Conga and, on top of that, [Rumba](https://github.com/the-carlisle-group/Rumba), a lightweight HTTP server implemented in Dyalog.
* requires Dyalog APL 18.2 Unicode or higher.
* comes with a sample application that demonstrates how to build an application on top of Plodder.

## Features

Plodder is a fully-featured HTTP(S) server. It supports all HTTP 1.1 features except one: [Server-Sent Events (SSE)](https://en.wikipedia.org/wiki/Server-sent_events) are not yet implemented.

Apart from serving HTTP requests, features include:

1. **Log Files**  
   Plodder keeps a server log and an application log. Conga, HTTP, and Rumba events can be logged
   independently. By default, only HTTP events are logged. See [Logging](#logging) for details.

2. **Windows Event Log**  
   When running under a Dyalog Runtime instance on Windows, Plodder logs events such as "Start", "Pause", "Continue", and "Stop" to the Windows Event Log.

3. **Error Trapping**  
   Plodder traps critical errors and captures as much information as possible to aid debugging, whether the issue stems from Plodder, Conga, or Rumba.  
   Application errors are also logged, and Plodder returns "500: Internal Server Error" to the client.

4. **Running as a Windows Service**  
   On Windows, Plodder can run as a Windows Service. Use `Plodder.Admin.PrintInstallAsServiceCommand` to print the command that must be executed with admin rights to install Plodder as a service. For example:

   ```
         Plodder.Admin.PrintInstallAsServiceCommand 0
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1
         Plodder.Admin.PrintInstallAsServiceCommand 0
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1 -ride=4501
         Admin.PrintUninstallServiceCommand
   sc delete Plodder
   ```

5. Adding `-ride=n` (where `n` is a positive number like 4502) enables RIDE access on the specified port.

## Sample Application

Typically, Plodder resides in `#.Plodder`, while an application such as `Foo` would reside in `#.Foo`. The INI file connects the two.

In the sample application, however, the "application" is part of Plodder itself and resides in `#.Plodder.SampleApp`.

## Configuration: The INI File

The INI file specifies a namespace with at least one function to handle the `OnRequest` event, and optionally functions for the `OnStart`, `OnHeader`, and `OnCongaTimeout` events. These functions provide the interface between Plodder and the application that responds to HTTP(S) requests.

The `Context` entry in the INI file specifies the fully qualified name of the namespace containing these functions.

The INI file, named `server.ini`, is extensively documented.

## Logging

Plodder maintains two log files, both of them in the folder named by the `[LOGGING]Folder` entry of the INI file:

| File | Written by | Holds |
|:--|:--|:--|
| `log.txt` | Rumba, and Plodder's `ServerLog` | HTTP requests, Conga and Rumba events, and the start-up report: command line arguments, INI settings, versions, `⎕TRAP` and so on |
| `app-log.txt` | Plodder's `AppLog` and `AppLogError` | Whatever the application chooses to log, plus the details of every error processed by `HandleError` |

### What an application should use

An application should log with `AppLog` and `AppLogError`. Both write to `app-log.txt`, and neither needs anything but the message:

```apl
      AppLog 'User "Fred" has logged in'
      AppLogError 'Cannot reach the database'
```

`AppLog` takes either a simple character vector or a vector of character vectors, and stamps every line with date and time.

`AppLogError` prefixes the message with `*** Error` and then calls `AppLog`. It is also registered as the log function of `HandleError`, so the details of any crash end up in the same file, in the same format.

`ServerLog` is Plodder's own function for writing to `log.txt`. Unlike the other two it needs the server instance as its left argument:

```apl
      server ServerLog 'Something noteworthy'
```

That is because it is called from `OnStart`, and at that moment the server reference is not yet available from `G`. An application rarely has a reason to call it.

Events at the level of the operating system are a separate matter; see the "Windows Event Log" feature above.

### The relevant INI entries

| Entry | Effect |
|:--|:--|
| `[LOGGING]Folder` | The folder both files are written to |
| `[LOGGING]Log` | Switches the application log (`app-log.txt`) on and off |
| `[LOGGING]LogHTTP` | Log HTTP requests to `log.txt` |
| `[LOGGING]LogConga` | Log Conga events to `log.txt` |
| `[LOGGING]LogRumba` | Log Rumba events to `log.txt` |

`log.txt` is created only when at least one of `LogHTTP`, `LogConga` and `LogRumba` is 1.

Note that with `Log = 0` there is no application log file at all. `AppLog` and `AppLogError` then have nothing to write to, and fail silently.

### Renamed in 1.12.0

Two of these functions were renamed, because the old names did not say which file they wrote to. This is a breaking change for any application that called them:

| Up to 1.11.0 | From 1.12.0 |
|:--|:--|
| `LogError` | `AppLogError` |
| `Log` | `ServerLog` |

## Conga

### The Conga namespace

By default, Plodder tries to copy Conga from the installation of the version of Dyalog Plodder is running on. 

If that does not work Plodder tries to copy it from a workspace `conga.dws` in the current directory.

### The Conga DLLs

By default Plodder uses the ones that are available from the currently running version of APL.

### I don't want that

If you don't want neither of this, and instead specify precisely which version to use, then you
must create an entry `[CONFIG]CongaFolder` in the INI file. This was introduced in version 1.9.0

The folder it points to must contain all Conga DLLs required for all platform you will run on, and a workspace `conga.dws` that must contain the class `Conga` that Plodder should use.

## Access to Server Configuration

All relevant data for running the server is collected in a namespace `G` (for "Globals),", which holds a reference to an unnamed namespace. This namespace also contains a reference to the server's INI file.

`G` is accessible from event handlers through the `Context` namespace defined in the INI file. 

Note that an applications should not modify any values in `G` or the INI file.

## Test Cases

Plodder includes a test suite. Use `]cider.HowToRunTests` to find out how.

To run the tests, you need a copy of the project; the release version will not suffice.

## Installation

1. Download a release.

2. `)xload` the `Plodder` workspace.

   You can now start the server with 

   ```
   ({1}Plodder.Run)0 1
   ``` 

   Access it at `http://localhost` (NOT `https://`!). The demo application built into Plodder will respond.

   (This works because Plodder looks for a `server.ini` file in the same folder the workspace was loaded from.)

In order to bring in your own application continue with the following steps:

3. Bring in the namespace or class containing your handlers for the up to four Plodder events.

4. Review the INI file settings to ensure they meet your needs.  
At a minimum, you must modify the `[APP]` section.

5. Execute `⎕LX`.








