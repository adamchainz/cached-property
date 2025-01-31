# cached-property

[![Github Actions status](https://github.com/pydanny/cached-property/workflows/Python%20package/badge.svg)](https://github.com/pydanny/cached-property/actions)
[![PyPI](https://img.shields.io/pypi/v/cached-property.svg)](https://pypi.python.org/pypi/cached-property)
[![Code style: ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)

A decorator for caching properties in classes.

## Why?

* Makes caching of time or computational expensive properties quick and easy.
* Because I got tired of copy/pasting this code from non-web project to non-web project.
* I needed something really simple that worked in Python 2 and 3.

## How to use it

Let's define a class with an expensive property. Every time you stay there the
price goes up by $50!

```python
class Monopoly:

    def __init__(self):
        self.boardwalk_price = 500

    @property
    def boardwalk(self):
        # In reality, this might represent a database call or time
        # intensive task like calling a third-party API.
        self.boardwalk_price += 50
        return self.boardwalk_price
```

Now run it:

```python
>>> monopoly = Monopoly()
>>> monopoly.boardwalk
550
>>> monopoly.boardwalk
600
```

Let's convert the boardwalk property into a `cached_property`.

```python
from cached_property import cached_property

class Monopoly(object):

    def __init__(self):
        self.boardwalk_price = 500

    @cached_property
    def boardwalk(self):
        # Again, this is a silly example. Don't worry about it, this is
        #   just an example for clarity.
        self.boardwalk_price += 50
        return self.boardwalk_price
```

Now when we run it the price stays at $550.

```python
>>> monopoly = Monopoly()
>>> monopoly.boardwalk
550
>>> monopoly.boardwalk
550
>>> monopoly.boardwalk
550
```

Why doesn't the value of `monopoly.boardwalk` change? Because it's a **cached property**!

## Invalidating the Cache

Results of cached functions can be invalidated by outside forces. Let's demonstrate how to force the cache to invalidate:

```python
>>> monopoly = Monopoly()
>>> monopoly.boardwalk
550
>>> monopoly.boardwalk
550
>>> # invalidate the cache
>>> del monopoly.__dict__['boardwalk']
>>> # request the boardwalk property again
>>> monopoly.boardwalk
600
>>> monopoly.boardwalk
600
```

## Working with Threads

What if a whole bunch of people want to stay at Boardwalk all at once? This means using threads, which
unfortunately causes problems with the standard `cached_property`. In this case, switch to using the
`threaded_cached_property`:

```python
from cached_property import threaded_cached_property

class Monopoly:

    def __init__(self):
        self.boardwalk_price = 500

    @threaded_cached_property
    def boardwalk(self):
        """threaded_cached_property is really nice for when no one waits
            for other people to finish their turn and rudely start rolling
            dice and moving their pieces."""

        sleep(1)
        self.boardwalk_price += 50
        return self.boardwalk_price
```

Now use it:

```python
>>> from threading import Thread
>>> from monopoly import Monopoly
>>> monopoly = Monopoly()
>>> threads = []
>>> for x in range(10):
>>>     thread = Thread(target=lambda: monopoly.boardwalk)
>>>     thread.start()
>>>     threads.append(thread)

>>> for thread in threads:
>>>     thread.join()

>>> self.assertEqual(m.boardwalk, 550)
```

## Working with async/await

The cached property can be async, in which case you have to use await
as usual to get the value. Because of the caching, the value is only
computed once and then cached:

```python
from cached_property import cached_property

class Monopoly:

    def __init__(self):
        self.boardwalk_price = 500

    @cached_property
    async def boardwalk(self):
        self.boardwalk_price += 50
        return self.boardwalk_price
```

Now use it:

```python
>>> async def print_boardwalk():
...     monopoly = Monopoly()
...     print(await monopoly.boardwalk)
...     print(await monopoly.boardwalk)
...     print(await monopoly.boardwalk)
>>> import asyncio
>>> asyncio.get_event_loop().run_until_complete(print_boardwalk())
550
550
550
```

Note that this does not work with threading either, most asyncio
objects are not thread-safe. And if you run separate event loops in
each thread, the cached version will most likely have the wrong event
loop. To summarize, either use cooperative multitasking (event loop)
or threading, but not both at the same time.

## Timing out the cache

Sometimes you want the price of things to reset after a time. Use the `ttl`
versions of `cached_property` and `threaded_cached_property`.

```python
import random
from cached_property import cached_property_with_ttl

class Monopoly(object):

    @cached_property_with_ttl(ttl=5) # cache invalidates after 5 seconds
    def dice(self):
        # I dare the reader to implement a game using this method of 'rolling dice'.
        return random.randint(2,12)
```

Now use it:

```python
>>> monopoly = Monopoly()
>>> monopoly.dice
10
>>> monopoly.dice
10
>>> from time import sleep
>>> sleep(6) # Sleeps long enough to expire the cache
>>> monopoly.dice
3
>>> monopoly.dice
3
```

**Note:** The `ttl` tools do not reliably allow the clearing of the cache. This
is why they are broken out into seperate tools. See https://github.com/pydanny/cached-property/issues/16.

## Credits

* Pip, Django, Werkzeug, Bottle, Pyramid, and Zope for having their own implementations. This package originally used an implementation that matched the Bottle version.
* Reinout Van Rees for pointing out the `cached_property` decorator to me.
* My awesome wife [@audreyfeldroy](https://github.com/audreyfeldroy) who created [`cookiecutter`](https://github.com/cookiecutter/cookiecutter), which meant rolling this out took me just 15 minutes.
* @tinche for pointing out the threading issue and providing a solution.
* @bcho for providing the time-to-expire feature

