+++
title="How to freeze a pool"
date=2022-06-26
draft=false

[extra]
disable_comments = true
+++
> Originally this was a part of [Python is easy](@/posts/python_is_easy/index.md) article, but it quickly grew way to big
> for a tangent, so I've decided to give it its own place under the sun. 

Let's imagine you are running a `multiprocessing.Pool` and want some clever stuff to happen when your worker terminates.
You should set up a signal handler with [signal](https://docs.python.org/3/library/signal.html?highlight=signal#module-signal) module
like this, and you'll be good to go

```python
def _handle_signal(sig: int, frame: Any) -> None:
    logger.info(f'Received signal {sig}')
    _cleanup()
    sys.exit()


signal.signal(signal.SIGINT, _handle_signal)
signal.signal(signal.SIGTERM, _handle_signal)
```

As you probably know, `sys.exit()` just raises `SystemExit`, so what happens when you raise some other exceptions?
First of all, ~~all hell breaks loose~~ you really should not do that. But if it happens to be a subclass of `Exception`
it will deadlock the pool.

_Almost_ all exceptions in python are subclasses of `Exception` class. It makes it very easy to just write
```python
try:
    func()
except Exception:
    logger.exception('Oh-oh something went wrong:')
    raise
```
But there was one big caveat. And this caveat was `CancelledError`. In `asyncio`, python built-in async library,
`CancelledError` signals cancellation of a coroutine or a task and can be raised on basically any `await`. So, if 
your particular workload needs to do some cleanup or logging before exiting you do it like this
```python
try:
    await afunc()
except CancelledError:
    # do some cleanup 
    raise
```

But this popular pattern suddenly becomes a problem.
```python
try:
    await afunc()
except Exception:
    pass
```
Why? Because until Python 3.8 `CancelledError` was a subclass of `Exception`. It would just straight up ignore a 
cancellation of a coroutine. And it becomes extremely dangerous when event loop is stopping, since it has to 
cancel all currently scheduled coroutines.

So async code was riddled with stuff like this:

```python
try:
    await afunc()
except CancelledError:
    raise
except SomeException:
    logger.info('bad')
```

Now `CancelledError` is a subclass of `BaseException`, so there is no need to check for it every time you want to
recover from an exception. 

But why all this tangent about `CancelledError`? Well, you see, problem with deadlock in the `multiprocessing.Pool` revolves around similar issue.
When you create a pool it starts several management threads to deal with queues and what not. And it 
also spawns worker processes, running a special [worker function](https://github.com/python/cpython/blob/3.10/Lib/multiprocessing/pool.py#L97). 
It roughly looks like this, minus exception handling and setup part.
```python
def worker(inqueue, outqueue, initializer=None, initargs=(), maxtasks=None):
    completed = 0
    while maxtasks is None or (maxtasks and completed < maxtasks):

        task = get()  # Get a job to execute
        
        if task is None:
            util.debug('worker got sentinel -- exiting')
            break

        job, i, func, args, kwds = task
        try:
            result = (True, func(*args, **kwds))  # Runs a function submitted with .apply()
        except Exception as e:
            result = (False, e)  # Stores the exception as the result to send back to the parent process
        
        put((job, i, result))  # Puts the result in finished queue
        
        # Remove references, so the data will not linger on the next iteration
        task = job = result = func = args = kwds = None  
        completed += 1
    util.debug('worker exiting after %d tasks' % completed)
```

When pool shuts down is sends `SIGTERM` to workers, and then joins them, so it won't leave any zombies.
But when you register a signal handler inside a worker process that raises some subclass of `Exception`, worker, in fact,
does not terminate. It just sends your exceptions as a result and waits for more work to come its way. But pool has already
closed all management threads and now is waiting for worker to exit. Classic deadlock. 

So yeah, be careful with setting signal handlers in worker processes. Actually, it's just one of many caveats you need 
to be wary about when using multiprocessing in Python. Some of them are described in 
[this article](https://pythonspeed.com/articles/python-multiprocessing/). 

## And one more thing
As documentation for signal module [states](https://docs.python.org/3/library/signal.html?highlight=signal#note-on-signal-handlers-and-exceptions),
if signal handler raises an exception it can be executed after _any_ bytecode instruction on the main thread. So with this
bit of information, where the memory leak occurs in `_wait_response` method?

```python
class Client:
    def __init__(self):
        # snip
        self._responses: Dict[str, Any] = {}
        self._response_events: Dict[str, Event] = {}

    # snip
    def _wait_response(self, correlation_id: str, expiration: int) -> Dict:
        if not self._response_events[correlation_id].wait(expiration):
            self._response_events.pop(correlation_id)
            raise TimeoutError
        self._response_events.pop(correlation_id)
        return self._responses.pop(correlation_id)
```
It's a trick question. Because there is no memory leak. 

Yet. 

If you set up a signal handler for `SIGALARM` like this for the worker process running `Client`
```python
def raise_timeout_error(signal, frame):
    raise RuntimeError("Time is up!")

signal.signal(signal.SIGALRM, raise_timeout_error)
signal.alarm(_timeout)
```
You will leak `Event` instances in the `_response_events` dict. If your `expiration` for `wait` method on `Event` happens 
to be a bit longer (or equal, for that matter) than `_timeout` on `SIGALARM`, `RuntimeError` will be raised from 
`wait(expiration)` call. Because `RuntimeError` is subclass of `Exception`, worker will not terminate on it, and you will
skip the resource cleanup code.

So yeah, your process pool _is_ full of sharks, and you are the one who released them.