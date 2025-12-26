# C++11 Smart Pointers

## Overview

C++11 introduced **smart pointers** to the Standard Library, providing automatic resource management and helping developers avoid resource leaks and dangling pointers. Smart pointers manage the lifetime of resources (memory, files, network connections, etc.) and automatically release them when they're no longer needed. They leverage the **RAII (Resource Acquisition Is Initialization)** principle to ensure resources are properly cleaned up.

## Smart Pointers Introduced in C++11

C++11 introduced **three main smart pointers** as **class templates**:

### 1. `std::unique_ptr`
A smart pointer that provides exclusive ownership of a resource, ensuring only one pointer can own it at a time.

### 2. `std::shared_ptr`
A smart pointer that allows multiple pointers to share ownership of the same resource using reference counting.

### 3. `std::weak_ptr`
A non-owning smart pointer that holds a reference to a resource managed by `std::shared_ptr`, useful for breaking circular references.

## Key Benefits

- **Automatic resource management**: Resources are automatically released when no longer needed
- **Exception safety**: Memory is properly released even if exceptions occur
- **No overhead for unique ownership**: `std::unique_ptr` has zero-cost abstraction
- **Clear ownership semantics**: Code intent is explicit about who owns the resource

## Quick Comparison

| Smart Pointer | Ownership | Use Case |
|---|---|---|
| `std::unique_ptr` | Exclusive | Single owner scenarios |
| `std::shared_ptr` | Shared | Multiple owners of the same object |
| `std::weak_ptr` | Non-owning | Breaking circular references |