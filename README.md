# kairos

An event loop for sysl, written for a microcontroller first: numbered sources an interrupt raises,
timers on one clock, and idle and check handles — libuv's shape, in callback form, with nothing in
it a bare Cortex-M cannot run. The same loop is an async executor: tasks spawned on it sleep on its
timers and wait on its sources.

*Kairos* is Greek for the opportune moment: not the time on the clock, but the right time to act.

```hocon
dependencies {
  kairos { git = "github.com/sysl-lang/kairos", version = "0.1.5" }
}
```

kairos needs sysl 0.1.0-alpha.11 or later: `brew install sysl-lang/tap/sysl`.

```sysl
import sh.sysl.kairos.{event_loop, signal}
import sh.sysl.kairos.host.host
import sysl.time.{millis, whole_micros}
import sysl.buf.{Buf, buf}

@export("DMA1_Stream0_IRQHandler")
dma_done()
    signal(0)                                   // the whole of the interrupt's side

var ev = event_loop(host())
var polls: &Buf[int] = buf()

ev.on(0, (lp) -> print("a buffer is ready"))
ev.every(millis(100), (lp) ->
    print("tick at", whole_micros(lp.now()) / 1000, "ms")
    dma_done())                                 // a host has no DMA; stand in for one
ev.after(millis(350), (lp) -> lp.stop())
ev.check((lp) -> polls.push(1))                 // where TinyUSB's tud_task would go

ev.run()
print("passes:", polls.len())
```

```output
tick at 100 ms
a buffer is ready
tick at 200 ms
a buffer is ready
tick at 300 ms
a buffer is ready
passes: 5
```

## What v0.1 is

| | |
|---|---|
| `signal(n)` | raise source `n` (0–31). **The one function an interrupt calls.** |
| `signalled()` | whether a source is raised and not yet drained — for a `Driver` to check before it sleeps |
| `event_loop(driver)` | a `&Loop[D]` over a driver, with nothing registered |
| `ev.on(n, cb)` / `ev.off(n)` | the handler for source `n`; `on` answers `Err(OutOfRange(n))` or `Err(Taken(n))` |
| `ev.after(d, cb)` / `ev.every(d, cb)` | a one-shot or repeating timer, answering a `Timer` |
| `ev.cancel(t)` | cancel a timer; answers whether it was still pending |
| `ev.idle(cb)` / `ev.check(cb)` / `ev.remove(h)` | handles that run on every pass |
| `ev.run()` / `ev.stop()` / `ev.now()` | run until stopped or until nothing is left; the pass's time |

Every callback is a `&Fn(*Loop[D]) -> unit`: it is handed the loop, so it can set a timer or stop the
loop without capturing it.

**What one pass does, in order:** read the clock once; fire the timers due at that reading, earliest
first, equal deadlines in the order they were set; run the idle handles; drain the pending sources and
run each raised one's handler, lowest number first; step each task woken so far, once, in the order
it was woken; run the check handles. Then it returns if `stop` was called or nothing is left, and
otherwise waits — not at all if an idle handle exists, a task is woken, a timer is already due or a
source is raised, and otherwise `idle_until` the nearest live timer, or `idle_until(None)` if there is
none.

- **A source is a bit, not a queue.** Two signals before a pass run the handler once; the handler
  drains whatever its hardware buffered.
- **A source has one handler.** A second `on` for the same number is refused with `Taken` and the
  first stays; `off` frees it.
- **A repeating timer keeps exact multiples of its period**: a late pass does not push the next
  deadline back.
- **A timer set during a pass fires on a later pass**, even at a delay of zero, so a callback cannot
  keep a pass from ending. A timer may cancel itself or any other from a callback.
- **An out-of-range source number** is refused at run time by `on` with `OutOfRange`, and ignored by
  `signal`, which has nobody to refuse it to. A source is a bit of a 32-bit word and sysl has no
  type to make the bound a compile-time one without costing every call site a constructor.

## Tasks

The loop is also an executor for sysl's `async` functions, written against the language's own
contract (`step`, `park`, `yield_now`): a task runs until it parks, and the waker its park hands out
is kept where the event will find it — a timer's callback, a source's handler.

```sysl
import sh.sysl.kairos.{Loop, event_loop}
import sh.sysl.kairos.host.{Host, host}
import sysl.time.millis

async blink(lp: &Loop[Host], n: int)
    for i in 0..<n
        print("on")
        await lp.sleep(millis(100))

async sample(lp: &Loop[Host]) -> int
    val r = await lp.wait_for(0)                  // the next DMA interrupt
    if r.is_ok() then 1 else 0

val lp = event_loop(host())

lp.spawn(blink(lp, 3))
print(lp.block_on(lp.timeout(millis(250), sample(lp))))   // no DMA on a host, so it times out
```

```output
on
on
on
Ok(None)
```

| | |
|---|---|
| `lp.spawn(t)` / `lp.abort(j)` | hand a `Task[unit]` to the loop, answering a `Job`; drop it, which cancels it |
| `lp.block_on(t)` | run passes until `t` ends; `Ok(v)`, `Err(Stalled(n))` or `Err(Stopped)` |
| `await lp.sleep(d)` | end `d` after the pass that first steps it |
| `await lp.wait_for(n)` | end the next time source `n` is raised; `Err(Taken(n))` if something else handles it |
| `await lp.timeout(d, t)` | `Some` of `t`'s result, or `None` once `d` has passed, dropping `t` |
| `lp.running()` | how many spawned tasks have not finished |

- **Cancelling is dropping.** A task that is aborted, or dropped by a `timeout`, runs its `defer`s,
  and `sleep` and `wait_for` give back their timer and their source there — so a cancelled task
  leaves nothing in the loop to keep it alive.
- **A task woken during a pass's stepping is stepped on the next pass**, so a task that only yields
  takes one step a pass and cannot keep a pass from ending.
- **A parked task does not keep the loop alive by itself.** Once nothing is left that could wake it —
  no timer, source or handle — `run` cancels it, and `block_on` answers `Stalled`.
- **A task that finishes in the pass its timeout falls due wins.**
- The loop is a `&Loop[D]` because a task reaches it through the box for as long as it runs.

## Driving the loop yourself, and embedding it

`run` and `block_on` own the program's loop. A program that has a loop already — a GUI toolkit's, a
language runtime's — drives kairos one pass at a time instead:

| | |
|---|---|
| `lp.pass(NoWait)` | take in what the driver has ready, run what is due and woken, return at once |
| `lp.pass(Once)` | as `NoWait`, but first wait for the next timer, source, task or driver event if nothing is ready |
| `lp.pass(Until(d))` | as `Once`, waiting no longer than `d` |
| `lp.timeout()` | how long the embedder may wait: zero if a task is woken, an idle handle exists or a source is raised; else the nearer of the next kairos timer and the driver's own; `None` if neither sets a limit |
| `lp.backend_fd()` | the descriptor the driver's events make readable — libuv's epoll or kqueue fd on the `uv` driver, `None` elsewhere |

Each `pass` answers whether anything is left that could run, so `while lp.pass(Once)` is a serve
loop; a `stop` during a pass makes that one answer false. **A pass never cancels**: a task parked on
nothing stays parked, because the embedder may yet wake it. An embedder waits on `backend_fd` for
`timeout()` and then calls `pass(NoWait)`:

```sysl
val fd = lp.backend_fd().expect("a uv driver")

while lp.pass(NoWait)
    wait_readable(fd, lp.timeout())        // poll(2), or the toolkit's own fd watch
```

kairos's timers are not libuv's, so libuv's `uv_backend_timeout` cannot see them, nor a task already
woken; `timeout()` adds both. The `uv` driver answers `None` rather than libuv's zero for a libuv loop
with nothing alive, which would otherwise spin the embedder.

## Other threads, and background waits

A loop belongs to the thread that runs it. Another thread reaches it through a value it is handed,
and reaches that loop and no other:

| | |
|---|---|
| `lp.notifier()` | a `&Notifier`: a doorbell of this loop's own |
| `n.waker()` | a `Waker`, a copyable value safe on any thread; `w.wake()` wakes `n` and ends the loop's wait |
| `await n.next()` | ends at the next wake, or at once if one came since the last `next` took it |
| `lp.remote(k)` | a `Remote` whose `raise()` raises source `k` of this loop — not the process's — and ends its wait; `Err(OutOfRange(k))` past 31 |
| `n.background()`, `n.keep()` | stop and start counting a task waiting in `next` as keeping the loop alive; either may be called while it waits |
| `s.background()`, `s.keep()` | the same for a `uv` signal watcher, which is node's rule for a signal handler |

**Wakes coalesce**: any number of them before a `next` takes them are one, so the task woken goes and
looks at whatever the other threads left behind — a mailbox — rather than counting. One task waits
on a notifier at a time. A notifier keeps the loop alive only while a task waits on it, and not in
the background. A wake is a flag of the notifier's own, so a thousand loops on a thousand threads are
each woken by exactly the wakes sent to them.

**A background wait is left parked, not cancelled.** When nothing else is left, `run` ends rather
than waiting for it, and leaves every parked task where it is, since the wait may still end — a later
`run` or `pass` hears it. Without one, `run` cancels what nothing can wake, as before.

The wake is the driver's: `Driver.waker` answers the closure a wake calls after setting its flag. The
`uv` driver sends a `uv_async` it makes on the first ask, so a loop waiting on nothing but another
thread blocks in the kernel and uses no processor time; the host driver notices within its
millisecond slice; a board's default is nothing, since WFI wakes for any interrupt, and a second
core's is `sev`. **The loop has to outlive every waker and remote** made from it.

`signal(n)` stays the interrupt half: one word for the whole process, drained by whichever loop
passes first, which on a board is the only one. Threads use a remote.

## The interrupt contract

`signal` is one atomic OR into a word of module storage:

```sysl
private var pending: Atomic[u32] = Atomic(0)

signal(n: u32)
    if n < max_sources
        pending.or(1 << n)
```

The word is zero at reset, so it folds into `.bss` and needs no constructor — an interrupt that fires
before `main` has built the loop reads a real zero and its bit is waiting when the loop starts. On a
Cortex-M7 `pending.or` compiles to an `ldrex`/`orr`/`strex` loop. **Nothing on the interrupt path
allocates, touches a closure, or can block.** Everything else — the handler table, the timer queue,
the handles — lives on the `Loop`, and runs on the loop's side.

## The driver

```sysl
trait Driver
    now(*self) -> Duration
    idle_until(*self, deadline: Option[Duration])

    busy(*self) -> bool = false
    poll(*self) = ()

    backend_fd(*self) -> Option[int] = None
    backend_timeout(*self) -> Option[Duration] = None

    waker(*self) -> &sync Fn() -> unit = () -> ()
    quiet(*self) -> bool = false
```

The machine is a value chosen in code — `event_loop(host())`, `event_loop(sim())`, a board's — and the
loop is generic over it, so the choice costs nothing at run time and needs no package feature.

Time is a `sysl.time.Duration` from an origin the driver chooses, and never goes backwards.
`idle_until` waits until the deadline or until a source is signalled, whichever is first, and may
return early for any reason; `None` means no timer is pending, so only an interrupt can give the loop
work. **It must return at once if `signalled()` is already true** — the loop checks before calling,
but an interrupt can land between that check and the sleep.

`busy` and `poll` matter only to a driver that is itself a source of events, which today is the
libuv one: `busy` keeps the loop alive while the driver holds work a task waits on, and `poll` takes
in what has already happened, on a pass that does not idle. `backend_fd` and `backend_timeout` are
for embedding ([above](#driving-the-loop-yourself-and-embedding-it)): the descriptor an embedder
waits on, and the longest it may wait before the driver has something of its own to do. `waker` is
what a notifier or remote calls from another thread to end the driver's wait — it must neither
allocate nor block — and `quiet` says a task waits in the background on something of the driver's
([above](#other-threads-and-background-waits)).

Two ship in the core, and a third behind the `uv` feature ([below](#the-uv-feature-libuv-sockets-and-files)):

- **`sh.sysl.kairos.sim`** — a manual clock for tests. `idle_until` jumps the clock to the deadline,
  or to the first interrupt scheduled with `irq_at(t, n)` before it, raises that source with `signal`,
  and returns. `advance(d)` stands in for a callback that takes time, and `waits()` records every
  `idle_until` deadline, which is how the suite asserts the contract above. It also panics if the loop
  reads its clock 100,000 times, so a defect that spins the loop without ever waiting fails a test
  rather than hanging the suite.
- **`sh.sysl.kairos.host`** — the host's monotonic clock, for demos. It carries `@requires(posix)` on
  its own module, so the core and a board build never ask for an operating system. It sleeps in
  slices of a millisecond, checking `signalled()` and its waker's flag between them, so a second
  thread standing in for a peripheral, or waking a notifier, is noticed within a millisecond.

**The board driver comes with the board.** It is the trait above and nothing more: `now` reads a
SysTick-driven tick count (or a free-running timer), and `idle_until` masks interrupts, returns if
`signalled()`, programs the SysTick or a timer compare for the deadline when there is one, executes
`wfi`, and unmasks. WFI wakes on a pending interrupt even while interrupts are masked, which is what
closes the window between the loop's last look and the sleep.

## What v0.1 is not

- **No priorities.** One loop, one level. Two-priority executors (an urgent loop run from a low
  priority interrupt beside the background one) are a later version, as further `Loop`s over the
  same trait.
- **No board driver.** It comes with the hardware (an STM32H747) and is described above.

## The `uv` feature: libuv, sockets and files

One feature, off unless asked for. It brings the `libuv` package (so libuv has to be installed —
`brew install libuv`, Debian's `libuv1-dev`) and the module `sh.sysl.kairos.uv`; with it off that
module is empty and nothing of libuv is fetched or linked, so a board's link line never carries
`-luv`. It only adds.

```hocon
dependencies {
  kairos { git = "github.com/sysl-lang/kairos", version = "0.1.5", features = [uv] }
}
```

The machine is still a driver value — `uv()` over libuv's default loop, `uv_on(lp)` over one of the
program's own — and its I/O is tasks, awaited beside `sleep`, `wait_for` and `timeout`:

```sysl
import sh.sysl.kairos.event_loop
import sh.sysl.kairos.uv.{uv, Uv, Listener}
import sh.sysl.libuv.ip4

async echo(l: &Listener)
    val conn = (await l.accept()).expect("a connection")
    val got = (await conn.read()).expect("bytes")      // empty once the peer has closed

    (await conn.write(got)).expect("written back")
    conn.close()
    l.close()

val io = uv()
val lp = event_loop(io)
val l = io.listen(ip4("127.0.0.1", 0).expect("an address")).expect("listening")

lp.spawn(echo(l))
lp.run()
```

| | |
|---|---|
| `io.connect(addr)` | a task ending with a `&Connection`, or `ECONNREFUSED` |
| `io.listen(addr, backlog, reuse_port, ipv6_only)` | a `&Listener` at once; `l.accept()` is a task, `l.port()` the port the kernel chose. `reuse_port = true` shares the port across listeners (`ENOTSUP` where the kernel does not distribute, macOS included); `ipv6_only = true` keeps v4 clients off a v6 address |
| `conn.read()`, `conn.write(bytes)` | tasks; a dropped read consumes nothing, so `lp.timeout(d, conn.read())` loses no bytes. **A write is issued at the call**, in call order, and its task only reports how it ended, so `conn.write(a); conn.write(b); conn.close()` sends both |
| `conn.shutdown()` | issued at the call; closes this side's half once queued writes have gone, so the peer reads the end while it can still answer |
| `conn.nodelay(on)` | Nagle's algorithm off (`true`) or on |
| `conn.peer()`, `conn.local()` | the address of the other end, and of this one |
| `ByteStream` | the trait `Connection` and `Pipe` share -- `read`, `write`, `shutdown`, `close` -- so a protocol over a stream (TLS, a framing) is written once as `async f[S: ByteStream](s: &S)` |
| `io.read_file(path)`, `io.write_file(path, bytes)` | whole files, on libuv's thread pool |
| `io.resolve(host, service)` | a name lookup, on the thread pool |
| `io.open(path, flags, mode)` | a task ending with a `&File`; `f.read(n)`, `f.read_at(n, offset)`, `f.write(bytes)`, `f.write_at(bytes, offset)` and `f.close()` are tasks, and a `File` dropped open is closed |
| `io.stat`, `io.scandir`, `io.mkdir`, `io.rmdir`, `io.unlink`, `io.rename`, `io.chmod` | the file system, each a task on the thread pool |
| `io.stdin()`, `io.stdout()`, `io.stderr()`, `io.adopt(fd)` | a `&Pipe` over a descriptor this process already has: a terminal, a pipe or a socket |
| `io.socket_pair()` | two connected `&Pipe`s |
| `p.read()`, `p.write(bytes)`, `p.shutdown()` | `ByteStream`'s, on a `Pipe` as on a `Connection`; `p.is_terminal()` says which it is |
| `io.spawn(file, args, env, cwd, stdin, stdout, stderr, env_add, extra)` | a `&Process`; each descriptor is `Inherit`, `Piped`, `Ignore` or `Channel`, `extra` lists descriptors 3 on, and a piped one is `proc.stdin()`, `proc.stdout()`, `proc.stderr()` or `proc.pipe(fd)` |
| `io.channel(fd)` | the child's end of a `Channel`: a `&Pipe` that carries handles as well as bytes |
| `ch.write_handle(bytes, h)` | a task sending `PassConnection(c)`, `PassListener(l)` or `PassPipe(p)` beside `bytes` over a channel; each side keeps, and closes, its own descriptor |
| `ch.read_handles()` | a task ending with a `Message`: the bytes, and each handle sent with them as `PassedSocket(s)` (taken with `s.connection()` or `s.listen()`) or `PassedPipe(p)` |
| `proc.exit()`, `proc.kill(signum)`, `proc.pid()` | a task ending with how the child ended (`Exit`'s `status` or `signal`); `kill` after it has exited is `ESRCH`, never a signal to a process that has taken over its id |
| `io.signal(signum)`, `io.signals(signum)` | a task ending at the next arrival; a `&Signals` whose `next()` is a task per arrival, and which `background()` stops holding the loop open |
| `io.work(job)` | a task running a `&sync Fn() -> T` on the thread pool; dropping it takes a job no thread has started off the queue |
| `io.close()` | close the driver's own libuv handles — its timer and its `uv_async` — so a libuv loop of the program's own (`uv_on(l)`) can then `l.close()`, as an actor's thread does before it ends |

A failure is libuv's own `Error` (`e.name()` is `ENOENT`, `ECANCELED`, …). `close` on a connection,
listener, pipe or signal watcher ends a task waiting on it with `ECANCELED`. Closing twice does nothing, and a connection
or listener dropped without being closed is closed then, so a forgotten `close` leaks no descriptor
and does not keep the loop alive.

`idle_until` runs libuv once, with a libuv timer set for the deadline, so a socket wakes the loop as
a source does. `busy` is libuv's own answer to whether its loop is alive — an open listener, a read
or request in flight, a handle still closing — so `run` drains closes before it returns, and a task
waiting on nothing libuv watches is still cancelled when nothing else is left. A notifier's or
remote's wake from another thread sends the driver's `uv_async`, which ends libuv's wait at once;
`io.remote(n)` is gone in 0.1.5, for `lp.remote(n)` ([above](#other-threads-and-background-waits)).

## What it needs

`requires { heap = true }`: handlers and timers are closures and the tables holding them grow — all on
the loop's side, while the program registers and schedules. The core asks for no clock and no
operating system.

## Proving it builds for the board

A program that reaches every part of the loop a firmware would -- an interrupt handler calling
`signal`, a source handler, a repeating and a one-shot timer, a check handle, a task that sleeps,
waits on a source and times out, `run` -- compiled for the Cortex-M7 target:

```
sysl build-c <probe dir> --target thumbv7em-freestanding --lib <this checkout>
```

It builds, and `arm-none-eabi-objdump -d` on the archive shows `DMA1_Stream0_IRQHandler` as
`dmb; ldrex; orr; strex; bne; dmb; bx lr` and nothing else. A build that reaches module storage an
initializer has to fill is refused for a freestanding target, so a green build is also the proof that
the package has none.

**The probe is not in this repository**, because it cannot be: `sysl test .` walks every `.sysl`
under the root, a program's entry file declares no module, and there is no way to tell the walk to
leave a directory alone (a hidden directory is walked too, and `--lib .` then compiles the probe
twice). So it is here in full -- a `package.hocon` with `requires { heap = true }` beside one
`main.sysl`:

```sysl
import sh.sysl.kairos.{Driver, Loop, event_loop, signal, signalled}
import sysl.time.{Duration, micros, millis}

// A stand-in for the board's driver: a tick count the loop's own waits move on. The real one
// reads SysTick and executes WFI; what this proves is that the loop, its timers, its handles and its
// tasks compile for the target and reach no module storage an initializer would have to fill.
struct Ticks
    at: Duration

impl Driver for Ticks
    now(*self) -> Duration = self.at

    idle_until(*self, deadline: Option[Duration])
        if signalled() then return

        deadline match
            Some(d) -> self.at = d
            None -> self.at = self.at + millis(1)

@export("DMA1_Stream0_IRQHandler")
dma_done()
    signal(0)

async worker(lp: &Loop[Ticks])
    await lp.sleep(millis(5))
    val r = await lp.wait_for(1)
    val v = await lp.timeout(millis(3), lp.sleep(millis(1)))

@export("main")
boot() -> int
    val board: &Ticks = Ticks(micros(0))
    val ev = event_loop(board)

    ev.on(0, (lp) -> lp.off(0))
    ev.every(millis(10), (lp) -> ())
    ev.after(millis(100), (lp) -> lp.stop())
    ev.check((lp) -> ())
    ev.spawn(worker(ev))
    ev.run()
    0
```

The archive's symbols show every `Loop` member and every task instantiated at `Ticks` —
`sh.sysl.kairos$Loop.run.Ticks`, `sh.sysl.kairos$sleep_on.Ticks` — so the driver is chosen at
compile time and costs nothing at run time.

## Testing

```
sysl test . --no-default-features
sysl test . --features uv
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test . --features uv
```

`sysl test .` on its own turns every feature on. The core's tests run on the simulated driver:
deterministic, no real sleeping, each asserting a whole timeline of what ran and when, and the
deadlines `idle_until` was handed; one on the host driver checks that two sleeps take at least their
sum of real time; `pass` and `timeout` are checked the same way, in every mode, with a parked task
that survives empty passes and a serve loop over `pass(Once)`. The `uv` tests run on the loopback and the temporary directory — a TCP echo, every
refusal (`ECONNREFUSED`, `EADDRINUSE`, `ENOENT`, `EAI_NONAME`, `ECANCELED` on a close), a timeout
around a read nothing answers, cancellation by drop, and a yielding task that must not starve libuv —
and on child processes and pipes: exit statuses and signals, `ESRCH` for a child already gone, piped
standard streams both ways, a socket pair, signals caught by task and by stream, every file-system
call and a `File` read and written at offsets, a job on the thread pool, and a remote raised from
another thread. An embedder over `poll(2)` on `backend_fd`, waiting for `timeout()` and driving
`pass(NoWait)`, runs kairos timers, a TCP echo, a read under a timeout and a child with no timer at
all, failing on a wait that would have been for ever and on a thousand waits. Notifiers and remotes
are checked on the simulated driver for coalescing, what keeps a loop alive, background waits left
parked rather than cancelled, and no cross-talk between notifiers or loops; on the host driver across
a real thread; and on the `uv` driver with 1,024 loops on 1,024 threads, each woken alone and in turn,
with the process's processor time measured while all of them wait — and a loop waiting only on another
thread counted at a handful of passes in 300 ms, where the old one-millisecond slice made hundreds.
ASan instruments the sysl half only: libuv is a `pkg_config` library someone else compiled.

## License

ISC
