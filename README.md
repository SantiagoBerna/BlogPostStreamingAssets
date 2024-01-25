## Introduction

In most AAA game titles nowadays, we have seen a rising trend to creating bigger worlds with more content and even more detail. The open-world genre has become very prevalent in the gaming industry during the last 10 years and nowadays game download sizes are getting to the 100GB+ mark (mind you the downloaded files are also heavily compressed). 

Therefore, if there is one thing that these games (and the engines they run on) must do right is Asset Management. Managing the lifetime of hundreds of assets while making sure that a game has an acceptable framerate and is as lightweight as possible is a very hard software design problem, especially when you are designing an open world game that must be devoid of loading screens or zones when you are traversing the world.

There are three main techniques game developers use to achieving what is commonly called world/level streaming:

- Multithreading 
- Partitioning/Chunking
- Levels of Detail (LODs)

World streaming is the process where we only have the necessary section of a world in memory at all times, discarding the rest. Normally, a radius around the player is loaded, loading new objects as they travel and unloading the ones they get far from. Multithreading is used to make the loading process happen in the background (possibly) over multiple frames and LODs are used to optimize the rendering of far away objects.

### Who am I?

My name is Santiago Bernardino. During my time at BUAS - Game Programming course, I tasked myself with implementing this concept for a self study block. For six weeks, I researched and created a basic showcase and application of world streaming. I delved more into the topics of Multithreading and Partitioning. I did not work with LODs, mostly due to them being especially complex and time consuming.

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

Creating a universal Asset Manager is very hard. There is a reason there aren't many libraries to handle resource management for you, since it is extremely dependent on your engine's capabilities and requirements.

I still tried to create a generalized solution that boiled resource management to it's simplest terms (and still ended up being reasonably complex).

### Part 1 - The game plan

The Libstream Library mainly works through the `Streaming Manager`. Let's walk through the requirements of this system:

- Needs to be able to accept a request to a resource and return some sort of handle to the caller.
- Must schedule and eventually collect the results from loading a resource on another thread without blocking.
- It must ensure that an asset is loaded only once in memory.
- Functionality for unloading unused Assets.

This looks like a small list, but deceptively so. Let us start with the simplest parts first.

### Part 2 - Storing the assets

The simplest way I found to implement this aspect is to store a ``HashMap`` with all of the resources indexed by a ``string`` (does not necessarily need to be a path, could also be a GUID).

- Takes care of making sure there is only one copy of a resource.
- Has the best lookup speed for an element: O(1)

Even though it seems like this is the best choice, it already limits us to using assets that can be uniquely identified through a string (which you can assume is true most of the time).

However, we will not be storing the resources themselves in this map. First of all, we can't store resources of the same type in the same container, unless we use an inheritance hierarchy and ``dynamic_cast`` for ensuring proper type safety. 

We also need to take in mind that this system is asynchronous and we need to keep track of which resources are loading, loaded and unloaded. If we request two times for the same resource, we cannot send two tasks and load the resource twice, since it would break our initial requirement of unique assets. To solve this, I used a double indirection setup.

//extra level of indirection

The map stores a pointer to a `Resource Entry`, which can contain a pointer to a loaded Resource.

For those familiar with shared pointers, the `Resource Entry` acts like a control block for the resource, controlling access to it and serving to mark if an asset is loaded or unloaded. 

It must contain an optional pointer to the actual resource it refers to, a mutex for controlling concurrent access to changing the state of the entry and an enum that indicates the many states a resource can be in. Translating to C++:

```c++
class ResourceEntry {
public:

	ResourceEntry(const std::string_view& path, std::type_index type)
		: origin_path(path), type(type) {}


	enum class ResourceState {
		UNLOADED,
		LOADING,
		READY,
		FAILED,
	};

private:
    //Retrieval
    std::shared_ptr<void> retrieve() const;

	mutable std::mutex mutex; // must be mutable to lock in getter
	ResourceState state = ResourceState::UNLOADED;

    //Resource Metadata
    std::type_index type;
	std::string origin_path;
	size_t memory_consumption = 0;

    //Optional Handle to the actual resource
	std::shared_ptr<void> resource = nullptr;
};
```

A keen eye will have noticed 3 things: 
- I have added an extra resource state for in case loading fails.
- I store extra metadata related to the asset. This comes in handy quite a lot of times, especially for debugging.
- The optional reference to a possibly loaded asset is in the form of `std::shared_ptr void`

Shared pointer void is a very smart use of type deletion to avoid template / inheritance hell. It acts like a shared pointer, so it deallocates itself once no more references remain, but we can store any data type inside it. To use it though, we must `std::static_pointer_cast` it back to the correct type. Storing the type_index of the stored handle allows for enforcing type-checking when we do the cast by comparing it with the type we want to cast to.

### Part 3 - Loaders and Resource Types

Another step I have implemented for allowing decoupling the library from the specifics of loading resources is to bind a function to the `StreamingManager` that will act upon all resources of the same type:

```c++
std::shared_ptr<void> LoadImageAsset(ls::RequestCommands& commands, const std::string_view& path) {
    //logic for loading an image from file
    return std::make_shared<Image>(width, height, std::move(image_data));
}

//...

StreamManager.RegisterAssetLoader<Image>(LoadImageAsset, "Image");
```

This is necessary for the `StreamingManager` to keep track of all the resource types and how to load them. This loading function must always have the same signature and takes a path and `RequestCommands` as parameters. I understand taking a path, but what are request commands?

Let us look at another example:

```c++
std::shared_ptr<void> bee::LoadMaterialAsset(ls::RequestCommands& resources, const std::string_view& path)
{
    // Get paths of all involved textures in the materials

    base_texture = resources.RequestDependency<Image>(base_path);
    normal_texture = resources.RequestDependency<Image>(normal_path);
    metallic_texture = resources.RequestDependency<Image>(metallic_path);

    //Return material asset
}
```

A material asset is essentially a collection of images that are arranged per their function in rendering. It is actually very useful to seperate both into different files, so usually a material asset will contain the paths to the textures it references (think of .mtl file for the OBJ format).

The `RequestCommands` are a parameter that allow the request of dependencies or references that a resource will need. More concretely, it simply queues up all the textures the material depends on when the material is requested for loading. 

It also ensures proper propagation of the loading type: if you request a synchronous load, all the dependencies will also be loaded synchronously.

### Part 4 - Request Logic

We are now ready to start asking our worker threads to load assets for us.
The constructor of `StreamingManager` looks as follows:

```c++
StreamingManager(ILoadingScheduler& scheduler, IDebugLogger* debug_logger);
```

The ``ILoadingScheduler`` is an interface I created for the threadpool, which can be expands, however the default one I have implemented in the project `ThreadPoolScheduler` is enough. It also accepts an interface to a debug logger which is optional for printing some debug messages.

To request an Asset we call `StreamingManager::RequestAsset<T>(path, load_type)` which returns a structure of type `ResourceHandle<T>`.
The resource handles returned essentially contain a shared reference to the resource entry of the asset, as well as functionality for upcasting it to the correct type:

```c++
class BasicResourceHandle {
protected:
    BasicResourceHandle(std::shared_ptr<ResourceEntry> entry_ref)
     : ref(entry_ref) {}
    
	virtual ~BasicResourceHandle() = default;

	std::shared_ptr<ResourceEntry> ref = nullptr;

    //Returns the resource pointer if the resource is loaded, nullptr otherwise
	std::shared_ptr<void> retrieve() const { return ref->retrieve(); }
};


template<typename T>
class ResourceHandle : public BasicResourceHandle {
public:
	ResourceHandle() : BasicResourceHandle(nullptr) {};

	ResourceHandle(std::shared_ptr<ResourceEntry> entry_ref) :
		BasicResourceHandle(entry_ref) {}

	std::shared_ptr<T> Retrieve() const {
		return std::static_pointer_cast<T>(BasicResourceHandle::retrieve());
	};
};
```

Then, the loading logic we require to recover assets we have loaded  and avoid loading an asset twice is as follows:

//Show flowchart

Note that, everytime an entry is accessed or modified, it must be locked to not cause any race conditions, since it is the point where the main thread and a loading thread will share memory. If you are implementing something similar, keep this in mind.

### Part 5 Unloading and freeing resources

