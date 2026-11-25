# Plodder

## Overview

Plodder...

* is a general server for HTTP(S) requests of any kind.
* runs on Windows, Linux, and MacOS. It might run on a Raspberry Pi, but it is not compatible with AIX.
* uses Dyalog's Conga and, on top of that, [Rumba](https://github.com/the-carlisle-group/Rumba), a lightweight HTTP server implemented in Dyalog.
* requires Dyalog APL 18.2 Unicode or higher.
* comes with a sample application that demonstrates how to build an application on top of Plodder.

## Features

Plodder is a fully-featured HTTP(S) server. It supports all HTTP 1.1 features except one: [Server-Sent Events (SSE)](https://en.wikipedia.org/wiki/Server-sent_events) are not yet implemented.

Apart from serving HTTP requests, features include:

1. **Log Files**

   Plodder keeps a server log and an application log. Conga, HTTP, and Rumba events can be logged independently; the INI file says which of them are. See [Logging](#logging) for details.

2. **Windows Event Log**

   When running under a Dyalog Runtime instance on Windows, Plodder logs events such as "Start", "Pause", "Continue", and "Stop" to the Windows Event Log.

3. **Error Trapping**

   Plodder traps critical errors and captures as much information as possible to aid debugging, whether the issue stems from Plodder, Conga, or Rumba.

   Application errors are also logged, and Plodder returns "500: Internal Server Error" to the client. When a crash happens while a request is being handled, that request is saved next to the crash files as `<crash>.request.json`, with credentials masked and without the request body: see [Credentials in crash reports and logs](#credentials-in-crash-reports-and-logs).

   When the same error keeps occurring, not every occurrence produces crash files: see [Crash storms](#crash-storms).

4. **Running as a Windows Service**

   On Windows, Plodder can run as a Windows Service. Use `Plodder.Admin.PrintInstallAsServiceCommand` to print the command that must be executed with admin rights to install Plodder as a service. For example:

   ```
         Plodder.Admin.PrintInstallAsServiceCommand 0
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1
         Plodder.Admin.PrintInstallAsServiceCommand 4501
   /.../dyalogrt.exe /.../Plodder.dws APL_ServiceInstall=Plodder DYALOG_NOPOPUPS=1 -ride=4501
         Plodder.Admin.PrintUninstallServiceCommand ''
   sc delete Plodder
   ```

5. **Early Ride**

   Adding `-ride=n` to the command line (where `n` is a port number like 4502) enables Ride access on that port before the INI file is read. The `[RIDE]` section of the INI file can enable it later.

## Sample Application

Typically, Plodder resides in `#.Plodder`, while an application such as `Foo` would reside in `#.Foo`. The INI file connects the two.

In the sample application, however, the "application" is part of Plodder itself and resides in `#.Plodder.SampleApp`.

## Configuration: The INI File

The INI file specifies a namespace with at least one function to handle the `OnRequest` event, and optionally functions for the `OnStart`, `OnHeader`, and `OnCongaTimeout` events. These functions provide the interface between Plodder and the application that responds to HTTP(S) requests; see [The handlers](#the-handlers).

The `Context` entry in the INI file specifies the fully qualified name of the namespace containing these functions.

The INI file, named `server.ini`, is extensively documented.

### Housekeeping

The function named by `[APP]OnCongaTimeout` is called at regular intervals, so that an application can carry out whatever work it needs to repeat. `[CONFIG]HouseKeepingInterval` says how often, in seconds; it defaults to 5.

Note that the function is called on the server loop thread, and no request is processed while it runs. Keep it short, and start any long-running work in a thread of its own.

### Idle connections

Idle connections are closed on the same schedule. Every time the housekeeping interval expires, and before the housekeeping function is called, the server closes any connection that has neither sent nor received anything for `[CONFIG]IdleConnectionTimeout` seconds. It defaults to 30. A connection whose request has not been answered yet is never closed, so a request that is still being processed cannot be interrupted.

The two entries therefore depend on each other, in two ways.

First, connections are only examined when the housekeeping interval expires, so that interval is also the accuracy of the idle timeout. With the default settings — an interval of 5 seconds and a timeout of 30 — a connection is closed between 30 and 35 seconds after it last sent or received anything.

Second, `HouseKeepingInterval` may be set higher than `IdleConnectionTimeout`, but then the timeout has no effect: no connection can be closed before the interval expires, so the interval becomes the timeout that actually applies. An interval of 60 with a timeout of 30 keeps idle connections open for 60 to 90 seconds, not 30. Plodder accepts such a configuration, because there are good reasons to run expensive housekeeping rarely, but the timeout is then not honoured.

In general, set `HouseKeepingInterval` well below `IdleConnectionTimeout`. If your housekeeping is expensive, keep the interval short anyway and let the housekeeping function itself decide how often to do the expensive part.

## The handlers

The `[APP]` section of the INI file names the functions through which Rumba hands work to your application. `Context` is the namespace they live in, and the other four entries name functions in that namespace:

| Entry | Called | Argument | Result |
|:--|:--|:--|:--|
| `OnStart` | Once, when the server starts, before it accepts connections | The server instance | Ignored |
| `OnHeader` | For every request, as soon as its header has arrived and before its body is read | The request | 0, or a response |
| `OnRequest` | For every complete request. This one is mandatory | The request | A response |
| `OnCongaTimeout` | Every `[CONFIG]HouseKeepingInterval` seconds; see [Housekeeping](#housekeeping) | The server instance | Ignored |

At start-up Plodder checks that every name given exists in `Context`, can be called with a right argument only, and returns a result: a monadic or ambivalent tradfn, or a dfn. Otherwise the server does not start. Plodder also establishes `G` in `Context`, so the handlers can reach the INI file and the other globals through `G`.

Rumba looks a name up each time it calls the handler, so when you change a handler in a running server, the change takes effect with the next call.

### OnStart

`OnStart` runs in the thread that executes `Run`, before the server accepts connections, so it is the place for anything the application must set up first. HandleError's parameters exist at that point; see [Your own crash handling](#your-own-crash-handling).

### OnHeader

`OnHeader` lets the application look at a request before its body arrives, for example to reject a request that is too large or lacks authorization without receiving the body first. It returns either 0, and the request carries on, or a response, which Rumba sends at once. Rumba then closes the connection, and the request does not reach `OnRequest`.

When the client sent `Expect: 100-continue` and `OnHeader` returns 0, Rumba answers with `100 Continue`, so that the client sends the body. While the server is paused, Rumba answers every request with 503 and does not call `OnHeader`.

### OnRequest

`OnRequest` gets the complete request and returns the response. Create the response with Rumba's `NewResponse`, as the sample application's `OnRequest` does, and set at least `StatusCode` and `Content`. Among other things the request offers `Method`, `Path`, `Query`, `QueryParameters`, `Headers` and `Content`. Route on `Path`, not on `RequestTarget`, because `RequestTarget` includes the query string.

A request with one of the headers `If-Match`, `If-None-Match`, `If-Modified-Since` and `If-Unmodified-Since` is handled in two steps. Rumba first calls `OnRequest` with `Method` set to `GET`, to learn about the current state of the resource. If the condition is not met, Rumba answers with 304 or 412 itself; otherwise it calls `OnRequest` a second time, with the original method. `OnRequest` must therefore be able to answer such a `GET` without side effects.

### Threads

Every connection has a thread of its own, and `OnHeader` and `OnRequest` run in it. Requests from different clients are therefore processed at the same time, while the requests that arrive on one connection are processed one after the other. Anything the handlers share, in `G` or elsewhere, must allow for that.

`OnCongaTimeout` runs on the server loop's thread instead. All data for all connections passes through that thread, so no new request is processed while the function runs.

### Errors in a handler

With error trapping on (see `[TRAP]Policy` in [Starting the server](#starting-the-server)), HandleError records every error in a handler. What happens next depends on the handler:

| Handler | After the error |
|:--|:--|
| `OnHeader`, `OnRequest` | The client gets a 500, and the server carries on |
| `OnStart`, `OnCongaTimeout` | The server stops with exit code 125, as after any other crash |

With error trapping off, execution stops where the error happened. For `OnHeader` and `OnRequest` that is the connection's thread, and the client waits for an answer while the rest of the server carries on.

## Starting the server

The server is started by the operator `Run`:

```
{iniFile}(fns Run)flags
```

* `iniFile` is optional and names the INI file. It defaults to `server.ini`. A relative name is resolved against the current directory, which Plodder sets to the folder the workspace was loaded from.
* `fns` is a function that gets `G` as its right argument (see [Access to Server Configuration](#access-to-server-configuration)) and must return 1. It is called after the INI file has been processed and the log files have been opened, and before the server starts. Use it for whatever your application has to establish first, or pass `{1}` if there is nothing to do.
* `flags` holds up to two flags: the test flag and the trap flag. A missing flag, or ¯1, means that the INI file decides: `[CONFIG]TestFlag` (default 0) for the first, `[TRAP]Policy` and `[TRAP]Rumba` for the second. A 0 or 1 overrides the INI file; the trap flag then sets both `Policy` and `Rumba`. Production code normally passes `⍬`.

`[TRAP]Policy` takes three values: 1 establishes Plodder's general error trap in any case, ¯1 only in a Runtime, and 0 never. With 0, an error that reaches the general trap is not processed by HandleError: no crash files are written, and `Run` does not end with `⎕OFF 125`, so whatever restarts the server on that exit code is not triggered. Use 0 for debugging only.

`Run` returns only when the server is stopped by its quit event. In a runtime it then ends with `⎕OFF`.

After a crash that Plodder's general error trap catches, HandleError saves the crash information, and `Run` then executes `⎕OFF 125`, so the server stops with exit code 125. Configure whatever starts the server to restart it, for example the recovery options of a Windows Service. With a Ride attached, Plodder does not end the session but stops, so that you can investigate.

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
| `[LOGGING]LogHTTP` | Log HTTP requests to `log.txt`. 0: no. 1: one line per request, with the client's address, the time, the request line, the status code and the size of the response. 2: every request and every response in full, headers and bodies included (see [Credentials in crash reports and logs](#credentials-in-crash-reports-and-logs)) |
| `[LOGGING]LogConga` | Log Conga events to `log.txt` |
| `[LOGGING]LogRumba` | Log Rumba events to `log.txt` |

`log.txt` is created only when at least one of `LogHTTP`, `LogConga` and `LogRumba` is not 0.

`[LOGGING]Log` has no effect on `log.txt`. Plodder passes it on to Rumba, but Rumba does not use it.

Note that with `Log = 0` there is no application log file at all. `AppLog` and `AppLogError` then have nothing to write to, and fail silently. That includes the details of every crash: see [Crash storms](#crash-storms) for why `Log` should be 1.

### Renamed in 1.12.0

Two of these functions were renamed, because the old names did not say which file they wrote to. This is a breaking change for any application that called them:

| Up to 1.11.0 | From 1.12.0 |
|:--|:--|
| `LogError` | `AppLogError` |
| `Log` | `ServerLog` |

## Crash storms

When the same error keeps occurring, every occurrence would normally produce a full set of crash files: the HTML report, the component file with the crash information and the saved request. That can fill a disk quickly, for example when a server crashes during start-up and is restarted every time, or when a client keeps sending a request that makes the server crash.

To prevent this, Plodder switches on HandleError's crash storm guard by default. Two errors count as the same error when they have the same error number and occurred on the same line of the same function. Such an error gets the full treatment only at its 1st, 2nd, 4th, 8th, 16th... occurrence. At all other occurrences only the entry in `app-log.txt` and, if configured, the Windows event are written. Both carry a note that says how often the error has occurred and that no crash files were written.

That is why the shipped INI file sets `[LOGGING]Log = 1`, and why it should stay 1 in production. With `Log = 0` there is no `app-log.txt`, and unless the Windows Event Log is written to, an occurrence that produces no crash files leaves no trace at all: you can tell neither how often an error occurred nor that it is still occurring.

The guard is configured in the `[TRAP]` section of the INI file:

| Entry | Default | Effect |
|:--|:--|:--|
| `StormGuard` | 1 | 1 switches the guard on, 0 switches it off |
| `StormCeiling` | 10 | The maximum number of full crash reports within one hour, for all errors together |
| `StormResetAfter` | 24 | When an error has not occurred for this many hours, its count starts again at 1. Fractions are allowed |

The guard keeps its records in the file `StormGuard.txt` in the error folder, `[TRAP]Folder`.

### Restriction: one process per error folder

Only one process may write to an error folder. The guard reads `StormGuard.txt`, updates it and writes it back, and HandleError cannot synchronise this between processes. When two processes crash at the same moment, one of them may overwrite the update of the other, so an error may be counted too few times, or all counts may start again at 1. If you run several Plodder servers on one machine, give each of them its own `[TRAP]Folder`. The default, `<TEMP>/Plodder-Errors`, is the same for all of them.

## Credentials in crash reports and logs

Two places can end up holding credentials: the request saved with a crash (`<crash>.request.json` and the summary in the HTML report), and the start-up report in `log.txt`, which lists the command line and every INI entry. Plodder masks what it can recognize and leaves out what it cannot check. A third place, `log.txt` with `[LOGGING]LogHTTP = 2`, is not masked at all; see the end of this section.

### Crash reports

* The values of these headers are replaced by `***`: `Authorization`, `Proxy-Authorization`, `Cookie`, `Set-Cookie`, `API-Key`, `X-API-Key` and `X-Auth-Token`. Add your own with `[TRAP]MaskHeaders`, a comma-separated list.
* The values of these query parameters are replaced by `***`: `access_token`, `api_key`, `apikey`, `auth`, `key`, `password`, `pwd`, `secret` and `token`. Add your own with `[TRAP]MaskQueryParms`, a comma-separated list.
* Request bodies are not saved, because a body may carry anything: a password in a form, a token in JSON, personal data. The report still says how long each body was. Set `[TRAP]SaveRequestBody = 1` when you need the bodies to reproduce a crash; they are then capped at 64 KB.

Names are compared without regard to case. Masking works from these lists, so a credential under a name that is not on them is saved as it is. Plodder does not look into other headers either: a token inside a `Referer` URL, for example, is not masked.

### The start-up report

An INI entry whose key contains `PASSWORD`, `PASSWD`, `SECRET`, `TOKEN`, `APIKEY`, `API_KEY` or `CREDENTIAL` is shown as `***`, regardless of case. Add your own parts of key names with `[LOGGING]MaskIniKeys`, a comma-separated list. `KEY` on its own is not on the list, because it would hide entries such as `PrivateKeyFile`, which hold paths. The command line is reported as it is.

### `log.txt` with `LogHTTP = 2`

With `[LOGGING]LogHTTP = 2`, Rumba writes every request and every response to `log.txt` in full, headers and bodies included. Nothing is masked, so authorization headers, cookies and passwords sent in a form all end up in the file. Use it for debugging only, and delete the file afterwards.

## Your own crash handling

HandleError, which processes every crash, can call a function of your own: the one named by its parameter `customFns`. Plodder uses this parameter itself. Its function `SaveCrashRequest` saves the request the crashed thread was working on as `<crash>.request.json` and adds a summary of it to the HTML report.

`customFns` holds a single name. If you assign your own function to it, Plodder's function is no longer called, unless your function calls it.

HandleError's parameters are in `#.ErrorParms`, which is established before `OnStart` is called. In `OnStart`, save the name Plodder put there and then put the name of your function in its place:

```apl
 r←OnStart server
 ...
 #.MyApp.PloddersCustomFns←#.ErrorParms.customFns
 #.ErrorParms.customFns←'#.MyApp.OnCrash'
```

Your function gets HandleError's parameter namespace as its right argument. Its result is ignored. Call Plodder's function first and pass the namespace on:

```apl
 {r}←OnCrash parms
 r←⍬
 (⍎#.MyApp.PloddersCustomFns)parms    ⍝ Plodder's function first: it saves the request
 ⍝ Your own code, for example sending an email
```

Some details:

* Call Plodder's function first. If your own code fails, HandleError stops your function and notes "customFns failed" in the HTML report. Nothing after the failing line is executed, and that would include the call to Plodder's function.
* Save the name, not the function, and resolve it with `⍎` when you call it. A function captured at start-up may be outdated by the time a crash happens, for example after Link has fixed a changed version of it.
* Use a fully qualified name for your function. Otherwise HandleError also needs `customFnsParent`.
* With the crash storm guard switched on, the function is called only for the crashes that get the full treatment. See [Crash storms](#crash-storms).

## Conga

### The Conga namespace

By default, Plodder tries to copy Conga from the installation of the version of Dyalog Plodder is running on. 

If that does not work Plodder tries to copy it from a workspace `conga.dws` in the current directory.

### The Conga DLLs

By default Plodder uses the ones that are available from the currently running version of APL.

### I don't want that

If you want neither, and would rather specify precisely which version to use, add an entry `[CONFIG]CongaFolder` to the INI file. This was introduced in version 1.9.0.

The folder it points to must contain all Conga DLLs required for all platforms you will run on, and a workspace `conga.dws` that contains the namespace `Conga` that Plodder should use.

## Access to Server Configuration

All relevant data for running the server is collected in an unnamed namespace that Plodder refers to as `G` (for "Globals"). It also holds the instance of the server's INI file as `G.INI`.

At start-up Plodder assigns a reference to this namespace to the name `G` in the `Context` namespace defined in the INI file, so the event handlers can use `G` directly. Anything called `G` in that namespace is overwritten.

Note that an application should not modify any values in `G` or the INI file.

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

4. Review the INI file settings to ensure they meet your needs. At a minimum, you must modify the `[APP]` section.

5. Set `⎕LX` so that it starts the server, for example `⎕LX←'({1}#.Plodder.Run)⍬'`, and save the workspace. See [Starting the server](#starting-the-server).









