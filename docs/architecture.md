# Gaulle: architecture

Design stage, November 2026. Nothing here is built yet. Each part is marked
decided, proposed or open. The numbers come from a trial that linked a Go
engine into a static Tcl/Tk host and measured it (see Evidence).

## What Gaulle is

Decided.

Gaulle is a compact Windows runtime for Tcl/Tk programs: one executable that
installs nothing. It holds a small host written in C, Tcl and Tk linked in
statically, SQLite, and an engine written in Go. Programs are written in Tcl.
The engine belongs to the runtime. It gives every program evented
operations, parallel work, an HTTPS server and more of Go's standard
library, and no program adds to it.

The runtime changes rarely. A program changes as often as it likes.

## Principles

Decided.

1. **Tcl is the program language, Tk the toolkit.** No second scripting
   language is embedded.
2. **C and Go stay underneath.** They appear only as Tcl-shaped commands,
   the palette. The palette is the compatibility boundary.
3. **One executable that installs nothing.** `wrap` makes a standalone tool
   from a program, as a console or a GUI host.
4. **A program opts in.** It begins with `package require gaulle`.
5. **Linear by default, concurrency explicit.** A command returns when it
   has its result or reaches its deadline, unless the program asked
   otherwise.
6. **Everything started is owned.** Processes, tasks and handles end with
   their owner. Ownership is not isolation: programs are trusted code and
   run with the user's rights.
7. **Errors are contract.** A failure Gaulle defines carries
   `{GAULLE DOMAIN code}`.
8. **Bounds are stated.** Frames, queues, widths and deadlines have limits,
   and the reference names them.
9. **The palette describes itself.** The runtime carries its own reference
   and a machine-readable manifest of its commands.
10. **External software is an ordinary supervised process**, in any
    language.

## Layers

Decided.

| Layer | Holds | Written in | Changes |
|---|---|---|---|
| Host | `main`, the entry check, the start of Tcl and Tk, the commands that reach the engine, SQLite, Windows interfaces that need C, and the bridge that queues an event for the Tcl thread | C | rarely |
| Engine | tasks (tickets, deadlines, cancelling), the scheduler for parallel work, the pool's plumbing, the HTTPS server, the rest of the palette's native work | Go | rarely |
| Prelude | the Tcl side of the palette, package `gaulle N` | Tcl | rarely |
| Program | an opted-in entry, its Tcl and its assets | Tcl | always |

The first three are the runtime. A change to any of them makes a new Gaulle
number, a natural number. A program changes without one.

Placement rule: a native facility is written in Go unless it has to be C. C
is for the host, the Tcl C interface, SQLite, Windows interfaces that are
only practical from C, and commands so small and so often called that the
crossing would dominate. One call from Tcl into Go and back cost 1.3
microseconds in the trial.

## Processes and threads

Decided.

```text
 director process: gaulle.exe, or a wrapped tool
 +------------------------------------------------------------------+
 |  host (C): Tcl and Tk, SQLite, the event bridge                  |
 |                                                                  |
 |  Tcl thread                 commands -->   engine (Go threads)   |
 |  program, prelude, Tk       <-- events     tasks, scheduler      |
 +---------|-----------------------------------|-----------|--------+
           |                                   | pipes     | pipes
           |                          +--------+------+    |
           |                          |               |    |
           |                    worker process  worker process   external
           |                    Tcl, host,      (as many as      program
           |                    engine          the pool is      (any
           |                          |         wide) |          language)
           |                          +---reads---+---+
           |                                      |
           +------------- writes ----------> store: one SQLite file
```

- **One Tcl thread in each process.** No other thread runs Tcl.
- **The engine never runs Tcl.** It reaches Tcl only by queueing an event
  for the Tcl thread.
- **What crosses is copied.** C keeps no pointer into Go's memory after a
  call returns, and Go keeps none into Tcl's.
- **A worker is a whole Gaulle**: the same executable in a worker role, with
  its own Tcl thread and its own engine, owned by the director, with caps on
  its memory and CPU time.

## Linear and evented

Decided.

Linear is the default. Every command that can take long can also be run the
evented way, and all of that goes through one mechanism in the engine, the
task:

1. The call returns a ticket at once.
2. The work runs on an engine thread, with a deadline if one was given.
3. The result arrives as one Tcl event: `ok 1` and the result, or `ok 0`
   with the code and the message.
4. Cancelling by the ticket stops the work at its next cancellation point. A
   result that arrives after that is dropped.

A ticket has an owner and is closed or cancelled explicitly. A panic in a
task is recovered at the task's entry and becomes a failed result.

One command starts any palette call, which is handed over as a list. No
palette command has an evented twin of its own:

```tcl
# linear, the default
set page [http get $url]

# evented: one command starts any palette call
set ticket [task start {http get $url} -then showPage]
task cancel $ticket
```

The result can be taken in three ways, all thin layers over the same ticket:

```tcl
# a callback, through the event loop
task start {http get $url} -then showPage

# a blocking wait, for tools without a window
set ticket [task start {http get $url}]
set page [task wait $ticket]

# await, inside a coroutine: this line pauses, the event loop goes on,
# and a failure is an ordinary Tcl error
set page [await {http get $url}]
```

## Parallel work

Decided.

The program's Tcl says what to run. The engine's scheduler fans it out,
bounds it, cancels it, and returns the results in the order of the items.
One command does this for both kinds of work:

```tcl
# palette calls on goroutines, 16 at a time
set prints [task map {hash file sha256} $files -width 16]

# the program's own operations, in worker processes
set months [task map {rebuild month} $ranges -pool $pool]
```

There are two executors:

| | Runs | Where | For |
|---|---|---|---|
| Engine tasks | Go or C code that is part of the runtime | goroutines, inside the process | palette calls: HTTP, hashing, files, SQL, codecs |
| Worker operations | the program's Tcl | worker processes | the program's own logic |

Whether an item runs on a goroutine or in a worker depends only on whether
the operation is the runtime's or the program's. A goroutine never runs the
program's Tcl.

The pool of worker processes:

- **Named operations, installed once.** A worker holds the program's
  operations. A request carries data and no code.
- **The dispatcher is in the engine.** It hands the next request to a free
  worker, reads the replies, and notices a worker that died or ran late. The
  workers keep flowing while the director's Tcl is busy.
- **A pool is warm.** It serves batch after batch.
- **Workers are replaced before they age.** A long-running Tcl process slows
  as it goes (see Evidence), so a worker is replaced after a stated number
  of requests or a stated time, between requests.
- **Rows do not travel.** A request carries keys and ranges. A worker reads
  its own slice from the store, and a large result goes to the store or to a
  file.
- **The host takes the worker role.** The runtime sees a reserved argument
  before it starts Tcl, starts Tcl without Tk and without a window, loads
  the file that holds the program's operations, and serves. Any other
  command line can still be a worker, such as a script in another language
  that speaks the worker protocol.

```tcl
# ops.tcl, loaded by every worker
operation rebuild {month} {
    # ordinary Tcl: read the month from the store, work it out
    return $result
}

# the director
set pool   [pool create -width 12 -operations ops.tcl]
set months [task map {rebuild month} $ranges -pool $pool]
pool close $pool
```

## Vocabulary

Decided.

| Word | Meaning |
|---|---|
| `task` | the command for evented work: `task start`, `task wait`, `task cancel`, `task info`, `task map` |
| ticket | what `task start` returns at once; it is waited for, looked at or cancelled |
| `-then` | the option that names the script to call when the result is there |
| `await` | inside a coroutine: pause this line until the result is there |
| `task map` | fan one call out over many items, results in the order of the items |
| `pool` | the group of worker processes: `pool create`, `pool close` |
| `operation` | in the file every worker loads: declares one thing a worker can be asked to do |

## The store

Decided.

Gaulle has full SQLite: the library linked in statically, with its complete
Tcl interface. The engine reaches the same library for the evented form of
statements and for reading in parallel. In a pool the director writes and
the workers read, each worker opening the same file itself.

## Network and standard library

Decided.

- **The HTTPS server** comes from Go's standard library, from the first
  version. It hands each request to Tcl as an event and takes Tcl's answer
  now or later. Whoever calls it bounds its connections and tries again
  after a refusal: Windows gives a Go listener 200 waiting connections (see
  Evidence).
- **The HTTPS client is Windows' own.** It follows the system's proxy and
  signs in with the Windows user's credentials by itself, which Go's client
  does not. It runs through tasks like any long command. Go's client can be
  added later if a need shows.
- **From Go's standard library, in the first version:** JSON and CSV;
  hashing and encryption; fast file walks, with hashing of many files at
  once; encrypted connections for protocols other than HTTP, and name
  lookups; and gauges of the engine itself.
- **Left out for now:** compression and archives, images, and Go's
  templates, pattern matching and time zones. Tcl has its own for most of
  these.

## Also in the first version

Decided. None of it is tried.

1. **Data straight into the store.** A command decodes JSON or CSV in the
   engine and inserts the rows into SQLite without their becoming Tcl
   values. The raw answer can be stored beside them.
2. **A table for a million rows.** A grid tied to a query asks the store
   only for the rows on screen. Sorting and filtering happen in SQL.
3. **Where the time goes.** A freeze report says which Tcl procedure was
   running when the event loop has not turned for a stated time. A sampling
   profiler notes every few milliseconds where Tcl is.
4. **Tests that stay off the screen.** A test runner that fails on leaked
   timers, tasks and processes; every window of a test run on a hidden
   desktop; a command that captures a window as a picture; and an HTTP layer
   that records answers once and replays them.
5. **A door for an AI agent.** A program's operations can be offered to an
   outside agent as tools over a local connection: off by default, each run
   in a worker under the pool's limits, with read-only questions to the
   store. No AI is inside the runtime.
6. **Sign-in and secrets.** A store for sign-in material that only the
   Windows user can read, and a sign-in window that the runtime owns.
7. **Spreadsheets in and out.** Reading and writing `.xlsx`, and copying a
   table so that it pastes as a table.
8. **SQLite built on purpose.** Text search, the interval index, JSON,
   maths, backup and change sets are named parts of the build.
9. **One running instance** of a program. A second start hands its
   arguments to the first.
10. **A diagnostics file** that a user can send to the program's maker.
11. **Updating from a share**, with a check of the signature.

## Not in Gaulle

Decided.

- Tcl on threads inside one process.
- A second scripting language.
- Go that belongs to a program. Go cannot load code at run time on Windows,
  so whatever is Go is the runtime's.
- Compute kernels sent by a program, and a column library.
- Confined Tcl, on hold.

## What this shape gives up

A program's heavy work is Tcl, so it gets the cores and not Go's speed. From
the trial, for one rebuild of a year of made-up event history:

| How | Time |
|---|---|
| Tcl, one process | 12.8 to 13.7 s |
| Tcl, twelve worker processes | 2.2 s |
| Go, one goroutine | 380 to 420 ms |
| Go, twelve goroutines | 80 to 110 ms |

When parallel Tcl is not enough for some computation, the ways out are, in
this order: say it in SQL; a supervised external program; or a narrow
palette command, which is a change to the runtime.

## Evidence

A trial in November 2026, on one laptop (Intel i7-1255U, 2 fast and 8 slow
cores, Windows 11), with Go 1.26, GCC 16.1 and Tcl/Tk 9.0:

- **It links and runs.** A Go engine built as a C archive links into a
  static Tcl/Tk host as one executable. Go adds no DLL import.
- **Size.** Go adds 1.3 MB for its runtime, 1.8 MB with an engine that
  sorts, formats and locks, and 6.9 MB with an HTTPS client and server.
- **The boundary holds.** A million results were posted from goroutines to
  Tcl with the working set level. One call from Tcl into Go and back costs
  1.3 microseconds.
- **Faults.** A Go panic recovered at the engine's entry reaches Tcl as a
  message. One that nothing recovers ends the whole program.
- **HTTPS.** Client and server over HTTP/2 and TLS 1.3, 2,000 requests at
  once answered by Tcl in 51 to 86 ms, a request cancelled after 100 ms
  ended at 102 to 106 ms.
- **The listener queue.** Without a limit on the client, a burst of 2,000
  requests lost requests in some bursts; with the client held to 8
  connections being opened at a time, in none.
- **Speed.** On work that steps through rows, Tcl takes 50 to 135 times as
  long as Go.
- **Ageing.** One computation took a long-running Tcl process 92 to 99 ms
  at first and 216 to 261 ms after eleven more rounds. A new process was
  fast again.
- **Rows in Tcl.** Getting a value into Tcl costs 40 to 170 ns, and Tcl
  holds rows at nine to ten times their size.

Not tried: the engine calling the host's C on its own threads, the pool's
dispatcher in the engine, workers reading the store while the director
writes, anything through a company's proxy, and runs of hours.

## Toolchain

Decided: written from scratch, on the newest versions, each pinned by hash.

| Part | Version in November 2026 |
|---|---|
| C | C23, with GCC 16.2 (MSYS2 UCRT64) |
| Tcl/Tk | 9.1.0 |
| Go | 1.27.4 |
| SQLite | 3.54.1 |

Go fetches nothing while building, and the engine uses no third-party
module.

## Open

1. The shape of each palette command: its arguments, results and bounds.
2. Everything listed as not tried under Evidence.
