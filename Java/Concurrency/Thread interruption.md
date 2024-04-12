 If a thread is interrupted when it is not blocked, its interrupted status is set, and it is up to the activity being cancelled to poll the interrupted status to detect interruption. In this way interruption is “sticky”—if it doesn’t trigger an InterruptedException, evidence of interruption persists until someone deliberately clears the interrupted status.

Otherwise, if a thread interrupted when it is blocked by methods such as `Thread.sleep` or `BlockingQueue.put`, interrupt status may be cleared and it will receive an *InterruptedException*.
When a method can throw InterruptedException, it is telling you that it is a blocking method, and further that if it is interrupted, it will make an effort to stop blocking early.

*Calling interrupt does not necessarily stop the target thread from doing what it is doing; it merely delivers the message that interruption has been requested*.
```
class PrimeProducer extends Thread {
  private final BlockingQueue<BigInteger> queue;
  
  PrimeProducer(BlockingQueue<BigInteger> queue) {
    this.queue = queue;
  }
  
  public void run() {
    try {
      BigInteger p = BigInteger.ONE;
      while (!Thread.currentThread().isInterrupted())
        queue.put(p = p.nextProbablePrime());
    } catch (InterruptedException consumed) {
      /* Allow thread to exit */
    }
  }
  
  public void cancel() {
    interrupt();
  }
}
```
A task needn’t necessarily drop everything when it detects an interruption request—it can choose to postpone it until a more opportune time by remembering that it was interrupted, finishing the task it was performing, and then throwing InterruptedException or otherwise indicating interruption. This technique can protect data structures from corruption when an activity is interrupted in the middle of an update.

A task should not assume anything about the interruption policy of its executing thread unless it is explicitly designed to run within a service that has a specific interruption policy. Whether a task interprets interruption as cancellation or takes some other action on interruption, it should take care to preserve the executing thread’s interruption status. If it is not simply going to propagate InterruptedException to its caller, it should restore the interruption status after catching InterruptedException: `Thread.currentThread().interrupt();`

Just as task code should not make assumptions about what interruption means to its executing thread, cancellation code should not make assumptions about the interruption policy of arbitrary threads. A thread should be interrupted only by its owner; the owner can encapsulate knowledge of the thread’s interruption policy in an appropriate cancellation mechanism such as a shutdown method.

*Because each thread has its own interruption policy, you should not interrupt a thread unless you know what interruption means to that thread.*

When you call an interruptible blocking method such as Thread.sleep or BlockingQueue.put, there are two practical strategies for handling InterruptedException: 
- Propagate the exception (possibly after some task-specific cleanup), making your method an interruptible blocking method, too; or 
- Restore the interruption status so that code higher up on the call stack can deal with it.

*Only code that implements a thread’s interruption policy may swallow an interruption request. General-purpose task and library code should never swallow interruption requests.*
## Cancellation via Future
ExecutorService.submit returns a Future describing the task. Future **has a cancel method** that takes a boolean argument, mayInterruptIfRunning, and returns a value indicating whether the cancellation attempt was successful. When mayInterruptIfRunning is true and the task is currently running in some thread, then that thread is interrupted. Setting this argument to false means “don’t run this task if it hasn’t started yet”, and should be used for tasks that are not designed to handle interruption.
The task execution threads created by the standard Executor implementations implement an interruption policy that lets tasks be cancelled using interruption, so it is safe to set mayInterruptIfRunning when cancelling tasks through their Futures when they are running in a standard Executor. **You should not interrupt a pool thread directly when attempting to cancel a task**, because you won’t know what task is running when the interrupt request is delivered—do this only through the task’s Future.

Listing shows a version of timedRun that submits the task to an ExecutorService and retrieves the result with a timed Future.get. If get terminates with a TimeoutException, the task is cancelled via its Future. (To simplify coding, this version calls Future.cancel unconditionally in a finally block, taking advantage of the fact that cancelling a completed task has no effect.) If the underlying computation throws an exception prior to cancellation, it is rethrown from timedRun, which is the most convenient way for the caller to deal with the exception. Listing also illustrates another good practice: cancelling tasks whose result is no longer needed.
```
public static void timedRun(Runnable r, long timeout, TimeUnit unit)
    throws InterruptedException {
  Future<?> task = taskExec.submit(r);
  try {
    task.get(timeout, unit);
  } catch (TimeoutException e) {
    // task will be cancelled below
  } catch (ExecutionException e) {
    // exception thrown in task; rethrow
    throw launderThrowable(e.getCause());
  } finally {
    // Harmless if task already completed
    task.cancel(true); // interrupt if running
  }
}
```
*When Future.get throws InterruptedException or TimeoutException and you know that the result is no longer needed by the program, cancel the task with Future.cancel.*