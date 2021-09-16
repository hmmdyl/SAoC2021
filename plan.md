# LWDR Milestone Plan

Author/Participant: Dylan Graham

Mentor: Adam D. Ruppe

## Milestone 1
Start: September 15, 2021

Submission Deadline: October 15, 2021

Milestone Report Deadline: October 22, 2021

**Tasks:**

1. Implementation of `static` and `shared static` constructors and destructors for classes and modules.
    * `shared static` constructors and destructors
        * `shared static` constructors and destructors must be explicitly run once. The `LWDR` helper class will implement a function to run this. Same goes for destruction. The function will also handle other once-only initialisation and deinitialisation. For example:
            ```D
            extern(C) void myMain()
            {
                LWDR.startRuntime; // runs the shared static constructors
                scope(exit) LWDR.stopRuntime; // runs the shared static destructors
                LWDR.registerCurrentThread; 
                scope(exit) LWDR.deregisterCurrentThread;
            }

            extern(C) void secondThreadEntryPoint()
            {
                // Don't call LWDR.startRuntime, as it should already be initialised.
                // If the runtime is already initialised, do nothing (log warning if optioned)
                LWDR.registerCurrentThread; 
                scope(exit) LWDR.deregisterCurrentThread;
            }
            ```
            The example assumes that all threads are created external to the D code, and simply call `myMain` and `secondThreadEntryPoint` respectively.
        * The `LWDR` helper class function will wrap an `extern(C)` function which allows external code to initialise and deinitialise the D runtime, if that is easier for the application.
        * A flag will be stored to keep track of if runtime has been initialised.
    * `static` constructors and destructors
        * `static` constructors and destructors are run once per thread and must be manually invoked. The `LWDR` helper class already has an function ([`registerCurrentThread()`](https://github.com/hmmdyl/LWDR/blob/master/source/lwdr/package.d#L58)) that does something similar, so this functionality will be appended to it. Destructions must also be manually invoked. For example:
            ```D
            extern(C) void myMain()
            {
                LWDR.startRuntime;
                scope(exit) LWDR.stopRuntime;
                LWDR.registerCurrentThread; // runs the static constructors for main thread
                scope(exit) LWDR.deregisterCurrentThread; // runs the static deconstructors for main thread
            }

            extern(C) void secondThreadEntryPoint()
            {
                LWDR.registerCurrentThread; // runs the static constructors for second thread
                scope(exit) LWDR.deregisterCurrentThread; // runs the static deconstructors for second thread thread
            }
            ```
            The example assumes that all threads are created external to the D code, and simply call `myMain` and `secondThreadEntryPoint` respectively.
        * Unlike `LWDR.startRuntime`/`stopRuntime`, `LWDR.registerCurrentThread`/`deregisterCurrentThread` won't be exposed to C - it should rely on D's `scope(..)` guards. If a compelling use-case arises, then this can be re-considered.
        * A flag will be stored in the thread control block if the runtime has been initialised, to prevent accidental re-initialisations or duplicate de-initialisations.
    * `static` and `shared static` constructors and destructors cannot be run if LWDR is not aware about them in the user's modules. DRuntime solves this by storing information about each module, including the locations of such constructors and destructors. LWDR should provide a similar implementation.
        * DRuntime's module information system performs things like [sorting constructors](https://github.com/dlang/druntime/blob/stable/src/rt/minfo.d#L44) and such, LWDR must reimplement this behaviour.
        * The compiler produces an array of `ModuleInfo` structs in the data segment, which contain the necessary information that LWDR needs. It can be accessed via the symbols [`__start_minfo`](https://github.com/dlang/dmd/blob/8fda137fe187c66a0b4d0fd7f4f7c6596e24145d/src/dmd/backend/elfobj.d#L3395) and [`__stop_minfo`](https://github.com/dlang/dmd/blob/8fda137fe187c66a0b4d0fd7f4f7c6596e24145d/src/dmd/backend/elfobj.d#L3400). This differs from DRuntime, which adds another layer of abstraction to handle shared libraries (sections, [`_d_dso_registry`](https://github.com/dlang/druntime/blob/stable/src/rt/sections_elf_shared.d#L413), etc). Because these are not available on microcontrollers, it can be omitted. 
    * Support for `static` and `shared static` constructors will be hidden behind a compile-time opt-in flag. 
    * Estimated duration: 2 weeks.
2. Implementation of `Object` monitor and synchronisation primitives.
    * Should be fairly straight-forward. It will mostly be reimplementing/porting the functionality found in [`monitor_.d`](https://github.com/dlang/druntime/blob/a17bb23b418405e1ce8e4a317651039758013f39/src/rt/monitor_.d) in DRuntime. 
    * Some [backend hooks](https://github.com/hmmdyl/LWDR/blob/master/source/rtoslink.d) will be exposed so that LWDR can take advantage of the existing RTOS mutex system.
    * Implement a basic mutex class like [`core.sync.mutex.Mutex`](https://github.com/dlang/druntime/blob/a17bb23b418405e1ce8e4a317651039758013f39/src/core/sync/mutex.d).
    * Support for `Object` monitor and synchronisation primitives will be hidden by a compile-time opt-in flag.
    * Estimated duration: 1.5 weeks.
3. Manual deallocation of delegates/closures
    * Delegates may store a local context on the heap, a struct pointer or a class reference. To store local context, D will allocate some space on the memory via the runtime hook `_d_allocmemory`. This function will need to be implemented. LWDR will need to store a list of these allocations.
    * LWDR will provide a function that accepts the local context pointer. If the pointer is found in its list of allocations, then the local context will be deallocated. If not, no operation, so that any delegate can passed and the user doesn't need to write code to discriminate between different delegate types.
    * This  will be opt-in via a compile-time flag.
    * Estimated duration: 0.5 weeks

## Milestone 2

Start: October 15, 2021

Submission Deadline: November 15, 2021

Milestone Report Deadline: November 22, 2021

**Tasks:**

1. Exception handling
    * Bindings to libunwind must be written. There are some differences between the compilers, this must accounted for. There are two different types ([LLVM](https://github.com/dlang/druntime/blob/a17bb23b418405e1ce8e4a317651039758013f39/src/core/internal/backtrace/libunwind.d), [GNU](https://github.com/gcc-mirror/gcc/tree/releases/gcc-11/libphobos/libdruntime/gcc/unwind)).
    * Exception handling on GDC already works. The code in [`deh.d`](https://github.com/gcc-mirror/gcc/blob/releases/gcc-11/libphobos/libdruntime/rt/deh.d) will need to be ported.
    * Exception handling with LDC has been more difficult. The code in [`dwarfeh.d`](https://github.com/ldc-developers/druntime/blob/ldc/src/rt/dwarfeh.d) will need to be ported. LDC also has [some assembly code](https://github.com/ldc-developers/druntime/blob/ldc/src/ldc/eh_asm.S) that will need to be implemented. It is unknown whether it will have to remain a separate build-step or can be implemented into LWDR D code.
    * LWDR will need to be able to select against the exception handling implementations depending on which compiler is being used.
    * The [`Throwable`, `Error` and `Exception`](https://github.com/hmmdyl/LWDR/blob/master/source/object.d#L439) classes have already been implemented.
    * Exception handling will be opt-in via a compile-time flag.
    * Expected duration: 3.5 weeks
2. Centralisation of configuration
    * Create a centralised module that manages which sections of code are included and excluded depending on compile-time flags. Currently, compile-time selection code is split and/or duplicated across modules. For example:
        ```D
        version(LWDR_DynamicArray) // what happens when we introduce other flags?
        {
            version = LWDR_INTERNAL_ti_next;
        }

        private class TypeInfoArrayGeneric(T, Base = T) 
        {
            version(LWDR_INTERNAL_ti_next)
            override @property inout(TypeInfo) next() inout
            { /* ... */ }
        }
        ```
        The above is duplicated amongst modules which have type information classes. This is because `LWDR_INTERNAL_ti_next` only exists within the module. What the centralised configuration module will do is create a series of enumerations set by the compile-time versions. Then, other modules will use `static if` on those enumerations.
    * Expected duration: 0.5 weeks.

## Milestone 3

Start: November 15, 2021

Submission Deadline: December 15, 2021

Milestone Report Deadline: December 22, 2021

**Tasks:**

1. Naive mark-and-sweep garbage collector implementation
    * Allocations via `new` will be routed to the GC.
    * The GC will interact with the RTOS to temporarily suspend other LWDR threads.
    * The GC will keep track of a list of allocations/pointers. 
    * Each allocation will contain a header, which contains data such as whether the destructors have been run, or if the object is reachable, and other such flags and the allocation size. The flags should be [`core.memory.GC.BlkAttr`](https://dlang.org/library/core/memory/gc.blk_attr.html).
    * Before marking starts, all reachable flags are set to false.
    * The mark algorithm will start at thread stacks and registers, TLS and static areas. It will traverse these, and if the scanned data contains a pointer that it can find in its allocation list, then it will mark the data as reachable. The mark algorithm will then run recursively on that data, until there are no more pointers to mark. 
    * Sweep will then happen, which simply walks the allocation list and calls destructors (if needed and if not already done on the object). It will then sweep again, this time freeing the unreachable data.
    * Allocations and deallocations are simply wrapping the RTOS heap memory functions. This GC implementation does not do any pooling or other advanced features. The implementation is slow (for example, the allocation list will ruin memory locality) and simple, but it is meant to be be usable and allow usage of some Phobos code. In the future, alternate garbage collectors can be offered. Unline DRuntime, such garbage collectors will have to be selected at compile time.
    * GC will need to implement some helper functions in [`core.memory.GC`](https://dlang.org/library/core/memory/gc.html). 
    * Garbage collection will be opt-in via a compile-time flag.
    * The programmer will be able to configure, at runtime, under which circumstances the GC runs, and these circumstances vary depending on power state.
        * LWDR will introduce two power states: low-power and full-power. GC collection policies can be for each, and the programmer can signal to LWDR which power state to use.
    * Estimated duration: 4 weeks.

## Milestone 4

Start: December 15, 2021

Submission Deadline: January 15, 2022

Milestone Report Deadline: January 22, 2022

**Tasks:**

1. Implement reference counting with a Bacon cycle collector
    * Introduce a new wrapper type, `RefCountCycleCollect` or `RcCc` for short. This is like a normal reference counted wrapper in D. However, 
    * A shared queue of pointers to `RefCountCycleCollect`s will need to be kept. Suspected (when the internal count drops to a non-zero value) reference cycles will be stored in this queue.
    * LWDR will expose an `extern(C)` function that an external, low-priority thread can enter. It will be non-returning. It will run the cycle collection concurrently when memory usage has passed a configured threshold, every `x` amount of time, or when the system is out of memory, according to user requirements.     
        * It will rely on thread notifications. For example, if a `new` operation is done, and the memory consumption has increased above a certain threshold, then the allocating thread will signal the collection thread to wake up (when possible) and collect. 
            * Thresholds and collection policies will be dependent on the power state, as discussed in milestone 3, task 1.
        * If the system is out of memory and the task cannot be switched, the allocating code will directly call the collection function. 
    * Another function that is responsible for running the cycle detection, destruction and freeing code will be exposed via `extern(C)`. If the user doesn't want to add a collection thread for above, then cycle collections will need to be manually invoked, or invoked during memory allocations (configurable).
    * This will be opt-in via a compile-time flag.
    * Estimated time: 2 weeks.
2. Associative arrays
    * Associative arrays will mostly be a port of DRuntime's [`aaA.d`](https://github.com/dlang/druntime/blob/master/src/rt/aaA.d), accounting for differences with LWDR's memory allocation functions. When a GC is present, associative arrays will allocate through that and rely on it for lifetime management. Otherwise, associate arrays will rely on manual memory managment.
        * To manually deallocate an associative array, LWDR will provide a helper function that will deallocate heap memory the associate array allocates for internal usage. It will not deallocate the keys or values, that is up to the programmer to manage.
    * This will be opt-in via a compile-time flag.
    * Estimated time: 1 week
3. Diagnostics
    * Basic statistics (memory usage, pressure, frequency, etc) will need to be gathered by the Bacon cycle collector and garbage collector. When either of these systems are invoked, they will update their statistics, and invoke the diagnostic code.
        * LWDR will need to be aware of the system tick, if it is available. This will be behind a compile-time flag. This is so that LWDR can report information such as how many allocations or collections have been made during a specific time period.
    * The diagnostic system will ultimately call an `extern(C)` function that the user implements. It will pass an information identification code, a corrective action identification code, and an `void*` for extra data. What this `void*` represents is dependent on upon the identification code. Ideally, the pointer will point to a `__gshared` structure that contains helpful information in regards to the code.
    * If the cycle collector queue fills up, the diagnostic function will run. Information will be passed as to which corrective actions were taken.
    * If the garbage collector allocation list fills up, the diagnostic function will run. Information will be passed as to which corrective actions were taken.
    * If the Bacon cycle collector or garbage collector are frequently allocating, the diagnostic function will run. Information in regards to the respective memory usage and pressure will be passed via the pointer.
    * If the Bacon cycle collector or garbage collector are frequently collection, the diagnostic function will run. Information about respective memory usage, pressure and collection performance will be passed via the pointer.
    * The diagnostic system will be opt-in via a compile-time flag.
    * Estimated time: 1 week.