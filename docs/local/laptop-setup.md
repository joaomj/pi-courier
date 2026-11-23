# Pi from Element

[pi-courier](https://github.com/Hi-Barry/pi-courier) 1.8.3 lets you talk to Pi
from Element. It signs in to your Matrix server as a bot account and runs Pi in
RPC mode. Each machine runs its own bot, and each repo gets its own room.

This page records how the laptop bot, `@pi-laptop:bilbobaggins.app`, behaves.
The facts come from the pi-courier source at commit `b241034` and from the
service log (`journalctl --user -u pi-courier`).

## Project rooms need the full path for repos outside ~/projects

`/pmctl new <name> [path]` binds a new room to a folder. The bot resolves the
path as follows (`resolveProjectPath` in `src/rpc/pmctl-controller.ts`):

- Without a path, it uses `<workdir>/<name>`. On the laptop, `workdir` is
  `/home/marcos/projects`.
- A relative path resolves under `workdir`. A path that leaves `workdir`, such
  as `../x`, is refused.
- A path that starts with `/` or `~/` is used as written.

If the folder does not exist, the bot creates it empty. It does not search for
a repo with the same name.

Example: `/pmctl new pi-agent-config` created an empty
`~/projects/pi-agent-config`, but the repo is at `~/pi-agent-config`. Pi in
that room then reported a folder with no project code.

To bind a repo outside `~/projects`, send in the management room:

```
/pmctl new pi-agent-config ~/pi-agent-config
```

To move an existing room to another folder:

```
/pmctl mv pi-agent-config /home/marcos/pi-agent-config
```

`/pmctl mv` starts a new session, so the room's earlier Pi history is lost.

Each room's session is saved in `<folder>/.pi-session`. The global Git ignore
file, `~/.config/git/ignore`, lists `.pi-session/`, so these folders do not show
in `git status`.

## Hide tool calls with /toggletools in the management room

By default, a project room shows each tool call that Pi runs. The calls appear
on a "working" card that the bot edits as Pi works. There are two settings,
and only the bot-wide one hides the calls.

| Command | Where | Effect |
|---|---|---|
| `/toggletools` | management room | Switches `hideToolCalls` for every room of this bot. When on, no tool calls are posted. |
| `/toggletools <auto\|on\|off>` | project room | Sets `showToolCalls` for that room only. It controls the "working" card. |

To hide tool calls, send `/toggletools` once in the management room. The reply
says "Tool calls hidden in remote messages". Send it again to show them.

Do not use `/toggletools off` in a project room for this. It turns off the
"working" card only. The tool lines are then added to each reply instead,
because the reply path removes them only when `hideToolCalls` is on
(`buildTurnReply` and `roomToolCallsMode` in `src/rpc/message-router.ts`).

## Use unencrypted project rooms

`/pmctl new` creates project rooms without encryption, by design
(`createProjectRoom` in `src/transports/matrix-rooms.ts`). Keep them that way:

- The bot account has no cross-signing, so Element shows each bot message in
  an encrypted room as sent by an unverified device.
- The pi-courier README says that the bot can stop replying in an encrypted
  room when its device has no room keys.
- In one encrypted room, your own messages appeared twice in Element. The bot
  log showed each message received once. The duplicates did not occur in an
  unencrypted room.

The rooms stay on the home server and are reachable only over the tailnet.
Encryption cannot be turned off in a room after you turn it on. If you turn it
on by mistake, run `/pmctl rm <name>`, leave the room, and create it again with
`/pmctl new`.

## "Event not found" errors in the log are expected

The log shows lines like this after most bot messages:

```
[ERROR] [matrix-sdk:MatrixHttpClient] (REQ-333) {"errcode":"M_NOT_FOUND","error":"Event not found."}
```

They come from the Matrix library, `matrix-bot-sdk` 0.8. Before the bot sends
a message, the library asks the server for the room's `m.room.encryption`
state (`queueRoomCheck` in `lib/e2ee/RoomTracker.js`). An unencrypted room has
no such state, so the server answers `M_NOT_FOUND`. The HTTP client logs that
answer as an error. The library then catches it and treats the room as
unencrypted. It stores a result only for encrypted rooms, so the check repeats
for every message.

A debug capture on 2026-11-23 confirmed this for one reply in a project room:

```
(REQ-36) GET  /_matrix/client/v3/rooms/!…/state/m.room.encryption/
(REQ-36) {"errcode":"M_NOT_FOUND","error":"Event not found."}
(REQ-37) PUT  /_matrix/client/v3/rooms/!…/send/m.room.message/…
```

The send in REQ-37 follows the error, and the reply arrived in Element. The
only cost is one extra request to the home server for each bot message.

The lines at room creation were not captured. They probably have the same
cause, or come from similar checks for room names and avatars that are not
set yet.

### Capture the requests again

The library filters its own debug lines, so `--level debug` alone does not
show request URLs. To see them:

1. Create `/tmp/sdk-debug.cjs`:

   ```js
   const sdk = require("/home/marcos/.local/lib/node_modules/pi-courier/node_modules/matrix-bot-sdk");
   sdk.LogService.setLevel(sdk.LogLevel.DEBUG);
   ```

2. Create `~/.config/systemd/user/pi-courier.service.d/sdk-debug.conf`:

   ```ini
   [Service]
   ExecStart=
   ExecStart=/home/marcos/.local/opt/node-24/bin/node /home/marcos/.local/lib/node_modules/pi-courier/dist/standalone.js --level debug
   Environment=NODE_OPTIONS=--require=/tmp/sdk-debug.cjs
   ```

3. Run `systemctl --user daemon-reload && systemctl --user restart pi-courier`.
4. Send a message in a project room, then read the request before each error:

   ```sh
   journalctl --user -u pi-courier --since -5min -o cat | grep -B2 M_NOT_FOUND
   ```

5. Delete both files and run step 3 again. Debug output includes message
   text, so do not leave it on.
