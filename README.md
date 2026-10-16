# Build your own me-box

A prompt for a coding agent to build a private workspace that you can use from your phone or computer.

You get VS Code in a browser, a phone-friendly agent chat app, and the same agent sessions in both. Your files and agent processes live on an always-on Linux machine. Close the browser, change devices, and return to the same work.

This guide describes a fresh implementation of that setup. It contains no source code from the original system and requires no access to its repositories. The interface and behaviour are specified below so that your coding agent can build its own version.

## How to use this document

Give this whole document to a coding agent that can run commands and edit files. Start on your computer if you do not yet have a server. Once the server exists, the agent needs SSH access or a terminal session on it. A chat window without those tools can explain the steps but cannot complete the build.

Tell the agent your domain if you already own one. Leave other choices open if you want the defaults below. Do not paste passwords, API keys, or recovery codes into the conversation. The agent should prepare local credential files and explain how to fill them in securely.

Expect several stages and some account setup. The prompt tells the agent to continue through implementation and tests, but domain purchases, billing, identity checks, and browser sign-ins may still require you. It must ask for those steps individually. A single prompt provides the specification; it does not remove those account requirements or guarantee a particular agent's implementation quality.

The core build needs a Linux server, a domain, Cloudflare, and at least one coding-agent provider. Use Tailscale for private administration between your computer and the server. GitHub stores your new code. Vercel, Supabase, and Resend are optional additions for applications you build in the workspace. They are not prerequisites for the editor or persistent chat.

The specification includes the later interface refinements that make several simultaneous sessions manageable. Queues, forks, status indicators, and provider handovers are part of the requested result. Build them after the basic connection works, then test them before calling the interface complete.

Copy from “Prompt begins” through “Prompt ends”, or attach this file and say “Follow the build prompt in this document”.

## Prompt begins

Build my own private me-box. Implement, deploy, and verify it. Do not stop at a plan, a mock-up, or a chat UI that only returns model text.

### 1. What I want

I want one always-on machine where my projects, files, and coding-agent sessions live. I want to use it through my own domain from a phone and a computer.

Deliver these core interfaces:

- `code.<my-domain>` opens a browser-based VS Code editor with my project files, terminal, and a custom agent panel.
- `chat.<my-domain>` opens a responsive agent chat app. It supports installation on my phone's home screen.
- The editor panel and chat app attach to the same sessions. A session continues when I close either interface.
- `dev.<my-domain>` opens the current application preview. Give me stable HTTPS review links that work from my phone.

Use these companion interfaces after the core works:

- `files.<my-domain>` provides authenticated file browsing and downloads within my workspace.
- `term.<my-domain>` provides an authenticated browser terminal with tmux for maintenance and recovery.
- `hub.<my-domain>` lists the private tools and projects with links to their stable hostnames.

Each person who follows this prompt should get their own installation and accounts. Build for one trusted owner. Shared access for unrelated users would need a separate isolation design.

### 2. How to work with me

Inspect before you install or replace anything. Check existing software, services, DNS records, repositories, and logins. Preserve working resources and unrelated edits.

Maintain `BUILD-STATE.md` in the new private repository. Record completed stages, next steps, blockers, installed versions, configuration paths, and test results. Record resource identifiers where necessary, but never secrets. Read this file when a session resumes.

Ask for one action or decision at a time. First do everything you can independently. Each request must include the exact dashboard link or local command, the setting and value, why I must do it, and what I should reply when it is complete. Do not ask me to research an implementation detail you can resolve yourself.

Get my approval before purchases, paid resource creation, destructive changes, or public publication. Use existing authorisation for routine implementation and testing. Do not send email or other messages under my name. Prepare drafts and let me send them.

Use current official documentation for installation, authentication, API contracts, and account capabilities. Record the versions that work. Do not hard-code a supposedly latest model or copy an old login workaround. If a provider lacks a feature, show that limitation in the UI.

Build in small stages. After each stage, run its acceptance checks, fix failures, and continue. Mark a browser-only test as pending until it has actually been performed. Do not treat a successful build or HTTP 200 response as proof that the complete interface works.

### 3. Defaults and initial discovery

Start by discovering what is already available. Then resolve missing inputs in dependency order:

1. My domain, or the domain I want to buy.
2. An existing always-on Linux machine, or my preferred VPS provider and spending limit.
3. The email address that may access my private tools.
4. The first coding-agent provider I can authenticate with.
5. My private GitHub repository or permission to create one.

The reference installation uses a Vultr VPS in Seoul, with 4 virtual CPUs, about 8 GB RAM, and Ubuntu 24.04.4 LTS. Its original provisioning record described a 160 GB storage plan; the running machine reported a 180 GiB virtual disk when inspected for this guide. These describe that installation, not a current purchasable SKU or price guarantee.

For a fresh installation, propose a supported Ubuntu LTS release, 4 virtual CPUs, 8 GB RAM, and roughly 160-200 GB SSD. Use a smaller disk only after estimating project files, uploads, local database copies, and build caches. These are planning defaults, not measured minimum requirements. Check current plans and prices and ask before provisioning. No GPU is required when models run through remote providers. Measure RAM, disk, and build load before recommending a larger machine.

Prefer a server region near me. Keep a recovery route through the provider's console or SSH. Do not make the chat app the only way to administer the machine that runs it.

Use a dedicated non-root Linux account, called `<box-user>` below. My workspace should be `/home/<box-user>/me`. Keep credentials outside that directory.

### 4. Architecture and repository

Use this architecture unless inspection reveals a concrete incompatibility:

```text
Phone or computer
  -> HTTPS on my domain
  -> Cloudflare Access identity check
  -> Cloudflare Tunnel
  -> services on the Linux box
       code-server -> remote VS Code extension host -> agent daemon
       chat app -> authenticated WebSocket -> same agent daemon
       preview, file browser, terminal, and hub

Agent daemon
  -> local persistent session store
  -> provider adapters
  -> coding-agent processes with access to authorised workspace files
  -> remote model providers

Owner's computer
  -> Tailscale private network
  -> SSH on the box for administration and recovery

Optional applications
  -> Vercel for web deployment
  -> Supabase for application data, authentication, and storage
  -> Resend for application email
```

Keep the persistent daemon and coding-agent processes on the Linux box. Serve the chat app from the daemon's origin. Do not tie a running agent to a browser connection or an ordinary web request.

Use TypeScript for the daemon, React with Vite for the shared UI, WebSockets for live events, and SQLite for local session storage. Use systemd for service supervision. These are implementation defaults, not a requirement to reproduce someone else's internals.

Create one private repository for the new system, with these parts:

```text
daemon/                  session lifecycle, persistence, provider adapters
client/                  shared chat UI
extension/               VS Code webview and remote-host bridge
shared/                  protocol types and validation
ops/                     service templates, setup, backup, restore
tests/                   protocol, integration, and browser checks
docs/                    setup, operations, and recovery instructions
.env.example             variable names and safe placeholders only
BUILD-STATE.md           progress and verified configuration
```

Use separate repositories for applications I later build. Use task branches and worktrees for code changes. Keep deployable releases identifiable and retain a working release for rollback. Do not put unrelated projects, live databases, or credentials into the system repository.

#### Organise the workspace for later agent work

Make `me/` an umbrella directory, not a Git repository. Each project owns its own repository. Use this starter structure and adapt the project names to my work:

```text
me/
  CLAUDE.md                 shared workspace instructions
  AGENTS.md -> CLAUDE.md     same instructions for the other agent entrypoint
  CHAT.md                   communication preferences
  WRITING.md                prose preferences
  workspace-config/         private repository for these shared documents
  agentd/                   private repository for the system built here
  notes/                    private notes repository
  projects/
    example/
      app/                  project's main checkout, with its own .git
      worktrees/            one Git worktree per concurrent code task
      data/                 runtime data, outside the code repository
  inbox/                    incoming files, outside Git
  outbox/                   files prepared for download, outside Git
  archive/                  inactive projects with clear ownership
```

Store the shared instruction documents in `workspace-config/` and expose them at the workspace root through symlinks. The tree above shows their logical locations. Keep `CLAUDE.md` as the canonical shared entrypoint and make root `AGENTS.md` resolve to it. Do not maintain two independent copies. Credentials stay outside the entire workspace. A directory being outside Git does not make it inaccessible to the file browser or an agent.

In the shared instructions, maintain a concise project directory map. Record each repository's purpose, visibility, base branch, stable review URL, and deployment method. Explain which parent directories are only containers. Verify the repository root before every commit. Never stage files from another repository or move private material into a public project without explicit authorisation.

Start a session in its project's directory. Before parallel implementation, create a task branch and worktree under that project's `worktrees/`. Keep worktrees under `me/` so the workspace instructions remain discoverable, and give each its own progress record. Do not let multiple agents edit the same checkout concurrently. Keep live deployment checkouts outside the task directories and deploy known commits.

For a new project, create its local instruction file, README, and a short task-state document. Record the actual install, build, test, and preview commands, runtime requirements, and data locations. Update the root directory map. Leave unrelated folders and projects untouched.

#### Set up AGENTS.md and CLAUDE.md

Use the filename `AGENTS.md`, with an S. Codex documents this entrypoint in its [custom-instructions guide](https://developers.openai.com/codex/guides/agents-md/). Claude uses `CLAUDE.md`; consult its [project-memory documentation](https://code.claude.com/docs/en/memory). Verify discovery through our installed integrations, especially when sessions start inside separate repositories or worktrees. An SDK may require configuration to load filesystem instructions.

Write a compact shared entrypoint with these instructions, then fill in the local project map:

```text
This is my private workspace. The workspace root is not a Git repository.
Read CHAT.md for communication preferences and WRITING.md before prose work.
Find the target repository in the directory map before editing files.
Read its project instructions and current task-state file.
Check existing changes and preserve work outside the current task.
Keep credentials outside the workspace and out of logs, chat, and Git.
Use a separate branch and worktree for each concurrent code task.
Run the project's checks and record what actually passed.
Never infer permission to publish private material or send messages as me.
Give stable authenticated review links that work on my phone.
Keep task state current so another session can continue the work.
Ask for one necessary user action at a time, with exact instructions.
```

Define merge and deployment permissions explicitly with me. Do not inherit another owner's standing approval. Keep communication and writing preferences in their own files so agents can read them when relevant. Keep project commands and constraints in project-local instructions. Use a local `AGENTS.md` symlink to that project's `CLAUDE.md`, or another tested single-source arrangement, when both providers should read the same rules.

Do not assume that a repository-local instruction file automatically loads the umbrella file. Configure each adapter's documented instruction discovery or explicitly load the shared entrypoint and applicable project instructions. Check overrides and instruction-size limits. Keep private umbrella instructions out of public repositories; public projects get only project-specific instructions suitable for publication.

Instructions express preferences; they do not enforce filesystem permissions. Keep the runtime access controls described elsewhere in this prompt. Keep durable task state in files, not only in the chat transcript or provider memory.

Acceptance: start fresh Claude and Codex sessions in the umbrella directory, a project repository, and a task worktree. Use a harmless temporary instruction marker to check that each loads the shared and project rules, then remove it. Verify that both choose the correct repository and test command. Confirm that the shared instruction files survive backup and restore.

### 5. Prepare the machine

Install supported versions of Git, a Node.js runtime compatible with the selected providers, the package manager, build tools, tmux, and the GitHub CLI. Install a browser test runner if resources allow. Pin dependencies with a lockfile.

Set up key-based SSH and test a second connection before changing SSH or firewall settings. Keep application ports private. Permit only the required administration path and tunnel connectivity. Configure security updates and log rotation.

Create these directories with appropriate ownership:

```text
/home/<box-user>/me/                   projects and documents
/home/<box-user>/.config/me-box/       private configuration and credentials
/home/<box-user>/.local/share/me-box/  database, uploads, and runtime state
/home/<box-user>/.local/state/me-box/  logs and maintenance state
```

Use mode `0700` for private credential directories and `0600` for credential files. Credentials must not appear in command output, browser bundles, Git, screenshots, or public downloads. Pass only the credentials each service needs. Run agent processes without root privileges or unrestricted passwordless sudo.

Keep an inventory of local ports. Suggested loopback bindings are `127.0.0.1:8080` for code-server, `:8086` for chat and the daemon, `:4321` for previews, `:8081` for files, `:8084` for the terminal, and `:8082` for the hub. Reuse or change these after checking for conflicts. Bind explicitly to IPv4 loopback.

Acceptance: I can reconnect through the recovery route. Required tools run as the dedicated user. No application port is reachable directly from the public network.

#### Private administration through Tailscale

Install Tailscale on the box and my computer, then join them to my own tailnet. Restrict access so only my authorised devices can reach the box's SSH service. Configure ordinary key-based OpenSSH over its tailnet address by default. Verify it from a second terminal before closing public SSH. Follow the [Tailscale quickstart](https://tailscale.com/docs/how-to/quickstart).

Record whether we use ordinary OpenSSH over Tailscale or the separate Tailscale SSH feature. The latter has its own authentication and access-policy configuration; enabling the network alone does not enable it. Consult [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh) if we select that option.

Use this route when Cloudflare or a browser service fails. Keep the VPS provider console as a further recovery option. Do not require Tailscale on my phone for normal browser access through Cloudflare. Do not enable an exit node, subnet router, or public Funnel for this setup. Optional Taildrop can deliver files to a workspace inbox later.

Acceptance: SSH works over the tailnet, public SSH is closed after that test, and the browser interfaces still work on a phone without Tailscale. Document device approval, revocation, and expiry so that maintenance access does not disappear unexpectedly.

### 6. Register the domain and protect the interfaces

If the domain already has DNS records, export or record them first. Preserve existing websites and mail records. If Cloudflare will manage DNS, add the zone and follow its current nameserver instructions at the registrar. Confirm delegation before you depend on the new hostnames.

Create a named Cloudflare Tunnel and install its connector as a service. Store its credentials outside the workspace. Prefer the current documented dashboard or API flow over scripts that depend on undocumented endpoints.

Create Cloudflare Access applications for each private hostname before exposing the corresponding service. Allow only my specified identity. Choose an identity provider or email one-time PIN according to my account. Do not use an allow-everyone or bypass policy for these tools.

Route each hostname to its loopback service. Keep unmatched routes closed. Use a named tunnel and stable hostnames, not temporary tunnel URLs. For local ingress configuration, include a final reject rule. Do not overwrite an existing tunnel configuration wholesale.

Protect both HTTP and WebSocket routes. Validate Access JWTs at the origin or through the connector's documented Access-validation facility, with the correct issuer and audience. Do not trust a forwarded email header by itself. Give the local editor bridge its own authenticated route or local transport; do not bypass authentication merely because a proxied connection comes from loopback. Check WebSocket origins against an explicit allowlist.

Cloudflare Tunnel provides connectivity; Access supplies the identity policy. Follow the [official private-web-application guide](https://developers.cloudflare.com/cloudflare-one/setup/secure-private-apps/private-web-app/) for their configuration.

Acceptance: an allowed identity reaches each enabled tool. A fresh unauthenticated browser and a disallowed identity cannot read data or open an agent socket. A direct request to the server's public IP cannot reach these services. Verify TLS and WebSocket connections through the real hostnames.

### 7. Install browser VS Code

Install code-server from its official distribution. Configure it as a service under `<box-user>`, with the workspace as its starting folder. Keep its own authentication enabled unless a verified Access-enforced setup justifies a documented alternative. Follow the [code-server access guide](https://coder.com/docs/code-server/guide).

Confirm that the extension host runs on the box and can access workspace files. Test the extensions we need on code-server rather than assuming every desktop VS Code extension is compatible.

Acceptance: `https://code.<my-domain>` opens after authentication. I can create a harmless file, edit it, run a terminal command, and see the result on disk. A browser refresh preserves the workspace. The public IP does not expose the editor.

### 8. Connect real coding agents

Implement one provider fully, then add the second. Support Claude and Codex through separate adapters. Use their supported coding-agent integrations so that sessions can inspect files, change code, and run tools on the box.

For Claude, use the official Agent SDK. Start with API-key authentication through a private server-side environment file. Consult the [Agent SDK documentation](https://code.claude.com/docs/en/agent-sdk/overview). Do not reproduce a consumer-login flow from another installation or assume subscription access is interchangeable with API access.

For Codex, prefer the documented app-server integration for a custom interactive client with streamed events and approvals. The TypeScript SDK is another option for suitable automation, but check its capabilities before promising the same controls. Consult the [app-server documentation](https://developers.openai.com/codex/app-server/) and [SDK documentation](https://developers.openai.com/codex/sdk/). Select and test one integration rather than implementing both unnecessarily.

Check available authentication methods for the chosen integration. Distinguish browser subscriptions, API billing, and organisation permissions. Prove that the provider works under the daemon's Linux user and service environment, not just in an interactive shell.

The adapter must support start, resume, stream, interrupt, and error reporting. Where supported, it must also relay tool approvals and user questions. Discover available models through documented mechanisms or maintain a verified configuration. Display the provider, model, account label, and actual permission mode. Never display account secrets.

Explain the distinction between a model API and a coding-agent integration in the setup record. A raw model API supplies model responses and tool requests. A coding-agent integration also supplies an agent loop, local tool execution, session context, and permission controls. Our daemon supervises that integration and maps its events into the shared interface. It should not rebuild the provider's agent loop unnecessarily.

For each adapter, document the installed package or executable, its version, authentication method, thread identifier, resume behaviour, event mapping, cancellation mechanism, and supported approval controls. Include a minimal executable smoke test in the new repository. Keep model identifiers configurable and test actual account access.

Keep provider sessions separate from our own session IDs. Our store preserves what the user sees; the provider's session state preserves its native context. Losing one is not repaired merely by retaining the other. If native resume fails, preserve the visible transcript and offer an explicit fresh start with a text-history handover.

Normalise provider events carefully. Some integrations emit incremental text; others emit updated snapshots. Neither may duplicate completed text in the UI. Preserve tool-call identifiers, results, failures, and approval decisions. Test the normaliser against recorded, synthetic fixtures with no private data.

Start with the narrowest useful workspace permissions. Offer broader access only as an explicit setting. If an SDK cannot enforce a mode, disable it or label its actual behaviour. Do not display an “Ask” mode that silently permits tool execution. Keep administration credentials separate from ordinary agent access.

Acceptance: each configured provider creates and reads a test file in a disposable workspace, runs a harmless command, streams its answer, resumes a previous conversation, and reports invalid credentials clearly. A rejected tool approval prevents the action. The Stop control interrupts the active turn.

### 9. Build the persistent daemon

The daemon owns sessions. Clients only attach to them. A lost socket must never cancel an agent turn.

Persist session metadata, transcript events, provider thread IDs, drafts, queue state, and message acknowledgements. Store account references rather than raw credentials in session records. Use migrations and transactional writes.

Use one provider-independent event protocol for both clients. Define events for user messages, assistant text, tool calls, tool results, permission requests, user questions, status changes, errors, and turn completion. Preserve provider-specific metadata when necessary. Only display reasoning or summaries that the provider actually exposes.

Each durable event needs a session ID, sequence number, timestamp, and type. A client reconnects with the last sequence number it has received. Replay the missing events, then continue live delivery. Make the replay-to-live transition atomic or buffer events so it cannot lose or duplicate a message.

Stream partial text live. Persist completed blocks. Keep the current partial available to clients that attach mid-turn. Make partial replacement explicit so that the completed block does not appear twice. If the daemon crashes before a block completes, mark the interruption honestly.

Give each submitted message a client-generated ID. Store it on the client until the daemon acknowledges durable receipt. Deduplicate retries in the database. A reconnect must not execute the same user message twice. Keep a durable distinction between accepted, queued, started, completed, and interrupted work; transport deduplication does not guarantee exactly-once external side effects after a crash.

Permit one active turn per session. Queue further messages with visible status and cancellation. Define Stop to interrupt the current turn and clear remaining queued work. Make that effect clear beside the control, and preserve the cancelled text for recovery. Other sessions may run concurrently within configured resource limits.

On daemon restart, recover transcripts and session metadata. Mark abandoned active turns as interrupted. Resume provider context on the next explicit message where supported. Do not claim that an in-flight process survives a machine restart, and do not silently rerun a potentially destructive command.

Store drafts per session. Do not overwrite text while a user is actively editing it on another device. Use revisions or a clear conflict policy. Distinguish unread activity from work the owner has marked resolved.

Acceptance: start a real task, close every browser, then attach from another device. The task has continued and its transcript is intact. Repeat with a forced connection loss and repeated message delivery. Verify one execution. Restart the daemon during a disposable test task and confirm an explicit interrupted state and a usable next turn.

### 10. Build the phone chat app

Use the shared React client for the standalone app and the editor webview. Start with a session list and a conversation view.

The session list needs titles, folders, provider and model information, running or waiting status, unread indicators, and search. Support create, rename, archive, and resolve. Preserve manually chosen titles.

The conversation view needs streamed text, sanitised Markdown, code blocks, expandable tool calls and results, approval cards, answerable user questions, visible errors, a message queue, and Stop. Keep long tool output collapsed or load it on demand. Do not interpret model output as trusted HTML.

The composer needs multiline input, a durable draft, file references, and attachments. Resolve paths on the server against allowed roots. Prevent traversal and symlink escapes in file and upload endpoints. Set upload size limits. Show whether an image will be supplied as image input or merely as a file path.

Make the app usable with a phone keyboard. Keep the composer visible, respect safe-area insets, and test viewport changes. Long code blocks should scroll within their own area, not push the entire page sideways. Keep tap targets large enough and provide clear focus states and accessible labels.

Add a web app manifest and icons. Show connection status. If you add a service worker, restrict it to the app shell; do not cache authenticated API responses or transcripts by default. Preserve drafts and unsent message IDs locally, clear them on explicit sign-out, and explain that running an agent still requires a connection to the box.

When Access expires, show a sign-in action and preserve the draft. A failed WebSocket must not create an endless silent reconnect loop. Restore the connection and replay after authentication.

Acceptance: test desktop and mobile viewport sizes, the phone keyboard, a long transcript, code overflow, upload errors, and an expired login. Test home-screen installation on an actual supported phone browser. Record tests that still require my device.

#### Layout and everyday controls

Use a compact desktop sidebar for sessions and a main conversation pane. On a phone, show one pane at a time with a clear way back to the list. Use a neutral light or dark theme, a readable text scale, and consistent colours for paths, tools, and state. Keep the header compact with the title, folder, status, and session actions. Put provider, model, effort where supported, account, and permissions in a compact settings strip near the composer.

Keep the header and composer visible while only the transcript scrolls. Open long conversations at the newest turns and load older history on demand. Once I scroll upwards, stop following new output until I return to the bottom. Preserve my scroll position as older history loads. Switching quickly between sessions must not let a slower response replace the newly selected thread.

Offer a new-session shortcut that inherits the current folder, provider, model, account, and permissions. Save the most recently used choices for the ordinary new-session form. Generate a short title after the first completed turn and refresh it periodically if the conversation changes. A manual rename disables automatic retitling.

Show context usage only if the provider reports a suitable measure. Label estimates and distinguish context size from cumulative token usage or cost. An aggregate per-turn token count must not masquerade as an exact context-window fill level.

#### Queued messages and connection failures

Distinguish two queues. A message may wait on the device for server acceptance, or it may already be accepted by the daemon and wait for the current agent turn to end. Label those states separately.

While a turn runs, let me type and send follow-up messages. Show accepted follow-ups below the active turn in order, with a visible “queued” label and an individual Cancel control. A server acknowledgement determines whether cancellation beat the start of execution. If the message has already started, report that and offer Stop; do not pretend cancellation succeeded.

Run queued messages in order after the current turn ends normally. Keep the session's overall status as working while further accepted work remains. Do not flash a green completion dot or send a final completion notification between queued turns. If a turn fails, loses authentication, or is interrupted, halt the queue and preserve the unsent text with an explanation.

Accept attachment-only messages. Preserve readable original filenames, including non-English characters, while keeping storage paths safe. Reauthentication must retain queued submissions and drafts. A long tool call must remain visibly active even when it produces no text; show elapsed time and Stop. Do not kill active work because a browser or stream has been quiet for a fixed period.

#### Working, done, error, and ask indicators

Store execution status, unread state, outcome, and the user's resolved mark separately. They answer different questions. Use these visible states with both colour and text:

- Yellow, “Working”: a turn is active or accepted follow-ups are still pending execution.
- Blue, “Needs your answer”: an approval or structured user question currently blocks progress. This takes precedence over read state.
- Blue, “For you”: a completed, unread turn left a question or required action for the user.
- Red, “Failed” or “Interrupted”: the last unread turn failed or was cut off.
- Green, “Finished”: an unread turn completed without a detected required user action.
- Grey, “Read”: I have caught up with an inactive session. Keep the actual outcome visible in its transcript.

Synchronise the last-read sequence through the daemon. Opening the latest events on one device updates the other device's unread dot. Do not mark unseen new events as read merely because the session was selected earlier.

Keep the manual resolved mark independent. New sessions start unresolved and sort above resolved sessions. Mark done clears that mark; the next user message reopens it. Reading a conversation must not mark its work resolved. Show counts for working sessions, failures, items for me, and new completed turns above the list.

For a completed answer, use explicit structured action metadata where available. Otherwise, a documented heuristic may inspect the final answer for a required action or closing question. Treat that as an estimate. Do not classify optional suggestions as obligations, and do not claim that a green dot proves the user's goal was met. A live approval request is authoritative and remains visible until answered or cancelled.

#### Fork a conversation from a message

Place a Fork action beside messages. It creates a new session and leaves the original unchanged. Inherit the selected folder, provider, model, account, and permission settings. Show a link to the parent conversation and the branch point.

Forking at a user message copies the history before that message and places its text and attachment references in the new composer for editing. Forking at an assistant message includes that message. Keep tool calls and their results together at a valid boundary. Never rerun a historical tool call merely to reconstruct the transcript.

Use a native provider fork when the selected integration supports it at the requested boundary. Otherwise, create a new provider session with the selected history as context on the next message. Label this explicitly as a text-history handover. Copying chat rows alone is not a native provider fork.

A conversation fork does not rewind workspace files. State that beside the Fork action. Ask before changing project files to an earlier revision, and use a separate worktree if the user wants an independent code experiment.

#### Change provider, account, or folder

Allow a conversation to move between Claude and Codex without losing its visible transcript. Switch only at a safe turn boundary or after an explicit interruption. Give the new provider the relevant history as text, state that it is a handover, and record the switch in the transcript. Avoid pretending it shares the previous provider's hidden session state.

Reset incompatible model and effort choices, then show the effective settings. Keep account credentials separate. An expired login should offer reauthentication without deleting conversations. A folder change must update the actual tool-execution directory and verify access there. If native resume cannot follow that change, explain the fallback before the next turn.

#### Interface acceptance scenarios

Verify these behaviours with both the phone client and the editor panel:

1. Queue two follow-ups during a long task. Cancel the second. The first runs once, the second never runs, and the status stays working between turns.
2. Disconnect before acknowledgement, reconnect, and retry the same message ID. The message appears and executes once.
3. Produce a normal finish, a tool error, a user question, and a permission request. Each has the correct text and colour. Reading an inactive thread clears its unread dot; reading a blocked thread does not hide its pending question.
4. Mark a session resolved, then send another message. It becomes unresolved on both clients.
5. Fork at a user message, edit the draft, and run it. The original transcript and its queue remain unchanged. Also test a tool-call boundary and a provider without native fork support.
6. Hand a thread to the other provider. The transcript remains, the effective account and settings are visible, and the new agent receives the declared history.
7. Open a long conversation, scroll up while new text arrives, then switch sessions quickly. The UI preserves the selected session and does not force-scroll over what I am reading.

### 11. Add the VS Code agent panel

Create a VS Code extension with a webview that reuses the chat client. Provide commands to open an agent panel, open an additional panel, send the selected code with its path and line range, and open referenced files at a line.

The browser webview must communicate through the extension host. The extension host connects to the daemon locally and relays validated protocol messages. A browser's `127.0.0.1` refers to the user's device; it does not refer to the server.

Keep local bridge credentials in the extension host. Use a restricted content-security policy and validate messages across the bridge. Restrict file-open commands to authorised workspace paths.

Each panel can attach to a different session. Panel titles should reflect the selected conversation. Include a visible button for a new panel because browsers may intercept keyboard shortcuts.

Package and install the extension in code-server. Document when a UI rebuild needs a panel reload and when an extension-host change needs a service restart. Perform disruptive restarts from the independent maintenance route.

Acceptance: attach the phone app and editor panel to the same session. Both show the same ordered events. Send selected code from the editor, open a file reference, use a second panel, and close the editor during a task. The phone session continues.

### 12. Add previews, files, terminal, and the hub

Reserve `dev.<my-domain>` for the application currently under review. Check port use before starting a preview and record which project owns it. Bind the preview to loopback and keep it behind Access. Configure HMR and allowed hosts for the stable hostname where the framework requires them.

For applications with login, register the stable callback URL with the application's authentication provider. If CAPTCHA is used, register the preview hostname there too. Verify the whole sign-in journey before handing me a review link.

Install a maintained file-browser service and restrict it to the workspace. Keep its own authentication where supported, behind Access. Check symlinks and download routes so that private configuration outside the workspace cannot be read through them. Map workspace paths to authenticated HTTPS links for phone access.

Add a browser terminal with tmux for recovery tasks. Keep it owner-only behind Access and a documented authentication arrangement. Add a simple hub page with links to enabled tools and projects.

Acceptance: I can review a demo app on my phone, download a workspace file, and reach the maintenance terminal. Requests for credential paths or paths outside the file-browser root fail. Preview login returns to the correct hostname.

### 13. Optional application services

After the core build passes, ask whether I want these integrations. Keep their credentials and data separate from the agent daemon. Build one small example app only if I enable this stage.

For Vercel, create or link a project to the example app's private Git repository. Set the root directory, build settings, and environment variables for the intended deployment environments. Add `app.<my-domain>` in the project's domain settings, then create the exact DNS records Vercel specifies. Keep existing tunnel hostnames intact. Verify the deployment, domain, and TLS certificate. Follow [Vercel's domain guide](https://vercel.com/docs/domains/working-with-domains/add-a-domain).

For Supabase, create or reuse an appropriate project and record the region. Add database migrations and enable row-level security on exposed application tables. Use the browser-safe key with authenticated policies; keep secret or service-role keys server-side. Configure the Site URL and exact permitted callback URLs for production and the stable preview. Test sign-in, sign-out, and denial of another user's rows. Consult [redirect URL configuration](https://supabase.com/docs/guides/auth/redirect-urls) and [row-level security](https://supabase.com/docs/guides/database/postgres/row-level-security).

For Resend, verify a sending subdomain such as `mail.<my-domain>` with the DNS records its dashboard specifies. Preserve the domain's existing mail records. Store a suitably scoped key server-side and configure the sender address. Prepare a test email for me to send to my own address. Add Supabase custom SMTP only if the application needs it. Follow [Resend's domain verification guide](https://resend.com/docs/dashboard/domains/introduction).

Cloudflare Access protects the box's private interfaces. Application authentication protects application users and data. Verify each application's own controls even if its preview also sits behind Access. Never assume the Access policy on one hostname protects a Vercel deployment URL or a Supabase endpoint.

Acceptance: the enabled example app has a working stable domain, the intended login behaviour, and tested data isolation. No provider secret appears in the browser bundle. Email remains a draft until I send it. Disabled integrations remain documented as optional, not failed requirements.

### 14. Make the installation recoverable

Create systemd units with explicit users, working directories, executable paths, environment files, restart policies, and log locations. Do not rely on shell startup files. Test automatic startup after a reboot through the independent recovery route.

Back up the database through a consistent SQLite backup mechanism, not a blind copy of a live file. Include uploads, configuration templates, private repository state where appropriate, and provider session data needed for resume. Treat provider state as sensitive. Keep an encrypted off-box backup and its recovery key separately. Agree a retention policy and record it.

Test restore into a separate directory or machine. Confirm that transcripts and attachments can be read, and document any provider reauthentication required. Git alone is not a backup of runtime state or uncommitted work.

#### Concrete backup mechanism

Use restic with a private Backblaze B2 bucket as the default off-box backup mechanism. This is the combination used by the reference installation. Create my own bucket and scoped application credentials after I approve the account and storage cost. Initialise an encrypted restic repository and save its password in my password manager as well as the restricted local configuration. Follow [restic's repository setup documentation](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html).

Back up workspace files, uncommitted and untracked work, Git metadata, uploads, shared instructions, required provider state, and private configuration. Include consistent database snapshots. Exclude regenerable dependencies, build caches, temporary files, and the live database files when a consistent snapshot replaces them. Review exclusions explicitly; do not assume everything Git ignores is disposable. Back up required system-level service and tunnel configuration too, through a restricted staging process where necessary.

Run a systemd backup timer nightly at 03:30 in my chosen server timezone, with missed-run recovery enabled. The reference installation uses that schedule. Record success, failure, and the last completed snapshot. Prevent overlapping backup and maintenance jobs. On a partial or failed backup, report it and do not run the automatic retention step.

Default retention is 7 daily, 4 weekly, and 12 monthly snapshots. After a successful backup, use restic's retention and pruning commands with a reviewed policy. Follow the [retention documentation](https://restic.readthedocs.io/en/stable/060_forget.html). Schedule integrity checks and periodic restore drills. Nightly backups can lose changes since the last successful snapshot; take an additional snapshot before risky maintenance.

Test recovery of a deleted file, the session database, attachments, and shared instruction symlinks into an isolated destination. Document how to provision a replacement VPS, restore ownership and permissions, reinstall services, recover or rotate credentials, and reconnect the tunnel. Record the time needed for a restore drill instead of promising an unmeasured recovery time.

Distinguish this file-level backup from a full server image. Provider snapshots are an optional additional layer. Supabase and other external services need their own backup and restore procedures; the VPS backup does not automatically include their hosted data. Encrypted backups also need deletion protection if protection from a compromised server is required: encryption alone does not prevent deletion by credentials with that permission.

Provide setup, health-check, backup, restore, update, and rollback commands. Make setup safe to rerun. Keep secrets out of diagnostics. Document how to rotate each credential and revoke a lost device's access.

Use controlled dependency updates. Back up before migrations, run checks, and preserve a rollback release. Do not automatically update provider SDKs and restart active sessions without a maintenance policy. The agent must not restart the daemon that carries its own deployment conversation. Use SSH or the maintenance terminal.

Record service costs and provider usage where available. Show unknown usage as unknown. Configure available budget alerts and concurrency limits. Keep API billing separate from server, domain, and optional service costs. Never promise that a free tier covers the intended use without checking it.

Acceptance: after reboot, the services return and saved sessions remain readable. A restore rehearsal passes. Rollback works for a release without incompatible data changes, and the runbook explains recovery for a database migration. The health check identifies a stopped service without printing secrets.

### 15. Final verification and handover

Maintain a checklist with evidence and dates for every acceptance test above. Use automated tests for protocol ordering, replay, deduplication, permissions, path validation, and persistence. Use browser tests for navigation, the editor bridge, authentication recovery, and responsive layout. Use a small real-provider task for end-to-end validation rather than relying entirely on mocks.

The final demonstration must show this sequence:

1. I open chat on my phone and start a coding task in a disposable project.
2. I close the phone browser while the task runs.
3. I open browser VS Code and attach its agent panel to that session.
4. The transcript includes the work that happened while I was away.
5. I inspect the changed file and send a selection back to the agent.
6. I open the result through the stable preview hostname.
7. I return to the phone and see the same conversation and task state.
8. A separate unauthenticated browser cannot access any private interface.

Deliver the private repository, installed services, stable links, setup record, and recovery runbook. Distinguish verified behaviour from remaining device tests or account blockers. Keep the final handover short, with the next specific action only if one remains.

Begin with discovery and the first missing prerequisite. Continue until the authorised build and verification are complete.

## Prompt ends

## What can come later

After the specified interface works, the same agent can add push notifications, a fuller account-management interface, speech input, scheduled tasks, and connectors to other services. Those features should use the existing session store and event protocol. Notifications should distinguish completion, failure, and a request for user input, then deep-link to the relevant session without exposing private transcript text.

Provider switching needs particular care. A transcript from one provider is not another provider's native session state. An implementation should label a text-history handover honestly. Similarly, an installable phone app does not imply offline agent execution, and browser-disconnect persistence does not imply uninterrupted work through a server restart.

This document specifies the intended build and its tests. It has not been validated through a fresh installation from start to finish. Check current provider documentation during implementation and retain the test record for the installation you actually build.
