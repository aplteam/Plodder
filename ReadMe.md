# Plodder

## Overview 

Plodder...

* is a general server for HTTP(S) requests of any kind. 

* may run under Windows, Linux or MacOS; there are currently no plans to make it work on a PI, and it won't work under AIX.

* uses Dyalog's Conga and on top of that [Rumba](https://github.com/the-carlisle-group/Rumba), a light http server implemented in Dyalog.

* requires Dyalog APL 18.0 Unicode or better. 

* comes with a sample application that shows the details for how to implement an application on top of Plodder.


## Features

Plodder is a fully-fledged HTTP(S) server. It supports all HTTP 1.1 features but one: [SSE](https://en.wikipedia.org/wiki/Server-sent_events) (Server Side Events) are currently not implemented.

In particular Plodder offers these features:

1. Log files. 

   Conga events, HTTP events and Rumba events can all be toggled independently of each other. By default only HTTP events are logged.

2. Windows Event Log

   Under Windows Plodder writes to the Windows Event Log when it runs under a Dyalog Runtime instance. The events "Start", "Pause", "Continue" and "Stop" are logged.  

3. Error trapping

   Plodder traps any major errors and saves as much information as possible in order to allow getting to the bottom of the problem in case either Plodder itself or Conga or Rumba crashes.

   Application errors are also trapped: they are logged, and Plodder returns "500: Internal Server Error" to the client. 
 
4. Running as a Windows Service

   When running under Windows Plodder may run as a Windows Service. The function `Plodder.Admin.PrintInstallAsServiceCommand` can be used to print the command to the session that needs to be executed with Admin rights for installing Plodder as a Windows Service. These are examples of what would be printed:

   ```
         Plodder.Admin.PrintInstallAsServiceCommand 0
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1
         Plodder.Admin.PrintInstallAsServiceCommand 0
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1 -ride=4501
         Admin.PrintUninstallServiceCommand
   sc delete Plodder
   ```

5. By adding `-ride=n` (`n` being a positiv number like 4502) one can trigger Plodder to allow the user a RIDE on port `n`.



## The sample application

Usually Plodder would reside in `#.Plodder` while an application `Foo` would typically reside in `#.Foo`. The Ini file has entries that connects the two.

The sample application is slightly different insofar as the "application" is itself part of Plodder; therefore it resides in `#.Plodder.SampleApp`.


## Configuration: the INI file

The INI file allows to specify a namespace that has at least a function that handles the `OnRequest` event, and it may have functions for the `OnStart`, `OnHeader` and `OnCongaTimeout` events. These functions are the interface between Plodder and the application that reacts in one way or another on HTTP(S) requests.

In which namespace these function can be found is defined by the `Context` entry in the INI file: that must be the fully qualified name of a namespace in the workspace.

The INI file is called `server.ini`. It is sorrowly documented.


## Access to the server configuration etc.

All data that is relevant in running the server is collected in a namespace `G` for "Globals", though it is not really a global but rather a local variable holding a reference to an unnamed namespace.

This namespace also holds a reference to in instance of the INI file of the server.

`G` is accessible from the handlers because a reference is injected into the namespace defined by the INI definition `[APP]Context`. However, the application is not supposed to change any of the values in `G` or in the INI file instance.


## Test cases

Plodder comes with a test suite; enter `]Cider.RunTests` in order to get the command printed to the session.

However, in order to run the test cases you must have a copy of the project: a release won't do.


## Installation

1. Download a release.

2. )xload the `Plodder` workspace

   At this point you could already start the server with `({1}Plodder.Run)0 1` and put `http://localhost` into the address bar of a browser and it would work thanks to the demo application build into Plodder.

   (This would work because Plodder would look for a file `server.ini` in the same folder the workspace was loaded from)

3. Bring in the namespace (or class) that holds the handlers for the four Plodder events as well as everything they need.

4. Check the INI files in order to make sure that the settings suit your needs.

   At the very least you need to amend the settings in the `[APP]` section.

5. Execute `⎕LX`.
