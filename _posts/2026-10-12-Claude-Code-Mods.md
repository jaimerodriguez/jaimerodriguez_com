---
layout: post
title: "Claude Code Mods"
date: 2026-10-12 00:00:00 -0700
tags:  claude-code, mods, plugins
---

The Claude Code team recently introduced mods: a mechanism for any user to customize or extend Claude Code.
This post has the TL;DR on mods, a learning example to use as your first mod, and a deep dive comparing mods to settings hooks, the less powerful predecessors that are still applicable under some circumstances.   



# The 3-minute drill

## 90-second summary

Mods are plugins, written in TypeScript or JavaScript, that turn Claude Code from a harness into a platform.
To write a mod, you register for events (still called hooks) and tell Claude Code what to do in each handler.
Hooks as we knew them before are now called settings hooks. They are still supported (and still applicable).

- **Pick a settings hook** to block, allow, or log an event with a script you already have. Settings hooks run on every surface, in any language, and a managed `PreToolUse` hook is the one control no mod can override.
- **Pick a mod** when you need what settings hooks cannot do: rewrite a prompt, replace a feature, keep state across events, add a command or tool, see per-request token usage, or draw in the user interface.
- **Performance:** a mod avoids a process spawn per event, but mod hooks run as a sequential chain while settings hooks run in parallel. Slow work inside either one adds directly to the latency of every tool call.
- **Cost:** neither is free by default. Settings hooks spend tokens through injected context and through the `prompt` and `agent` hook types. Mods can call models, fork the conversation, reroute requests, and invalidate the prompt cache, all on the user's plan or API key.
- **Security:** both run with full user permissions and neither is sandboxed. Mods add in-process reach: they see every prompt, can approve tool calls before the permission prompt, and on an unmanaged machine can approve a call that a `deny` rule would refuse.
- **Strategy:** mods are Anthropic's move from a product with extension points to a small core with a programmable shell. The direction is credible but early. 
- **My personal take:** Mods are a very powerful customization mechanism. Where they fall short for me is that I don't yet see anything that saves me from writing my own harnesses. For now, I plan to use them judiciously, so I don't add overhead to Claude Code or end up maintaining code that is just bells and whistles. 



## Tips and lessons for creating a mod

Note: I assume you are familiar with Claude Code and hooks, so I focus on the deltas. If you are not, check the sample and the long write-up below, then revisit this section.

1. **To create a mod, use the `/plugin-authoring` skill**, or tell Claude, "I want to create a new mod." <u>It is easier than starting from a sample or a clone.</u>
   - If you need it, [here is the full reference for API ($) and hook signatures](https://code.claude.com/docs/en/plugins/mods/reference). 
2. **Understand the configuration story.**
   - Every mod has a `.claude-plugin/plugin.json` manifest file where you enter the name, description, and keywords. If your mod takes parameters, they go in a `userConfig` block in the manifest.
   - There is also a `hooks/hooks.json` file where you list the JavaScript modules you want to load. Each module contains one `export function register(on, options)` where you hook up your event listeners. The 40+ events are listed in section 2.2 below.
3. **Understand the publishing story.**
   Mods are distributed through the same channels as plugins: organization sharing, marketplaces, GitHub repos, zip archives, etc. Learn about the different options in the [plugin publishing docs](https://code.claude.com/docs/en/plugins/publish).
   - The key commands during development and debugging are `claude --plugin-dir ./turn-receipt`, which loads the mod from its folder for one session, and `claude plugin marketplace add <folder>` followed by `claude plugin install turn-receipt@<marketplace> --scope user` when you want it enabled across your sessions. 
4. **If you are using the Agent SDK, know that the mod uses a different store depending on how it is loaded.** All the tallies are under `~/.claude/plugins/store`; but when I use the CLI, I get a tally in a file called `turn-receipt_<marketplace>-<hash>.json`, and when the mod is loaded from a folder (`--plugin-dir`, or the SDK's `plugins` option), I get a tally in `turn-receipt_inline-<hash>.json`. The `$.store` API keys the store by the plugin's identity; I liked it, so I did not change it. 
   - Also, update the Agent SDK to 0.2.165 or later: it is the first release whose bundled Claude Code binary loads mods. 



## My sample mod: Turn Receipt

Turn Receipt is a Claude Code mod that answers the question: what would my usage cost if I were using the API? After every turn, it prints a receipt with the token counts, split into uncached input, output, cache reads, and cache writes, along with the cost in dollars and the model that answered. If the turn spawned subagents, a second line shows how many ran, what they consumed, and what they cost, so you can see when a delegated task was the expensive part. A third line keeps a running total for the session, and the mod also keeps a tally per calendar month. Here is a sample receipt: 

```text
# The receipt beneath a turn:
- receipt · in 4 · out 26 · cache read 109.5k · cache write 1.3k · $0.054 · fable-5-1
- agents · 2 runs · in 292 · out 3.1k · cache read 314.7k · cache write 76.3k · $0.269 · turn total $0.323 · fable-5-1
- session · 4 turns · 3 agent runs · 1.5M tokens · $4.88

# A command:
/receipt-month
  ⎿  turn-receipt: 2026-10: 179 sessions
       turns 2361 | agent runs 1804 | $381.41
       tokens 158.8M: in:3M | out:4.9M | cache read:137.4M | cache write:13.4M
```



**My Sample use cases:**

1. I do a lot of code-generation comparisons on ports or on the creation of small desktop and mobile apps. I often compare Claude Code and Codex on quality, speed, effort, and cost. Turn Receipt gives me insights I was collecting manually (or via CLAUDE.md).
2. I do manual tweaking and evals across multi-agent systems: the canonical "What if I swap this subagent for a less capable model? How will my accuracy, latency, or cost change?" Turn Receipt provides data for each.
3. If you want to know how much Claude Code you are using on your subscription, Turn Receipt can give you a good overview.


**Technical details:**

Token counts come from the `turn.complete` event, and dollars come from the same session ledger that `/cost` reads, so there is no pricing table to maintain. The one trick is the subagent split: the ledger is a single figure, so the mod reads it at each spawn and at each turn's end and credits every movement to the loop that closed the window.

The mod has persistence. Each session's totals are saved as one small record, written every ten turns (and at session end and on every `/receipt-*` command) rather than on every event, and read back on resume. Each save also banks the new figures into a per-month tally, so `/receipt-month` still works after old session records are trimmed.

The mod has four commands: `/receipt` shows the current session, `/receipt-total [days]` sums the sessions saved in the last N days (14 by default), `/receipt-month [YYYY-MM]` shows a calendar month's tally, and `/receipt-clear` drops the session records but keeps the months.

You can find source code and deployment instructions at [https://github.com/jaimerodriguez/claude-code-plugins/](https://github.com/jaimerodriguez/claude-code-plugins/).
Feedback and improvement suggestions are very welcome. 



---

# Diving Deeper into Mods and Settings hooks

The write-up might be useful to anyone who wants an in-depth comparison or a one-stop reference for mods & hooks. 
It is  AI aggregated, but I reviewed and edited it. 

### 1. Comparison between Mods and Settings hooks 

|                      | Settings hook                                                | Mod                                                          |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| What it is           | A shell command, HTTP endpoint, MCP tool call, LLM prompt, or subagent that Claude Code runs on a lifecycle event [6] | Functions in a plugin that Claude Code calls in its own process when an event happens [1] |
| Where the code runs  | Outside Claude Code: a child process, a remote server, or a model call | Inside Claude Code. Installed mods share one worker thread [7] |
| Contract             | JSON on stdin or POST body in; exit code and JSON on stdout out [6] | `on(event, matcher, async ($, e, next) => ...)`. Call `next(e)` to pass on, pass a changed copy to rewrite, or return a result to answer [2] |
| Language             | Anything that can read stdin or serve HTTP                   | JavaScript or TypeScript ES module, loaded directly with no build step [8] |
| Where it is declared | `settings.json` (user, project, local, managed), plugin `hooks/hooks.json`, skill or subagent frontmatter [6] | A plugin: `plugin.json`, `hooks/hooks.json` with a `modules` entry, and the hooks module [3] |
| Outside access       | Whatever the script can do as the user                       | Only through the mods API `$` (files, processes, network, model, UI). The module has no Node.js APIs or timers of its own [4] |
| Status               | Mature. "Nothing about them is deprecated" [5]               | Shipped October 1, 2026 in v2.1.287, on by default. API may change between releases [1][16] |

## 2. Events each one listens to

### 2.1 Settings hooks: 33 lifecycle events

Source: Hooks reference [6]. "Can block" means exit code 2 or a JSON decision stops the action. Matchers filter on the field shown.

| Event                                | Fires when                                                   | Can block                                        | Matcher filters on                    |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------- |
| `SessionStart`                       | A session begins or resumes                                  | No (adds context)                                | startup, resume, clear, compact, fork |
| `Setup`                              | `--init-only`, or `--init` / `--maintenance` with `-p`       | No                                               | init, maintenance                     |
| `InstructionsLoaded`                 | A CLAUDE.md or rules file is loaded into context             | No                                               | load reason                           |
| `UserPromptSubmit`                   | A prompt is submitted, including scheduled tasks and cross-session messages | Yes                                              | none                                  |
| `UserPromptExpansion`                | A typed command expands into a prompt                        | Yes                                              | command name                          |
| `PreToolUse`                         | Before a tool call executes                                  | Yes (allow, deny, ask, defer; can rewrite input) | tool name                             |
| `PermissionRequest`                  | A tool call needs a permission decision                      | Via JSON decision                                | tool name                             |
| `PermissionDenied`                   | Auto mode denies a tool call                                 | No (can allow retry)                             | tool name                             |
| `PostToolUse`                        | After a tool call succeeds                                   | No (feedback; can rewrite output)                | tool name                             |
| `PostToolUseFailure`                 | After a tool call fails                                      | No (feedback)                                    | tool name                             |
| `PostToolBatch`                      | After a batch of parallel tool calls, before the next model call | Yes                                              | none                                  |
| `MessageDisplay`                     | While assistant text streams to the screen                   | No (display-only rewrite)                        | none                                  |
| `Notification`                       | Claude Code sends a notification                             | No                                               | notification type                     |
| `SubagentStart` / `SubagentStop`     | A subagent is spawned / finishes                             | No / Yes                                         | agent type                            |
| `TaskCreated` / `TaskCompleted`      | A task is created / marked complete                          | Yes / Yes                                        | none                                  |
| `TeammateIdle`                       | An agent-team teammate is about to go idle                   | Yes                                              | none                                  |
| `Stop` / `StopFailure`               | Claude finishes responding / the turn ends on an API error   | Yes / No                                         | none / error type                     |
| `PreCompact` / `PostCompact`         | Before / after context compaction                            | Yes / No                                         | manual, auto                          |
| `PreModelSwitch` / `PostModelSwitch` | Before / after the session model changes                     | Yes / No                                         | target model                          |
| `ConfigChange`                       | A configuration file changes during a session                | Yes (not policy settings)                        | config source                         |
| `CwdChanged` / `DirectoryAdded`      | Working directory changes / a directory is added             | No                                               | none / how added                      |
| `FileChanged`                        | A watched file changes on disk                               | No                                               | filenames to watch                    |
| `WorktreeCreate` / `WorktreeRemove`  | A worktree is created / removed                              | Yes (any non-zero exit)                          | none                                  |
| `Elicitation` / `ElicitationResult`  | An MCP server requests user input / the user responds        | Yes                                              | MCP server name                       |
| `SessionEnd`                         | A session terminates                                         | No                                               | end reason                            |

Handler types: `command`, `http`, `mcp_tool`, `prompt` (single-turn model evaluation), and `agent` (a subagent with Read, Grep, and Glob, experimental). `SessionStart` supports only `command` and `mcp_tool` [6].

### 2.2 Mods: about 45 named events in nine groups

Source: Mods reference, as of v2.1.289 [3]. Every hook can observe. The last column lists what else it can return.

| Group                         | Events                                                       | What a mod hook can do                                       |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Tools                         | `tool.call`, `tool.check`, `tool.describe`                   | Deny, rewrite arguments, retry, or return a result without running the tool. `tool.check` can replace the permission decision (allow, ask, deny). `tool.describe` rewrites the description Claude reads |
| Prompts and what Claude reads | `prompt.submit`, `prompt.fill`, `prompt.suggest`, `prompt.edit`, `prompt.compose`, `prompt.section`, `prompt.context`, `prompt.attachment`, `skill.prompt`, `attribution.text` | Rewrite or drop a prompt, add context, reorder or remove **system prompt sections**, change first-message context, reword reminders, skill text, and commit attribution |
| Commands and config           | `command.run`, `command.describe`, `config.set`, `config.describe` | Answer or replace a `/command`, hide or relabel commands and `/config` rows, veto a config change |
| Turns                         | `turn.start`, `turn.step`, `turn.complete`                   | `turn.step` fires for **every model request**: read token usage, send the request to a different model or effort, or answer without calling the model |
| Session                       | `session.start`, `session.end`, `session.compact`, `session.receive`, `session.send`, `session.append`, `session.attach`, `session.detach`, `session.measure` | Skip compaction, consume or block cross-session messages, rewrite each transcript row before it is stored, react to usage changes |
| Subagents                     | `agent.offer`, `agent.spawn`                                 | Withhold a subagent type, choose its model, or deny the spawn |
| Interface                     | `ui.render`, `ui.resolve`, `ui.press`, `ui.input`, `ui.select`, `ui.focus`, `ui.scroll`, `ui.close`, `ui.message`, `ui.fault` | Draw panes and the band above the prompt, restyle or replace 13 render sites (spinner, tool rows, messages, the AskUserQuestion dialog), respond to buttons and inputs |
| Other mods                    | `plugin.register`, `engine.create`                           | Refuse another mod at load based on the events and API calls it uses. Add or withhold parts of the mods API for other mods |
| Telemetry                     | `telemetry.log`, `telemetry.mark`                            | Observe or deny usage records                                |

Two wildcard families extend that list:

- **`classic.<Event>`**: every settings-hook event in section 2.1 is also a mod event, such as `classic.Stop` or `classic.PostToolUse`. The event payload is the same JSON a settings hook reads on stdin [2].
- **Mods API calls**: every `$` method is itself an event (`fs.read`, `http.fetch`, `model.complete`, `process.run`, and so on). A mod earlier in the chain can observe, rewrite, or deny another mod's call. This is how an organization audits or restricts mods [3][4].

### 2.3 Rosetta table: the same job in each mechanism

This mapping is my own synthesis from the two references, not an official table. Treat the pairings as approximate.

| Job                                                          | Settings hook                                                | Mod                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Gate a tool call                                             | `PreToolUse`                                                 | `tool.call` (before `next`)                                  |
| Decide permission                                            | `PermissionRequest`                                          | `tool.check` (runs after rules and hooks; can replace their answer) |
| React after a tool                                           | `PostToolUse`, `PostToolUseFailure`                          | `tool.call` (after `await next(e)`), with retry              |
| Inspect a prompt                                             | `UserPromptSubmit` (block or add context; **cannot replace the text**) | `prompt.submit` (rewrite, add context, or drop)              |
| Session start and end                                        | `SessionStart`, `SessionEnd`                                 | `session.start` (not after `/clear` or `/resume`), `session.end` |
| Compaction                                                   | `PreCompact` (block)                                         | `session.compact` (skip)                                     |
| End of a turn                                                | `Stop` (can force Claude to continue)                        | `turn.complete` (observe, add a line); `classic.Stop` for the rest |
| Subagent lifecycle                                           | `SubagentStart`, `SubagentStop`                              | `agent.offer`, `agent.spawn` (choose model, deny)            |
| Change displayed text                                        | `MessageDisplay` (assistant text only)                       | `ui.render` on any render site                               |
| Per-request model usage, model routing                       | Not available                                                | `turn.step`                                                  |
| Edit the system prompt                                       | Not available                                                | `prompt.compose`, `prompt.section`                           |
| Draw UI, add commands or tools                               | Not available                                                | `ui.*`, `$.command.register`, `$.tool.register`              |
| File watch, cwd change, worktrees, MCP elicitation, config change, notifications | Native events                                                | Only through `classic.*`                                     |

## 3. Where each runs

| Surface                                                      | Settings hooks                                               | Mod hooks run                              | Mod UI appears                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------ |
| `claude` in a terminal (incl. IDE integrated terminals, JetBrains plugin) | Yes                                                          | Yes                                        | Yes                                                    |
| Desktop app, Code tab                                        | Yes                                                          | Yes                                        | Yes, except terminal-only elements (`Raster`, `Image`) |
| Desktop app, WSL session                                     | Not stated                                                   | No (plugins unavailable)                   | No                                                     |
| VS Code extension chat panel                                 | Yes                                                          | Yes                                        | **No**                                                 |
| `claude -p` and the Agent SDK                                | Yes                                                          | Yes                                        | No                                                     |
| Remote Control (claude.ai, mobile)                           | Yes, on your machine                                         | Yes, on your machine                       | Only in the terminal on your machine                   |
| Cloud session (Claude Code on the web)                       | Repo `.claude/settings.json` hooks and managed settings. <br />Not `~/.claude/settings.json` | Only for a plugin that reaches the session | No                                                     |

Sources: mods surface table [1], hooks reference [6], cloud environments [13].

> **WATCH OUT:** Cloud sessions are the weak spot for mods today. The mods overview says hooks run there "for a plugin that reaches the cloud session", but the cloud carry-over table lists both repo-declared plugins and user-enabled plugins as **not** carried over [1][13]. I could not find a documented route by which an ordinary user's mod reaches a cloud session. Settings hooks committed to the repo do carry over. If a control must work in cloud sessions, use a settings hook.

> **INSIGHT:** A mod that draws should degrade on purpose. The docs suggest checking the surface and falling back to a transcript line or a command reply where nothing draws [1]. For your "light UI" goal this is good news: `$.ui.log`, `$.ui.status`, and command replies are the portable subset.

## 4. Capabilities head to head

| Criterion                            | Settings hook                                                | Mod                                                          |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Observe events                       | Yes                                                          | Yes, including a `*` wildcard                                |
| Block                                | Yes on blockable events (exit 2 or JSON decision)            | Yes: return `{ deny }`, `{ drop }`, `{ refuse }` and so on   |
| Rewrite inputs                       | Tool input (`updatedInput`). Prompts cannot be replaced      | Any event field, by passing a changed copy to `next`         |
| Rewrite outputs                      | Tool output (`updatedToolOutput`), displayed assistant text  | Any result, by changing what `next` returned                 |
| Replace behavior                     | No (except `WorktreeCreate`)                                 | Yes: answer the event without calling `next`                 |
| Wrap (before and after in one place) | No, two separate hooks and no shared memory                  | Yes, one function around `next(e)`                           |
| State across events                  | None. Each run is a fresh process; persist to files yourself | Module variables, reactive `$.state`, and `$.store` (4 MiB, shared by all sessions on the machine) |
| Background work                      | `async: true` command hooks, one process per event           | `$.clock.every` / `$.clock.after` timers that outlive an event |
| Add commands or tools                | No                                                           | `$.command.register`, `$.tool.register`                      |
| Call a model                         | `prompt` and `agent` hook types                              | `$.model.complete`, `$.model.fork`, `$.model.classify`, `$.agent.spawn` |
| User interface                       | `systemMessage`, `statusMessage`, terminal notifications     | Panes, band above the prompt, buttons, inputs, toasts, status line, restyled built-in rows |
| Execution model                      | All matching hooks run **in parallel**                       | One **sequential** middleware chain per event                |
| Time limit                           | 600 s default (30 s on `UserPromptSubmit`, 10 s on `MessageDisplay`) | 10 s of the hook's own running time (50 ms for `prompt.edit`) |
| On timeout or crash                  | Fails **open**: the action proceeds                          | Fails **open**: the hook is skipped, unless you attach `.catch` |
| Inspect before install               | Read the script                                              | `claude plugin validate` lists `hooks:` and `calls:` without running the mod |
| Test                                 | Pipe JSON into the script; no framework                      | `claude plugin test`: no session, sign-in, or network; events can be raised and API calls stubbed |
| Portability of your code             | High. A script is a script                                   | Low. `$` exists only in Claude Code                          |

Sources: [1][2][3][4][6][9][10].

## 5. Advantages and disadvantages

### Settings hooks

#### Advantages

- **Runs everywhere.** Same events in the terminal, IDE extensions, Desktop app, and cloud sessions [6].
- **Language-agnostic and reusable.** Wrap the linter, secret scanner, or policy service you already own. An HTTP hook centralizes policy in one service.
- **Process isolation.** A hook that crashes or leaks memory cannot take down the session.
- **Strongest precedence.** A managed `PreToolUse` hook runs before any mod sees the call, its block is final, and it runs again if a mod rewrites the call [5].
- **Parallel by default,** with an `if` filter that avoids spawning a process when the pattern does not match [6].
- **Stable and declarative.** Easy to diff and audit in a settings file.

#### Disadvantages

- **No memory.** Every invocation starts cold. Correlating a `PreToolUse` with its `PostToolUse` means writing your own state files.
- **Narrow verbs.** Mostly allow, block, or add context. Hooks "can't rewrite events, draw new UI, or replace features" [15].
- **Process spawn per event,** plus JSON serialization, on the hottest path in the agent loop.
- **Fail-open footguns.** Exit code 1 does not block; only exit 2 does. A timed-out hook does not block. A mistyped script path "leaves the gate silently disabled" [6].
- **Blind to the model loop.** No per-request token usage, no system prompt access, no model routing.

### Mods

#### Advantages

- **Full middleware semantics.** Observe, rewrite, answer, wrap, and retry in one function, with shared state [2].
- **Reaches the model loop.** `turn.step` exposes usage per request and can reroute model or effort. `prompt.compose` exposes the system prompt [3].
- **First-class UI on two surfaces** from one codebase, with lightweight options (`$.ui.status`, `$.ui.toast`, `$.ui.log`) [4].
- **Statically inspectable.** All outside access goes through `$`, so Claude Code can list a mod's capabilities without running it and refuses mods it cannot analyze [5].
- **Governable by code.** Tiers (`prependPlugins`, user, `appendPlugins`, built-in), `plugin.register`, and API-call interception let an organization write policy as a mod [5].
- **Fast inner loop.** Hot reload on save, a no-session test runner, and Claude can write the mod for you in-session [8][10].

#### Disadvantages

- **Bigger blast radius.** A mod sees every prompt and tool call, reads secrets, can submit prompts as you, can approve tool calls before you are asked, and can spend your usage [1].
- **Weaker guarantees outside managed machines.** Without managed settings or a Team or Enterprise sign-in, a mod can approve a call that a `deny` rule refuses [11].
- **Shared fate.** Installed mods share one worker thread. If it crashes three times without a culprit, every installed mod is unloaded, **including the organization's policy mods** [7].
- **Sequential chain.** Each mod on an event adds its own latency, and an earlier mod can swallow an event before a later audit mod sees it [2][21].
- **Young API.** "The API can change between releases" and the generated types are "the authority for your version" [16]. No semver commitment was found in the design thread [21].
- **No UI in VS Code, `-p`, the SDK, or cloud** [1].
- **Vendor lock-in.** Mod code is not portable to any other harness.

## 6. Ordering, precedence, and failure semantics

This is the section most likely to matter in an enterprise design review. For a single tool call, the documented order is [2][5][11]:

1. **Managed `PreToolUse` hooks.** A block here is final. No mod sees the call.
2. **Organization mods in `prependPlugins`**, including the built-in guard `sec-default` where it loads.
3. **User-installed mods**, then organization mods in `appendPlugins`, then other built-in mods. Each sees the event before the next and the result after it.
4. **Claude Code's own behavior:** `PreToolUse` hooks from every other settings file and from plugins, then permission rules.
5. **`tool.check`:** a mod can return a different decision after the rules and hooks have decided.

Three consequences follow:

- A mod that answers `tool.call` without calling `next` keeps user, project, and plugin `PreToolUse` hooks from running at all.
- A mod's `tool.check` can approve a call that an `ask` rule would prompt for, or that a non-managed `PreToolUse` hook blocked. In auto mode, a call the mod approves runs without a classifier check [11].
- Deny rules hold over a mod only where the guard loads (managed settings, or a Team or Enterprise sign-in). Deny rules never cover a mod's own `$.fs` and `$.process` calls: with `Read(.env)` denied, a mod can still read that file [5].

> **WATCH OUT:** **Both mechanisms fail open.** A settings hook that times out does not block. A mod hook that throws, times out, or returns the wrong shape is skipped and the next handler runs. For a mod guard, attach `.catch` to fail closed; since v2.1.290, `claude plugin validate` lists each gating hook and whether it has one [2][18]. Community testers raised the obvious attack: make a deny guard slow and it stops guarding [21].

> **INSIGHT:** There is a third family worth knowing: **Agent SDK callback hooks**. Unlike the other two, an SDK callback that times out on `PreToolUse` or `UserPromptSubmit` **blocks**, "because a callback there can be acting as a policy gate that must not fail open" [6]. If you build a product on the SDK, that is the fail-closed option.

## 7. Performance implications

Anthropic publishes limits, not latency benchmarks. The table separates documented facts from second-hand measurements.

| Dimension          | Settings hook                                                | Mod                                                          |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Per-event overhead | Process spawn, shell startup, JSON in and out. HTTP hooks add a network round trip. The `if` filter skips the spawn when the pattern misses [6] | A function call on a worker thread. No spawn. Cost is whatever your code does [7] |
| Concurrency        | All matching hooks run in parallel; identical handlers are de-duplicated [6] | Sequential chain. N mods on `tool.call` means N handlers in series on every tool call [2] |
| Blocking the agent | Synchronous unless `async: true`. A `UserPromptSubmit` hook "blocks model processing until it completes" [6] | Every `tool.call` and `turn.step` hook sits on the hot path. A loop that never awaits stalls the shared worker [7] |
| Budgets            | 600 s default; 30 s `UserPromptSubmit`; 10 s `MessageDisplay`; 1.5 s shared at `SessionEnd` | 10 s own time; 50 ms `prompt.edit`; 1 s `.catch`; 1.5 s shared at `session.end` [3] |
| Startup            | `SessionStart` hooks run in the background, but Claude's first response waits for them [6] | Claude Code waits for `session.start` hooks before the first prompt [4] |
| Rendering          | Not applicable                                               | Redraws throttled to 10 per second (30 in the terminal for visible regions) and coalesced [3] |
| Prompt cache       | Injected context is added at the point the hook fired        | Text from `prompt.section`, `prompt.context`, or `skill.prompt` that changes between requests **invalidates the prompt cache** [2] |

#### Measurements, with caveats

- **Sequential vs parallel.** A pre-release tester in the design thread reportedly measured eight 300 ms function hooks at 2,427 ms against 640 ms for the same work as parallel command hooks. I read this in a third-party guide that cites the thread and could not locate the original comment [25].
- **Load time.** Another pre-release tester reportedly saw a `/diff`-sized module evaluated 3 to 4.4 s after process start while the prompt was live at 1.7 s, and about 320 ms for a `ui.render` repaint (v2.1.278) [21]. Second-hand, pre-release.
- **Stability.** The first five days of changelog include fixes for endless redraw loops, a redraw stall on long non-Latin text, and session-ending interface errors caused by a mod's `ui.render` output [18].

> **INSIGHT:** The intuition "in-process means faster" is only half right. A single cheap mod beats a single cheap command hook because it skips the spawn. But stacking is where they diverge: hooks fan out, mods queue. Five community mods that each hook `tool.call` are five serial stops on every tool call. Budget mod count the way you would budget browser extensions.

Practical guidance: keep mod hooks to bookkeeping and return `next(e)` immediately; move slow work to `$.clock` timers; use matchers so hooks fire only where needed; and never put dynamic text in `prompt.section`.

## 8. Cost implications

| Cost driver       | Settings hook                                                | Mod                                                          |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Baseline          | Zero tokens for `command`, `http`, and `mcp_tool` hooks that add no context | Zero tokens for a mod that only observes or draws            |
| Context injection | `additionalContext` and plain stdout on some events, capped at 10,000 characters per string [6] | `context` on `prompt.submit`, plus system prompt and attachment rewrites |
| Model calls       | `prompt` hooks call a model per event (default: the background model). `agent` hooks run a subagent for up to 50 turns [6] | `$.model.complete` (1,024 output tokens default, up to 64,000), `$.model.fork`, `$.model.classify`, `$.agent.spawn`, `$.prompt.submit` (starts whole turns) [3][4] |
| Model routing     | `PreModelSwitch` can only block a switch                     | `turn.step` can send any request to a different model or effort; `agent.spawn` picks a subagent's model [3] |
| Cache effects     | Low risk                                                     | Dynamic system prompt text invalidates the cache [2]. Rerouting models mid-session likely forfeits the cached prefix (my inference: caches are per model) |
| Who pays          | The session's plan or API key                                | "These calls use the user's plan or API key" [4]             |
| Visibility        | Hook model usage rolls into session usage                    | `/usage` attributes by plugin among other things. The costs page does not mention mods [14] |

Two points deserve emphasis.

- **Mods are the best cost instrument Claude Code has shipped.** `turn.step` returns input, output, cache-read, and cache-write tokens per request, and `$.session.usage()` returns context fill, rate limits, and cost [2][3]. Hooks see none of this. If a customer asks "where do my tokens go?", the answer is now a 50-line mod.
- **Mods are also the easiest way to spend without noticing.** `$.model.fork` is cheap because it reuses the cached conversation, but a mod that forks on every `turn.complete` (as the community "next steps" mod does) adds a model call to every turn [27]. The built-in "You should know" mod runs a side agent and is off by default [1].

> **INSIGHT:** Cache invalidation is the hidden line item. One community mod author recorded a $6.61 estimate to rewrite the cache on a 330k-token session after it went cold (author's own recording, API-equivalent pricing) [28]. A mod that perturbs the system prompt every request would pay a version of that tax continuously.

## 9. Security and governance

Neither mechanism is sandboxed. "Command hooks execute shell commands with your full user permissions" [6], and "a mod is code that runs with your permissions, inside Claude Code" [1]. The differences are in reach and in control.

| Control                             | Settings hooks                                               | Mods                                                         |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Workspace trust                     | Held back until the folder is trusted in interactive sessions. `-p` treats the folder as trusted [6] | No mod loads until the trust prompt is answered [5]          |
| Stop everything the user added      | `allowManagedHooksOnly`                                      | `allowManagedModsOnly` (mods only; user hooks keep working) or `allowManagedHooksOnly` (both) [5][12] |
| Kill switch                         | `disableAllHooks`                                            | `disableAllHooks`, `--safe-mode`, and a remote switch: "Anthropic has turned installed mods off remotely" is a documented state [7] |
| Default guard                       | Not applicable                                               | `sec-default` loads first on managed machines and Team or Enterprise sign-ins. It protects managed hooks, the system prompt, managed instructions, and managed MCP servers from user mods [5] |
| Network policy                      | Script decides                                               | Org web-fetch policy covers `$.http.fetch`, but not a program started with `$.process.run` [5] |
| Restrict install source             | Not applicable to settings files                             | Marketplace allowlists and `disableSideloadFlags` [5]        |
| What counts as "the organization's" | Hooks in managed settings                                    | Only a plugin enabled in managed settings **and** loaded in place from an admin-owned local directory. A plugin from GitHub, git, URL, or npm counts as a user's mod even when managed settings enable it [5] |

#### What the first week showed

- **v2.1.289 (Oct 3):** fixed "a deny or ask rule on a nested part of a compound shell command not holding over a user-installed mod's approval on managed machines" [18].
- **v2.1.290 (Oct 5):** fixed "a case where a user-installed mod could make an organization's guard skip its check" and one where it "could get an organization's plugin unloaded" [18].
- **Independent researchers** (pre-launch builds) described credential theft through `$.fs.read` plus `$.http.fetch`, UI spoofing through redrawn dialogs, and code that changes after review [22][23]. One argues that in-process logic is indistinguishable from Claude Code to endpoint security tools [23]. These are vendor blogs with products to sell; the mechanics match the documented capability list.
- **Rollout speed:** one outlet reports the feature moved from opt-in flag to on-by-default in 29 days [24]. I could not confirm the early-access date from the changelog.

> **WATCH OUT:** For an enterprise rollout, do not treat a policy mod as a hard control yet. It fails open by default, it is unloaded by `--safe-mode`, it is unloaded if the shared worker crashes three times, and the guard had two bypass fixes in week one. Put non-negotiables in managed deny rules and managed `PreToolUse` hooks, and use policy mods for audit and for graded policy [5][7][18].

## 10. The strategic value of mods

Anthropic is putting weight behind mods. 

- **Six built-in mods ship in the product:** `/diff` (1,036 lines), `AGENTS.md` loading (305 lines), the `sec-default` guard (115 lines), telemetry (369 lines), plugin-authoring, and the opt-in "You should know" side agent. Source for four of them is public [1][19].
- **Stated intent:** "We plan to move more built-in features to mods over time, so you can pare Claude Code down to a small core and add back only what you want" [15].
- **Qualification:** of the six, one holds a skill and no mod code, two are infrastructure (guard, telemetry), and three are user-facing features. The "small core" is a direction, not yet a description. The interesting evidence is `/diff`: a real interactive feature, rebuilt on the public API, that you can disable or replace.

### 10.2 Why the bet matters long-term

#### 1. It turns the harness into a platform

Hooks let you react to Claude Code. Mods let you change it. When the vendor builds its own features on the same API it gives you, the API tends to get real investment, because gaps hurt the vendor first. This is the pattern behind editors and browsers that became ecosystems. The security policy itself being a mod (`sec-default`) is the strongest signal of commitment: the enforcement layer is written in the extension model.

#### 2. It relieves the roadmap

The launch post is explicit that developers wanted control "without waiting for us to ship a feature" [15]. Mods push long-tail requests (a CI pane, a cost band, a different question dialog) to the community, and the popular ones become free product research. An independent catalogue counted 1,752 public mods three days after launch, though that scan includes early-access work and its count is self-reported [26].

#### 3. It makes the tool self-modifying

Claude can write a mod, load it, and hot-reload it in the session that asked for it [8][15]. That collapses the cost of personal tooling to a sentence. Over time this points to interfaces generated per user and per task, which no fixed product UI can match. It is also the reason the trust model matters so much: the same loop means a session can extend its own harness, gated only by a per-session approval.

#### 4. It gives enterprises a programmable control plane

Tiers, `plugin.register`, and API-call interception mean an organization can express policy and audit as code that runs ahead of everything a user installs [5]. That is a richer story than allowlists, and it is a clear enterprise differentiator if the guarantees harden.

#### 5. It is a proprietary surface in a stack Anthropic has otherwise opened

MCP and Skills travel across vendors. Mods do not: `$` exists in one runtime, a concern raised in the design thread [21]. Every mod a team writes raises its switching cost. From Anthropic's side that is a moat; from a customer's side it is lock-in to weigh. This is the point I would challenge hardest in any "standardize on mods" recommendation.

#### 6. It unifies the surfaces

One element set renders in the terminal and the Desktop app. If VS Code and cloud drawing follow, mods become the cross-surface UI layer for Claude Code. Today that is an inference from the design, not an announcement.

### 10.3 What could undermine it

- **API churn.** Community mods already warn that builds on other versions "may hit load errors" [31]. An ecosystem cannot compound on an API that breaks monthly.
- **A supply-chain incident.** Unsandboxed, on by default, installable from any marketplace: one commentator calls it "early npm, not App Store" [32]. A single credential-stealing mod could force a retreat to allowlists.
- **Fail-open semantics** make mods a poor fit for the compliance buyers who would value the control plane most.
- **Supportability.** A heavily modded Claude Code is a different product on every machine. `--safe-mode` exists because of this.
- **Performance drift** as users stack mods on hot-path events.

> **INSIGHT:** My read: the strategic value is real and asymmetric. For individuals and small teams, mods are immediately worth learning because the downside is contained and the leverage is high. For regulated enterprises, the right posture this quarter is `allowManagedModsOnly` plus a small set of reviewed internal mods, while hooks and deny rules keep doing the enforcement. Revisit when the API carries a stability commitment and mods get a sandbox or permission manifest.

## 11. Decision guide

| If you need to...                                            | Use                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Enforce a rule that must hold, including in cloud sessions   | Managed deny rule or managed `PreToolUse` hook               |
| Run an existing linter, formatter, or scanner after edits    | Settings hook (`PostToolUse`)                                |
| Send events to a central policy or logging service           | HTTP settings hook                                           |
| React to file changes, cwd changes, worktrees, or MCP elicitation | Settings hook (native events)                                |
| Rewrite a prompt, or redact tool output with shared context  | Mod                                                          |
| Track anything across events (counts, timings, per-turn summaries) | Mod                                                          |
| See token usage per request, or route requests by model      | Mod (`turn.step`)                                            |
| Add a `/command` that runs instantly without a model turn    | Mod                                                          |
| Show status, a pane, or a confirmation dialog                | Mod                                                          |
| Audit or restrict what other mods do                         | Policy mod in `prependPlugins`, backed by managed settings   |
| Ship one package with instructions, tools, guards, and UI    | A plugin holding a skill, an MCP server, settings hooks, and a mod |

## 12. Other sample mods worth reading

Ordered by how useful they are for learning. Run `claude plugin validate <dir>` on any of them before loading, and prefer `--plugin-dir` for a one-session trial [1].

### Official samples and built-ins

| Mod            | What it does and why read it                                 | Link                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| token-weather  | Context-window forecast in the band above the prompt. **122 lines.** The best first read for a light, informative mod: `session.start`, `turn.complete`, `ui.render`, `$.session.usage` [20] | [playground/token-weather](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods/token-weather) |
| replay-theater | `/replay` steps through the last turn's file edits. 249 lines. Shows commands, recording across events, and a pane [20] | [playground/replay-theater](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods/replay-theater) |
| blast-radius   | Holds risky shell commands and shows what they would change. 528 lines. Shows a held tool call; its README admits it is "a safety net, not a permission system" [20] | [playground/blast-radius](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods/blast-radius) |
| sec-default    | The built-in guard. **115 lines** hooking 15 events. The model for a policy mod, and the fastest way to understand the trust tiers [19] | [claude-code/mods/sec-default](https://github.com/anthropics/claude-code/tree/main/mods/sec-default) |
| agents-md      | Loads `AGENTS.md` as project instructions. 305 lines. Shows `prompt.context` and a `userConfig` option [19] | [claude-code/mods/agents-md](https://github.com/anthropics/claude-code/tree/main/mods/agents-md) |
| diff           | The `/diff` pane. 1,036 lines. The reference for a full interactive feature built on the public API [19] | [claude-code/mods/diff](https://github.com/anthropics/claude-code/tree/main/mods/diff) |
| telemetry      | Adds methods other mods can call, via `engine.create` [19]   | [claude-code/mods/telemetry](https://github.com/anthropics/claude-code/tree/main/mods/telemetry) |

##  Bibliography

- **[1]** Anthropic. "Mods overview." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/overview](https://code.claude.com/docs/en/plugins/mods/overview)
- **[2]** Anthropic. "React to events with a mod." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/events](https://code.claude.com/docs/en/plugins/mods/events)
- **[3]** Anthropic. "Mods reference" (as of v2.1.289). Claude Code Docs. [code.claude.com/docs/en/plugins/mods/reference](https://code.claude.com/docs/en/plugins/mods/reference)
- **[4]** Anthropic. "Use the mods API." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/api](https://code.claude.com/docs/en/plugins/mods/api)
- **[5]** Anthropic. "Manage mods for your organization." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/admin](https://code.claude.com/docs/en/plugins/mods/admin)
- **[6]** Anthropic. "Hooks reference." Claude Code Docs. [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks)
- **[7]** Anthropic. "Troubleshoot a mod." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/troubleshoot](https://code.claude.com/docs/en/plugins/mods/troubleshoot)
- **[8]** Anthropic. "Create a mod." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/create](https://code.claude.com/docs/en/plugins/mods/create)
- **[9]** Anthropic. "Draw in the interface." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/interface](https://code.claude.com/docs/en/plugins/mods/interface)
- **[10]** Anthropic. "Test a mod." Claude Code Docs. [code.claude.com/docs/en/plugins/mods/test](https://code.claude.com/docs/en/plugins/mods/test)
- **[11]** Anthropic. "Permissions: Extend permissions with hooks." Claude Code Docs. [code.claude.com/docs/en/permissions](https://code.claude.com/docs/en/permissions)
- **[12]** Anthropic. "Settings reference" (`disableAllHooks`, `allowManagedHooksOnly`). Claude Code Docs. [code.claude.com/docs/en/settings-reference](https://code.claude.com/docs/en/settings-reference)
- **[13]** Anthropic. "Cloud environments: What carries over from your setup." Claude Code Docs. [code.claude.com/docs/en/cloud-environments](https://code.claude.com/docs/en/cloud-environments)
- **[14]** Anthropic. "Manage costs." Claude Code Docs. [code.claude.com/docs/en/costs](https://code.claude.com/docs/en/costs)
- **[15]** Anthropic. "Customize Claude Code with mods." Claude blog, October 1, 2026. [claude.com/blog/claude-code-mods](https://claude.com/blog/claude-code-mods)
- **[16]** Osmani, Addy. "Getting started with Claude Code mods." claude.dev, October 1, 2026. [claude.dev/blog/getting-started-with-claude-code-mods](https://claude.dev/blog/getting-started-with-claude-code-mods/)
- **[17]** Anthropic. "Claude Code mods" gallery. claude.dev. [claude.dev/mods](https://claude.dev/mods/)
- **[18]** Anthropic. Claude Code CHANGELOG, v2.1.287 to v2.1.291 (October 1 to 6, 2026). [github.com/anthropics/claude-code/blob/main/CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- **[19]** Anthropic. Built-in mods source (`diff`, `agents-md`, `sec-default`, `telemetry`). [github.com/anthropics/claude-code/tree/main/mods](https://github.com/anthropics/claude-code/tree/main/mods)
- **[20]** Anthropic. Sample mods (`token-weather`, `blast-radius`, `replay-theater`). [github.com/anthropics/claude-code-playground/tree/main/claude-code/mods](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods)
- **[21]** Anthropic (@poteat) and contributors. "Mods - make Claude 10x more extensible." GitHub issue #91870, September 2026. [github.com/anthropics/claude-code/issues/91870](https://github.com/anthropics/claude-code/issues/91870)
- **[22]** Melzer, Ehud. "Claude Code function hooks security." Pluto Security, September 22, 2026. [pluto.security/blog/claude-code-function-hooks-security](https://pluto.security/blog/claude-code-function-hooks-security/)
- **[23]** Fischer, Yuval. "Claude mods: the new attack surface, built in." Dash Security, September 25, 2026. [dash.security/blog/claude-mods-the-new-attack-surface-built-in](https://dash.security/blog/claude-mods-the-new-attack-surface-built-in)
- **[24]** Park, Owen. "Claude Code mods: unsandboxed plugins, on by default." Threatfrontier, October 2, 2026. [threatfrontier.com/articles/claude-code-mods-unsandboxed-plugins-on-by-default](https://threatfrontier.com/articles/claude-code-mods-unsandboxed-plugins-on-by-default)
- **[25]** Daniel, Agrici. "Claude Code mods guide." October 3, 2026. [agricidaniel.com/blog/claude-code-mods-guide](https://agricidaniel.com/blog/claude-code-mods-guide)
- **[26]** karanb192. "Awesome Claude Code Mods" and catalogue. [github.com/karanb192/awesome-claude-code-mods](https://github.com/karanb192/awesome-claude-code-mods) · [mods.aidojo.si](https://mods.aidojo.si/)
- **[27]** Shihipar, Thariq. "next-steps" mod. [github.com/anthropics/claude-plugins-community/tree/main/next-steps](https://github.com/anthropics/claude-plugins-community/tree/main/next-steps)
- **[28]** karanb192. "Cache Tax for Claude Code." [github.com/karanb192/cache-tax](https://github.com/karanb192/cache-tax)
- **[29]** tomstagl. "cctop." [github.com/tomstagl/cctop](https://github.com/tomstagl/cctop)
- **[30]** tzafrir. "What's Agent Doing." [github.com/tzafrir/whats-agent-doing](https://github.com/tzafrir/whats-agent-doing)
- **[31]** juulsverne. "Claude Code mods" (cost-git). [github.com/juulsverne/claude-code-mods](https://github.com/juulsverne/claude-code-mods)
- **[32]** Quimby, Max. "Claude Code mods just turned agents into a platform." DEV Community, October 4, 2026. [dev.to/max_quimby/claude-code-mods-just-turned-agents-into-a-platform-5gc0](https://dev.to/max_quimby/claude-code-mods-just-turned-agents-into-a-platform-5gc0)
- **[33]** KilimcininKorOglu. "claude-code-mods" collection. [github.com/KilimcininKorOglu/claude-code-mods](https://github.com/KilimcininKorOglu/claude-code-mods)