# Two sources that each want to block

**Status: design, no code.** Answers FunL's finding F2 (`funl/docs/design/graphics.md`, "What the
packages lack"): a program with a window on screen and an `async` task reading a file waits on libuv
*and* on a GUI toolkit's event queue at once, and today no kairos driver can.

## The problem

A kairos loop waits in exactly one place: `Driver.idle_until(deadline)`. The `uv` driver waits by
running libuv once (`uv_run(UV_RUN_ONCE)`, with a libuv timer set for the deadline), so while it waits,
libuv's kqueue or epoll is the only thing that can wake it. SDL3 waits by `SDL_WaitEventTimeout`,
which on macOS runs the Cocoa run loop (`nextEventMatchingMask:untilDate:`); while it waits, only a
window-system event, the timeout or an `SDL_PushEvent` from another thread can wake it. Each call
blocks the thread, and neither can see the other's wake-ups.

Three constraints shape the answer:

1. **SDL's video calls, and its wait, run on the process's main thread.** macOS refuses them from any
   other; on Linux the window system is merely happier there. FunL's instruction loop and kairos run
   on that thread, and FunL starts no thread for graphics.
2. **libuv is not thread-safe.** Every libuv call on a loop, `uv_run` included, is made on the
   thread that runs it, which is the kairos thread. The exceptions are `uv_async_send` and reading
   the backend descriptor with `poll(2)`/`select(2)`, which touches no libuv state.
3. **kairos's core runs on a Cortex-M.** Nothing here may reach the core's interrupt path (`signal`
   stays one atomic OR, no closure), and nothing threaded or POSIX may land in a module a board
   imports.

kairos already has half of it. `Loop.pass(NoWait)`, `Loop.timeout()` and `Loop.backend_fd()` let a
program that owns its loop drive kairos one pass at a time, and `uv/embed_tests.sysl` proves an
embedder that `poll(2)`s libuv's backend descriptor — kqueue's, on macOS — for `timeout()` neither
stalls nor spins. What is missing is the other side: SDL has no way to watch a descriptor, so "the
toolkit's own fd watch" in the README's embedding example does not exist for SDL, nor for webview.

## How others solve it

**(a) libuv primary; the toolkit polled from a `uv_prepare`/`uv_check` and a repeating timer.** Each
tick calls `SDL_PumpEvents`/`SDL_PollEvent`. Simple, and what most game loops do (they render every
frame anyway). The tick *is* the latency: 8 ms is a click answered within 8 ms, and an idle window
wakes 125 times a second for nothing. On macOS, while libuv sits in `kevent`, Cocoa is not run, so a
window does not redraw or resize between ticks. sysl-lang/webview's `pump` documents exactly this
trade. **Rejected for FunL**: an Icon program typically sits in `Event()` for minutes, and must cost
nothing while it does.

**(b) Toolkit primary; libuv's backend descriptor watched by a helper thread that wakes the
toolkit.** The main thread waits in the toolkit (`SDL_WaitEventTimeout`) for no longer than libuv's
next timer and kairos's next timer. A helper thread `poll(2)`s `uv_backend_fd` and, when it becomes
readable, calls the toolkit's one thread-safe wake (`SDL_PushEvent` of a user event,
`webview_dispatch`, `CFRunLoopWakeUp`, `PostMessage`). The main thread wakes, runs libuv with
`UV_RUN_NOWAIT`, which takes in what made the descriptor readable, and re-arms the helper. A
semaphore between the two stops the helper spinning on a descriptor that stays readable until the
main thread has drained it. **This is the standard trick**: Electron's `NodeBindings` (`select` on the
kqueue descriptor on macOS, `epoll_wait` on Linux, then a task posted to Chromium's UI loop and an
`embed_sem` handshake), NW.js the same, and libuv's own `test-embed.c`. Zero CPU while idle, latency
of one thread hand-off.

**(c) libuv on a thread of its own, posting completions into the toolkit's queue.** Wake-ups are
natural, but every operation a task starts — a read, a connect, a timer — is a libuv call that must
now be marshalled to the libuv thread, and every completion marshalled back. kairos's tasks, wakers
and `&Loop` are single-threaded by design, and the `uv` driver's whole API starts libuv work from the
loop's thread. **Rejected**: it rewrites the `uv` driver and puts a thread boundary under every
operation, to save the one hand-off (b) pays only on wake.

**(d) macOS: the descriptor as a run-loop source.** A `CFFileDescriptor` over libuv's kqueue
descriptor, added to the main `CFRunLoop` in `kCFRunLoopCommonModes`, fires *inside* SDL's own wait,
because SDL's Cocoa wait runs that run loop. No thread at all, and — in the common modes — libuv keeps
turning even inside Cocoa's modal loops (a live window resize, an open menu), which (b) does not
manage. GLib (`g_unix_fd_add`) and Qt (`QSocketNotifier`) offer the same, and for GTK this is the
idiomatic form. But it is one mechanism per platform and per toolkit, SDL on X11 or Wayland exposes
nothing equivalent, and it runs libuv's callbacks from inside the toolkit's dispatch rather than from
kairos's pass. **A later refinement of (b), not a replacement** (K7).

## Recommendation: (b), as a driver that composes any driver with any "pump"

kairos adds **one host-only module, `sh.sysl.kairos.pump`**: a trait for the other blocking source,
a driver that composes it with an inner driver, and the helper thread. The `Driver` trait does not
change, the `uv` driver does not change, and nothing in it names SDL.

### The pump: the other source

```text
trait Pump
    // Whether the pump has anything to wait on now: a window open, a webview alive. While false the
    // composed driver is the inner driver, unchanged.
    active(*self) -> bool

    // Block this thread until the pump has events, until `wake` is called from any thread, or for
    // `timeout`, whichever is first; `None` waits with no limit. Take in what arrived: route each
    // event to whoever waits on it and wake their tasks. May return early for any reason.
    wait(*self, timeout: Option[Duration])

    // Take in what has already arrived, without waiting.
    poll(*self)

    // Whether the pump holds work a task waits on, which keeps the loop alive (a window on screen).
    busy(*self) -> bool = false

    // The one thing callable from another thread: end a `wait` in progress, or make the next one
    // return at once. Coalescing is the pump's: at most one wake event need sit in its queue.
    waker(*self) -> &sync Fn() -> unit
```

It is the `Driver` trait's waiting half with the clock taken out — the inner driver owns time — and
a thread-safe `waker` added, which is the "external wakeup hook" in the only form that keeps `signal`
untouched: the wake belongs to the pump, never to the core.

**The SDL pump** (outside kairos, K4) is about thirty lines: `wait` is `SDL_WaitEventTimeout`
(milliseconds, rounded up, clamped to `i32`) followed by an `SDL_PollEvent` drain, routing each event
to its window's queue and settling the oldest promise waiting there; `poll` is the drain alone; the
waker pushes a user event with a reserved code, guarded by an `Atomic[bool]` so a burst of wakes
leaves one event on the queue, and the drain discards it; `active` is "SDL's events subsystem is
initialised"; `busy` is "a window is open". Its constructor refuses unless `SDL_IsMainThread()`.

### The composed driver

`pumped(inner: &D, pump: &P) -> &Pumped[D, P]` implements `Driver`:

| member | composed |
|---|---|
| `now` | `inner.now()` — one clock |
| `busy` | `inner.busy() \|\| pump.busy()` |
| `poll` | `inner.poll()`, then `pump.poll()` when active |
| `idle_until(deadline)` | **inactive pump:** `inner.idle_until(deadline)`, unchanged. **Active:** return at once if `signalled()`; arm the watcher if the inner driver has a `backend_fd` and it is not armed; `pump.wait(` the nearer of `deadline - now` and `inner.backend_timeout()` `)`; then `inner.poll()`, which takes in whatever made the descriptor readable |
| `backend_fd` | `None` — the composed driver is the program's loop, not something to embed |
| `backend_timeout` | `None` |

The inner driver's own `idle_until` is never called while the pump is active, so the inner driver
need only offer `now`, `poll`, `busy`, `backend_fd` and `backend_timeout` — exactly what the `uv`
driver already does for an embedder. Over `host()` or `sim()`, which have no descriptor, no watcher
starts, and the pump's wait is bounded by kairos's own timers alone.

### The watcher: the helper thread

Started on the first arm, joined when the composed driver is dropped. It owns two descriptors, the
inner driver's backend descriptor (read once, on the loop's thread; libuv's is fixed for the loop's
life) and the read end of a control pipe, and a gate:

```text
watcher thread:                                  loop thread, in idle_until:
    loop                                             if !armed
        wait on the gate                                 armed = true; open the gate
        if stopping: return                          pump.wait(t)
        poll([backend_fd, control], -1)              inner.poll()
        if control readable: drain; continue         if fired.swap(false): armed = false
        fired = true
        waker()
```

- **The gate is what keeps it from spinning.** libuv's descriptor stays readable until `uv_run` on
  the loop's thread takes the events in; without the gate the helper would wake the pump again and
  again in that window.
- **Nothing is lost.** The descriptor is level-triggered from the helper's side: an event that
  arrives before the helper polls makes the poll return at once.
- **A wake that lands while the loop is awake costs one empty pass**: the pump's wait returns at once,
  `inner.poll()` finds nothing, and the loop waits again. That is the price of not synchronising the
  helper with every pass, and it is paid only when libuv had work.
- **The helper calls two things**: `poll(2)` and the pump's waker. It never calls libuv, SDL's video
  API or any FunL code.
- **Infinite timeout on the helper.** libuv's own timers bound the *pump's* wait through
  `backend_timeout`, so the helper need not wake for them.

**Registration is flushed before the wait.** libuv applies a newly started watch to kqueue or epoll
only inside its next `uv_run`, so a read a task started in this pass is not yet on the descriptor the
helper watches. `pass` already ends with `if driver.busy() then driver.poll()` for exactly this
reason; `run` and `block_on` do not, because the `uv` driver's `idle_until` runs libuv itself. The
composed driver needs the same flush on that path (K2).

### Raising a source from another thread

`signal(n)` from another thread sets its bit and wakes nothing, and must stay that way — it is the
interrupt path. Over the `uv` driver, `Remote.raise()` already wakes libuv through a `uv_async`, which
makes the backend descriptor readable, which the watcher turns into a pump wake: it works unchanged.
For an inner driver with no descriptor, the composed driver offers the same value: `remote(n)`, whose
`raise` is `signal(n)` then the pump's waker (K5).

### The main thread, and how FunL's tasks keep working

**Every pass runs on the thread that called `run`, `block_on` or `pass`, and the pump's wait runs
inside that pass**, so with an SDL pump that thread must be the main thread — the SDL pump checks it
once, at construction. kairos itself stays thread-agnostic: it neither knows nor checks which thread
is main. FunL already runs its instruction loop and kairos on the main thread, so nothing moves.

- **FunL builds its loop as `event_loop(pumped(uv(), sdl_pump()))` at start-up**, before it knows
  whether the program will open a window, because a loop's driver is fixed for its life. Until SDL is
  initialised the pump is inactive and every wait is the `uv` driver's own; from the first window on,
  waits go through SDL (K6).
- **A task awaiting `w.next_event()`** parks on the canvas's queue. The pump's `wait` or `poll`, on
  the main thread, routes the SDL event to that queue and wakes the task; since the wait is the first
  step of a pass, the task is stepped in the same pass — the README's "whatever the wait took in is
  handled in the same pass".
- **A task reading a file** parks on libuv. The read completes on libuv's pool, libuv's async makes
  the backend descriptor readable, the watcher wakes SDL's wait, `inner.poll()` runs the callback that
  wakes the task, and the pass steps it.
- **A task sleeping** on a kairos timer bounds the pump's wait directly; a libuv timer bounds it
  through `backend_timeout`.
- **`event()`, the blocking form**, is `block_on(w.next_event())` — every other task keeps running
  while the program "blocks".
- **The loop stays alive while a window is open** (`pump.busy`), so a task awaiting a click is not
  cancelled as stalled; once the last window closes, one still awaiting an event on it is, as on any
  driver.

FunL's chapter says FunL starts no thread for graphics; that stays true. The watcher is kairos's,
starts only when an active pump meets a driver with a descriptor, and never runs FunL code.

### Testing, without a window

The composed driver is tested in kairos with a **pipe pump**: `wait` is `poll(2)` on a pipe with the
timeout, `waker` writes a byte, `poll` drains it. That needs no window and no SDL, so it runs from a
`Background` shell as the rest of the suite does. What it owes:

- a kairos timer ends the pump's wait on time, with no libuv activity;
- a libuv timer, a file read on libuv's pool and a TCP echo each end the pump's wait through the
  watcher alone (no timer anywhere), and each wakes exactly one pass — not zero (a stall, caught by a
  two-second cap as in `embed_tests.sysl`) and not many (a spin, caught by counting waits);
- a read a task begins in the same pass that then waits is seen (the K2 flush);
- `remote(n).raise()` from another thread ends the wait over `host()`;
- an inactive pump leaves the inner driver's behaviour exactly as it was (the existing `uv` tests,
  re-run over `pumped(uv(), inactive)`);
- dropping the driver joins the watcher (no thread left behind).

The SDL pump's own test needs only `SDL_Init(SDL_INIT_EVENTS)`, no video, so it too runs headless:
a pushed user event from a second thread ends `SDL_WaitEventTimeout` early.

## What kairos adds

1. `sh.sysl.kairos.pump` — `trait Pump`, `pumped(inner, pump)`, the watcher, `remote(n)`; carries
   `@requires(posix)` like `sh.sysl.kairos.host`, so a board never imports it; needs no feature,
   since it names neither libuv nor SDL.
2. One line in the core: `run` and `block_on` flush with `driver.poll()` after a pass while the
   driver is busy, as `pass` already does (K2).
3. README: a section after "Driving the loop yourself, and embedding it", and the module in the
   package comment's list.

## Open questions

**K1. Shape — a new driver, a composing driver, or a hook on `Driver`?**
Recommendation: **a composing driver over a new `Pump` trait.** A hook on `Driver` (an optional
`set_waker`) would make every driver, the board's included, carry a closure it can never use; a
dedicated SDL driver would be copied for webview and every toolkit after. The composition keeps
`Driver` as it is and gives each toolkit a thirty-line pump.

**K2. Should `run`/`block_on` flush registrations with `driver.poll()` after each pass while the driver
is busy, as `pass` does?**
Recommendation: **yes, in the core.** Without it a read begun in the last pass is not on the
descriptor the watcher polls, which is a stall with no timer to end it. The cost is one
`uv_run(NOWAIT)` per pass on the `uv` driver, and nothing on a driver whose `busy` is false.

**K3. Where does the module live, and behind which feature?**
Recommendation: **`sh.sysl.kairos.pump`, in the core package, `@requires(posix)` on the module, no
feature.** It uses threads and `poll(2)`, never libuv or SDL, so it costs a non-user nothing and a
board never reaches it.

**K4. Where does the SDL pump live?**
Recommendation: **in `sysl-lang/sdl3`, behind an optional `kairos` feature** (an optional dependency,
as kairos's own `uv` feature is), so `syslui-sdl`, FunL and any SDL program share one. FunL's routing
of events to canvases stays FunL's: the sdl3 pump takes a `&Fn(Event)` sink.

**K5. How does another thread raise a source when the pump is the one waiting?**
Recommendation: **`remote(n)` on the composed driver (`signal` then the pump's waker); `signal` itself
untouched.** Over `uv` the existing `Remote` already works through the watcher, so both spellings
wake.

**K6. A loop's driver is fixed at creation, but FunL learns it needs SDL only when a window opens.**
Recommendation: **`Pump.active()`; an inactive pump makes the composed driver its inner driver
exactly**, so FunL composes at start-up and a program that never opens a window pays one boolean test
per wait.

**K7. Should a pump be able to watch the descriptor natively (CFFileDescriptor, `g_unix_fd_add`,
`QSocketNotifier`) instead of the helper thread?**
Recommendation: **not now.** The helper works on every POSIX host with one mechanism; add an optional
`Pump.watch(fd) -> bool` later, answered true by a pump that took the descriptor itself, which
switches the watcher off. The case that would force it is K9.

**K8. Windows, where libuv's backend is an I/O completion port and `uv_backend_fd` answers -1.**
Recommendation: **out of scope until sysl has a Windows target**, with one rule now so it can never
stall: while the inner driver is busy and has no descriptor, the composed driver caps each pump wait
at a one-millisecond slice, as the `host` and `uv` drivers already do where nothing can wake them. (Electron's form there is `GetQueuedCompletionStatus` on the port and
re-posting the packet; the watcher is the place it would go.)

**K9. Cocoa's modal loops — a live window resize, an open menu — run inside SDL's event pump and
starve the kairos pass for their duration.**
Recommendation: **accept, and document.** Timers and libuv work resume when the gesture ends; SDL3's
event watch still delivers expose events during a resize, so a canvas can repaint. The cure, if a
program needs it, is K7's run-loop source in the common modes.

**K10. webview has a thread-safe wake (`dispatch`) but no blocking wait with a timeout — only `run`,
which keeps the thread, and `pump`, which does not wait.**
Recommendation: **send sysl-lang/webview a finding: a `wait(timeout)` in `pump.c`**
(`nextEventMatchingMask:untilDate:` + `sendEvent:` on macOS, `g_main_context_iteration` with a timeout
source on GTK). With it, webview's pump is the same thirty lines as SDL's.

**K11. sysl-lang/sdl3 binds `SDL_PushEvent` and `user_event`, but not `SDL_RegisterEvents`, so the
pump's wake would use the shared `EVENT_USER` with a reserved code.**
Recommendation: **bind `SDL_RegisterEvents` in sdl3 (a finding to that repo)**, so the pump's wake
event cannot collide with a program's own user events.
