# What happens when you spawn?


Say you write

    int a;
    ...
    int x = cilk_spawn f1(g(a));
    int y = f2();
    cilk_sync;
    return x + y;

How does OpenCilk turn this into a parallel program?

## Defining the parallelism

One thing the compiler does is decide what needs to run in parallel.

Despite the syntax spawns do not really return values.  The assignment
to `x` has to be part of the spawn because it runs after the call to
`f1` and not in parallel with it.  The compiler pretends you wrote:

    int a;
    ...
    int x, y;
    cilk_spawn x = f1(g(a));
    y = f2();
    cilk_sync;
    return x + y;

Other code can't run in parallel even though it appears to.  The
calculation of `g(a)` has to be sequenced with the function.
What if `a` is modified after the spawn?

    int a;
    ...
    int x, y;
    int arg = g(a);
    cilk_spawn x = f1(arg);
    y = f2();
    cilk_sync;
    return x + y;

The approximate rule is the spawned code is everything from the call
to the spawned function to the end of the statement it is in.

Another transformation extracts the spawned code into a separate
function and adds calls into the OpenCilk runtime library.

    static void spawn_helper(int *x, int a)
    {
        __cilkrts_frame frame; // see cheetah/runtime/frame.h
        __cilkrts_enter_frame(&frame);
        int arg = g(a);        // argument evaluation is before detach
        __cilkrts_detach(&frame);
        *x = f1(arg);          // side effects are before leave frame
        __cilkrts_leave_frame(&frame); // may or may not return
    }

        int a;
        ...
        int x;
        spawn_helper(&x, a);
        int y = f2();
        __cilkrts_sync();
        return x + y;

The rewritten code is equivalent to the original except for the Cilk
runtime library calls.  Like the _{% defn "serial projection" %}_ of
user-written code is formed by removing the Cilk keywords, the serial
projection of the rewritten code is formed by removing these runtime
calls.

What do they do?

## Terminology

First, a bit of terminology.  The OpenCilk runtime automatically
creates a bunch of threads to run a program.  They are called
_workers_.  In a program with a lot of parallelism workers spend most
of their time in user code.  A worker with nothing to do tries to
_steal_ work from other workers, meaning it takes a function that is
not currently running and starts it running again.  In Cilk source
code you may see references to _thief_ and _victim_.  Cilk has what is
called a _work stealing_ scheduler.

## Runtime

The rewritten code above has one data type and four functions that
were not present in the user code.

The frame descriptor is present in every function that interacts with
the Cilk scheduler.  Every function that spawns has one and so do the
compiler-generated spawn helpers.  Other functions are invisible to
Cilk.  Only functions with frame descriptors can be moved from one
processor to another (stolen).

Every worker has a linked list of frame descriptors for active Cilk
functions.  The head of the list is the most recently entered Cilk
function on that worker.  The list continues up the call stack, but
not necessarily all the way to `main()`.  Every worker also has a
deque (double-ended queue) of functions that have spawned.  The _head_
of the deque holds the oldest spawn and the _tail_ holds the youngest.

`__cilkrts_enter_frame` prepends the current function to the linked
list.

The `__cilkrts_detach` call is the one that makes spawn work.  It is
very simple.  It pushes the parent function's frame descriptor onto
the tail of the deque of spawns.  It pushes the parent, not itself.
The deque only contains work that can be stolen by other processors.
The spawned function can not be stolen because it is still running.

Stealing the oldest frame, which is suspended at a spawn, is
fundamental to the way Cilk works.  Cilk never migrates a running
function.  It takes functions which are not running and resumes them
on an otherwise idle processor.

A stolen function can be resumed because it saved a _continuation_
when it spawned.  A Cilk continuation is not the first class data type
of Scheme.  It is a few words of data, mainly a program counter and a
frame pointer, that are just enough to allow a function to be resumed.
There is at most one saved continuation at a time per function
activation.  (Spawns within a function are always sequential;
`cilk_for` uses recursion to get around this limitation.)

The behavior of `__cilkrts_leave_frame` depends on whether this
function or its parent has ever been stolen.  If not, it removes the
function from the linked list of Cilk functions and returns normally.
Otherwise, it does not return because there is nothing to return to.
The parent function may be running on a different processor.  The
worker makes a note that the function has finished and looks for
something to steal.  (In fact there is an optimization for the case
where the parent is waiting at a sync for this function and no other.)

`__cilkrts_sync` also has a fast and a slow path.  If the function
has run serially, meaning it was never stolen, sync does nothing.
