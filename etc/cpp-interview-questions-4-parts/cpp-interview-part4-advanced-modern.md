# C++ Interview Questions — Part 4 (Advanced Templates, Design, Modern C++)

## 81. What is SFINAE?
Substitution Failure Is Not An Error — failed template substitutions remove overloads instead of causing errors.

---

## 82. What are concepts in C++20?
Constraints on template parameters that improve readability and error messages.

---

## 83. Difference between concepts and SFINAE
Concepts provide explicit constraints and better diagnostics; SFINAE is implicit and harder to read.

---

## 84. What is CRTP?
Curiously Recurring Template Pattern — enables static polymorphism and compile-time behavior customization.

---

## 85. What is tag dispatching?
Selecting overloads at compile time using tag types like `std::true_type` and `std::false_type`.

---

## 86. What is type erasure?
Hiding concrete types behind a uniform interface, used in `std::function` and `std::any`.

---

## 87. What is `std::function`?
A type-erased callable wrapper that can store functions, lambdas, or functors.

---

## 88. What is ABI and why does it matter?
Application Binary Interface defines how binaries interact; breaking ABI breaks binary compatibility.

---

## 89. What is the pimpl idiom?
Pointer-to-implementation hides implementation details and reduces compile-time dependencies.

---

## 90. What is the rule of five violation?
Defining some but not all special member functions, causing incorrect copy/move behavior.

---

## 91. What is strong vs basic exception safety?
Strong guarantee: operation either succeeds or has no effect.  
Basic guarantee: object remains valid but state may change.

---

## 92. What is `std::expected`?
A C++23 type representing either a value or an error, avoiding exceptions.

---

## 93. What is monadic chaining?
Using functions like `and_then` or `transform` to compose operations on optional-like types.

---

## 94. What is coroutine?
A function that can suspend and resume execution, used for async workflows in C++20.

---

## 95. What is a generator coroutine?
A coroutine that yields values lazily one at a time.

---

## 96. What is `co_await`?
Suspends execution until an awaitable object completes.

---

## 97. What is `co_yield`?
Produces a value from a coroutine generator.

---

## 98. What is `co_return`?
Ends a coroutine and optionally returns a value.

---

## 99. What is structured binding?
Decomposes tuples, pairs, and structs into named variables.

---

## 100. What is `if constexpr`?
Compile-time conditional that removes unused branches from compilation.

---

## 101. What is `std::ranges`?
C++20 library for composable, lazy algorithms operating on ranges instead of iterators.

---

## 102. Difference between `std::span` and `std::vector`
`span` is non-owning view; `vector` owns its memory.

---

## 103. What is `std::format`?
Type-safe string formatting introduced in C++20.

---

## 104. What is `std::print`?
C++23 fast output API replacing iostream-heavy formatting.

---

## 105. What is `[[nodiscard]]`?
Attribute that warns if a return value is ignored.

---

## 106. What is `[[maybe_unused]]`?
Attribute that suppresses unused warnings intentionally.

---

## 107. What is `[[fallthrough]]`?
Marks intentional fallthrough in switch statements.

---

## 108. What is `[[deprecated]]`?
Marks entities as obsolete and generates warnings when used.

---

## 109. What are `[[likely]]` and `[[unlikely]]`?
Hints to the compiler about branch probability for optimization.

---

## 110. What is an unknown attribute?
An attribute not recognized by a compiler; since C++17, it must be ignored safely.

---

## 111. What is UB sanitization?
Using tools like UBSan/ASan to detect undefined behavior at runtime.

---

## 112. What is AddressSanitizer?
A runtime memory error detector for buffer overflows and use-after-free.

---

## 113. What is ThreadSanitizer?
A tool for detecting data races in concurrent programs.

---

## 114. What is ODR (One Definition Rule)?
Each entity must have exactly one definition across the entire program.

---

## 115. What is header-only library?
A library implemented entirely in headers, avoiding linking issues.

---

## 116. What is ADL?
Argument-Dependent Lookup — finds functions based on argument namespaces.

---

## 117. What is name mangling?
Encoding function signatures into symbol names for linking.

---

## 118. What is inline namespace?
Allows versioning APIs without breaking ABI.

---

## 119. What is reflection in C++?
Compile-time introspection of types and members (partially arriving in C++26).

---

## 120. What is metaprogramming?
Writing programs that run at compile time to generate or manipulate code.

---
