---
title: "Keeping an eye on several Claude Code sessions at once"
date: 2026-11-17
categories: [devops]
tags: [docker, traefik, homelab, claude-code]
description: "A small self-hosted dashboard to see what all your Claude Code sessions and subagents are doing, in real time."
translationKey: "claude-agents-board"
---

One Claude Code session in one terminal is easy to follow. Three sessions spread across two machines, each one spawning subagents, is another story. One is refactoring, one is running tests, one has been waiting for twenty minutes for me to approve a `psql` command, and I only find out when I happen to switch to that tab.

That last one is the real problem. Not the agents working, the agents *waiting*.

So I made a small dashboard for it: [claude-agents-board][repo]. It runs in my homelab, every Claude Code session on the local network reports to it, and a single browser tab shows who is working, who is idle, and who needs me. It's not a revolutionary tool. It's a convenient one.

## The idea fits in one sentence

Claude Code [hooks][hooks] fire on almost everything a session does: a prompt, a tool call, a subagent starting or stopping, a permission request, the end of a turn. Each machine sends those events to a small Node server, which keeps the state and pushes changes to a web UI over server-sent events. That's it. No agent to install, no change to how you use Claude Code.

There's one small detail worth knowing. Claude Code supports HTTP hooks natively, but it refuses to call a private address like `10.0.0.10`: only loopback is allowed. So the hooks are plain `command` hooks that pipe the event JSON to `curl`. They're synchronous, so events arrive in order, and they give up after two seconds if the dashboard is down. A dead dashboard never blocks an agent.

The app serves its own install script, so plugging in a machine is one line:

```bash
curl -fsSL http://agents.10.0.0.10.sslip.io/install-hooks.sh | sh -s -- --host laptop
```

The script merges the hooks into `~/.claude/settings.json` without touching existing ones, and a PowerShell version exists for Windows. The rest of the setup is in the [README][repo].

## One page, every session

![Home page: you at the top, sessions grouped by machine in the middle, shared stations at the bottom](/images/claude-agents-board/home-tree.png)

The home page is a tree. You at the top, every open session in the middle grouped by machine, and at the bottom the "stations" they use: model, files, terminal, network, agents, and so on. Each tool call sends a pulse from the session to the station and back. A session that's working breathes in orange; a session waiting for you turns amber, and the link between you and it starts scrolling.

That amber state is the reason the whole thing exists. A banner at the top lists every session waiting for an answer or a permission, and an optional chime plays when one asks. I no longer discover a blocked agent by accident.

The sidebar groups sessions by machine, then by project folder. Sessions running inside a git worktree are attached to their repository instead of showing up as strangers.

## What a session is actually doing

![Timeline of a session: the main thread, two subagents, and the task list](/images/claude-agents-board/session-timeline.png)

Clicking a session opens its detail, and the timeline is the view I use most. One row for the main thread with a bar per turn, one row per subagent from launch to finish, one row per task from Claude's task list, and the tool calls as small marks underneath.

For a running agent, a dashed line shows when it should finish. The estimate is just the median duration of past agents of the same type, on the same project when there is enough history. Nothing clever, but enough to tell "normal" from "something's stuck". The session header also copies `claude --resume <id>` in one click, handy for picking a session back up from another terminal.

The journal view lists prompts and tool calls in order, and each turn ends with the final answer of the model. That's all the hooks give: the intermediate messages never leave Claude Code.

## How much it costs, roughly

![Tokens and estimated cost over 7 days, stacked by model](/images/claude-agents-board/home-tokens.png)

Hooks don't carry token usage, but Claude Code can export [OpenTelemetry][otel] events, one per model request. The install script turns that on and points it at the dashboard. Each session gets a small badge with its tokens and estimated cost, and the home page shows the last seven days by model, including how much was read from the cache.

The cost is Claude Code's own estimate at public API prices. On a subscription, that's not what you pay. I read it as an order of magnitude, and as a way to notice which session has been burning through Opus all afternoon.

## The part that's just for fun

![The office view: the main agent at the front desk, a subagent at its table](/images/claude-agents-board/session-office.png)

Every session also has an office, in pixel art. The main agent sits at the front desk in an orange jacket, each subagent walks in through the door, takes a table, types or reads depending on the tool it's using, then gets up and leaves when it's done. When the main agent has been idle for a while, it goes for a coffee or sits on the couch in front of the TV. A session that stopped responding takes a nap.

Is it useful? Not really. The timeline says the same thing more precisely. But it's by far the most fun one to watch. The sprites come from the [MetroCity][metrocity] packs by JIK-A-4, released under CC0.

## What it doesn't do

It's built for a local network, and it shows it. There's no authentication on the UI: anyone on the network can read the prompts, the commands, and the files the agents read. An optional token protects the endpoint that receives events, not the page. I wouldn't expose it outside the house.

The hooks don't report everything either. No intermediate model messages, no context usage, and subagents launched by a workflow show up without their instructions. A session whose terminal was closed without a clean exit stays "idle" until the automatic cleanup removes it.

None of that bothers me for what it's meant to be: a tab I glance at to know whether something needs me. It runs as one more stack next to the others in the [homelab]({{< relref "2026-02-17-homelab-docker-compose-traefik.md" >}}), behind Traefik, on an [sslip.io]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}) hostname. The screenshots above come from a demo instance fed with made-up sessions.

<a class="gh-card" href="https://github.com/guillaumedelre/claude-agents-board" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/claude-agents-board</span>
  </span>
  <span class="gh-card__desc">Real-time web dashboard of Claude Code sessions and subagents, fed by hooks. Docker Compose, Node, React. MIT licensed.</span>
</a>

[repo]: https://github.com/guillaumedelre/claude-agents-board
[hooks]: https://code.claude.com/docs/en/hooks
[otel]: https://code.claude.com/docs/en/monitoring-usage
[metrocity]: https://jik-a-4.itch.io/metrocity
