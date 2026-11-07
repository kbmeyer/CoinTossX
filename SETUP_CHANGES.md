# CoinTossX Windows setup: change log

Every change made to get CoinTossX building and running on Windows.
Add a row for each change. Mark status as Done, Applied (not yet tested), or Reverted.

| # | Date | Change | Where | Why | Status |
|---|------|--------|-------|-----|--------|
| 1 | 2026-11-07 | Installed Oracle JDK 8u511 (Windows x64 installer). The JRE was installed alongside it. | `C:\Program Files\Java\jdk1.8.0_511` | README requires Java 8 | Done |
| 2 | 2026-11-07 | Set system variable `JAVA_HOME` to the JDK folder. Typo `_551` was corrected to `_511`. | Windows system environment variables | Build and scripts use `JAVA_HOME`; `javac` needs it | Done |
| 3 | 2026-11-07 | Added `%JAVA_HOME%\bin` to the system `Path`, placed before the `java8path` entry. | Windows system environment variables | Makes `javac` available in terminals | Done |
| 4 | 2026-11-07 | Changed `SOFTWARE_PATH`, `DATA_PATH` and `MEDIA_DRIVER_DIR` from Linux paths to `C:/Projects/CoinTossX-run/...`. | `windows.properties` | Linux paths do not exist on Windows | Done (`MEDIA_DRIVER_DIR` value not yet verified) |
| 5 | 2026-11-07 | Added a Gradle `resolutionStrategy` block that forces `com.google.guava:guava:28.2-jre` for all projects. | `build.gradle` (root, inside `allprojects`) | `mapdb 3.0.8` requests Guava `[15.0,)`, which resolves to 33.8.0-jre. Gradle 6.7.1 cannot choose a variant from it and the build fails in `:Common:compileJava`. | Done (build passes) |
| 6 | 2026-11-07 | Built with `gradlew.bat -Penv=windows build -x test`, then deployed with `clean installDist bootWar copyResourcesToInstallDir copyToDeploy deployLocal`. | Build output and `C:\Projects\CoinTossX-run` | Creates the runnable application. Build passed; deploy printed "Clean Software directory" then "Copy files to server" and finished successfully. | Done (contents not yet checked) |
| 7 | 2026-11-07 | Changed the two Windows Firewall rules "Java(TM) Platform SE binary" (inbound allow) from Public to Private networks only, using `Set-NetFirewallRule -Profile Private`. | Windows Firewall | Firewall prompt was answered with Public ticked. Private only is safer for a local simulation. | Done |

## Not changed by us
- The working copy showed 90 modified files before any edit. The changes are line endings only (CRLF vs LF). Check with `git diff --ignore-cr-at-eol --stat`.

## Open items
- Verify `MEDIA_DRIVER_DIR` works on Windows. The README and `deploy_local.gradle` (which deletes `/dev/shm/aeron`) assume Linux.
- Run the deploy step and confirm files appear in `C:\Projects\CoinTossX-run`.

## Run log
| # | Date | Step | Result | Status |
|---|------|------|--------|--------|
| R1 | 2026-11-07 | Started `scripts\startLowLatencyDriver.bat` (Aeron media driver, `aeron.LowLatencyMediaDriver`) in its own window. It runs in the foreground, so keep that window open. | Java process 20624 is running with `C:/Projects/CoinTossX-run/aeron`. `aeron` folder created. `Socket.log` is empty, which is expected because the driver prints little to standard output. | Running |
| — | 2026-11-07 | Java process 7540 is the Gradle daemon left over from the build. Not part of CoinTossX. | — | Harmless |
| R2 | 2026-11-07 | Started `scripts\startTGDriver.bat` (trading gateway media driver) in its own window. | Java process 9532 running with `aeron\tradingGateway`. `TGSocket.log` empty, as expected. | Running |
| R3 | 2026-11-07 | Started `scripts\startMDGDriver.bat` (market data gateway media driver) in its own window. | Java process 25816 running with `aeron\marketDataGateway`. | Running |
| R4 | 2026-11-07 | Started `scripts\startNativeGateway.bat` (client order entry gateway) in its own window. | Java process 12960 running `gateway.incoming.NativeGateway` from `NativeGateway\lib`. | Running |
| R5 | 2026-11-07 | Started `scripts\startMatchingEngine.bat` (trade matching engine) in its own window. Output goes to `MatchingEngine.log`. | Java process 26036 running `engine.MatchingEngine` from `MatchingEngine\lib`. | Running |
| R6 | 2026-11-07 | Started `scripts\startMarketDataGateway.bat` (market data gateway). A Windows Firewall prompt appeared and was allowed, then changed to Private only (see change 7). | Java process 6532 running `gateway.marketdata.NativeMarketDataGateway` from `MarketDataGateway\lib`. | Running |
| 8 | 2026-11-07 | Pinned `eclipse-collections`, `eclipse-collections-api` and `eclipse-collections-forkjoin` to `11.1.0` in the same `resolutionStrategy` block. | `build.gradle` (root) | The web app bundled `eclipse-collections` 13.0.0, which is compiled for Java 17 (class file version 61.0). Java 8 only accepts up to version 52.0, so `Web` failed at start-up in `dao.OffHeapStorage`. The build passed because this error appears only at run time. | Applied (not yet tested) |
| R7 | 2026-11-07 | Web app failed at start-up with "Web client is unable to login to Trading Gateway". The real cause was that `data\webStorage.db` didn't exist. The web app opens it read-only, and `WebEventListener` creates it. Started the web driver manually (`Socket.bat C:/Projects/CoinTossX-run/aeron/web`) and then `WebEventListener.bat`. | `webStorage.db` created; web app started. Windows `startAll.bat` does not start the web driver or WebEventListener. | Done |
| R8 | 2026-11-07 | Web app running at `http://localhost:8080`. Stock list shows STKA to STKJ, all in Continuous Trading. | Working end to end. | Done |

## Windows start-up gaps found
- `scripts\startAll.bat` skips the web driver (`startWebDriver`) and `WebEventListener`. The Linux `startAll.sh` starts both. Windows also has no `.bat` launcher for either, so they were started by hand.
- The web app must start after `WebEventListener` has created `data\webStorage.db`.
| R9 | 2026-11-07 | Ran `scripts\startHawkesSimulation.bat 1 1` from the terminal, stopped after about 30 seconds with Ctrl+C. | Log shows the simulator connected, logged in three times, received market data on the multicast address, sent a start message and reported `Size = 110889`. It then stopped at the prompt `Terminate batch job (Y/N)?`. Meaning of Y/N not yet confirmed. | Partly working |

## Open items (added 2026-11-07)
- **Web Start button does nothing:** Wicket fails to serialize a `mapdb` collection (`org.mapdb.BTreeMap$values$1`) on the page, so the Start action never runs. Web.log shows the repeated `NotSerializableException`. Not yet traced to a specific page.
- **Web Stop button:** on Windows, `stopHawkesSimulation.bat` doesn't exist, so processes must be stopped with Ctrl+C.
- **Stopping a simulation:** Ctrl+C in the window, then Y, to stop. The `Terminate batch job (Y/N)?` text is the Windows cmd prompt for Ctrl+C in a .bat file. The simulator code has no keyboard input.
| R10 | 2026-11-07 | Ran the Hawkes simulator for client 1, stock 1 (`HawkesSimulation 1 1`) and viewed STKA in the web app. | Orders appeared in the order book, then trades appeared in the trades table. End-to-end path confirmed: simulator, native gateway, matching engine, market data, web app. | Done |
