# fiber-interop

**One way to write concurrent PHP that runs on every event loop.**

PHP has fibers since 8.1, and several good event loops built on them: AMPHP, ReactPHP, Revolt,
phasync. Each one has its own way to start a task, wait for it, sleep, and wait for a stream. A
library that wants to be asynchronous today picks one loop and loses the users of the others.

fiber-interop is the small layer that removes that choice. A handful of functions, one adapter per
runtime, and a conformance suite that every adapter must pass identically:

```php
use function async\{run, go, await_all};

[$a, $b] = run(function () {
    $a = go(fn() => fetch('https://example.com/a'));
    $b = go(fn() => fetch('https://example.com/b'));
    return await_all([$a, $b]);
});
```

`run()` is the only entry point: it guarantees the event loop is running for the code inside it and
returns when that code and everything it started have finished.

The same code runs unchanged on each supported loop. Install the api and the adapter for the loop
you already use; nothing else in your application changes.

## What this is

- **`api`**: `go`, `await`, `await_all`, `await_any`, `run`, `sleep`, `readable`, `writable`;
  an adapter interface; two exception interfaces (`CancelledException`, `TimeoutException`) that
  loops implement on their existing classes. Frozen after 1.0.
- **Adapters**: one package per runtime (`phasync`, `revolt`, `amp`, `react`, `swoole`, which
  also covers OpenSwoole; TrueAsync when it ships), each certified by the same conformance suite: identical
  output, or it does not ship. A library written against the api reaches the users of every
  runtime that has an adapter, which is the reason the adapters exist.
- **Scope**: the identity of the operation code belongs to, an HTTP request or a job, preserved
  across fibers and generators. Proposed for PHP core in the *Scope RFC*; available today as an
  extension (`ext-scope`) with a userland polyfill (`scope`) for installs without it.
- **Integrations**: blocking libraries made cooperative, built only on the api so they work under
  every loop at once, with native drivers where a runtime has one.
  [`curl`](https://github.com/fiber-interop/curl) (`async\curl\exec()` with `curl_exec()`'s contract)
  and [`guzzle`](https://github.com/fiber-interop/guzzle) (a Guzzle handler on top of it)
  and [`pg`](https://github.com/fiber-interop/pg) (`async\pg\query()` with `pg_query()`'s contract)
  and [`mysqli`](https://github.com/fiber-interop/mysqli) (`async\mysqli\query()` with `$m->query()`'s
  contract)
  and [`process`](https://github.com/fiber-interop/process) (`async\process\run()` over `proc_open()`)
  are all available.

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
3. **Interoperability by convention, not by a shared scheduler.** A suspension primitive asked to
   park a fiber it does not own runs its own loop in place instead. Loops that follow this rule
   work together. Today's loops were not built for it, so each adapter needs one small addition
   from its runtime, named in that adapter's README; until then the adapters make do, and say how.
4. **Existing code keeps its behaviour.** Nothing here changes what a program does when it does
   not use it. That applies to the Scope RFC as well: code outside a scope sees what it sees today.
5. **Owned by the loops it serves.** Adapters are meant to be maintained by the projects they
   adapt. The api is owned jointly.

## Status

Early, 0.1. The api, the Scope extension and polyfill, and adapters for AMPHP, phasync, ReactPHP,
Revolt, Swoole and OpenSwoole are on Packagist and pass the whole conformance suite; the matrix at
https://fiber-interop.github.io/conformance.html is rebuilt daily. Read `api/README.md` for the specification, and an adapter's README for how it maps.

Issues and pull requests are welcome in each repository. Maintainers of event loops: if you want
to own the adapter for your loop, open an issue in it and say so.
