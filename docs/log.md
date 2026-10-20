# Log

Every run of `engined` records everything that crosses its IO boundary in a
SQLite database. The inputs are the record of truth: replaying them through the
same code reproduces the run. The outputs are kept to check replay against what
actually happened, and for debugging.

## Structure of the daemon

```
IO shell (sockets, real timers, log writer)
   │ input rows                         ▲ output rows
   ▼                                    │
 router ──► ibkr adapter (pure) ⇄ engine (pure)
```

- The IO shell moves bytes over sockets, runs timers and writes the log. It
  makes no decisions.
- The IBKR adapter and the engine are pure: no IO, no wall clock, no internal
  timers. "Now" is the `time` of the input being handled.
- Inputs are routed by channel prefix: `ibkr/*` to the adapter, `engine/*` and
  `client/*` to the engine.
- The adapter and the engine exchange domain events and commands directly.
  Those are derived from the inputs and are not logged.

## Schema

```sql
CREATE TABLE channel (
  id   INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE
);

CREATE TABLE input (
  sequence_num        INTEGER PRIMARY KEY,  -- shared numbering with output
  parent_sequence_num INTEGER,              -- output this answers (response, timer fire); NULL if unsolicited
  time                INTEGER NOT NULL,     -- unix nanoseconds, non-decreasing; "now" for pure code
  channel_id          INTEGER NOT NULL REFERENCES channel(id),
  kind                INTEGER NOT NULL,     -- 0 open, 1 message, 2 close, 3 error
  payload             BLOB
);

CREATE TABLE output (
  sequence_num        INTEGER PRIMARY KEY,  -- shared numbering with input
  parent_sequence_num INTEGER NOT NULL,     -- input being handled when this was produced
  time                INTEGER NOT NULL,
  channel_id          INTEGER NOT NULL REFERENCES channel(id),
  kind                INTEGER NOT NULL,     -- 0 open, 1 message, 2 close
  payload             BLOB
);
```

- `sequence_num` is handed out by the single log writer to every row, whichever
  table it goes into, and keeps counting across runs. Across both tables it has
  no gaps; it is the order in which things happened and were handled.
- A channel gets a row in `channel` when it is first used.

## Channels

| channel | what it carries |
|---|---|
| `engine` | run lifecycle: `open` at start, `message` with the carryover, `close` at shutdown |
| `engine/timer` | engine timers |
| `ibkr/timer` | adapter timers (keepalive, resubscribe, reconnect backoff) |
| `ibkr/http` | REST requests and responses |
| `ibkr/websocket/N` | the Nth websocket connection of the run |
| `client/N` | client connection N |

Each component names its own channels under its own prefix.

## Kinds

| kind | as output | as input |
|---|---|---|
| open | we're connecting | connection established |
| message | we sent this payload | we received this payload |
| close | we're closing | connection closed (either side) |
| error | — | the attempt, request or connection failed; the payload says why |

## Payloads

Payloads are as close as possible to the bytes that crossed the wire.

- **HTTP request** (output on `ibkr/http`): request line, blank line, body.
  Headers and cookies are never logged.
- **HTTP response** (input on `ibkr/http`): status line, blank line, body.
  `parent_sequence_num` is the request.
- **HTTP error** (input on `ibkr/http`): a description of the failure.
  `parent_sequence_num` is the request.
- **Websocket frames**: the frame text, exactly.
- **Client messages**: the message as received.
- **Timer set** (output on `*/timer`): the deadline in unix nanoseconds.
- **Timer fire** (input on `*/timer`): empty. `parent_sequence_num` is the
  output that set it.
- **Run start** (`open` on `engine`): git revision, mode (live or paper) and
  payload versions.
- **Carryover** (`message` on `engine`): spreads and rules from the end of the
  previous run, so that each run replays on its own. Positions and orders are
  not carried over; they are re-fetched from IBKR.

## Pairing

- Every HTTP request is answered by exactly one input: a response or an error.
- An error means no answer arrived, not that nothing happened. An order request
  answered by an error is in an unknown state until IBKR's order list says
  otherwise. Order requests are never resent automatically.
- Timers are never cancelled. When a timer fires, the component that set it
  decides whether it still cares.

## Rules for the log writer

- **Log, then handle.** An input is written before any pure code sees it.
- **Log, then send.** An output is written before the IO shell acts on it.
- **Durability.** WAL mode with `synchronous=NORMAL`, one commit per row. Before
  an order request is sent, commit with a full sync.

## Replay

- Replay is a mode of `engined`. It reads `input` in `sequence_num` order and
  feeds it through the same router and pure components. Clients can connect but
  are read-only.
- The regenerated outputs for each input are compared with the logged outputs
  that have that input as parent, in `sequence_num` order. The first mismatch is
  where the current code behaves differently from the recorded run.
- No new log is written during replay.

## Not decided yet

- Compression. Options: filesystem compression, zstd on `payload` with a
  trained dictionary, or archiving finished runs as compressed files.
- HTTP client. `reqwest` with redirects, cookies and decompression switched off
  is enough to start, because the log does not need connection identity.
