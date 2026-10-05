# kairos

An event loop for sysl, written for a microcontroller first: numbered sources an interrupt raises,
timers on one clock, and idle and check handles — libuv's shape, in callback form, with nothing in
it a bare Cortex-M cannot run.

*Kairos* is Greek for the opportune moment: not the time on the clock, but the right time to act.

```hocon
dependencies {
  kairos { git = "github.com/sysl-lang/kairos", version = "0.1.0" }
}
```

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
| `signalled()` | whether a source is raised and not yet drained — for a `Platform` to check before it sleeps |
| `event_loop(platform)` | a `Loop` over a platform, with nothing registered |
| `ev.on(n, cb)` / `ev.off(n)` | the handler for source `n`; `on` answers `Err(OutOfRange(n))` or `Err(Taken(n))` |
| `ev.after(d, cb)` / `ev.every(d, cb)` | a one-shot or repeating timer, answering a `Timer` |
| `ev.cancel(t)` | cancel a timer; answers whether it was still pending |
| `ev.idle(cb)` / `ev.check(cb)` / `ev.remove(h)` | handles that run on every pass |
| `ev.run()` / `ev.stop()` / `ev.now()` | run until stopped or until nothing is left; the pass's time |

Every callback is a `&Fn(*Loop) -> unit`: it is handed the loop, so it can set a timer or stop the
loop without capturing it. (A sysl closure captures **by value**, so a closure that captured a `Loop`
would be holding a copy.)

**What one pass does, in order:** read the clock once; fire the timers due at that reading, earliest
first, equal deadlines in the order they were set; run the idle handles; drain the pending sources and
run each raised one's handler, lowest number first; run the check handles. Then it returns if `stop`
was called or nothing is left, and otherwise waits — not at all if an idle handle exists, a timer is
already due or a source is raised, and otherwise `idle_until` the nearest live timer, or
`idle_until(None)` if there is none.

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

## The platform

```sysl
trait Platform
    now(*self) -> Duration
    idle_until(*self, deadline: Option[Duration])
```

Time is a `sysl.time.Duration` from an origin the platform chooses, and never goes backwards.
`idle_until` waits until the deadline or until a source is signalled, whichever is first, and may
return early for any reason; `None` means no timer is pending, so only an interrupt can give the loop
work. **It must return at once if `signalled()` is already true** — the loop checks before calling,
but an interrupt can land between that check and the sleep.

Two ship:

- **`sh.sysl.kairos.sim`** — a manual clock for tests. `idle_until` jumps the clock to the deadline,
  or to the first interrupt scheduled with `irq_at(t, n)` before it, raises that source with `signal`,
  and returns. `advance(d)` stands in for a callback that takes time, and `waits()` records every
  `idle_until` deadline, which is how the suite asserts the contract above. It also panics if the loop
  reads its clock 100,000 times, so a defect that spins the loop without ever waiting fails a test
  rather than hanging the suite.
- **`sh.sysl.kairos.host`** — the host's monotonic clock, for demos. It carries `@requires(posix)` on
  its own module, so the core and a board build never ask for an operating system. It sleeps in
  slices of a millisecond, checking `signalled()` between them, so a second thread standing in for a
  peripheral is noticed within a millisecond.

**The board platform comes with the board.** It is the trait above and nothing more: `now` reads a
SysTick-driven tick count (or a free-running timer), and `idle_until` masks interrupts, returns if
`signalled()`, programs the SysTick or a timer compare for the deadline when there is one, executes
`wfi`, and unmasks. WFI wakes on a pending interrupt even while interrupts are masked, which is what
closes the window between the loop's last look and the sleep.

## What v0.1 is not

- **No async face.** sysl does not ship `async`/`await` yet; callbacks are the API, and an async face
  can be added over them later without changing these signatures.
- **No priorities.** One loop, one level. Two-priority executors (an urgent loop run from a low
  priority interrupt beside the background one) are a later version, as further `Loop`s over the
  same trait.
- **No board platform.** It comes with the hardware (an STM32H747) and is described above.
- **No libuv backend.** A `Platform` over `sh.sysl.libuv` would give the same API on a desktop with
  real I/O; it is a later package, so this one never puts `-luv` on a board's link line.

## What it needs

`requires { heap = true }`: handlers and timers are closures and the tables holding them grow — all on
the loop's side, while the program registers and schedules. The core asks for no clock and no
operating system.

## Proving it builds for the board

A program that reaches every part of the loop a firmware would -- an interrupt handler calling
`signal`, a source handler, a repeating and a one-shot timer, a check handle, `run` -- compiled for the
Cortex-M7 target:

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
import sh.sysl.kairos.{Platform, event_loop, signal, signalled}
import sysl.time.{Duration, micros, millis}

// A stand-in for the board's platform: a tick count the loop's own waits move on. The real one
// reads SysTick and executes WFI; what this proves is that the loop, its timers and its handles
// compile for the target and reach no module storage an initializer would have to fill.
struct Ticks
    at: Duration

impl Platform for Ticks
    now(*self) -> Duration = self.at

    idle_until(*self, deadline: Option[Duration])
        if signalled() then return

        deadline match
            Some(d) -> self.at = d
            None -> self.at = self.at + millis(1)

@export("DMA1_Stream0_IRQHandler")
dma_done()
    signal(0)

@export("main")
boot() -> int
    val board: &Ticks = Ticks(micros(0))
    var ev = event_loop(board)

    ev.on(0, (lp) -> lp.off(0))
    ev.every(millis(10), (lp) -> ())
    ev.after(millis(100), (lp) -> lp.stop())
    ev.check((lp) -> ())
    ev.run()
    0
```

## Testing

```
sysl test .
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .
```

Every test runs on the simulated platform: deterministic, no real sleeping, each asserting a whole
timeline of what ran and when, and the deadlines `idle_until` was handed.

## License

ISC
