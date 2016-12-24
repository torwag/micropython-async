# Application of uasyncio to hardware interfaces

This document is a "work in progress" as I learn the content myself. Further
at the time of writing uasyncio is itself under development. It is likely that
these notes may contain errors; they will be subject to substantial revision. 

The MicroPython uasyncio library comprises a subset of Python's asyncio library
designed for use on microcontrollers. As such it has a small RAM footprint and
fast context switching. This document describes its use with a focus on
interfacing hardware devices. Another major application area is in network
programming: many guides to this may be found online.

# 1. Cooperative scheduling

The technique of cooperative multi-tasking is widely used in embedded systems.
It offers lower overheads than pre-emptive scheduling and avoids many of the
pitfalls associated with truly asynchronous threads of execution. For those new
to asynchronous programming there is an introduction in section 7 below.

## 1.1 Modules

The following modules and test programs are provided. The first two are the
most immediately rewarding as they produce visible results by accessing Pyboard
hardware.

 1. ``aledflash.py`` Flashes the four Pyboard LED's asynchronously for 10s. The
 simplest uasyncio demo. Import it to run.
 2. ``apoll.py`` A device driver for the Pyboard accelerometer. Demonstrates
 the use of a coroutine to poll a device. Runs for 20s. Import it to run.
 3. ``aswitch.py`` This provides classes for interfacing switches and
 pushbuttons and also a software retriggerable delay object. Pushbuttons are a
 generalisation of switches providing logical rather than physical status along
 with double-clicked and long pressed events.
 4. ``astests.py`` Test/demonstration programs for the above.
 5. ``asyn.py`` Synchronisation primitives ``Lock``, ``Event``, ``Barrier`` and
 ``Semaphore``.
 6. ``asyntest.py`` Example/demo programs for above.
 7. ``roundrobin.py`` Demo of round-robin scheduling. Also a benchmark of
 scheduling performance.
 8. ``awaitable.py`` Demo of an awaitable class. One way of implementing a
 device driver which polls an interface.
 9. ``chain.py`` Copied from the Python docs. Demo of chaining coros.
 10. ``aqtest.py`` Demo of uasyncio ``Queue`` class.

# 2. uasyncio

The asyncio concept is of cooperative multi-tasking based on coroutines,
referred in this document as coros.

## 2.1 Differences from CPython

Here CPython refers to Python 3.5. In the interests of small size and
efficiency uasyncio is a subset of asyncio with some differences.

It doesn't support objects of type ``Future`` and ``Task``. Routines to run
concurrently are defined as coroutines instantiated with ``async def``.

At the time of writing the ``__await__`` special method (to create an awaitable
class) is not supported but a workround is described in section 4.1.

For timing asyncio uses floating point values of seconds. The uasyncio ``sleep``
method accepts floats (including sub-second values) or integers. For
performance reasons, and to support ports lacking floating point, uasyncio
also supports a ``sleep_ms`` method accepting integer millisecond values. For
similar reasons a ``call_later_ms_`` event loop method is provided.

## 2.2 Program structure: the event loop

Consider the following example:

```python
import uasyncio as asyncio
loop = asyncio.get_event_loop()
async def bar():
    count = 0
    while True:
        count += 1
        print(count)
        await asyncio.sleep(1)  # Pause 1s

loop.create_task(bar()) # Schedule ASAP
loop.run_forever()
```

Program execution proceeds normally until the call to ``loop.run_forever``. At
this point execution is controlled by the scheduler. A line after
``loop.run_forever`` would never be executed. The scheduler runs ``bar``
because this has been placed on the queue by ``loop.create_task``. In this
trivial example there is only one coro: ``bar``. If there were others, the
scheduler would schedule them in periods when ``bar`` was paused.

Many embedded applications have an event loop which runs continuously. The event
loop can also be started in a way which permits termination, by using the event
loop's ``run_until_complete`` method. Examples of this may be found in the
``astests.py`` module.

The event loop instance is a singleton. If a coro needs to call an event loop
method, calling ``asyncio.get_event_loop()`` will efficiently return it.

## 2.3 Coroutines (coros)

A coro is instantiated as follows:

```python
async def foo(delay_secs):
    await asyncio.sleep(delay_secs)
    print('Hello')
```

A coro can allow other coroutines to run by means of the ``await coro``
statement and must contain at least one ``awwait`` statement. This causes
``coro`` to run to completion before execution passes to the next instruction.
Consider these lines of code:

```python
await asyncio.sleep(delay_secs)
await asyncio.sleep(0)
```

The first causes the code to pause for the duration of the delay, with other
coros being scheduled for the duration. A delay of 0 causes any pending coros
to be scheduled in round-robin fashion before the following line is run. See
the ``roundrobin.py`` example.

### 2.3.1 Queueing a coro for scheduling

 * ``EventLoop.create_task`` Arg: the coro to run. Starts the coro ASAP and
 returns immediately. The coro is specified with function call syntax with any
 required arguments being passed.
 * ``await``  Arg: the coro to run, specified with function call syntax. Starts
 the coro ASAP and blocks until it has run to completion.

### 2.3.2 Running a callback function

Callbacks should be Python functions designed to complete in a short period of
time as coroutines will have no opportunity to run for the duration.

The following ``EventLoop`` methods schedule callbacks:

 1. ``call_soon`` Call as soon as possible. Args: ``callback`` the callback to
 run, ``*args`` any positional args may follow separated by commas.
 2. ``call_later`` Call after a delay in secs. Args: ``delay``, ``callback``,
 ``*args``
 3. ``call_later_ms_`` Call after a delay in ms. Args: ``delay``, ``callback``,
 ``args``. Args are stored in a tuple for efficiency. Default an empty
 tuple ``()``.
 4. ``call_at`` Call at a future time in secs. Args: ``time``, ``*args``.
 5. ``call_at_`` Call at a future time in secs. Args: ``time``, ``args``. Args
 stored in a tuple, default ``()``.

```python
loop = asyncio.get_event_loop()
loop.call_soon(foo, 5) # Schedule callback 'foo' ASAP with an arg of 5
loop.call_later(2, foo, 5) # Schedule after 2 seconds
loop.call_later_ms_(50, foo, (5,)) # Schedule after 50ms. Note arg in tuple.
loop.call_at(time.ticks_add(loop.time(), 4), foo, 5) # after 4s
loop.run_forever()
```

### 2.3.3 Returning values

A coro can contain a ``return`` statement with arbitrary return values. To
retrieve them issue:

```python
result = await my_coro()
```

## 2.4 Delays

Where a delay is required in a coro there are two options. For longer delays and
those where the duration need not be precise, the following should be used:

```python
async def foo(delay_secs, delay_ms):
    await asyncio.sleep(delay_secs)
    print('Hello')
    await asyncio.sleep_ms(delay_ms)
```

While these delays are in progress the scheduler will schedule other coros.
This is generally highly desirable, but it does introduce uncertainty in the
timing as the calling routine will only be rescheduled when the one running at
the appropriate time has yielded. The amount of latency depends on the design
of the application, but is likely to be on the order of tens of ms.

Very precise delays may be issued by using the ``utime`` functions ``sleep_ms``
and ``sleep_us``. These are best suited for short delays as the scheduler will
be unable to schedule other coros while the delay is in progress.

# 3 Synchronisation

There is often a need to provide synchronisation between coros. A common
example is to avoid what are known as "race conditions" where multiple coros
compete to access a single resource. An example is provided in the ``astests.py``
program and discussed in [the docs](./DRIVERS.md). Another hazard is the "deadly
embrace" where two coros wait on the other's completion.

In simple applications these are often addressed with global flags. A more
elegant approach is to use synchronisation primitives. The module ``asyn.py``
offers "micro" implementations of ``Lock``, ``Event``, ``Barrier`` and ``Semaphore``
primitives. These are for use only with asyncio. They are not thread safe and
should not be used with the ``_thread`` module.

Another synchronisation issue arises with producer and consumer coros. The
producer generates data which the consumer uses. Asyncio provides the ``Queue``
object. The producer puts data onto the queue while the consumer waits for its
arrival (with other coros getting scheduled for the duration). The ``Queue``
guarantees that items are removed in the order in which they were received.
Alternatively a ``Barrier`` instance can be used if the producer must wait
until the consumer is ready to access the data.

The interface specification for the primitves is [here](./PRIMITIVES.md).

## 3.1 Lock

This guarantees unique access to a shared resource. The preferred way to use it
is via an asynchronous context manager. In the following code sample a ``Lock``
instance ``lock`` has been created and is passed to all coros wishing to access
the shared resource. Each coro issues the following:

```python
async def bar(lock):
    async with lock:
        # Access resource
```

While the coro ``bar`` is accessing the resource, other coros will pause at the
``async with lock`` statement until the context manager in ``bar()`` is
complete.

## 3.2 Event

This provides a way for one or more coros to pause until another one flags them
to continue. An ``Event`` object is instantiated and passed to all coros using
it. Coros waiting on the event issue ``await event.wait()``. Execution pauses
until a coro issues ``event.set()``. ``event.clear()`` must then be issued.

In the usual case where a single coro is awaiting the event this can be done
immediately after it is received:

```python
async def eventwait(event):
    await event.wait()
    event.clear()
```

The coro raising the event may need to check that it has been serviced:

```python
async def foo(event):
    while True:
        # Acquire data from somewhere
        while event.is_set():
            await asyncio.sleep(1) # Wait for coro to respond
        event.set()
```

Especially if multiple coros are to wait on a single event, consider using a
``Barrier`` object described below. This is because the coro which raised the
event has no way to determine whether all others have received it; determining
when to clear it down requires further synchronisation. One way to achieve this
is with an acknowledge event, but each coro would need to provide one:

```python
async def eventwait(event, ack_event):
    await event.wait()
    ack_event.set()
```

An example of this is provided in the ``event_test`` function in ``asyntest.py``.

## 3.3 Barrier

This enables multiple coros to rendezvous at a particular point. For example
producer and consumer coros can synchronise at a point where the producer has
data available and the consumer is ready to use it. At that point in time the
``Barrier`` can optionally run a callback before releasing the barrier and
allowing all waiting coros to continue.

The callback can be a function or a coro. In most applications a function is
likely to be used: this can be guaranteed to run to completion before the
barrier is released.

An example is the ``barrier_test`` function in ``asyntest.py``.

## 3.4 Semaphore

A semaphore limits the number of coros which can access a resource. It can be
used to limit the number of instances of a particular coro which can run
concurrently. It performs this using an access counter which is initialised by
the constructor and decremented each time a coro acquires the semaphore.

The easiest way to use it is with a context manager:

```python
async def foo(sema):
    async with sema:
        # Limited access here
```
An example is the ``semaphore_test`` function in ``asyntest.py``.

### 3.4.1 BoundedSemaphore

This works identically to the ``Semaphore`` class except that if the ``release``
method causes the access counter to exceed its initial value, a ``ValueError``
is raised.

## 3.5 Queue

The sample program ``aqtest.py`` demonstrates simple use of this class. A
typical producer coro might work as follows:

```python
async def producer(q):
    while True:
        result = await slow_process()  # somehow get some data
        await q.put(result)  # may pause if a size limited queue fills
```

and the consumer works along these lines:

```python
async def consumer(q):
    while True:
        result = await(q.get())  # Will pause if q is empty
        print('Result was {}'.format(result))
```

The ``Queue`` class provides significant additional functionality in that the
size of queues may be limited and the status may be interrogated. The behaviour
on empty status and (where size is limited) the behaviour on full status may be
controlled. Documentation of this is in the code.

An example of its use is provided in ``aqtest.py``.

# 4 Designing classes for asyncio

## 4.1 Awaitable classes

A coro can pause execution by waiting on an ``awaitable`` object:
``await asyncio.sleep(delay_secs)`` is an example. Under CPython a custom class
is made ``awaitable`` by implementing an ``__await__`` special method. This
returns a generator. An awaitable class is used as follows:

```python
async def bar():
    foo = Foo()  # Foo is an awaitable class
    print('waiting for foo')
    await foo
    print('done')
```

Execution proceeds to ``print('done')`` when the generator terminates.
Currently MicroPython doesn't support ``__await__`` (issue #2678) and
``__iter__`` must be used. Portability between CPython and MicroPython may be
achieved as follows:

```python
class Foo():
    def __await__(self):
        for n in range(5):
            print('__await__ called')
            yield

    __iter__ = __await__  # workround for issue #2678
```

## 4.2 Asynchronous context managers

Classes can be designed to support asynchronous context managers. These are CM's
having enter and exit procdures which are coros. An example is the ``Lock``
class described above. This has an ``__aenter__`` coro which is logically
required to run asynchronously. To support the asynchronous CM protocol its
``__aexit__`` method also must be a coro, achieved by including
``await asyncio.sleep(0)``. Such classes are accessed from within a coro with
the following syntax:

```python
async def bar(lock):
    async with lock:
        print('bar acquired lock')
```

As with normal context managers an exit method is guaranteed to be called when
the context manager terminates, whether in the normal way or via an exception.
To achieve this the special methods ``__aenter__`` and ``__aexit__`` must be
defined, both being coros waiting on an awaitable object. This example comes
from the ``Lock`` class:

```python
    async def __aenter__(self):
        await self.acquire()  # a coro defined with async def

    async def __aexit__(self, *args):
        self.release()  # A conventional method
        await asyncio.sleep_ms(0)
```

# 5 Device driver examples

Many devices such as sensors are basically read-only in nature and need to be
polled to acquire data. There are two ways to do this using asyncio. One is
simply to have a coro which does this periodically. The other is to delegate
the polling to the scheduler using the IORead mechanism. The latter is more
efficient, especially for devices which need to be polled frequently or with
a (fairly) repeatable polling interval. At the time of writing the latter
approch is not yet supported in the firmware.

Note that where a very repeatable polling interval is required, it should be
done using a timer callback. For "very" repeatable read microsecond level.
"Fairly" repeatable is application dependent but likely to be variable on the
order of tens of milliseconds: the latency being determined by the coro with
the longest run time between yields.

## 5.1 Using a coro to poll hardware

This is a simple approach, but is only appropriate to hardware which is to be
polled at a relatively low rate. This is for two reasons. Firstly the variable
latency caused by the execution of other coros will result in variable polling
intervals - this may or may not matter depending on the application. Secondly,
attempting to poll at high speed may cause the coro to consume more processor
time than is desirable.

The example ``apoll.py`` demonstrates this approach by polling the Pyboard
accelerometer at 100ms intervals. It performs some simple filtering to ignore
noisy samples and prints a message every two seconds if the board is not moved.

Further examples may be found in ``aswitch.py`` which provides drivers for
switch and pushbutton devices.

## 5.2 Using IORead to poll hardware

The uasyncio ``IORead`` class is provided to support IO to stream devices. It
may be employed by drivers of devices which need to be polled: the polling will
be delegated to the scheduler which uses ``select`` to schedule the first
stream or device driver to be ready. This is more efficient, and offers lower
latency, than running multiple coros each polling a device.

At the time of writing firmware support for using this mechanism in device
drivers has not yet been implemented.

# 6 Hints and tips

## 6.1 Program hangs

Hanging usually occurs because a thread has blocked without yielding: this will
hang the entire system. When developing it is useful to have a coro which
periodically toggles an onboard LED. This provides confirmtion that the
scheduler is running.

## 6.2 uasyncio retains state

When running programs using ``uasyncio`` at the REPL, issue a soft reset
(ctrl-D) between runs. This is because ``uasyncio`` retains state between runs
which can lead to confusing behaviour.

# 7 Notes for beginners

These notes are intended for those unfamiliar with asynchronous code or unsure
of the relative merits of asyncio and the _thread module (i.e. cooperative vs
pre-emptive scheduling).

## 7.1 Why Scheduling?

Using a scheduler doesn't enable anything that can't be done with conventional
code. But it does make the solution of certain types of problem simpler to code
and easier to read and maintain.

It facilitates a style of programming based on the concept of routines offering
the illusion of running concurrently. They do this by periodically passing
control to the scheduler which will allow another routine waiting for execution
to run - until it in turn yields control. This can simplify the process of
interacting with physical devices. Consider the task of reading 12
push-buttons. Mechanical switches such as buttons suffer from contact bounce.
This means that several rapidly repeating transitions can occur when the button
is pushed or released.

A simple way to overcome this is as follows. On receipt of the first transition
perform any programmed action. Then wait (typically 50ms) and read the state
of the button. By then the bouncing will be over and its state can be stored to
detect future transitions. Doing this in linear code for 12 buttons can get
messy. If you extend this to support long press or double-click events the code
will get positively convoluted. Using asyncio with the ``aswitch.py`` module we
can write:

```python
async def cb(button_no):  # user code omitted. This runs when
                    # button pressed, with the button number passed

buttons = ('X1', 'X2', 'X3', 'X4', 'X5', 'X6', 'X7', 'X8', 'X9', 'X10', 'X11', 'X12')
for button_no, button in enumerate(buttons):
    pb = Pushbutton(Pin(button, Pin.IN, Pin.PULL_UP)
    pb.press_coro(cb, (button_no,))
```

The ``Pushbutton`` class hides the detail, but for each button a coroutine is
created which polls the ``Pin`` object and performs the debouncing. It can also
start user supplied coroutines on button release events, long presses and
double clicks. The code in ``aswitch.py`` achieves this using asyncio.

Scheduling also solves the problem of blocking. If a routine needs to wait for
a physical event to occur before it can continue it is said to be blocked. You
may not want the entire system to be blocked. While this can be solved in
linear code, in threaded code the solution is trivial. The coroutine blocks,
but while it does so it periodically yields execution. Hence the rest of the
system continues to run.

## 7.2 Why cooperative rather than pre-emptive?

The initial reaction of beginners to the idea of cooperative multi-tasking is
often one of disappointment. Surely pre-emptive is better? Why should I have to
explicitly yield control when the Python virtual machine can do it for me?

When it comes to embedded systems the cooperative model has two advantages.
Fistly, it is lightweight. It is possible to have large numbers of coroutines
because unlike descheduled threads, paused coroutines contain little state.
Secondly it avoids some of the subtle problems associated with pre-emptive
scheduling. In practice cooperative multi-tasking is widely used, notably in
user interface applications.

To make a case for the defence a pre-emptive model has one advantage: if
someone writes

```python
for x in range(1000000):
    # do something time consuming
```

it won't lock out other threads. Under cooperative schedulers the loop must
explicitly yield control every so many iterations e.g. by putting the code in
a coro and periodically issuing ``await asyncio.sleep(0)``.

Alas this benefit of pre-emption pales into insignificance compared to the
drawbacks. Some of these are covered in the documentation on writing
[interrupt handlers](http://docs.micropython.org/en/latest/reference/isr_rules.html).
In a pre-emptive model every thread can interrupt every other thread, changing
data which might be used in other threads. It is generally much easier to find
and fix a lockup resulting from a coro which fails to yield than locating the
sometimes deeply subtle and rarely occurring bugs which can occur in
pre-emptive code.

To put this in simple terms, if you write a MicroPython coroutine, you can be
sure that variables won't suddenly be changed by another coro: your coro has
complete control until it issues ``await asyncio.sleep(0)``.

Bear in mind that interrupt handlers are pre-emptive.

## 7.3 Communication

In non-trivial applications coroutines need to communicate. Conventional Python
techniques can be employed. These include the use of global variables or
declaring coros as object methods: these can then share instance variables.
Alternatively a mutable object may be passed as a coro argument.

Pre-emptive systems mandate specialist classes to achieve "thread safe"
communications; in a cooperative system these are seldom required.

## 7.4 Polling

Some hardware devices such as the accelerometer don't support interrupts, and
therefore must be polled. One option suitable for slow devices is to write a
coro which polls the device periodically. A faster and more elegant way is to
delegate this activity to the scheduler. The thread then suspends execution of
that thread pending the result of a user supplied callback function, which is
run by the scheduler. From the thread's point of view it blocks pending an
event - with an optional timeout available. See paragraph "Using IORead to poll
hardware" above.