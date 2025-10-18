---
layout: single
title: "C++ Object Model & RAII (Ownership, Lifetime, Memory)"
date: 2025-10-15 10:30:00 +0530
categories: [notes]
tags: [cpp]
excerpt: ""
classes: wide
---

# 

* TOC
{:toc}

---

## 1. Types & Layout
### 1.1 Classes & Structs
*What is a type (in this context)?*
A type describes a set of values and the operations valid on them. For user-defined types (classes/structs), it also fixes a memory layout for each object: the order, alignment, and presence of data members, plus what functions can act on those objects.

*Classes vs Structs (quick)*
Refer classes here, and structs here.

Same thing in C++—the only difference is default access:
struct: members are public by default
class: members are private by default

*Objects & Identity*
An object is a concrete instance of a type, stored somewhere (stack/heap/static). Each object gets its own storage for non-static (will discuss soon) data members, so different objects can hold different values.

```cpp

```

**The `this` pointer (how member functions know “which one?”)**
Non-static member functions receive a hidden first parameter: `this`, a pointer to the specific object being used. This help in 'pointing' to and identifying that particular instance of a type. So:


```cpp
struct C {
  int value = 0;  
  // 1) member: operates on *this* object
  void bump(int by) {          // implicit C* this
    value += by;               // means this->value += by;
  }
};

C a;
a.bump(3);          // a's value in particular increases by 3 from 0 to 3.
```
Mental equivalent of what's going on with `this` working implicitly ->

```cpp
//Conceptual Translation
void bump(C* this_, int by) { this_->value += by; } // implicitly pointing to a location for 'a' in particular.
```
**Static vs non-static members (preview)**
- Non-static data: per-object storage (affects sizeof(C)), accessed via this.
- Static data: one per type (does not affect sizeof(C)), lives in static storage.
- Static functions: belong to the type, have no this, great for factories/utilities.


### 1.2 Data members, access-control, standard layout vs trivial
Access control
- public → visible to everyone
- protected → visible to derived classes
- private → internal to the type
Use access to enforce invariants (e.g., keep representation valid).

### 1.2 Data Members, Access Control, Standard-Layout vs Trivial

**What this section covers:** where member data lives, who can see it, how layout is formed in memory, and what “standard-layout” and “trivial” actually mean (modern replacement for the old “POD” idea).

---

#### Access Control (public/protected/private)

Use access to enforce invariants and express intent.

```cpp
struct BankAccount {
private:
  double balance_ = 0.0;        // hidden; callers can't break invariants directly
public:
  void deposit(double x) { if (x >= 0) balance_ += x; }
  double balance() const { return balance_; }
protected:
  void audit_adjust(double x) { balance_ += x; } // accessible to derived types
};


### 1.3 `static` members (type-level) vs non-static (object-level)



## 2. Object Model

## 3. Construction & Destruction (RAII Core)

## 4. Initialization patterns

## 5. Ownership buffers

## 6. Resource Management (RAII in practice)

## 7. Storage duration & Lifetime pitfalls


