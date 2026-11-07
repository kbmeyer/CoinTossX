# CoinTossX Windows setup: change log

Every change made to get CoinTossX building and running on Windows, and every run. Add a row for each new change or run. Status: Done, Applied (not yet tested), Running, or Open.

Source copy: `SETUP_CHANGES.md` in the repo, branch `windows-setup` on kbmeyer/CoinTossX. Keep the Claude project copy in step with it.

## Changes to the set-up

| # | Date | Change | Where | Why | Status |
|---|------|--------|-------|-----|--------|
| 1 | 2026-11-07 | Installed Oracle JDK 8u511 (Windows x64 installer). The JRE was installed alongside it. | `C:\Program Files\Java\jdk1.8.0_511` | README requires Java 8 | Done |
| 2 | 2026-11-07 | Set system variable `JAVA_HOME` to the JDK folder. Typo `_551` was corrected to `_511`. | Windows system environment variables | Build and scripts use `JAVA_HOME`; `javac` needs it | Done |
| 3 | 2026-11-07 | Added `%JAVA_HOME%\bin` to the system `Path`, placed before the `java8path` entry. | Windows system environment variables | Makes `javac` available in terminals | Done |
| 4 | 2026-11-07 | Changed `SOFTWARE_PATH`, `DATA_PATH` and `MEDIA_DRIVER_DIR` from Linux paths to `C:/Projects/CoinTossX-run/...`. | `windows.properties` | Linux paths do not exist on Windows | Done (`MEDIA_DRIVER_DIR` value not yet verified beyond this run) |
| 5 | 2026-11-07 | Added a Gradle `resolutionStrategy` block forcing `com.google.guava:guava:28.2-jre` for all projects. | `build.gradle` (root, inside `allprojects`) | `mapdb 3.0.8` requests Guava `[15.0,)`, which resolved to 33.8.0-jre. Gradle 6.7.1 cannot choose a variant from it, so `:Common:compileJava` failed. | Done (build passes) |
| 6 | 2026-11-07 | Built with `gradlew.bat -Penv=windows build -x test`, then deployed with `clean installDist bootWar copyResourcesToInstallDir copyToDeploy deployLocal`. | Build output and `C:\Projects\CoinTossX-run` | Creates the runnable application. Deploy finished with `BUILD SUCCESSFUL`. | Done |
| 7 | 2026-11-07 | Changed the two Windows Firewall rules "Java(TM) Platform SE binary" (inbound allow) from Public to Private networks only, with `Set-NetFirewallRule -Profile Private`. | Windows Firewall | The firewall prompt had Public ticked. Private only is safer for a local simulation. | Done |
| 8 | 2026-11-07 | Pinned `eclipse-collections`, `eclipse-collections-api` and `eclipse-collections-forkjoin` to `11.1.0` in the same `resolutionStrategy` block. | `build.gradle` (root) | The web app bundled `eclipse-collections` 13.0.0, compiled for Java 17 (class file version 61.0). Java 8 accepts up to version 52.0, so `Web` failed at start-up in `dao.OffHeapStorage`. The build passed because this error only appears at run time. | Done (web app starts, see R8) |
| 9 | 2026-11-07 | Line-ending clean-up: restored the 90 files that differed from the commit only in CRLF vs LF, and reapplied the two edits (`build.gradle`, `windows.properties`) with LF endings. | Working copy | Makes the commit contain only the real changes | Done |

## Run log

| # | Date | Step | Result | Status |
|---|------|------|--------|--------|
| R1 | 2026-11-07 | Started `scripts\startLowLatencyDriver.bat` (Aeron media driver, `aeron.LowLatencyMediaDriver`) in its own window. It runs in the foreground, so keep that window open. | Java process 20624 running with `C:/Projects/CoinTossX-run/aeron`. `aeron` folder created. `Socket.log` empty, which is expected. | Running |
| — | 2026-11-07 | Java process 7540 is the Gradle daemon left over from the build. Not part of CoinTossX. | — | Harmless |
| R2 | 2026-11-07 | Started `scripts\startTGDriver.bat` (trading gateway media driver) in its own window. | Java process 9532 running with `aeron\tradingGateway`. `TGSocket.log` empty, as expected. | Running |
| R3 | 2026-11-07 | Started `scripts\startMDGDriver.bat` (market data gateway media driver) in its own window. | Java process 25816 running with `aeron\marketDataGateway`. | Running |
| R4 | 2026-11-07 | Started `scripts\startNativeGateway.bat` (client order entry gateway) in its own window. | Java process 12960 running `gateway.incoming.NativeGateway` from `NativeGateway\lib`. | Running |
| R5 | 2026-11-07 | Started `scripts\startMatchingEngine.bat` (trade matching engine) in its own window. Output goes to `MatchingEngine.log`. | Java process 26036 running `engine.MatchingEngine` from `MatchingEngine\lib`. | Running |
| R6 | 2026-11-07 | Started `scripts\startMarketDataGateway.bat` (market data gateway). A Windows Firewall prompt appeared; allowed, then changed to Private only (change 7). | Java process 6532 running `gateway.marketdata.NativeMarketDataGateway` from `MarketDataGateway\lib`. | Running |
| R7 | 2026-11-07 | Web app failed at start-up with "Web client is unable to login to Trading Gateway". The real cause was that `data\webStorage.db` did not exist. The web app opens it read-only; `WebEventListener` creates it. Started the web driver by hand (`Socket.bat C:/Projects/CoinTossX-run/aeron/web`), then `WebEventListener.bat`. | `webStorage.db` created; web app started. Windows `startAll.bat` does not start the web driver or WebEventListener. | Done |
| R8 | 2026-11-07 | Web app running at `http://localhost:8080`. Stock list shows STKA to STKJ, all in Continuous Trading. | Working. | Done |
| R9 | 2026-11-07 | Ran `scripts\startHawkesSimulation.bat 1 1` from the terminal, stopped after about 30 seconds with Ctrl+C. | Simulator connected, logged in three times, received market data on the multicast address, sent a start message and reported `Size = 110889`. Stopped at the prompt `Terminate batch job (Y/N)?`. | Partly working |
| R10 | 2026-11-07 | Ran the Hawkes simulator for client 1, stock 1 (`HawkesSimulation 1 1`) and viewed STKA in the web app. | Orders appeared in the order book, then trades appeared in the trades table. End-to-end path confirmed: simulator, native gateway, matching engine, market data, web app. | Done |

## Findings about the Windows start-up

- `scripts\startAll.bat` skips the web driver (`startWebDriver`) and `WebEventListener`. The Linux `startAll.sh` starts both. Windows has no `.bat` launcher for either, so they were started by hand.
- The web app must start after `WebEventListener` has created `data\webStorage.db`.
- `stopHawkesSimulation` exists only as a `.sh` script, so Windows has no stop script.

## Not changed by us

- The working copy showed 90 modified files before any edit. The changes were line endings only (CRLF vs LF). Resolved in change 9. Check with `git diff --ignore-cr-at-eol --stat`.

## Open items

- **Web Start button does nothing.** Wicket cannot serialise an `org.mapdb.BTreeMap$values$1` on a page (`NotSerializableException` in `Web.log`), so the Start action never runs. Not yet traced to a page class. Terminal runs are unaffected.
- **Web Stop button on Windows.** Sends `SimulationComplete`, which is what writes results. Whether it completes cleanly on Windows is unverified.
- **Results files not yet seen.** No `TestResults_` folder or `hdrLatency.txt` has been produced, because no Stop with `SimulationComplete` has been sent.
- **`MEDIA_DRIVER_DIR` on Windows.** Works for this run, but the README and `deploy_local.gradle` (which deletes `/dev/shm/aeron`) assume Linux.
- **Stopping a simulation.** Ctrl+C in the window, then Y. The `Terminate batch job (Y/N)?` text is the Windows cmd prompt for Ctrl+C in a `.bat` file. The simulator code has no keyboard input.
- **Multi-client and auction sessions** are not tested.
