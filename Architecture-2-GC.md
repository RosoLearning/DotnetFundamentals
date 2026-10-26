# .NET Garbage Collector (GC) — Complete End-to-End Architecture Flowcharts

> Scope: .NET / CoreCLR, including allocation internals, managed heap organization, GC roots, reachability analysis, generations, write barriers, collection phases, compaction, finalization, background GC, and production diagnostics.
>
> **Diagram convention:** These are conceptual architecture flowcharts. Actual execution paths and phase ordering vary by runtime version, GC mode, object type, and collection strategy.

---

## Table of Contents

1. [Complete End-to-End .NET GC Architecture](#flowchart-1-complete-end-to-end-net-gc-architecture)
2. [Object Allocation — From `new` to Constructor Execution](#flowchart-2-object-allocation--from-new-to-constructor-execution)
3. [Managed Heap Organization](#flowchart-3-managed-heap-organization)
4. [What Triggers GC?](#flowchart-4-what-triggers-gc)
5. [GC Roots and Object Reachability](#flowchart-5-gc-roots-and-object-reachability)
6. [Generational GC and Promotion](#flowchart-6-generational-gc-and-promotion)
7. [Write Barrier and Remembered-Set Architecture](#flowchart-7-write-barrier-and-remembered-set-architecture)
8. [Complete GC Collection Phases](#flowchart-8-complete-gc-collection-phases)
9. [Sweep vs. Compact vs. Relocate vs. Promote](#flowchart-9-sweep-vs-compact-vs-relocate-vs-promote)
10. [Blocking GC vs. Background GC](#flowchart-10-blocking-gc-vs-background-gc)
11. [Finalization and Object Reclamation](#flowchart-11-finalization-and-object-reclamation)
12. [Workstation GC vs. Server GC](#flowchart-12-workstation-gc-vs-server-gc)
13. [Allocation Rate, Live Set and Memory Retention](#flowchart-13-allocation-rate-live-set-and-memory-retention)
14. [Production Troubleshooting — High CPU and Latency](#flowchart-14-production-troubleshooting--high-cpu-and-latency)
15. [Complete GC Decision Tree for Revision](#flowchart-15-complete-gc-decision-tree-for-revision)

---

# Flowchart 1: Complete End-to-End .NET GC Architecture

This is the master flowchart connecting application execution, object allocation, GC triggering, reachability analysis, memory reclamation, and application resumption.

```mermaid
flowchart TD
    A["C# Application Starts"] --> B["C# Compiler: Roslyn"]
    B --> C["IL / CIL + Metadata"]
    C --> D["CLR Loads and Executes Application"]
    D --> E["JIT Compiles Methods"]
    E --> F["Application Executes Managed Code"]

    F --> G["Application Requests Object"]
    G --> H["JIT-Generated Allocation Code"]
    H --> I{"Allocation Context Has Enough Space?"}

    I -->|Yes| J["Allocation Fast Path"]
    I -->|No| K["Allocation Slow Path"]

    J --> L["Reserve Space and Initialize Object"]
    K --> M["Request Suitable Allocation Space"]
    M --> N{"Can Allocation Proceed Without GC?"}

    N -->|Yes| L
    N -->|No| O["GC Trigger / Collection Decision"]

    O --> P["Select Collection Strategy"]
    P --> Q["Generation 0 Collection"]
    P --> R["Generation 1 Collection"]
    P --> S["Generation 2 / Full Collection"]

    Q --> T["Establish GC-Safe Execution State"]
    R --> T
    S --> T

    T --> U["Discover GC Roots"]
    U --> V["Trace Reachable Object Graph"]
    V --> W["Identify Live and Unreachable Objects"]
    W --> X["Plan Reclamation"]

    X --> Y["Sweep Dead-Object Space"]
    X --> Z["Compact and Relocate Where Required"]

    Y --> AA["Update Heap and Free-Space State"]
    Z --> AB["Update References and Heap State"]

    AA --> AC["Promote Survivors Where Applicable"]
    AB --> AC

    AC --> AD["Update Allocation Budgets and GC Metadata"]
    AD --> AE["Resume Suspended Managed Threads"]
    AE --> AF["Continue Application Execution"]

    L --> AG["Execute Constructor"]
    AG --> AH["Return Object Reference"]
    AH --> AF

    AF --> AI{"More Allocation or Memory Pressure?"}
    AI -->|Yes| G
    AI -->|No| AJ["Application Continues / Terminates"]
```

### How to read the master flowchart

1. The compiler generates IL; the CLR and JIT execute the program.
2. The allocation path attempts to allocate from an existing allocation context.
3. If the fast path cannot satisfy the request, the slow path obtains suitable space or participates in a GC decision.
4. When collection occurs, the runtime identifies live objects and determines which memory can be reclaimed.
5. Depending on the collection strategy, the GC sweeps, compacts, or performs a combination of operations.
6. The runtime updates heap state and resumes execution after any required pauses.

A collection does not necessarily compact the heap, and allocation-context exhaustion does not necessarily trigger a collection.

---

# Flowchart 2: Object Allocation — From `new` to Constructor Execution

Consider:

```csharp
var customer = new Customer(42);
```

```mermaid
flowchart TD
    A["Application Executes new Customer(42)"] --> B["IL Represents Object Construction"]
    B --> C["JIT Generates Runtime-Specific Code"]
    C --> D["Determine Required Object Size"]
    D --> E["Account for Object Layout and Alignment"]

    E --> F{"Ordinary Small Managed Object?"}

    F -->|Yes| G["Use Small-Object Allocation Path"]
    F -->|No| H["Use Appropriate Special Allocation Path"]

    G --> I["Read Thread Allocation Context"]
    I --> J["Read Allocation Pointer and Limit"]
    J --> K{"Enough Space in Current Context?"}

    K -->|Yes| L["Reserve Space by Advancing Pointer"]
    K -->|No| M["Enter Allocation Slow Path"]

    M --> N["Request New Allocation Context or Heap Space"]
    N --> O{"Can Space Be Obtained?"}

    O -->|Yes| L
    O -->|No| P["Evaluate GC or Other Allocation Handling"]
    P --> Q["Perform Required Collection or Memory Management"]
    Q --> R{"Allocation Can Now Proceed?"}

    R -->|Yes| L
    R -->|No| S["Allocation Fails if Runtime Cannot Satisfy Request"]

    H --> T["Handle Large Object or Special Allocation"]
    T --> U["Obtain and Initialize Suitable Memory"]

    L --> V["Initialize Object Representation"]
    U --> V

    V --> W["Establish Valid Object Header and Default Field Values"]
    W --> X["Execute Constructor"]
    X --> Y{"Constructor Completes?"}

    Y -->|Yes| Z["Return Reference to Caller"]
    Y -->|No| AA["Propagate Exception"]
    AA --> AB["Object May Become Garbage if No Live Reference Remains"]
```

### Allocation internals

- The allocator generally obtains space from memory already managed by the CLR rather than requesting memory from the operating system for every object.
- The allocation fast path is often implemented using a pointer and limit check.
- Per-thread allocation contexts reduce contention between threads allocating small objects.
- The slow path can obtain another allocation context, request more space, or trigger a collection when necessary.
- A newly allocated object has valid runtime metadata and default field values before its constructor's logic executes.
- If a constructor throws, the object is not necessarily reclaimed immediately; it becomes collectible if it is no longer reachable.

---

# Flowchart 3: Managed Heap Organization

```mermaid
flowchart TD
    A["CLR Managed Memory Management"] --> B["Managed Heap"]

    B --> C["Small Object Heap - SOH"]
    B --> D["Large Object Heap - LOH"]
    B --> E["Pinned Object Heap - POH"]

    C --> F["Generation 0"]
    C --> G["Generation 1"]
    C --> H["Generation 2"]

    F --> I["Newly Allocated Small Objects"]
    G --> J["Objects Surviving Earlier Collections"]
    H --> K["Longer-Lived Objects"]

    D --> L["Large Managed Objects"]
    L --> M["Typically Collected During Gen 2 Collections"]

    E --> N["Certain Explicitly Pinned Allocations"]
    N --> O["Addresses Remain Stable While Pinned"]

    F --> P["Ephemeral Collection Focus"]
    G --> P
    P --> Q["Gen 0 or Gen 1 Collection"]

    H --> R["Full Collection Scope"]
    R --> S["Considers Gen 0, Gen 1, Gen 2 and Relevant Other Heap Structures"]
```

### Important distinctions

- Generations 0, 1, and 2 apply to the ordinary generational organization of the SOH.
- The LOH is generally collected as part of generation 2 collections.
- The POH is used for certain pinned allocations and is not a fourth ordinary generation.
- The commonly used LOH allocation threshold is 85,000 bytes; actual classification depends on the runtime's object-size calculation.
- The GC may retain reclaimed heap memory for future managed allocations instead of returning it immediately to the operating system.

---

# Flowchart 4: What Triggers GC?

```mermaid
flowchart TD
    A["Application Executes"] --> B["Allocation or Memory-Pressure Event"]

    B --> C{"What Caused the Event?"}

    C --> D["Generation Allocation Budget Reached"]
    C --> E["Allocation Cannot Be Satisfied Efficiently"]
    C --> F["Operating-System Memory Pressure"]
    C --> G["Runtime GC Heuristics"]
    C --> H["Explicit GC.Collect Request"]
    C --> I["Other Runtime or Host Conditions"]

    D --> J["GC Evaluates Collection Need"]
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J

    J --> K{"Which Collection Strategy Is Appropriate?"}

    K -->|Young Objects Likely Sufficient| L["Gen 0 Collection"]
    K -->|More Ephemeral Reclamation Needed| M["Gen 1 Collection"]
    K -->|Full-Heap Reclamation Needed| N["Gen 2 / Full Collection"]

    L --> O["Execute Selected Collection"]
    M --> O
    N --> O

    O --> P["Reclaim Eligible Memory"]
    P --> Q["Update Heap State and Future Budgets"]
    Q --> R["Resume Normal Execution"]
```

### Allocation budgets

Allocation budgets are dynamic. They depend on runtime heuristics, previous collection outcomes, survival rates, memory pressure, and other factors.

There is no single universal Gen 0 size or fixed allocation threshold that applies to every .NET application.

`GC.Collect()` can request a collection, but it should not ordinarily be used as a routine optimization strategy.

---

# Flowchart 5: GC Roots and Object Reachability

The GC determines liveness through reachability rather than reference counting.

```mermaid
flowchart TD
    A["GC Collection Begins"] --> B["Establish Required GC-Safe State"]
    B --> C["Discover Relevant Roots"]

    C --> D["Live Stack and Register References"]
    C --> E["Static Field References"]
    C --> F["GC Handles"]
    C --> G["Other Runtime-Maintained Roots"]

    D --> H["Initialize Reachability Traversal"]
    E --> H
    F --> H
    G --> H

    H --> I["Mark Objects Directly Referenced by Roots"]
    I --> J["Inspect References Inside Marked Objects"]
    J --> K{"Unprocessed Referenced Objects Remain?"}

    K -->|Yes| L["Mark Newly Discovered Objects"]
    L --> J

    K -->|No| M["Reachability Traversal Complete"]
    M --> N["Reachable Objects Are Live"]
    M --> O["Unreachable Objects Are Candidates for Reclamation"]

    O --> P{"Does Finalization or Other Runtime Handling Apply?"}
    P -->|No| Q["Eligible for Normal Reclamation"]
    P -->|Yes| R["Apply Additional Lifetime Handling"]

    N --> S["Preserve Live Objects"]
    Q --> T["Reclaim Through Collection Strategy"]
    R --> U["Schedule or Preserve as Required"]
```

### Example object graph

```text
Live Root
   |
   v
Customer
   |
   +------> Address
   |           |
   |           v
   |         Country
   |
   +------> Preferences

Unreachable Object A ------> Unreachable Object B
         ^                           |
         |___________________________|
```

The `Customer`, `Address`, `Country`, and `Preferences` objects are live if reachable from a live root.

The circular reference between `Object A` and `Object B` does not keep them alive if neither can be reached from any GC root.

### JIT liveness

A local variable's source-code scope is not necessarily its exact GC lifetime. The JIT may determine that a reference is no longer needed before the method ends.

Statics, long-lived collections, closures, event subscriptions, and GC handles can extend object lifetimes.

---

# Flowchart 6: Generational GC and Promotion

```mermaid
flowchart TD
    A["New Small Object Allocated"] --> B["Generation 0"]

    B --> C["Gen 0 Collection"]
    C --> D{"Object Reachable?"}

    D -->|No| E["Object Becomes Eligible for Reclamation"]
    D -->|Yes| F["Preserve Object"]

    F --> G["Promote to Generation 1 Where Applicable"]
    G --> H["Future Gen 0 or Gen 1 Collection"]

    H --> I{"Object Reachable During Collection?"}

    I -->|No| J["Object Becomes Eligible for Reclamation"]
    I -->|Yes| K["Promote to Generation 2 Where Applicable"]

    K --> L["Future Generation 2 Collection"]
    L --> M{"Object Reachable?"}

    M -->|No| N["Object Becomes Eligible for Reclamation"]
    M -->|Yes| O["Remain Live in Generation 2"]

    E --> P["Reclaim Through Applicable Heap Strategy"]
    J --> P
    N --> P
```

### Collection scope

```text
Collect Gen 0:
    Gen 0 is collected.

Collect Gen 1:
    Gen 0 and Gen 1 are collected.

Collect Gen 2:
    Gen 0, Gen 1 and Gen 2 are considered.
    LOH and relevant other heap structures are also considered.
```

Promotion is a logical generation change. Physical relocation may or may not occur during the same collection.

---

# Flowchart 7: Write Barrier and Remembered-Set Architecture

An older object can reference a younger object. The GC must account for this when collecting only younger generations.

```mermaid
flowchart TD
    A["Long-Lived Object Exists in Older Generation"] --> B["Application Updates Reference Field"]
    B --> C["Older Object Now References Younger Object"]

    C --> D["Managed Reference Write Executes"]
    D --> E["Write Barrier Records Relevant Reference Update"]
    E --> F["Remembered Information / Card Table Updated"]

    F --> G["Young-Generation Collection Begins"]
    G --> H["GC Discovers Ordinary Roots"]
    H --> I["GC Examines Relevant Remembered Information"]
    I --> J["Find Older Objects That May Reference Young Objects"]

    J --> K["Trace Relevant Younger Objects"]
    K --> L["Preserve Reachable Younger Objects"]
    K --> M["Identify Unreachable Younger Objects"]

    L --> N["Continue Collection"]
    M --> O["Reclaim Eligible Memory"]
```

### Why the write barrier matters

Without remembered information, a young-generation collection could miss a young object referenced only by an older object.

The write barrier helps preserve correctness while allowing the collector to avoid scanning every older object during every young-generation collection.

The card table is one implementation mechanism. It records relevant memory regions rather than acting as a direct inventory of all live references.

---

# Flowchart 8: Complete GC Collection Phases

This is the core collector workflow.

```mermaid
flowchart TD
    A["Collection Requested"] --> B["Determine Collection Scope"]
    B --> C["Select GC Mode and Collection Strategy"]

    C --> D["Establish GC-Safe Execution State"]
    D --> E["Suspend Managed Threads Where Required"]

    E --> F["Discover Roots"]
    F --> G["Trace Reachable Object Graph"]
    G --> H["Mark Live Objects"]

    H --> I["Identify Reclaimable Objects"]
    I --> J["Evaluate Heap Layout and Reclamation Strategy"]

    J --> K{"Which Strategy Is Appropriate?"}

    K -->|Sweep| L["Identify Dead Regions and Record Free Space"]
    K -->|Compact| M["Plan Locations for Surviving Objects"]
    K -->|Mixed Strategy| N["Apply Appropriate Strategy to Different Heap Regions"]

    M --> O["Move Surviving Objects as Required"]
    N --> P["Move Objects Only Where the Strategy Requires"]

    O --> Q["Update Managed References"]
    P --> Q

    L --> R["Reclaim Dead-Object Space"]
    Q --> R

    R --> S["Promote Surviving Objects Where Applicable"]
    S --> T["Update Heap Metadata and Allocation State"]
    T --> U["Update Future GC Budgets"]
    U --> V["Resume Suspended Threads"]
    V --> W["Application Continues"]
```

### Phase explanations

1. **Collection request:** The allocator or runtime identifies a need for collection.
2. **Collection scope:** The runtime determines which generations and heap structures must be considered.
3. **Safe execution:** Managed threads are coordinated so the GC can correctly inspect references and update memory.
4. **Root discovery and marking:** Reachability determines which objects are live.
5. **Reclamation planning:** The GC determines how to recover space.
6. **Sweep or compact:** Dead memory is reclaimed, and surviving objects may be moved.
7. **Reference correction:** Managed references are updated if objects relocate.
8. **Promotion and bookkeeping:** Surviving objects and heap structures are updated.
9. **Resume:** Suspended application threads continue execution.

Not every collection performs every operation in precisely this sequence. Background collection, LOH handling, and different heap segments can involve different phase schedules.

---

# Flowchart 9: Sweep vs. Compact vs. Relocate vs. Promote

```mermaid
flowchart TD
    A["Live and Dead Objects Identified"] --> B["Choose Heap Reclamation Strategy"]

    B --> C["Sweep"]
    B --> D["Compact"]
    B --> E["Mixed Strategy"]

    C --> F["Reclaim Dead Regions"]
    F --> G["Record Reusable Free Space"]
    G --> H["Surviving Objects May Stay at Existing Addresses"]

    D --> I["Plan New Locations for Live Objects"]
    I --> J["Move Surviving Objects"]
    J --> K["Correct Managed References"]
    K --> L["Create Larger Contiguous Free Region"]

    E --> M["Apply Different Strategies to Different Regions"]
    M --> N["Move Objects Only Where Needed"]
    N --> O["Update References Where Objects Moved"]

    H --> P["Update Heap State"]
    L --> P
    O --> P

    P --> Q["Promotion Decision"]
    Q --> R["Update Logical Generation of Survivors Where Applicable"]
```

### Visual example

Before compaction:

```text
+--------------------------------------------------+
| Live A | Dead B | Live C | Dead D | Live E       |
+--------------------------------------------------+
```

After compaction:

```text
+--------------------------------------------------+
| Live A | Live C | Live E |       Free Space      |
+--------------------------------------------------+
```

Compaction is useful for reducing fragmentation, but moving objects requires work. Pinning and heap-specific constraints can limit compaction.

---

# Flowchart 10: Blocking GC vs. Background GC

```mermaid
flowchart TD
    A["GC Collection Needed"] --> B{"Collection Mode / Type?"}

    B -->|Blocking Collection| C["Coordinate Managed Threads"]
    C --> D["Suspend Threads for Required GC Phases"]
    D --> E["Perform Required Collection Work"]
    E --> F["Update Heap and References"]
    F --> G["Resume Threads"]

    B -->|Background Gen 2 Collection| H["Initial Coordination and Required Pause"]
    H --> I["Start Background Collection Work"]
    I --> J["Resume Application Threads When Permitted"]

    J --> K["Concurrent Background Marking"]
    K --> L{"Foreground Ephemeral Collection Needed?"}

    L -->|Yes| M["Coordinate Foreground Gen 0 / Gen 1 Collection"]
    M --> N["Perform Required Foreground Work"]
    N --> O["Resume Background Work Where Applicable"]
    O --> K

    L -->|No| P["Continue Background Collection"]

    P --> Q["Final Coordination and Required Pause"]
    Q --> R["Complete Remaining Collection Work"]
    R --> S["Update Heap State"]
    S --> T["Resume Threads"]
```

### Important interpretation

- Blocking GC can stop managed application execution during required collection phases.
- Background GC allows significant generation 2 collection work to overlap with application execution.
- Background GC still requires pauses for specific phases.
- A foreground Gen 0 or Gen 1 collection may occur while a background Gen 2 collection is in progress.
- Actual phase ordering depends on the runtime implementation and the state of the heap.

---

# Flowchart 11: Finalization and Object Reclamation

```mermaid
flowchart TD
    A["Object Allocated"] --> B["Object Used by Application"]
    B --> C["Object Becomes Unreachable"]
    C --> D["GC Discovers Object During Collection"]

    D --> E{"Finalization Required?"}

    E -->|No| F["Object Eligible for Normal Reclamation"]
    E -->|Yes| G["Preserve and Schedule for Finalization"]

    G --> H["Finalizer Infrastructure Processes Object"]
    H --> I["Finalizer Executes"]
    I --> J{"Object Resurrected?"}

    J -->|Yes| K["Object Becomes Reachable Again"]
    K --> L["Object May Survive Further Collections"]
    L --> M["If It Becomes Unreachable Again, Reevaluate Lifetime"]

    J -->|No| N["Object Can Become Eligible for Later Reclamation"]
    M --> C

    F --> O["Reclaim Managed Memory"]
    N --> O
```

### Cleanup architecture

```text
Application
    |
    +-- using / Dispose
    |       |
    |       v
    |   Deterministic Cleanup
    |
    +-- SafeHandle
    |       |
    |       v
    |   Reliable Native-Resource Lifetime Management
    |
    +-- Finalizer
            |
            v
        Fallback Cleanup
        with Nondeterministic Timing
```

A finalizer is not a substitute for deterministic disposal. Finalization can delay memory reclamation, and its execution time is not predictable.

---

# Flowchart 12: Workstation GC vs. Server GC

```mermaid
flowchart TD
    A[".NET Application Configuration"] --> B{"GC Mode"}

    B -->|Workstation GC| C["Workstation-Oriented GC Strategy"]
    B -->|Server GC| D["Server-Oriented GC Strategy"]

    C --> E["Typically Uses a Workstation-Oriented Heap and Thread Model"]
    E --> F["Evaluate Interactive Latency and Resource Usage"]

    D --> G["Typically Uses Multiple GC Heaps and GC Threads"]
    G --> H["Evaluate Parallel Collection and Throughput"]

    F --> I["Measure Application Under Representative Load"]
    H --> I

    I --> J["Inspect CPU, Memory, Allocation Rate and Pause Behavior"]
    J --> K["Choose Mode Based on Workload and Resource Constraints"]
```

### Architecture trade-offs

| Consideration | Workstation GC | Server GC |
|---|---|---|
| Primary orientation | Client and interactive workloads | Server throughput and parallel workloads |
| Parallelism | Workstation-oriented collection strategy | Typically parallel across multiple GC heaps |
| CPU use | Depends on workload and configuration | Can use more CPU resources during collection |
| Memory use | Depends on workload and configuration | Can have additional heap and thread overhead |
| Best choice | Must be measured | Must be measured |

Neither mode is universally best. Consider CPU quotas, container memory limits, concurrency, and latency requirements.

---

# Flowchart 13: Allocation Rate, Live Set and Memory Retention

```mermaid
flowchart TD
    A["Application Under Load"] --> B["Collect Runtime Counters and Traces"]

    B --> C["Measure Allocation Rate"]
    B --> D["Measure Managed Heap and Live-Set Behavior"]
    B --> E["Measure GC Counts and GC Time"]
    B --> F["Measure Process Memory"]

    C --> G{"Allocation Rate High?"}
    D --> H{"Live Set Growing?"}
    E --> I{"GC Contributes Materially to Latency or CPU?"}
    F --> J{"Process Memory Exceeds Managed Heap Significantly?"}

    G -->|Yes| K["Inspect Temporary Objects and Hot Allocation Paths"]
    G -->|No| L["Investigate Other Sources of Cost"]

    H -->|Yes| M["Inspect Retained Object Graphs and Heap Snapshots"]
    H -->|No| N["Assess Whether Short-Lived Allocation Is Expected"]

    I -->|Yes| O["Correlate GC Events with Latency and CPU"]
    I -->|No| P["Investigate Other Bottlenecks"]

    J -->|Yes| Q["Investigate Native Memory, Stacks, Mappings and Other Process Memory"]
    J -->|No| R["Continue Managed Heap Analysis"]

    K --> S["Apply Targeted Optimization"]
    M --> S
    N --> T["Validate Under Representative Load"]
    O --> S
    P --> T
    Q --> T
    R --> T
    L --> T

    S --> T
    T --> U["Compare Throughput, Tail Latency, Allocation and Memory"]
```

### Metric interpretation

- **High allocation rate:** The application creates objects quickly.
- **High live set:** Many objects remain reachable and must be preserved.
- **High retained memory:** Long-lived references keep objects alive.
- **High GC time:** Collection work is consuming a meaningful share of execution time.
- **High process memory:** May include managed heap memory, native allocations, stacks, and other operating-system-managed memory.

High allocation rate alone does not prove a memory leak. Growing retention and an unexpectedly increasing live set deserve separate investigation.

---

# Flowchart 14: Production Troubleshooting — High CPU and Latency

```mermaid
flowchart TD
    A["API Latency or CPU Regression"] --> B["Capture Baseline Metrics"]
    B --> C["Measure Throughput, p95 / p99 Latency and CPU"]
    C --> D["Inspect System.Runtime Counters"]

    D --> E["Allocation Rate"]
    D --> F["GC Collection Counts"]
    D --> G["GC Time and Heap Size"]
    D --> H["Process Memory"]

    E --> I{"High Allocation Rate?"}
    F --> J{"Frequent or Expensive Collections?"}
    G --> K{"Live Set or Heap Size Growing?"}
    H --> L{"Managed Heap Explains Process Memory?"}

    I -->|Yes| M["Capture Allocation Trace"]
    I -->|No| N["Investigate Other CPU and Latency Causes"]

    J -->|Yes| O["Correlate GC Events with Latency Spikes"]
    J -->|No| P["Inspect Other Runtime and Application Bottlenecks"]

    K -->|Yes| Q["Capture Heap Snapshot and Inspect Retention"]
    K -->|No| R["Check Whether Allocation Pressure Is Transient"]

    L -->|No| S["Investigate Native Memory, Stacks and Mappings"]
    L -->|Yes| T["Continue Managed Heap Investigation"]

    M --> U["Identify Hot Allocation Sites"]
    O --> V["Identify Collection Type and Pause Contribution"]
    Q --> W["Identify Long-Lived Reference Chains"]

    U --> X["Apply Targeted Changes"]
    V --> X
    W --> X
    N --> Y["Profile CPU, Locks, I/O and Thread Pool"]
    P --> Y
    R --> Z["Validate Under Representative Load"]
    S --> Z
    T --> Z
    X --> Z
    Y --> Z

    Z --> AA["Compare Baseline and New Results"]
```

### Example optimization choices

| Finding | Potential action |
|---|---|
| Many temporary strings | Reduce repeated concatenation; measure alternatives such as `StringBuilder`. |
| Repeated temporary collections | Reduce unnecessary intermediates or reuse storage where safe. |
| Large arrays repeatedly allocated | Stream data or use appropriate buffer pooling. |
| Cache grows without bounds | Apply explicit cache capacity and expiration policies. |
| Event subscriptions retain objects | Unsubscribe or redesign ownership. |
| High native memory | Profile native allocations and interop lifetimes. |
| GC is not a major bottleneck | Investigate CPU, database, networking, locking, and thread-pool behavior. |

---

# Flowchart 15: Complete GC Decision Tree for Revision

Use this final flowchart to revise the whole architecture quickly.

```mermaid
flowchart TD
    A["Application Creates Objects"] --> B{"Fast Allocation Possible?"}

    B -->|Yes| C["Allocate and Initialize Object"]
    B -->|No| D["Slow Allocation Path"]

    D --> E{"Need GC or Additional Heap Space?"}
    E -->|Additional Space| F["Obtain Suitable Allocation Space"]
    E -->|GC Required| G["Select Collection Scope"]

    F --> C
    G --> H["Coordinate Managed Threads as Required"]
    H --> I["Discover Roots and Trace Object Graph"]

    I --> J{"Object Reachable?"}

    J -->|Yes| K["Preserve Object"]
    J -->|No| L{"Finalization or Other Handling Required?"}

    L -->|Yes| M["Apply Additional Lifetime Handling"]
    L -->|No| N["Mark Object Memory as Reclaimable"]

    M --> O["Object May Require Later Reclamation"]
    K --> P["Sweep / Compact According to Heap Strategy"]
    N --> P
    O --> P

    P --> Q["Relocate Objects and Update References Where Required"]
    Q --> R["Promote Surviving Objects Where Applicable"]
    R --> S["Update Heap Metadata and Budgets"]
    S --> T["Resume Application Execution"]

    C --> U["Constructor Executes"]
    U --> V["Application Uses Object"]
    V --> W["Object Eventually Becomes Unreachable or Remains Live"]
    W --> A

    T --> A
```

---

**Final mental model:** Allocation creates objects efficiently; roots and tracing determine liveness; generational policies limit collection work; write barriers preserve cross-generation correctness; sweep and compaction recover usable memory; promotion tracks survivors; finalization can delay reclamation; and runtime diagnostics tell you whether allocation, retention, GC pauses, or non-managed memory is causing a production problem.
