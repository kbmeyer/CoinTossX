# CLAUDE.md: CoinTossX on Windows

Guidance for Claude working in this repo. Facts here are verified from the code and from runs on this machine unless marked **[Unverified]**.

## Repository and branches
- Upstream: `origin` = IvanJericevich/CoinTossX (read only, never push here).
- Our fork: `myfork` = kbmeyer/CoinTossX (push target).
- Branch `windows-setup` is based on `abm-experiment` at commit `08f4780`. It holds the Windows set-up changes.
- Planned later: a `notebook` branch from `windows-setup`.

## Environment
- Windows with PowerShell. Java 8 only: JDK 8u511 at `C:\Program Files\Java\jdk1.8.0_511`. `JAVA_HOME` and `Path` are set (system variables).
- Run folder: `C:\Projects\CoinTossX-run`. It is a separate deployed copy. The deploy task clears it, so edit files in the repo and redeploy. Never edit the run folder.
- Repo folder: `C:\Projects\CoinTossX`.
- Deletion is enabled for `C:\Projects\CoinTossX` for the current session only.

## Working rules
1. Ask before editing repo files, deleting files, building, deploying, or pushing.
2. Never push to `origin`. Push only to `myfork`.
3. Keep repo files in LF line endings. The commit stores LF. Do not reintroduce CRLF.
4. Log every change and run in `SETUP_CHANGES.md`, with a row number, the reason and the status.
5. Mark anything unverified as **[Unverified]**. Prefer reading the code to guessing.
6. Commit messages end with:
   ```
   Co-Authored-By: Claude Haiku 5.5 <noreply@anthropic.com>
   Claude-Session: https://claude.ai/code/session_01KoKqhj8GSCY4ZAaAr1gzzc
   ```

## Build and deploy
```powershell
cd C:\Projects\CoinTossX
.\gradlew.bat -Penv=windows build -x test
.\gradlew.bat -Penv=windows clean installDist bootWar copyResourcesToInstallDir copyToDeploy deployLocal
```
Close all running CoinTossX windows before deploying. Open files cannot be overwritten.

Version pins in `build.gradle` (root, inside `allprojects`), needed for Gradle 6.7.1 and Java 8:
- `com.google.guava:guava:28.2-jre`. mapdb 3.0.8 requests `[15.0,)`, which resolves to a release Gradle 6.7.1 cannot read.
- `eclipse-collections`, `-api` and `-forkjoin` at `11.1.0`. Version 13.0.0 is compiled for Java 17.

## Start-up order
Each process runs in its own PowerShell window, from `C:\Projects\CoinTossX-run\scripts`:
1. `.\startLowLatencyDriver.bat` (main Aeron driver)
2. `.\startTGDriver.bat` (trading gateway driver)
3. `.\startMDGDriver.bat` (market data gateway driver)
4. Web driver, by hand: `cd ..\Socket\bin` then `.\Socket.bat C:/Projects/CoinTossX-run/aeron/web`
5. `.\startNativeGateway.bat`
6. `.\startMatchingEngine.bat`
7. `.\startMarketDataGateway.bat` (allow Java on Private networks if the firewall asks)
8. WebEventListener, by hand: `cd ..\WebEventListener\bin` then `.\WebEventListener.bat`. It creates `data\webStorage.db`.
9. `.\startWeb.bat`, then open `http://localhost:8080`

Windows `startAll.bat` skips steps 4 and 8. Check the process list with `Get-CimInstance Win32_Process -Filter "Name='java.exe'"`.

## Simulation
- Terminal: `.\startHawkesSimulation.bat 1 1` (client 1, stock 1). Output goes to `HawkesSimulation_1_1.log` in the run folder.
- Stop with Ctrl+C, then Y. The `Terminate batch job` prompt is the standard Windows prompt for Ctrl+C in a `.bat` file.
- Results are written only when the matching engine receives a `SimulationComplete` admin message. Only the web app's Stop button sends it. Killing the simulator does not write results.

## Known issues
- **Web Start button does nothing.** Wicket cannot serialise an `org.mapdb.BTreeMap$values$1` on a page (`NotSerializableException` in `Web.log`). Not yet traced to a page class. Terminal runs are unaffected.
- **Web Stop button on Windows** sends `SimulationComplete`. Whether it completes cleanly on Windows is **[Unverified]**.
- **`stopHawkesSimulation`** exists only as `.sh`. There is no Windows equivalent.
- **`MEDIA_DRIVER_DIR` on Windows** works for this run. The README and `deploy_local.gradle` (which deletes `/dev/shm/aeron`) assume Linux.
- **HORIZON** in `data\hawkesData.properties` is a model time horizon, not wall-clock seconds. It sets the number of simulated events.

## Key files
- `windows.properties`: Windows paths and ports.
- `build.gradle` (root): Gradle config and dependency pins.
- `data\clientData.csv`: clients. Client 999 is the admin client the web app logs in as.
- `data\hawkesData.properties`: simulator parameters.
- `SETUP_CHANGES.md`: change and run log.
- `CoinTossX_Windows_Setup_and_Requirements.md`: full set-up and requirements document.
