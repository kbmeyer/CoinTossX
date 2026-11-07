# CoinTossX: Windows Set-up and Requirements

**Status:** working end to end on Windows (one client, one stock, simulator to web view). Items marked **[Unverified]** have not been tested.
**Repository:** IvanJericevich/CoinTossX, branch `abm-experiment`, commit `08f4780` ("Prepare for asynchronous ABM (13)").
**Our branch:** `windows-setup` on kbmeyer/CoinTossX (fork), commit `24ab49b` (setup changes and docs), then `222a3c8` (CLAUDE.md).
**Run folder:** `C:\Projects\CoinTossX-run` (set in `windows.properties`).

---

## 1. Requirements

| Item | Requirement | Source |
|---|---|---|
| Operating system | Windows with PowerShell | Used for this set-up. Exact Windows version not recorded. |
| Java | **JDK 8** (Oracle JDK 8u511 used). Java 8 only. | README: "CoinTossX is currently only compatible with version 8 of Java" |
| Build tool | Gradle 6.7.1, supplied by the wrapper (`gradlew.bat`). No separate install. Needs internet on first build. | README; `gradle/wrapper/gradle-wrapper.properties` |
| Git | Any recent version (2.54.0 used) | |
| Hardware | README suggests 4 cores and 32 GB RAM ideally. Smaller setups work with fewer clients and stocks. **[Unverified]** what this machine has. | README |
| Disk | The web app pre-allocates a 10 GB storage file (`webStorage.db`). Check free space first. | `Common/.../OffHeapStorage.java` |
| Network | Components talk over `localhost` (UDP) and one multicast address (`224.0.1.1`). | `windows.properties` |
| Firewall | Allow Java inbound on **Private** networks only. | Set during this set-up |
| Browser | Any modern browser, for the web app at `http://localhost:8080` | README |

---

## 2. Install Java 8

1. Download the **JDK 8** Windows x64 installer (`jdk-8uNNN-windows-x64.exe`) from Oracle's JDK 8 archive. The installer also installs a runtime (JRE), which can be left at its default.
2. The JDK installs to `C:\Program Files\Java\jdk1.8.0_NNN`. The JRE goes to `jre1.8.0_NNN`. **Use the JDK folder** in the next step.
3. Set environment variables (System variables, not User):
   - `JAVA_HOME` = `C:\Program Files\Java\jdk1.8.0_NNN`
   - Add `%JAVA_HOME%\bin` to `Path`, above any `java8path` entry.
4. Open a **new** PowerShell and check:
   ```powershell
   java -version      # expect 1.8.0_NNN
   javac -version     # expect javac 1.8.0_NNN (confirms the JDK, not only the JRE)
   echo $env:JAVA_HOME
   where.exe javac    # should point inside jdk1.8.0_NNN\bin
   ```

---

## 3. Get the code

```powershell
mkdir C:\Projects -Force
cd C:\Projects
git clone https://github.com/IvanJericevich/CoinTossX.git
cd CoinTossX
git checkout abm-experiment
git log -1 --oneline   # expect 08f4780
```

**Line endings.** On this machine, 90 files showed as modified with identical content (CRLF vs LF only). They were restored to the commit. Only the two edited files differ from `abm-experiment`, with LF endings kept. Check with `git diff --ignore-cr-at-eol --stat`. The changes that matter are listed in Section 6.

---

## 4. Configure

### 4.1 `windows.properties` (repo root)

The file had Linux paths. Set to:

```
SOFTWARE_PATH=C:/Projects/CoinTossX-run
DATA_PATH=C:/Projects/CoinTossX-run/data
MEDIA_DRIVER_DIR=C:/Projects/CoinTossX-run/aeron
```

`MEDIA_DRIVER_DIR` is **[Unverified]** on Windows. It worked for this run, but the README only documents `/dev/shm` for Linux.

The other entries are port and stream settings. They match the ports listed in Section 8.

### 4.2 Data files (`data\` in the run folder)

| File | Purpose |
|---|---|
| `clientData.csv` | Client logins and their ports. Client **999** is the admin client the web app uses to log in. It is removed from the client list when the web app starts. |
| `Stock.csv` | The stocks traded (STKA to STKJ in this run). |
| `Trader.csv` | Trader data. **[Unverified]** what it controls. |
| `tradingSessionsCron.properties` | When trading sessions start (cron-style schedule). |
| `hawkesData.properties` | Parameters for the Hawkes simulation. `HORIZON=3600` sets the length of the simulated model time. It is not wall-clock seconds. |

**Warning:** the deploy step clears the run folder. Edit data files in the **repo**, then redeploy.

---

## 5. Build and deploy

Close all running CoinTossX windows before deploying. Windows will not overwrite files that are open.

```powershell
cd C:\Projects\CoinTossX
.\gradlew.bat -Penv=windows build -x test
.\gradlew.bat -Penv=windows clean installDist bootWar copyResourcesToInstallDir copyToDeploy deployLocal
```

Check the Eclipse Collections version inside the web app. It should be `11.1.0`:

```powershell
tar -tf C:\Projects\CoinTossX-run\Web\Web-1.0.war | Select-String "eclipse-collections"
```

Expected: `build` ends with `BUILD SUCCESSFUL` (warnings are normal). `deployLocal` ends with `BUILD SUCCESSFUL`.

---

## 6. Changes made to the repo (summary)

| # | File | Change | Reason |
|---|---|---|---|
| 1 | `windows.properties` | Linux paths replaced with `C:/Projects/CoinTossX-run/...` | Linux paths do not exist on Windows |
| 2 | `build.gradle` (root, `allprojects`) | `resolutionStrategy` forcing `com.google.guava:guava:28.2-jre` | `mapdb 3.0.8` asks for Guava `[15.0,)`, which resolved to 33.8.0. Gradle 6.7.1 cannot choose a variant from it, so `:Common:compileJava` failed. |
| 3 | `build.gradle` (same block) | Force `eclipse-collections`, `-api` and `-forkjoin` to `11.1.0` | The web app bundled 13.0.0, which is compiled for Java 17 (class version 61). Java 8 accepts up to 52, so `Web` failed at start-up in `OffHeapStorage`. The build passed because this only fails at run time. |

These are the only code-level changes. They are fixes to the build, not to CoinTossX's behaviour. They are committed on `windows-setup` in the fork, with LF line endings.

**Why the pins are needed:** both dependencies were declared with open version ranges or left unpinned, so Maven delivered the newest release. The newest releases target newer Java or newer Gradle than this project uses.

---

## 7. Start-up

The processes must start in this order. Each one runs in **its own PowerShell window**, because each window stays busy while its process runs. Use a separate free window to check the process list.

| Order | Component | Command (run from `C:\Projects\CoinTossX-run\scripts`) | Output |
|---|---|---|---|
| 1 | Socket driver (main) | `.\startLowLatencyDriver.bat` | `Socket.log` (stays empty, which is normal) |
| 2 | Trading gateway driver | `.\startTGDriver.bat` | `TGSocket.log` |
| 3 | Market data gateway driver | `.\startMDGDriver.bat` | `MDGSocket.log` |
| 4 | Web driver | `cd ..\Socket\bin` then `.\Socket.bat C:/Projects/CoinTossX-run/aeron/web` | **Manual** (see 7.1) |
| 5 | Native gateway | `.\startNativeGateway.bat` | `NativeGateway.log` |
| 6 | Matching engine | `.\startMatchingEngine.bat` | `MatchingEngine.log` |
| 7 | Market data gateway | `.\startMarketDataGateway.bat` | Windows Firewall prompt may appear. Allow on Private networks. |
| 8 | WebEventListener | `cd ..\WebEventListener\bin` then `.\WebEventListener.bat` | **Manual** (see 7.1). Creates `data\webStorage.db`. |
| 9 | Web app | `.\startWeb.bat` | `Web.log`. Open `http://localhost:8080` once running. |

### 7.1 Gaps in `startAll.bat`

The Linux `startAll.sh` starts a web driver and `WebEventListener`. The Windows `startAll.bat` does not, and there are no `.bat` launchers for them in `scripts\`. Start steps 4 and 8 by hand, in that order, before step 9. Step 9 fails without `data\webStorage.db`.

### 7.2 Check each process

In a free window:

```powershell
Get-CimInstance Win32_Process -Filter "Name='java.exe'" | Select-Object ProcessId, CommandLine | Format-List
```

Each component shows its main class in its command line:

| Component | Text to look for |
|---|---|
| Socket driver | `aeron.LowLatencyMediaDriver` ending `\aeron` |
| TG driver | `aeron.LowLatencyMediaDriver` ending `\tradingGateway` |
| MDG driver | `aeron.LowLatencyMediaDriver` ending `\marketDataGateway` |
| Web driver | `aeron.LowLatencyMediaDriver` ending `\web` |
| Native gateway | `gateway.incoming.NativeGateway` |
| Matching engine | `engine.MatchingEngine` |
| Market data gateway | `gateway.marketdata.NativeMarketDataGateway` |
| WebEventListener | `WebEventListener` |
| Web app | `Web-1.0.war` |

The Gradle daemon (`GradleDaemon`) is harmless and can stay.

---

## 8. Architecture and ports

| Component | Role | Ports (from `windows.properties`) |
|---|---|---|
| Media drivers (Aeron) | Carry messages between components on the same machine. One per messaging group. | Shared-memory directories under `aeron\` |
| Native gateway | Receives client orders and passes them to the matching engine | Subscribes `udp://localhost:5571`. Publishes `udp://localhost:5569`. |
| Matching engine | Matches orders and publishes trades and market data | Receives `5569`. Publishes to the trading gateway on `5571`, and to market data on `5700`. Admin on `5580`. |
| Market data gateway | Passes market data to clients | Receives `5700`. Publishes to multicast `224.0.1.1:40456`. |
| Clients (for example client 1, from `clientData.csv`) | Send orders and receive data | Client 1 output ports `5001` and `5003` were observed in logs. |
| Web app | Browser interface and admin | Logs in as client 999 on `udp://localhost:9000` (output `9001`). Web events on `5701` and `5572`. |
| WebEventListener | Writes events to `data\webStorage.db` for the web app to read | **[Unverified]** exact ports |

**Message flow for one order:** simulator or client, then native gateway, then matching engine. Trades and prices go from the matching engine to the market data gateway, which publishes them to the multicast address.

---

## 9. Running a simulation (terminal)

From `C:\Projects\CoinTossX-run\scripts`, in its own window:

```powershell
.\startHawkesSimulation.bat 1 1
```

The arguments are client ID and stock ID. Output goes to `C:\Projects\CoinTossX-run\HawkesSimulation_1_1.log`. A simulation run is confirmed to:
- connect to the output ports and the multicast feed,
- log in three times,
- wait for initialisation (bid and offer prices from the exchange),
- send a start message, and report the batch size (about 110,000 events for a full horizon).

Orders then appear in the order book and trades in the trades table of the web app. Check the STKA (or chosen stock) view.

---

## 10. Stopping and results

### 10.1 Stop a simulation

- **Terminal run:** press **Ctrl+C**, then **Y** in that same window. The `Terminate batch job (Y/N)?` text is the standard Windows prompt for Ctrl+C in a `.bat` file. The simulator has no keyboard input.
- **Web Stop button:** sends the `SimulationComplete` message to the matching engine, and saves order and trade CSVs. See 10.2.
- **`stopHawkesSimulation.sh`:** Linux only. There is no Windows equivalent.

### 10.2 Where results are written

Results are written **only** when the matching engine receives a `SimulationComplete` message. The only component that sends it is the web app's Stop button. Killing a simulator does not send it.

| Output | Location | Written by | When |
|---|---|---|---|
| Order and trade CSVs | `data\TestResults_<date time>\` (`LOBOrders_<stock>.csv` and similar) | Web app (`saveLOBDataToFile`) | Web Stop button |
| Latency histogram | `data\hdrLatency.txt` | Matching engine (`HDRData.storeHDRStats`) | `SimulationComplete` |
| Storage file | `data\webStorage.db` | WebEventListener (write), web app (read) | Continuously while running |
| Simulation log | `HawkesSimulation_<client>_<stock>.log` in the run folder | Simulator | While running. Overwritten on each run. |
| Component logs | `NativeGateway.log`, `MatchingEngine.log`, `Web.log`, and others, in the run folder | Each component | While running |

**Results in this run:** orders and trades were confirmed in the web app. No `TestResults_` folder or `hdrLatency.txt` had been written, because no Stop was sent.

---

## 11. Troubleshooting (problems met, and fixes)

| Symptom | Cause | Fix |
|---|---|---|
| `java` not recognised | JDK not on `Path` | Set `JAVA_HOME` and add `%JAVA_HOME%\bin` to `Path`. Open a new PowerShell. |
| `javac` not recognised, `java` works | `Path` points to the JRE shim (`java8path`) and `JAVA_HOME` is wrong | Correct `JAVA_HOME` to the JDK folder. Put `%JAVA_HOME%\bin` above `java8path`. |
| Build: `Could not resolve com.google.guava:guava:[15.0,)` | Newest Guava chosen | Pin Guava (Section 6, change 2) |
| Web app: `UnsupportedClassVersionError ... class file version 61.0` | Eclipse Collections 13 bundled | Pin to 11.1.0 (Section 6, change 3) |
| Web app: `Web client is unable to login to Trading Gateway` | Wrapper message. The real cause was `Caused by: FileNotFoundException data\webStorage.db`. | Start WebEventListener before the web app (Section 7.1) |
| Web app: `Caused by: ... NotSerializableException: org.mapdb.BTreeMap$values$1` (in `Web.log`) | Wicket is serializing a page that holds a `mapdb` collection | **Open.** See 12. |
| Simulation started from the web Start button creates no log | The Start button's Wicket action fails before running the script (same serialization error) | **Open.** Use the terminal instead. |
| `Terminate batch job (Y/N)?` in a simulation window | Ctrl+C was pressed in a `.bat` window | Answer **Y** to stop, in that window. |
| Firewall prompt when starting the market data gateway | Java needs inbound access | Allow on Private networks only. Check: `Get-NetFirewallApplicationFilter -Program "*java.exe" \| Get-NetFirewallRule`. |

---

## 12. Open items

1. **Web Start button does not launch simulations.** Wicket fails to serialize a `mapdb` collection (`org.mapdb.BTreeMap$values$1`) on a page. Not yet traced to a page class. Terminal runs are unaffected.
2. **Web Stop button on Windows.** Its stop-simulation path calls `stopHawkesSimulation.bat` for warm-up only. For a normal stop, it sends `SimulationComplete`. Whether this completes cleanly on Windows is **[Unverified]**. The Hawkes process itself must still be stopped with Ctrl+C.
3. **No clean stop from outside the web app.** Writing a notebook that sends `SimulationComplete` would need Java interop. Planned for later.
4. **Simulator end condition.** Not confirmed whether the simulator exits on its own after `HORIZON` events. **[Unverified]**.
5. **`MEDIA_DRIVER_DIR` on Windows.** Works for this run, but the README and `deploy_local.gradle` (which deletes `/dev/shm/aeron`) assume Linux. **[Unverified]** beyond this run.
6. **Hardware.** The README's recommended specification was not checked on this machine.

---

## 13. Verification status

| Area | Status |
|---|---|
| Java 8 install and environment | Verified (`java`, `javac`, `JAVA_HOME`) |
| Build and deploy | Verified (after the two pins) |
| Driver and component start-up, all processes | Verified (process check) |
| Web app and stock list | Verified (screenshot) |
| Simulator connects, logs in, sends orders | Verified (log and web view) |
| Trades appear in the web app | Verified (web view) |
| Results files (TestResults, hdrLatency.txt) | **Not yet produced** |
| Web Start and Stop buttons | **Not working** (Start). Stop **[Unverified]**. |
| Multi-client and auction sessions | **Not tested** |
