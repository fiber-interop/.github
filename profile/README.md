# fiber-interop

**One way to write concurrent PHP that runs on every event loop.**

PHP has fibers since 8.1, and several good event loops built on them: AMPHP, ReactPHP, Revolt,
phasync. Each one has its own way to start a task, wait for it, sleep, and wait for a stream. A
library that wants to be asynchronous today picks one loop and loses the users of the others.

fiber-interop is the small layer that removes that choice. Six functions, one adapter per loop,
and a conformance suite that every adapter must pass identically:

```php
use function async\{go, await, sleep};

$a = go(fn() => fetch('https://example.com/a'));
$b = go(fn() => fetch('https://example.com/b'));

[$ra, $rb] = await_all([$a, $b]);
```

The same code runs unchanged on each supported loop. Install the api and the adapter for the loop
you already use; nothing else in your application changes.

## What this is

- **`api`**: `go`, `await`, `await_all`, `await_any`, `run`, `sleep`, `readable`, `writable`;
  an adapter interface; two exception interfaces (`CancelledException`, `TimeoutException`) that
  loops implement on their existing classes. Frozen after 1.0.
- **Adapters**: one package per loop (`phasync`, `revolt`, `amp`, `react`), each certified by the
  same conformance suite: identical output, or it does not ship.
- **Scope**: the identity of the operation code belongs to, an HTTP request or a job, preserved
  across fibers and generators. Proposed for PHP core in the *Scope RFC*; available today as an
  extension (`ext-scope`) with a userland polyfill (`scope`) for installs without it.
- **Integrations** (coming): `curl_exec`, `mysqli_query`, `pg_query` and others as cooperative
  functions built only on the api, so they work under every loop at once.

## What this is not

- Not an event loop, a scheduler or a framework. It has no opinion on how a loop is built.
- Not a replacement for any existing library. Amp code stays Amp code; the adapter makes it
  reachable from code that does not know it is Amp.
- Not a new execution primitive. PHP has `Fiber`; this adds identity and a shared vocabulary on
  top of it, and nothing underneath.

## Principles

1. **The surface is small and stops growing.** A standard that changes is not one.
2. **Semantics are tested, not described.** Every rule in the specification is a runnable example
   with expected output. Adapters are certified by running them.
3. **Interoperability without coordination.** A suspension primitive asked to park a fiber it does
   not own runs its own loop in place. Two loops that both follow this work together without
   either knowing the other exists.
4. **Existing code keeps its behaviour.** Nothing here changes what a program does when it does
   not use it. That applies to the Scope RFC as well: code outside a scope sees what it sees today.
5. **Owned by the loops it serves.** Adapters are meant to be maintained by the projects they
   adapt. The api is owned jointly.

## Status

Early. The api, the Scope extension and polyfill, and adapters for phasync, Revolt, AMPHP and
ReactPHP exist and pass the conformance suite (one known gap on phasync, tracked). Nothing is
tagged yet. Read `api/README.md` for the specification, and an adapter's README for how it maps.

Issues and pull requests are welcome in each repository. Maintainers of event loops: if you want
to own the adapter for your loop, open an issue in it and say so.
