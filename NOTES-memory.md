# Server memory growth: findings

Observation being investigated: a Linux `herdr server` 0.9.3, started by hand,
reached 16 GB RSS after 3.5 days with swap full. A restart brought it back to
24 MB. The server had about 15 panes (5 long-lived Claude Code panes, task
panes created and closed all day), a web terminal (ttyd) attached and often
open in a browser, two plugins with event hooks, scripts polling `pane list`,
`pane read`, `workspace list` and `agent list` every one to two minutes, and
`--machine` calls through an SSH tunnel.

## Summary

- **Leak reproduced, on 0.9.3 and on current master alike.** A server keeps
  every control frame it owes a TUI client in an unbounded queue, and a TUI
  client gets a full endpoint snapshot (the whole session as JSON) on every
  session change, including every terminal title change of an agent pane.
  When one TUI client stops reading its socket without disconnecting, the
  server queues those snapshots for as long as the client stays connected.
  Nothing times it out.
- **Not fixed upstream.** The code path is unchanged from v0.9.3 to v0.9.5 and
  master. The 0.9.4 "hidden panes use less memory" work (#4873 and the related
  pull requests) removes a fixed per-pane cost; it does not touch this queue.
  #3612 (fixed in 0.9.1, so already in 0.9.3) added a write timeout for
  `terminal session observe` clients only, not for TUI clients.
- **No other growth found.** With the same load and no stalled client, RSS
  levels off on both versions; FD and thread counts stay flat in every run.
- **Fix in this branch** (`fix: drop clients that stop reading instead of
  queueing their frames forever`): the unread control bytes of a client are
  capped at 128 MiB. Past the cap the client's queue is dropped, its writer is
  closed as after a failed write (the server already removes such clients),
  and on Unix the socket is shut down so the blocked writer and reader threads
  exit and the FD is released. Two tests fail without the change.
- **Likely production trigger**: the ttyd web terminal. ttyd stops reading its
  pty when the browser stops acknowledging output (flow control; background
  tabs are throttled), so the `herdr` client in it blocks writing to its
  terminal, stops reading the server socket, and the server queue grows. A
  saved-machine client attached over SSH from a machine that sleeps would do
  the same. This was not checked on the production server (see "Checking
  production" below for a read-only check the owner can run).

## Mechanism, with file references

References are to master `143bf5fd`; v0.9.3 (`7b116c05`) has the same code,
line numbers in parentheses where they differ.

1. Every client connection gets a `ClientWriterQueue` with a reliable
   `control: VecDeque<Vec<u8>>` and a one-slot droppable `render` frame:
   `src/server/client_transport.rs:273-280`. `send_control` pushes without any
   bound (`src/server/client_transport.rs:310-318`). Render frames are bounded
   to one slot, control frames are not.
2. On each render pass, for each client-rendered shell (the TUI), the server
   builds a `ClientShellSnapshot` of the whole session and compares it with the
   last one it sent: `src/server/headless/render.rs:635-724` (v0.9.3:
   `565-654`). When it differs, it frames three control messages (agent view
   projection, agent completions, endpoint snapshot) and pushes them with
   `writer.control.send` (`src/server/headless/render.rs:720-722`).
3. The snapshot contains every workspace, tab and pane, and every agent's
   `terminal_title` (`src/server/client_shell.rs:17`, `:169`). Agents such as
   Claude Code animate their terminal title while they work, so a session
   with busy agents produces a new snapshot many times a minute. Other
   broadcasts to shell clients also go through the same unbounded queue
   (`send_to_client_shells`, `src/server/headless.rs:1616`, v0.9.3 `:1568`).
4. The writer thread writes with a blocking `write_all`
   (`src/platform/unix_common.rs:130-141`, v0.9.3 `:68-79`) unless the socket
   has a send timeout, and only `ObserveTerminal` connections get one
   (`src/server/client_transport.rs:1078`). A TUI client that stops reading
   therefore blocks the writer forever once the kernel socket buffer
   (about 208 KiB) is full, and every later snapshot stays in `control`.
5. The client is only removed when a send fails, and a send to this queue
   never fails while the writer thread is alive. So the backlog lives until
   the client process exits or resumes reading.

heaptrack confirms it (master `143bf5fd`, 20-minute run with one stalled TUI
client, see "Profile" below): of a 59.6 MB heap peak, **48.95 MB in 6,136
allocations** come from `protocol::wire::write_message` called from
`HeadlessServer::render_and_stream_with_graphics_limit`, that is the framed
endpoint snapshots in the stalled client's queue, and the same 49.1 MB is
still allocated when the server exits.

The production growth (16 GB in 84 h, about 190 MB per hour) is close to the
rate measured here with about 15 panes (about 270 MB per hour, round 2), and
the production snapshot is larger (33 workspaces after a restore), so fewer
changes per second would produce it.

## Reproduction

Everything ran on a separate server instance with its own socket, config and
state, never the production server. One directory under `/var/tmp` holds the
kit (`$REPRO` below); the scripts are reproduced at the end of this note.

Build (Zig 0.16.0 from ziglang.org, checksum verified, set as `ZIG`):

```sh
git archive v0.9.3 | tar -x -C "$REPRO/src-v0.9.3"
(cd "$REPRO/src-v0.9.3" && nice -n 10 cargo build --release --locked --jobs 4)
nice -n 10 cargo build --release --locked --jobs 4   # master, then the fix
```

An instance is isolated by the wrapper `env.sh` writes: it unsets
`HERDR_SESSION`, `HERDR_CLIENT_SOCKET_PATH` and the pane/tab/workspace IDs
inherited from the outer pane, and sets `HERDR_SOCKET_PATH` (the client
socket is derived from it, `src/server/socket_paths.rs:23`),
`HERDR_CONFIG_PATH` (a config with `[experimental] allow_nested = true`
because `HERDR_ENV=1` is inherited, and `resume_agents_on_restore = false`),
`XDG_CONFIG_HOME` and `XDG_STATE_HOME`. The wrapper is first on the server's
`PATH`, so plugin hooks calling `herdr` reach the test instance.

Load per instance (`run.sh` + `load.py`):

- N long-lived workspaces, each running `chatty.sh`: a primary-screen agent
  imitation that redraws a spinner, prints colored lines, bursts 400 lines
  every few seconds and changes its terminal title about five times a second.
- Every 8 s a task workspace is created in a fresh `git worktree`, runs
  `chatty.sh`, reports an agent, is renamed, and is closed 40 s later with its
  worktree removed (about 210 created and closed per 30 minutes).
- Every 3 s each long-lived pane flips `pane report-agent` between working and
  idle, which fires `pane.agent_status_changed`.
- A linked plugin with hooks on `pane.agent_status_changed` (runs
  `herdr pane list` and `herdr workspace list`) and `pane.agent_detected`
  (runs `herdr agent list`); about 2,600 hook runs per 30 minutes.
- Every 2 s: `pane list`, `workspace list`, `agent list`, and
  `pane read --source recent --lines 200` of every long-lived pane.
- A raw `events.subscribe` client that reads, and a TUI client on a pty that
  keeps reading (`ptyhost.py -1 -1`).
- Stalled variants only: a second TUI client on a pty whose host stops reading
  after 60 s and never resumes (`ptyhost.py 60 -1`, the ttyd flow-control
  case), and an `events.subscribe` client that never reads.

```sh
STALL_TUI=1 STALL_SUB=1 scripts/run.sh v093-stall   bin/herdr-v0.9.3 1800
STALL_TUI=1 STALL_SUB=1 scripts/run.sh master-stall bin/herdr-master 1800
scripts/run.sh v093-nostall   bin/herdr-v0.9.3 1800
scripts/run.sh master-nostall bin/herdr-master 1800
scripts/sample.sh <server-pid> 30 rss.csv     # VmRSS, VmSwap, VmHWM, FDs, threads
```

`stop.sh <instance>` stops an instance by its recorded PIDs.

## Numbers

### Round 1: 5 long-lived panes, 30 minutes, four instances side by side

Binaries: `herdr 0.9.3` built from `v0.9.3` (`7b116c05`), and `herdr 0.9.5`
built from master `143bf5fd`. VmRSS in MB (FDs / threads), sampled every 30 s.
VmSwap was 0 throughout.

| minute | 0.9.3 stalled TUI | master stalled TUI | 0.9.3 no stall | master no stall |
|---:|---:|---:|---:|---:|
| 0  | 37 (48/38) | 34 (38/26) | 35 (36/25) | 33 (35/23) |
| 5  | 77 (49/35) | 64 (49/34) | 61 (47/33) | 61 (48/33) |
| 10 | 89 (49/36) | 79 (49/34) | 63 (47/33) | 57 (50/36) |
| 15 | 119 (50/36) | 111 (50/36) | 75 (47/35) | 82 (47/32) |
| 20 | 120 (50/37) | 107 (55/40) | 82 (47/34) | 68 (47/32) |
| 25 | 143 (49/35) | 139 (49/34) | 86 (47/33) | 79 (47/32) |
| 30 | 152 | 153 | 75 | 65 |
| VmHWM | 163 | 157 | 92 | 90 |

The stalled instances grow by about 4 MB per minute and do not level off; the
others level off between 60 and 90 MB once scrollback is full. 0.9.3 and
master behave the same. The stalled TUI client's server-side socket sat at a
full send queue (212,992 to 222,720 bytes in `ss -x`) for the whole run.

### Round 2: 10 long-lived panes, 60 minutes, master against the fix

Binaries: `herdr 0.9.5` from master `143bf5fd`, and `herdr 0.9.5` from this
branch's fix commit. Same load as round 1 with `LOAD_LONG_LIVED=10` (about 15
panes open at any time, as on the production server). VmRSS in MB.

| minute | master stalled TUI | fix stalled TUI | master no stall |
|---:|---:|---:|---:|
| 0  | 41  | 43  | 35 |
| 5  | 83  | 84  | 64 |
| 10 | 116 | 112 | 77 |
| 15 | 138 | 139 | 75 |
| 20 | 162 | 159 | 77 |
| 25 | 199 | 195 | 88 |
| 30 | 212 | 207 | 82 |
| 35 | 252 | 214 | 95 |
| 40 | 265 | 208 | 87 |
| 45 | 285 | 186 | 68 |
| 50 | 306 | 198 | 72 |
| 55 | 320 | 188 | 63 |
| 60 | 345 | 188 | 66 |
| VmHWM | 395 | 253 | 126 |

The stalled TUI client stopped reading at 11:01:33. On the fixed server the
cap was reached 29.6 minutes later:

```text
2026-10-21T11:31:12Z WARN herdr::server::client_transport: client stopped reading control frames, dropping it pending_bytes=134210392
```

From then on the fixed server stays flat while master keeps growing by about
5 MB per minute. The backlog reached 128 MiB in 30 minutes here, about
270 MB per hour; at that rate a 3.5-day stall would queue about 22 GB, the
same order as the 16 GB observed.
FDs stayed between 62 and 73 and threads between 42 and 68 in all three
instances; neither grew over the hour.

### Profile

heaptrack 1.5.0 (Ubuntu package, unpacked user-locally) wrapped the master
server for a 20-minute run of the round 2 stalled load:

```text
peak heap memory consumption: 59.59M
peak RSS (including heaptrack overhead): 184.36M
total memory leaked: 53.10M

48.95M consumed over 6136 calls from:
    alloc::raw_vec::RawVecInner<A>::reserve::do_reserve_and_handle
    herdr::protocol::wire::write_message
    herdr::server::headless::render::<impl HeadlessServer>::render_and_stream_with_graphics_limit
    herdr::server::headless::HeadlessServer::run::{{closure}}
    ...
```

The next largest peak consumer is 2 MB, and `EventHub::push` (bounded to 512
events) is 0.6 MB.

### What was ruled out

- Event subscription history: `EventHub` keeps at most 512 events
  (`src/api/event_hub.rs:19`), and a subscriber that falls behind gets
  `events_lost` and is disconnected. A never-reading subscriber was attached in
  every stalled run; it is not where the memory goes.
- Plugin hook runs: at most 200 command logs with 64 KiB of output each
  (`src/app/api/plugins/runtime.rs:11-13`), so at most about 25 MB.
- Pane churn, `pane read`, `pane list`, `agent list`, workspace renames, git
  worktrees: present in the no-stall runs, which level off.
- FDs and threads (#3527, #3612): flat in every run, including the stalled
  ones.
- #4873 (0.9.4): each pane used to keep a full screen copy from creation; that
  is a fixed cost per open pane (on the order of 0.7 MB), not growth over time.

Not reproduced here: `--machine` calls through an SSH tunnel, a restore of 33
workspaces, and real Claude Code panes. The `--machine` CLI calls run in the
calling process, not in this server; a saved-machine TUI on another host
attached to this server would be a shell client and is covered by the stall
case.

## Checking production (read-only, for the owner)

On the production server, a TUI client the server is stuck writing to shows
as a connection on the client socket with a full send queue:

```sh
ss -x -p | grep herdr-client.sock     # Send-Q near 212992 on one connection
```

If that connection belongs to the `herdr` process under ttyd, reloading the
browser tab (or restarting `herdr-web.service`) ends that client and the
growth stops. RSS falls only partly, because the allocator keeps most of the
freed pages: the fixed server went from 214 to 188 MB after dropping a
128 MiB backlog, and in a smoke test RSS stayed where it was after the
stalled client was killed. A server restart returns it all.

## The fix, and follow-up work for the owner to decide

In this branch, `src/server/client_transport.rs`:

- `MAX_PENDING_CONTROL_BYTES` = 4 × `MAX_GRAPHICS_FRAME_SIZE` = 128 MiB of
  unread control frames per client.
- `send_control` counts queued bytes; past the cap it clears the queue, marks
  the writer dead and returns an error, which every caller already handles by
  removing the client. On Unix it also shuts the socket down, so the writer
  blocked in `write_all` and the reader waiting on the same socket exit and
  the FD is released. On Windows the memory is released, but the blocked
  writer thread stays until the peer reads or exits.
- The client that is dropped sees its connection end when it resumes. A
  healthy client never comes near 128 MiB of unread control data.
- Tests: `unread_control_backlog_is_bounded` and
  `control_backlog_overflow_releases_stalled_writer_and_reader`. Both fail
  with the cap check disabled and pass with it.

Validation of the branch on the reproduction machine:

- `cargo fmt --all --check` and
  `cargo clippy --all-targets --locked --jobs 4 -- -D warnings`: pass.
- `cargo test --bin herdr -- control_backlog client_writer stalled_observer`:
  8 passed (the two new tests and the existing writer and observer tests).
- Full `cargo test --locked --jobs 4`: passes, after the `test:` commit on
  this branch. On untouched master `143bf5fd` it does not pass under plain
  `cargo test` (the project's own recipes use nextest, one process per test):
  about a dozen tests fail and the test process is killed by SIGPIPE. The
  causes were all in the tests sharing one process: CLI output tests made
  SIGPIPE fatal for the whole process, tests changing `HOME`, `PATH`,
  `TMPDIR` and XDG paths used separate locks or none, a caught panic stopped
  every running test server, and three waits were too tight for a loaded
  machine. The `test:` commit fixes those without changing product code.

Follow-up worth deciding on, not written here:

1. **Coalesce superseded snapshots.** Endpoint snapshots, completions and the
   agent view projection are full state with a rising revision, so a client
   only needs the newest unsent one. Keeping a single pending snapshot per
   client would keep a slow client connected with constant memory instead of
   dropping it at the cap. It needs care with ordering against other control
   messages and with the activation handshake in `src/client/endpoint/`.
2. **A write timeout for TUI clients**, like observers have (#3612), so a
   client that stops reading entirely is dropped after a fixed time rather
   than after 128 MiB. Its length is a product decision: a browser tab in the
   background can legitimately pause for minutes.
3. **Return freed memory to the system.** After a large backlog is freed, glibc
   keeps most of the pages, so RSS stays near its peak. `malloc_trim` after
   dropping a client, or a different allocator, would let RSS fall.
4. **A client that is stopped and continued never reads again.** In a smoke
   test, a TUI client stopped with `SIGSTOP` and continued with `SIGCONT`
   kept 120 KB unread on its socket for good: its `endpoint-reader` thread
   waited in `blocking_send` on the full client event channel
   (`src/client/transport.rs:72`, channel of 256 at `src/client/mod.rs:533`)
   while the main loop slept. A client blocked by terminal backpressure did
   recover when the terminal read again. Not investigated further; with the
   fix such a client is now dropped at the cap instead of growing the server.

## Reproduction kit

The scripts as run, with `REPRO=/var/tmp/herdr-memrepro`: `scripts/` holds the
scripts below, `plugin/` the plugin manifest and `bin/` the built binaries.

<details><summary><code>plugin/herdr-plugin.toml</code></summary>

```toml
id = "memrepro.hooks"
name = "Memrepro hooks"
version = "0.1.0"
min_herdr_version = "0.9.0"
description = "Event hooks for the memory reproduction"
platforms = ["linux"]

[[events]]
on = "pane.agent_status_changed"
command = ["sh", "-c", "herdr pane list >/dev/null 2>&1; herdr workspace list >/dev/null 2>&1; echo hook-ran"]

[[events]]
on = "pane.agent_detected"
command = ["sh", "-c", "herdr agent list >/dev/null 2>&1; echo detected"]
```

</details>

<details><summary><code>scripts/env.sh</code></summary>

```sh
# Usage: . env.sh <instance-name> <binary>
# Defines the isolated environment of one test instance.
D=/var/tmp/herdr-memrepro
INST=$1
BIN=$2
I=$D/inst/$INST
mkdir -p $I/run $I/xdg-config $I/xdg-state $I/bin $I/logs
cat > $I/config.toml <<CFG
[experimental]
allow_nested = true

[session]
resume_agents_on_restore = false
CFG
cat > $I/bin/herdr <<WRAP
#!/bin/sh
exec env -u HERDR_SESSION -u HERDR_CLIENT_SOCKET_PATH -u HERDR_WORKSPACE_ID -u HERDR_TAB_ID -u HERDR_PANE_ID \
  HERDR_SOCKET_PATH=$I/run/herdr.sock HERDR_CONFIG_PATH=$I/config.toml \
  XDG_CONFIG_HOME=$I/xdg-config XDG_STATE_HOME=$I/xdg-state \
  PATH=$I/bin:$D/scripts:/usr/local/bin:/usr/bin:/bin \
  $BIN "\$@"
WRAP
chmod +x $I/bin/herdr
H=$I/bin/herdr
```

</details>

<details><summary><code>scripts/run.sh</code></summary>

```sh
#!/bin/bash
# Starts one isolated herdr server, attaches clients, runs the synthetic load.
# argv: instance binary duration-seconds
# env: HEAPTRACK=1, STALL_TUI=1 (STALL_AFTER seconds, default 60), STALL_SUB=1
set -u
. /var/tmp/herdr-memrepro/scripts/env.sh "$1" "$2"
DUR=$3
S=$D/scripts
L=$I/logs
pids=$I/pids; : > $pids
"$BIN" --version > $L/version.txt 2>&1
cd $I
if [ "${HEAPTRACK:-0}" = 1 ]; then
  # heaptrack must wrap the herdr binary itself, so set the wrapper's environment here
  setsid nohup env -u HERDR_SESSION -u HERDR_CLIENT_SOCKET_PATH -u HERDR_WORKSPACE_ID -u HERDR_TAB_ID -u HERDR_PANE_ID \
    HERDR_SOCKET_PATH=$I/run/herdr.sock HERDR_CONFIG_PATH=$I/config.toml \
    XDG_CONFIG_HOME=$I/xdg-config XDG_STATE_HOME=$I/xdg-state \
    PATH=$I/bin:$D/scripts:/usr/local/bin:/usr/bin:/bin \
    LD_LIBRARY_PATH=$D/ht/usr/lib/x86_64-linux-gnu \
    $D/ht/usr/bin/heaptrack -o $L/heaptrack $BIN server > $L/server.out 2>&1 < /dev/null &
else
  setsid nohup $H server > $L/server.out 2>&1 < /dev/null &
fi
echo "launcher $!" >> $pids
for i in $(seq 100); do [ -S $I/run/herdr.sock ] && break; sleep 0.2; done
sleep 1
$H plugin link $D/plugin > $L/plugin-link.txt 2>&1
# a TUI client that keeps reading, hosted on a pty like ttyd does
setsid python3 -I $S/ptyhost.py -1 -1 $H > $L/tui-live.out 2>&1 < /dev/null &
echo "tui-live $!" >> $pids
# a TUI client whose terminal stops reading after STALL_AFTER seconds
if [ "${STALL_TUI:-0}" = 1 ]; then
  setsid python3 -I $S/ptyhost.py ${STALL_AFTER:-60} -1 $H > $L/tui-stall.out 2>&1 < /dev/null &
  echo "tui-stall $!" >> $pids
fi
python3 -I $S/subscriber.py $I/run/herdr.sock read > $L/sub-read.out 2>&1 &
echo "sub-read $!" >> $pids
if [ "${STALL_SUB:-0}" = 1 ]; then
  python3 -I $S/subscriber.py $I/run/herdr.sock stall > $L/sub-stall.out 2>&1 &
  echo "sub-stall $!" >> $pids
fi
python3 -I $S/load.py $H $I $DUR > $L/load.out 2>&1 &
echo "load $!" >> $pids
echo "server pid $(head -1 $pids | cut -d' ' -f2)"
```

</details>

<details><summary><code>scripts/stop.sh</code></summary>

```sh
#!/bin/bash
# Stops one test instance: every recorded PID and their descendants, server last.
I=/var/tmp/herdr-memrepro/inst/$1
desc() { for c in $(cat /proc/$1/task/*/children 2>/dev/null); do desc $c; echo $c; done; }
server=""
while read name pid; do
  [ "$name" = launcher ] && { server=$pid; continue; }
  [ -d /proc/$pid ] || continue
  for p in $(desc $pid) $pid; do kill -9 $p 2>/dev/null; done
done < $I/pids
if [ -n "$server" ] && [ -d /proc/$server ]; then
  kids=$(desc $server)
  kill -TERM $server; sleep 3
  [ -d /proc/$server ] && kill -9 $server
  for p in $kids; do kill -9 $p 2>/dev/null; done
fi
echo "stopped $1"
```

</details>

<details><summary><code>scripts/sample.sh</code></summary>

```sh
#!/bin/bash
# Samples a process every N seconds: RSS, swap, FDs, threads. argv: pid interval out
pid=$1; iv=$2; out=$3
echo "epoch,elapsed_s,vmrss_kb,vmswap_kb,vmhwm_kb,fds,threads" > $out
t0=$(date +%s)
while [ -d /proc/$pid ]; do
  rss=$(awk '/^VmRSS/{print $2}' /proc/$pid/status); sw=$(awk '/^VmSwap/{print $2}' /proc/$pid/status)
  hwm=$(awk '/^VmHWM/{print $2}' /proc/$pid/status); th=$(awk '/^Threads/{print $2}' /proc/$pid/status)
  fds=$(ls /proc/$pid/fd 2>/dev/null | wc -l); now=$(date +%s)
  echo "$now,$((now-t0)),$rss,$sw,$hwm,$fds,$th" >> $out
  sleep $iv
done
```

</details>

<details><summary><code>scripts/chatty.sh</code></summary>

```sh
#!/bin/bash
# Imitates a busy agent TUI on the primary screen: colored lines, a spinner
# redrawn in place, title changes, and occasional bursts.
n=0
while :; do
  n=$((n+1))
  printf '\e]0;agent %d\a' $((n % 7))
  for s in '|' '/' '-' '\'; do printf '\r\e[2K\e[36m%s\e[0m working %d' "$s" $n; sleep 0.05; done
  printf '\r\e[2K\e[32m●\e[0m line %d %s\n' $n "$(head -c 60 /dev/urandom | base64 -w0)"
  if [ $((n % 50)) -eq 0 ]; then seq 1 400 | sed 's/^/burst /'; fi
done
```

</details>

<details><summary><code>scripts/ptyhost.py</code></summary>

```python
"""Hosts a TUI client on a pty like ttyd does, and can stop reading its output.

argv: stall_after_s resume_after_s(-1 = never) cmd...
While stalled, the pty buffer fills and the client blocks writing its frames,
which is what ttyd's flow control does when the browser stops acknowledging.
"""
import fcntl, os, pty, select, struct, sys, termios, time
stall_after, resume_after = float(sys.argv[1]), float(sys.argv[2])
cmd = sys.argv[3:]
pid, fd = pty.fork()
if pid == 0:
    os.execvp(cmd[0], cmd)
fcntl.ioctl(fd, termios.TIOCSWINSZ, struct.pack("HHHH", 50, 200, 0, 0))
os.kill(pid, 28)  # SIGWINCH
t0 = time.time()
stalled = False
total = 0
while True:
    el = time.time() - t0
    if not stalled and stall_after >= 0 and el > stall_after and (resume_after < 0 or el < resume_after):
        stalled = True
        print(f"{time.strftime('%T')} stall (read {total} bytes)", flush=True)
    if stalled and resume_after >= 0 and el > resume_after:
        stalled = False
        print(f"{time.strftime('%T')} resume", flush=True)
    if stalled:
        time.sleep(1)
        try:
            os.waitpid(pid, os.WNOHANG)
        except ChildProcessError:
            break
        continue
    r, _, _ = select.select([fd], [], [], 1)
    if r:
        try:
            b = os.read(fd, 65536)
        except OSError:
            break
        if not b:
            break
        total += len(b)
print("client exited", flush=True)
```

</details>

<details><summary><code>scripts/subscriber.py</code></summary>

```python
# Event subscriber. argv: socket path, mode ("read" drains, "stall" never reads).
import json, socket, sys, time
sock_path, mode = sys.argv[1], sys.argv[2]
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect(sock_path)
types = ["pane.created", "pane.closed", "pane.agent_detected",
         "workspace.created", "workspace.closed", "workspace.renamed", "tab.created", "tab.closed"]
req = {"id": "sub", "method": "events.subscribe",
       "params": {"subscriptions": [{"type": t} for t in types]}}
s.sendall((json.dumps(req) + "\n").encode())
if mode == "stall":
    while True:
        time.sleep(3600)
n = 0
while True:
    b = s.recv(65536)
    if not b:
        print("closed", flush=True)
        break
    n += b.count(b"\n")
    if n % 200 == 0:
        print("events", n, flush=True)
```

</details>

<details><summary><code>scripts/load.py</code></summary>

```python
"""Synthetic load for an isolated herdr test instance.

argv: herdr-wrapper instance-dir duration-seconds
Long-lived chatty panes, API pollers, agent status reports that fire plugin
hooks, and workspaces created in fresh git worktrees and closed again.
"""
import json, os, random, subprocess, sys, threading, time

H, INST, DURATION = sys.argv[1], sys.argv[2], float(sys.argv[3])
LONG_LIVED = int(os.environ.get("LOAD_LONG_LIVED", "5"))
POLL_INTERVAL = float(os.environ.get("LOAD_POLL_INTERVAL", "2"))
STATUS_INTERVAL = float(os.environ.get("LOAD_STATUS_INTERVAL", "3"))
CHURN_INTERVAL = float(os.environ.get("LOAD_CHURN_INTERVAL", "8"))
CHURN_LIFETIME = float(os.environ.get("LOAD_CHURN_LIFETIME", "40"))
CHATTY = "/var/tmp/herdr-memrepro/scripts/chatty.sh"
stop_at = time.time() + DURATION
counters = {"api_calls": 0, "api_errors": 0, "churn_created": 0, "churn_closed": 0, "status_reports": 0}
lock = threading.Lock()


def h(*args, check=False):
    p = subprocess.run([H, *args], capture_output=True, text=True, timeout=60)
    with lock:
        counters["api_calls"] += 1
        if p.returncode != 0:
            counters["api_errors"] += 1
    if check and p.returncode != 0:
        raise RuntimeError(f"{args}: {p.stderr.strip()}")
    return p.stdout


def js(*args):
    return json.loads(h(*args, check=True))


def find_pane_id(obj):
    if isinstance(obj, dict):
        for k, v in obj.items():
            if k == "pane_id" and isinstance(v, str):
                return v
            r = find_pane_id(v)
            if r:
                return r
    elif isinstance(obj, list):
        for v in obj:
            r = find_pane_id(v)
            if r:
                return r
    return None


def find_key(obj, key):
    if isinstance(obj, dict):
        if key in obj and isinstance(obj[key], str):
            return obj[key]
        for v in obj.values():
            r = find_key(v, key)
            if r:
                return r
    elif isinstance(obj, list):
        for v in obj:
            r = find_key(v, key)
            if r:
                return r
    return None


def create_workspace(cwd, label):
    out = js("workspace", "create", "--cwd", cwd, "--label", label, "--no-focus")
    return find_key(out, "workspace_id"), find_pane_id(out)


def start_agent(pane):
    h("pane", "run", pane, CHATTY)
    h("pane", "report-agent", pane, "--source", "memrepro", "--agent", "claude", "--state", "working")


long_lived = []
for i in range(LONG_LIVED):
    d = f"{INST}/work/long-{i}"
    os.makedirs(d, exist_ok=True)
    ws, pane = create_workspace(d, f"long-{i}")
    start_agent(pane)
    long_lived.append(pane)
print("long-lived panes", long_lived, flush=True)


def poller():
    while time.time() < stop_at:
        h("pane", "list")
        h("workspace", "list")
        h("agent", "list")
        for p in list(long_lived):
            h("pane", "read", p, "--source", "recent", "--lines", "200")
        time.sleep(POLL_INTERVAL)


def status():
    state = {}
    while time.time() < stop_at:
        for p in list(long_lived):
            s = "idle" if state.get(p) == "working" else "working"
            state[p] = s
            h("pane", "report-agent", p, "--source", "memrepro", "--agent", "claude", "--state", s)
            with lock:
                counters["status_reports"] += 1
        time.sleep(STATUS_INTERVAL)


def churn():
    repo = f"{INST}/repo"
    if not os.path.isdir(repo):
        subprocess.run(["git", "init", "-q", repo], check=True)
        subprocess.run(["git", "-C", repo, "-c", "user.email=t@t", "-c", "user.name=t", "commit", "-q",
                        "--allow-empty", "-m", "init"], check=True)
    live = []
    n = 0
    while time.time() < stop_at:
        n += 1
        wt = f"{INST}/worktrees/task-{n}"
        subprocess.run(["git", "-C", repo, "worktree", "add", "-q", "-b", f"task-{n}", wt],
                       capture_output=True)
        try:
            ws, pane = create_workspace(wt, f"task-{n}")
            start_agent(pane)
            h("workspace", "rename", ws, f"[Opus] TASK-{n} synthetic task title {n}")
            live.append((time.time(), ws, wt, n))
            with lock:
                counters["churn_created"] += 1
        except Exception as e:
            print("churn create failed", e, flush=True)
        now = time.time()
        while live and now - live[0][0] > CHURN_LIFETIME:
            _, ws, wt, k = live.pop(0)
            h("workspace", "close", ws)
            subprocess.run(["git", "-C", repo, "worktree", "remove", "--force", wt], capture_output=True)
            subprocess.run(["git", "-C", repo, "branch", "-q", "-D", f"task-{k}"], capture_output=True)
            with lock:
                counters["churn_closed"] += 1
        time.sleep(CHURN_INTERVAL)
    for _, ws, wt, k in live:
        h("workspace", "close", ws)
        subprocess.run(["git", "-C", repo, "worktree", "remove", "--force", wt], capture_output=True)


threads = [threading.Thread(target=f, daemon=True) for f in (poller, status, churn)]
for t in threads:
    t.start()
while time.time() < stop_at:
    time.sleep(60)
    with lock:
        print(time.strftime("%H:%M:%S"), json.dumps(counters), flush=True)
for t in threads:
    t.join(timeout=120)
print("done", json.dumps(counters), flush=True)
```

</details>

