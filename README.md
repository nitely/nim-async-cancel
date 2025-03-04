# Async cancel

Nim stdlib asyncdispatch does not support [future cancelation](https://github.com/nim-lang/RFCs/issues/304) yet. In the meantime, we can apply the [Golang context](https://pkg.go.dev/context) approach to get explicit async canceling.

> Note: I've never needed this, and just closing some FD has worked in all cases I've encountered so far. Also, I find the approach described here too explicit, and somewhat annoying; but that's personal taste. May also not be optimal compared to a native solution.

## The problem

The following code will take 10 seconds to run. How can we cancel `foo` so it returns sooner?

```nim
import std/asyncdispatch

proc foo {.async.} =
  await sleepAsync(10_000)

proc main {.async.} =
  await foo()

discard getGlobalDispatcher()  # set up the event loop
waitFor main()
echo "ok"
```

## The solution

Enter the Golang context approach (simplified). This let us cancel `foo` immediately or at whatever point we choose to cancel it:

```nim
import std/asyncdispatch

proc foo(c: Future[void]) {.async.} =
  await sleepAsync(10_000) or c

proc main {.async.} =
  let c = newFuture[void]()
  c.complete()
  await foo(c)

discard getGlobalDispatcher()
waitFor main()
echo "ok"
```

We usually want `foo` to break immediately on cancelation to avoid running the happy path as if `await` completed succesfully. We can use `fail` to throw an `AsyncCancelError` on cancel:

```nim
import std/asyncdispatch

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc foo(c: Future[void]) {.async.} =
  try:
    await sleepAsync(10_000) or c
  except AsyncCancelError:
    echo "foo got canceled"
    raise

proc main {.async.} =
  var c = newFuture[void]()
  c.cancel()
  try:
    await foo(c)
  except AsyncCancelError:
    # ignore cancel error
    discard

discard getGlobalDispatcher()
waitFor main()
echo "ok"

# Output:
# foo got canceled
# ok
```

## Derived context

Rather than deriving a context (the child) from another context (the parent), we can use the same pattern to cancel a child when either the child or the parent are canceled:

```nim
import std/asyncdispatch

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc bar(c: Future[void]) {.async.} =
  try:
    await sleepAsync(10_000) or c
  except AsyncCancelError:
    echo "bar got canceled"
    raise

proc foo(c: Future[void]) {.async.} =
  let cc = newFuture[void]()
  cc.cancel()
  try:
    # do not `or c` here, just let the most inner future propagate the cancel error
    await bar(c or cc)
  except AsyncCancelError:
    echo "foo got canceled"
    raise

proc main {.async.} =
  # note c is never canceled
  var c = newFuture[void]()
  try:
    await foo(c)
  except AsyncCancelError:
    discard

discard getGlobalDispatcher()
waitFor main()
echo "ok"

# Output:
# bar got canceled
# foo got canceled
# ok
```

Usually pass the context around and let the most inner future propagate
the cancel error up the stack. This enables except/finally
to do clean up in proper order. A common clean up example is for an
inner future to `socket.send` an error message, before the socket is closed by a parent future.

## With Timeout

We can cancel on a timeout using `withTimeout`:

```nim
import std/asyncdispatch

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc foo(c: Future[void]) {.async.} =
  try:
    await sleepAsync(10_000) or c
  except AsyncCancelError:
    echo "foo got canceled"
    raise

proc main {.async.} =
  var c = newFuture[void]()
  try:
    let f = foo(c)
    if not await withTimeout(f, 100):
      # timeout
      c.cancel()
    await f
  except AsyncCancelError:
    discard

discard getGlobalDispatcher()
waitFor main()
echo "ok"

# Output:
# foo got canceled
# ok
```

A more ergonomic timeout:

```nim
import std/asyncdispatch

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc withTimeoutCancel(timeout: Natural): Future[void] =
  let c = newFuture[void]()
  let timeoutFuture = sleepAsync(timeout)
  timeoutFuture.callback =
    proc () =
      if not c.finished: c.cancel()
  return c

proc foo(c: Future[void]) {.async.} =
  try:
    await sleepAsync(10_000) or c
  except AsyncCancelError:
    echo "foo got canceled"
    raise

proc main {.async.} =
  var c = withTimeoutCancel(100)
  try:
    await foo(c)
  except AsyncCancelError:
    discard

discard getGlobalDispatcher()
waitFor main()
echo "ok"

# Output:
# foo got canceled
# ok
```

## Just close the FD?

An alternative to explicit canceling is to close some FD. For web servers/clients this is usually a socket. After closing the socket, trying to recv or send on it will throw an error that will propagate up the call stack. Effectively canceling the futures. The downside is the socket needs to be passed around down the call stack, which some may not like.

## (Bonus) Cancelable sleepAsync

Whenever we call sleepAsync, a timer gets registered into the "global dispatcher", and it's not removed until the timer expires. If we do `await sleepAsync(900000) or c` in the above example, a timer will live for 15 minutes before it finally expires. Under some circumstances this may not be desirable.

Here's a demonstration:

```nim
import std/asyncdispatch
import std/heapqueue

proc foo(c: Future[void]) {.async.} =
  await sleepAsync(10_000) or c

proc main {.async.} =
  var c = newFuture[void]()
  c.complete()
  for _ in 0 .. 100:
    await foo(c)

discard getGlobalDispatcher()
waitFor main()
waitFor sleepAsync(100)  # let the event loop do some work just in case
echo "has pending operations:", hasPendingOperations()
echo "timers:", getGlobalDispatcher().timers.len
echo "ok"

# Output:
# has pending operations:true
# timers:101
# ok
```

Lets make it cancelable:

```nim
import std/asyncdispatch
import std/heapqueue

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc sleepAsync(timeout: Natural, c: Future[void]) {.async.} =
  var timeLeft = timeout.int
  let ms = min(timeLeft, 100)
  while timeLeft > 0:
    if c.finished:  # avoid sleep if already canceled
      c.read()
    await sleepAsync(min(timeLeft, ms)) or c
    timeLeft -= ms

proc foo(c: Future[void]) {.async.} =
  await sleepAsync(10_000, c)

proc main {.async.} =
  var c = newFuture[void]()
  c.cancel()
  for _ in 0 .. 100:
    try:
      await foo(c)
    except AsyncCancelError:
      discard

discard getGlobalDispatcher()
waitFor main()
echo "has pending operations:", hasPendingOperations()
echo "timers:", getGlobalDispatcher().timers.len
echo "ok"

# Output:
# has pending operations:false
# timers:0
# ok
```

However, this cancelable `sleepAsync` has at least one flaw: what if we block the event loop for too long? the sleep time will accumulate the blocked time. If we sleep 15 minutes, and block for 15 minutes, the total sleep time will be 30 minutes.

Let's fix that using a monotonic clock:

```nim
import std/[asyncdispatch, heapqueue, monotimes, times]

type AsyncCancelError = object of CatchableError

proc cancel(c: Future[void]) =
  c.fail(newException(AsyncCancelError, "canceled"))

proc sleepAsync(timeout: Natural, c: Future[void]) {.async.} =
  let ms = min(timeout, 100)
  let deadline = getMonoTime()+initDuration(milliseconds=timeout)
  var timeLeft = inMilliseconds(deadline-getMonoTime()).int
  while timeLeft > 0:
    if c.finished: c.read()
    await sleepAsync(min(timeLeft, ms)) or c
    timeLeft = inMilliseconds(deadline-getMonoTime()).int

proc foo(c: Future[void]) {.async.} =
  await sleepAsync(10_000, c)

proc main {.async.} =
  var c = newFuture[void]()
  c.cancel()
  for _ in 0 .. 100:
    try:
      await foo(c)
    except AsyncCancelError:
      discard

discard getGlobalDispatcher()
waitFor main()
echo "has pending operations:", hasPendingOperations()
echo "timers:", getGlobalDispatcher().timers.len
echo "ok"

# Output:
# has pending operations:false
# timers:0
# ok
```

There is a trade off that both implementations share. Because of `min(timeout, 100)` the timer may live up to 100ms after getting canceled. We could decrease the sleep timer at the cost of increasing the event loop work load. Alternatively, we could implement an FD based timer which can be canceled right away (closing the FD).

## License

MIT
