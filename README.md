# omp intercom

Local messaging between independent top-level [omp](https://github.com/can1357/oh-my-pi) sessions, including sessions in separate Tern panes. Unlike omp's `agent://` messaging, intercom reaches sessions in different processes on the same machine.

Tern is not required. The extension uses Bun's Unix sockets and omp's extension API; it has no runtime package dependencies. Verified with omp 18.4.9 and Bun 1.4.2 on macOS.

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

Send an aside that wakes an idle receiver:

```text
/intercom <to> --aside <message>
```

`<to>` resolves in order by session id, full working-directory path, project folder name, or a case-insensitive title substring. Ambiguous matches fail rather than choosing a session. Use an id when several sessions share a folder. The command's target must be a single whitespace-free token; the tool accepts paths and title substrings containing spaces.

## Agent tool

The extension registers the `intercom` tool with these arguments:

```json
{"action": "list"}
```

```json
{
  "action": "send",
  "to": "<session id, folder, path, or title substring>",
  "message": "The change is committed; you can continue.",
  "delivery": "quiet"
}
```

`delivery` is optional and defaults to `quiet`; use `aside` to wake an idle receiver. Agents should send only when the user asks. Use `agent://` for subagents in the same session tree.

## Delivery and lifecycle

- **Quiet:** appends an agent-attributed custom message with `deliverAs: "nextTurn"`. It does not interrupt a running turn or start a turn in an idle receiver.
- **Aside:** sends an agent-attributed user-style message with `deliverAs: "aside"`. It arrives at the next step boundary and starts a turn if the receiver is idle.
- Received notes identify their sender and explicitly distinguish peer-agent text from the user's instructions.
- Only top-level sessions own listeners. Session switches and branches update the card while retaining the listener; completed turns refresh its title.
- A delivery acknowledgement means omp accepted the message, not that the receiving agent acted on it.

Listeners live in `~/.omp/agent/intercom`: an owner-only directory (`0700`) containing a Unix socket and JSON session card (`0600`) per process. The roster excludes the current process and removes cards and sockets belonging to dead processes. Normal shutdown removes the listener's files.

This is local IPC for processes running as the same OS user, not a network service or a security boundary between those processes. The sender limits message text to 64 KiB and waits up to five seconds for an acknowledgement. Session cards contain the working directory, title, process id, and session id; no conversation transcript is stored there.

## Verification

The extracted package and the installed extension symlink were loaded in two real headless omp processes. The smoke run exercised quiet delivery without starting a turn, aside delivery that woke the receiver, delivery after `/new`, file permissions, and clean shutdown. The aside-triggered turn was aborted after the wake event.
