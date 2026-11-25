# Plodder Release Notes


## 2.0.0 from 2026-11-25
* **BREAKING CHANGES** in MarkAPL 16, which applications can use as `M`:
  * `Init` and `MakeHTML_Doc` no longer change the parameter namespace passed to them. Whatever `Init` establishes, including parameters embedded in the document, is now in `ns.parms`. So call `ns.parms M.MakeHTML_Doc ns.html` rather than `parms M.MakeHTML_Doc ns.html`.
  * A link in a header is now reported and shown as text. It used to produce invalid HTML.
  * The sample application's `Markdown2HTML` was adjusted accordingly.
* New crash storm guard: when the same error keeps occurring, only its 1st, 2nd, 4th, 8th... occurrence produces crash files. Every other occurrence is recorded only in `app-log.txt` and, if configured, in the Windows Event Log. See "Crash storms" in the ReadMe.
  * Three new INI entries in the `[TRAP]` section control it: `StormGuard` (default 1), `StormCeiling` (default 10) and `StormResetAfter` (default 24 hours). An existing INI file without these entries gets the defaults, so the guard is on. Set `StormGuard = 0` for the old behaviour.
  * Only one process may write to an error folder. If you run several Plodder servers on one machine, give each of them its own `[TRAP]Folder`. The value in the INI file shipped so far, `<TEMP>/Plodder-Errors`, is the same for all of them.
* A crash no longer attempts to save an error workspace. Plodder is multi-threaded by design, and a workspace cannot be saved while other threads are running.
* The start-up report in `log.txt` shows the storm guard settings in force, including the defaults for entries missing from the INI file.
* **BREAKING CHANGE**: an early Ride is now requested with `-ride=n` on the command line, as the ReadMe and `Admin.PrintInstallAsServiceCommand` always said. The code used to look for `-port=n`.
* `[CONFIG]TestFlag` now defaults to 0. `Run` failed when neither its right argument nor the INI file supplied a test flag.
* Plodder now checks more of the INI file at start-up: it rejects an empty `OnRequest`, handlers that are niladic, dyadic or return no result, and invalid values of `Port`, `Secure`, `MaxNoOfThreads`, `[LOGGING]Log`, `[LOGGING]LogConga` and `[LOGGING]LogRumba`.
* The headers `X-API-KEY` and `X-AUTH-TOKEN` are now masked in saved requests as well.
* The ReadMe has a new section "Starting the server" that explains the arguments of `Run` and what happens after a crash.
* Crash reports are more careful with credentials. See "Credentials in crash reports and logs" in the ReadMe.
  * Request bodies are no longer saved with a crash, unless the new INI entry `[TRAP]SaveRequestBody` is 1.
  * The values of query parameters such as `token` and `api_key` are masked, like those of headers such as `Authorization`.
  * The new INI entries `[TRAP]MaskHeaders` and `[TRAP]MaskQueryParms` add names to the built-in lists.
  * The start-up report in `log.txt` shows INI entries whose keys contain `PASSWORD`, `SECRET`, `TOKEN` and the like as `***`. The new INI entry `[LOGGING]MaskIniKeys` adds to that list.
* The shipped INI file now sets `[LOGGING]Log = 1`. Without `app-log.txt`, a crash that the storm guard does not save leaves no trace at all. Check your own INI file.
* **BREAKING CHANGE**: `GetEnvVars` was renamed to `GetCommandLineValue`, because it reads the command line, not environment variables.
* When `conga.dws` cannot be copied from the interpreter's folder and is copied from the current directory instead, `Run` writes a warning to `app-log.txt`: the Conga DLLs are still taken from the interpreter's folder, and they may not match.
* Bug fixes
  * `app-log.txt` got a wrong name when `[LOGGING]Folder` had no trailing separator, as in the shipped INI file.
  * An invalid `[APP]Context` caused a SYNTAX ERROR or a VALUE ERROR instead of a proper message.
  * The fallback in `ServerLog` for a message that cannot be processed failed itself with a DOMAIN ERROR.
  * `GetCommandLineValue` (formerly `GetEnvVars`) returned values in lower case.
  * `[LOGGING]WindowsEventLog = ¯1` never wrote to the Windows Event Log. It now does so in a Runtime, as documented.
  * A relative INI file name was replaced by the interpreter's folder when the file existed in the current directory. It is now always taken relative to the current directory, which `Run` sets to the folder of the workspace, on all platforms.
  * `ServerLog` failed with a FILE TIE ERROR on a single message when there was no `log.txt`.
  * `[TRAP]Policy = 0`, a trap flag of 0 in the right argument of `Run`, and `0 SetTrap` still established error trapping in a Runtime. 0 now means no error trapping at all, as documented.
  * The sample application referred to an undefined `H` for its error pages.
  * The sample application answered any request with a query string with 404: it compared the whole request target, query included, with its routes.
  * `Shutdown` never closed Conga, because it looked for `DRC` in the wrong namespace.
  * `GetAppFolder` returned the folder without the file name when running as a Windows Service.
  * `[LOGGING]LogHTTP = 2` stopped the server from starting: Rumba's `OpenLogFile` failed with a DOMAIN ERROR. Fixed by a new version of RumbaLean.
  * An error in the application's `OnHeader` stopped the server with exit code 125. The client now gets a 500, and the server carries on. Fixed by a new version of RumbaLean.
  * When `OnHeader` answered a request itself, the request still went on to `OnRequest`, which processed it although the client never saw that answer. An `OnHeader` that rejected a request, for example for missing authorization, therefore did not stop it from being processed. Fixed by a new version of RumbaLean.
* `GetAppFolder` now returns a folder the user can write to on Linux and macOS: `$XDG_DATA_HOME/{appName}/` (or `~/.local/share/{appName}/`) and `~/Library/Application Support/{appName}/`. It used to return `/etc/{appName}/` and `/usr/local/{appName}/`. For a Windows Service it now takes the folder of `⎕WSID` instead of the second command line argument, which a bound EXE does not have.
* The ReadMe has a new section "The handlers" that describes what `OnStart`, `OnHeader`, `OnRequest` and `OnCongaTimeout` get, must return, and what an error in each of them does.
* `Shutdown` now ends by calling `OFF`. In a development environment it therefore quits only when the command line carries `OFF=1`; it used to check an environment variable `OFF` as well.
* `Shutdown` now kills all other threads, Rumba's included, before it closes Conga, and waits up to five seconds for them to end.
* Packages updated: HandleError 5.2.0, MarkAPL 16.0.0, and with it CommTools 3.2.0.

## 1.14.0 from 2026-10-25
 * New Rumba included that has workarounds for the `⎕UCS`-bugs and shortcomings.
 * New HandleError with improved handling of crash info.
 * In addition to creating the JSON file with the request in cases of an error, a summary is now added to the HTML report.

## 1.13.0 from 2026-10-03
* New version of Rumba with improved error trapping
* Bug fix: saving variables in error snapshots was accidentally off.
* When a crash happens while a request is being handled, the request is saved next to the crash   files as `<crash>.request.json`: the last block of data received, exactly as it arrived, and the request(s) of the connection. Values of headers that carry credentials are masked, and data is   capped. Requires RumbaLean 3.5.0

## 1.12.0 from 2026-09-09
* **BREAKING NAME CHANGES**
  * The function `LogError` was renamed to `AppLogError`.
  * The function `Log` was renamed to `ServerLog`.
* New INI entry `[CONFIG]HouseKeepingInterval` introduced.
* New INI entry `[CONFIG]IdleConnectionTimeout` introduced.
* New version of Rumba with improved (`OnRequest`) and enhanced (`OnHeader`) error trapping.
* Optional left argument removed from `AppLogError`: was not used when specified.
* `CopyRumba` now treats its optional left argument as a flag that can be used to enforce the copy op. Was ignored so far.

## 1.11.0 from 2025-09-17
* Folder Licenses/ replaced by a file NOTIFY.md that keeps all license information

## 1.10.0 from 2025-09-13
* Folder "Licenses/" introduced that holds all LICENSE files of Plodder's dependencies
* Packages updated

## 1.9.1 from 2025-07-22
* Important bug fix by updatingthe `HandleError` package

## 1.9.0 from 2025-07-11
* Plodder copies Conga from the currently running version of Dyalog by default, and uses the suitable Conga DLLs.
* A new (optional) INI entry `[CONFIG]CongaFolder` can be used to specify a folder that is expected to hold a suitable workspace `conga.dws` as well as suitable Conga DLLs if you don't want Plodder to dynamically switch between Conga versions depending on the currently running version of Dyalog
* Packages updated
* Bug fixes
  * Conga was copied into the wrong place. As a result Plodder always used Conga 3.4

## 1.8.0 from 2025-04-10
* Packages updated
* `Make` process improved and streamlined

## 1.7.2 from 2025-03-16
* Bug fixes
  * `##.RumbaLean.Core.DRC.SetProp` was called no matter whether `CONFIG:Secure` was 1 or not.

## 1.7.1 from 2025-01-19
* RumbaLean updated
* Bug fixes
  * Fixed a VALUE ERROR in ServerLoop that I introduced myself

## 1.7.0 from 2024-11-30
* RumbaLean updated
  * Now ready for 20.0
  * Error trapping for general errors brought back

## 1.6.0 from 2024-04-20
* RumbaLean updated

## 1.5.0 from 2024-04-19
* Comments in `CheckForRide` polished
* RumbaLean updated
* Packages updated

## 1.4.1 from 2023-05-20
* New RumbaLean (3.1.1) integrated with several bug fixes

## 1.4.0 from 2023-05-02
* New RumbaLean (3.1) integrated: supports proxy
* `Prepare`, `Initial` and `Cleanup` improved
* `LogError` now adds `(⊂'*** Error')` to the message provided

## 1.3.2 from 2022-11-17
* Comes with a new version of RumbaLean that in turn fixes a minor problem in WaitForData that only hits when Rumba is interrupted by the user and no threads are running

## 1.3.1 from 2022-10-08
* Saving visible variables in the DCF file created in case of an error is now suppressed: in the vast majority of cases it would just save local Rumba variables anyway, and that does not help
* Bug fixes
  * "Make" created a new version from the currently running version of APL; since it creates a workspace this should be made explicit
  * Logging was buggy

## 1.3.0 from 2022-08-05
* Documentation corrected
* New "Make" implemented
* Packages updated

## 1.2.2 from 2022-07-19
* New version of Rumba integrated that fixed a problem with PercentDecode

## 1.2.1 from 2022-06-24
* New version of Rumba integrated that comes with a workaround for bug `<02024>`

## 1.2.0 from 2022-06-21
* Additional logging level added for application-specific logging
* New version of Rumba integrated

## 1.1.0 from 2022-06-10
* New version of Rumba whiich uses Conga 3.4 rather than 3.3

## 1.0.4 from 2022-05-29
* Improved version of Rumba (error handling)

## 1.0.3 from 2022-05-18
* Port 0 should default to either 80 or 443 (depending on "Secure") but did not

## 1.0.2 from 2021-12-23
* New version of RumbaLean

## 1.0.1 from 2021-11-20
* Problem with the "Trap" flag fixed: was not propagated from the INI file

## 1.0.0 from 2021-11-11
* Finally!
* New version of FilesAndDirs included










