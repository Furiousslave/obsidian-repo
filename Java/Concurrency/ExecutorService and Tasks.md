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