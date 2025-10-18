---
layout: single
title: "Pointers & References"
date: 2025-10-16 10:30:00 +0530
categories: [notes]
tags: [cpp]
excerpt: ""
classes: wide
---

# Pointers & References

* TOC
{:toc}

---

## Pointer
A **pointer** holds the *address* of an object or `nullptr`. Use pointers when you need *indirection*, *optionality*, *polymorphism*. Prefer smart pointers (std::unique_ptr, std::shared_ptr) for ownership. Use raw pointers for non-owning observation.

#### Indirection
Accessing an object through something else (an address/handle) rather than directly.
Consider Widget to have a paint() method belonging to it, and if we want to access it ->
- Direct: `Widget w; w.paint();`
- Indirect: `Widget* p = &w; p->paint();` (you go “through” a pointer)
- References (`T&`) are also indirection: `Widget& r = w; r.paint();`. References are discussed in detail below.

Uses->
- Pointing to different objects at runtime (reseating)
    less fancily, to simply point to differet objects while running.
    ```cpp
    Animal* pet;

    Dog dog;
    Cat cat;

    pet = &dog;
    pet->speak();  // Woof!

    pet = &cat;
    pet->speak();  // Meow!
    ```
    We aren't creating new memory, we're simply making it point to different objects.
- Point to something that lives elsewhere (on heap, pool, library)
    - When we want create something that outlives the current scope
        By default, variables in C++ have automatic (static) lifetimes.

        That means:
        When a function ends, all local variables are destroyed — their memory is automatically cleaned up.

        So if we create an object inside a function, and try to return a pointer to it — we’d be pointing to garbage after the function ends!

        Example:
        ```cpp
        int* makeNumber() {
            int x = 5;      // x lives only inside this function
            return &x;      //  BAD: x will be destroyed when function ends
        }

        int* ptr = makeNumber();
        std::cout << *ptr;  //  Undefined behavior: x is already gone!
        ```

        Solution:
        ```cpp
        int* makeNumber() {
            int* x = new int(5);   // x lives on the heap now!
            return x;
        }
        int* ptr = makeNumber();
        std::cout << *ptr << std::endl;  //  Works: memory still exists
        delete ptr;  //  must free it when done
        ```

    - Don't know size/type during compile time. Will need to be allocated on heap.

    - Created by a library and given to us

- Needed for polymorphism (call virtual methods through a base handle)
Virtual functions only work through pointers or references. This is explained in detail in the polymorphism section below.

#### Optionality
A value may be present or may be absent.
Ways to model optionality in C++:
- Raw pointer: `T*` (use nullptr for “absent”). Non-owning by default.
- `std::optional<T>`: the object may or may not exist (by value).
- `std::optional<std::reference_wrapper<T>>`: optional reference-like without null refs.

```cpp
// Non-owning, nullable handle
void maybe_draw(const Widget* w) {
    if (!w) return;  // absent
    w->paint();      // present
}

// Optional by value (copies or moves T)
std::optional<int> find_index(bool hit) {
    if (!hit) return std::nullopt;
    return 42;
}
```

#### Polymorphism
Allows one interface (e.g. a function call) to work with different types of objects.
**Call the right function for the right object**

Example -
```cpp
class Animal {
public:
    virtual void speak() {
        std::cout << "Animal speaks" << std::endl;
    }
};

class Dog : public Animal {
public:
    void speak() override {
        std::cout << "Woof!" << std::endl;
    }
};

class Cat : public Animal {
public:
    void speak() override {
        std::cout << "Meow!" << std::endl;
    }
};
```
[*Above code snippet reference (dynamic polymorphism in c)](https://hackernoon.com/understanding-dynamic-polymorphism-in-c)

##### Object Slicing Porblem without Pointers (loses derived info)
```cpp
// IF WE USE POINTERS -
Animal* a1 = new Dog();
Animal* a2 = new Cat();
a1->speak();  // Output: Woof!
a2->speak();  // Output: Meow!

// ELSE!
Dog d;
Animal a = d; // Object slicing!
a.speak();    // Output: Animal speaks
```

`Animal` is the base class (aka parent class)
`Dog` is the derived class (aka child class)
When we use a pointer, a pointer to `Animal` is actually pointing to a `Dog` object.
Since `speak()` is declared `virtual` in `Animal`, C++ uses the virtual table (vtable) mechanism to resolve the function at runtime.
The call to `a1->speak()` goes to `Dog::speak()` because the actual object is a Dog.

**Here, virtual dispatch kicks in. Polymorphism works.**


#### Copying vs Moving
```cpp
int* a = new int(42);
int* b = a; // simple pointer copy
delete a;
delete b; // ❌ undefined behavior (double delete)
```

This is becasue it looks like ->
```css
a ─┬──▶ [ int: 42 ]
   │
b ─┘
```

This can be avoided without `unique_ptr`:
```cpp
std::unique_ptr<int> a = std::make_unique<int>(42);

// arranged like:
 a ───▶ [ int: 42 ] (on heap)
```


```cpp
std::unique_ptr<int> a = std::make_unique<int>(42);
std::unique_ptr<int> b = std::move(a); // a is now nullptr, b owns the int
```


### When to use Pointers

#### Dynamic Allocation (heap memory)
`Animal* a = new Dog();  // Heap allocation`
We need to make sure to mange memory in these cases, using `delete` or smart pointers like `unique_ptr`.

#### Linked Data Structures (Trees, Graphs, etc.)
```cpp
class Node {
public:
    int data;
    Node* next;
};
```

#### Polymorpism (Dynamic Dispatch)


#### Avoiding Copies (Performance & Semantics) {*REFERENCES*}
```cpp
void process(const MyClass& obj); // no copy
```

---

## Reference? What is it?

✅ A reference is an alias to an existing variable.
- It's not a new object
- It refers to the same memory as the original
- After initialization, a reference cannot be changed to refer to something else

```cpp
int x = 42;
int& ref_to_x = x;

ref_to_x = 100;
std::cout<< x; // Yields 100.
```

## Reference to a Pointer

```cpp
void closefile(std::File*& f){ // Here we are referencing a pointer. file must be a pointer.
    if(f){
        std::fclose(f);
        f = nullptr;
    }
}

// This will allow us to directly manipulate where f is pointing to.
```

A relevant pass by value mistake -->
If we had done this instead. The pointer that was passed by value ultimately leads to the same location. File would definitely be closed. However, we'd have a dangling pointer. We need to set the original itself to null.
```cpp
// If we did something like ->
void safeclose_wrong(std::File* f){
    if(f){
        std::fclose(f);
        f = nullptr;
    }
}
// We are actually getting a copy of the pointer instead. We aren't setting the original pointer to nullptr.
```

### When we use this

**To modify a caller's pointer itself**
When would we want to modify?
- You're managing dynamic memory
- You're closing or freeing a resource
- You want to prevent reuse of a now-invalid pointer

**Stealing a resource**
```cpp
std::unique_ptr<int> a = std::make_unique<int>(42);
std::unique_ptr<int> b = std::move(a);  // b now owns it, a is nullptr
```
Here, `a` no longer owns the pointer — `b` "stole" it.

### Examples

#### 1. Freeing Heap Memory
```cpp
void safe_delete(int*& ptr) {
    delete ptr;
    ptr = nullptr;
}

int* p = new int(42);
safe_delete(p);  // cleans and nulls p

```

#### 2. Taking Ownership
```cpp
void take_ownership(std::unique_ptr<MyClass>& ptr) {
    do_something(std::move(ptr));  // ptr becomes nullptr
}

// caller:
std::unique_ptr<MyClass> p = std::make_unique<MyClass>();
take_ownership(p);
// p is now nullptr, ownership transferred

```

NEED TO PUT INFO std::move AND MOVE SEMANTICS IN DETAIL IN A RELEVANT PLAC EWITH PROPER SEGUE, LINK THIS TO CONSTRUCTORS.