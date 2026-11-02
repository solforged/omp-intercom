# omp intercom

Local messaging between independent top-level [omp](https://github.com/can1357/oh-my-pi) sessions, including sessions in separate Tern panes. Unlike omp's `agent://` messaging, intercom reaches sessions in different processes on the same machine.

Tern is not required. The extension uses Bun's Unix sockets and omp's extension API, and imports omp's bundled `@oh-my-pi/pi-tui` for its transcript cards; it has no other runtime dependencies. Inside Tern it also uses the `tern` command to name sessions by tab, show them, and read their screens. Verified with omp 18.13.2 and Tern 0.9.9 on Linux.

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

Check whether another session is busy, and read its last reply:

```text
/intercom status <to>
```

`<to>` resolves in order by session id, Tern pane id, Tern place, full working-directory path (`~` allowed), project folder name, or a case-insensitive title substring. A Tern place is the label `list` prints, such as `tab 2` or, in a split tab, `tab 2.1` (panes count left or top first); spaces are optional, so `tab2.1` works too. When the window has several Tern sessions, `list` prefixes places with the session's name (`work tab 2.1`); the place without the prefix still works while only one session has it. Tab numbers follow Tern's current layout, so check `list` after moving tabs. Ambiguous matches fail rather than choosing a session. The command's target must be a single whitespace-free token; the tool accepts paths and title substrings containing spaces. A session whose folder is named `focus` or `status` can't be messaged with the command; use the tool.

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
{"action": "status", "to": "tab 2.1"}
```

```json
{"action": "peek", "to": "tab 2.1", "lines": 20}
```

`delivery` is optional and defaults to `quiet`; use `aside` when a busy receiver should see the note before its current turn ends. Both wake an idle receiver. `status` asks the target itself whether a turn is running and returns its last assistant reply (at most 4,000 characters); it works outside Tern and costs far fewer tokens than `peek`. Targets running an older intercom without `status` answer with an error that points to `peek`. `peek` returns the last `lines` (default 20, at most 400 and 16 KiB) of what the target's Tern pane shows, through `tern capture --surfaces`, including tool output, without messaging it. `list` marks busy sessions and the pane each Tern session shows focused. Agents should send only when the user asks or to answer a peer. Use `agent://` for subagents in the same session tree.

omp mounts the tool as the `xd://intercom` device. The first line of its description is the catalog summary every request carries; the rest loads only when the model reads the device. `list`, `status`, and `peek` use the `read` approval tier, `send` the `write` tier.

## Tern

When omp runs in a Tern pane, the session card records the pane id from `TERN_PANE`. `list` and every lookup join the cards with `tern ls --json`, which adds each session's place and its current pane title. omp keeps the pane title current, while the card's own title only changes when a turn ends, so the live title is shown and matched as well. Outside Tern, or when `tern ls` fails, sessions are listed from their cards alone, and `focus` and `peek` report that the target is not in a Tern pane. `tern` commands act on the Tern window the sender runs in; a session in another window has no place there.

## Delivery and lifecycle

Both modes deliver the same agent-attributed `intercom` custom message, so a note always renders as an intercom card naming its sender rather than as text you typed. In Tern the card is drawn natively, and intercom tool calls show as `Intercom send` (or `list`, `status`, `peek`) with their target.

- **Quiet:** delivered with `deliverAs: "nextTurn", triggerTurn: true`. It never interrupts a running turn; the receiver starts a new turn for it once the current turn ends, or immediately if idle. A notification says it arrived, since omp keeps it out of sight until then.
- **Aside:** delivered with `deliverAs: "aside"`, the path omp's own `agent://` asides take. It arrives at the next step boundary and starts a turn if the receiver is idle. In plan mode, or after an Esc interrupt, it joins the context without starting a turn.
- The model sees the note inside an `<intercom from="…">` block naming the sender's id, title, Tern place, and folder, followed by one line marking it as a peer's note rather than the user's instruction and naming the id to answer. The peer's text cannot close that block or open the tags omp's system prompt treats as harness-authored (`<irc>`, `<system-…>`); their `<` is escaped, as omp does for `agent://` messages. `status` and `peek` output is escaped the same way.
- The note reaches the model as an agent-attributed `developer` message, not a user prompt. Providers without a developer role carry it as a user block that omp still marks as agent-initiated; for GitHub Copilot that means `X-Initiator: agent`, which uses no premium request.
- Only interactive and RPC top-level sessions own listeners. Print and JSON runs (`omp -p`) inherit their pane's `TERN_PANE` but never listen, so they don't appear as peers. Session switches and branches update the card while retaining the listener; turns mark it busy and refresh its title when they end.
- A delivery acknowledgement means omp accepted the message, not that the receiving agent acted on it.

Listeners live in `~/.omp/agent/intercom`: an owner-only directory (`0700`) containing a Unix socket and JSON session card (`0600`) per process. The roster excludes the current process and removes cards and sockets belonging to dead processes. Normal shutdown removes the listener's files.

This is local IPC for processes running as the same OS user, not a network service or a security boundary between those processes. The sender limits message text to 64 KiB and waits up to five seconds for an acknowledgement. Session cards contain the working directory, title, process id, session id, Tern pane id, and whether a turn is running; no conversation transcript is stored there. `status` hands the asking session the target's last reply, and `peek` reads whatever the target's pane shows, which is the same access any process of this user has through its session files or `tern capture`.

## Verification

Earlier versions were loaded in two real headless omp processes, exercising quiet delivery, aside delivery that woke the receiver, delivery after `/new`, file permissions, and clean shutdown. The current version ran in two interactive omp 18.13.2 sessions in split Tern 0.9.9 panes: `list` with busy and focused marks, quiet and aside sends both ways (targets by folder name and by an unprefixed place in a window with several Tern sessions), `status` of an idle peer, `peek`, an unknown target, and the native transcript frames for the tool head and the received card. A stubbed load beside a session running the previous version confirmed the `status` fallback error, and that print mode does not listen.
