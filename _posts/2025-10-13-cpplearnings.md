---
layout: single
title: "CPP Learnings"
date: 2025-10-15 10:30:00 +0530
categories: [notes]
tags: [cpp]
excerpt: ""
---

# CPP - Learnings & Notes

* TOC
{:toc}

---
## Reading the Language (hopefully like a professional)

### Value Categories
### Lifetime & Storage
### ODR One Definition Rule

## Object Model & Memory

### Layouts and Invariants

### Ownerships vs Views

#### Non-owning Views: string_view, span<T>
    - Idea: separate ownership from access. Views are pointer+length, no allocation.
    - std::string_view: read-only contiguous chars; lifetime must outlive the view.
    - std::span<T>: typed view over contiguous T[]/vector<T>; read or read-write.
    - Rule: Prefer views for parameters; avoid storing them unless you control lifetime.
    - Patterns: “own + view” (own std::string, keep string_view member bound to it).

### Small Buffer Optimizations (SBO)

### Allocators

## Constructor, Initialization and Destruction

### Member-Initializer lists

#### Construction vs Assignment
#### Initialization Order = Declaration Order

### Special Member Functions: rule of 0/3/5/-of-thumbs; defaulted & deleted
### Explicit vs Implicit constructors & conversions

Implicit & Explicit Conversions

A *non-explicit* constructor that can be called with a single argument enables implicit conversion:
```cpp
struct Meter {
  double v;
  Meter(double x) : v(x) {}  // implicit conversion from double -> Meter
};

void f(Meter m);
f(3.0);  // OK: 3.0 implicitly becomes Meter{3.0}
```
Make it `explicit` to ban that implicit conversion (but still allow direct initialization):
```cpp
struct Meter {
  double v;
  explicit Meter(double x) : v(x) {}
};

f(3.0);           // ERROR: no implicit conversion
f(Meter{3.0});    // OK: direct-init
Meter m = 3.0;    // ERROR: copy-init uses implicit conversion
Meter m{3.0};     // OK: direct-list-init

```

This goes the other way as well, when we don't wish to let our type be implicitly cast and used for another purpose. e.g,

```cpp
struct ID {
  std::string s;
  explicit operator std::string() const { return s; } // only explicit casts allowed
};

ID id{"abc"};
std::string a = id;        // ERROR
std::string b = std::string(id); // OK

```

Notes:
- Even multi-parameter ctors can enable implicit conversion if all but one have defaults; explicit still useful.

- Places where implicit conversions happen: copy-initialization (T x = a;), passing args, returning from functions, conditional operator, etc.


### Uniform initialization, brace elision, most vexing parse (how to avoid)


## value semantics, moves & perfect forwarding
### Value Semantics vs Reference Semantics
    - `T` vs `const T&`
        - Passing `T` by value copies the object.
        - Passing `const T&` avoids copying; just an alias. Also this a read only reference.
        - Prefer `const T&` for large/user-defined types (like `std::string`, `std::vector`).

    - `&v` when `v` is `const T&`: gives address of the original object.
        - Even though `v` is a reference, `&v` is just the address of the actual object.

### Reference vs Pointer Semantics

Both **references** (`const T&`) and **pointers** (`const T*`) give access to an object's memory, but are used in different contexts.

##### 🔍 Key Differences

| Concept      | `const T& v` (reference)                | `const T* p` (pointer)               |
|--------------|------------------------------------------|--------------------------------------|
| Usage        | Clean syntax: `func(obj)`                | Must pass address: `func(&obj)`      |
| Nullability  | 🔒 Cannot be null                        | ⚠️ Can be null                       |
| Rebinding    | ❌ Cannot be reseated                    | ✅ Can point to different objects    |
| Access       | Like a normal object: `v.size()`         | Requires dereferencing: `p->size()`  |
| Use Case     | Default for safe aliasing, no mutation   | Use for optional, low-level, or C APIs |

References are ideal for most read-only function arguments where null is not allowed.

---

#### Example: Reference vs Pointer

```cpp
void print_ref(const std::string& msg) {
    std::cout << msg << "\n";
}

std::string s = "hi";
print_ref(s);     // ✅ clean
```

```cpp
void print_ptr(const std::string* msg) {
    if (msg) std::cout << *msg << "\n";
}

std::string s = "hi";
print_ptr(&s);    // ✅ must pass an address explicitly
```

### Move Semantics
    - `std::move(obj)` → casts an lvalue to an rvalue reference
    - Allows you to transfer ownership without copying
    - Used in initializer lists: `: member_(std::move(param))`
    - Also used in function returns

### Copy vs move: when moves actually happen; move-after-move validity

### Pass-by-value + move vs T&& perfect-forwarding parameter patterns

### Return Value Optimization (RVO), NRVO, and [[nodiscard]]

## functions, linkage, and the build model

### Headers vs source files: declarations, definitions

### Inline functions & variables (C++17+); static in headers; anonymous namespaces
- What inline means in C++: ODR-ok to have identical defs in multiple TUs.
- When to use: functions defined in headers; header-only libs; small utilities.
An illustration:
```cpp
explicit ColWriter(fs::path outdir, std::string_view inst_id)
            : outdir_(std::move(outdir))
```

this initializes `outdir_` before the ctor body. Any advantages to initilizaing beforehand?

Without inline, suppose we had -> 
```cpp
// colstore.hpp
void ensure_dir(const fs::path& p) {
    ...
}
```

And included this in two `.cpp` files. We will get a linker error that says something like
```text
multiple definition of `ensure_dir(...)`.
```

- Alternatives: put defs in a single .cpp; or static for TU-local copies (not one entity).
- Since C++17: inline variables for header globals.
- Gotcha: inline ≠ “hint to optimize.” It’s a linkage/ODR feature.

### Templates: where to put definitions; explicit instantiation to shrink ABI

### PIMPL to stabilize ABI and speed up builds

## error handling & contracts

### Exceptions vs status codes; strong/basic/nothrow guarantees

### noexcept correctness; unwinding costs; RAII as the backbone

### Assertions, preconditions, postconditions (and [[expects]]/[[ensures]] style)

## interfaces & ownership APIs

### Function parameters: input (string_view, span<const T>), output (std::string&, vector&, return value), in-out

### Avoiding allocations at boundaries: reserve, append, fmt vs ostringstream

### std::optional, expected (if available), and error transport

## containers & iterators

### Choosing the right container (map/unordered_map/flat_map)

### Iterator invalidation rules; node vs contiguous containers

### Views and ranges (C++20): lazy pipelines, std::ranges::views

## strings, text, and formatting

### std::string vs string_view; UTF-8 realities; slicing & lifetime

### Formatting: std::format (C++20), compile-time format checks; log sinks

## filesystem & I/O

### std::filesystem::path (ties back to your ctor)

### Buffered vs unbuffered I/O; memory-mapped files; sync vs async

### Binary vs text; endian & struct packing

## concurrency & atomics

### std::thread, executors (when available), futures

### Data races, memory orders (relaxed, acq_rel, seq_cst)

### Lock-free basics; std::atomic_ref; hazards & ABA

## templates & generic programming

### Concepts & constraints; SFINAE → requires-clauses

### Perfect-forwarding constructors; CTAD; policy-based design

### CRTP and zero-overhead abstractions

## performance engineering

### Copy elision, small objects, cache friendliness

### Avoiding virtual when hot; devirtualization; final

### Measuring, not guessing: godbolt, perf, VTune, Cachegrind

## tooling & hygiene

### Warnings as errors; sanitizers; static analyzers

### CMake idioms; modules (when practical)

### Coding guidelines: ES.20, NL.26, R.11 (GotW/CppCoreGuidelines)

