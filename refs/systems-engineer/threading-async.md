# Threading & Async Safety — UE5 Reference

## The Fundamental Rule

**All `UObject` reads and writes must happen on the game thread.** UE5's garbage collector, actor lifecycle, and reflection system are not thread-safe. Accessing any `UObject*` from a background thread — even a read — is undefined behavior that produces intermittent crashes that are nearly impossible to reproduce.

This applies without exception to: `AActor`, `UActorComponent`, `UGameInstance`, subsystems, GAS attributes, and any class derived from `UObject`.

---

## Marshaling Back to the Game Thread

When an async operation completes on a background thread, marshal the result back before touching any UObject:

```cpp
// WRONG — callback fires on a background thread; UObject access is UB
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this]()
{
    // ... do expensive work ...
    HealthComponent->SetHealth(NewHealth); // UB: HealthComponent is a UObject
});

// CORRECT — expensive work on background, UObject mutation on game thread
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, WeakThis = TWeakObjectPtr<AMyActor>(this)]()
{
    float ComputedValue = ExpensiveComputation(); // safe on background

    AsyncTask(ENamedThreads::GameThread, [WeakThis, ComputedValue]()
    {
        if (AMyActor* Self = WeakThis.Get()) // safe — we're on game thread now
        {
            Self->HealthComponent->SetHealth(ComputedValue);
        }
    });
});
```

**Always capture `this` as `TWeakObjectPtr`** in lambdas that cross thread boundaries — the actor may be destroyed before the game thread callback fires.

---

## Shared Non-UObject State — Protecting With Locks

For non-UObject data accessed from multiple threads (e.g., a work queue, a results buffer, a counter):

```cpp
// Thread-safe work queue example
class FMyAsyncWorker
{
    FCriticalSection ResultLock;
    TArray<FMyResult> PendingResults; // written from background, read from game thread

public:
    // Called from background thread
    void PostResult(FMyResult Result)
    {
        FScopeLock Lock(&ResultLock);
        PendingResults.Add(Result);
    }

    // Called from game thread (Tick or timer)
    void FlushResults()
    {
        TArray<FMyResult> LocalResults;
        {
            FScopeLock Lock(&ResultLock);
            LocalResults = MoveTemp(PendingResults); // swap under lock, process outside
        }
        for (const FMyResult& R : LocalResults)
        {
            // Safe: game thread, no lock held
            ProcessResult(R);
        }
    }
};
```

**Move-and-process pattern:** swap the shared buffer into a local copy under the lock, then process the local copy outside the lock. This minimizes lock hold time and avoids processing under contention.

---

## `TAtomic<T>` — Simple Flags Without Locks

For single boolean or integer flags shared between threads, `TAtomic<T>` is cheaper than a full `FCriticalSection`:

```cpp
TAtomic<bool> bAsyncComplete{false};
TAtomic<int32> PendingTaskCount{0};

// Background thread
void DoWork()
{
    // ... work ...
    bAsyncComplete.Store(true);
}

// Game thread
void Tick(float DeltaTime)
{
    if (bAsyncComplete.Load())
    {
        // Safe to read results now
    }
}
```

Use `TAtomic` only for simple flags and counters. For anything more complex (reading and conditionally writing based on the value), use `FCriticalSection` — atomic read-modify-write without a lock is almost always a race condition.

---

## `ParallelFor` — Safe Usage

UE's `ParallelFor` distributes loop iterations across the task graph. The loop body must be stateless with respect to shared mutable data:

```cpp
// CORRECT — each iteration writes to its own index, no shared mutable state
TArray<FComputedResult> Results;
Results.SetNum(InputData.Num());

ParallelFor(InputData.Num(), [&](int32 Index)
{
    Results[Index] = ComputeResult(InputData[Index]); // independent writes
});
// Results are ready here — ParallelFor blocks until all iterations complete

// WRONG — all iterations write to the same variable without a lock
int32 TotalCount = 0;
ParallelFor(Items.Num(), [&](int32 Index)
{
    TotalCount++; // data race — undefined behavior
});
```

Do not access any `UObject` inside a `ParallelFor` body, even read-only — GC may run concurrently.

---

## Mass Entity, PCG, and Chaos — Threading Model

These systems run processors/solvers on background threads by design. The contract for each:

| System | Your Code Runs On | UObject Access |
|---|---|---|
| Mass Processors | Task graph (background) | No UObject access in processors — use `FMassFragment` data only |
| PCG Graph Execution | Background threads | No direct UObject mutation — write to PCG point data, apply results on game thread |
| Chaos Solver | Chaos thread | No UObject access — use `FChaosDestructionListener` callbacks which fire on game thread |
| `AsyncTask` (your code) | Background thread | Marshal back via game thread `AsyncTask` before touching any UObject |

For Mass: bridge simulation results to Actors via `UMassRepresentationSubsystem` — it handles the thread boundary for you. Never reach into Mass processor context from an Actor's `Tick`.
