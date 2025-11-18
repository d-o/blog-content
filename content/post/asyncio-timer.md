+++
date = '2025-11-21T09:29:26+11:00'
draft = false
title = 'C++ Asio Resource Managed Timer'
author = 'Dean Sellers'
tags = [
    "C++",
    "asio",
    "async",
    "embedded",
    "lambda",
    "synchronisation"
]
+++

# A resource managed class for C++ Asio timers

A common pattern in an embedded system is a repeating timer, *never* for polling anything as that is officially a *bad thing*. Often though we want to run an event loop, update some state, blink a LED ... all sorts of things. If you are using the [asio C++ library](https://think-async.com/Asio/), it provides a timer implementation, the [steady_timer](https://think-async.com/Asio/asio-1.36.0/doc/asio/reference/steady_timer.html), which can be setup to run a callback when it expires.

For my use case I wanted to start a timer that would reschedule itself, provide it with a function to call on expiry and run until there is no reference to it anymore. Once the timer object leaves scope, or no one holds a reference to it, it and the context that is passed to it is cleaned up.

For extra credits I added optional callbacks to run at start and on finish. Later in the use for it I needed to preempt it, or reschedule it to run *now*, then continue on. I wrote a wrapper around `asio::steady_timer` to do this for me.

## The code

In my github there is some source for a [repeating_timer](https://github.com/d-o/repeating_timer) class that does just this. The [README](https://github.com/d-o/repeating_timer) gives examples of use and some information about it. In the *features and benefits* is interesting `Thread‑safe | std::mutex protects the context while the user callback runs`, it is thread safe, awesome. Lets try it out ...

### Going to the races

We will write a simple test that creates a bunch of timers that run in a bunch of threads and just outputs to `cout`, the code is below.

```cpp
/*
* Copyright (c) 2025 Dean Sellers (dean@sellers.id.au)
* SPDX-License-Identifier: MIT
*/

#include "repeatable_timer.hpp"
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

/* Context for any timer */
struct timer_context {
    const int index;
    int counter;
};

/* The callback function, all timers can use this */
void timer_cb(timer_context& context) {
    /* A shared resource is being accessed here - ie: cout */
    std::cout << "\tContext " <<  this->index << ":#" << ++this->counter << '\n';
}

int main() {

    constexpr static int NUM_TIMERS = 5;

    /* Create an empty vector of threads */
    std::vector<std::thread> threads;

    /* asio context */
    asio::io_context io;

    /* Now create a timer running in each thread in a new scope */
    {
        /* We need to a reference to each timer somewhere */
        std::vector<std::shared_ptr<RepeatingTimer<timer_context>>> timers;
        /* Create the timers */
        for(int i=1; i<=NUM_TIMERS; i++)
        {
            /* Create a timer that prints a counter every 1 millisecond. */
            auto timer = RepeatingTimer<timer_context>::create(
                io,
                &timer_cb,
                std::chrono::milliseconds(1),
                timer_context{index: i, counter: 0}  /* Timer context */
            );
            /* Save the timer in our vector in the outer scope */
            timers.push_back(timer);
        }
        /* Create a new thread for each timer
        This doesn't mean that each timer WILL run in a separate thread
        I am just trying to give asio the opportunity to do that */
        std::cout << "Create threads.\n";
        for(size_t i=1; i<=NUM_TIMERS; i++) {
            threads.push_back(std::thread ([&io]{ io.run();}));
        }
        /* Let them all tick for a bit */
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    /* Once our timers are out of scope, they are stopped */
    for(auto& t : threads) {
        t.join();
    }
    std::cout << "Threads finished." << std::endl;
}
```
Built and run the output looks like below.

```console
deano@deano-hm:~/Documents/github/asynctimer/test/build$ ./multi_timer_test
Create threads.
        Context 1:#1
        Context 3:#1
        Context 2:#1
        Context 4:#1
        Context 5:#1
.
.
        Context 1:#10
        Context 2:#10
        Context 4:#10
        Context 5:#10
        Context 3:#10
.
.
        Context 1:#26
        Context 2:#26
        Context 5:#26
        Context 4:#26
        Context 3:#26
.
.
        Context 1:#100
        Context 2:#100
        Context 3:#100
        Context 4:#100
        Context 5:#100
Timers finished.
deano@deano-hm:~/Documents/github/asynctimer/test/build$
```
Looks like it is doing the right things, 100 runs of the callback in 5 different contexts! Cool. One thing to note is that there is no synchronisation in the order the threads are run. How do we know that it really is thread safe? What does that mean, and what would happen if it wasn't?

I'll create a `race condition` and see. if you change line 105 `inline static std::mutex mtx_;` to `/*inline static */ std::mutex mtx_;` in `repeatable_timer.hpp` you move the synchronisation from shared by all instances of the timer class to being unique to each one.

The output now looks like this.

```
deano@deano-hm:~/Documents/github/asynctimer/test/build$ ./multi_timer_test
Create threads.
        Context 1:#1
        Context 2:#1
        Context 3:#1
        Context 4:#1
        Context 5:#1
.
.
        Context 1:#4
        Context 3:#4
        Context         Context 42:#    Context :#445

:#4
        Context 1:#5
        Context 2:#5
        Context         Context 5:#3    Context 4:#5:#5
5

.
.
        Context 1:#96
        Context 2:#96
        Context 4:#     Context         Context 5:#96
3:#96
96
        Context 1:#97
        Context 2:#97
        Context         Context 34:#97
:#97
        Context 5:#97
        Context 1:#98
        Context 2:#98
.
.
        Context 1:#100
        Context 2:#100
        Context 3:#100
        Context 4:#100
        Context 5:#100
Timers finished.
deano@deano-hm:~/Documents/github/asynctimer/test/build$
```

Things start off fine, but pretty soon weird things start to happen. You can see the OS task switching happening here, even a simple callback like the one we have gets suspended and others are run outputting to the same stream and corrupting the output. The reality is the callback isn't really that *simple*, it is IO bound to `cout` and  anything IO is notoriously slow compared to processor bound tasks. So it is no surprise that the execution is preempted, the OS is doing it's job and swapping to tasks that may have something to do rather than sit and wait in a task that is waiting on IO.

Even if our callback is just incrementing a variable `counter++` there is no guarantee that it won't be interrupted. There are `atomic` types that do offer atomic operations that are thread safe. It is also important to note that for this timer the lock is held for the duration of the callback. This means that you block all of the other timers while you do your thing here. That is fine for the use cases I have for it, if you need more control over the synchronisation (or no synchronisation, as a mutex isn't free) then there is an option to do that with this class, read on.

### Back from the races but my callback is a method

Resource sharing aside, what about calling a method of a class from the timer? Of course you can do this, it just takes another leap. Remember a method is a function *bound* to an instance of a class/struct. By *bound* I mean that the function is called with an implicit pointer `this` which refers to the instance the call is made on, which is (kind of) what makes a function inside a class a *method*. Let's refactor the above example to use a context that includes the callback.

```cpp
/* Context for any timer */
struct timer_context {

    /* Instance data */
    const int index;
    int counter;

    timer_context(int i, int c = 0) : index(i), counter(c) {}

    void on_timer() {
        /* A shared resource is being accessed here - ie: cout */
        std::cout << "\tContext " <<  this->index << ":#" << ++this->counter << '\n';

    }
};
    .
    .
    .   /* Create the timers */
        for(int i=1; i<=NUM_TIMERS; i++)
        {
            /* Create a timer that prints a counter every 1 milli. */
            auto timer = RepeatingTimer<timer_context>::create(
                io,
                [](timer_context& c) { c.on_timer(); },
                std::chrono::milliseconds(1),
                std::make_shared<timer_context>(i)  /* Timer context */
            );
            /* Save the timer in our vector in the outer scope */
            timers.push_back(timer);
        }
    .
    .
    .
std::cout << "Timers finished." << std::endl;
}
```

Pretty simple really, although the simplicity hides a really useful C++ feature, the *lambda* function. Rather than pass a function pointer as a callback, like we did earlier, the construct `[](timer_context& c) { c.on_timer(); }` is creating and passing an anonymous function as the callback. There are other ways to achieve this method binding, like using `std::bind` from `<functional>` but that is way more verbose, less clear and as you'll see less flexible than the above. An advantage of using a lambda is that it creates a `closure` in which we can bind some external state to the function if we need to, for instance;

```cpp
int main() {
    .
    .
    .
    /* An application specific variable */
    std::atomic<size_t> my_counter(0);
    .
    .
    .
                [&my_counter](timer_context& c) { my_counter++; c.on_timer(); },
    .
    .
    .
    std::cout << "Timers finished callback counter - " << my_counter << std::endl;
}
```

```
deano@deano-hm:~/Documents/github/asynctimer/test/build$ cmake --build .
[ 50%] Built target repeating_timer_test
[ 75%] Building CXX object CMakeFiles/multi_timer_test.dir/multi_thread_test.cpp.o
[100%] Linking CXX executable multi_timer_test
[100%] Built target multi_timer_test
deano@deano-hm:~/Documents/github/asynctimer/test/build$ ./multi_timer_test
Create threads.
        Context 1:#1
    .
    .
        Context 4:#100
        Context 5:#100
Timers finished callback counter - 500
deano@deano-hm:~/Documents/github/asynctimer/test/build$
```
So we have counted 500 invocations of the callback, and not affected the original callback logic.

Some of the advantages of using lambda functions like this are demonstrated in that one snippet;

1. *Clarity* - It is really clear what it is going on right where the call is made
2. *Flexibility* - You can add to the callback whatever you like without touching the implementation.
3. *Encapsulation* - The anonymous function is *closed* around it's context. You don't have to be aware of it's environment, unless you need to be.

This begs the question, why have the option of passing a context at all? You can just bind any old variables into the lambda function, and off you go. In this case the *context* shared pointer allows the timer class to do your resource locking, if you don't want the global lock you can pass a *nullptr* as the context and pass your context in the lambda. The timer class checks if the context is valid, that is *not null*, before acquiring the lock.

```cpp
            /* Create a timer that prints a counter every 1 milli. */
            auto timer = RepeatingTimer<timer_context>::create(
                io,
                [lambda_ctx = timer_context(i), &my_counter](timer_context& c) mutable { my_counter++; lambda_ctx.on_timer(); },
                std::chrono::milliseconds(1),
                nullptr  /* Empty context */
            );
```

Notice that as the timer context isn't needed outside the lambda we don't have to keep a copy of it, it's lifetime is restricted to the lifetime of the lambda which is bound to the timer. This isn't a potential resource leak, the language guarantees that the context will be destroyed when it goes out of scope. Also note the `mutable` keyword. This is necessary as the `on_timer` function isn't `const` as it modifies class state. Without `mutable` the context is passed to the lambda as `const timer_context` which then won't compile, more type safety at work.

Outputs

```
deano@deano-hm:~/Documents/github/asynctimer/test/build$ ./multi_timer_test
Create threads.
        Context 1:#1
        Context 2:#1
    .
    .
        Context 1:#6
        Context 2:#6
        Context 5:#6    Context 4:#
6
        Context 3:#6
        Context 1:#7
        Context 2:#7
        Context 3:#7
        Context         Context 5:#47:#7
    .
    .
        Context 3:#100
        Context 4:#100
        Context 5:#100
Timers finished callback counter - 500
deano@deano-hm:~/Documents/github/asynctimer/test/build$
```

Racing again, expected because now the callback is responsible for resource locking, and we aren't doing it.

```cpp
/* Context for any timer */
struct timer_context {
    /* Shared by all instances */
    inline static std::mutex context_lock;
    .
    .
    .
    void on_timer() {
        /* Do some stuff ... */
        {
            /* Critical section */
            /* A shared resource is being accessed here - ie: cout */
            std::lock_guard<std::mutex> lock(context_lock);
            std::cout << "\tContext " <<  this->index << ":#" << ++this->counter << '\n';
        }
        /* Unlocked .. do other stuff*/
    }
```

Adds class specific resource locking, which you may need depending on your use case. We are back to a thread safety!

## Conclusion

That's my timer class, and hopefully some helpful background about synchronisation and using lambda/closures in modern C++. Although lambda functions were introduced in C++11, so they are probably not *modern* anymore.

Another really good refactor for the timer class would be to use more template meta programming to move the check for a valid context and the lock into a compile time option. That would mean that in cases where you want an unmanaged callback you don't have the overhead of a lock and one *if* statement per call. Not a huge optimisation, but probably a useful exercise.
