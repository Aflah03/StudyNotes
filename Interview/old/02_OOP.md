# Object-Oriented Programming (OOP) — Conceptual Interview Guide (HPE)

> Leaning toward **C++** since your project is in C++ and interviewers will connect OOP theory to your code. Each concept has the plain definition, the "why," and a short example. The Q&A bank at the end is your self-test.

---

## 1. What is OOP and Why?

**One-line answer:** Object-Oriented Programming is a programming paradigm that organizes code around **objects** — bundles of data (attributes) and behavior (methods) — rather than around functions and logic alone.

**Why OOP exists / its benefits:**
- **Modularity** — code is split into self-contained objects.
- **Reusability** — inheritance and composition let you reuse code.
- **Maintainability** — changes are localized to objects.
- **Scalability** — easier to extend large systems.
- **Real-world modeling** — objects map naturally to real entities (a `Car`, a `BankAccount`, a `Student`).

**Class vs Object (fundamental):**
- A **class** is a blueprint/template that defines attributes and methods.
- An **object** is an instance of a class — a concrete thing created from the blueprint with actual values.
- Analogy: a class is the architectural plan of a house; each built house is an object.

```cpp
class Car {          // class = blueprint
    string brand;
    void drive();
};
Car myCar;           // object = instance
```

---

## 2. The Four Pillars of OOP (MUST know cold)

### Pillar 1 — Encapsulation
**Definition:** Bundling data and the methods that operate on that data into a single unit (the class), and restricting direct access to the internal data using access modifiers.

**Why:** Protects the internal state from unintended modification (data hiding). You expose a controlled interface (getters/setters) and hide implementation details.

```cpp
class BankAccount {
private:
    double balance;              // hidden
public:
    void deposit(double amt) {   // controlled access
        if (amt > 0) balance += amt;
    }
    double getBalance() { return balance; }
};
```

Key phrase: *"Encapsulation is about data hiding and controlled access."*

### Pillar 2 — Abstraction
**Definition:** Hiding complex implementation details and exposing only the essential features to the user.

**Why:** Reduces complexity. The user knows *what* an object does, not *how* it does it.

- Example: When you call `car.drive()`, you don't need to know how the engine works internally.
- Achieved in C++ via **abstract classes** (classes with pure virtual functions) and interfaces.

**Encapsulation vs Abstraction (they'll ask this):**
- Encapsulation = *how* you hide (wrapping data + code together, using access modifiers). It's about data hiding at the implementation level.
- Abstraction = *what* you hide (hiding complexity, showing only essentials). It's about design level.
- Simple: encapsulation hides data; abstraction hides complexity/implementation.

### Pillar 3 — Inheritance
**Definition:** A mechanism where a new class (derived/child) acquires the properties and behaviors of an existing class (base/parent).

**Why:** Code reuse and establishing an "is-a" relationship (a `Dog` *is an* `Animal`).

```cpp
class Animal {
public:
    void eat() { cout << "eating"; }
};
class Dog : public Animal {   // Dog inherits from Animal
public:
    void bark() { cout << "barking"; }
};
```

**Types of inheritance (C++ supports all):**
1. **Single** — one base, one derived.
2. **Multiple** — a class inherits from more than one base class. *(C++ allows this; Java doesn't.)*
3. **Multilevel** — a chain: A → B → C.
4. **Hierarchical** — one base, many derived classes.
5. **Hybrid** — a combination of the above.

**The Diamond Problem** (classic C++ question): In multiple inheritance, if class D inherits from B and C, and both B and C inherit from A, then D gets two copies of A's members — an ambiguity. **Solution in C++: virtual inheritance** (`class B : virtual public A`), which ensures only one shared copy of A.

### Pillar 4 — Polymorphism
**Definition:** "Many forms" — the ability of the same interface/function to behave differently depending on the object or arguments.

**Two types:**

**a) Compile-time (static) polymorphism** — resolved at compile time.
- **Function overloading** — same function name, different parameters (number/type).
- **Operator overloading** — giving operators custom meaning for user-defined types.

```cpp
int add(int a, int b);              // overloaded
double add(double a, double b);     // by parameter type
```

**b) Runtime (dynamic) polymorphism** — resolved at runtime via **virtual functions**.
- **Function overriding** — a derived class provides its own version of a base class method.

```cpp
class Shape {
public:
    virtual void draw() { cout << "shape"; }   // virtual
};
class Circle : public Shape {
public:
    void draw() override { cout << "circle"; } // overrides
};
Shape* s = new Circle();
s->draw();   // calls Circle::draw() — runtime polymorphism
```

**Overloading vs Overriding (very common):**
| | Overloading | Overriding |
|---|-------------|------------|
| When resolved | Compile time | Runtime |
| Relationship | Same class | Base–derived classes |
| Signature | Different params | Same signature |
| Keyword | — | `virtual` (base) |

---

## 3. Virtual Functions & the vtable (deep C++ question)

- A **virtual function** is a member function declared with `virtual` in the base class, meant to be overridden. It enables runtime polymorphism.
- **How it works internally:** The compiler creates a **vtable (virtual table)** — an array of function pointers — for each class with virtual functions. Each object holds a hidden **vptr (virtual pointer)** pointing to its class's vtable. At runtime, a virtual call is resolved by looking up the correct function through the vptr → vtable. This is called **dynamic dispatch / late binding.**

- **Pure virtual function:** `virtual void draw() = 0;` — has no implementation and *must* be overridden. A class with at least one pure virtual function is an **abstract class** and cannot be instantiated.

- **Virtual destructor:** If you delete a derived object through a base pointer, the destructor must be `virtual` in the base — otherwise only the base destructor runs and you get a memory leak / undefined behavior. **Always make base class destructors virtual when using inheritance and polymorphism.**

---

## 4. Constructors and Destructors

- **Constructor** — a special method called automatically when an object is created; used to initialize it. Same name as the class, no return type.
  - **Default constructor** — no parameters.
  - **Parameterized constructor** — takes arguments.
  - **Copy constructor** — creates an object as a copy of another: `ClassName(const ClassName& other)`.
  - **Constructor overloading** — multiple constructors with different parameters.
- **Destructor** — `~ClassName()` — called automatically when an object goes out of scope or is deleted; used to free resources. No parameters, no return type.

**Shallow vs Deep copy (important with pointers):**
- **Shallow copy** — copies member values as-is, including pointers (both objects point to the same memory → dangerous double-free).
- **Deep copy** — allocates new memory and copies the actual pointed-to data (independent objects). You write a custom copy constructor for deep copy.

---

## 5. Access Modifiers (C++)

- **private** — accessible only within the class itself.
- **protected** — accessible within the class and its derived classes.
- **public** — accessible from anywhere.

**Inheritance access:** `public`, `protected`, `private` inheritance change how base members are inherited (public inheritance keeps public as public — the most common and the "is-a" case).

---

## 6. Static Members

- **Static variable** — shared across all objects of the class; only one copy exists. Belongs to the class, not any object.
- **Static method** — can be called without an object; can only access static members.

```cpp
class Counter {
    static int count;      // shared by all objects
public:
    Counter() { count++; }
    static int getCount() { return count; }
};
```

---

## 7. Abstract Class vs Interface

- **Abstract class** — a class with at least one pure virtual function; cannot be instantiated; can have both implemented and pure virtual methods, plus data members. Represents an "is-a" with partial implementation.
- **Interface** — in C++, simulated by a class with *only* pure virtual functions and no data members. Represents a pure contract of "what to do."

*(In Java, `interface` and `abstract class` are separate keywords — mention this if they compare languages. In C++ both are done with abstract classes.)*

---

## 8. Relationships Between Classes

- **Association** — a general "uses-a" relationship between two independent objects (a Teacher and a Student).
- **Aggregation** — a "has-a" relationship where the part *can exist independently* of the whole (a Department has Professors, but Professors survive if the Department closes). Weak ownership.
- **Composition** — a "has-a" relationship where the part *cannot exist without* the whole (a House has Rooms; destroy the house and the rooms go too). Strong ownership.
- **Inheritance** — an "is-a" relationship.

**Composition vs Inheritance (design principle):** Prefer **composition over inheritance** when possible. Inheritance creates tight coupling and can break with deep hierarchies; composition is more flexible ("has-a" is often safer than "is-a").

---

## 9. SOLID Principles (senior-level bonus — very impressive to know)

Five design principles for maintainable OOP code:

1. **S — Single Responsibility** — a class should have only one reason to change (one job).
2. **O — Open/Closed** — open for extension, closed for modification (extend behavior without editing existing code).
3. **L — Liskov Substitution** — objects of a derived class should be substitutable for the base class without breaking the program.
4. **I — Interface Segregation** — many small, specific interfaces are better than one big general one.
5. **D — Dependency Inversion** — depend on abstractions, not concrete implementations.

Even naming these correctly signals maturity.

---

## 10. OOP vs Procedural Programming

- **Procedural** (C) — code organized as functions/procedures operating on data; data and functions are separate; top-down approach. Harder to maintain at scale.
- **OOP** (C++, Java) — code organized as objects bundling data and behavior; bottom-up; better for large, complex systems; supports encapsulation, inheritance, polymorphism.

---

## 11. Interview Q&A Bank (self-test)

**Q: What are the four pillars of OOP?**
Encapsulation (bundling data + methods and hiding internal state), Abstraction (hiding complexity, showing essentials), Inheritance (deriving new classes from existing ones), and Polymorphism (same interface, many forms).

**Q: Difference between a class and an object?**
A class is a blueprint defining attributes and methods; an object is a concrete instance of that class with actual values.

**Q: Encapsulation vs Abstraction?**
Encapsulation is data hiding — wrapping data and methods together and controlling access with modifiers. Abstraction is hiding implementation complexity and exposing only essential features. Encapsulation is implementation-level; abstraction is design-level.

**Q: Overloading vs Overriding?**
Overloading is compile-time — same function name, different parameters, in the same class. Overriding is runtime — a derived class redefines a base class virtual function with the same signature.

**Q: What is a virtual function?**
A member function declared `virtual` in the base class that can be overridden in a derived class, enabling runtime polymorphism through dynamic dispatch.

**Q: How does runtime polymorphism work internally?**
Via a vtable (a per-class table of virtual function pointers) and a vptr in each object. At runtime the correct function is looked up through the vptr → vtable — this is late/dynamic binding.

**Q: What is a pure virtual function and an abstract class?**
A pure virtual function (`= 0`) has no implementation and must be overridden. A class with at least one pure virtual function is abstract and cannot be instantiated.

**Q: Why do you need a virtual destructor?**
So that deleting a derived object through a base class pointer calls the derived destructor too. Without it, only the base destructor runs, causing resource/memory leaks.

**Q: What is the diamond problem and how is it solved?**
In multiple inheritance, when a class inherits two classes that share a common base, it gets duplicate base members (ambiguity). C++ solves it with virtual inheritance so only one shared copy of the base exists.

**Q: Types of inheritance in C++?**
Single, multiple, multilevel, hierarchical, and hybrid.

**Q: Shallow copy vs deep copy?**
A shallow copy duplicates member values including pointers, so both objects share the same memory (risky). A deep copy allocates new memory and copies the actual data, making the objects independent. Deep copy needs a custom copy constructor.

**Q: What is a copy constructor?**
A constructor that creates a new object as a copy of an existing one, taking a const reference to the same class.

**Q: Static vs non-static members?**
Static members belong to the class and are shared across all objects (one copy). Non-static members belong to each object individually.

**Q: private vs protected vs public?**
Private = accessible only inside the class. Protected = accessible in the class and derived classes. Public = accessible everywhere.

**Q: Association vs Aggregation vs Composition?**
Association is a general relationship between objects. Aggregation is "has-a" where the part can exist independently (weak ownership). Composition is "has-a" where the part cannot exist without the whole (strong ownership).

**Q: Composition vs Inheritance — which to prefer?**
Prefer composition where possible. Inheritance creates tight coupling; composition ("has-a") is more flexible and avoids fragile deep hierarchies. Use inheritance for genuine "is-a" relationships.

**Q: Abstract class vs interface?**
An abstract class can have both implemented and pure virtual methods plus data. An interface (in C++, a class with only pure virtual functions) is a pure contract with no implementation.

**Q: Can you achieve polymorphism without inheritance?**
Yes — compile-time polymorphism via function/operator overloading doesn't need inheritance. Runtime polymorphism does need inheritance and virtual functions.

**Q: What are the SOLID principles?**
Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion — five principles for maintainable OOP design.

**Q: What's the difference between C and C++ / procedural vs OOP?**
Procedural programming (C) organizes code as functions operating on separate data, top-down. OOP (C++) organizes code as objects bundling data and behavior, supporting encapsulation, inheritance, and polymorphism — better for large systems.

---

### Quick-recall cheat sheet
- 4 pillars: Encapsulation, Abstraction, Inheritance, Polymorphism.
- Encapsulation = hide data; Abstraction = hide complexity.
- Overloading = compile-time; Overriding = runtime (needs `virtual`).
- Runtime polymorphism = vtable + vptr = dynamic dispatch.
- Always: virtual destructor when using inheritance.
- Diamond problem → virtual inheritance.
- Prefer composition over inheritance.
- SOLID for good design.
