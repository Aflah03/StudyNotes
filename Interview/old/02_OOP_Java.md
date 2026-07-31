# Object-Oriented Programming (OOP) — Conceptual Interview Guide — JAVA (HPE)

> Java-focused version. The four pillars are universal, but Java differs from C++ in important ways interviewers probe: **no destructors (garbage collection), no multiple inheritance of classes (interfaces instead), no pointers or operator overloading, `interface` and `abstract class` are separate keywords, and all non-static methods are virtual by default.** Those distinctions are flagged throughout. Q&A bank at the end for self-testing.

---

## 1. What is OOP and Why?

**One-line answer:** Object-Oriented Programming is a paradigm that organizes code around **objects** — bundles of data (fields) and behavior (methods) — rather than around functions and logic alone.

**Benefits:** modularity, reusability (inheritance/composition), maintainability, scalability, and natural real-world modeling.

**Class vs Object:**
- A **class** is a blueprint defining fields and methods.
- An **object** is an instance of a class created with `new`, having actual values.

```java
class Car {              // class = blueprint
    String brand;
    void drive() { }
}
Car myCar = new Car();   // object = instance (created with new)
```

**Java-specific note:** Everything in Java (except primitives like `int`, `char`, `boolean`) is an object, and every class implicitly extends the root class `Object`.

---

## 2. The Four Pillars of OOP (MUST know cold)

### Pillar 1 — Encapsulation
**Definition:** Bundling data (fields) and the methods that operate on them into a single unit (the class), and restricting direct access to fields using access modifiers.

**Java way:** make fields `private`, expose controlled `public` getters and setters.

```java
class BankAccount {
    private double balance;                 // hidden

    public void deposit(double amt) {       // controlled access
        if (amt > 0) balance += amt;
    }
    public double getBalance() { return balance; }
}
```

Key phrase: *"Encapsulation is data hiding + controlled access via getters/setters."*

### Pillar 2 — Abstraction
**Definition:** Hiding complex implementation details and exposing only essential features.

**Java way:** achieved through **abstract classes** and **interfaces**. The caller knows *what* a method does, not *how*.

- Example: you call `list.add(x)` without knowing how `ArrayList` resizes internally.

**Encapsulation vs Abstraction (common question):**
- Encapsulation = *how* you hide (wrapping data + methods, using access modifiers) — implementation level.
- Abstraction = *what* you hide (hiding complexity, exposing essentials) — design level.
- Simple line: encapsulation hides *data*; abstraction hides *implementation/complexity*.

### Pillar 3 — Inheritance
**Definition:** A mechanism where a child class (subclass) acquires fields and methods of a parent class (superclass), establishing an "is-a" relationship.

**Java keyword:** `extends` (for classes), `implements` (for interfaces). Use `super` to call the parent's constructor or methods.

```java
class Animal {
    void eat() { System.out.println("eating"); }
}
class Dog extends Animal {        // Dog is-a Animal
    void bark() { System.out.println("barking"); }
}
```

**Types of inheritance in Java:**
1. **Single** — one parent, one child. ✅
2. **Multilevel** — a chain A → B → C. ✅
3. **Hierarchical** — one parent, many children. ✅
4. **Multiple inheritance of classes** — ❌ **NOT allowed in Java** (a class cannot `extends` two classes).
5. **Multiple inheritance of type** — ✅ allowed **through interfaces** (a class can `implement` many interfaces).

**⭐ The Diamond Problem — Java's approach (very common Java question):**
Java forbids multiple *class* inheritance specifically to avoid the diamond problem (ambiguity when two parents share a common base). Instead, Java uses **interfaces**. Before Java 8 interfaces had no method bodies, so there was no ambiguity. Since **Java 8**, interfaces can have **default methods** — so a diamond *can* reappear: if a class implements two interfaces with the same default method, the compiler forces you to **override it explicitly** and you can pick one with `InterfaceName.super.method()`.

### Pillar 4 — Polymorphism
**Definition:** "Many forms" — the same interface/method behaves differently depending on the object or arguments.

**a) Compile-time (static) polymorphism — Method Overloading**
Same method name, different parameter lists (number/type/order), in the same class. Resolved at compile time.

```java
int add(int a, int b) { return a + b; }
double add(double a, double b) { return a + b; }   // overloaded
```

**Java note:** Java does **NOT** support operator overloading (C++ does). Java only has method overloading for compile-time polymorphism.

**b) Runtime (dynamic) polymorphism — Method Overriding**
A subclass provides its own version of a superclass method with the same signature. Resolved at runtime via **dynamic method dispatch**.

```java
class Shape {
    void draw() { System.out.println("shape"); }
}
class Circle extends Shape {
    @Override
    void draw() { System.out.println("circle"); }   // overrides
}
Shape s = new Circle();
s.draw();   // prints "circle" — runtime polymorphism
```

**⭐ Java-specific and important:** In Java, **all non-static, non-final, non-private methods are "virtual" by default** — they participate in dynamic dispatch automatically. There is no `virtual` keyword (unlike C++). To *prevent* overriding you use `final`. `static` and `private` methods are NOT overridden (they're resolved at compile time / bound to the class); if a subclass declares a static method with the same signature, that's **method hiding**, not overriding.

**Overloading vs Overriding:**
| | Overloading | Overriding |
|---|-------------|------------|
| Resolved | Compile time | Runtime |
| Where | Same class | Subclass vs superclass |
| Params | Must differ | Must be identical |
| Return type | Can differ | Same (or covariant) |
| Annotation | — | `@Override` (recommended) |

---

## 3. Constructors (no destructors in Java)

- **Constructor** — a special method called automatically when an object is created with `new`; used to initialize it. Same name as the class, **no return type**.
- **Default constructor** — if you write none, Java provides a no-arg constructor automatically.
- **Parameterized constructor** — takes arguments.
- **Constructor overloading** — multiple constructors with different parameters.
- **Constructor chaining** — `this()` calls another constructor in the same class; `super()` calls the parent constructor (and `super()` must be the first statement, called implicitly if you omit it).

```java
class Student {
    String name; int age;
    Student() { this("Unknown", 0); }        // this() chaining
    Student(String n, int a) { name = n; age = a; }
}
```

**⭐ No destructors in Java:** Unlike C++, Java has **no destructors**. Memory is reclaimed automatically by the **Garbage Collector (GC)**. There *was* a `finalize()` method but it's **deprecated** and unreliable — don't rely on it. For resource cleanup (files, connections) use **try-with-resources** and the `AutoCloseable` interface.

**Shallow vs Deep copy:**
- **Shallow copy** — copies field values; object references are shared (both objects point to the same nested objects).
- **Deep copy** — recursively copies nested objects so the copy is fully independent.
- Java's default `Object.clone()` (with `Cloneable`) does a shallow copy; deep copy requires custom logic or copy constructors.

---

## 4. Access Modifiers (Java has four)

| Modifier | Same class | Same package | Subclass (other pkg) | Anywhere |
|----------|:---------:|:-----------:|:-------------------:|:--------:|
| **private** | ✅ | ❌ | ❌ | ❌ |
| **default** (no keyword) | ✅ | ✅ | ❌ | ❌ |
| **protected** | ✅ | ✅ | ✅ | ❌ |
| **public** | ✅ | ✅ | ✅ | ✅ |

**Java note:** the **default (package-private)** level — visible within the same package — is unique and often forgotten; mention it to stand out. (C++ has no package concept.)

---

## 5. Static Members and the `static` Keyword

- **Static variable** — belongs to the class, shared by all objects (one copy).
- **Static method** — called on the class without an object; can only access static members (no `this`).
- **Static block** — runs once when the class is loaded, used for static initialization.

```java
class Counter {
    static int count = 0;                 // shared
    Counter() { count++; }
    static int getCount() { return count; }
}
```

- **`main` is static** because the JVM calls it without creating an object.

---

## 6. Abstract Class vs Interface ⭐ (a signature Java question)

This distinction is asked far more in Java than in C++, because they're separate language features.

**Abstract class** (`abstract` keyword):
- Cannot be instantiated.
- Can have both abstract methods (no body) and concrete methods (with body).
- Can have constructors, fields (any access), and static/instance state.
- A class extends **only one** abstract class.
- Represents an "is-a" relationship with shared code.

**Interface** (`interface` keyword):
- A contract of methods a class must implement.
- A class can `implement` **many** interfaces (this is how Java gets "multiple inheritance of type").
- Fields are implicitly `public static final` (constants).
- Since **Java 8**: can have `default` and `static` methods (with bodies). Since **Java 9**: can have `private` methods.
- Methods are implicitly `public abstract` (before default/static).

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Instantiate | No | No |
| Multiple inheritance | One only | Many |
| Fields | Any kind | `public static final` constants |
| Methods | Abstract + concrete | Abstract + default/static (Java 8+) |
| Constructor | Yes | No |
| Use when | Sharing code among closely related classes | Defining a capability/contract for unrelated classes |

**Rule of thumb:** use an **abstract class** for an "is-a" with shared implementation; use an **interface** for a "can-do" capability (e.g., `Comparable`, `Runnable`, `Serializable`) across unrelated classes.

---

## 7. The `Object` Class & Key Overridable Methods

Every Java class extends `Object`. Three methods you're often expected to override:

- **`toString()`** — returns a string representation of the object (for logging/printing).
- **`equals(Object o)`** — defines logical equality (default compares references). Override to compare by content.
- **`hashCode()`** — returns an int hash. **Contract:** if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must also be true. Always override `hashCode()` when you override `equals()`, or hash-based collections (`HashMap`, `HashSet`) break.

**`==` vs `.equals()` (extremely common Java question):**
- `==` compares **references** (whether they're the same object in memory) for objects, and values for primitives.
- `.equals()` compares **logical content** (as defined by the class).
- Example: two different `String` objects with the same text are `.equals()` but may not be `==`.

---

## 8. `final`, `this`, and `super`

- **`final` variable** — a constant (can't be reassigned).
- **`final` method** — cannot be overridden.
- **`final` class** — cannot be extended (e.g., `String` is final).
- **`this`** — reference to the current object; also `this()` calls another constructor.
- **`super`** — reference to the parent; `super.method()` calls the parent's method, `super()` calls the parent's constructor.

**Immutability:** an immutable object's state can't change after creation (e.g., `String`). Achieved by making fields `private final`, providing no setters, and defensively copying mutable inputs. Immutable objects are thread-safe and safe as map keys.

---

## 9. Relationships Between Classes

- **Association** — a general "uses-a" relationship between independent objects (Teacher–Student).
- **Aggregation** — "has-a" where the part *can exist independently* of the whole (a Department has Professors; professors survive if the department closes). Weak ownership.
- **Composition** — "has-a" where the part *cannot exist without* the whole (a Car has an Engine created and destroyed with it). Strong ownership.
- **Inheritance** — an "is-a" relationship (`extends`).

**Composition vs Inheritance (design principle):** Prefer **composition over inheritance**. Inheritance creates tight coupling and fragile hierarchies; composition (holding a reference to another object) is more flexible. Use inheritance only for genuine "is-a" relationships that satisfy the Liskov principle.

---

## 10. SOLID Principles (senior-level bonus)

1. **S — Single Responsibility** — a class should have one reason to change (one job).
2. **O — Open/Closed** — open for extension, closed for modification.
3. **L — Liskov Substitution** — a subclass should be usable anywhere its superclass is expected, without breaking behavior.
4. **I — Interface Segregation** — prefer many small, specific interfaces over one large one (very natural in Java).
5. **D — Dependency Inversion** — depend on abstractions (interfaces), not concrete classes.

---

## 11. OOP vs Procedural Programming

- **Procedural** (C) — code as functions operating on separate data; top-down; harder to scale.
- **OOP** (Java) — code as objects bundling data and behavior; bottom-up; supports encapsulation, inheritance, polymorphism; better for large systems.

---

## 12. Interview Q&A Bank (self-test)

**Q: What are the four pillars of OOP?**
Encapsulation (bundling data + methods and hiding internal state), Abstraction (hiding complexity, exposing essentials), Inheritance (a subclass acquiring a superclass's members), and Polymorphism (one interface, many forms).

**Q: Class vs Object?**
A class is a blueprint of fields and methods; an object is an instance of it created with `new`, holding actual values.

**Q: Encapsulation vs Abstraction?**
Encapsulation is data hiding — private fields with controlled getters/setters (implementation level). Abstraction is hiding complexity and exposing only essentials via abstract classes/interfaces (design level).

**Q: Does Java support multiple inheritance?**
Not for classes — a class can only `extends` one class, to avoid the diamond problem. Java supports multiple inheritance of *type* through interfaces: a class can `implement` many interfaces.

**Q: How does Java handle the diamond problem?**
By disallowing multiple class inheritance. Since Java 8, interfaces can have default methods, so if two implemented interfaces have the same default method the compiler forces you to override it explicitly, and you can choose one with `InterfaceName.super.method()`.

**Q: Overloading vs Overriding?**
Overloading is compile-time — same method name, different parameter lists, in one class. Overriding is runtime — a subclass redefines a superclass method with the same signature, resolved by dynamic dispatch. Java has no operator overloading.

**Q: Are Java methods virtual by default?**
Yes. All non-static, non-final, non-private methods participate in dynamic dispatch automatically — there's no `virtual` keyword. You use `final` to prevent overriding.

**Q: What is method hiding?**
When a subclass declares a static method with the same signature as the parent's static method. Static methods aren't overridden (they're bound at compile time to the class), so this is hiding, not overriding.

**Q: Does Java have destructors?**
No. Memory is reclaimed by the garbage collector. The old `finalize()` is deprecated and unreliable; for resource cleanup use try-with-resources with `AutoCloseable`.

**Q: What is garbage collection?**
Automatic memory management where the JVM reclaims memory of objects no longer reachable/referenced, so the programmer doesn't manually free memory.

**Q: Abstract class vs Interface?**
An abstract class can have constructors, fields, and both abstract and concrete methods, and a class extends only one. An interface is a contract; a class can implement many, its fields are constants, and since Java 8 it can have default/static methods. Use an abstract class for shared code among related classes; an interface for a capability across unrelated classes.

**Q: `==` vs `.equals()`?**
`==` compares references (same object) for objects and values for primitives. `.equals()` compares logical content as defined by the class. Two Strings with the same text are `.equals()` but may not be `==`.

**Q: Why override hashCode() when you override equals()?**
Because of the contract: equal objects must have equal hash codes. Otherwise hash-based collections like HashMap/HashSet will misbehave (can't find equal objects).

**Q: What are the four access modifiers in Java?**
private (class only), default/package-private (same package), protected (package + subclasses), and public (everywhere).

**Q: Static vs instance members?**
Static members belong to the class and are shared across all objects (one copy, accessible without an object). Instance members belong to each object.

**Q: Why is main() static?**
So the JVM can call it without creating an object of the class.

**Q: What is constructor chaining?**
Calling one constructor from another — `this()` for the same class and `super()` for the parent — to reuse initialization logic.

**Q: What does `final` do?**
On a variable it makes a constant; on a method it prevents overriding; on a class it prevents extension (e.g., String).

**Q: What is an immutable object?**
An object whose state can't change after creation (like String). Built with private final fields, no setters, and defensive copying — making it thread-safe.

**Q: `this` vs `super`?**
`this` refers to the current object (and `this()` calls another constructor in the same class); `super` refers to the parent (and `super()` calls the parent constructor, `super.method()` the parent's method).

**Q: Association vs Aggregation vs Composition?**
Association is a general relationship between objects. Aggregation is "has-a" where the part can exist independently (weak ownership). Composition is "has-a" where the part can't exist without the whole (strong ownership).

**Q: Composition vs Inheritance — which to prefer?**
Prefer composition; it's more flexible and avoids the tight coupling and fragility of deep inheritance hierarchies. Use inheritance only for true "is-a" relationships.

**Q: What are the SOLID principles?**
Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion — five principles for maintainable OOP design.

**Q: What is the Object class?**
The root class every Java class implicitly extends, providing methods like toString(), equals(), and hashCode() that you commonly override.

---

### Quick-recall cheat sheet
- 4 pillars: Encapsulation, Abstraction, Inheritance, Polymorphism.
- Encapsulation = hide data (private + getters/setters); Abstraction = hide complexity (abstract class/interface).
- No multiple class inheritance → use interfaces (multiple `implements`).
- Overloading = compile-time; Overriding = runtime. All methods virtual by default; `final` stops overriding.
- No destructors → Garbage Collection; cleanup via try-with-resources.
- `==` compares references; `.equals()` compares content. Override hashCode() with equals().
- 4 access modifiers: private, default, protected, public.
- Abstract class = one, shared code; Interface = many, a contract (default methods since Java 8).
- Prefer composition over inheritance. Know SOLID.
