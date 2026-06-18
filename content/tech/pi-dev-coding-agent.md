---
title: "Pi.dev: The Coding Agent You Actually Control"
date: "2026-06-03T22:30:00+05:30"
description: "A hands-on walkthrough of Pi — the minimal terminal-based coding agent with extensions, skills, and no vendor lock-in"
summary: "Pi is a coding agent that doesn't tell you how to work. Terminal-native, extensible via TypeScript, with 15+ providers and four operating modes. Here's how it works, what makes it different from Cursor and Claude Code, and how to build your first extension."
draft: false
tags: ["tech", "pi", "coding-agent", "ai", "extensions", "terminal", "developer-tools", "open-source"]
categories: ["Tech"]
author: "Sanjay Upadhyay"
toc: true
slug: "pi-dev-coding-agent"
---

There's a new coding agent in town. It's called Pi. Not the Raspberry Pi — the *other* Pi. pi.dev. Yeah, the naming is confusing. But the tool itself is anything but.

I've been watching the AI coding tool landscape for a while. Cursor, Claude Code, Windsurf, Copilot. They're all good. But they all make assumptions about how you should work. Cursor wants you in their editor. Claude Code wants you in their ecosystem. Copilot wants you in GitHub's world.

Pi doesn't care. It's a terminal app that reads files, runs commands, and writes code. That's it. Four tools: read, write, edit, bash. No opinionated UI. No forced workflow. Just a prompt and the ability to touch your filesystem.

I installed it, broke it, fixed it, and here's what I learned.

## Installing Pi

The install is stupid simple:

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

Or if you prefer curl:

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

The `--ignore-scripts` flag matters. Pi doesn't need lifecycle scripts. It's a conscious design choice — they keep the install clean and secure.

Once installed:

```bash
cd ~/projects/your-project
pi
```

If you have an Anthropic or OpenAI API key, set it before starting:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

Or use a subscription-based provider with `/login` — it supports Claude Pro/Max, ChatGPT Plus/Pro (Codex), and GitHub Copilot.

## The first thing you notice

Launch Pi and you get a clean terminal UI. A startup header shows your loaded AGENTS.md files, available prompt templates, skills, and extensions. Below that, an editor with a border that changes color based on the thinking level.

Type a request and press Enter. The model reads files, writes edits, runs bash commands. It's fast. The streaming output is smooth.

Here's what I like immediately: **@file fuzzy search.** Type `@` and start typing a filename. Pi fuzzy-searches your project. Select one, and it gets passed as context. No dragging files around. No clicking through folder trees.

```bash
pi @src/server.ts @src/config.ts "Refactor these to use the shared types"
```

Or inside an active session, just type `@server.ts` and it works the same.

## The four modes

Pi can run in four ways, and this is where the design philosophy shows:

| Mode | Use Case |
|---|---|
| **Interactive** | Full TUI for daily work |
| **Print (`-p`)** | One-shot prompts for scripting: `pi -p "summarize this"` |
| **JSON (`--mode json`)** | Structured output for automation |
| **RPC (`--mode rpc`)** | Embed Pi in other apps via stdin/stdout JSONL |

The print mode is surprisingly useful. I use it for quick code reviews in my CI pipeline:

```bash
git diff HEAD~1 | pi -p "Review this diff for bugs and security issues"
```

Pipes right through. No UI. Just the answer.

## Context files — the AGENTS.md system

This is the killer feature that no other tool does quite right.

Pi loads AGENTS.md (or CLAUDE.md) files from multiple levels:
- `~/.pi/agent/AGENTS.md` — your global instructions for *every* project
- Parent directories walking up from cwd
- Current directory

All matching files get concatenated. So you can have:

`~/.pi/agent/AGENTS.md` (global):
```
- Always ask before running destructive commands
- Use `npm run check` before suggesting changes
- Prefer TypeScript over JavaScript
```

`~/projects/my-app/AGENTS.md` (project-specific):
```
- This is a React app with Vite
- Run `npm run dev` to test
- API routes are in /src/api/
- Database schema is in /prisma/
```

Pi merges them. The model sees both sets of instructions. If there's a conflict, the project-level file overrides. This is *context engineering* — not prompt engineering. You're shaping what the model knows about your project before it does anything.

## The model that backs it

Pi supports 15+ providers and hundreds of models. I'm running it with Anthropic Claude for the main work and using DeepSeek for quick, cheap operations.

Switching models mid-session is Ctrl+L or `/model`. You can chain models — use a fast one for file exploration and a slow, smart one for complex refactoring.

## Extensions — where Pi gets interesting

The extensions system is the reason I'd pick Pi over anything else. Extensions are TypeScript modules. You drop them in `~/.pi/agent/extensions/` and they load automatically on next startup.

Here's a minimal extension:

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // Register a custom command
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify("Hello!", "info");
    },
  });

  // Register a custom tool
  pi.registerTool({
    name: "greet",
    label: "Greeting",
    description: "Generate a greeting",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, onUpdate, ctx, signal) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });
}
```

That's it. A fully functional custom tool and command in ~30 lines.

What can extensions do?

- **Override built-in tools** — wrap `read` with an access control layer
- **Add tools** — fetch, deploy, ask the user for clarification
- **Subscribe to events** — `tool_call`, `session_start`, `model_select`
- **Custom UI** — status bars, widgets, modal editors, overlays
- **Security gates** — permission prompts before dangerous commands

## Skills — reusable instructions with tools

Skills follow the Agent Skills standard (agentskills.io), which is the same format OpenClaw uses. A skill is a `SKILL.md` file with instructions and optional tools:

```
~/.pi/agent/skills/
  docker-debug/
    SKILL.md
    tools/
      docker-inspect.sh
```

When the model detects the user needs a skill, it loads it on demand. No context wasted on capabilities you're not using. Progressive disclosure without busting the prompt cache.

## The extensions I actually built

I needed a permission gate — something that blocks destructive commands unless I explicitly approve. There's an example in the repo but I wanted it custom:

I wanted it to block:
- `rm -rf` on anything outside the project dir
- `sudo` commands
- `systemctl` restarts
- Docker volume deletions

Pasted my requirements into Pi. It generated the extension on the spot, I dropped it in the extensions folder, hit `/reload`, and it was live. No restarts.

The `/reload` command is underrated. Live-reloads extensions, skills, prompts, and context files. You can iterate on an extension by editing it and reloading immediately.

## How it compares to other tools

| Feature | Pi | Claude Code | Cursor | Copilot |
|---|---|---|---|---|
| Terminal-native | ✅ | ✅ | ❌ (editor) | ❌ (editor) |
| Multiple providers | ✅ (15+) | ❌ (Anthropic only) | ✅ | ❌ (OpenAI) |
| Custom extensions | ✅ (TypeScript) | ✅ (Shell scripts) | ❌ | ❌ |
| AGENTS.md loading | ✅ (multi-level) | ❌ | ❌ | ❌ |
| Offline mode | ✅ | ❌ | ❌ | ❌ |
| Session tree / branching | ✅ | ❌ | ❌ | ❌ |
| Package ecosystem | ✅ (npm/git) | ❌ | ✅ (marketplace) | ❌ |
| RPC/SDK embedding | ✅ | ❌ | ❌ | ❌ |

The biggest gap is the ecosystem. Cursor has a marketplace. Pi has npm. But since you can `pi install npm:@foo/pi-tools`, it's pretty close.

## Things that broke

Not everything was smooth:

- **Default Openai provider hit my exhausted quota immediately.** Switched to Anthropic and everything worked.
- **The `!` shell command didn't work on first try** — forgot I was in a restricted environment. Works fine on a regular terminal.
- **Extensions can conflict.** I loaded two that both registered a `deploy` command. Pi picked the last one silently. Took me a minute to figure out.
- **Session files accumulate.** They're stored in `~/.pi/agent/sessions/` and never cleaned up. After a week of heavy use, I had 200+ session files. A weekly cleanup script would be nice.

## Where I'd use this

I'm now using Pi for:

1. **Daily code work** — refactoring, debugging, writing tests. It's my default coding assistant.
2. **CI pipeline reviews** — `pi -p "review this diff"` piped into my git hooks
3. **Learning new codebases** — clone a repo, `pi -p "explain the architecture"`, get a summary
4. **Automated bug fixing** — feed error logs into Pi via pipe, get fixes back

It's not replacing Cursor for people who want an IDE-integrated experience. But if you live in the terminal, Pi is hard to beat.

The philosophy — primitives, not features — means you build what you need and nothing else. That's refreshing in a world where every tool tries to do everything.

And if you don't want to build it yourself, there are already 50+ extension examples and a growing npm package ecosystem. Someone probably already built what you need.

{{< comments >}}
