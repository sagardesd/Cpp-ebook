# C++ Interview Questions — Part 2 (STL, Memory, Containers)

## 31. Difference between `std::vector` and `std::list`
`vector` offers contiguous storage and fast random access.  
`list` allows constant-time insertion/removal but no random access.

---

## 32. Difference between `std::map` and `std::unordered_map`
`map` is ordered (tree-based, O(log n)).  
`unordered_map` is hash-based (average O(1), no order).

---

## 33. When would you use `std::deque`?
When you need fast insertion/removal at both ends and random access.

---

## 34. What is iterator invalidation?
Some container operations invalidate iterators, references, or pointers, making them unsafe to use afterward.

---

## 35. What is a custom allocator?
An allocator controls memory allocation strategy for containers, useful for performance or memory tracking.

---

## 36. Difference between `reserve()` and `resize()`
`reserve()` increases capacity only.  
`resize()` changes size and constructs or destroys elements.

---

## 37. What is emplacement?
`emplace` constructs elements directly in-place, avoiding unnecessary copies or moves.

---

## 38. What is `std::move_if_noexcept`?
Moves if the move constructor is noexcept; otherwise copies to maintain exception safety.

---

## 39. What is the small string optimization (SSO)?
An optimization where small strings are stored directly inside the string object without heap allocation.

---

## 40. What is object slicing?
When a derived object is copied into a base object by value, losing derived-specific data.

---

## 41. What is alignment and why does it matter?
Alignment ensures objects are placed at memory boundaries required by hardware, improving performance and correctness.

---

## 42. What is padding in structs?
Extra bytes inserted to satisfy alignment requirements, which can increase object size.

---

## 43. What is `std::byte`?
A type-safe representation of raw memory introduced in C++17.

---

## 44. What is placement new?
Constructs an object at a specified memory address instead of allocating new memory.

---

## 45. What is `std::launder`?
Used to safely access objects created in reused storage, avoiding undefined behavior.

---

## 46. What is a dangling reference?
A reference or pointer that refers to an object that no longer exists.

---

## 47. What is undefined behavior?
Program behavior not defined by the standard — can crash, misbehave, or appear to work.

---

## 48. Difference between `delete` and `delete[]`
`delete` destroys one object; `delete[]` destroys arrays and calls all destructors.

---

## 49. What is the difference between shallow and deep const?
Shallow const prevents modifying the object itself; deep const prevents modifying pointed-to data too.

---

## 50. What is cache locality and why does it matter?
Accessing memory sequentially improves cache hits and performance, favoring contiguous containers like `vector`.

---
