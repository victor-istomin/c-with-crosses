---
title: "Win32 UCRT surprise at exit: static variables destruction"
date:  2023-07-09T21:42:55Z
summary: "Well, we all know that Singletons are hard to get right, but a small static variable won't hurt. And the code works for me. -- This may lead to several hours of interesting multi-thread code debugging"
tags: ["cpp", "win32", "multi-threading"]
author: "Me"
draft: false
---

# How I went into an issue

> Well, we all know that Singletons are hard to get right, but a small static variable won’t hurt. It works for me.
<cite> — a library developer by the end of the day</cite>

A little quiz: how long does it take for the following code to terminate?

{{< highlight cpp >}}
#include <thread>

volatile int g_sideEffect = 0;
struct Something
{
    Something()             { g_sideEffect++; }
    ~Something()            { g_sideEffect++; }
};

std::thread g_workerThread;
void workerThreadFunction()
{
    std::this_thread::sleep_for(std::chrono::seconds(1));

    static Something s;
}

struct Module
{
    ~Module()
    {
        if (g_workerThread.joinable())
            g_workerThread.join();
    }
};

Module g_service;

int main()
{
    g_workerThread = std::thread(workerThreadFunction);
    return 0;
}
{{< /highlight >}}

<details><summary>A hint:</summary>
If compiled with glibc/stdlib++, it typically takes around 1 second. However, on Windows with MSVC, as of today, it depends on the developer's patience and power supply due to a deadlock.<br><br></details>

Despite appearing artificial, such a use case occurred on a typical MFC application that joins an HTTP listener thread upon exit in the destructor of a module with static storage duration. Usually, it works, but a deadlock occurs if the last-moment HTTP request is received. 

Long story short: a platform-dependent code gathers additional information when the connection is interrupted. It looked like a good idea to encapsulate platform-dependent error handing in a lazily-initialized singleton, so the resulting code mirrors the example code above.

# A bit of debugging

Let’s dive into the example in the Visual Studio debugger. Hit the pause button and load the symbols as necessary: we observe the main thread joining the worker during the destruction of the static module. At first glance, it looks like a worker thread bug (you know, real-world thread functions are a bit more complex than a given example), so let’s find and examine it.

While the `std::thread` conveniently provides us a thread ID within its object internals, the RAW Win32 threads API won’t, so we might see the `WaitForSingleObjectEx` waiting on some handle with no idea about the thread ID.

## Searching for the deadlock counterpart thread

A hardworking developer could examine all the application threads call-stacks one by one, but as our serious application has a lot of threads, including some auxiliary ones, we'll try to speed up the investigation process.

There are few options came to my mind:
* get the thread ID from the handle;
* stare at the Parallel Stacks in the VS debugger until enlightenment occurs.

## Getting a thread ID from the handle

Let's imagine that there is no <code>std::thread&#58;:_Id</code> member variable. Only a RAW Win32 thread handle, `0x0000009C`, for example. 
![handle: `0x0000009C`](images/handle.png "handle: `0x0000009C`")

A handle is a descriptor of the opened resource, just like the `fopen()` return value. Technically, it’s an index in the resources table. It would be nice if we could glance at it with a little effort, and actually, we can do this using the [Handle tool](https://learn.microsoft.com/en-us/sysinternals/downloads/handle/) from the Sysinternals Utilities by Mark Russinovich. 

By the way, I recommend eventually examining the whole suite of Sysinternals Utilities, as there are a lot of utilities to help the user find out what is going on in the system.

Looking at the manual, let's craft a command line: `-a` for _all_, `-v` for _CSV_:
{{< highlight bat>}}c:\tools\handles>handle -a -v > handles.csv
{{< /highlight >}}

Hit <kbd>ENTER</kbd>, wait until your patience runs out, realizing that Handle.exe is trying to inject its code into a frozen process to gather details. To let the tool execute the code inside the debugged process, hit 'Continue' in the debugger. Then a CSV with all the opened handles appears. Quite a lot, but we have a grep or a table processor to search for our handle. 

![ID: `10596`](images/handle-csv.png "ID: `10596`")

Having a process ID, jump to this thread and examine whether there is a bug in our worker thread. We’ll start by investigating its call stack after describing the alternate solution.

## VS debugger: Parallel Stacks

However, a handle insight may be unavailable. For example, when investigating a minidump taken a long time ago far away from the developer’s PC.

In this case, a Parallel Stack feature of VS debugger may group call stacks of all threads to give an insight into the overall state.

![Debug > Windows > Parallel Stacks](images/parallel-stacks.png "Debug > Windows > Parallel Stacks")

By carefully examining all of the application's threads, we can identify two threads waiting in suspicious locations: the main one has acquired a mutex through the `__acrt_lock_and_call` and is currently waiting for another thread termination inside `std::exit -> ... -> _execute_onexit_table`. Additionally, thread 10596 is waiting to acquire a critical section lock inside `std::atexit -> ... -> _register_onexit_function -> __acrt_lock_and_call`. We might be close to the solution.

# The origin of `std::exit` and `std::atexit` calls 

The reason why `std::exit` is called is that it handles a normal C++ program termination, either when the `main()` completed its execution or upon the explicit call to `std::exit`[^automatic]. It performs the necessary cleanup, including the cleanup of static variables. 

Regarding `std::atexit`, its purpose here is to maintain a correct <abbr title="Last In - First Out">LIFO</abbr> destruction order of the static variables, because the order may be runtime-defined and can't be hard-coded:

[^automatic]: although it's hard to call an explicit call to `std::exit` a "normal program termination" in C++, because it does not unwind the stack, thus not destroying objects with automatic storage duration. 

{{< highlight cpp>}}
int foo()
{
    static SomethingA a; // initialized once the 'foo' is called
}

int bar()
{
    static SomethingB b; // initialized once the 'bar' is called
}

int main()
{
    // foo::a and bar::b initialization order is determined by run-time, happy QA-ing!
    if (0 != (time(nullptr) % 1))
        foo();

    bar();
    foo();
}
{{< /highlight >}}

Thus the destructor of an object with a static lifetime is registered by its constructor. I didn't find the statement that the compiler has to register it using `std::atexit`, but this assumption looks reasonable given the `std::exit` documentation:

> Causes normal program termination to occur.                                                                            <br>
> Several cleanup steps are performed: <br>
> 1)The destructors of objects with thread local storage duration that are associated with the current thread, the destructors of objects with static storage duration, and the functions registered with <code>std::atexit</code> are executed concurrently, while maintaining the following guarantees:<br>
> a) The last destructor for thread-local objects is sequenced-before the first destructor for a static object<br>
> b) If the completion of the constructor or dynamic initialization for thread-local or static object A was sequenced-before thread-local or static object B, the completion of the destruction of B is sequenced-before the start of the destruction of A<br>
> c) If the completion of the initialization of a static object A was sequenced-before the call to <code>std::atexit</code> for some function F, the call to F during termination is sequenced-before the start of the destruction of A<br>
> d) If the call to <code>std::atexit</code> for some function F was sequenced-before the completion of initialization of a static object A, the start of the destruction of A is sequenced-before the call to F during termination.<br>
> e) If a call to <code>std::atexit</code> for some function F1 was sequenced-before the call to <code>std::atexit</code> for some function F2, then the call to F2 during termination is sequenced-before the call to F1 <br>
> ...
<cite> -- [cppreference on std::exit](https://en.cppreference.com/w/cpp/utility/program/exit)</cite>

In essence, a static variable with a non-trivial destructor will call `std::atexit` or its internal counterpart to register the destructor. Later, `std::exit` will call registered destructors alongside other deinitialization callbacks using a LIFO order. 

## A glance at various `std::exit` implementations

Since the `std::atexit` [is thread-safe](https://en.cppreference.com/w/cpp/utility/program/atexit), it's synchronized with `std::exit` to avoid race conditions. However, the synchronization may incur a thread lock, which may introduce a possibility of a deadlock.

Now, let's explore different implementations to gain a better understanding of the underlying mechanisms and potential issues involved.

### Win32: Universal CRT

Since Windows 10, the C Runtime is a separate system component. Its source code is a part of Windows SDK and is located in `C:\Program Files (x86)\Windows Kits\10\Source\[SDK-version]\ucrt\startup\onexit.cpp`. Let's dive into the `_execute_onexit_table` and `_register_onexit_function` which, as we already know from callstacks, are responsible for `std::exit` and `std::atexit` callbacks processing:

{{< highlight cpp "hl_lines=8 33 55">}}
// This function executes a table of _onexit()/atexit() functions.  The
// terminators are executed in reverse order, to give the required LIFO
// execution order.  If the table is uninitialized, this function has no
// effect.  After executing the terminators, this function resets the table
// so that it is uninitialized.  Returns 0 on success; -1 on failure.
extern "C" int __cdecl _execute_onexit_table(_onexit_table_t* const table)
{
    return __acrt_lock_and_call(__acrt_select_exit_lock(), [&]
    {
        // ... 
        // This loop calls through caller-provided function pointers.
        // ...
        {
            // ... 
            for (;;)
            {
                // Find the last valid function pointer to call:
                while (--last >= first && *last == encoded_nullptr)
                {
                    // Keep going backwards
                }

                if (last < first)
                {
                    // There are no more valid entries in the list; we are done:
                    break;
                }

                // Store the function pointer and mark it as visited in the list:
                _PVFV const function = __crt_fast_decode_pointer(*last);
                *last = encoded_nullptr;

                function();

                _PVFV* const new_first = __crt_fast_decode_pointer(table->_first);
                _PVFV* const new_last  = __crt_fast_decode_pointer(table->_last);

                // Reset iteration if either the begin or end pointer has changed:
                if (new_first != saved_first || new_last != saved_last)
                {
                    first = saved_first = new_first;
                    last  = saved_last  = new_last;
                }
            }
        }
        // ...
    });
}

// Appends the given 'function' to the given onexit 'table'.  Returns 0 on
// success; returns -1 on failure.  In general, failures are considered fatal
// in calling code.
extern "C" int __cdecl _register_onexit_function(_onexit_table_t* const table, _onexit_t const function)
{
    return __acrt_lock_and_call(__acrt_select_exit_lock(), [&]
    {
        // ...
    }
}
{{< /highlight >}}
Both functions are technically lambdas guarded by the same Critical Section and a 'try..except' SEH guard. 

In the case of `_execute_onexit_table`, it iterates over a linked list of handlers, decodes the next non-null callback `function()` using the LIFO rule, marks it as executed by setting it to nullptr, executes it, and resets the iteration from the last added callback point if a new hander was added during the execution of the callback.  

This kind of recursion works fine because Critical Section functions like a _recursive_ mutex, thus `_register_onexit_function` will not be blocked when called from an `atexit` handler of the same thread. 

However, a deadlock will occur if `function()` is waining on another thread while that thread tries to lock the same Critical Section to add another `atexit` handler. 

### GNU C Library 

Another widespread C library is a glibc. Let's look at its `/stdlib/exit.c: __run_exit_handlers()` [source code](https://github.com/bminor/glibc/blob/4290aed05135ae4c0272006442d147f2155e70d7/stdlib/exit.c#L98):
{{< highlight cpp "hl_lines=12 50-52 65">}}
/* Call all functions registered with `atexit' and `on_exit',
   in the reverse of the order in which they were registered
   perform stdio cleanup, and terminate program execution with STATUS.  */
void
attribute_hidden
__run_exit_handlers (int status, struct exit_function_list **listp,
             bool run_list_atexit, bool run_dtors)
{
  /* First, call the TLS destructors.  */
  // ...

  __libc_lock_lock (__exit_funcs_lock);

  /* We do it this way to handle recursive calls to exit () made by
     the functions registered with `atexit' and `on_exit'. We call
     everyone on the list and use the status value in the last
     exit (). */
   while (true)
   {
      struct exit_function_list *cur;

     restart:
      cur = *listp;

      if (cur == NULL)
      {
        /* Exit processing complete.  We will not allow any more
           atexit/on_exit registrations.  */
        __exit_funcs_done = true;
        break;
      }

      while (cur->idx > 0)
      {
        struct exit_function *const f = &cur->fns[--cur->idx];
        const uint64_t new_exitfn_called = __new_exitfn_called;
  
        switch (f->flavor)
        {
            void (*atfct) (void);
            void (*onfct) (int status, void *arg);
            void (*cxafct) (void *arg, int status);
            void *arg;
            // ...
          case ef_at:
            atfct = f->func.at;
            PTR_DEMANGLE (atfct);
  
            /* Unlock the list while we call a foreign function.  */
            __libc_lock_unlock (__exit_funcs_lock);
            atfct ();
            __libc_lock_lock (__exit_funcs_lock);
            break;
            // ...
        }
  
        if (__glibc_unlikely (new_exitfn_called != __new_exitfn_called))
          /* The last exit function, or another thread, has registered
             more exit functions.  Start the loop over.  */
          goto restart;
      }
      // ...
    }

  __libc_lock_unlock (__exit_funcs_lock);
  // ...
}
{{< /highlight >}}
It's a bit more intricate than Window UCRT, but in general, it's the same, except for one thing that avoid the deadlock:
{{< highlight cpp>}}
    /* Unlock the list while we call a foreign function.  */
    __libc_lock_unlock (__exit_funcs_lock);
    atfct ();
    __libc_lock_lock (__exit_funcs_lock);
    break;
{{< /highlight >}}

The GNU Libc implementation unlocks the mutex for a callback, avoiding the described deadlock.

# Conclusions

* `static` object creation and destruction may incur a mutex lock.
* `static` objects will implicitly add their destructors to `std::atexit` callbacks list, maintaining thread-safety with a mutex.
* `std::exit` locks the callbacks list and may not unlock it (although glibc does) during the callback execution. 
* the deadlock may occur if cleanup callback (for example, a static object destructor) waits for another thread, while another thread wants to add another cleanup callback.

In my case, the issue was fixed by joining the worker thread before exiting from the `main()` but I think it's an interesting and non-trivial multithreading pitfall. 

There is an [issue reported in a VS feedback tracker](https://developercommunity.visualstudio.com/t/atexit-deadlock-with-thread-safe-static-/1654756) in 2022, but it doesn't seem to get enough votes to address it yet.

Have a nice day!