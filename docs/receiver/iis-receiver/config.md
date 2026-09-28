Prerequisites

1. A Windows machine with the IIS role installed and running. IIS's default website counts as the "server", so you don't have to build anything.
2. IDOT running on that same machine, since performance counters can only be read locally.
3. Some traffic to IIS so the numbers change.


Install IIS (PowerShell as Administrator)

1. Check which Windows edition you have
(Get-CimInstance Win32_OperatingSystem).Caption

2a. On Windows Server (2016, 2019, 2022 or 2025):
Install-WindowsFeature Web-Server -IncludeManagementTools

2b. On Windows 10 or 11:
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole, IIS-WebServer, IIS-ManagementConsole -All
If it asks you to restart, restart.

3. Check that it's running
Get-Service W3SVC                                    # Status should be Running
Start-Service W3SVC                                  # only if it's Stopped
Invoke-WebRequest http://localhost -UseBasicParsing  # should return StatusCode 200 (the IIS welcome page)

If Get-Service W3SVC says "Cannot find any service with service name 'W3SVC'", IIS isn't installed yet. W3SVC is the service IIS installs, so it doesn't exist until IIS is added.

If step 2 looks stuck, open a second PowerShell window as Administrator:
Get-Process TiWorker, TrustedInstaller -ErrorAction SilentlyContinue | Select Name, CPU, Id
Get-WindowsFeature Web-Server
- If TiWorker is there and its CPU number goes up when you run the command again, it's still installing. Wait.
- If Get-WindowsFeature shows Install State: Installed, it's finished and the first window will catch up.
- InstallPending means it's done but needs a restart.


[Optional] Generate IIS traffic if you don't have any yet

1..200 | % { Invoke-WebRequest http://localhost -UseBasicParsing | Out-Null }


[Important] Collector configuration to add the IIS receiver

1. Under receivers: (Added the iis Receiver)

  # IIS receiver - collects Internet Information Services metrics (IIS must be installed on this host)
  iis:
    collection_interval: 30s


2. Under service.pipelines: (Added iis to the metrics/collector Pipeline)
    # Metrics data pipeline for system metrics
    metrics/collector:
      receivers: [hostmetrics, iis]                   # Collect metrics from host
      processors: [resource/host, batch]              # Add collector identity
      exporters: [otlphttp/exporter]                  # Send metrics to Instana backend


The config.yaml is in the IDOT install folder (C:\Program Files\Instana\instana-collector). Restart IDOT after editing it so the changes take effect.


Start / stop IDOT (PowerShell as Administrator)

1. Find the IDOT service
Get-Service | Where-Object { $_.DisplayName -like "*Instana*" -or $_.Name -like "*instana*" }

2. Start, stop or restart it (replace <service-name> with the Name from step 1)
Start-Service <service-name>
Stop-Service <service-name>
Restart-Service <service-name>                       # use this after editing config.yaml

3. Check that it's running
Get-Service <service-name>                           # Status should be Running

If the service stops right after starting, the config.yaml usually has a YAML error (for example, wrong indentation under receivers: or service.pipelines:). Check it, fix it, and start the service again.



Check the IDOT logs in PowerShell

1. Find the IDOT log files
Get-ChildItem "C:\Program Files\Instana\instana-collector" -Recurse -Filter *.log | Sort-Object LastWriteTime -Descending | Select FullName, LastWriteTime

2. Follow the newest log live (replace <log-file> with a FullName from step 1)
Get-Content "<log-file>" -Tail 50 -Wait

3. Look for IIS or error lines only
Select-String -Path "<log-file>" -Pattern "iis", "error" | Select -Last 20

4. If the service won't start at all, check the Windows service events
Get-WinEvent -LogName System -MaxEvents 50 | Where-Object { $_.ProviderName -eq "Service Control Manager" -and $_.Message -like "*Instana*" } | Format-List TimeCreated, Message

5. To see the output directly, stop the service and run the collector in the foreground (replace <collector>.exe with the exe in the bin folder)
Stop-Service <service-name>
cd "C:\Program Files\Instana\instana-collector\bin"
.\<collector>.exe --config "<path-to>\config.yaml"
Press Ctrl+C to stop it, then Start-Service <service-name> again.
With the debug exporter added to the metrics/collector pipeline (exporters: [otlphttp/exporter, debug]), the console prints each IIS metric (iis.request.count, iis.connection.active and so on) as it is collected. Remove debug again when you're done.


Check the IIS entity on the Instana UI

1. Generate some IIS traffic (see above) and wait a minute or two. The iis receiver collects every 30s.
2. In the Instana UI, go to Infrastructure and search for the host name of the Windows machine.
3. Open the host. The IIS entity shows up with the host's other entities.
4. Open the IIS entity to see its metrics, such as requests, connections, network I/O, threads and uptime. The request numbers should go up after you send more traffic.
5. If the IIS entity doesn't show up, check that W3SVC is running, that IDOT is running, and that iis is listed in receivers: under the metrics/collector pipeline.
