# Multi Threading

### Threading - without daemon

```python
import logging
import threading
import time

def thread_function(name):
	logging.info("Thread %s	:	starting", name)
	time.sleep(2)
	logging.info("Thread %s	:	finishing", name)

if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
	logging.info("Main		:	Before creating thread")
	x = threading.Thread(target=thread_function, args=(1,))
	logging.info("Main		:	Before running thread")
	x.start()
	logging.info("Main		:	wait for the thread to finish")
	# x.join()
	logging.info("Main		:	all done")
```

```bash
14:15:36: Main          :       Before creating thread
14:15:36: Main          :       Before running thread
14:15:36: Thread 1      :       starting
14:15:36: Main          :       wait for the thread to finish
14:15:36: Main          :       all done
14:15:38: Thread 1      :       finishing
```

- Here we have two thread one is `main` thread and another we started with `threading.Thread`.
- A **daemon** is a process that runs in the background.
- If a program is running **Threads** that are not **daemons**, then the program will wait for those threads to complete before it terminates. **Threads** that are **daemons**, however, are just killed wherever they are when the program is exiting.
- The pause when before the last line get printed is actually Python waiting for the **non-daemonic** thread to complete. When your Python program ends, part of the shutdown process is to clean up the threading routine.
- `threading._shutdown()` walks through all of the running threads and calls `.join()` on every one that does not have the daemon flag set. So your program waits to exit because the thread itself is waiting in a sleep.

### Threading - with daemon

```python
x = threading.Thread(target=thread_function, args=(1,), daemon=True)
```

Again running the programm gives the output

```bash
14:33:41: Main          :       Before creating thread
14:33:41: Main          :       Before running thread
14:33:41: Thread 1      :       starting
14:33:41: Main          :       wait for the thread to finish
14:33:41: Main          :       all done
```

The difference here is that the final line of the output is missing. `thread_function()` did not get a chance to complete. It was a daemon thread, so when `__main__` reached the end of its code and the program wanted to finish, the daemon was killed.

To tell one thread to wait for another thread to finish, you call `.join()`. If you uncomment that line `x.join()`, the main thread will pause and wait for the thread x to complete running.

### Multiple Threads - Manual Handeling

```python
import logging
import threading
import time


def thread_function(name):
	logging.info("Thread %s: starting", name)
	time.sleep(4)
	logging.info("Thread %s: finishing", name)


if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
	
	threads = list()
	for index in range(3):
		logging.info("Main	:	create and start thread %d.", index)
		x = threading.Thread(target=thread_function, args=(index,))
		threads.append(x)
		x.start()

	for index, thread in enumerate(threads):
		logging.info("Main	:	before joining thread %d", index)
		thread.join()
		logging.info("Main	:	thread %d done", index)
```

```bash
16:17:04: Main  :       create and start thread 0.
16:17:04: Thread 0: starting
16:17:04: Main  :       create and start thread 1.
16:17:04: Thread 1: starting
16:17:04: Main  :       create and start thread 2.
16:17:04: Thread 2: starting
16:17:04: Main  :       before joining thread 0
16:17:08: Thread 0: finishing
16:17:08: Main  :       thread 0 done
16:17:08: Thread 2: finishing
16:17:08: Thread 1: finishing
16:17:08: Main  :       before joining thread 1
16:17:08: Main  :       thread 1 done
16:17:08: Main  :       before joining thread 2
16:17:08: Main  :       thread 2 done
```

The order in which threads are run is determined by the operating system and can be quite hard to predict. 

### Multiple Threads - ThreadPoolExecutor

The easiest way to create it is as a **context manager**, using the `with` statement to manage the creation and destruction of the pool.

```python
import logging
import time
import concurrent.futures


def thread_function(name):
	logging.info("Thread %s: starting", name)
	time.sleep(4)
	logging.info("Thread %s: finishing", name)


if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")

	logging.info("Before Thread pool")
	with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
		executor.map(thread_function, range(3))
	logging.info("After Thread pool")
```

```bash
19:39:32: Before Thread pool
19:39:32: Thread 0: starting
19:39:32: Thread 1: starting
19:39:32: Thread 2: starting
19:39:36: Thread 1: finishing
19:39:36: Thread 0: finishing
19:39:36: Thread 2: finishing
19:39:36: After Thread pool
```

Again, notice how Thread 1 finished before Thread 0. The scheduling of threads is done by the operating system and does not follow a plan that’s easy to figure out.

### Race Conditions 

Race conditions can occur when two or more threads access a shared piece of data or resource. In this example, you’re going to create a large race condition that happens every time, but be aware that most race conditions are not this obvious. Frequently, they only occur rarely, and they can produce confusing results. As you can imagine, this makes them quite difficult to debug.

```python
import time
import logging
import concurrent.futures


class FakeDatabase:
    def __init__(self):
        self.value = 0

    def update(self, name):
        logging.info("Thread %s: starting update", name)
        local_copy = self.value
        local_copy += 1
        time.sleep(0.1)
        self.value = local_copy
        logging.info("Thread %s: finishing update", name)


if __name__ == "__main__":
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO,
                        datefmt="%H:%M:%S")
        
    database = FakeDatabase()
    logging.info("Testing update. Starting value is %d.", database.value)
    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        for index in range(2):
            # each of the threads in the pool will call database.update(index)
            executor.submit(database.update, index)
    logging.info("Testing update. Ending value is %d.", database.value)
```

```bash
20:30:25: Testing update. Starting value is 0.
20:30:25: Thread 0: starting update
20:30:25: Thread 1: starting update
20:30:25: Thread 0: finishing update
20:30:25: Thread 1: finishing update
20:30:25: Testing update. Ending value is 1.
```

The program creates a `ThreadPoolExecutor` with two threads and then calls `.submit()` on each of them, telling them to run `database.update()`. `.submit()` has a signature that allows both positional and named arguments to be passed to the function running in the thread:

```python
.submit(function, *args, **kwargs)
```

When the thread starts running `.update()`, it has its own version of all of the data local to the function. In the case of `.update()`, this is `local_copy`. This is definitely a good thing. Otherwise, two threads running the same function would always confuse each other. It means that all variables that are scoped (or local) to a function are **thread-safe**.

In case of two thread they will each have their own version of `local_copy` and will each point to the same database. It is this shared database object that is going to cause the problems. When Thread 1 calls `time.sleep()`, it allows the other thread to start running. The two threads have interleaving access to a single shared object, overwriting each other’s results. Similar race conditions can arise when one thread frees memory or closes a file handle before the other thread is finished accessing it.

### Basic Synchronization Using Lock

A Lock is an object that acts like a hall pass. Only one thread at a time can have the Lock. Any other thread that wants the Lock must wait until the owner of the Lock gives it up.

The basic functions to do this are `.acquire()` and `.release()`. A thread will call `my_lock.acquire()` to get the lock. If the lock is already held, the calling thread will wait until it is released. There’s an important point here. If one thread gets the lock but never gives it back, your program will be stuck. 

Fortunately, Python’s Lock will also operate as a context manager, so you can use it in a `with` statement, and it gets released automatically when the `with` block exits for any reason.

```python
import time
import logging
import concurrent.futures
import threading
from unicodedata import name


class FakeDatabase:
	def __init__(self):
		self.value = 0
		self._lock = threading.Lock()

	def update(self, name):
		logging.info("Thread %s: starting update", name)
		with self._lock:
			local_copy = self.value
			local_copy += 1
			time.sleep(0.1)
			self.value = local_copy
		logging.info("Thread %s: finishing update", name)


if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO,
						datefmt="%H:%M:%S")
		
	database = FakeDatabase()
	logging.info("Testing update. Starting value is %d.", database.value)
	with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
		for index in range(2):
			# each of the threads in the pool will call database.update(index)
			executor.submit(database.update, index)
	logging.info("Testing update. Ending value is %d.", database.value)
```

```bash
23:50:57: Testing update. Starting value is 0.
23:50:57: Thread 0: starting update
23:50:57: Thread 1: starting update
23:50:57: Thread 0: finishing update
23:50:57: Thread 1: finishing update
23:50:57: Testing update. Ending value is 2.
```

Change here is to add a member called `._lock`, which is a `threading.Lock()` object. This `._lock` is initialized in the unlocked state and locked and released by the with statement. This `._lock` is initialized in the unlocked state and locked and released by the with statement.

It’s worth noting here that the thread running this function will hold on to that Lock until it is completely finished updating the database. In this case, that means it will hold the Lock while it copies, updates, sleeps, and then writes the value back to the database.

### Deadlock

```python
>>> import threading
>>> l = threading.Lock()
>>> l.acquire()
True
>>> l.acquire()
^CTraceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyboardInterrupt
>>> 
>>> l.release()
>>> l.release()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
RuntimeError: release unlocked lock
>>> 
```

When the program calls `l.acquire()` the second time, it hangs waiting for the Lock to be released. In this example, you can fix the deadlock by removing the second call, but deadlocks usually happen from one of two subtle things:

1. An implementation bug where a Lock is not released properly
2. A design issue where a utility function needs to be called by functions that might or might not already have the Lock

Python threading has a second object, called `RLock`, that is designed for just this situation. It allows a thread to .`acquire()` an `RLock` multiple times before it calls `.release()`. That thread is still required to call `.release()` the same number of times it called `.acquire()`, but it should be doing that anyway.

```python
>>> import threading
>>> l = threading.RLock()
>>> l.acquire()
True
>>> l.acquire()
True
>>> l.acquire()
True
>>> l.release()
>>> l.release()
>>> l.release()
>>> l.release()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
RuntimeError: cannot release un-acquired lock
>>>
```

`Lock` and `RLock` are two of the basic tools used in threaded programming to prevent race conditions.

### Threading Objects

#### Semaphore

The first Python threading object to look at is `threading.Semaphore`. A **Semaphore** is a counter with a few special properties (counter = 3 means it will allow 3 thread to access the resource at a time). The first one is that the counting is atomic. This means that there is a guarantee that the operating system will not swap out the thread in the middle of incrementing or decrementing the counter.

The internal counter is incremented when you call `.release()` and decremented when you call `.acquire()`.

The next special property is that if a thread calls `.acquire()` when the counter is zero, that thread will block until a different thread calls `.release()` and increments the counter to one.

Semaphores are frequently used to protect a resource that has a limited capacity. An example would be if you have a pool of connections and want to limit the size of that pool to a specific number.

#### Timer

A `threading.Timer` is a way to schedule a function to be called after a certain amount of time has passed. You create a **Timer** by passing in a number of seconds to wait and a function to call:

```python
t = threading.Timer(30.0, my_function)
```

You start the **Timer** by calling `.start()`. The function will be called on a new thread at some point after the specified time, but be aware that there is no promise that it will be called exactly at the time you want.

If you want to stop a Timer that you’ve already started, you can cancel it by calling `.cancel()`. Calling `.cancel()` after the Timer has triggered does nothing and does not produce an exception.

A Timer can be used to prompt a user for action after a specific amount of time. If the user does the action before the Timer expires, `.cancel()` can be called.

#### Barrier

A `threading.Barrier` can be used to keep a fixed number of threads in sync. When creating a **Barrier**, the caller must specify how many threads will be synchronizing on it. Each thread calls `.wait()` on the Barrier. They all will remain blocked until the specified number of threads are waiting, and then all of them will be released at the same time.

Remember that threads are scheduled by the operating system so, even though all of the threads are released simultaneously, they will be scheduled to run one at a time.

One use for a Barrier is to allow a pool of threads to initialize themselves. Having the threads wait on a Barrier after they are initialized will ensure that none of the threads start running before all of the threads are finished with their initialization.

### Producer-Consumer Threading

#### Producer-Consumer : Lock

```python
import concurrent.futures
import random
import threading
import logging
from unittest.mock import sentinel

SENTINEL = object()

class Pipeline:
	"""
	Class to allow a single element pipeline between producer and consumer.
	"""
	def __init__(self):
		self.message = 0
		self.producer_lock = threading.Lock()
		self.consumer_lock = threading.Lock()
		self.consumer_lock.acquire()

	def get_message(self, name):
		logging.debug("%s:about to acquire getlock", name)
		self.consumer_lock.acquire()
		logging.debug("%s:have getlock", name)
		message = self.message
		logging.debug("%s:about to release setlock", name)
		self.producer_lock.release()
		logging.debug("%s:setlock released", name)
		return message

	def set_message(self, message, name):
		logging.debug("%s:about to acquire setlock", name)
		self.producer_lock.acquire()
		logging.debug("%s:have setlock", name)
		self.message = message
		logging.debug("%s:about to release getlock", name)
		self.consumer_lock.release()
		logging.debug("%s:getlock released", name)

def producer(pipeline):
	"""
	pretend we are getting a message from network
	"""
	for _ in range(10):
		message = random.randint(1,101)
		logging.info("Producer got message: %s", message)	
		pipeline.set_message(message, "Producer")

	pipeline.set_message(SENTINEL, "Producer")

def consumer(pipeline):
	"""
	Pretend we are saving number in DB
	"""
	message = 0
	while message is not SENTINEL:
		message = pipeline.get_message("Consumer")
		if message is not SENTINEL:
			logging.info("Consumer storing message: %s", message)


if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
	# logging.getLogger().setLevel(logging.DEBUG)

	pipeline = Pipeline()
	with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
		executor.submit(producer, pipeline)
		executor.submit(consumer, pipeline)
```
```bash
02:50:40: Producer got message: 10
02:50:40: Producer got message: 5
02:50:40: Consumer storing message: 10
02:50:40: Producer got message: 75
02:50:40: Consumer storing message: 5
02:50:40: Consumer storing message: 75
02:50:40: Producer got message: 28
02:50:40: Producer got message: 18
02:50:40: Consumer storing message: 28
02:50:40: Producer got message: 38
02:50:40: Consumer storing message: 18
02:50:40: Consumer storing message: 38
02:50:40: Producer got message: 83
02:50:40: Producer got message: 54
02:50:40: Consumer storing message: 83
02:50:40: Producer got message: 4
02:50:40: Producer got message: 19
02:50:40: Consumer storing message: 54
02:50:40: Consumer storing message: 4
02:50:40: Consumer storing message: 19
```

- message stores the message to pass.
- `producer_lock` is a threading.Lock object that restricts access to the message by the producer thread.
- `consumer_lock` is also a threading.Lock that restricts access to the message by the consumer thread.

As soon as the consumer calls `.producer_lock.release()`, it can be swapped out, and the producer can start running. That could happen before `.release()` returns! This means that there is a slight possibility that when the function returns self.message, that could actually be the next message generated, so you would lose the first message. This is another example of a race condition.

The operating system can swap threads at any time, but it generally lets each thread have a reasonable amount of time to run before swapping it out. That’s why the producer usually runs until it blocks in the second call to `.set_message()`.

While it works for this limited test, it is not a great solution to the producer-consumer problem in general because it only allows a single value in the pipeline at a time. When the producer gets a burst of messages, it will have nowhere to put them.

Let’s move on to a better way to solve this problem, using a Queue.

#### Producer-Consumer : Queue

Let’s start with the **Event**. The `threading.Event` object allows one thread to signal an event while many other threads can be waiting for that event to happen. The key usage in this code is that the threads that are waiting for the event do not necessarily need to stop what they are doing, they can just check the status of the Event every once in a while.

```python
from queue import Queue
import random
import threading
import concurrent.futures
import logging
import time

def producer(queue, event):
	while not event.is_set():
		message = random.randint(1,101)
		logging.info("Producer got message: %s (size=%d)", message, queue.qsize())
		queue.put(message)
	logging.info("Producer receive event: Exiting")

def consumer(queue, event):
	while not event.is_set() or not queue.empty():
		message = queue.get()
		logging.info("Consumer storing message: %s (size=%d)", message, queue.qsize())
	logging.info("Consumer received event: Exiting")


if __name__ == "__main__":
	format = "%(asctime)s: %(message)s"
	logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
	
	pipeline = Queue(maxsize=10)
	event = threading.Event()
	with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
		executor.submit(producer, pipeline, event)
		executor.submit(consumer, pipeline, event)

		time.sleep(0.01)
		logging.info("Main: about to set event")
		event.set()
```

```bash
16:03:11: Producer got message: 33 (size=0)
16:03:11: Producer got message: 72 (size=1)
16:03:11: Producer got message: 1 (size=2)
16:03:11: Producer got message: 69 (size=3)
16:03:11: Producer got message: 11 (size=4)
16:03:11: Producer got message: 75 (size=5)
16:03:11: Producer got message: 27 (size=6)
16:03:11: Producer got message: 65 (size=7)
16:03:11: Producer got message: 40 (size=8)
16:03:11: Producer got message: 15 (size=9)
16:03:11: Producer got message: 79 (size=10)
16:03:11: Consumer storing message: 33 (size=9)
16:03:11: Consumer storing message: 72 (size=8)
16:03:11: Consumer storing message: 1 (size=7)
16:03:11: Consumer storing message: 69 (size=6)
16:03:11: Consumer storing message: 11 (size=5)
16:03:11: Consumer storing message: 75 (size=4)
16:03:11: Consumer storing message: 27 (size=3)
...
16:03:11: Consumer storing message: 11 (size=0)
16:03:11: Main: about to set event
16:03:11: Producer got message: 48 (size=4)
16:03:11: Producer receive event: Exiting
16:03:11: Consumer storing message: 48 (size=0)
16:03:11: Consumer received event: Exiting
```

Let’s start with the `Event`. The `threading.Event` object allows one thread to signal an event while many other threads can be waiting for that event to happen. The key usage in this code is that the threads that are waiting for the event do not necessarily need to stop what they are doing, they can just check the status of the Event every once in a while.

In this example, the main thread will simply sleep for a while and then `.set()` event. This set of event tells producer to stop producing and consumer to stop worring about new message comming into queue and just finish the remaing ones.

If you give a positive number for `maxsize`, it will limit the queue to that number of elements, causing `.put()` to block until there are fewer than `maxsize` elements. If you don’t specify `maxsize`, then the queue will grow to the limits of your computer’s memory.

The core devs who wrote the standard library knew that a `Queue` is frequently used in multi-threading environments and incorporated all of that locking code inside the Queue itself. **Queue is thread-safe**.
