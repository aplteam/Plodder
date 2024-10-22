# Plodder

## Overview

Plodder...

* is a general server for HTTP(S) requests of any kind.
* runs on Windows, Linux, or MacOS. Currently, there are no plans for Raspberry Pi support, and it is not compatible with AIX.
* uses Dyalog's Conga and, on top of that, [Rumba](https://github.com/the-carlisle-group/Rumba), a lightweight HTTP server implemented in Dyalog.
* requires Dyalog APL 18.0 Unicode or higher.
* comes with a sample application that demonstrates how to build an application on top of Plodder.

## Features

Plodder is a fully-featured HTTP(S) server. It supports all HTTP 1.1 features except one: [Server-Sent Events (SSE)](https://en.wikipedia.org/wiki/Server-sent_events) are not yet implemented.

Key features include:

1. **Log Files**  
   Conga, HTTP, and Rumba events can be logged independently. By default, only HTTP events are logged.

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

## Access to Server Configuration

All relevant data for running the server is collected in a namespace `G` (for "Globals),", which holds a reference to an unnamed namespace. This namespace also contains a reference to the server's INI file.

`G` is accessible from event handlers through the `Context` namespace defined in the INI file. However, applications should not modify any values in `G` or the INI file.

## Test Cases

Plodder includes a test suite. Use `]Cider.RunTests` to print the test command to the session.

To run the tests, you need a copy of the project; the release version will not suffice.

## Installation

1. Download a release.

2. `)xload` the `Plodder` workspace.

   You can now start the server with 

   ```
   ({1}Plodder.Run)0 1
   ``` 

   Access it at `http://localhost` (NOT `https://`!). The demo application built into Plodder will respond.

   (This works because Plodder looks for a `server.ini` file in the same folder as the workspace.)

In order to bring in your own application continue with the following steps:

3. Bring in the namespace or class containing your handlers for the four Plodder events.

4. Review the INI file settings to ensure they meet your needs.  
At a minimum, you must modify the `[APP]` section.

5. Execute `⎕LX`.


