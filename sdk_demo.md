# Copilot SDK Harness Explorer — WPF Implementation Guide

A WPF demo app that hosts the GitHub Copilot SDK and exposes the agent harness settings in the UI: model, system prompt, context management, tools and permissions, agents, skills, MCP servers, and hooks. A live event inspector shows what the harness does.

> **About SDK API names.** The SDK (`GitHub.Copilot.SDK`) is GA but still changes quickly. Type and member names below follow the public docs and samples (`CopilotClient`, `SessionConfig`, `SkillDirectories`, `CustomAgents`, `Hooks.OnPreToolUse`, `InfiniteSessions`, …). All SDK calls go through **one adapter class** (`CopilotSdkAdapter`), so when a name differs in your package version, you fix it in one file. Pin the package version.

---

## 1. Goals and non-goals

**Goals**
- Show that the SDK provides the agent loop, and that the app controls everything *around* it.
- Map every UI setting directly onto `SessionConfig` (or a session RPC call).
- Make the harness visible: tool calls, permission requests, compaction, sub-agents, errors.
- Support side-by-side "same prompt, different setting" demos.

**Non-goals**
- Production auth, multi-user, or a polished design. This is a demo/teaching tool.
- Reimplementing anything the runtime already does (history, compaction, planning).

---

## 2. Tech stack

| Item | Choice | Notes |
|---|---|---|
| Framework | .NET 8 or .NET 10, `net8.0-windows` / `net10.0-windows` | .NET Framework 4.8 works via the netstandard2.0 build, but use modern .NET for the demo |
| UI | WPF, MVVM | |
| MVVM helpers | `CommunityToolkit.Mvvm` | `[ObservableProperty]`, `[RelayCommand]` |
| Hosting/DI | `Microsoft.Extensions.Hosting` | Client lifetime, services, logging |
| SDK | `GitHub.Copilot.SDK` (pinned) | Requires the Copilot CLI installed and logged in (or BYOK) |
| Markdown in chat | `Markdig.Wpf` (optional) | Render assistant messages |
| JSON viewer | `TextBox` with pretty-printed JSON | Enough for the event inspector |
| Persistence | JSON files in `%LOCALAPPDATA%\HarnessExplorer\` | Profiles, agent definitions, session index |

Prerequisites on the demo machine:
1. Copilot CLI installed, and `copilot` run once with `/login` (or the BYOK env/config ready).
2. If the seat comes from your org: the **Copilot CLI policy enabled** for the org.
3. A sample repo to use as the working directory (e.g. a small C# or C project).

---

## 3. Solution structure

```
HarnessExplorer.sln
└─ src/HarnessExplorer/
   ├─ App.xaml / App.xaml.cs              // Generic Host bootstrap
   ├─ Core/
   │  ├─ CopilotSdkAdapter.cs             // ONLY place that touches SDK types
   │  ├─ IHarnessService.cs               // App-facing API (sessions, send, abort, rpc)
   │  ├─ HarnessService.cs
   │  ├─ HarnessEvent.cs                  // App-level event record (normalized)
   │  ├─ SessionConfigBuilder.cs          // HarnessSettings -> SessionConfig
   │  ├─ Permissions/
   │  │  ├─ PermissionPolicy.cs           // ApproveAll / Ask / DenyAll / Rules
   │  │  └─ IPermissionPrompt.cs          // UI callback for "Ask"
   │  ├─ Hooks/
   │  │  ├─ HookPipeline.cs               // Composes enabled hook behaviours
   │  │  ├─ LogToolCallsHook.cs
   │  │  ├─ RestrictToWorkingDirHook.cs
   │  │  ├─ SuppressVerboseOutputHook.cs
   │  │  └─ TopicFilterHook.cs
   │  ├─ Tools/
   │  │  ├─ CustomToolRegistry.cs         // Demo C# tools exposed to the model
   │  │  └─ DemoTools.cs
   │  └─ Agents/
   │     ├─ AgentDefinition.cs
   │     └─ AgentMarkdownLoader.cs        // Import *.agent.md (YAML frontmatter + body)
   ├─ Settings/
   │  ├─ HarnessSettings.cs               // Plain model of all UI settings
   │  └─ ProfileStore.cs                  // Save/load named profiles (JSON)
   ├─ ViewModels/
   │  ├─ MainViewModel.cs
   │  ├─ SessionsViewModel.cs
   │  ├─ ChatViewModel.cs
   │  ├─ ChatItemViewModel.cs             // Message / ToolCall / Permission / System items
   │  ├─ InspectorViewModel.cs
   │  └─ Settings/
   │     ├─ ModelSettingsViewModel.cs
   │     ├─ SystemPromptViewModel.cs
   │     ├─ ContextSettingsViewModel.cs
   │     ├─ ToolsSettingsViewModel.cs
   │     ├─ AgentsSettingsViewModel.cs
   │     ├─ SkillsSettingsViewModel.cs
   │     ├─ McpSettingsViewModel.cs
   │     └─ HooksSettingsViewModel.cs
   ├─ Views/
   │  ├─ MainWindow.xaml
   │  ├─ SessionsPanel.xaml
   │  ├─ ChatPanel.xaml
   │  ├─ InspectorPanel.xaml
   │  ├─ Settings/*.xaml                  // one UserControl per tab
   │  └─ Dialogs/AgentEditorDialog.xaml
   └─ Resources/Styles.xaml, DataTemplates.xaml
```

---

## 4. Window layout

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Toolbar: [Profile ▼] [Save profile] [Compare mode ☐]   CLI ● user@org     │
├──────────────┬──────────────────────────────────┬─────────────────────────┤
│ SESSIONS     │ CHAT                             │ SETTINGS (TabControl)   │
│ ┌──────────┐ │ [Agent: harness-generator] [Plan]│ Model | Prompt | Context│
│ │ session 1│ │                                  │ Tools | Agents | Skills │
│ │ session 2│ │  user: ...                       │ MCP   | Hooks           │
│ └──────────┘ │  ▸ tool: view src/Parser.cs      │                         │
│ [New][Resume]│  ▸ PERMISSION: write tests/...   │  (tab content)          │
│ [Fork][Del]  │     [Approve once][Deny]         │                         │
│              │  assistant: ... (streaming)      │  ⚠ Changes apply to new │
│              │ ┌──────────────────────────────┐ │    sessions [Recreate]  │
│              │ │ prompt                       │ │                         │
│              │ └──────────────────────────────┘ │                         │
│              │ [📎][Send][■ Stop]  ctx ▓▓▓░ 62% │                         │
├──────────────┴──────────────────────────────────┴─────────────────────────┤
│ EVENT INSPECTOR  [filter ▼] [search] [Export]   turns 4 · tools 11 · cmp 1│
│ time      type                    summary                    │ raw JSON   │
└───────────────────────────────────────────────────────────────────────────┘
```

`MainWindow.xaml` skeleton:

```xml
<Window x:Class="HarnessExplorer.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:v="clr-namespace:HarnessExplorer.Views"
        Title="Copilot SDK Harness Explorer" Width="1600" Height="950">
  <DockPanel>
    <ToolBar DockPanel.Dock="Top">
      <ComboBox Width="180" ItemsSource="{Binding Profiles}" SelectedItem="{Binding SelectedProfile}"/>
      <Button Content="Save profile" Command="{Binding SaveProfileCommand}"/>
      <Separator/>
      <CheckBox Content="Compare mode" IsChecked="{Binding CompareMode}"/>
      <Separator/>
      <TextBlock Text="{Binding ConnectionStatus}" VerticalAlignment="Center"/>
    </ToolBar>

    <Grid>
      <Grid.RowDefinitions>
        <RowDefinition Height="*"/>
        <RowDefinition Height="5"/>
        <RowDefinition Height="240"/>
      </Grid.RowDefinitions>
      <Grid.ColumnDefinitions>
        <ColumnDefinition Width="240"/>
        <ColumnDefinition Width="5"/>
        <ColumnDefinition Width="*"/>
        <ColumnDefinition Width="5"/>
        <ColumnDefinition Width="420"/>
      </Grid.ColumnDefinitions>

      <v:SessionsPanel Grid.Column="0" DataContext="{Binding Sessions}"/>
      <GridSplitter Grid.Column="1" HorizontalAlignment="Stretch"/>
      <v:ChatPanel Grid.Column="2" DataContext="{Binding Chat}"/>
      <GridSplitter Grid.Column="3" HorizontalAlignment="Stretch"/>
      <TabControl Grid.Column="4">
        <TabItem Header="Model"   Content="{Binding ModelSettings}"/>
        <TabItem Header="Prompt"  Content="{Binding SystemPrompt}"/>
        <TabItem Header="Context" Content="{Binding ContextSettings}"/>
        <TabItem Header="Tools"   Content="{Binding ToolsSettings}"/>
        <TabItem Header="Agents"  Content="{Binding AgentsSettings}"/>
        <TabItem Header="Skills"  Content="{Binding SkillsSettings}"/>
        <TabItem Header="MCP"     Content="{Binding McpSettings}"/>
        <TabItem Header="Hooks"   Content="{Binding HooksSettings}"/>
      </TabControl>

      <GridSplitter Grid.Row="1" Grid.ColumnSpan="5" HorizontalAlignment="Stretch"/>
      <v:InspectorPanel Grid.Row="2" Grid.ColumnSpan="5" DataContext="{Binding Inspector}"/>
    </Grid>
  </DockPanel>
</Window>
```

The settings ViewModels are resolved to views through implicit `DataTemplate`s in `DataTemplates.xaml`:

```xml
<DataTemplate DataType="{x:Type vm:ModelSettingsViewModel}">
  <views:ModelSettingsView/>
</DataTemplate>
<!-- one per settings VM -->
```

---

## 5. Settings model (single source of truth)

All tabs edit one `HarnessSettings` object. It's serializable, so it doubles as a **profile** ("Read-only reviewer", "Full autonomy", "Skills off", …).

```csharp
public sealed class HarnessSettings
{
    // Model tab
    public string Model { get; set; } = "auto";
    public string? ReasoningEffort { get; set; }          // null | low | medium | high
    public AuthMode AuthMode { get; set; } = AuthMode.LoggedInUser;
    public string? GitHubToken { get; set; }              // AuthMode.Token (do not persist in plain text)
    public ByokSettings? Byok { get; set; }               // AuthMode.Byok
    public bool Streaming { get; set; } = true;

    // Prompt tab
    public SystemPromptMode SystemPromptMode { get; set; } = SystemPromptMode.Append;
    public string SystemPrompt { get; set; } = "";

    // Context tab
    public bool InfiniteSessions { get; set; } = true;
    public double BackgroundCompactionThreshold { get; set; } = 0.80;
    public double BufferExhaustionThreshold { get; set; } = 0.95;

    // Tools tab
    public string WorkingDirectory { get; set; } = "";
    public List<ToolToggle> BuiltInTools { get; set; } = new();   // name + enabled
    public List<ToolToggle> CustomTools { get; set; } = new();
    public PermissionPolicyKind PermissionPolicy { get; set; } = PermissionPolicyKind.Ask;
    public bool RuleWritesOnlyInWorkingDir { get; set; } = true;
    public bool RuleDenyShell { get; set; } = false;

    // Agents tab
    public List<AgentDefinition> Agents { get; set; } = new();
    public string? PreselectedAgent { get; set; }
    public bool AllowAgentInference { get; set; } = true;

    // Skills tab
    public List<string> SkillDirectories { get; set; } = new();
    public List<string> DisabledSkills { get; set; } = new();

    // MCP tab
    public List<McpServerDefinition> McpServers { get; set; } = new();

    // Hooks tab
    public bool HookLogToolCalls { get; set; } = true;
    public bool HookRestrictToWorkingDir { get; set; } = true;
    public bool HookSuppressVerboseOutput { get; set; } = false;
    public List<string> VerboseTools { get; set; } = new() { "list_directory", "search_files" };
    public bool HookTopicFilter { get; set; } = false;
    public string TopicFilterKeywords { get; set; } = "harness, test, swc, build, code";
}

public enum AuthMode { LoggedInUser, Token, Byok }
public enum SystemPromptMode { Append, Replace }
public enum PermissionPolicyKind { ApproveAll, Ask, DenyAll, Rules }

public sealed record ToolToggle(string Name, string Description, bool Enabled);
public sealed record ByokSettings(string ProviderType, string BaseUrl, string ApiKeyRef, string? DeploymentOrModel);
public sealed record McpServerDefinition(string Name, McpTransport Transport, string CommandOrUrl,
                                         string[] Args, bool Enabled);
public enum McpTransport { Stdio, Http }
```

**When settings apply.** Almost everything is fixed at session creation. Each settings VM sets `MainViewModel.SettingsDirty = true` on change, and the UI shows **"Changes apply to new sessions — [Recreate session]"**. Exceptions that apply live: permission policy, hooks (they read settings at call time), and manual context actions.

---

## 6. The SDK adapter (the only SDK-aware code)

```csharp
public sealed class CopilotSdkAdapter : IAsyncDisposable
{
    private CopilotClient? _client;

    public async Task StartAsync(HarnessSettings s, CancellationToken ct)
    {
        var options = new CopilotClientOptions();
        if (s.AuthMode == AuthMode.Token)
            options.GitHubToken = s.GitHubToken;            // verify property name
        _client = new CopilotClient(options);
        await _client.StartAsync();
    }

    public Task<IReadOnlyList<string>> ListModelsAsync() =>
        _client!.ListModelsAsync();                          // fills Model ComboBox

    public Task<CopilotSession> CreateSessionAsync(SessionConfig cfg) =>
        _client!.CreateSessionAsync(cfg);

    public Task<CopilotSession> ResumeSessionAsync(string id, ResumeSessionConfig? cfg = null) =>
        _client!.ResumeSessionAsync(id, cfg);

    public Task<IReadOnlyList<SessionInfo>> ListSessionsAsync() =>
        _client!.ListSessionsAsync();

    public Task DeleteSessionAsync(string id) => _client!.DeleteSessionAsync(id);

    public async ValueTask DisposeAsync()
    {
        if (_client is not null) await _client.DisposeAsync();
    }
}
```

Session-level RPCs used by the UI (names from the SDK compatibility table; check the .NET casing):

| UI action | SDK call |
|---|---|
| Send | `session.SendAsync(new MessageOptions { Prompt, Attachments })` |
| Stop | `session.AbortAsync()` |
| Compact now | `session.Rpc.History.CompactAsync()` |
| Truncate from message | `session.Rpc.History.TruncateAsync(eventId)` |
| Fork | `server.Rpc.Sessions.ForkAsync(sessionId, eventId)` |
| Show plan | `session.Rpc.Plan.ReadAsync()` |
| Fleet mode | `session.Rpc.Fleet.StartAsync(objective)` |
| Select agent live | `session.Rpc.Agent.SelectAsync(name)` (experimental) |
| History for redraw | `session.GetMessagesAsync()` / `GetEventsAsync()` |

---

## 7. Building `SessionConfig` from settings

```csharp
public sealed class SessionConfigBuilder(
    CustomToolRegistry tools,
    HookPipeline hooks,
    PermissionPolicy permissions)
{
    public SessionConfig Build(HarnessSettings s)
    {
        var cfg = new SessionConfig
        {
            Model            = s.Model,
            ReasoningEffort  = s.ReasoningEffort,
            Streaming        = s.Streaming,
            WorkingDirectory = s.WorkingDirectory,

            SystemMessage = string.IsNullOrWhiteSpace(s.SystemPrompt) ? null : new SystemMessageConfig
            {
                Mode    = s.SystemPromptMode == SystemPromptMode.Replace ? "replace" : "append",
                Content = s.SystemPrompt
            },

            InfiniteSessions = new InfiniteSessionConfig
            {
                Enabled                        = s.InfiniteSessions,
                BackgroundCompactionThreshold  = s.BackgroundCompactionThreshold,
                BufferExhaustionThreshold      = s.BufferExhaustionThreshold
            },

            // Tools: allow-list of built-ins + our custom C# tools
            AvailableTools = s.BuiltInTools.Where(t => t.Enabled).Select(t => t.Name).ToList(),
            Tools          = tools.GetEnabled(s.CustomTools),

            // Agents & skills
            SkillDirectories = s.SkillDirectories,
            DisabledSkills   = s.DisabledSkills,
            CustomAgents     = s.Agents.Select(a => a.ToSdk(s.AllowAgentInference)).ToList(),
            Agent            = s.PreselectedAgent,

            // MCP
            McpServers = s.McpServers.Where(m => m.Enabled).ToDictionary(m => m.Name, m => m.ToSdk()),

            // Governance
            OnPermissionRequest = permissions.HandleAsync,
            Hooks               = hooks.Build()
        };

        if (s.AuthMode == AuthMode.Byok && s.Byok is not null)
            cfg.Provider = new ProviderConfig
            {
                Type    = s.Byok.ProviderType,                  // "openai" | "azure" | "anthropic"
                BaseUrl = s.Byok.BaseUrl,
                ApiKey  = SecretStore.Resolve(s.Byok.ApiKeyRef)
            };

        return cfg;
    }
}
```

Show a read-only **"Effective config" JSON preview** at the bottom of the settings panel (serialize `cfg` minus secrets). It's very effective in a demo: change a checkbox and the config visibly changes.

---

## 8. Harness service and event normalization

The UI never handles raw SDK events. `HarnessService` subscribes to the session and normalizes events into a `HarnessEvent` stream:

```csharp
public sealed record HarnessEvent(
    DateTimeOffset Time,
    string SessionId,
    HarnessEventKind Kind,
    string Summary,
    string RawJson,
    string? ToolCallId = null,
    string? Text = null,
    double? ContextUsage = null);

public enum HarnessEventKind
{
    UserMessage, AssistantDelta, AssistantMessage,
    ToolStart, ToolEnd, ToolError,
    PermissionRequested, PermissionResolved,
    SubAgentStarted, SubAgentCompleted,
    CompactionStarted, CompactionCompleted,
    PlanUpdated, Idle, Error, Other
}
```

```csharp
public sealed class HarnessService(CopilotSdkAdapter sdk, SessionConfigBuilder builder) : IHarnessService
{
    private readonly Subject<HarnessEvent> _events = new();   // or a Channel<T>
    public IObservable<HarnessEvent> Events => _events;
    public CopilotSession? Current { get; private set; }

    public async Task<string> NewSessionAsync(HarnessSettings s)
    {
        Current = await sdk.CreateSessionAsync(builder.Build(s));
        Current.On(evt => _events.OnNext(Normalize(Current.SessionId, evt)));   // verify subscribe API
        return Current.SessionId;
    }

    public Task SendAsync(string prompt, IEnumerable<string> attachments) =>
        Current!.SendAsync(new MessageOptions
        {
            Prompt = prompt,
            Attachments = attachments.Select(Attachment.FromPath).ToList()
        });

    public Task AbortAsync() => Current!.AbortAsync();

    private static HarnessEvent Normalize(string sid, SessionEvent e) => e.Type switch
    {
        "assistant.message_delta" => new(Now, sid, HarnessEventKind.AssistantDelta, "", Raw(e), Text: e.DeltaContent),
        "assistant.message"       => new(Now, sid, HarnessEventKind.AssistantMessage, Trim(e.Content), Raw(e), Text: e.Content),
        "tool.execution_start"    => new(Now, sid, HarnessEventKind.ToolStart, $"{e.ToolName} {Args(e)}", Raw(e), e.ToolCallId),
        "tool.execution_complete" => new(Now, sid, HarnessEventKind.ToolEnd, e.ToolName, Raw(e), e.ToolCallId),
        "session.compaction_start"    => new(Now, sid, HarnessEventKind.CompactionStarted, "compaction started", Raw(e)),
        "session.compaction_complete" => new(Now, sid, HarnessEventKind.CompactionCompleted, "compaction done", Raw(e)),
        "session.idle"            => new(Now, sid, HarnessEventKind.Idle, "idle", Raw(e)),
        "session.error"           => new(Now, sid, HarnessEventKind.Error, e.Message, Raw(e)),
        _                         => new(Now, sid, HarnessEventKind.Other, e.Type, Raw(e))
    };
}
```

> Event type strings are examples. Log `e.Type` for everything once, then map the ones your SDK version emits. The `Other` bucket keeps unknown events visible in the inspector, so nothing is lost.

**Threading.** SDK callbacks arrive on background threads. Every VM update goes through `Application.Current.Dispatcher.InvokeAsync(...)`. For streaming deltas, batch updates every ~30 ms (a `DispatcherTimer` that flushes a `StringBuilder`) so the UI doesn't stutter.

---

## 9. Panel-by-panel implementation

### 9.1 Sessions panel

| Control | Binding / Command | Behaviour |
|---|---|---|
| `ListBox` sessions | `Sessions`, `SelectedSession` | Name, model, agent, last activity |
| `New` | `NewSessionCommand` | Builds config from current settings, creates session, clears chat |
| `Resume` | `ResumeCommand` | `ResumeSessionAsync(id)`, then redraws history via `GetMessagesAsync()` |
| `Fork` | `ForkCommand` | Forks at the selected chat message (or the end) and opens the fork as a new session |
| `Delete` | `DeleteCommand` | Confirm, then `DeleteSessionAsync` |
| Status block | `ConnectionStatus` | CLI running, auth user, `ping` latency; red if not logged in or policy blocked |

Keep a small local index (`sessions.json`) with display names and the settings profile used, because the SDK's session list won't know your profile names.

### 9.2 Chat panel

**Items** are a polymorphic `ObservableCollection<ChatItemViewModel>`:

| Item VM | Template |
|---|---|
| `UserMessageItem` | Right-aligned bubble |
| `AssistantMessageItem` | Markdown (Markdig.Wpf); `IsStreaming` shows a caret |
| `ToolCallItem` | `Expander`: header `🔧 view src/Parser.cs (120 ms)`; body has args JSON and a result preview; red border on error |
| `PermissionItem` | Card: kind (shell/write/read/url/mcp), target, reason; buttons **Approve once / Approve for session / Deny**; disabled after decision |
| `SystemNoticeItem` | Grey line: "Compaction completed", "Sub-agent `researcher` started", "Aborted by user" |

**Input row**
- `TextBox` (AcceptsReturn, Ctrl+Enter = send).
- Attach button: file or folder picker, shown as chips above the box.
- `Send` (disabled while busy), `Stop` (enabled while busy, calls `AbortAsync`).
- Context usage `ProgressBar` bound to the latest `ContextUsage`, colour-coded: green < threshold, amber ≥ compaction threshold, red ≥ blocking threshold. Draw two tick marks at the configured thresholds.

**Header row**
- Active agent badge, updated on sub-agent and agent-selection events.
- `Plan` toggle button: opens a side `Popup` with `plan.md` content (refresh on `PlanUpdated`).

**Auto-scroll:** stick to the bottom unless the user scrolled up (attached behaviour on the `ScrollViewer`).

### 9.3 Model tab

| Control | Maps to |
|---|---|
| `ComboBox` Model (filled by `ListModelsAsync`, plus "auto") | `SessionConfig.Model` |
| `ComboBox` Reasoning effort (default/low/medium/high) | `ReasoningEffort` |
| `RadioButton`s Auth: Logged-in user / Token / BYOK | Client options / `Provider` |
| `PasswordBox` Token (Token mode) | `CopilotClientOptions.GitHubToken` (restart client) |
| BYOK group: provider type `ComboBox`, base URL, API key (`PasswordBox`), deployment | `ProviderConfig` |
| `CheckBox` Streaming | `Streaming` |
| `Button` Test connection | `client.PingAsync()` / `GetAuthStatusAsync()` |

Show a policy note under BYOK: *"BYOK bypasses Copilot org policies; use only endpoints your company approves."*

### 9.4 System prompt tab

- `RadioButton`s **Append** / **Replace**.
- A warning `Border` that's visible only for Replace: *"Replace removes the SDK's built-in guardrails and tool guidance."*
- Multiline `TextBox` (monospace, ~20 lines) with a character count.
- `ComboBox` of prompt **snippets** to insert: "Only answer PIL harness topics", "Terse reviewer", "Explain before acting".

### 9.5 Context tab

| Control | Maps to |
|---|---|
| `CheckBox` Infinite sessions | `InfiniteSessions.Enabled` |
| `Slider` 0.30–0.95 "Start background compaction at" + value label | `BackgroundCompactionThreshold` |
| `Slider` 0.50–0.99 "Block until compacted at" | `BufferExhaustionThreshold` (validate: must be greater than background) |
| Live usage bar (same data as the chat footer) | read-only |
| `Button` Compact now | `Rpc.History.CompactAsync()` |
| `Button` Truncate from selected message | `Rpc.History.TruncateAsync(eventId)` (confirm dialog) |
| Session workspace path + `Open folder` | `session.WorkspacePath` (checkpoints, plan.md) |

Demo tip: set background compaction to 0.30 and paste a long file, and compaction events appear quickly.

### 9.6 Tools and permissions tab

**Working directory:** `TextBox` + Browse button; validate that it exists.

**Built-in tools:** `ItemsControl` of `CheckBox`es with descriptions. Example names (confirm against your CLI version): `view`, `create`, `edit`, `shell`/`bash`, `grep`, `glob`, `web_fetch`. Quick-set buttons: **Read-only**, **Read + write**, **Everything**.

**Custom C# tools:** `CheckBox` list from `CustomToolRegistry`. Ship 2–3 demo tools:

```csharp
public sealed class DemoTools
{
    [Description("Return build info for the current repo: target, last build status, warnings count.")]
    public static string GetBuildInfo(string repoPath) => /* read a local json or run msbuild -version */ "...";

    [Description("Look up a requirement by ID from the demo requirements file.")]
    public static string GetRequirement(string id) => Requirements.Load()[id];

    [Description("List SWCs found in the project's ARXML/C sources.")]
    public static string[] ListSwcs(string repoPath) => SwcScanner.Scan(repoPath);
}

public sealed class CustomToolRegistry
{
    private readonly Dictionary<string, AIFunction> _all = new()
    {
        ["get_build_info"]  = AIFunctionFactory.Create(DemoTools.GetBuildInfo,  "get_build_info"),
        ["get_requirement"] = AIFunctionFactory.Create(DemoTools.GetRequirement, "get_requirement"),
        ["list_swcs"]       = AIFunctionFactory.Create(DemoTools.ListSwcs,       "list_swcs"),
    };
    public List<AIFunction> GetEnabled(IEnumerable<ToolToggle> toggles) =>
        toggles.Where(t => t.Enabled && _all.ContainsKey(t.Name)).Select(t => _all[t.Name]).ToList();
}
```

#### App-control tools: the agent calls methods of the running app

The demo tools above read data. To show clearly that **the model can call into your app**, add a second group of tools that act on the WPF window itself. They make the clearest point in the demo: the user types "make the window dark blue" and the window changes colour.

| Tool name | What it does | Demo prompt |
|---|---|---|
| `set_window_color` | Changes the main window background (named colour or `#RRGGBB`) | "Make the window dark blue" |
| `get_app_info` | Returns app state: version, theme colour, window size, current session/model/agent, enabled tools, working dir | "What's the current state of the app?" |
| `show_notification` | Shows a toast/banner in the app | "Notify me when you're done" |
| `set_accent_color` *(optional)* | Changes the accent brush (buttons, badges) | "Use our company orange as accent" |

Implementation: a dedicated service with access to the window, called through the dispatcher because tool calls arrive on background threads.

```csharp
public interface IAppControl
{
    Task<string> SetWindowColorAsync(string color);
    Task<AppInfo> GetAppInfoAsync();
    Task ShowNotificationAsync(string message);
}

public sealed record AppInfo(
    string AppVersion, string WindowBackground, string AccentColor,
    int Width, int Height, string? SessionId, string Model, string? ActiveAgent,
    string WorkingDirectory, string[] EnabledTools, string PermissionPolicy);

public sealed class WpfAppControl(MainWindow window, MainViewModel vm, SettingsHolder settings) : IAppControl
{
    public Task<string> SetWindowColorAsync(string color) =>
        window.Dispatcher.InvokeAsync(() =>
        {
            // Accepts "DarkBlue", "#1E3A8A", "#FF1E3A8A"
            if (ColorConverter.ConvertFromString(color) is not Color c)
                return $"Unknown colour '{color}'. Use a named colour or #RRGGBB.";
            var brush = new SolidColorBrush(c);
            brush.Freeze();
            window.Background = brush;
            vm.Chat.AddSystemNotice($"🎨 Window colour changed to {color} by the agent");
            return $"Window background set to {c}.";
        }).Task;

    public Task<AppInfo> GetAppInfoAsync() =>
        window.Dispatcher.InvokeAsync(() =>
        {
            var s = settings.Current;
            return new AppInfo(
                AppVersion:       Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "dev",
                WindowBackground: (window.Background as SolidColorBrush)?.Color.ToString() ?? "default",
                AccentColor:      (Application.Current.Resources["AccentBrush"] as SolidColorBrush)?.Color.ToString() ?? "default",
                Width:            (int)window.ActualWidth,
                Height:           (int)window.ActualHeight,
                SessionId:        vm.Sessions.SelectedSession?.Id,
                Model:            s.Model,
                ActiveAgent:      vm.Chat.ActiveAgent,
                WorkingDirectory: s.WorkingDirectory,
                EnabledTools:     s.BuiltInTools.Concat(s.CustomTools).Where(t => t.Enabled).Select(t => t.Name).ToArray(),
                PermissionPolicy: s.PermissionPolicy.ToString());
        }).Task;

    public Task ShowNotificationAsync(string message) =>
        window.Dispatcher.InvokeAsync(() => vm.ShowToast(message)).Task;
}
```

Expose it as tools. The descriptions matter: the model decides *when* to call a tool based on its description.

```csharp
public sealed class AppControlTools(IAppControl app)
{
    [Description("Change the background colour of the application's main window. " +
                 "Accepts a WPF colour name (e.g. 'DarkBlue') or hex '#RRGGBB'.")]
    public Task<string> SetWindowColor(
        [Description("Colour name or hex code")] string color) => app.SetWindowColorAsync(color);

    [Description("Get the current state of the application: version, colours, window size, " +
                 "current session, model, active agent, working directory, enabled tools and permission policy.")]
    public async Task<string> GetAppInfo() =>
        JsonSerializer.Serialize(await app.GetAppInfoAsync(), new JsonSerializerOptions { WriteIndented = true });

    [Description("Show a short notification banner to the user inside the app.")]
    public Task ShowNotification([Description("Message text")] string message) => app.ShowNotificationAsync(message);
}
```

Register them in `CustomToolRegistry` next to the demo tools, so they show up as toggles in the Tools tab:

```csharp
public CustomToolRegistry(AppControlTools appTools)
{
    _all["get_build_info"]    = AIFunctionFactory.Create(DemoTools.GetBuildInfo,  "get_build_info");
    _all["get_requirement"]   = AIFunctionFactory.Create(DemoTools.GetRequirement, "get_requirement");
    _all["list_swcs"]         = AIFunctionFactory.Create(DemoTools.ListSwcs,       "list_swcs");
    _all["set_window_color"]  = AIFunctionFactory.Create(appTools.SetWindowColor,  "set_window_color");
    _all["get_app_info"]      = AIFunctionFactory.Create(appTools.GetAppInfo,      "get_app_info");
    _all["show_notification"] = AIFunctionFactory.Create(appTools.ShowNotification, "show_notification");
}
```

In the Tools tab, show custom tools in two groups, **Data tools** and **App-control tools**, each with an "enable group" checkbox.

DI registration (add to §11):

```csharp
s.AddSingleton<IAppControl, WpfAppControl>();
s.AddSingleton<AppControlTools>();
```

Things to show with these tools:
- **The tool call is visible:** the chat shows a `🔧 set_window_color("DarkBlue")` card, the inspector shows the start and end events, and then the window changes colour.
- **Permissions cover your own tools too:** with policy **Ask**, the custom-tool call shows an approve/deny card first (if your SDK version routes custom tools through the permission handler; otherwise add an `OnPreToolUse` rule for `set_window_color`).
- **Hooks can veto app actions:** add an optional hook rule, "block `set_window_color` for black or white" (pure white or black makes the UI unreadable), and show the agent getting the deny reason and picking another colour.
- **The tool's description decides when the model uses it:** disable `get_app_info` and ask "what model are you using in this app?". The agent can no longer answer from the app's real state.
- **Tools can be combined:** "Check the app info; if the permission policy is ApproveAll, turn the window red as a warning, otherwise green." The agent calls `get_app_info`, reasons about the result, then calls `set_window_color`.

**Permission policy:** `RadioButton`s Approve all / Ask me / Deny all / Rules, plus rule `CheckBox`es (writes only in working dir, deny shell, deny URL fetch).

```csharp
public sealed class PermissionPolicy(IPermissionPrompt prompt, Func<HarnessSettings> settings)
{
    public async Task<PermissionDecision> HandleAsync(PermissionRequest req, PermissionInvocation inv)
    {
        var s = settings();
        return s.PermissionPolicy switch
        {
            PermissionPolicyKind.ApproveAll => PermissionDecision.ApproveOnce(),
            PermissionPolicyKind.DenyAll    => PermissionDecision.Deny("Denied by policy"),
            PermissionPolicyKind.Rules      => ApplyRules(req, s),
            _                               => await prompt.AskAsync(req)   // shows PermissionItem in chat
        };
    }

    private static PermissionDecision ApplyRules(PermissionRequest r, HarnessSettings s)
    {
        if (s.RuleDenyShell && r.Kind == "shell") return PermissionDecision.Deny("Shell disabled");
        if (s.RuleWritesOnlyInWorkingDir && r.Kind == "write" &&
            !PathUtil.IsUnder(r.Path, s.WorkingDirectory))
            return PermissionDecision.Deny("Write outside working directory");
        return PermissionDecision.ApproveOnce();
    }
}
```

`IPermissionPrompt.AskAsync` adds a `PermissionItem` to the chat and awaits a `TaskCompletionSource<PermissionDecision>` that the item's buttons complete. Add a timeout (e.g. 2 min → deny) so the agent never hangs.

### 9.7 Agents tab

- `DataGrid` of agents: Name, Display name, Description, Tools (comma list), Skills (comma list).
- Buttons: **Add**, **Edit** (opens `AgentEditorDialog`), **Remove**, **Import .agent.md**, **Export**.
- `ComboBox` **Pre-selected agent** (`(none)` + agent names).
- `CheckBox` **Allow runtime to auto-select agents** (sets `Infer` on each agent; verify property name).
- `Button` **Run fleet…**: asks for an objective, then calls `Rpc.Fleet.StartAsync`. Sub-agent events appear as `SystemNoticeItem`s and in the inspector.

`AgentEditorDialog`: Name, Display name, Description, Prompt (multiline), Tools (`CheckBox` list of built-in and custom tools), Skills (`CheckBox` list of discovered skills), MCP servers (`CheckBox` list).

Importing `*.agent.md`:

```csharp
public static class AgentMarkdownLoader
{
    // ---\nname: x\ndescription: y\ntools: [view, grep]\n---\n<prompt body>
    public static AgentDefinition Load(string path)
    {
        var text = File.ReadAllText(path);
        var m = Regex.Match(text, @"^---\s*\n(.*?)\n---\s*\n(.*)$", RegexOptions.Singleline);
        if (!m.Success) throw new FormatException("Missing YAML frontmatter");
        var meta = new YamlDotNet.Serialization.Deserializer()
                       .Deserialize<Dictionary<string, object>>(m.Groups[1].Value);
        return new AgentDefinition
        {
            Name        = meta.GetString("name") ?? Path.GetFileNameWithoutExtension(path).Replace(".agent", ""),
            Description = meta.GetString("description") ?? "",
            Tools       = meta.GetList("tools"),
            Prompt      = m.Groups[2].Value.Trim()
        };
    }
}
```

### 9.8 Skills tab

- `ListBox` of skill directories, with **Add folder** / **Remove** buttons.
- Discovered skills: scan each directory for `*/SKILL.md`, parse the frontmatter (`name`, `description`), and show a `DataGrid` with an **Enabled** `CheckBox` (unchecked → `DisabledSkills`).
- Preview pane: the selected SKILL.md rendered as markdown.
- **Open in editor** button (shell-execute the file).
- Hint text: *"Agents get no skills by default; assign skills per agent in the Agents tab."*

### 9.9 MCP tab

- `DataGrid`: Enabled, Name, Transport (Stdio/HTTP), Command or URL, Args.
- Add / Remove / **Import from `.vscode/mcp.json`** (reuse existing configs).
- Status column, filled from MCP connection events: Connected / Failed / Blocked by policy.
- Note: *"Org MCP allowlist policies apply to the CLI; blocked servers show here as failed."*

### 9.10 Hooks tab

Each toggle enables one small hook implementation. `HookPipeline` composes them into the SDK `Hooks` object:

| Toggle | Hook | Behaviour |
|---|---|---|
| Log tool calls | `OnPreToolUse` | Emits an inspector event with tool + args, then allows |
| Restrict to working directory | `OnPreToolUse` | Denies read/write tool calls whose path is outside the working dir, with reason |
| Hide verbose tool output | `OnPreToolUse` | `SuppressOutput = true` for tools in `VerboseTools` |
| Add context for tools | `OnPreToolUse` | Example: when `get_requirement` runs, add "Requirements are ASIL-B; cite IDs" |
| Topic filter | `OnUserPromptSubmitted` | If no keyword matches, block the prompt and show "Off-topic for this app" |
| Post-tool annotate | `OnPostToolUse` | Adds a note to failing build results ("Check compiler flags first") |
| Error notice | `OnErrorOccurred` | Shows a `SystemNoticeItem` and a red row in the inspector |

```csharp
public sealed class HookPipeline(Func<HarnessSettings> settings, IEventSink sink)
{
    public SessionHooks Build() => new()
    {
        OnPreToolUse = async (input, inv) =>
        {
            var s = settings();
            if (s.HookLogToolCalls)
                sink.Emit(HarnessEventKind.Other, $"pre-tool {input.ToolName}", input.ToolArgsJson);

            if (s.HookRestrictToWorkingDir && TryGetPath(input, out var p) &&
                !PathUtil.IsUnder(p, s.WorkingDirectory))
                return new PreToolUseHookOutput
                {
                    PermissionDecision = "deny",
                    PermissionDecisionReason = $"'{p}' is outside the working directory"
                };

            return new PreToolUseHookOutput
            {
                PermissionDecision = "allow",
                SuppressOutput = s.HookSuppressVerboseOutput && s.VerboseTools.Contains(input.ToolName)
            };
        },

        OnUserPromptSubmitted = async (input, inv) =>
        {
            var s = settings();
            if (!s.HookTopicFilter) return null;
            var keys = s.TopicFilterKeywords.Split(',', StringSplitOptions.TrimEntries);
            return keys.Any(k => input.Prompt.Contains(k, StringComparison.OrdinalIgnoreCase))
                ? null
                : new UserPromptSubmittedHookOutput { Block = true, Reason = "Off-topic for this app" };
        }
    };
}
```

Hooks read `settings()` **at call time**, so hook toggles apply live, without recreating the session. Point this out during the demo.

### 9.11 Event inspector

- `DataGrid` (virtualized) bound to `ObservableCollection<HarnessEvent>`, capped at e.g. 5,000 rows.
- Columns: Time (HH:mm:ss.fff), Session, Kind (coloured pill), Summary.
- Filter `ComboBox` (multi-select by kind) + search `TextBox` (summary/raw contains).
- Right pane: pretty-printed raw JSON of the selected row.
- Counters strip: turns, tool calls, denied calls, compactions, sub-agents, errors, elapsed time.
- **Export**: JSONL of all events, or a markdown transcript (from `GetMessagesAsync`).
- **Pause** toggle to freeze the view while events keep buffering.

---

## 10. Compare mode (the strongest demo feature)

With **Compare mode** on, the chat area splits into two columns (A and B), each with its own session built from a **different profile**. One prompt box sends the same prompt to both.

Implementation:
- `ChatViewModel` becomes `CompareChatViewModel` with `Left` and `Right` (each a `ChatViewModel` + `HarnessService` instance).
- Two `ComboBox`es pick the profile per side.
- The inspector gets a Session column filter (A / B / both).

Good A/B pairs:
1. **Read-only vs read+write**: the agent explains the change vs makes it.
2. **Skill on vs off**: generated code follows team conventions only with the skill.
3. **Append vs Replace** system prompt: tone, verbosity and safety behaviour change.
4. **Topic filter on vs off**: an off-topic question is blocked on one side.
5. **Agent pre-selected (reviewer) vs none**: focused review vs general answer.

---

## 11. App bootstrap and lifetime

```csharp
public partial class App : Application
{
    private IHost _host = default!;

    protected override async void OnStartup(StartupEventArgs e)
    {
        _host = Host.CreateDefaultBuilder()
            .ConfigureServices(s =>
            {
                s.AddSingleton<ProfileStore>();
                s.AddSingleton<SettingsHolder>();               // current HarnessSettings
                s.AddSingleton<CopilotSdkAdapter>();
                s.AddSingleton<CustomToolRegistry>();
                s.AddSingleton<IPermissionPrompt, ChatPermissionPrompt>();
                s.AddSingleton<PermissionPolicy>();
                s.AddSingleton<HookPipeline>();
                s.AddSingleton<SessionConfigBuilder>();
                s.AddSingleton<IHarnessService, HarnessService>();
                s.AddSingleton<MainViewModel>();
                s.AddSingleton<MainWindow>();
            })
            .Build();

        await _host.StartAsync();
        var adapter = _host.Services.GetRequiredService<CopilotSdkAdapter>();
        var settings = _host.Services.GetRequiredService<SettingsHolder>().Current;

        try { await adapter.StartAsync(settings, CancellationToken.None); }
        catch (Exception ex) { StartupErrors.Show(ex); }  // see §12

        _host.Services.GetRequiredService<MainWindow>().Show();
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.Services.GetRequiredService<CopilotSdkAdapter>().DisposeAsync();
        await _host.StopAsync();
        _host.Dispose();
    }
}
```

---

## 12. Error handling (show these clearly in the demo)

| Situation | Detection | UI |
|---|---|---|
| CLI not installed | Start throws (file not found) | Banner with an install command and a **Retry** button |
| Not logged in | Auth status / 401 | Banner: "Run `copilot` and `/login`" + Retry |
| Org policy disables CLI | Auth / policy error text | Banner: "Copilot CLI is disabled by your organization's policy" |
| Rate limited (429) | Error event | Toast + inspector row; auto-retry is the runtime's job, don't hammer |
| MCP server failed / blocked | MCP status event | Red status in MCP tab |
| Permission prompt timeout | TCS timeout | Auto-deny + `SystemNoticeItem` |
| Invalid settings | VM validation | Disable **New session**, show inline errors (e.g. thresholds, missing working dir) |

Wrap every adapter call in `try/catch`, log to the inspector as `Error`, and never let an exception kill the dispatcher.

---

## 13. Security notes for the demo

- Default the profile to **Ask** permissions + **Restrict to working directory**, and point the working directory at a throwaway copy of the sample repo.
- Never persist tokens or API keys in profile JSON. Store them with DPAPI (`ProtectedData.Protect`) and keep only a key reference in settings.
- Custom tools run in your process. Content exclusion doesn't protect files *your* tools read, so make the tools respect the same boundaries.
- If you use a work seat, tell your Copilot admins what the demo does before you show it widely.

---

## 14. Build order (milestones)

1. **Skeleton:** host, DI, adapter start, `New session`, plain send/receive (no streaming). *Exit: a message round-trips.*
2. **Streaming + chat items:** deltas, tool call cards, Stop. *Exit: you can watch tools run live.*
3. **Inspector:** event normalization, grid, raw JSON, counters. *Exit: every event is visible.*
4. **Settings core:** Model, Prompt, Context tabs + `SessionConfigBuilder` + effective-config preview + Recreate session.
5. **Tools and permissions:** built-in allow-list, custom tools (data + app-control: `get_app_info`, `set_window_color`), policy, inline permission cards. *Exit: "make the window dark blue" works.*
6. **Hooks tab:** pipeline + live toggles.
7. **Agents + Skills tabs:** editors, `.agent.md` import, skill discovery, per-agent skills.
8. **MCP tab.**
9. **Sessions panel:** resume, fork, delete, local index; context actions (compact, truncate).
10. **Profiles + Compare mode.**
11. **Polish:** error banners, validation, markdown rendering, demo profiles shipped with the app.

---

## 15. Demo script (12–14 minutes)

1. **Baseline (1 min):** default profile, ask "Explain what this repo does." Show tool calls and the inspector filling up.
2. **The loop is not yours (1 min):** point to planning, tool calls, history, compaction: none of it is app code.
3. **The agent calls the app (2 min):** "What's the current state of the app?" calls `get_app_info`. Then "Make the window dark blue" calls `set_window_color`, and the window changes. Finish with the chained prompt: "If the permission policy is ApproveAll turn the window red, otherwise green."
4. **Permissions (2 min):** "Add unit tests for Parser.cs" with **Ask**. Approve the read, deny the write; the agent adapts. Switch to Rules and retry.
5. **Hooks live (2 min):** turn on Topic filter and send an off-topic question, which is blocked. Turn it off without recreating the session.
6. **Skills (2 min):** Compare mode, skill on vs off, same prompt. The conventions show up only on one side.
7. **Agents (2 min):** pre-select `reviewer` (read-only tools) vs `generator`. Then run fleet mode on a small objective and watch the sub-agent events.
8. **Context (1 min):** low compaction threshold plus a long input, and compaction events appear in the inspector.
9. **Wrap-up (1 min):** show the effective-config JSON. Every setting is one line in `SessionConfig`.

---

## 16. Shipped demo profiles

| Profile | Key settings |
|---|---|
| `01-baseline` | auto model, Append prompt (empty), Ask permissions, all read tools, no agents |
| `02-read-only-reviewer` | Agent `reviewer` pre-selected, tools: view/grep/glob only, Deny write/shell |
| `03-generator` | Agent `generator`, read+write, Rules: writes only in working dir, shell denied |
| `04-skills-on` / `04-skills-off` | Same as 03, team conventions skill enabled / disabled |
| `05-replace-prompt` | Replace mode, terse reviewer prompt |
| `06-topic-locked` | Topic filter hook on, custom tools only |
| `07-tight-context` | Background compaction 0.30, blocking 0.60 |
| `08-app-control` | App-control tools only (`get_app_info`, `set_window_color`, `show_notification`), built-in tools off, Ask permissions |

Store them as JSON in `Profiles/` next to the exe so you can tweak them before the demo without rebuilding.
