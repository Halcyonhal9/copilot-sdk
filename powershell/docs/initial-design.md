# CopilotPS - PowerShell Wrapper for GitHub Copilot SDK

## 1. Goals

- Expose the full surface of the .NET `GitHub.Copilot.SDK` library as PowerShell cmdlets.
- Write as little code as possible: thin pass-through to SDK, zero custom business logic.
- Run on Windows, macOS, and Ubuntu via PowerShell 7.6+ (.NET 9).
- Default to streaming output and interactive permission prompts.
- Design for a future `-NoWait` async pattern without requiring a rewrite.

## 2. Technology Choices

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Module type | **C# binary cmdlet** (.dll) | Type-safe, direct SDK references, no reflection hacks. Fully cross-platform on .NET 9 — no platform-specific code needed. |
| Target framework | `net9.0` | PowerShell 7.5/7.6 runs on .NET 9. |
| SDK reference | `GitHub.Copilot.SDK` NuGet 0.1.x | The official .NET SDK. |
| PS SDK reference | `System.Management.Automation` 7.5.x | `PrivateAssets="all"` — not shipped with the module. |
| Naming convention | `Verb-CopilotNoun` | Standard PowerShell verb-noun, `Copilot` prefix on all nouns. |

### Cross-Platform Notes

No platform-specific code is required. The Copilot SDK manages CLI process lifecycle cross-platform internally. `Console.ReadLine()` (used for interactive prompts from background threads) works identically on all three platforms in pwsh 7+.

## 3. Project Layout

```
powershell/
├── src/
│   ├── CopilotPS.csproj        # Project file
│   ├── ModuleState.cs          # Singleton state + permission/input handlers + module cleanup
│   ├── ClientCmdlets.cs        # New-CopilotClient, Stop-CopilotClient, Test-CopilotConnection
│   ├── SessionCmdlets.cs       # New/Resume/Get/Remove/Close-CopilotSession
│   └── MessageCmdlets.cs       # Send-CopilotMessage, Get-CopilotMessage
├── CopilotPS.psd1              # Module manifest (copied to output on build)
└── build.ps1                   # Convenience: dotnet publish + tells you the import path
```

**Five C# files, one manifest, one build script.** That's the entire module.

## 4. Module State (Singleton Pattern)

```
ModuleState  (internal static class)
├── client          : CopilotClient?      ← the default client
├── currentSession  : CopilotSession?     ← the default session
├── RequireClient(explicit?)              ← returns explicit ?? client ?? throw
├── RequireSession(explicit?)             ← returns explicit ?? currentSession ?? throw
└── CleanupAsync()                        ← disposes session + stops client
```

Every cmdlet that needs a client or session accepts an optional `-Client` / `-Session` parameter. If omitted, the module-scoped singleton is used. This gives a simple `Connect-AzAccount`-style UX for interactive use while still supporting multi-client scripts.

An `IModuleAssemblyCleanup` implementation calls `ModuleState.CleanupAsync()` when the module is removed, preventing orphaned CLI processes.

### Session Persistence

The SDK persists session state to `~/.copilot/session-state/{sessionId}/` automatically. **We store no additional local state.** Users list sessions via `Get-CopilotSession` (which calls `ListSessionsAsync`) and resume by session ID. If users want friendly names, they can provide their own `-SessionId` at creation time (the SDK supports this).

## 5. Cmdlet Inventory

### 5.1 Client Lifecycle

#### `New-CopilotClient`

Creates a `CopilotClient`, calls `StartAsync()`, stores it as the module default, and writes it to the pipeline.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-GitHubToken` | string | — | `CopilotClientOptions.GithubToken` |
| `-CliPath` | string | — | `CopilotClientOptions.CliPath` |
| `-CliUrl` | string | — | `CopilotClientOptions.CliUrl` (also sets `UseStdio = false`) |
| `-LogLevel` | string | `"info"` | `CopilotClientOptions.LogLevel` |

**Output:** `CopilotClient`

#### `Stop-CopilotClient`

Stops and disposes the client. Clears module state if it was the default.

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `-Client` | CopilotClient | module default | Explicit override |
| `-Force` | switch | — | Calls `ForceStopAsync` instead of `StopAsync` |

**Output:** none

#### `Test-CopilotConnection`

Pings the CLI server.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Client` | CopilotClient | module default | |
| `-Message` | string | — | `PingAsync(message)` |

**Output:** `PingResponse` (Message, Timestamp, ProtocolVersion)

### 5.2 Session Lifecycle

#### `New-CopilotSession`

Creates a session, registers permission/input handlers, stores it as the module default.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Client` | CopilotClient | module default | |
| `-SessionId` | string | (SDK generates) | `SessionConfig.SessionId` |
| `-Model` | string | — | `SessionConfig.Model` |
| `-SystemMessage` | string | — | `SessionConfig.SystemMessage` |
| `-ReasoningEffort` | string | — | `SessionConfig.ReasoningEffort` |
| `-AutoApprove` | switch | — | Uses auto-approve handler instead of interactive |
| `-InfiniteSessions` | switch | — | `SessionConfig.InfiniteSessions` |
| `-WorkingDirectory` | string | — | `SessionConfig.WorkingDirectory` |
| `-AvailableTools` | string[] | — | `SessionConfig.AvailableTools` |
| `-ExcludedTools` | string[] | — | `SessionConfig.ExcludedTools` |

Streaming is always enabled (`SessionConfig.Streaming = true`).

Permission handler: interactive `Console.ReadLine` prompt by default; `-AutoApprove` overrides to approve-all.

User input handler: always registered, uses interactive `Console.ReadLine`.

**Output:** `CopilotSession`

#### `Resume-CopilotSession`

Resumes an existing session by ID. Stores it as the module default.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-SessionId` | string | **required** | `ResumeSessionAsync(sessionId, ...)` |
| `-Client` | CopilotClient | module default | |
| `-Model` | string | — | `ResumeSessionConfig.Model` |
| `-AutoApprove` | switch | — | Permission handler override |
| `-SystemMessage` | string | — | `ResumeSessionConfig.SystemMessage` |
| `-ReasoningEffort` | string | — | `ResumeSessionConfig.ReasoningEffort` |
| `-WorkingDirectory` | string | — | `ResumeSessionConfig.WorkingDirectory` |

**Output:** `CopilotSession`

#### `Get-CopilotSession`

Lists all sessions known to the CLI server.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Client` | CopilotClient | module default | `ListSessionsAsync()` |

**Output:** `List<SessionMetadata>` (SessionId, StartTime, ModifiedTime, Summary, IsRemote, Context)

#### `Remove-CopilotSession`

Permanently deletes a session and its persisted state.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-SessionId` | string | **required** | `DeleteSessionAsync(sessionId)` |
| `-Client` | CopilotClient | module default | |

Supports `ShouldProcess` for `-WhatIf` / `-Confirm`.

**Output:** none

#### `Close-CopilotSession`

Disposes the session object (closes the connection) but preserves on-disk state so it can be resumed later.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Session` | CopilotSession | module default | `DisposeAsync()` |

Clears `ModuleState.currentSession` if it was the default.

**Output:** none

### 5.3 Messaging

#### `Send-CopilotMessage`

Sends a prompt and blocks until the session goes idle. Streams assistant output to the host in real time. Returns a result object to the pipeline.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Prompt` | string | **required**, position 0 | `MessageOptions.Prompt` |
| `-Session` | CopilotSession | module default | |
| `-Attachment` | string[] | — | File paths → `UserMessageDataAttachmentsItemFile` |
| `-Timeout` | TimeSpan | 5 minutes | Passed to internal wait logic |

**Streaming behavior:**
1. Before calling `SendAsync`, subscribe to `session.On(handler)`.
2. In the handler (runs on the JSON-RPC thread):
   - `AssistantMessageDeltaEvent` → `Console.Write(delta.Data.DeltaContent)` (real-time streaming to terminal).
   - `AssistantMessageEvent` → accumulate final content.
   - `ToolExecutionStartEvent` → `Console.Error.WriteLine` (shows as verbose-like info; avoids polluting pipeline).
   - `ToolExecutionCompleteEvent` → `Console.Error.WriteLine` with success/failure.
   - `SessionIdleEvent` → signal the blocking wait to complete.
   - `SessionErrorEvent` → signal the blocking wait with an error.
3. After idle, write a result object to the pipeline:

```
CopilotMessageResult
├── MessageId   : string
├── Content     : string     ← full accumulated assistant response
├── SessionId   : string
└── Events      : SessionEvent[]  ← all events received during this send
```

`CopilotMessageResult` is a small POCO defined in `MessageCmdlets.cs`. It wraps the SDK types without adding logic.

**Why `Console.Write` for streaming instead of `WriteObject`/`WriteVerbose`:** PowerShell cmdlet Write* methods can only be called from the pipeline thread. The SDK event handler runs on the JSON-RPC background thread. `Console.Write` is thread-safe and works cross-platform. The final structured result still goes through `WriteObject` on the pipeline thread after the wait completes.

#### `Get-CopilotMessage`

Retrieves conversation history for a session.

| Parameter | Type | Default | Maps to |
|-----------|------|---------|---------|
| `-Session` | CopilotSession | module default | `GetMessagesAsync()` |

**Output:** `IReadOnlyList<SessionEvent>`

## 6. Permission Handling

Two modes, selected per-session at creation/resume time:

### Interactive (default)
A `PermissionHandler` delegate that:
1. Prints `[Permission] {request.Kind}` in yellow via `Console.ForegroundColor`.
2. Dumps any `ExtensionData` key-value pairs for context.
3. Prompts `Allow? (y/n):` via `Console.ReadLine()`.
4. Returns `PermissionRequestResult { Kind = "approve" }` or `Kind = "deny"`.

### Auto-Approve (`-AutoApprove` switch)
Returns `Kind = "approve"` unconditionally. Suitable for unattended scripts where the user accepts all tool executions.

### User Input (agent asks a question)
Always registered. Uses `Console.ReadLine()` with choice display when `request.Choices` is provided.

## 7. Future: Async / Non-Blocking Pattern

The current design blocks the pipeline thread during `Send-CopilotMessage`. The future async design adds:

```powershell
# Future: returns immediately with just a MessageId
$msg = Send-CopilotMessage "Do something complex" -NoWait

# Future: poll or wait for results
$result = Wait-CopilotMessage -MessageId $msg.MessageId
# or
$result = Get-CopilotMessageResult -MessageId $msg.MessageId
```

### What's needed to support this:

1. **`-NoWait` switch on `Send-CopilotMessage`** — calls `SendAsync` (not `SendAndWaitAsync`), immediately returns a `CopilotMessageResult` with only `MessageId` populated.
2. **`Wait-CopilotMessage` cmdlet** — subscribes to events and blocks until idle, returning the full result. Essentially the second half of the current synchronous flow, factored out.
3. **Background event accumulator** — `ModuleState` gains a `ConcurrentDictionary<string, CopilotMessageResult>` keyed by message ID. The event handler populates results in the background. `Get-CopilotMessageResult` reads from this dictionary.

This is why `CopilotMessageResult` includes `MessageId` from day one — it's the correlation key for the async pattern. The current synchronous path is just the special case where we wait immediately.

**No changes to the current cmdlet signatures are needed** — `-NoWait` is additive.

## 8. Build & Usage

### build.ps1

```powershell
#!/usr/bin/env pwsh
dotnet publish src/CopilotPS.csproj -c Release -o out
Write-Host "Import with: Import-Module ./out/CopilotPS.psd1"
```

### CopilotPS.psd1 (module manifest)

```powershell
@{
    RootModule        = 'CopilotPS.dll'
    ModuleVersion     = '0.1.0'
    GUID              = '<generated>'
    Author            = 'GitHub'
    Description       = 'PowerShell wrapper for the GitHub Copilot SDK'
    PowerShellVersion = '7.4'
    CompatiblePSEditions = @('Core')
    CmdletsToExport   = @(
        'New-CopilotClient'
        'Stop-CopilotClient'
        'Test-CopilotConnection'
        'New-CopilotSession'
        'Resume-CopilotSession'
        'Get-CopilotSession'
        'Remove-CopilotSession'
        'Close-CopilotSession'
        'Send-CopilotMessage'
        'Get-CopilotMessage'
    )
    FunctionsToExport = @()
    AliasesToExport   = @()
    VariablesToExport = @()
}
```

### Example Usage

```powershell
Import-Module ./out/CopilotPS.psd1

# Connect
New-CopilotClient

# Verify
Test-CopilotConnection

# Start a session with a memorable ID
New-CopilotSession -SessionId "my-review" -Model "gpt-5"

# Chat (streams to terminal, returns structured result)
$result = Send-CopilotMessage "Analyze the code in src/ for security issues"
$result.Content  # full response text

# History
Get-CopilotMessage | Where-Object { $_ -is [GitHub.Copilot.SDK.AssistantMessageEvent] }

# Later: resume
Close-CopilotSession
Resume-CopilotSession -SessionId "my-review"
Send-CopilotMessage "What did we discuss earlier?"

# List all sessions
Get-CopilotSession | Format-Table SessionId, Summary, ModifiedTime

# Cleanup
Remove-CopilotSession -SessionId "my-review"
Stop-CopilotClient
```

## 9. What's NOT in Scope (v1)

| Feature | Why deferred |
|---------|-------------|
| BYOK `-Provider` parameter | "For now, we'll use GitHub hosted models." Plumbing the `Provider` object into `SessionConfig` is straightforward when needed. |
| Custom tools | Requires defining `AIFunction` objects — complex UX for PowerShell. Defer until there's a clear need. |
| Session hooks | The SDK's `SessionHooks` type requires delegates. Could be exposed later as script blocks. |
| MCP servers / custom agents | Pass-through config objects. Easy to add as `-McpServers` / `-CustomAgents` hashtable params. |
| `-NoWait` async pattern | Designed for (see section 7) but not implemented in v1. |
| Tab completion | Argument completers for `-Model`, `-SessionId` etc. Nice-to-have, add later. |
| Format/type data (.ps1xml) | Custom formatting for `SessionMetadata`, `CopilotMessageResult`. Polish item. |

## 10. File Inventory (5 C# files)

| File | Contents | Approximate lines |
|------|----------|-------------------|
| `ModuleState.cs` | `ModuleState` static class, `PermissionHandlers`, `UserInputHandlers`, `ModuleCleanup` | ~80 |
| `ClientCmdlets.cs` | `NewCopilotClientCmdlet`, `StopCopilotClientCmdlet`, `TestCopilotConnectionCmdlet` | ~70 |
| `SessionCmdlets.cs` | `NewCopilotSessionCmdlet`, `ResumeCopilotSessionCmdlet`, `GetCopilotSessionCmdlet`, `RemoveCopilotSessionCmdlet`, `CloseCopilotSessionCmdlet` | ~120 |
| `MessageCmdlets.cs` | `SendCopilotMessageCmdlet`, `GetCopilotMessageCmdlet`, `CopilotMessageResult` | ~100 |
| `CopilotPS.csproj` | Project file | ~20 |
| **Total** | | **~390** |
