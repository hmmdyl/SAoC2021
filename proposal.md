# Project Proposal
Dylan Graham on Saturday the 7th of August, 2021  

The [Light Weight D Runtime](https://github.com/hmmdyl/LWDR), abbreviated LWDR, is a ground-up implementation of a small D runtime suitable for embedded, Internet of Things (IoT), Real Time Operating System (RTOS) and, potentially, webassembly applications. 

D has a potentially bright future in these work areas, owing to its unique mixture of low and high level paradigms and focus on memory and logical safety. The existing D runtime is much too large and dependent on the features and comforts afforded by a modern desktop operating system. D as BetterC, by contrast, is a very stripped-down, barebones subset of the language. There is no middle ground, which LWDR intends to fill, akin to nanoFramework for .NET or TinyGo.

LWDR works by implementing the supporting software for various D features and presenting a limited number of basic function hooks to the user for implementation, also increasing the portability of the runtime. Such hooks can be for memory allocation (`malloc`/`free`), panics, thread control, etc. 

The long-term plan for LWDR is to introduce full, idiomatic D to embedded environments, yet responding to the constraints they pose. Given that working memory is very limited, memory management strategies for dynamic allocations should emphasize both safety and an ability to avoid leaks. 

I, Dylan Graham, have selected this project due to my ongoing interest in utilising D in automotive and embedded environments, desire to use more of D's features and developing interest in programming language design. LWDR is my own personal project that I believe will be beneficial to other D users who wish utilise "full" D in constrained environments, as there does not yet exist a satisfactory runtime.

Within the time constraints of the Symmetry Autumn of Code (SAoC), a sizeable portion of D's features that are currently not supported can be implemented in LWDR, however, new features, language updates and continued maintenance will require indefinite support. 

## Project status
As of time of writing.

LWDR supports:

1. Classes

2. Heap allocated structs

3. Assertions

4. Invariants, contracts

5. Basic Runtime Type Information (RTTI)

6. Interfaces

7. Static arrays

8. Class inheritance

9. Dynamic arrays (opt-in)

10. Thread local storage (opt-in)

11. Primitive memory tracking (opt-in)

## Goals
1. Support for `static` and `shared static` constructors and destructors
    - This will require support for "module info".
2. Support for `Object` monitors and synchronisation primitives.
    - D-style lock-based programming is impossible with these features.
3. Support for exception handling
    - D has no alternative error handling, so it will be supported.
4. Introduction of a conservative mark-and-sweep garbage collector, similar to the existing one in a full D implementation
    - GC is standard in D, so it will be supported.
    - For some projects a garbage collector is acceptable. The GC will be non-deterministic. It will be invoked, if needed, during calls to `new` or when the process runs low on memory. 
    - The GC should be flexible. For example, thresholds that determine when the GC will run can be set at runtime, and it should be able to react to various power states (if the processor goes into a low-power state, the GC should be used in emergency, if not, it can be more aggressive).
    - The GC will suspend all threads. Destructors will be called from the thread the GC was invoked on. This is in line with current D practice.
5. Reference counted objects (with lifetime cycles handled by a [Bacon collector](https://researcher.watson.ibm.com/researcher/files/us-bacon/Bacon01Concurrent.pdf))
    - This will be an alternative memory management strategy. All objects will need to be encapsulated in a `ReferenceCount` structure. References that are suspected to be cycles will be placed in a dedicated queue. The cycle collector, operating in its own low-priority thread, will concurrently iterate over the queue and free any detected reference cycles. 
    - Destructors will be called from any thread where the final dereference happens, or from the cycle collector thread.
    - The collector can run every `x` amount of system ticks, when its queue is full or when the system is low on memory.
    - In bare metal or no-threading environments, the cycle collector should be manually invoked.
6. Associative array implementation
    - This will require support for object hashing
7. Centralisation of hardware and runtime configuration
    - Hardware-specific enumerations and runtime feature controls are split across various files. This makes it hard to keep track of, potentially introduces bugs and increases the complexity of porting. For sake of brevity, these must be centralised into a single module or package.
8. Support for manual deallocation of delegates/closures
9. In debug builds, the runtime should be able to emit warnings and diagnostic data
    - This should not rely on strings due to memory constraints
    - For example, if the cycle collector queue runs out of space and it wants to allocate a second queue, this should emit a runtime warning, but keep the program running. If the GC is running often within a given time period, a warning and some data should be emitted.

## Proposed Timeline
Milestone 1:

- (Goal 1) `static` and `shared static` constructors and destructors

- (Goal 2) `Object` monitors and synchronisation primitives.

- (Goal 8) Manual deallocation of delegates/closures

Milestone 2:

- (Goal 3) Exception handling
    - Previous attempts have been problematic and time consuming.

- (Goal 7) Centralisation of configuration

Milestone 3:

- (Goal 4) Garbage collector

- (Goal 5) (Potentially) Bacon cycle collector

Milestone 4:

- (Goal 5) Bacon cycle collector

- (Goal 6) Asssociative arrays

- (Goal 9) Runtime warnings and diagnostics
