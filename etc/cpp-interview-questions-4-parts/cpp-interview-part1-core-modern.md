# C++ Interview Questions — Part 1 (Core Language & Modern C++)

## 1. What is RAII and why is it important?
RAII (Resource Acquisition Is Initialization) ties resource lifetime to object lifetime. Resources like memory, file handles, or locks are acquired in constructors and released in destructors, ensuring exception safety and preventing leaks.

---

## 2. Explain the Rule of Zero, Three, and Five
- Rule of Zero: Prefer designs that don’t require custom destructors, copy, or move operations.
- Rule of Three: If you define one of destructor, copy constructor, or copy assignment, you likely need all three.
- Rule of Five: Adds move constructor and move assignment in C++11.

---

## 3. Difference between `std::move` and `std::forward`
`std::move` unconditionally casts to rvalue, enabling moves.  
`std::forward` conditionally casts based on template type, preserving value category in perfect forwarding.

---

## 4. What are lvalues and rvalues?
An lvalue has a persistent addressable location; an rvalue is temporary. Move semantics mainly operate on rvalues.

---

## 5. What is copy elision and RVO?
Copy elision allows the compiler to skip copy/move operations when returning objects. Since C++17, it’s mandatory in many cases.

---

## 6. What is `constexpr`?
`constexpr` means a value or function can be evaluated at compile time if given constant inputs, improving performance and safety.

---

## 7. Difference between `struct` and `class`
Only default access differs: `struct` members are public by default, `class` members are private.

---

## 8. What is a virtual destructor and when is it needed?
A virtual destructor ensures proper cleanup when deleting derived objects through base pointers. Required for polymorphic base classes.

---

## 9. What are forwarding references?
A template parameter of type `T&&` that can bind to both lvalues and rvalues, enabling perfect forwarding.

---

## 10. What is perfect forwarding?
Passing arguments while preserving their value category using `std::forward<T>(arg)`.

---

## 11. Explain `auto` type deduction vs template deduction
`auto` drops top-level references and const unless explicitly stated, while template deduction preserves more qualifiers.

---

## 12. What is `decltype`?
It deduces the exact type of an expression, including references and cv-qualifiers.

---

## 13. Difference between `const int* p` and `int* const p`
`const int* p` → pointer to const int.  
`int* const p` → const pointer to int.

---

## 14. What is move semantics?
Allows transferring resources instead of copying them, improving performance for temporary objects.

---

## 15. What is the difference between shallow and deep copy?
Shallow copy copies pointers; deep copy duplicates owned resources. RAII types typically perform deep copies.

---

## 16. What is a smart pointer?
An object that manages a raw pointer using RAII to ensure automatic memory cleanup.

---

## 17. Difference between `unique_ptr` and `shared_ptr`
`unique_ptr` has sole ownership and zero overhead.  
`shared_ptr` uses reference counting for shared ownership.

---

## 18. What is `weak_ptr`?
A non-owning reference to a `shared_ptr` that prevents reference cycles.

---

## 19. What is `nullptr` and why is it better than `NULL`?
`nullptr` is a typed null pointer, avoiding ambiguity with integers.

---

## 20. What is `explicit` keyword?
Prevents unintended implicit conversions in constructors and conversion operators.

---

## 21. What is `mutable`?
Allows a member to be modified even in `const` objects, typically for caches or lazy evaluation.

---

## 22. What is `noexcept`?
Specifies that a function won’t throw. Enables optimizations and affects move semantics.

---

## 23. Difference between `new/delete` and `malloc/free`
`new/delete` call constructors/destructors and are type-safe; `malloc/free` don’t.

---

## 24. What is a lambda expression?
An inline anonymous function, often used for callbacks and algorithms.

---

## 25. What are capture modes in lambdas?
By value `[=]`, by reference `[&]`, or explicitly capturing selected variables.

---

## 26. What is `std::optional`?
Represents a value that may or may not be present without using null pointers or sentinel values.

---

## 27. What is `std::variant`?
A type-safe union that holds exactly one of several alternative types.

---

## 28. What is `std::any`?
A type-erased container that can hold any copyable type.

---

## 29. What is `std::span`?
A non-owning view over a contiguous sequence, safer than raw pointers + size.

---

## 30. What is `constexpr if`?
A compile-time conditional that discards non-selected branches during compilation.

---
