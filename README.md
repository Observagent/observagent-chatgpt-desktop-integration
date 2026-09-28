# ChatGPT desktop app integration

Configure all three supported telemetry signals: logs, metrics, and traces. This guide applies the documented exporter settings to local Codex runs in the ChatGPT desktop app. It does not establish automatic export for ordinary ChatGPT chats, Voice, or hosted ChatGPT Work sessions, or promise that every internal event is exported. Check the mode and execution host before changing settings. Official sources were reviewed on September 20, 2026.

## 1. Check which experience you use

The desktop app provides ChatGPT and Codex experiences; ChatGPT also has Chat and Work modes. See [the desktop app guide](https://learn.chatgpt.com/docs/app). The [OpenTelemetry documentation](https://learn.chatgpt.com/docs/config-file/config-advanced#observability-and-telemetry) describes local Codex run export. This walkthrough applies to that runtime.

- Local Codex task in the desktop app: follow the steps below.
- Ordinary ChatGPT chat: a native OTLP export setting was not established by the official documentation reviewed. Do not assume this config captures those conversations.
- ChatGPT Work: local and hosted execution can differ; export coverage was not verified here. Confirm the task's runtime and administrator guidance before applying the Codex setup.
- Existing Codex CLI integration: the desktop app can share the same local Codex configuration. You may already be configured; avoid replacing its exporter just to create a new label.

This guide does not require an Ollama proxy, Python packages, an OpenAI API key, or an MCP server. It configures the runtime's own exporters. Observagent's ingestion endpoints accept OTLP; they are not MCP endpoints. Adding them to the app's MCP server list will not create a telemetry integration.

## 2. Prepare an Observagent project

Sign in to Observagent. Open Settings / Projects and create or choose a project. Enable Logs, Metrics, and Traces, enable monitoring, and use Connect source to generate a project API key with telemetry:write permission. You can call the project ChatGPT desktop, but that name does not change the event source emitted by the runtime.

Generating a new key replaces previous keys for that project only. Other projects, including Ollama, are unaffected. New data follows the key; old data stays where it was stored. If your Codex CLI and desktop app use the same config.toml, changing its exporter key can reroute both clients. This configuration does not promise separate desktop and CLI routing.

If Connect source reports that project key creation is unavailable, contact your Observagent administrator or support channel. This is a server-side account or deployment issue, not a desktop client setting.

## 3. Locate and back up the active configuration

On native Windows, the desktop app uses the Windows Codex home directory, normally %USERPROFILE%\.codex. WSL can use a different home; see [Windows app configuration](https://learn.chatgpt.com/docs/windows/windows-app#share-config-auth-and-sessions-with-wsl). Make sure you edit the configuration of the agent host actually running the task.

Run this in local PowerShell. It prints a path, backs up an existing file, and opens the editor; it does not print your config or keys. A CODEX_HOME value in this terminal must match the one used by the desktop app.

```powershell
$configDir = Join-Path $HOME ".codex"
if ($env:CODEX_HOME) { $configDir = $env:CODEX_HOME }
New-Item -ItemType Directory -Force -Path $configDir | Out-Null
$configPath = Join-Path $configDir "config.toml"
Write-Host "Configuration:" $configPath
if (Test-Path -LiteralPath $configPath) {
    $backupPath = "$configPath.backup-$(Get-Date -Format yyyyMMdd-HHmmss)"
    Copy-Item -LiteralPath $configPath -Destination $backupPath
}
notepad.exe $configPath
```

Configuration layers can override one another. Trusted project settings, profiles, and runtime overrides may supersede user settings; see [configuration precedence](https://learn.chatgpt.com/docs/config-file/config-basic#configuration-precedence). Do not overwrite the whole file. Keep existing model, plugin, MCP, permission, and other settings.

## 4. Configure logs, metrics, and traces

Merge the following TOML into the active config.toml. Replace the endpoint for your deployment and replace YOUR_OBSERVAGENT_PROJECT_KEY with the complete project key. Keep Bearer followed by one space. The endpoint below is the Observagent deployment used in these guides.

```toml
[otel]
environment = "desktop-local"
log_user_prompt = false

[otel.exporter.otlp-http]
endpoint = "https://dashboard.observagent.cloud/v1/logs"
protocol = "binary"

[otel.exporter.otlp-http.headers]
Authorization = "Bearer YOUR_OBSERVAGENT_PROJECT_KEY"

[otel.metrics_exporter.otlp-http]
endpoint = "https://dashboard.observagent.cloud/v1/metrics"
protocol = "binary"

[otel.metrics_exporter.otlp-http.headers]
Authorization = "Bearer YOUR_OBSERVAGENT_PROJECT_KEY"

[otel.trace_exporter.otlp-http]
endpoint = "https://dashboard.observagent.cloud/v1/traces"
protocol = "binary"

[otel.trace_exporter.otlp-http.headers]
Authorization = "Bearer YOUR_OBSERVAGENT_PROJECT_KEY"
```

The [official configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference) lists separate log, metric, and trace exporters. This template maps them to Observagent's three protobuf HTTP endpoints. Replace the key in all three Authorization entries; do not set just the log header. Signals are independent: receiving a log does not prove that metrics or traces arrived.

If [otel] or its exporter tables already exist, edit them in place. Do not duplicate tables. Remove the existing scalar or inline values for exporter, metrics_exporter, and trace_exporter before using their nested tables above. For example, metrics_exporter = "statsig" conflicts with the new metrics table. Preserve unrelated OTel settings intentionally. Do not paste TOML into PowerShell, an Ollama prompt, or the ChatGPT composer.

Setting OBSERVAGENT_API_KEY in the Ollama proxy terminal does not configure this desktop exporter. This template reads the Authorization value from config.toml. Keep that file and its backups private and outside source control.

The prompt flag keeps raw user prompts out of these logs, but tool results may include output snippets. This is not a complete transcript or a promise that every event contains token counts. Event coverage and fields depend on the runtime version. See [telemetry event coverage](https://learn.chatgpt.com/docs/config-file/config-advanced#what-gets-emitted).

## 5. Restart the app and create a local task

Save the configuration, finish any active work, and fully quit the desktop app. Reopen it so the local runtime loads the new configuration. No proxy window needs to stay open for this integration.

Select Codex, open a small local test folder, and start a new task. Use a harmless prompt such as: List the files in this folder without changing anything, then summarize what you found. Wait for the task to complete and allow a few seconds for export. This tests a local Codex run; a successful test does not establish export coverage for Chat or Work.

## 6. Verify storage in Observagent

Sign in to the Observagent account associated with your key. Open Telemetry Data, choose All sources and All available, select Logs or All, clear old queries, and refresh. Look for recent Codex events such as codex.conversation_starts, codex.api_request, or codex.tool_result. The actual transport may produce SSE or WebSocket events.

Inspect a record's timestamp, model, conversation/session fields, and available runtime metadata. The source may still display Codex, even when you named the project ChatGPT desktop. The UI's source label is derived from event attributes; it is not a separate app-specific project.

Check each signal separately. In Telemetry Data, select Logs for event records and Metrics for metric records. Check the selected project's history for traces, because the main telemetry view currently reads only logs and metrics. Last stored signal alone confirms recent activity, not all three pipelines. An empty connectivity test with count: 0 is not evidence of storage.

Metrics export periodically, so allow time for a reporting interval and finish the task normally. Counts and duration distributions for runtime requests and tools are examples of reported metrics; this is not a guarantee of GPU, operating-system, or every internal model metric. Trace structure depends on the runtime. Token summaries depend on recognized completion attributes; inspect records rather than assuming all signals carry token usage.

## 7. Diagnose connection and configuration issues

If no new records arrive, first confirm that you tested a local Codex task, edited the host's active configuration, and restarted the app. The already-working CLI may be using a different home or a project/profile override.

An optional authentication test can run in PowerShell without changing your desktop config. It uses an empty protobuf batch and stores no records:

```powershell
$backendUrl = (Read-Host "Backend base URL, such as https://dashboard.observagent.cloud").TrimEnd('/')
$projectKey = Read-Host "Paste the Observagent project key"
$headers = @{ Authorization = "Bearer $projectKey" }
try {
    foreach ($signal in @('logs', 'metrics', 'traces')) {
        try {
            $uri = "$backendUrl/v1/$signal"
            $result = Invoke-WebRequest -Uri $uri -Method Post -Headers $headers -ContentType "application/x-protobuf" -Body ([byte[]]@()) -UseBasicParsing
            Write-Host $signal "HTTP status:" $result.StatusCode
            Write-Host "Response:" $result.Content
        } catch {
            Write-Host $signal $_.Exception.Message
            Write-Host $_.ErrorDetails.Message
        }
    }
} finally {
    Remove-Variable projectKey,headers -ErrorAction SilentlyContinue
}
```

Expect HTTP 200 with status ok and count: 0 for each signal. This proves endpoint access and key acceptance, not desktop export. A 200 HTML page indicates incorrect routing. HTTP 401 means an invalid or missing key; 403 can mean missing scope, a paused project, or that signal disabled. A timeout or TLS error requires checking your hostname, network, and certificate.

- The app reports invalid TOML: check duplicate [otel] tables and a conflicting scalar or inline exporter. Restore from your private backup if needed.
- The CLI exports but the desktop does not: verify Windows-native versus WSL/remote host configuration and reopen the app.
- Records go to the old project: update the Authorization key in the exporter config actually in use, not just a PowerShell variable.
- You see Codex rather than ChatGPT desktop: that is expected for this runtime; use project history and timestamps to confirm routing.
- Logs arrive but token totals stay empty: inspect completion-event attributes; not all events and transports carry fields recognized by the dashboard.
- Logs arrive but metrics or traces do not: check all three exporter tables, endpoint suffixes, authorization headers, and project signal toggles. Allow time for metric export and inspect project history for traces.
- Regular ChatGPT chats produce no events: this guide does not verify native OTLP export for ordinary chats. Do not treat the Codex configuration as a universal ChatGPT exporter.

## 8. Rotate the key or stop export

To rotate a project key, replace the Authorization value in all three exporter tables and restart the desktop app. Update any CLI or IDE client sharing that configuration. Keep each endpoint on its corresponding /v1/logs, /v1/metrics, or /v1/traces path.

To stop all three exports configured here, remove their nested tables and headers, then set exporter = "none", metrics_exporter = "none", and trace_exporter = "none" inside the existing [otel] table and restart. Do not leave both scalar settings and nested tables for the same exporter. This does not delete stored records.

## Other ChatGPT experiences

For regular ChatGPT chats and unverified Work runtimes, there is no tested Observagent setup in this guide. An MCP integration can observe calls made to your own tools; it should not be presented as access to every conversation or internal model token count. The [MCP overview](https://learn.chatgpt.com/docs/extend/mcp) describes tool connections, not a universal telemetry feed.

If you control a separate application using the OpenAI API, instrument that application instead. Its requests and usage belong to that application; they do not represent activity from the ChatGPT desktop app. Ask your workspace administrator about supported audit integrations if you need organization-wide records.

## References and scope

- [ChatGPT desktop app](https://learn.chatgpt.com/docs/app): choosing ChatGPT, Work, or Codex.
- [Windows app](https://learn.chatgpt.com/docs/windows/windows-app): native and WSL execution and shared Codex home.
- [Configuration basics](https://learn.chatgpt.com/docs/config-file/config-basic): file locations and precedence.
- [OpenTelemetry setup](https://learn.chatgpt.com/docs/config-file/config-advanced#observability-and-telemetry): Codex export and events.
- [Configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference): exporter fields and authentication headers.

This is a documentation update. It does not modify your active desktop configuration, change a project key, or certify live desktop-to-backend delivery. Follow the setup and verify actual records in your own project.

## License

MIT. See [LICENSE](LICENSE).
