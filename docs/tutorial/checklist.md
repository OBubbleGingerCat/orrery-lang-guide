# The whole checklist

When a model is written, go through this list in order: is it right,
is it fast on 1 thread, does it get faster with threads, is it easy to
read. Each item names the chapter that explains it. Chapters 6, 7 and 9
are still to come; they will add their items here.

## Right

- Is the result the same on every thread count? Check a checksum of the
  final state on 1, 2, 4 and 8 threads ([chapter 1](./measure)).
- With every gate kept open, are the results the same, at the same
  cycles ([chapters 2](./gates) and [4](./wakes))?
- For each wake: does the woken task act before or after the task that
  wakes it, in the same cycle? Which row of [chapter 4](./wakes)'s
  table is it? In the third row, did I move the consumer a level later,
  or keep its gate open while a value can arrive?
- Does a producer wait for room in a FIFO whose consumer acts first?
  Then its gate stays open while it waits: the consumer's wake would be
  a cycle late ([chapter 4](./wakes)).
- Is each edge a gate closes until the edge where the waiting value is
  seen? Does a task at a later delta level count from delta 0 of its
  time ([chapter 3](./due))?
- Does every arbiter see all of the cycle's requests, and move its
  pointer the way the hardware does ([chapter 8](./arbitration))?
- Is every order within a cycle said by an access mode (`&`,
  `&'latest`)? Can I write down why each delta level is there
  ([chapter 5](./lines))?
- Do two units that send to each other have a take task and a send task
  each, or a line with a delay ([chapter 5](./lines))?
- Did I keep a plain version (counting down, resource by resource) as a
  test, giving the same results at the same cycles
  ([chapters 3](./due) and [8](./arbitration))?

## Fast on 1 thread

- Can I say in one sentence when each gate closes? Does every way work
  can reach the unit open it: an ungated event, or a wake from each
  sender ([chapter 2](./gates))?
- Is there a counter that goes down by one every cycle? Make it a due
  edge, and close the gate until it ([chapter 3](./due)).
- Is each arbitration domain one task, which looks at each requester
  once a cycle ([chapter 8](./arbitration))?
- Did I measure the model with its gates and with no gate at all (not
  only with the gates kept open), on the inputs I will run? A gate pays
  only where its unit is idle a lot; elsewhere leave it out
  ([chapter 2](./gates)).
- Do the FIFOs and queues that a task's instances meet all have the same
  capacity? Does the load report list no task under
  `counts.fits.generic` ([chapter 1](./measure))?

## Faster with threads

- Does every line that takes time in the hardware declare its delay
  ([chapter 5](./lines))?
- In a typical cycle, how many instances work, and are they neighbours?
  Did I try `Dealt` ([chapter 10](./threads))?
- Did I run each kind of input once on 1 thread and once on the most,
  and keep the faster? How many execution units are there on each
  ([chapter 10](./threads))?
- Is there a kind of task that runs, or keeps its gate closed, in every
  cycle while doing almost nothing ([chapter 10](./threads))?

## Easy to read

- Did I count the three readability numbers, before and after each
  change ([chapter 1](./measure))?
- Is one hardware unit's behaviour in one place ([chapter 1](./measure))?
- Does a comment at each wake name the pairing that makes it in time
  ([chapter 4](./wakes))?

## How I measured

- Did I keep the old way, and measure old and new in one run, taking
  turns ([chapter 1](./measure))?
- Did I time 1, 4 and 8 threads (2 adds nothing), three times each, the
  run apart from the build ([chapters 1](./measure) and
  [10](./threads))? This is for comparing two writings; to choose the
  threads for a run, see
  [How many threads](../guide/running#how-many-threads).
- Did I write down, for each change, "better" or "mixed", and what it
  does on 8 threads ([introduction](./#when-the-three-disagree))?
