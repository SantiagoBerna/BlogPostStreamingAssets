## Introduction

In most AAA game titles nowadays, we have seen a rising trend to creating bigger worlds with more content and even more detail. The open-world genre has become very prevalent in the gaming industry during the last 10 years and nowadays game download sizes are getting to the 100GB+ mark (mind you the downloaded files are also heavily compressed). 

Therefore, if there is one thing that these games (and the engines they run on) must do right is Asset Management. Managing the lifetime of hundreds of assets while making sure that a game has an acceptable framerate and is as lightweight as possible is a very hard software design problem, especially when you are designing an open world game that must be devoid of loading screens or zones when you are traversing the world.

There are three main techniques game developers use to achieving what is commonly called world/level streaming:

- Multithreading 
- Partitioning/Chunking
- Levels of Detail (LODs)

World streaming is the process where we only have the necessary section of a world in memory at all times, discarding the rest. Normally, a radius around the player is loaded, loading new objects as they travel and unloading the ones they get far from. Multithreading is used to make the loading process happen in the background (possibly) over multiple frames and LODs are used to optimize the rendering of far away objects.

### Who am I?

My name is Santiago Bernardino. During my time at BUAS - Game Programming course, I tasked myself with implementing this concept for a self study block. For two months, I researched and created a basic showcase and application of world streaming. I delved more into the topics of Multithreading and Partitioning. I did not work with LODs, mostly due to them being especially complex and time consuming.

I will also log here my main implementing steps in C++ for creating an asynchronous asset manager and using it to create a demo that utilizes world streaming to render a large and heavy scene.

Part of my project is present in the LibStream library, my own resource management library written in C++ for this very project. Then, I used the Bee Engine for developing the demo, which is an in-house small engine for students that utilizes the entt library and OpenGL as a rendering backend.

## Multithreading - The Good, the Bad and the Ugly

### What is multi-threading?

Most modern machines nowadays possess more than one core. A core is a vital part of a computer that is capable of executing a set of instructions, one at a time, REALLY fast. With more than one core, a computer can execute instructions in parallel, independent of each other.

//Multithreaded cores image

At a first glance, this seems amazing. Having two cores, for example, can halve the amount of time my computer takes to execute a task, right?

Not quite. Turns out, it is impossible to evenly divide work between all the cores and a lot of tasks cannot be executed at the same time. A pretty smart guy derived Amdahl's law, which determines the maximum speedup from parallelizing code.

//Amdahl's Law

A lot of problems also cannot be effectively parallelized because of race conditions, a phenomenon where if one thread is writing to a memory address and another is reading from it at the same time, it is impossible to predict if the value read will be the old, new or a garbled mess. This causes non-determinism in a program, more commonly called Undefined Behaviour.

There are quite some fixes for this. Synchronization primitives and Atomics are programming constructs we can use to avoid out-of-order operation and I will explain the ones I used for this implementation. Even then, using these has limitations and parallelizing code can actually make it slower.

Using and writing threaded-code is a very challenging, because it forces you to think about thread-safety: if your code can cause a race condition and create bugs that are usually hard to detect and replicate. 

The main culprit of race conditions is sharing memory between threads. Luckily, loading an image file or parsing a text file is usually a self contained task and it is not hard to have 16 threads loading 16 files at a time. In fact, my main challenge was to introduce the newly loaded assets into the game.

In order to schedule these loading operations, I used a very common pattern used by high performance threaded applications.

### The Thread Pool

A thread pool is a concurrency model composed of a task queue and a set of worker threads. These worker threads are initialized all at once and live for the duration of the thread pool, bypassing a lot of overhead that comes to starting / releasing threads by only doing it once and keeping them alive. 

//Worker thread visualization

These worker threads are continuously sleeping until a task is added to the queue. One thread wakes up, takes the task, executes it and goes back to sleep or takes another task if the queue still has tasks. This simple model allows for setting up more complex systems, but I only needed a simple implementation:

```c++
//Thanks: https://stackoverflow.com/questions/15752659/thread-pooling-in-c11

class ThreadPoolScheduler : public ILoadingScheduler {
public:

    ThreadPoolScheduler(size_t worker_thread_count);
    ~ThreadPoolScheduler();

    void ScheduleLoadingOperation(std::function<void()> task) override;

private:
    void worker_loop();
    bool exit_code = false;
    
    std::mutex queue_mutex;       
    std::condition_variable worker_signal; 
    
    std::queue<std::function<void()>> tasks;
    
    std::vector<std::thread> workers;
};
```

Lets unpack this. First, this class inherits from ILoadingScheduler merely as a way for users of the library to be able to implement their own thread pools.

For this class we need a vector of all the threads and a queue for the tasks. Less evidently, we need synchronization primitives in the form of a std::mutex and std::condition_variable in order to prevent race conditions related to the threads accessing and taking tasks from the queue (remember, they are all sharing the queue so we need to synchronize it).

The constructor initializes all the threads.
```c++
ls::ThreadPoolScheduler::ThreadPoolScheduler(size_t worker_thread_count)
{
    for (uint32_t i = 0; i < worker_thread_count; ++i) {
        workers.emplace_back(
            std::thread(ThreadPoolScheduler::worker_loop, this)
        );
    }
}
```

The worker_loop() function that we pass to the thread's constructor is the main() function it will execute during it's lifetime.

```c++

void ls::ThreadPoolScheduler::worker_loop()
{
    while (true) {
        std::function<void()> task;

        {
            std::unique_lock<std::mutex> lock(queue_mutex); // Lock queue mutex
            worker_signal.wait(lock, [this] { return !tasks.empty() || exit_code; }); // Unlock mutex, wait for condition

            if (exit_code) { return; }

            task = std::move(tasks.front());
            tasks.pop();
        } // mutex is unlocked here

        task();
    }
}
```

This is quite a lot, so we will go by parts:

The entire function is an endless loop, where the only condition for returning is if exit_code is true.

First we declare an std::function which is the structure I use for storing a task (more on that later).

Second, I attempt to lock the queue mutex. A mutex is one of the synchronization primitives that is used to ensure two threads access and run the same tasks. A mutex is essentially a key that can only belong to one thread. When one thread locks the mutex, all the others that attempt to lock it will stall until it is available.

Then, we use the condition_variable to wait on the lock and a condition. A condition variable combines a lock with a condition, and every time you signal a condition variable, it will attempt to unlock the lock and check the condition, proceeding if it returns true.

In this case, we will signal one thread that work is available. That thread will then try to lock the mutex or wait for it and then check the condition. If false, the lock is unlocked and the thread goes back to waiting. Otherwise the lock is kept, we check for the exit condition and if not, we take a task from the pile.

The mutex is unlocked after that entire section and the thread can now execute the task by itself.

//https://thispointer.comc11-multithreading-part-7-condition-variables-explained/

Now, adding new tasks is as simple as:

```c++
 void ls::ThreadPoolScheduler::ScheduleLoadingOperation(std::function<void()> task) {
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        tasks.push(std::move(task));
    } //be careful to unlock the mutex before signaling
    worker_signal.notify_one();
 }
```

We lock the mutex since remember, this function will be called on the main thread and we don't want a worker messing with the pile while we add a task. We then add the task, unlock the mutex and then signal one of the threads. The signal then wakes up one worker, that will pick up the task and work on it.

When the ThreadPool is destroyed, we lock the queue mutex, set the exit_code, unlock it and then signal all the threads (this is why we check the exit code after waiting on the condition variable).

```c++
ls::ThreadPoolScheduler::~ThreadPoolScheduler()
{
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        exit_code = true;
    }
    worker_signal.notify_all();

    for (std::thread& active_thread : workers) {
        active_thread.join();
    }

    workers.clear();
}
```

Also don't forget to join all the threads as well.

## Resource Management - With great power, comes great responsibility.

