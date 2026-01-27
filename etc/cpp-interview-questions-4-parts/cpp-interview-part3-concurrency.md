# C++ Interview Questions — Part 3 (Concurrency, Atomics, Memory Model)

## 51. What is a data race?
When two threads access the same memory concurrently and at least one write occurs without synchronization.

---

## 52. What is the C++ memory model?
It defines how operations on memory behave across threads and what reorderings are allowed.

---

## 53. Difference between `std::mutex` and `std::recursive_mutex`
`mutex` cannot be locked multiple times by the same thread.  
`recursive_mutex` allows reentrant locking but is slower.

---

## 54. What is a deadlock?
A situation where threads wait on each other forever due to circular resource dependencies.

---

## 55. How do you avoid deadlocks?
Use consistent lock ordering, scoped locks, `std::lock`, or lock-free designs.

---

## 56. What is `std::lock_guard`?
A RAII wrapper that locks a mutex in its constructor and unlocks in its destructor.

---

## 57. What is `std::unique_lock`?
A more flexible lock type that allows deferred locking, unlocking, and transferring ownership.

---

## 58. Difference between `condition_variable` and busy waiting
Condition variables block threads efficiently; busy waiting wastes CPU cycles.

---

## 59. What is `std::atomic`?
A type that provides lock-free, thread-safe operations on shared data.

---

## 60. What is memory ordering?
Rules that define how memory operations can be reordered across threads for performance.

---

## 61. Difference between `memory_order_relaxed` and `memory_order_seq_cst`
`relaxed` provides atomicity only.  
`seq_cst` provides strongest ordering guarantees across threads.

---

## 62. What is `memory_order_acquire` / `release`?
They create synchronization points — writes before release become visible after acquire.

---

## 63. What is `memory_order_consume`?
A weaker ordering based on data dependency; rarely implemented correctly, often treated as acquire.

---

## 64. What is a thread pool?
A set of worker threads that execute tasks from a shared queue to avoid frequent thread creation.

---

## 65. Difference between `std::async` and `std::thread`
`std::async` manages threads automatically and returns a future; `std::thread` requires manual management.

---

## 66. What is false sharing?
When unrelated variables share the same cache line, causing unnecessary cache invalidations.

---

## 67. What is lock-free programming?
Designing algorithms that guarantee progress without using locks, usually via atomics.

---

## 68. What is wait-free programming?
A stronger guarantee where every thread completes its operation in bounded steps.

---

## 69. What is ABA problem?
A concurrency issue where a value changes from A→B→A, making CAS operations incorrectly succeed.

---

## 70. What is a future and promise?
`promise` sets a value asynchronously; `future` retrieves it later, enabling synchronization.

---

## 71. What is `std::jthread`?
A C++20 thread that automatically joins on destruction and supports cooperative cancellation.

---

## 72. What is `stop_token`?
A C++20 mechanism for cooperative thread cancellation.

---

## 73. What is `std::barrier`?
A synchronization primitive that blocks threads until a fixed number arrive.

---

## 74. What is `std::latch`?
A one-time synchronization counter that blocks until it reaches zero.

---

## 75. What is memory fencing?
Explicit instructions that prevent reordering of memory operations.

---

## 76. What is thread-safe initialization of static locals?
Since C++11, static local variables are initialized in a thread-safe manner.

---

## 77. What is `volatile` and why is it not for concurrency?
`volatile` prevents certain optimizations but does not provide atomicity or synchronization.

---

## 78. What is `std::scoped_lock`?
Locks multiple mutexes safely without deadlock using RAII.

---

## 79. What is a race condition vs data race?
Race condition is logical timing bug; data race is undefined behavior per the memory model.

---

## 80. What is cooperative multitasking?
Tasks voluntarily yield control instead of being preempted by the scheduler.

---
