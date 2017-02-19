--- 
title: Implement an in-memory task queue by BlockingQueue
layout: post
---


In many times we encounter the demand that need to perform lots of same tasks which are produced constantly during our coding work. The tasks need to be consumed rapidly, otherwise they may pile up damaging the system. 

The following examples require this kind of demand:

* A web crawler that constantly browses the web and scans through web pages to create the indices.
* An application that process independent work items in the backgroud and in parallel (such as processing logs asynchronously).

### Basic ideas

An in-memory task queue can satisfy this kind of demand. A task queue is a data structure maintained by task manager containing queue to store tasks and workers to run.

In this post, we will implement a feasible task queue using Java BlockingQueue. In order to support most of the common demands we need the following features:

* Thread safe
* Multiple workers
* Support retry

### Java BlockingQueue

Before moving on, let's introduce the Java BlockingQueue first. The Java BlockingQueue interface is in the java.util.concurrent package, its implentations are thread-safe, and all of its queuing methods are atomic in nature and use internal locks.

There's four different behaviour of methods for inserting, removing and examining the elements in the queue with different ways of handling operations about throws exception, special value, blocks or timeout:

|          | Throws exception |  Special value |  Blocks |  Timout               |
|----------|------------------|----------------|---------|-----------------------|
|  Insert  |  add(e)          |  offer(e)      |  put(e) |  offer(e, time, unit) |
|  Remove  |  remove()        |  poll()        |  take() |  poll(time, unit)     |
|  Examine |  element()       |  peek()        |  -----  |  -----                |

These 4 different behaviour means this:

* Throws Exception: If the attempted operation is not possible immediately, an exception is thrown.
* Special Value: If the attempted operation is not possible immediately, a special value is returned (often true / false).
* Blocks: If the attempted operation is not possible immedidately, the method call blocks until it is.
* Timout: If the attempted operation is not possible immedidately, the method call blocks until it is, but waits no longer than the given timeout. Returns a special value telling whether the operation succeeded or not (typically true / false).

There's five essential implementations in java.util.concurrent package for BlockingQueue interface:

* ArrayBlockingQueue: A bounded queue that stores the limited elements internally in an array.
* DelayQueue: It blocks the elements internally until a certain delay has expired. The elements must implement the interface java.util.concurrent.Delayed.
* LinkedBlockingQueue: It keeyps the elements internally in a linked structure. It's upper bound can be Integer.MAX_VALUE.
* PriorityBlockingQueue: It's an unbounded queue, and the elements in the queue have their own priority.
* SynchronousQueue: There's only one element in the queue. A thread inserting an element into the queue is blocked until another thread takes that element from the queue.

### Implementation the TaskQueue
Let's give the TaskQueue's interface first, there's four method:

    public interface ITaskQueue<T> {

    	boolean start();

    	boolean stop();

    	void feed(T task);
    	
    	void consume(T task);
	}
Then our TaskQueue should have one queue and one worker list. Also, we need two integer to record its capacity and worker count. So, here's its internal members:
	    
	protected BlockingQueue<T> queue;
	protected List<WorkerThread> threads;
	protected int workerCount = DEFAULT_WORKER_COUNT;
	protected int capacity = DEFAULT_CAPACITY;
	    
By giving these internal members, the first three methods can be override easily:
   
    @Override
    public void feed(T task) {
        queue.add(task);
    }

    @Override
    public boolean start() {
        if (threads != null) {
            LOG.warn("Threads have already started.");
            return false;
        }

        LOG.info("Creating workerThreads, worker Count : " + workerCount);
        threads = new ArrayList<>(workerCount);

        IntStream.range(0, workerCount).forEach(i -> {
            WorkerThread thread = new WorkerThread(i + 1);
            thread.start();
            threads.add(thread);
            LOG.info(thread.getName() + " started");
        });

        return true;
    }

    @Override
    public boolean stop() {
        if (threads == null) {
            LOG.warn("threads == null while stop() called");
            return false;
        }

        threads.forEach(thread -> {
            thread.cancel(false);
            LOG.info("Cancelling " + thread.getName());
        });
        threads = null;

        return true;
    }


We use blockingqueue's two implentations: ArrayBlockingQueue and LinkedBlockingQueue for our task queue's internal queue. ArrayBlockingQueue is for bounded queue and LinkedBlockingQueue is for unbounded. Here's the init method:

    protected void init() {
        if (capacity <= 0) {
            queue = new LinkedBlockingQueue<>();
        } else {
            queue = new ArrayBlockingQueue<>(capacity);
        }
    }

Above is the simple implentation of our TaskQueue, we only need to override the consume method, then call the start method to start the TaskQueue. In order to make our TaskQueue more robust, we add a `retryEnabled` flag to support retry the task if it's failed. Here's the Worker thread which support retrying:

    protected class WorkerThread extends Thread {
        private final int id;

        public WorkerThread(int id) {
            this.id = id;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    T task = queue.take();
                    try {
                        consume(task);
                    } catch (Exception e) {
                        LOG.error("Consume task error.", e);
                        if (retryEnabled) {
                            boolean refeed = queue.offer(task);
                            LOG.error("Re-feed task to the queue: " + (refeed ? "succeed" : "failed"));
                        }
                    }
                } catch (Exception ee) {
                    LOG.error("", ee);
                    ThreadUtils.sleepQuietly(1000L);
                }
            }
        }
    }

The complete sourcode can be download in my [github](https://github.com/Itfly/commons).
