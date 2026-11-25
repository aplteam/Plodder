# Review of Plodder, 2026-11-24

This review covers the working tree on `dev` as of today, including the uncommitted 2.0.0 changes: everything in `APLSource/Plodder` (with `SampleApp` and `Admin`), `ReadMe.md`, `server.ini.remove-me` and `Tests/test.ini`. It does not cover the test cases themselves, Rumba, or the packages.

Line numbers follow the APL convention: the header is line 0.

Findings marked "checked" were confirmed in a throwaway Dyalog 20.0 interpreter. The others come from reading the code; for those I give the reasoning so that you can judge it.

## Summary

| # | Area | Finding | Weight |
|:--|:--|:--|:--|
| 1 | `CheckForRide`, ReadMe | `-ride=n` on the command line has no effect: the code reads `-port` | High |
| 2 | `Run`, INI | `Run` fails unless a test flag is passed or the INI file has `[CONFIG]TestFlag` | High |
| 3 | Logging defaults, crash storms | With the shipped INI file, crashes skipped by the storm guard leave no trace | High |
| 4 | `OpenAppLog` | `app-log.txt` gets the wrong name when `[LOGGING]Folder` has no trailing slash, as in the shipped INI | Medium |
| 5 | `CheckServerParms` | The check for invalid handlers never fires | Medium |
| 6 | `CheckServerParms` | An empty `OnRequest` is accepted | Medium |
| 7 | `Run` | An invalid `Context` fails before it is checked, with an unhelpful message | Medium |
| 8 | `ServerLog` | The fallback path itself fails with a DOMAIN ERROR | Medium |
| 9 | `SaveCrashRequest`, `CollectStartParms` | Credentials can still end up in crash files and in `log.txt` | Medium |
| 10 | `PolishInifilePath` | The fallback that looks next to the executable can never run, and would build a wrong path | Low |
| 11 | `GetEnvVars` | Returns values in lower case | Low |
| 12 | Windows Event Log default | Code, INI comment and ReadMe disagree | Medium |
| 13 | Sample application | Uses an undefined `H` | Low |
| 14 | Dead code | `Shutdown`, `GetAppFolder` and `r` are unused, and each is broken | Low |
| 15+ | Documentation | Missing and unclear parts, see below | |

## Bugs

### 1. `-ride=n` does nothing (checked by reading both ends)

The ReadMe (feature 5) and `Admin.PrintInstallAsServiceCommand` both use `-ride=n`. `CheckForRide[11]` asks for `0 GetEnvVars'-port'`, so it looks for `-port=n`. A service installed with the command that Plodder prints therefore gets no early Ride. Either the code or the two documents need to change. `-ride` is the better name, because `-port` could easily be read as the HTTP port.

The error message in `CheckForRide[31]` also reports `'Serve::',⍕port`, while the call on `CheckForRide[21]` used `'Serve:',address,':',⍕port`.

### 2. `Run` needs a test flag that the INI file does not have (checked)

`Run[25]` reads `G.INI.Get'CONFIG:TestFlag'` without a default when `y` does not supply a test flag. IniFiles signals "No default defined!" for a missing key; I confirmed this with IniFiles 6.1.0. Neither `server.ini.remove-me` nor `Tests/test.ini` has a `TestFlag` entry. So `Plodder.Run ⍬`, or any production start that passes no flags, fails at this line.

The ReadMe only shows `({1}Plodder.Run)0 1`, which passes both flags and therefore works. Suggestion: `0 G.INI.Get'CONFIG:TestFlag'`, plus a commented entry in the shipped INI file.

### 3. Skipped crashes leave no trace with the shipped defaults

This one is about the new storm guard. When a crash does not get the full treatment, HandleError writes only the log entry (through `logFunction`, which is `AppLogError`) and the Windows event. The shipped INI file has `[LOGGING]Log = 0`, so there is no `app-log.txt`, and `AppLog` fails silently. The Windows Event Log is off by default except for a service (see 12). The result is that a server installed with the defaults records nothing at all about skipped crashes, apart from the counter in `StormGuard.txt`.

The "Crash storms" section of the ReadMe says those occurrences are recorded in `app-log.txt`. That is true only with `Log = 1`. Either ship `Log = 1`, or say in the ReadMe and in the INI comment that the guard needs the application log to be on to leave a record.

### 4. Wrong file name for `app-log.txt` (checked)

`OpenAppLog[2]` builds the file name as `(⊃G.INI.Get'Logging:Folder'),'app-log.txt'`, without a separator. `FilesAndDirs.ExpandPath` keeps a trailing separator when there is one and does not add one otherwise (checked: `C:/Temp/Plodder-Logs` stays `C:\Temp\Plodder-Logs`). The shipped INI file has `Folder = '<TEMP>/Plodder-Logs'`, with no trailing slash, so switching on `Log = 1` creates `…\Temp\Plodder-Logsapp-log.txt`. `Tests/test.ini` has a trailing slash, which is why the tests do not notice.

I did not check whether Rumba builds the path of `log.txt` the same way. `[TRAP]Folder` has the same shape and is passed on to HandleError.

### 5. The handler check in `CheckServerParms` never fires (checked)

`CheckServerParms[8]` is meant to reject handlers that are not monadic or ambivalent or that have no result. For a single name `⎕AT` returns a nested 4-item vector, so `(1 ¯2⍷⍨|1 2⊃v)∧…` gives a two-item vector for every handler, and `b` becomes nested. `1∊b` then looks for a scalar 1 among two-item vectors and is always 0. I confirmed this with a niladic handler: `b` is `(1 1)(0 1) 0` and `1∊b` is 0. A niladic, dyadic or result-less handler passes the check and fails at the first request.

Even when unnested, the expression would be wrong: `|` turns an ambivalent ¯2 into 2, which then does not match the ¯2 in `1 ¯2`.

### 6. An empty `OnRequest` passes the checks

`CheckServerParms[5]` treats an empty handler name as valid (`0=≢⍵:3`), which is right for the three optional handlers. `OnRequest` is mandatory, as the INI comment says, but `OnRequest = ''` is accepted and the server only fails when the first request comes in.

### 7. An invalid `Context` fails before it is checked

`Run[32]` executes `⍎(⊃G.INI.Get_'App:Context'),'.G←G'` before `Run[34]` calls `CheckServerParms`, which checks `Context`. An empty `Context` gives a SYNTAX ERROR there, and a namespace that does not exist a VALUE ERROR, instead of the clear message "Invalid: Context". Moving the line after `CheckServerParms` fixes this. (`Get_` here instead of `Get` is harmless but looks accidental.)

### 8. `ServerLog`'s fallback fails itself (checked)

When building the message fails, `ServerLog[12]` writes `'Trying to set msg_ in "',(⊃⎕XSI),'" crashed with ',qdmx.DM`. `qdmx.DM` is a vector of three character vectors, so `msg_` is nested, and the untrapped `⎕NAPPEND` on `ServerLog[15]` then signals DOMAIN ERROR (checked). The fallback also drops the original message and the line ending.

A smaller inconsistency: the branch for a nested message checks `0>serverInstance.LogFileTie` before writing, and the branch for a simple message does not.

### 9. Credentials in crash files and in `log.txt`

`SaveCrashRequest` masks the headers listed in `SecretHeaders`. These can still reach the disk in clear text:

* the request body, up to 64 KB (`SaveCrashRequest[51]`), for example a login form with a password;
* tokens in the query string, which are part of `RequestTarget`;
* common headers that are not in the list, such as `X-API-Key` and `X-Auth-Token`;
* the raw `LastBlock` (`SaveCrashRequest[33]`) is masked line by line, but a body in it is not.

Separately, `CollectStartParms[8]` writes every INI entry into the start-up report in `log.txt`. The INI file is shared with the application, so whatever an application keeps there (database passwords, API keys) ends up in the log.

The ReadMe claims only that header values are masked, which is accurate. It would still help to say plainly that bodies and URLs are saved as they are.

### 10. `PolishInifilePath` cannot find an INI file next to the executable

On Windows, `PolishInifilePath[4]` enters the inner block only when `F.IsFile filename` is false. `F.IsFile F.PWD,'/',filename` on the next line tests the same file, because a relative name is resolved against the current directory. So the inner block can never run. If it did, `PolishInifilePath[8]` would put the executable's folder in front of a path that is already absolute.

The leading comment says the name is passed on to `GetAppFolder`, which does not happen.

### 11. `GetEnvVars` returns values in lower case

`GetEnvVars[15]` applies `⎕C` to the whole command line, so the values come back in lower case as well as the names. That does not matter for `-port` or `OFF`, but a path or a password passed this way would arrive changed. `GetEnvVars[22]` is unreachable, because the first branch already handles `name=value`. The name is also misleading: the function reads the command line, not environment variables. The examples in the comments mix `-flag` and `-foo`, and `GetEnvVars[4]` has the typo "∪alues".

### 12. Windows Event Log: three different descriptions

* ReadMe, feature 2: the Windows Event Log is written "when running under a Dyalog Runtime instance".
* INI comment for `WindowsEventLog = ¯1`: "Write Start/Stop/Error events to Windows Event Log in Runtime".
* Code, `CreateServerParms[30]`: ¯1 means `IsRunningAsService`, which tests for `RunAsService=1` on the command line.

Nothing in Plodder adds `RunAsService=1`; `PrintInstallAsServiceCommand` does not print it. Unless Windows or Dyalog adds it when it starts the service, the default never switches the Event Log on. I could not verify this without installing a service. Either way, the requirement should be documented.

### 13. The sample application uses an undefined `H`

`SampleApp.OnRequest[9]` and `[13]` call `H.ErrorPage`. `SampleApp.Init` sets `A F G M R`, but nothing defines `H`, so a POST or an unknown method causes a VALUE ERROR and a 500 instead of the intended 501 or 400. The 400 message ("unknown type; must be undefined or json…") also does not fit an unknown method.

Smaller points in the sample application:

* `CreateShutdownPage[4]` builds a link with the INI port, which is `http://localhost:0/` with the shipped INI file, and is always `http`.
* `DeliverFile` sends random text as `application/zip`.
* `Markdown2HTML` builds `meta` and never uses it, and `<title="…">` would not be valid HTML anyway.
* The function name `CompileHomepPage` has a typo.

### 14. Dead code, all of it broken

* `Shutdown` is called from nowhere. `Shutdown[4]` closes `##.DRC`, but Conga lives in `R.##`, so the `6::` guard swallows a VALUE ERROR and Conga is never closed.
* `GetAppFolder` is called from nowhere. Its service branch (`GetAppFolder[19]`) returns the folder without `filename`, while the other branches append it. The folders it suggests for Linux and macOS (`/etc/…`, `/usr/local/…`) are not writable for an ordinary user, which contradicts its own leading comment.
* `#.r` calls `Plodder.Run 1`, but `Run` is an operator, so the line cannot work.

I suggest deleting all three rather than fixing them.

### Smaller code points

* `CopyRumba` takes its flag as the optional left argument `enforce` and ignores the right argument. History 1.12.0 says that the right argument is the flag.
* `SetTrap[5]` defaults `force` to 0, so an omitted left argument means no trap even in a runtime. The leading comment says that trapping is always established in a runtime. `Run` always passes the flag, so this only affects other callers.
* `Run[30]` assigns `trapFlag` from the INI file and never uses it again.
* `Run[4]` and `Run[5]` set `⎕IO` and `⎕ML` without localising them, which changes them for the whole `Plodder` namespace. `quadVars` sets the same values, so there is no effect today.
* `Run[58]` passes `'TATIN_SRV'` to `R.Start`. That looks like a leftover from the Tatin server.
* `CreateServerParms[22]` hard-codes `TLSFlag←32` with no comment and no INI entry.
* `CheckServerParms[19]` breaks with a DOMAIN ERROR rather than the intended message when `LogHTTP` is not a scalar. `Port`, `Secure`, `MaxNoOfThreads`, `Log`, `LogConga` and `LogRumba` are not validated at all.
* `AppLog[1]` still refers to `Log`, renamed to `ServerLog` in 1.12.0.
* `CopyConga` copies `conga.dws` from the current directory when the copy from the interpreter fails, but still returns the interpreter's folder for the DLLs. The two can then belong to different versions of Conga.

### Conventions from your CLAUDE.md

* Inline I-beams: `3502⌶` four times in `CheckForRide`, and `739⌶0` in `ProcessIniFile[25]` and `SampleApp.DeliverFile[1]`. `IsConnectedToRide`, `EnableCONTINUE_workspaces` and `ExposeRootNameProperties` show the pattern to follow.
* `Assert` has three diamonds on one dfn line.
* The dfns in `ProcessIniFile[33]` to `[38]` read `G` and `F` from the function around them, and `[33]` assigns to `G.HTDOCS` from inside a dfn.
* `ProcessIniFile[26]` computes `home` in the `<TEMP>` block and never uses it (copied from the block above).

## Documentation

### Missing

* How to start Plodder in production. The release workspace has an empty `⎕LX`, because `SaveProjectWorkpace` sets `⎕LX←''`. Step 5 of the installation ("Execute `⎕LX`") therefore does nothing. The ReadMe should say what `⎕LX` an application should set, including for a Windows Service.
* The arguments of `Run`. The optional left argument (INI file name), the operand `fns` (what it gets, what it must return) and the right argument `testFlag trapFlag` with ¯1 meaning "take it from the INI file" are documented only in the leading comment of `Run`.
* The handler interface. What `OnStart`, `OnHeader`, `OnRequest` and `OnCongaTimeout` get as argument and must return. Today a reader has to work it out from the sample application.
* INI entries that the code reads but the shipped INI file does not show: `[CONFIG]TestFlag` (see 2), `[CONFIG]CongaName`, `[CERTIFICATES]CA_Folder`.
* `LogHTTP = 2`. `CheckServerParms` accepts 2, but nothing says what it does.
* What `[LOGGING]Log` controls. The ReadMe says it switches the application log on and off, but `CreateServerParms` also passes it to Rumba as `parms.Log`.
* What happens after a crash. `∆Crashed` ends with `⎕OFF 125` so that the service manager restarts the server. The ReadMe does not mention the exit code or that a restart has to be configured. The comment "We want to be able to continue!" on `killThreads←0` in `SetErrorParms` suggests the opposite.
* `G` in the application's namespace. `Run` assigns `G` into the `Context` namespace. The ReadMe says that `G` is accessible, but not under which name, and not that an existing `G` there is overwritten.

### Wrong or unclear

* "By default, only HTTP events are logged" (feature 1). The shipped INI file has `LogConga = 1` and `LogRumba = 1`.
* Feature 4 shows `PrintInstallAsServiceCommand 0` twice with different output. The second call should be `4501`. `PrintUninstallServiceCommand` is shown without its argument and with a different prefix (`Admin.` instead of `Plodder.Admin.`).
* Conga section: "If you don't want neither of this" should be "If you want neither"; "all platform" should be "all platforms". It says that `conga.dws` must contain "the class `Conga`", while the code copies the namespace `Conga`.
* "Access to Server Configuration": `(for "Globals),",` is garbled, and "an applications" should be "an application".
* The INI file's first comment says `<INIFILE>` refers to "the folder this log files lives in"; it means the INI file.
* Feature 2 and the INI comment on `WindowsEventLog`: see 12.

### Form

The items of the "Features" list and the paragraph under "I don't want that" contain hard line breaks (trailing double spaces and wrapped lines). According to your writing rules, each paragraph and list item should be one line.

## Release readiness for 2.0.0

* `packages/apl-buildlist.json` still points HandleError at `http://localhost:444/`. The folder name now says `5.2.0` without a beta suffix; the URL needs to become `https://tatin.dev/` once 5.2.0 is published there.
* `Version` returns `2.0.0-beta-1+271` dated 2026-11-24, while History and the release notes say 2.0.0 from 2026-11-23. The beta suffix has to go during Make, and the dates should agree.
* `ReleaseNotes.html` is copied into the distribution by `Make`. It already contains the 2.0.0 section, so it seems to have been regenerated; check that it matches the final `ReleaseNotes.md`.
* I have not run the test suite, because it needs the test server.
* Findings 1, 2 and 3 affect how a freshly installed 2.0.0 behaves. Fixing them before the release would be worthwhile, 3 in particular, because it concerns the new feature.

## How the checks were done

The checks ran in Dyalog 20.0 started with `DYALOG_NOPOPUPS=1`, with the project's packages loaded through `⎕SE.Tatin.LoadDependencies`, in a folder in the scratchpad. Two runs stopped on an error in my own input and saved a `CONTINUE.dws` there, which I deleted. No interpreter crashed.

One suspicion turned out to be unfounded: the `<TEMP>` replacement in `ProcessIniFile` does not break on Windows backslashes, because `739⌶0` returns forward slashes.

## Status at the end of the day

All changes are in the working tree and not committed.

Fixed:

* All findings, 1 to 14. For 14, `#.r` was deleted, and `Shutdown` and `GetAppFolder` were fixed and kept: `Shutdown` now kills all other threads first and then closes Conga in `RumbaLean`, and `GetAppFolder` appends the file name in every case and uses folders the user can write to on Linux and macOS.
* 3: the shipped INI file sets `[LOGGING]Log = 1`, and the "Crash storms" section explains why.
* 9: request bodies are left out of crash reports unless `[TRAP]SaveRequestBody` is 1. Query parameters are masked like headers, and the start-up report masks INI entries whose keys mark them as credentials. `[TRAP]MaskHeaders`, `[TRAP]MaskQueryParms` and `[LOGGING]MaskIniKeys` add names to the built-in lists. A new ReadMe section describes what is masked and what is not.
* 10: a relative INI file name is always resolved against the current directory; the executable's folder is no longer used.
* 11: `GetEnvVars` was also renamed to `GetCommandLineValue`.
* 12: `WindowsEventLog = ¯1` now means "in a Runtime", as the INI comment and the ReadMe say.
* `CheckServerParms` now also validates `LogConga` and `LogRumba`, and the ReadMe says that `[LOGGING]Log` controls only `app-log.txt`: Plodder passes it to Rumba, but no function in Rumba's `Core` reads it.
* All smaller code points. `CopyRumba`'s callers now pass the flag on the left. `0 SetTrap`, `[TRAP]Policy = 0` and a trap flag of 0 now mean no error trapping, in a Runtime as well. `CopyConga` reports its fallback to `./conga.dws` in `app-log.txt`, and `ServerLog` does nothing when there is no `log.txt`.
* All convention points and all documentation points. The ReadMe has a new section "The handlers", and it says what `LogHTTP` 1 and 2 write, with a warning that 2 writes credentials unmasked.
* The sample application now routes on `request.Path`, so a query string no longer leads to a 404.
* An error in `OnHeader` stopped the server, because Rumba called that handler without `TrapRequest`; and when `OnHeader` answered a request itself, the request still went on to `OnRequest`. Both are fixed in RumbaLean 3.5.3, which Plodder ships. `Test_Server_125` and a check in `Test_Server_999` cover them, and RumbaLean has `TestErrorInOnHeader`.
* `GetAppFolder` takes the folder of a Windows Service from `⎕WSID`, not from the second command line argument, which a bound EXE does not have.
* `LogHTTP = 2` stopped the server from starting, because Rumba's `OpenLogFile` failed with a DOMAIN ERROR. Fixed in RumbaLean; Plodder ships the new `RumbaLean.dws`, and `Test_Misc_015` covers it.
* The side note to 4 was checked: Rumba builds the path of `log.txt` correctly with or without a trailing separator, and HandleError copes with an error folder without one.

New tests: `Test_Misc_005` to `Test_Misc_015` and `Test_Server_125`. The full test suite passes. `Test_Misc_001`, `Test_Misc_003` and `Test_Server_120` were extended.

Still open:

* Release items: none. The HandleError source in `apl-buildlist.json` and the version number are normal for a development version: Tatin warns about the first when publishing, and the version is set when a new version is built, which also generates `ReleaseNotes.html`.

