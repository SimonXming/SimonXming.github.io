---
layout: post
title: python 源码阅读 - generator and coroutine
category: 语言
tags: python asyncio
keywords: python
description:
---

# Python generator and coroutine

```python
# ./lib/python3.6/asyncio/tasks.py
"""
## PEP 342
## 1. Calling send(None) is exactly equivalent to calling a generator's
## next() method.
## 2. As with the next() method, the send() method returns the
## next value yielded by the generator-iterator, or raises
## StopIteration if the generator exits normally, or has already exited.
"""

class Task(futures.Future):
    """Coroutines will be wrapped in Tasks."""
    ...

    def _step(self, exc=None):
        ...
        coro = self._coro
        self._fut_waiter = None

        self.__class__._current_tasks[self._loop] = self
        # Call either coro.throw(exc) or coro.send(None).
        try:
            if exc is None:
                # We use the `send` method directly, because coroutines
                # don't have `__iter__` and `__next__` methods.
                ## PEP 342
                ## 1. Calling send(None) is exactly equivalent to calling a generator's
                ## next() method.
                ## 2. As with the next() method, the send() method returns the
                ## next value yielded by the generator-iterator, or raises
                ## StopIteration if the generator exits normally, or has already exited.
                result = coro.send(None)
            else:
                result = coro.throw(exc)
        except StopIteration as exc:
            if self._must_cancel:
                # Task is cancelled right before coro stops.
                self._must_cancel = False
                self.set_exception(futures.CancelledError())
            else:
                self.set_result(exc.value)
        except futures.CancelledError:
            # I.e., Future.cancel(self).
            super().cancel()
        except Exception as exc:
            self.set_exception(exc)
        except BaseException as exc:
            self.set_exception(exc)
            raise
        else:
            blocking = getattr(result, '_asyncio_future_blocking', None)
            if blocking is not None:
                # Yielded Future must come from Future.__iter__().
                if result._loop is not self._loop:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'Task {!r} got Future {!r} attached to a '
                            'different loop'.format(self, result)))
                elif blocking:
                    if result is self:
                        self._loop.call_soon(
                            self._step,
                            RuntimeError(
                                'Task cannot await on itself: {!r}'.format(
                                    self)))
                    else:
                        result._asyncio_future_blocking = False
                        result.add_done_callback(self._wakeup)
                        self._fut_waiter = result
                        if self._must_cancel:
                            if self._fut_waiter.cancel():
                                self._must_cancel = False
                else:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'yield was used instead of yield from '
                            'in task {!r} with {!r}'.format(self, result)))
            elif result is None:
                # Bare yield relinquishes control for one event loop iteration.
                self._loop.call_soon(self._step)
            elif inspect.isgenerator(result):
                # Yielding a generator is just wrong.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'yield was used instead of yield from for '
                        'generator in task {!r} with {}'.format(
                            self, result)))
            else:
                # Yielding something else is an error.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'Task got bad yield: {!r}'.format(result)))
        finally:
            self.__class__._current_tasks.pop(self._loop)
            self = None  # Needed to break cycles when an exception occurs.
     ...

```



### Coroutines

AKA: abstract of subroutines (can be entered, exited, and resumed at many different points)

Coroutines is a more generalized form of subroutines. Subroutines are entered at one point and exited at another point. Coroutines can be entered, exited, and resumed at many different points. They can be implemented with the [`async def`](https://docs.python.org/3/reference/compound_stmts.html#async-def) statement. See also [**PEP 492**](https://www.python.org/dev/peps/pep-0492).

### coroutine function

AKA: `async def` function

A function which returns a [coroutine](https://docs.python.org/3/glossary.html#term-coroutine) object. A coroutine function may be defined with the [`async def`](https://docs.python.org/3/reference/compound_stmts.html#async-def) statement, and may contain [`await`](https://docs.python.org/3/reference/expressions.html#await), [`asyncfor`](https://docs.python.org/3/reference/compound_stmts.html#async-for), and [`async with`](https://docs.python.org/3/reference/compound_stmts.html#async-with) keywords. These were introduced by [**PEP 492**](https://www.python.org/dev/peps/pep-0492).

### generator iterator

AKA: generator function call return generator iterator

An object created by a [generator](https://docs.python.org/3/glossary.html#term-generator) function.

Each [`yield`](https://docs.python.org/3/reference/simple_stmts.html#yield) temporarily suspends processing, remembering the location execution state (including local variables and pending try-statements). When the *generator iterator* resumes, it picks-up where it left-off (in contrast to functions which start fresh on every invocation).

### generator

AKA: function contains `yield` (which return `generator iterator`)

A function which returns a [generator iterator](https://docs.python.org/3/glossary.html#term-generator-iterator). It looks like a normal function except that it contains [`yield`](https://docs.python.org/3/reference/simple_stmts.html#yield) expressions for producing a series of values usable in a for-loop or that can be retrieved one at a time with the [`next()`](https://docs.python.org/3/library/functions.html#next) function.

Usually refers to a generator function, but may refer to a *generator iterator* in some contexts. In cases where the intended meaning isn’t clear, using the full terms avoids ambiguity.

```
# example
import inspect

def example():
    for i in range(10):
        yield i

inspect.isgeneratorfunction(example) # True
inspect.isgenerator(example())       # True
inspect.isgenerator(example)         # False
```