# Implementations
We can create a thread pool by calling one of the static factory methods in **Executors**:
- **newFixedThreadPool**. A fixed-size thread pool creates threads as tasks are submitted, up to the maximum pool size, and then attempts to keep the pool size constant (adding new threads if a thread dies due to an unexpected Exception). 
- **newCachedThreadPool**. A cached thread pool has more flexibility to reap idle threads when the current size of the pool exceeds the demand for processing, and to add new threads when demand increases, but places no bounds on the size of the pool. 
- **newSingleThreadExecutor**. A single-threaded executor creates a single worker thread to process tasks, replacing it if it dies unexpectedly. Tasks are guaranteed to be processed sequentially according to the order imposed by the task queue (FIFO, LIFO, priority order).4 
- **newScheduledThreadPool**. A fixed-size thread pool that supports delayed and periodic task execution, similar to Timer.
# Lifecycle
The lifecycle management methods of ExecutorService are shown in listing:
```
public interface ExecutorService extends Executor { 
	void shutdown(); 
	List shutdownNow(); 
	boolean isShutdown();
	boolean isTerminated(); 
	boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException; // ... additional convenience methods for task submission 
}
```
The lifecycle implied by ExecutorService has three states—**running**, **shutting down**, and **terminated**. ExecutorServices are initially created in the **running** state. The shutdown method initiates a graceful shutdown: no new tasks are accepted but previously submitted tasks are allowed to complete—including those that have not yet begun execution. The shutdownNow method initiates an abrupt shutdown: it attempts to cancel outstanding tasks and does not start any tasks that are queued but not begun. Once all tasks have completed, the ExecutorService transitions to the **terminated** state. You can wa
it for an ExecutorService to reach the terminated state with awaitTermination, or poll for whether it has yet terminated with isTerminated. It is common to follow shutdown immediately by awaitTermination, creating the effect of synchronously shutting down the ExecutorService.
## Example
LifecycleWebServer in listing extends our web server with lifecycle support. It can be shut down in two ways: programmatically by calling stop, and through a client request by sending the web server a specially formatted HTTP request:
```
class LifecycleWebServer {
  private final ExecutorService exec = ...;
  
  public void start() throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (!exec.isShutdown()) {
      try {
        final Socket conn = socket.accept();
        exec.execute(new Runnable() {
          public void run() {
            handleRequest(conn);
          }
        });
      } catch (RejectedExecutionException e) {
        if (!exec.isShutdown())
          log("task submission rejected", e);
      }
    }
  }
  
  public void stop() {
    exec.shutdown();
  
  }
  void handleRequest(Socket connection) {
    Request req = readRequest(connection);
    if (isShutdownRequest(req))
      stop();
    else
      dispatchRequest(req);
  }
}
```
# Tasks
Many tasks are effectively deferred computations—executing a database query, fetching a resource over the network, or computing a complicated function. For these types of tasks, **Callable** is a better abstraction: it expects that the main entry point, call, will return a value and anticipates that it might throw an exception. Executors includes several utility methods for wrapping other types of tasks, including **Runnable** and **java.security.PrivilegedAction**, with a **Callable**.

**Runnable** and **Callable** describe abstract computational tasks. Tasks are usually finite: they have a clear starting point and they eventually terminate. The lifecycle of a task executed by an Executor has four phases: *created*, *submitted*, *started*, and *completed*. Since tasks can take a long time to run, we also want to be able to cancel a task. In the Executor framework, tasks that have been submitted but not yet started can always be cancelled, and tasks that have started can sometimes be cancelled if they are responsive to interruption. Cancelling a task that has already completed has no effect.

Future represents the lifecycle of a task and provides methods to test whether the task has completed or been cancelled, retrieve its result, and cancel the task. Callable and Future are shown in listing. Implicit in the specification of Future is that task lifecycle can only move forwards, not backwards—just like the ExecutorService lifecycle. Once a task is completed, it stays in that state forever.
```
public interface Callable<V> { V call() throws Exception; }

public interface Future<V> {
  boolean cancel(boolean mayInterruptIfRunning);
  boolean isCancelled();
  boolean isDone();
  V get() throws InterruptedException, ExecutionException, CancellationException;
  V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, CancellationException, TimeoutException;
}
```
The behavior of *get* varies depending on the task state (not yet started, running, completed). It returns immediately or throws an Exception if the task has already completed, but if not it blocks until the task completes. If the task completes by throwing an exception, get rethrows it wrapped in an ExecutionException; if it was cancelled, get throws CancellationException. If get throws ExecutionException, the underlying exception can be retrieved with getCause.

There are several ways to create a Future to describe a task. The submit methods in ExecutorService all return a Future, so that you can submit a Runnable or a Callable to an executor and get back a Future that can be used to retrieve the result or cancel the task. You can also explicitly instantiate a FutureTask for a given Runnable or Callable. (Because FutureTask implements Runnable, it can be submitted to an Executor for execution or executed directly by calling its run method.)
## Example
```
public class FutureRenderer {
  private final ExecutorService executor = ...;
  
  void renderPage(CharSequence source) {
    final List<ImageInfo> imageInfos = scanForImageInfo(source);
    Callable<List<ImageData>> task = new Callable<List<ImageData>>() {
      public List<ImageData> call() {
        List<ImageData> result = new ArrayList<ImageData>();
        for (ImageInfo imageInfo : imageInfos)
          result.add(imageInfo.downloadImage());
        return result;
      }
    };
    
    Future<List<ImageData>> future = executor.submit(task);
    renderText(source);
    
    try {
      List<ImageData> imageData = future.get();
      for (ImageData data : imageData) renderImage(data);
    } catch (InterruptedException e) {
      // Re-assert the thread’s interrupted status
      Thread.currentThread().interrupt();
      // We don’t need the result, so cancel the task too
      future.cancel(true);
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```
# CompletionService
If you have a batch of computations to submit to an Executor and you want to retrieve their results as they become available, you could retain the Future associated with each task and repeatedly poll for completion by calling get with a timeout of zero. This is possible, but tedious. Fortunately there is a better way: a *completion service*.

CompletionService combines the functionality of an Executor and a BlockingQueue. You can submit Callable tasks to it for execution and use the queuelike methods take and poll to retrieve completed results, packaged as Futures, as they become available. ExecutorCompletionService implements CompletionService, delegating the computation to an Executor.
## Example
```
public class Renderer {
  
  private final ExecutorService executor;
  
  Renderer(ExecutorService executor) {
    this.executor = executor;
  }
  
  void renderPage(CharSequence source) {
    List<ImageInfo> info = scanForImageInfo(source);
    CompletionService<ImageData> completionService =
        new ExecutorCompletionService<ImageData>(executor);
    for (final ImageInfo imageInfo : info)
      completionService.submit(new Callable<ImageData>() {
        public ImageData call() {
          return imageInfo.downloadImage();
        }
      });
    
    renderText(source);
    
    try {
      for (int t = 0, n = info.size(); t < n; t++) {
        Future<ImageData> f = completionService.take();
        ImageData imageData = f.get();
        renderImage(imageData);
      }
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```
# Placing time limits on tasks
Listing shows a typical application of a timed Future.get. It generates a composite web page that contains the requested content plus an advertisement fetched from an ad server. It submits the ad-fetching task to an executor, computes the rest of the page content, and then waits for the ad until its time budget runs out.8 If the get times out, it cancels the ad-fetching task and uses a default advertisement instead:
```
Page renderPageWithAd() throws InterruptedException {
  long endNanos = System.nanoTime() + TIME_BUDGET;
  Future<Ad> f = exec.submit(new FetchAdTask());
  // Render the page while waiting for the ad
  Page page = renderPageBody();
  Ad ad;
  try {
    // Only wait for the remaining time budget
    long timeLeft = endNanos - System.nanoTime();
    ad = f.get(timeLeft, NANOSECONDS);
  } catch (ExecutionException e) {
    ad = DEFAULT_AD;
  } catch (TimeoutException e) {
    ad = DEFAULT_AD;
    f.cancel(true);
  }
  page.setAd(ad);
  return page;
}
```
Next listing uses the timed version of invokeAll to submit multiple tasks to an ExecutorService and retrieve the results. The invokeAll method takes a collection of tasks and returns a collection of Futures. The two collections have identical structures; invokeAll adds the Futures to the returned collection in the order imposed by the task collection’s iterator, thus allowing the caller to associate a Future with the Callable it represents. The timed version of invokeAll will return when all the tasks have completed, the calling thread is interrupted, or the timeout expires. Any tasks that are not complete when the timeout expires are cancelled. On return from invokeAll, each task will have either completed normally or been cancelled; the client code can call get or isCancelled to find out which.
```
private class QuoteTask implements Callable<TravelQuote> {
  private final TravelCompany company;
  private final TravelInfo travelInfo;
  ... public TravelQuote call() throws Exception {
    return company.solicitQuote(travelInfo);
  }
}

public List<TravelQuote> getRankedTravelQuotes(TravelInfo travelInfo,
    Set<TravelCompany> companies, Comparator<TravelQuote> ranking, long time,
    TimeUnit unit) throws InterruptedException {
  List<QuoteTask> tasks = new ArrayList<QuoteTask>();
  for (TravelCompany company : companies)
    tasks.add(new QuoteTask(company, travelInfo));
  
  List<Future<TravelQuote>> futures = exec.invokeAll(tasks, time, unit);
  
  List<TravelQuote> quotes = new ArrayList<TravelQuote>(tasks.size());
  Iterator<QuoteTask> taskIter = tasks.iterator();
  for (Future<TravelQuote> f : futures) {
    QuoteTask task = taskIter.next();
    try {
      quotes.add(f.get());
    } catch (ExecutionException e) {
      quotes.add(task.getFailureQuote(e.getCause()));
    } catch (CancellationException e) {
      quotes.add(task.getTimeoutQuote(e));
    }
  }
  
  Collections.sort(quotes, ranking);
  return quotes;
}
```
# Thread starvation deadlock
If tasks that depend on other tasks execute in a thread pool, they can deadlock. In a single-threaded executor, a task that submits another task to the same executor and waits for its result will always deadlock. The second task sits on the work queue until the first task completes, but the first will not complete because it is waiting for the result of the second task. The same thing can happen in larger thread pools if all threads are executing tasks that are blocked waiting for other tasks still on the work queue. This is called thread starvation deadlock, and can occur whenever a pool task initiates an unbounded blocking wait for some resource or condition that can succeed only through the action of another pool task, such as waiting for the return value or side effect of another task, unless you can guarantee that the pool is large enough.

*Whenever you submit to an Executor tasks that are not independent, be aware of the possibility of thread starvation deadlock, and document any pool sizing or configuration constraints in the code or configuration file where the Executor is configured.*
## Example:
```
public class ThreadDeadlock {
  ExecutorService exec = Executors.newSingleThreadExecutor();
  
  public class RenderPageTask implements Callable<String> {
    public String call() throws Exception {
      Future<String> header, footer;
      header = exec.submit(new LoadFileTask("header.html"));
      footer = exec.submit(new LoadFileTask("footer.html"));
      String page = renderBody();
      // Will deadlock -- task waiting for result of subtask
      // The only one thread is busy doing RenderPageTask
      return header.get() + page + footer.get();
    }
  }
}
```
# Sizing thread pools
For compute-intensive tasks, an Ncpu-processor system usually achieves optimum utilization with a thread pool of Ncpu + 1 threads. (Even compute-intensive threads occasionally take a page fault or pause for some other reason, so an “extra” runnable thread prevents CPU cycles from going unused when this happens.) For tasks that also include I/O or other blocking operations, you want a larger pool, since not all of the threads will be schedulable at all times. In order to size the pool properly, you must estimate the ratio of waiting time to compute time for your tasks; this estimate need not be precise and can be obtained through profiling or instrumentation. Alternatively, the size of the thread pool can be tuned 8.3. Configuring ThreadPoolExecutor 171 by running the application using several different pool sizes under a benchmark load and observing the level of CPU utilization.
Given these definitions:
- Ncpu = number of CPUs
- Ucpu = target CPU utilization, 0 ≤ Ucpu ≤ 1
- W/C = ratio of wait time to compute time
The optimal pool size for keeping the processors at the desired utilization is:
$$
Nthreads = Ncpu ∗ Ucpu ∗ (1 + W/C)
$$
You can determine the number of CPUs using Runtime: 
`int N_CPUS = Runtime.getRuntime().availableProcessors();`

When tasks require a pooled resource such as database connections, thread pool size and resource pool size affect each other. If each task requires a connection, the effective size of the thread pool is limited by the connection pool size. Similarly, when the only consumers of connections are pool tasks, the effective size of the connection pool is limited by the thread pool size.
# Managing queued tasks
Bounded thread pools limit the number of tasks that can be executed concurrently. (The single-threaded executors are a notable special case: they guarantee that no tasks will execute concurrently, offering the possibility of achieving thread safety through thread confinement.)

If the arrival rate for new requests exceeds the rate at which they can be handled, requests will still queue up. With a thread pool, they wait in a queue of Runnables managed by the Executor instead of queueing up as threads contending for the CPU. Representing a waiting task with a Runnable and a list node is certainly a lot cheaper than with a thread, but the risk of resource exhaustion still remains if clients can throw requests at the server faster than it can handle them.

Requests often arrive in bursts even when the average request rate is fairly stable. Queues can help smooth out transient bursts of tasks, but if tasks continue to arrive too quickly you will eventually have to throttle the arrival rate to avoid running out of memory. Even before you run out of memory, response time will get progressively worse as the task queue grows

ThreadPoolExecutor allows you to supply a BlockingQueue to hold tasks awaiting execution. There are three basic approaches to task queueing:
- unbounded queue
- bounded queue 
- synchronous handoff. 
The choice of queue interacts with other configuration parameters such as pool size. The default for `newFixedThreadPool` and `newSingleThreadExecutor` is to use an unbounded `LinkedBlockingQueue`. Tasks will queue up if all worker threads are busy, but the queue could grow without bound if the tasks keep arriving faster than they can be executed.

A more stable resource management strategy is to use a bounded queue, such as an `ArrayBlockingQueue` or a bounded `LinkedBlockingQueue` or `PriorityBlockingQueue`. Bounded queues help prevent resource exhaustion but introduce the question of what to do with new tasks when the queue is full. With a bounded work queue, the queue size and pool size must be tuned together. A large queue coupled with a small pool can help reduce memory usage, CPU usage, and context switching, at the cost of potentially constraining throughput.

For very large or unbounded pools, you can also bypass queueing entirely and instead hand off tasks directly from producers to worker threads using a `SynchronousQueue`. A `SynchronousQueue` is not really a queue at all, but a mechanism for managing handoffs between threads. In order to put an element on a `SynchronousQueue`, another thread must already be waiting to accept the handoff. If no thread is waiting but the current pool size is less than the maximum, `ThreadPoolExecutor` creates a new thread; otherwise the task is rejected according to the saturation policy. Using a direct handoff is more efficient because the task can be handed right to the thread that will execute it, rather than first placing it on a queue and then having the worker thread fetch it from the queue. SynchronousQueue is a practical choice only if the pool is unbounded or if rejecting excess tasks is acceptable. *The newCachedThreadPool factory uses a SynchronousQueue.*

*The newCachedThreadPool factory is a good default choice for an Executor, providing better queuing performance than a fixed thread pool. A fixed size thread pool is a good choice when you need to limit the number of concurrent tasks for resource-management purposes, as in a server application that accepts requests from network clients and would otherwise be vulnerable to overload.*

Bounding either the thread pool or the work queue is suitable only when tasks are independent. With tasks that depend on other tasks, bounded thread pools or queues can cause thread starvation deadlock; instead, use an unbounded pool configuration like newCachedThreadPool.
# Saturation policies
When a bounded work queue fills up, the saturation policy comes into play. The saturation policy for a `ThreadPoolExecutor` can be modified by calling `setRejectedExecutionHandler`. (The saturation policy is also used when a task is submitted to an `Executor` that has been shut down.) Several implementations of `RejectedExecutionHandler` are provided, each implementing a different saturation policy: `AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy`, and `DiscardOldestPolicy`.

The default policy, **abort**, causes execute to throw the unchecked `RejectedExecutionException`; the caller can catch this exception and implement its own overflow handling as it sees fit. The **discard** policy silently discards the newly submitted task if it cannot be queued for execution; the **discard-oldest** policy discards the task that would otherwise be executed next and tries to resubmit the new task. (If the work queue is a priority queue, this discards the highest-priority element, so the combination of a discard-oldest saturation policy and a priority queue is not a good one.)

The **caller-runs** policy implements a form of throttling that neither discards tasks nor throws an exception, but instead tries to slow down the flow of new tasks by pushing some of the work back to the caller. It executes the newly submitted task not in a pool thread, but in the thread that calls execute.
# Extending ThreadPoolExecutor
`ThreadPoolExecutor` was designed for extension, providing several “hooks” for subclasses to override—`beforeExecute`, `afterExecute`, and `terminated`—that can be used to extend the behavior of `ThreadPoolExecutor`. The `beforeExecute` and `afterExecute` hooks are called in the thread that executes the task, and can be used for adding logging, timing, monitoring, or statistics gathering. The `afterExecute` hook is called whether the task completes by returning normally from run or by throwing an Exception. (If the task completes with an `Error`, `afterExecute` is not called.) If `beforeExecute` throws a `RuntimeException`, the task is not executed and `afterExecute` is not called. The `terminated` hook is called when the thread pool completes the shutdown process, after all tasks have finished and all worker threads have shut down. It can be used to release resources allocated by the `Executor` during its lifecycle, perform notification or logging, or finalize statistics gathering.
## Example:
```
public class TimingThreadPool extends ThreadPoolExecutor {
  private final ThreadLocal<Long> startTime = new ThreadLocal<Long>();
  private final Logger log = Logger.getLogger("TimingThreadPool");
  private final AtomicLong numTasks = new AtomicLong();
  private final AtomicLong totalTime = new AtomicLong();
  
  protected void beforeExecute(Thread t, Runnable r) {
    super.beforeExecute(t, r);
    log.fine(String.format("Thread %s: start %s", t, r));
    startTime.set(System.nanoTime());
  }
  
  protected void afterExecute(Runnable r, Throwable t) {
    try {
      long endTime = System.nanoTime();
      long taskTime = endTime - startTime.get();
      numTasks.incrementAndGet();
      totalTime.addAndGet(taskTime);
      log.fine(String.format("Thread %s: end %s, time=%dns", t, r, taskTime));
    } finally {
      super.afterExecute(r, t);
    }
  }
  
  protected void terminated() {
    try {
      log.info(String.format(
          "Terminated: avg time=%dns", totalTime.get() / numTasks.get()));
    } finally {
      super.terminated();
    }
  }
}
```




