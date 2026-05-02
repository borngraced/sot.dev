---
layout: post
title: "I Forked Helix to Make Agentic Coding Feel Native"
description: "Installing DoomHelix, configuring ACP, and using an agent panel for explain, chat, fix, and refactor commands inside Helix."
image: https://img.youtube.com/vi/8Yl6dZ5LhP4/maxresdefault.jpg
date: 2026-05-02
category: editors
---

I have been working on [DoomHelix](https://github.com/borngraced/doom-helix), my fork of Helix with an agent panel built in. The idea is simple: keep the things I love about Helix, like modal editing, speed, themes, LSP, multiple selections, and simple config, then put the agent inside the editor instead of bolting it on through a terminal pane.

DoomHelix uses ACP, the Agent Client Protocol behind Zed's agent integrations, so the editor does not have to be tied to one model or backend. I mostly use it with Codex ACP right now, but the editor side is built around the protocol, so other compatible backends can fit into the same flow.

I recorded a short walkthrough showing installation and a few places where the agent is useful while editing.

<iframe
  width="100%"
  height="400"
  src="https://www.youtube.com/embed/8Yl6dZ5LhP4"
  title="I Forked Helix to Add an AI Agent"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  allowfullscreen>
</iframe>

[Watch on YouTube](https://youtu.be/8Yl6dZ5LhP4)

## Installing DoomHelix

The installer builds from source when a prebuilt release is not available, installs the `dhx` launcher, installs the runtime files, and writes a small agent config if one does not already exist. The first run is mostly about getting the editor and the agent backend configured.

```sh
curl -fsSL https://raw.githubusercontent.com/borngraced/doom-helix/main/install.sh | sh
```

After installation:

```sh
dhx --health
```

If you are checking a specific language setup:

```sh
dhx --health go
dhx --health rust
```

The binary you run is:

```sh
dhx
```

DoomHelix still uses the normal Helix config and runtime locations, so I do not need to maintain a totally separate editor setup just because I am using the fork. Agent-specific config lives in:

```sh
~/.config/helix/agent.toml
```

A minimal Codex ACP config looks like this:

```toml
enable = true
name = "codex"
command = "codex-acp"
args = []
panel-position = "right"
panel-size = 30
auto-context-on-open = true
include-theme = true
include-command-history = true
include-visible-buffer = true
include-diagnostics = true
```

The installer does not try to own your full Helix config, which is important to me because I want DoomHelix to feel like Helix with an agent added, not a separate editor that forces me to rebuild all my habits from scratch. Your theme, keymaps, languages, and normal editor settings stay in `~/.config/helix/config.toml` and `~/.config/helix/languages.toml`.

## Starting the Agent

Inside the editor, I keep the agent commands under `Space a`, so chat, explain, fix, refactor, panel focus, restart, status, and resizing all live in one place instead of being scattered across random commands.

```text
Space a s   start agent
Space a S   status
Space a c   chat
Space a e   explain selection
Space a f   fix selection
Space a r   refactor selection
Space a E   edit
Space a P   show/focus panel
Space a +   grow agent panel
Space a -   shrink agent panel
Space a R   restart agent
Space a x   clear panel
```

The command form is also available when I want to type the action directly:

```text
:agent start
:agent status
:agent chat
:agent explain
:agent fix
:agent refactor
:agent panel
:agent restart
```

The panel can be resized with the keyboard and moved between the sides of the editor, which matters more than it sounds because the right layout depends on the task. If I am reading a longer response, bottom panel feels better; if I am editing code and want the agent beside me, right panel is usually better.

```text
:agent panel position right
:agent panel position bottom
:agent panel position left
:agent panel position top
```

## The Important Part: Editor Context

The thing I care about most is context, because an agent in the editor should know what I am looking at without making me manually paste half my workspace into a chat box. If I select code and ask the agent to explain it, DoomHelix sends the selected code, active file path, language, cursor position, diagnostics, and other useful editor state, and the chat view also shows the selected block so I can see exactly what the agent received.

That matters because "review this" is useless if the agent does not know what "this" means. I want the editor to collect the obvious context for me, while still making it clear what was sent.

For example, I can select a Go handler and run:

```text
Space a e
```

Then ask:

```text
does this handler have any edge cases?
```

The agent sees the selection and responds in the DoomHelix panel, so I do not need to copy the file path, paste the code into a browser, or manually explain where I am in the project before asking the actual question.

## Fixing and Refactoring Code

The second use case is asking the agent to change code. I can select a function and run:

```text
Space a f
```

or:

```text
Space a r
```

DoomHelix sends the editor context through ACP and lets the backend request tool permissions. That was the direction I wanted from the beginning: the editor should not just be a text box for prompts; it should understand enough about the current file, selection, and diagnostics to make the agent useful.

There is still a lot of work to do. Rendering, permission prompts, session restore, and support for multiple agents can all get better, but the basic loop already feels different from using a separate terminal. I can stay inside the editor, select code, ask for explanations, request edits, resize or move the panel when the response needs more room, and keep working without switching tools.

## Why Fork Helix?

I like Helix because it is fast and pragmatic, with strong defaults, good LSP support, tree-sitter, multiple selections, and a clean modal editing model, so I did not want to build a new editor from scratch. I wanted to see what happens when agent-assisted coding is part of a serious modal editor instead of living beside it as another chat window.

DoomHelix is still early, but the direction is simple:

- Keep Helix as the foundation.
- Use ACP as the agent protocol.
- Keep context visible and inspectable.
- Make agent work feel like editing, not a separate chat window beside the editor.

That is the experiment.

Repo:

[github.com/borngraced/doom-helix](https://github.com/borngraced/doom-helix)
