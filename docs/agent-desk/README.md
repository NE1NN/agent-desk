# Agent Desk

Agent Desk is Hindie's web app for running Claude Code agents on hindie-box.
It has four parts: a session list, a decision inbox, multiple Claude accounts, and an orchestrator session that manages the others.
Agents post decisions with images and video, Hindie answers on a card, and the answer goes back to that agent.
When one Claude account hits its weekly limit, sessions move to the other account and carry on.

This repo is a fork of [siteboon/claudecodeui](https://github.com/siteboon/claudecodeui) (CloudCLI, AGPL-3.0).
CloudCLI already gives us the web UI, sessions, chat, terminal, files and git.
We add the decision inbox and multi-account support on top.
Work lives on the `agent-desk` branch.
`upstream` points at siteboon/claudecodeui so we can pull their fixes.

## Open question: T3 Code instead of CloudCLI

On 2026-10-21 Hindie asked about [T3 Code](https://github.com/pingdotgg/t3code) (MIT, web, native iOS and Android apps, desktop).
It already has an MCP server that can start, message and stop threads (the orchestrator), multiple Claude accounts, visual HTML replies in threads, option questions, push notifications and a usage page.
It lacks balancing across accounts and one inbox for questions from every thread.
Forking it is costly: the official apps update the server to the newest npm release, and upstream changes daily.

The proposal is to run T3 as shipped, use a pinned orchestrator thread as the decision inbox (visual reply page plus an option question per decision), and run balancing as a background service on the box.
A demo ran in T3 on the box (a decision page with two renders and an option question), and it worked on desktop and phone widths.
The agent used still frames; whether visual replies can play video is not checked yet.
T3 is running ad hoc on the box (`t3 serve --host 100.74.186.66`, port 3773, log `~/t3-serve.log`), not as a service yet.
Waiting on Hindie's call before building further.

## Where it runs

It runs on hindie-box (Ubuntu, EC2 i-069bf8c49e5f996ab) as the `ubuntu` user.
Open it at `http://hindie-box:3001` (or `http://100.74.186.66:3001`) from any device on Hindie's tailnet.
The box has no inbound ports, so the app is reachable only over Tailscale.

It does not go on Vercel or Railway.
The server starts `claude` processes, reads `~/.claude`, and works in the repos on the box.
Vercel can't run long-lived processes, and Railway would be a different machine without the repos or the sessions.

Never stop, restart or reconfigure Tailscale on the box (see the box rules in solis-v1 `AGENTS.md`).
That includes `tailscale serve`: ask Hindie first.

## Design

The clickable mockup was made in Claude as an Artifact: https://claude.ai/artifact/1SzakHdTWCuV4vVZ6ga8V8
That link is private to the Claude account that made it.
The same design files are in [`design/`](design/), so any account can read them.
Screens: Decisions (`Main.dc.html`), Sessions (`Sessions.dc.html`), Accounts (`Accounts.dc.html`).
The look: off-white ground `#F6F5F1`, ink `#1C1C1A`, one blue accent `#2448C8`, Inter for UI, JetBrains Mono for session names and output.
In the real app, build the screens with CloudCLI's own components and theme, and keep the layout and flow from the mockup.

## How it works

### Decisions

Agents ask for a decision with an MCP tool, and Hindie answers in the Decisions page.

- **The tool.** The server passes an in-process MCP server (Agent SDK `createSdkMcpServer`) to every Claude session it runs.
  Each session's server knows its own session id, so an agent never has to say who it is.
  - `request_decision`: title, context, 2 to 4 options, the recommended option, whether it can be undone, whether it blocks the agent, and media file paths.
  - `reply_to_decision`: adds a message to a decision's conversation, and can replace the options.
- **How agents learn to ask.** The server appends a short guide to every session's system prompt: when to ask, one question per decision, always give a recommendation, attach renders and screenshots.
- **Storage.** Decisions and their messages go in CloudCLI's SQLite database.
  Media files are copied into `~/.cloudcli/decision-media/<decision id>/` at request time, so the agent can delete its temp files.
  The server serves them to the logged-in UI.
  Video plays in a `<video controls>` element.
- **Answers back to the agent.** Picking an option, or sending a reply, queues a message into the agent's session with CloudCLI's existing queued-message feature (`sessionDraftsDb` plus the dispatcher in `server/modules/scheduled-messages`).
  It is delivered as soon as the session is idle, so it never cuts off a running turn.
- **Notifications.** A new decision goes through CloudCLI's notification orchestrator (web push and desktop).

### Accounts

- **Logins.** Each account gets a long-lived token from `claude setup-token`, run once per account.
  Hindie pastes each token into the Accounts page.
  Tokens are stored in the app's SQLite database on the box, and are never shown again or logged.
- **Per session.** Every session records which account it runs on.
  When the server starts a Claude turn, it sets `CLAUDE_CODE_OAUTH_TOKEN` in that turn's env (the `sdkOptions.env` line in `server/modules/providers/list/claude/claude-runtime.provider.js`).
  All accounts share the one `~/.claude`, so history is shared and any account can resume any session.
- **Balancing.** Every reply carries a `rate_limit_event` with how much of the 5-hour and weekly limits that account has used.
  The server keeps the latest reading per account.
  An account's load is the higher of its two readings, and an account with no reading yet counts as unused.
  New sessions start on the account with the lowest load unless Hindie picks one.
  A running session stays on its account, because switching loses the prompt cache and costs more usage.
  It moves only when its account is limited, or when its weekly use is 25 points or more above the lightest account.
- **Hitting the limit.** When a turn ends on a usage limit (a `rejected` rate limit event, or the limit message in the result), the server marks that account as limited until its reset time.
  It moves the session to the lightest account and queues a "continue where you left off" message.
  When every account is limited, it notifies Hindie with the earliest reset time.
- **The box login counts as an account.** "Box login" uses `~/.claude/.credentials.json` and needs no token, so one added token is enough for two accounts.
  With no tokens added, the app works the way CloudCLI does today.

### Orchestrator

One session can run the others, like Canopy's central orchestrator.
Hindie marks a session as the orchestrator, and the server gives that session extra tools on the same in-process MCP server:

- `list_sessions`: every session with its project, status, account and last activity.
- `start_session`: a new session in a project, with a first prompt and an optional account.
- `send_to_session`: queues a message into a session, delivered when it is idle.
- `read_session`: the last messages of a session, so the orchestrator can check progress.
- `stop_session`: stops a running turn.

Other sessions get only the decision tools.
The orchestrator posts its own decisions to the same inbox.

## Deploy

On the box, as `ubuntu`:

```bash
cd ~/Projects/agent-desk
git pull
npm ci
npm run build
systemctl --user restart agent-desk
```

`agent-desk.service` is a systemd user unit (linger is on for `ubuntu`), so the app survives logouts and comes back after a reboot.
It runs `npm run server` with `PORT=3001`.
Logs: `journalctl --user -u agent-desk -f`.

The box powers itself off one hour after the last activity (`/usr/local/sbin/idle-shutdown`).
Claude turns count as activity, so the app stays up while agents work.
After a power-off, start the instance from AWS and the service starts on boot.

## What Hindie does once

1. Open `http://hindie-box:3001` and sign in as `hindie` with the password in `~/agent-desk-login.txt` on the box.
   Change it in Settings if you like.
2. On the Mac, for each Claude account the box is not already logged in to: run `claude setup-token`, sign in to that account in the browser, and copy the token.
3. In Agent Desk, Accounts, Add account: paste each token and give it a name.

## Status

Updated as work lands.
A new session continues from the first unchecked item.

- [x] Fork, rename to NE1NN/agent-desk, branch `agent-desk`, this doc
- [x] Install and run stock CloudCLI on the box as a systemd user service on :3001, login created
- [ ] Decisions backend: tables, MCP tools, media store, queued answers, system prompt guide
- [ ] Decisions page in the UI
- [ ] Accounts backend: token store, per-session account, limit detection and switching
- [ ] Accounts page in the UI, account shown on each session
- [ ] Orchestrator: session flag, management tools, a way to mark a session in the UI
- [ ] End-to-end test on the box: an agent posts a decision with a video, answer it, agent continues
- [ ] Hindie's one-time steps above

## Picking this up in a new session

Read this file, then `AGENTS.md` and the two skills in `.agents/skills/` (backend and frontend module standards).
The Mac clone is `~/Projects/agent-desk`, and the box clone is `~/Projects/agent-desk` on hindie-box.
Reach the box with `ssh -o ProxyCommand=none ubuntu@100.74.186.66` (Tailscale), or `ssh hindie-box` (SSM, needs `aws login --profile lyra`).
The box runs Node 22; the Mac has Node 26, so run installs and tests on the box.
