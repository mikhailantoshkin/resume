+++
title="Python is easy"
date=2022-06-25
draft=false

[extra]
disable_comments = true
+++

This story begins with me tracking down memory leaks in our python services. Yes, Python has GC, so it's rare to have memory leaks.
But when they do happen, oof, is it a doozy to hunt them down. And does this looks normal to you, for a simple web server
that basically just does `json.loads` and `json.dumps`?

![Service consumes several Gb of memory](memory_usage.jpg "Memory hog")

Yeah, it's not. So, I'm on the hunt for this illusive memory leak, when I stumble on a zombie processes in one of our 
service (you can actually see it on the screenshot above, it's processes with 0B memory usage). 

## What is a zombie anyway?
Just a quick refresher on a zombie processes (more for me, than for a reader). Zombie process is an already
terminated process, which exit code has not been red by the parent process. Zombie processes do not take up
any system resources like memory or CPU, but still occupy a place in the process table. So having a few of them is not
that big of a deal. But if your program is spawning a lot of child processes and not taking care to set up proper handling
of `SIGCHLD` (signal send by child process on its termination), either by issuing `wait` for child PID, or by ignoring 
it entirely with `SIG_IGN`, it can lead to DOS, because of PID starvation. In our case, the system
could be running for a long periods of time, so zombie processes could easily pile up.

To understand where this zombie apocalypse was coming from, we need a bit of trivia on what this service actually does. 

## How does it work?

This service is a simple aiohttp web server receiving work requests over REST and asynchronously applying them onto 
a pool of workers. Well, technically, it's a pool of _pools_ of workers. To handle everything nicely we have a wrapper around 
`multiprocessing.Pool`, called `CancellablePool`. It actually does not subclass `Pool`, it just spawns several of 
them with `processes=1`. Yes, it has overhead (crating a pool spawns threads and what not), but you don't need to implement 
`terminate` yourself and all other machinery, which is nice. 

We went with this solution for one simple reason: if user disconnects from the server it should cancel his work. 
Since there is no built-in way to shut down worker threads aside asking them nicely. Also, 
no GIL and all that stuff (it's not like we are going to be GIL limited, but why not blame it for a good measure, right?).

So the work loop for the server is simple: user connects, his work moves on the one of the worker processes, 
if user disconnects the process is terminated and new one is spawned, otherwise user gets result from the worker. 
As it happens, one workload needs to use some system binary, so we just run it with [pexpect](https://github.com/pexpect/pexpect). 
As you might already guess, when worker is killed with `SIGTERM` the process spawned with pexepct becomes a zombie.

So the fix should simple, register signal handler to catch `SIGTERM`, kill the child process and then `sys.exit()`. 
This fix should take 5 minutes, what could go wrong? So I apply the fix, push it to our repo, fight a linter and then deploy to one 
of our testing VMs. Like a _good developer_, who actually tests his code before it gets to QA. And yeah, it works as 
expected, I send command, worker starts execution, spawns a child process, I cancel the command, worker catches `SIGTERM`,
kills the child process and exits. All as planned. 

And then whole service shuts down. 

Huh? That's... Unexpected. I try it again, but without canceling the request, and it works fine. Okay, so, canceling
the request is shutting down the web server, that's weird... 
Does that `sys.exit()` in worker process propagate to parent? It does not, does it? 

![Nataly Portman form the meme "is it?"](is_it.jpg "funny")

> Spoiler, it kinda does, but not in the way you think. 

So the first thing I did is changed multiprocessing start method to `spawn`, because it's what I always do when debugging
services with multiprocessing involved, just for a good measure. And it... just fixed the bug? Huh? Well, that's odd, but 
at leas I have a fix, if I don't find anything.

Okay, time to debug this. I roll my sleeves and get to work. Putting `try except` all over the place didn't catch anything interesting. 
After some poking here and there, and finding absolutely nothing, I decide to switch `sys.exit()` with just raising a 
random exception. And that deadlocks the server. 

Wait, what? 

At this point I've spent about an hour on this and got a bit frustrated. So, to distract myself, I went on a little side
trip to confirm my theory on why exactly raising anything but `SystemExit` from worker deadlocks the `multiprocessing.Pool`.

> Originally I planned to include it here, but it quickly grew into way to big of a tangent, so I decided to move it 
> to a separate article, you can read it [here](@/posts/pool_freeze.md)

After that refreshing hunt, I've returned to main problem. Webserver is shutting down when client
disconnects, and I don't know why. As one of my colleagues likes to say:

> When in doubt - use strace

I guess the time has come.

## Strace for the rescue

In short, `sctrace` allows you to see all system calls made by a process and all signals it's receiving. Perfect 
tool for the job. So I've dropped amount of workers in the pool to 1 and went to town. I've started with simple 
`strace -e 'trace=!all'` on main process.
```bash
sudo strace -p 1116642 -e 'trace=!all'
strace: Process 1116642 attached
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=7, si_uid=999, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=12, si_uid=999, si_status=0, si_utime=0, si_stime=0} ---
+++ exited with 0 +++
```
> Don't be surprised by PIDs you'll see in strace output. The web server is in a docker container, so it will always have 
> PID 1, but in the system it will be different. That is just how docker works.

Huh? Two SIGCHLD? And no SIGTERM? Why does it shut down then? Okay, I guess it's not verbose enough. Also, let's run 
it on the worker too, for the good measure. Both processes do a lot of `read`, `stat` and `epoll`, so I'll spare you the fun
and leave only the interesting parts. Let's look at the worker first.
```bash,hl_lines=4 7 9 14-15
strace: Process 1273547 attached
...
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=50000}) = ? ERESTARTNOHAND (To be restarted if no handler)
--- SIGTERM {si_signo=SIGTERM, si_code=SI_USER, si_pid=1, si_uid=999} ---
write(5, "\17", 1)                      = 1
...
kill(18, SIGHUP)                        = 0
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=100000}) = ? ERESTARTNOHAND (To be restarted if no handler)
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=18, si_uid=999, si_status=SIGHUP, si_utime=4, si_stime=5} ---
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=99681}) = 0 (Timeout)
wait4(18, [{WIFSIGNALED(s) && WTERMSIG(s) == SIGHUP}], WNOHANG, NULL) = 18
getpid()                                = 7
write(1, "{\"level\": \"INFO\", \"time\": \"2022-"..., 223) = 223
rt_sigaction(SIGINT, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, 8) = 0
rt_sigaction(SIGTERM, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, 8) = 0
close(15)                               = 0
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=100000}) = 0 (Timeout)
exit_group(0)                           = ?
+++ exited with 0 +++
```
So, nothing unusual here. It received `SIGTERM`, issued `kill` syscall for PID 18 (it's a PID of process spawned with `pexpect`),
received `SIGCHLD`, reset signal handlers for `SIGINT` and `SIGTERM` (because you should always do that) and exited.

Okay, so nothing fishy here, let's take a look at main process. It has a lot more output (well, duh, it runs web server)
```bash,hl_lines=5 13 18 22 28 34 43 49-50 61
strace: Process 1273456 attached
...
futex(0x7f1069489608, FUTEX_WAKE_PRIVATE, 1) = 1
wait4(7, 0x7fffab17b66c, WNOHANG, NULL) = 0
kill(7, SIGTERM)                        = 0
futex(0x7f106948960c, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7f1050000b60, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY) = 0
getpid()                                = 1
wait4(7, 0x7fffab17b74c, WNOHANG, NULL) = 0
getpid()                                = 1
wait4(7, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 7
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=7, si_uid=999, si_status=0, si_utime=0, si_stime=0} ---
pipe2([14, 16], O_CLOEXEC)              = 0
getpid()                                = 1
...
pipe2([23, 24], O_CLOEXEC)              = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f1068d9ba10) = 19
close(22)                               = 0
close(23)                               = 0
getpid()                                = 1
clone(child_stack=0x7f105f7fdfb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[20], tls=0x7f105f7fe700, child_tidptr=0x7f105f7fe9d0) = 20
futex(0x559ba36ae2a0, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY) = 0
futex(0x7f1069489608, FUTEX_WAIT_BITSET_PRIVATE, 0, {tv_sec=248387, tv_nsec=464054828}, FUTEX_BITSET_MATCH_ANY) = 0
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
futex(0x7f106948960c, FUTEX_WAIT_BITSET_PRIVATE, 0, {tv_sec=248387, tv_nsec=464096937}, FUTEX_BITSET_MATCH_ANY) = 0
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
clone(child_stack=0x7f105fffefb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[21], tls=0x7f105ffff700, child_tidptr=0x7f105ffff9d0) = 21
futex(0x7f1069489608, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7f1058000b60, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY) = 0
futex(0x7f106948960c, FUTEX_WAIT_BITSET_PRIVATE, 0, {tv_sec=248387, tv_nsec=464339836}, FUTEX_BITSET_MATCH_ANY) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
clone(child_stack=0x7f1064a28fb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[22], tls=0x7f1064a29700, child_tidptr=0x7f1064a299d0) = 22
futex(0x7f1069489608, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x559ba36ae270, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f106948960c, FUTEX_WAIT_BITSET_PRIVATE, 0, {tv_sec=248387, tv_nsec=464498531}, FUTEX_BITSET_MATCH_ANY) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
getpid()                                = 1
...
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=19, si_uid=999, si_status=0, si_utime=0, si_stime=0} ---
futex(0x7f1069489610, FUTEX_WAKE_PRIVATE, 1) = 0
write(20, "\0\0\0\4\200\4N.", 8)        = 8
write(18, "\0\0\0\4\200\4N.", 8)        = 8
wait4(19, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 19
getpid()                                = 1
rt_sigaction(SIGINT, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK|SA_RESTART, sa_restorer=0x7f1068f4cd60}, 8) = 0
rt_sigaction(SIGTERM, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK|SA_RESTART, sa_restorer=0x7f1068f4cd60}, 8) = 0
stat("/usr/local/lib/python3.10/asyncio/events.py", {st_mode=S_IFREG|0644, st_size=27224, ...}) = 0
...
epoll_wait(3, [], 1, 0)                 = 0
getpid()                                = 1
epoll_ctl(3, EPOLL_CTL_DEL, 4, 0x7fffab17c9c4) = 0
close(4)                                = 0
close(5)                                = 0
close(3)                                = 0
getpid()                                = 1
write(1, "{\"level\": \"INFO\", \"time\": \"2022-"..., 190) = 190
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, {sa_handler=0x7f10691ba043, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f1068f4cd60}, 8) = 0
close(14)                               = 0
close(16)                               = 0
close(17)                               = 0
close(18)                               = 0
close(19)                               = 0
close(20)                               = 0
close(6)                                = 0
close(7)                                = 0
close(8)                                = 0
close(10)                               = 0
close(9)                                = 0
close(11)                               = 0
munmap(0x7f10694bd000, 32)              = 0
munmap(0x7f1069493000, 32)              = 0
munmap(0x7f1069492000, 32)              = 0
munmap(0x7f1068b1b000, 32)              = 0
munmap(0x7f1068b1a000, 32)              = 0
munmap(0x7f1067e69000, 32)              = 0
munmap(0x7f1064128000, 32)              = 0
munmap(0x7f1064127000, 32)              = 0
munmap(0x7f1064126000, 32)              = 0
munmap(0x7f1064125000, 32)              = 0
munmap(0x7f1064124000, 32)              = 0
munmap(0x7f1064123000, 32)              = 0
brk(0x559ba3b03000)                     = 0x559ba3b03000
munmap(0x7f1065542000, 1314816)         = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```
Here we have a `kill(7, SIGTERM)` followed by `SIGCHLD` form PID 7, which is a worker process, so everything as expected.
And after that we have several `clone` syscalls, so it spins another `multiprocessing.Pool`, as expected. Then a lot of
`stat`, `write` and all the `epoll` related business. And then `SIGCHLD` for the newly started pool, followed by 
resetting of signal handlers, some more `stat`, some `write` of logs to the terminal with final `rt_sigaction` to set
`SIGINT` to `SIG_DFL`. And finally `close` of some file descriptors followed by `munmap` and exit with 0. That is not what
I was expecting to see, to be honest.

I ran `strace` a few more times to see if something would change when leaving zombie process and running 
everything with `spawn`, but no, nothing major, it just shows me what I already can see in logs... Dead end. 
But at leas we know that no one is sending `SIGTERM` to the parent process. But it still shuts down for some reason. 

At this point I got desperate enough to start mingling with source code of aiohttp. Since it's Python, you can change 
the source code of any installed library (if it is not a C extension). So that is exactly what I did.

## Hacky hacks

At this point I'm pretty sure that web server itself is the problem. We use `aiohttp.web.run_app()` [function](https://github.com/aio-libs/aiohttp/blob/3.8/aiohttp/web.py#L460)
to set up and run aiohttp web server, so let's take a look how it knows it's time to shut down. The interesting part is [here](https://github.com/aio-libs/aiohttp/blob/3.8/aiohttp/web.py#L512)

```python
# snip
try:
    asyncio.set_event_loop(loop)
    loop.run_until_complete(main_task)
except (GracefulExit, KeyboardInterrupt):  # pragma: no cover
    pass
finally:
    # cleanup
```
Okay, so it waits for `KeyboardInterrupt`, which can be [raised after any bytecode instruction](https://docs.python.org/3/library/signal.html?highlight=signal#note-on-signal-handlers-and-exceptions),
and a `GracefulExit`. Let's see what kind of exception is this [`GracefulExit`](https://github.com/aio-libs/aiohttp/blob/3.8/aiohttp/web_runner.py#L31)
```python
class GracefulExit(SystemExit):
    code = 1

def _raise_graceful_exit() -> None:
    raise GracefulExit()
```
Oh, it's a subclass of `SystemExit`. And it is used in a mysterious `_raise_graceful_exit()` function. Which itself
is only used in `BaseRunner` as a [signal handler](https://github.com/aio-libs/aiohttp/blob/3.8/aiohttp/web_runner.py#L273)... 
Too much signal handlers for one day, in my opinion.
```python
async def setup(self) -> None:
    loop = asyncio.get_event_loop()

    if self._handle_signals:
        try:
            loop.add_signal_handler(signal.SIGINT, _raise_graceful_exit)
            loop.add_signal_handler(signal.SIGTERM, _raise_graceful_exit)
        except NotImplementedError:  # pragma: no cover
            # add_signal_handler is not implemented on Windows
            pass

    self._server = await self._make_server()
```

So okay, the only way to stop aiohttp web server, started with `run_app`, is to either get `KeyboardInterrupt`, `GracefulExit` or 
`SystemExit`. Let's hack source code for `run_app` a bit to see which one it actually is
```python,hl_lines=6-7
# snip
try:
    asyncio.set_event_loop(loop)
    loop.run_until_complete(main_task)
except (GracefulExit, KeyboardInterrupt):  # pragma: no cover
    import traceback
    traceback.print_exc()
finally:
    # cleanup
```

And now when we cancel the request we see this
```python
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/aiohttp/web.py", line 514, in run_app
    loop.run_until_complete(main_task)
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 628, in run_until_complete
    self.run_forever()
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 595, in run_forever
    self._run_once()
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 1873, in _run_once
    handle._run()
  File "/usr/local/lib/python3.10/asyncio/events.py", line 80, in _run
    self._context.run(self._callback, *self._args)
  File "/usr/local/lib/python3.10/site-packages/aiohttp/web_runner.py", line 37, in _raise_graceful_exit
    raise GracefulExit()
aiohttp.web_runner.GracefulExit
```
Okay so it is a `GracefulExit`. But it only can be triggered from a signal handler set on the event loop by aiohttp, and
we saw the `strace` output, there is no `SIGTERM` or `SIGINT` sent, it just does not make sense...

## Final push

Okay, so problem is definitely with the web server. Let's peel off some abstraction layers. `run_app` sets up a lot of 
stuff for you, so let's do it ourselves. So this
```python
logger = logging.getLogger(__name__)

handlers = [view('/execute', ExecuteHandler)]


async def process_pool_context(app: Application) -> AsyncGenerator:
    app['process_pool'] = CancellablePool(max_workers=SETTINGS.max_workers)

    yield

    app['process_pool'].shutdown()


def main() -> None:
    application = Application(middlewares=[error_handler_middleware])
    application.router.add_routes(handlers)

    application.cleanup_ctx.append(process_pool_context)

    logger.info('Service started')

    run_app(
        application,
        host='0.0.0.0',  # nosec possible binding to all interface
        port=8080,
    )

    logger.info('Service stopped')


if __name__ == '__main__':
    main()
```

Turns into this
```python
logger = logging.getLogger(__name__)

handlers = [view('/execute', ExecuteHandler)]

def sig_handler(loop: asyncio.AbstractEventLoop) -> None:
    logging.info('signal handler tripped')
    loop.stop()

async def process_pool_context(app: Application) -> AsyncGenerator:
    app['process_pool'] = CancellablePool(max_workers=SETTINGS.max_workers)

    yield

    app['process_pool'].shutdown()


def main() -> None:
    application = Application(middlewares=[error_handler_middleware])
    application.router.add_routes(handlers)

    application.cleanup_ctx.append(process_pool_context)
    
    loop = asyncio.get_event_loop()

    loop.add_signal_handler(signal.SIGINT, sig_handler, loop)
    # loop.add_signal_handler(signal.SIGTERM, sig_handler, loop)

    
    runner = web.AppRunner(application)
    await runner.setup()
    site = web.TCPSite(runner, '0.0.0.0', 8080)
    await site.start()
    
    logger.info('Service started')
    
    try:
        loop.run_forever()

    except KeyboardInterrupt:
        logger.info('KeyboardInterrupt')
        loop.stop()

    loop.close()

    logger.info('Service stopped')


if __name__ == '__main__':
    main()
```
Though it's not the recommended way, setting up your server like this is viable. We set signal handlers 
(I purposefully commented out the `SIGTERM` handler), set up `AppRunner` and a `TCPSite` and then just block
main thread to run the event loop. I've started the service again and send the work request not expecting anything at 
this point, because I was debugging this for 5 hours already (yeah, I'm a slowpoke). And then it worked. As it should have.
The work got canceled and the web server was still up.

So _something_ definitely trips the signal handler for `SIGTERM` which causes the web
server to shut down. But how is it possible? Since I've exhausted all other possibilities, it's time to look at the 
implementation of `loop.add_signal_handler`. While poking around in the `run_app` implementation, I've already taken a 
glance, but now was the time to actually figure out how it works. 

## In the belly of the beast
Let's look at the [source code](https://github.com/python/cpython/blob/3.10/Lib/asyncio/unix_events.py#L88) again, no
snipping this time
```python
def add_signal_handler(self, sig, callback, *args):
    """Add a handler for a signal.  UNIX only.
    Raise ValueError if the signal number is invalid or uncatchable.
    Raise RuntimeError if there is a problem setting up the handler.
    """
    if (coroutines.iscoroutine(callback) or
            coroutines.iscoroutinefunction(callback)):
        raise TypeError("coroutines cannot be used "
                        "with add_signal_handler()")
    self._check_signal(sig)
    self._check_closed()
    try:
        # set_wakeup_fd() raises ValueError if this is not the
        # main thread.  By calling it early we ensure that an
        # event loop running in another thread cannot add a signal
        # handler.
        signal.set_wakeup_fd(self._csock.fileno())
    except (ValueError, OSError) as exc:
        raise RuntimeError(str(exc))

    handle = events.Handle(callback, args, self, None)
    self._signal_handlers[sig] = handle

    try:
        # Register a dummy signal handler to ask Python to write the signal
        # number in the wakeup file descriptor. _process_self_data() will
        # read signal numbers from this file descriptor to handle signals.
        signal.signal(sig, _sighandler_noop)

        # Set SA_RESTART to limit EINTR occurrences.
        signal.siginterrupt(sig, False)
    except OSError as exc:
        del self._signal_handlers[sig]
        if not self._signal_handlers:
            try:
                signal.set_wakeup_fd(-1)
            except (ValueError, OSError) as nexc:
                logger.info('set_wakeup_fd(-1) failed: %s', nexc)

        if exc.errno == errno.EINVAL:
            raise RuntimeError(f'sig {sig} cannot be caught')
        else:
            raise
```
It does several notable things. First this `signal.set_wakeup_fd(self._csock.fileno())` business. If we look at the 
[docks](https://docs.python.org/3/library/signal.html?highlight=signal#signal.set_wakeup_fd) for it,
it states that this functions sets up wekeup file descriptor, were the signal numbers are written. This is also confirmed
by the comment for `signal.signal(sig, _sighandler_noop)`. Wait... Ain't file descriptors of a parent process get copied 
to child process when using `fork`, along with _some_ python modules?

No fucking way...

I need to check something real fast. 

As previously with aiohttp, we can hack asyncio source code. Comment says that `_process_self_data()` is in charge of 
reading signal numbers. Let's take a look at this data for 
[ourselves](https://github.com/python/cpython/blob/3.10/Lib/asyncio/unix_events.py#L81).
```python
def _process_self_data(self, data):
    print(data)
    for signum in data:
        if not signum:
            # ignore null bytes written by _write_to_self()
            continue
        self._handle_signal(signum)
```
Yeah, it's ***dirty***, but it's print-debug, what else did you expect? Okay, moment of truth. When running service we get
this output
```python
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/aiohttp/web.py", line 514, in run_app
    loop.run_until_complete(main_task)
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 628, in run_until_complete
    self.run_forever()
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 595, in run_forever
    self._run_once()
  File "/usr/local/lib/python3.10/asyncio/base_events.py", line 1873, in _run_once
    handle._run()
  File "/usr/local/lib/python3.10/asyncio/events.py", line 80, in _run
    self._context.run(self._callback, *self._args)
  File "/usr/local/lib/python3.10/site-packages/aiohttp/web_runner.py", line 37, in _raise_graceful_exit
    raise GracefulExit()
aiohttp.web_runner.GracefulExit
??? > b'\x0f\x11'
??? > b'\x00'
```
> Don't mind this `??? >`, we use json logs and that is how [fblog](https://github.com/brocode/fblog) 
> displays stuff you `print()` to the stdout

At this point I gave myself the biggest facepalm, but just to be sure let's run it through REPL

```python
>>> int.from_bytes(b'\x0f', 'big')
15
```

No way. It's a `SIGTERM` that worker just received, followed by `SIGCHLD` it got when process started with `pexpect` 
terminated!

## The explanation

Here is the summary of what is happening. When aiohttp web server is started with `run_app` it sets up signal handlers
to deal with `SIGINT` and `SIGTERM` gracefully. Because of how signal handling is implemented in asyncio, it registers
a file descriptor where all the signals are going to be written, so handlers can be called in _async_ way. By default, 
`multiprocessing` uses `fork` strategy on Linux when creating new subprocesses. When new worker is created it gets a copy
of a file descriptor, registered by parents' event loop, as well as a `signal` module settings. When it receives `SIGTERM`
from main process it writes `SIGTERM` to that file descriptor. So, by sending `SIGTERM` to child process we, basically, 
sent a `SIGTERM` to ourselves. This triggers signal handler and stops aiohttp web server.

In hindsight, I could've got a clue about this from implementation of our `CancellablePool`. It sets `initializer` 
argument for the pool to this function, otherwise workers just won't terminate.
```python
def _set_signal_handlers() -> None:
    signal.signal(signal.SIGINT, signal.default_int_handler)
    signal.signal(signal.SIGTERM, signal.SIG_DFL)
```
It resets the `_sighandler_noop` set by the `add_signal_handler`. So the fix is quite simple
```python
def _set_signal_handlers() -> None:
    signal.signal(signal.SIGINT, signal.default_int_handler)
    signal.signal(signal.SIGTERM, signal.SIG_DFL)
    signal.set_wakeup_fd(-1)
```
Also in `strace` output of the worker you can actually find a `write` syscall right after it receives `SIGTERM`
```bash
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=50000}) = ? ERESTARTNOHAND (To be restarted if no handler)
--- SIGTERM {si_signo=SIGTERM, si_code=SI_USER, si_pid=1, si_uid=999} ---
write(5, "\17", 1)                      = 1
rt_sigreturn({mask=[]})                 = -1 EINTR (Interrupted system call)
```
```python
>>> int.from_bytes("\17".encode(), 'big')
15
```

# FIN

This is as undefined behaviour as it gets. It'll be a good idea to open an issue about this for CPython, because
I haven't found one related to this problem (though I didn't look that thoroughly). In a meantime, here is a code to
reproduce this issue. You can just copy-paste it, and it should work on Linux.
```python
import asyncio
import logging
import signal
import sys
import time

from multiprocessing import context as ctx


logger = logging.getLogger(__name__)


def h():
    # This handler is going to be called even though SIGTERM was not send to the main process
    print("Caught signal in parent?")


async def main():
    loop = asyncio.get_event_loop()
    loop.add_signal_handler(signal.SIGINT, h)
    loop.add_signal_handler(signal.SIGTERM, h)
    p = ctx.Process(target=handler)
    p.start()

    await asyncio.sleep(2)
    # Send SIGTERM
    p.terminate()
    p.join()

    print('Continuing execution')
    # Pretend like we are doing some other stuff
    await asyncio.sleep(2)
    print('Done, bye!')


def handler():
    def sig_h(sig, frame):
        print(f'Caught signal in worker worker: {sig}')
        sys.exit()

    signal.signal(signal.SIGINT, sig_h)
    signal.signal(signal.SIGTERM, sig_h)
    print('Set signal handlers')
    try:
        time.sleep(10)
    finally:
        print("Cleaning up resources")
        signal.signal(signal.SIGINT, signal.default_int_handler)
        signal.signal(signal.SIGTERM, signal.SIG_DFL)


if __name__ == "__main__":
    # Uncomment this line to "fix"
    # multiprocessing.set_start_method('spawn')
    asyncio.run(main())
```