# Introduction: the sleep module

The code in this module governs when worker threads should go to
sleep. This is a tricky topic -- the work-stealing algorithm relies on
having active worker threads running around stealing from one
another. But, if there isn't a lot of work, this can be a bit of a
drag, because it requires high CPU usage.

The code in this module takes a fairly simple approach to the
problem. It allows worker threads to fall asleep if they have failed
to steal work after various thresholds; however, whenever new work
appears, they will wake up briefly and try to steal again. There are some
shortcomings in this current approach:

- it can (to some extent) scale *down* the amount of threads, but they
  can never be scaled *up*. The latter might be useful in the case of
  user tasks that must (perhaps very occasionally and unpredictably)
  block for long periods of time.
  - however, the preferred approach to this is for users to adopt futures
    instead (and indeed this sleeping work is intended to enable future
    integration).
- we have no way to wake up threads in a fine-grained or targeted
  manner. The current system wakes up *all* sleeping threads whenever
  *any* of them might be interested in an event. This means that while
  we can scale CPU usage down, we do is in a fairly "bursty" manner,
  where everyone comes online, then some of them go back offline.

# The interface for workers

Workers interact with the sleep module by invoking three methods:

- `work_found()`: signals that the worker found some work and is about
  to execute it.
- `no_work_found()`: signals that the worker searched all available sources
  for work and found none.
  - It is important for the coherence of the algorithm that if work
    was available **before the search started**, it would have been
    found. If work was made available during the search, then it's ok that
    it might have been overlooked.
- `tickle()`: indicates that new work is available (e.g., a job has
  been pushed to the local deque) or that some other blocking
  condition has been resolved (e.g., a latch has been set). Wakes up any
  sleeping workers.
  
When in a loop searching for work, Workers also have to maintain an
integer `yields` that they provide to the `sleep` module (which will
return a new value for the next time). Thus the basic worker "find
work" loop looks like this (this is `wait_until()`, basically):

```rust
let mut yields = 0;
while /* not done */ {
    if let Some(job) = search_for_work() {
        yields = work_found(self.index, yields);
    } else {
        yields = no_work_found(self.index, yields);
    }
}
```

# Getting sleepy and falling asleep

The basic idea here is that every worker goes through three states:

- **Awake:** actively hunting for tasks.
- **Sleepy:** still actively hunting for tasks, but we have signaled that
  we might go to sleep soon if we don't find any.
- **Asleep:** actually asleep (blocked on a condition variable).

At any given time, only **one** worker can be in the sleepy
state. This allows us to coordinate the entire sleep protocol using a
single `AtomicUsize` and without the need of epoch counters or other
things that might rollover and so forth.

Whenever a worker invokes `work_found()`, it transitions back to the
**awake** state. In other words, if it was sleepy, it stops being
sleepy. (`work_found()` cannot be invoked when the worker is asleep,
since then it is not doing anything.)

On the other hand, whenever a worker invokes `no_work_found()`, it
*may* transition to a more sleepy state. To track this, we use the
counter `yields` that is maintained by the worker's steal loop. This
counter starts at 0. Whenever work is found, the counter is returned
to 0. But each time that **no** work is found, the counter is
incremented. Eventually it will reach a threshold
`ROUNDS_UNTIL_SLEEPY`.  At this point, the worker will try to become
the sleepy one. It does this by executing a CAS into the global
registry state (details on this below). If that attempt is successful,
then the counter is incremented again, so that it is equal to
`ROUNDS_UNTIL_SLEEPY + 1`. Otherwise, the counter stays the same (and
hence we will keep trying to become sleepy until either work is found
or we are successful).

Becoming sleepy does not put us to sleep immediately. Instead, we keep
iterating and looking for work for some further number of rounds.  If
during this search we **do** find work, then we will return the
counter to 0 and also reset the global state to indicate we are no
longer sleepy.

But if again no work is found, `yields` will eventually reach the
value `ROUNDS_UNTIL_ASLEEP`. At that point, we will try to transition
from **sleepy** to **asleep**. This is done by the helper fn
`sleep()`, which executes another CAS on the global state that removes
our worker as the sleepy worker and instead sets a flag to indicate
that there are sleeping workers present (the flag may already have
been set, that's ok). Assuming that CAS succeeds, we will block on a
condition variable.

# Tickling workers

Of course, while all the stuff in the previous section is happening,
other workers are (hopefully) producing new work. There are three kinds of
events that can allow a blocked worker to make progress:

1. A new task is pushed onto a worker's deque. This task could be stolen.
2. A new task is injected into the thread-pool from the outside. This
   task could be uninjected and executed.
3. A latch is set. One of the sleeping workers might have been waiting for
   that before it could go on.
   
Whenever one of these things happens, the worker (or thread, more generally)
responsible must invoke `tickle()`. Tickle will basically wake up **all**
the workers:

- If any worker was the sleepy one, then the global state is changed
  so that there is no sleepy worker. The sleepy one will notice this
  when it next invokes `no_work_found()` and return to the *awake* state
  (with a yield counter of zero).
- If any workers were actually **asleep**, then we invoke
  `notify_all()` on the condition variable, which will cause them to
  awaken and start over from the awake state (with a yield counter of
  zero).
  
Because `tickle()` is invoked very frequently -- and hopefully most of
the time it is not needed, because the workers are already actively
stealing -- it is important that it be very cheap. The current design
requires, in the case where nobody is even sleepy, just a load and a
compare. If there are sleepy workers, a swap is needed.  If there
workers *asleep*, we must naturally acquire the lock and signal the
condition variable.

# The global state

We manage all of the above state transitions using a small bit of global
state (well, global to the registry). This is stored in the `Sleep` struct.
The primary thing is a single `AtomicUsize`. The value in this usize packs
in two pieces of information:

1. **Are any workers asleep?** This is just one bit (yes or no).
2. **Which worker is the sleepy worker, if any?** This is a worker id.

We use bit 0 to indicate whether any workers are asleep. So if `state
& 1` is zero, then no workers are sleeping. But if `state & 1` is 1,
then some workers are either sleeping or on their way to falling
asleep (i.e., they have acquired the lock).

The remaining bits are used to store if there is a sleepy worker. We
want `0` to indicate that there is no sleepy worker. If there a sleepy
worker with index `worker_index`, we would store `(worker_index + 1)
<< 1` . The `+1` is there because worker indices are 0-based, so this
ensures that the value is non-zero, and the shift skips over the
sleepy bit.

Some examples:

- `0`: everyone is awake, nobody is sleepy
- `1`: some workers are asleep, no sleepy worker
- `2`: no workers are asleep, but worker 0 is sleepy (`(0 + 1) << 1 == 2`).
- `3`: some workers are asleep, and worker 0 is sleepy.

# Correctness level 1: avoiding deadlocks etc

In general, we do not want to miss wakeups. Two bad things could happen:

- If this is a wakeup about a new job being pushed into a local deque,
  it won't deadlock, but it will cause things to run slowly. The
  reason that it won't deadlock is that we know at least one thread is active (the one
  doing the pushing), and it will (sooner or later) try to pop this item from its
  own local deque.
- If this is a wakeup about an injected job or a latch that got set, however,
  this can cause deadlocks. In the former case, if a job is injected but no thread ever
  wakes to process it, the injector will likely block forever. In the latter case,
  imagine this scenario:
  - thread A calls join, forking a task T1, then executing task T2
  - thread B steals T1, forks a task T3, and executes T4.
  - thread A completes task T2 and blocks on T1
  - thread A steals task T3 from thread B
  - thread B finishes T4 and goes to sleep, blocking on T3
  - thread A completes task T3 and makes a wakeup, but it gets lost
  At this point, thread B is still asleep and will never signal T2, so thread A will itself
  go to sleep. Bad.
  
So how do we ensure that we don't miss wakeups? Absent a central
queue, this can be a bit tricky. Let's consider the simplest scheme:
imagine we just had a boolean flag indicating whether anyone was
asleep. Then you could imagine that when workers find no work, they
flip this flag to true. When work is published, if the flag is true,
we issue a wakeup.

The problem here is that checking for new work is not an atomic
action. So it's possible that worker 1 could start looking for work
and (say) see that worker 0's queue is empty and then search workers
2..N.  While that searching is taking place, worker 0 publishes some
new work.  At the time when the new work is published, the "anyone
sleeping?" flag is still false, so nothing happens. Then worker 1, who
failed to find any work, goes to sleep --- completely missing the wakeup!

We use the "sleepy worker" idea to sidestep this problem. Under our
scheme, instead of going right to sleep at the end, worker 1 would
become sleepy.  Worker 1 would then do **at least** one additional
scan. During this scan, they should find the work published by worker
0, so they will stop being sleepy and go back to work (here of course
we are assuming that no one else has stolen the worker 0 work yet; if
someone else stole it, worker 1 may still go to sleep, but that's ok,
since there is no more work to be had).

Now you may be wondering -- how does being sleepy help? What if, instead of publishing
its job right before worker 1 became sleepy, worker 0 wait until right before worker 1
was going to go to sleep? In other words, the sequence was like this:

- worker 1 gets sleepy
- worker 1 starts its scan, scanning worker 0's deque
- worker 0 publishes its job, but nobody is sleeping yet, so no wakeups occur
- worker 1 finshes its scan, goes to sleep, missing the wakeup

The reason that this doesn't occur is because, when worker 0 publishes
its job, it will see that there is a sleepy worker. It will clear the
global state to 0.  Then, when worker 1 its scan, it will notice that
it is no longer sleepy, and hence it will not go to sleep. Instead it
will awaken and keep searching for work.

The sleepy worker phase thus also serves as a cheap way to signal that
work is around: instead of doing the whole dance of acquiring a lock
and issuing notifications, when we publish work we can just swap a
single atomic counter and let the sleepy worker notice that on their
own.

# Correctness level 2: ensuring sequential consistency

The reasoning in the previous section was assuming sequential
consistency, for the most part. In particular, we assumed that it's
reasonable to speak of publishing an event as coming "before" a scan
starts, and we assumed that if a worker is scanning for work, and work
has been published, it will be found. However, if we are not careful
with the atomic ordering constraints, this will not necessarily be the
case. We have opted to use SeqCst as the ordering constraints
throughout the sleep module. Other parts of Rayon tend use an
acquire-release ordering.

The place that this is most relevant is the load in the `tickle()`
routine. The routine begins by reading from the global state. If it
sees anything other than 0, it then does a swap and -- if necessary --
acquires a lock and does a notify. **The key point is that this
initial read is used to detect if the later writes are necessary and
it is not sufficient for it to be an "Acquire" load.** To see why,
let's walk through several scenarios.

## Scenario 1: tickle-then-get-sleepy

We want to ensure that this order of events does not occur:

1. worker 1 is blocked on latch L
2. worker 1 reads latch L, sees false, starts searching for work
3. worker 0 sets latch L, tickles
    - the tickle reads from the global state, sees 0
4. worker 1 finishes searching, becomes sleepy
    - becoming sleepy involves a CAS on the global state to set it to 4 ("worker 1 is sleepy")
5. worker 1 reads latch L **but does not see that worker 0 set it**
6. worker 1 may then proceed to become sleepy

We ensure, however, that this will not happen. The reason is that in
step 3 the tickle is a SeqCst load, and step 4 is also a SeqCst
operation. Therefore step 4 synchronizes-with step 3, and hence the
writes from worker 0 which happens-before the load in step 3 must be
visible to worker 1 in step 5. I think. :P

Note that this logic generalizes to any kind of action A that takes
place before the tickle: so long as the same thread which took the
action A then **later** executes the tickle, then if some other thread
gets sleepy, the action A should be visible to it.

## Scenario 2: get-sleepy-then-get-tickled

We want to ensure that this order of events does not occur:

1. worker 1 is blocked on latch L
2. worker 1 becomes sleepy
    - becoming sleepy involves a CAS on the global state to set it to 4 ("worker 1 is sleepy")
3. worker 0 sets latch L and then tickles **but does not see that worker 0 is sleepy**

In particular, if becoming sleepy were merely a *release* action and
tickling began with an acquire load, as you might expect (since that
is the typical pattern) then there would be **no guarantee** that the
tickle will observe that a sleepy worker occurred. We would be
guaranteed only that worker 0 would **eventually** observe that worker
1 had become sleepy (and, at that time, that it would see other
writes). But it could take time -- and if we indeed miss that worker 1
is sleepy, it could lead to deadlock or loss of efficiency, as
explained earlier.

These problems could also be alleviated by having tickle
unconditionally do a swap or compare-and-swap. This would enforce
ordering by the monotonic nature of a single atomic cell. But it would
also be more expensive than just doing a load -- and would force an
ordering on two different ticklers, which just having loads does not
(in principle there is an ordering, but the program cannot observe
which came first and which came second).

## Scenario 3: fall-asleep-and-then-get-tickled

The same as scenario 2, except replace "get sleepy" with "fall asleep".
