# omp intercom

Local messaging between independent top-level [omp](https://github.com/can1357/oh-my-pi) sessions, including sessions in separate Tern panes. Unlike omp's `agent://` messaging, intercom reaches sessions in different processes on the same machine.

Tern is not required. The extension uses Bun's Unix sockets and omp's extension API; it has no runtime package dependencies. Inside Tern it also uses the `tern` command to name sessions by tab, show them, and read their screens. Verified with omp 18.8.6, Bun 1.4.2, and Tern 0.3 on macOS.

## Installation

Clone this repository, then link the extension into omp's user extension directory:

```sh
mkdir -p ~/.omp/agent/extensions
ln -s "$PWD/intercom.ts" ~/.omp/agent/extensions/intercom.ts
```

Run these commands from the repository root. If an intercom extension is already installed at that path, move it aside before creating the link. Restart each omp session to load the extension.

For a single session without installing globally:

```sh
omp -e /absolute/path/to/omp-intercom
```

The package manifest declares `intercom.ts` through `omp.extensions`.

## Commands

List other listening sessions:

```text
/intercom
/intercom list
```

Send a quiet note:

```text
/intercom <to> <message>
```

Send an aside that reaches a busy receiver at its next step:

```text
/intercom <to> --aside <message>
```

Show another session's Tern pane in every Tern window:

```text
/intercom focus <to>
```

`<to>` resolves in order by session id, Tern pane id, Tern place, full working-directory path, project folder name, or a case-insensitive title substring. A Tern place is the label `list` prints, such as `tab 2` or, in a split tab, `tab 2.1` (panes count left or top first); spaces are optional, so `tab2.1` works too. Tab numbers follow Tern's current layout, so check `list` after moving tabs. Ambiguous matches fail rather than choosing a session. The command's target must be a single whitespace-free token; the tool accepts paths and title substrings containing spaces. A session whose folder is named `focus` can't be messaged with the command; use the tool.

## Agent tool

The extension registers the `intercom` tool with these arguments:

```json
{"action": "list"}
```

```json
{
  "action": "send",
  "to": "<session id, Tern place, folder, path, or title substring>",
  "message": "The change is committed; you can continue.",
  "delivery": "quiet"
}
```

```json
{"action": "peek", "to": "tab 2.1", "lines": 60}
```

`delivery` is optional and defaults to `quiet`; use `aside` when a busy receiver should see the note before its current turn ends. Both wake an idle receiver. `peek` returns the last `lines` (default 60, at most 400 and 16 KiB) of what the target's Tern pane shows, through `tern capture --surfaces`, without messaging it. Agents should send only when the user asks. Use `agent://` for subagents in the same session tree.

## Tern

When omp runs in a Tern pane, the session card records the pane id from `TERN_PANE`. `list` and every lookup join the cards with `tern ls --json`, which adds each session's place and its current pane title. omp keeps the pane title current, while the card's own title only changes when a turn ends, so the live title is shown and matched as well. Outside Tern, or when `tern ls` fails, sessions are listed from their cards alone, and `focus` and `peek` report that the target is not in a Tern pane. `tern` commands act on the Tern window the sender runs in; a session in another window has no place there.

## Delivery and lifecycle

Both modes deliver the same agent-attributed `intercom` custom message, so a note always renders as an intercom card rather than as text you typed.

- **Quiet:** delivered with `deliverAs: "nextTurn", triggerTurn: true`. It never interrupts a running turn; the receiver starts a new turn for it once the current turn ends, or immediately if idle.
- **Aside:** delivered with `deliverAs: "aside"`, the path omp's own `agent://` asides take. It arrives at the next step boundary and starts a turn if the receiver is idle. In plan mode, or after an Esc interrupt, it joins the context without starting a turn.
- Received notes identify their sender, explicitly distinguish peer-agent text from the user's instructions, and name the id to answer.
- The note reaches the model as an agent-attributed `developer` message, not a user prompt. Providers without a developer role carry it as a user block that omp still marks as agent-initiated; for GitHub Copilot that means `X-Initiator: agent`, which uses no premium request.
- Only top-level sessions own listeners. Session switches and branches update the card while retaining the listener; completed turns refresh its title.
- A delivery acknowledgement means omp accepted the message, not that the receiving agent acted on it.

Listeners live in `~/.omp/agent/intercom`: an owner-only directory (`0700`) containing a Unix socket and JSON session card (`0600`) per process. The roster excludes the current process and removes cards and sockets belonging to dead processes. Normal shutdown removes the listener's files.

This is local IPC for processes running as the same OS user, not a network service or a security boundary between those processes. The sender limits message text to 64 KiB and waits up to five seconds for an acknowledgement. Session cards contain the working directory, title, process id, session id, and Tern pane id; no conversation transcript is stored there. `peek` reads whatever the target's pane shows, which is the same access any process of this user has through `tern capture`.

## Verification

Earlier versions were loaded in two real headless omp processes, exercising quiet delivery, aside delivery that woke the receiver, delivery after `/new`, file permissions, and clean shutdown. The Tern additions were checked with the extension loaded under a stub extension API in separate processes inside Tern: lookups by tab, split pane, pane id, and live title; `peek`; `focus`; the errors for sessions outside Tern; and removal of every card on shutdown. A real omp 18.8.6 session started from a Tern pane wrote its pane id to its card.
