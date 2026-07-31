# C Programming — Learn-From-Scratch & Interview Guide (Pointer-Heavy)

> A teaching guide to C for interviews, with a deep focus on **pointers, memory, and the tricky output-prediction questions** that C rounds love. Read the concepts, trace the code examples by hand, then test yourself on the big pointer Q&A and output-prediction sections at the end.
>
> **Why C matters for your interview:** even though your project is in C++, C fundamentals — especially pointers, memory, and arrays — are classic "core concept" questions. C is where you prove you understand what's happening *underneath* the abstractions.

---

## PART A — C FUNDAMENTALS

## 1. What C Is & The Compilation Process

C is a **procedural, statically-typed, compiled** language that sits close to the hardware — it gives you direct memory access via pointers, which is exactly why it's a favorite in interviews and in systems/embedded work (and why HPE, a hardware/infrastructure company, cares about it).

**The compilation pipeline (common question):** source `.c` → **Preprocessor** (handles `#include`, `#define`, macros — produces expanded source) → **Compiler** (translates to assembly) → **Assembler** (assembly → object code `.o`) → **Linker** (combines object files + libraries into the executable). Summarized: **Preprocess → Compile → Assemble → Link.**

```c
#include <stdio.h>       // preprocessor pulls in the header
int main(void) {
    printf("Hello, C\n");
    return 0;            // 0 = success to the OS
}
```

## 2. Data Types, Sizes & Format Specifiers

Sizes are **platform-dependent** (the standard only guarantees minimums and `char` = 1 byte). Typical 64-bit sizes:

| Type | Typical size | `printf` specifier |
|------|-------------|--------------------|
| `char` | 1 byte | `%c` (char), `%d` (as int) |
| `int` | 4 bytes | `%d` / `%i` |
| `unsigned int` | 4 bytes | `%u` |
| `float` | 4 bytes | `%f` |
| `double` | 8 bytes | `%lf` |
| `long` | 8 bytes (Linux 64-bit) | `%ld` |
| `long long` | 8 bytes | `%lld` |
| pointer | 8 bytes (64-bit) | `%p` |

Always use `sizeof(type)` rather than assuming — `sizeof` returns a `size_t` (use `%zu`).

## 3. Storage Classes (common question)

Storage class = scope + lifetime + default value + storage location.

- **auto** — default for local variables; scope is the block; lifetime ends when the block exits; stored on the stack; garbage initial value.
- **register** — a hint to store the variable in a CPU register for speed (compiler may ignore it); you **cannot take its address** with `&`.
- **static** —
  - *local static:* retains its value between function calls (lifetime = whole program), initialized once, default 0.
  - *global static:* limits the variable/function's visibility to the current file (internal linkage).
- **extern** — declares a global variable/function defined in *another* file (external linkage); no storage allocated, just a reference.

```c
void counter(void) {
    static int count = 0;   // keeps its value across calls
    count++;
    printf("%d ", count);
}
// counter(); counter(); counter();  → prints 1 2 3
```

## 4. Functions: Call by Value vs Call by "Reference"

C is **strictly call by value** — arguments are copied. To let a function modify the caller's variable, you pass a **pointer** (this is C's "call by reference," really "call by address").

```c
void swap(int *a, int *b) {   // receives addresses
    int t = *a; *a = *b; *b = t;
}
int x = 3, y = 5;
swap(&x, &y);                 // pass addresses → x=5, y=3
```
If you wrote `void swap(int a, int b)` and swapped inside, the caller's `x` and `y` would be unchanged — because only copies were swapped.

---

## PART B — POINTERS (THE CORE)

## 5. Pointer Basics

A **pointer** is a variable that stores the **memory address** of another variable.

- **`&` (address-of)** — gives the address of a variable.
- **`*` (dereference / indirection)** — gives the *value* at the address a pointer holds.

```c
int x = 10;
int *p = &x;     // p holds the address of x
printf("%d", *p); // 10  → the value at that address
*p = 20;          // changes x through the pointer → x is now 20
```

**Reading a declaration:** `int *p;` means "`p` is a pointer to an int." The type matters: it tells the compiler how many bytes to read on dereference and how far to jump in pointer arithmetic.

## 6. Pointer Arithmetic (a favorite trap)

Pointer arithmetic is **scaled by the size of the pointed-to type.** `p + 1` moves forward by `sizeof(*p)` bytes, not 1 byte.

```c
int arr[3] = {10, 20, 30};
int *p = arr;        // points to arr[0]
p + 1;               // address of arr[1] (moved 4 bytes on a 4-byte int)
*(p + 2);            // 30
```
- Valid pointer operations: `+ int`, `- int`, subtracting two pointers (gives element count between them), comparison.
- You **cannot** add two pointers or multiply/divide pointers.

## 7. Pointers and Arrays (deep — and where interviews attack)

An array name **decays** to a pointer to its first element in most expressions. So `arr` ≈ `&arr[0]`, and these are all equivalent:
```c
arr[i]  ==  *(arr + i)  ==  *(i + arr)  ==  i[arr]   // yes, i[arr] is legal!
```
That last one surprises people: because `arr[i]` is defined as `*(arr+i)`, and addition is commutative, `i[arr]` works.

**But an array is NOT a pointer.** Two key differences:
- `sizeof(arr)` gives the **whole array's size** (e.g., 12 for `int arr[3]`), while `sizeof(p)` gives the **pointer's size** (8 on 64-bit).
- `arr` and `&arr` have the same *address* but different *types*: `arr` is `int*` (after decay), `&arr` is `int(*)[3]`. So `arr + 1` jumps one **int**, but `&arr + 1` jumps one **whole array** (12 bytes).

**The classic gotcha — array decay in functions:** when you pass an array to a function, it decays to a pointer, so `sizeof` inside the function gives the pointer size, not the array size. That's why you must pass the length separately.
```c
void f(int a[]) { printf("%zu", sizeof(a)); }  // prints 8 (pointer!), NOT the array size
```

## 8. Pointer to Pointer (double pointers)

A pointer can point to another pointer.
```c
int x = 5;
int *p = &x;
int **pp = &p;   // pp points to p
printf("%d", **pp);  // 5  (double dereference)
```
Used for: modifying a pointer inside a function (e.g., allocating memory and returning it via a parameter), and arrays of strings (`char **argv`).

## 9. Types of Pointers You Must Know

- **NULL pointer** — points to nothing (`int *p = NULL;`). Always check before dereferencing; dereferencing NULL crashes.
- **Void pointer (`void *`)** — a generic pointer that can hold any type's address but **can't be dereferenced directly** (unknown size); must be cast first. `malloc` returns `void *`.
- **Dangling pointer** — points to memory that has been freed or gone out of scope. Using it is **undefined behavior**.
  ```c
  int *dangle() {
      int local = 10;
      return &local;   // BUG: local dies when function returns
  }
  ```
- **Wild pointer** — an uninitialized pointer holding a garbage address. Initialize pointers (to NULL or a valid address) before use.

## 10. `const` with Pointers (spelled out — commonly confused)

Read declarations **right to left**:
- `const int *p;` — pointer to **const int**: you can't change `*p` (the value), but you can change `p` (point it elsewhere).
- `int * const p;` — **const pointer** to int: you can change `*p` (the value), but you can't repoint `p`.
- `const int * const p;` — const pointer to const int: change neither.

Trick: whatever `const` is *before* the `*` protects the pointed-to value; `const` *after* the `*` protects the pointer itself.

## 11. Function Pointers

A pointer can hold the address of a function, enabling callbacks and jump tables.
```c
int add(int a, int b) { return a + b; }
int (*fp)(int, int) = add;   // fp points to add
int r = fp(2, 3);            // 5   (calling through the pointer)
```
Syntax to remember: `return_type (*name)(param_types)`. The parentheses around `*name` are essential — without them it's a function *returning a pointer*.

## 12. Pointers and Strings

A C string is a `char` array terminated by `'\0'`. A string literal is stored in **read-only memory**.
```c
char arr[] = "hello";   // a modifiable copy on the stack; arr[0]='H' is OK
char *ptr = "hello";     // ptr points to a read-only literal; ptr[0]='H' is UNDEFINED (often crashes)
```
This distinction (array-of-char vs pointer-to-literal) is a very common interview trap.

---

## PART C — MEMORY

## 13. Dynamic Memory Allocation (heap)

Stack memory is automatic and limited; the **heap** is for memory whose size/lifetime you control at runtime. Functions from `<stdlib.h>`:

- **`malloc(size)`** — allocates `size` bytes, **uninitialized** (garbage). Returns `void*` (or NULL on failure).
- **`calloc(n, size)`** — allocates `n*size` bytes, **zero-initialized**.
- **`realloc(ptr, newsize)`** — resizes a previously allocated block (may move it).
- **`free(ptr)`** — releases the memory back to the heap.

```c
int *arr = malloc(5 * sizeof(int));   // don't cast in C; check for NULL
if (arr == NULL) { /* handle failure */ }
for (int i = 0; i < 5; i++) arr[i] = i;
free(arr);        // release
arr = NULL;       // avoid a dangling pointer
```

**Common bugs (interviewers love these):**
- **Memory leak** — allocating and never `free`-ing (lost memory grows over time).
- **Double free** — calling `free` twice on the same pointer (undefined behavior).
- **Use-after-free** — dereferencing a pointer after `free` (dangling).
- **Buffer overflow** — writing past the allocated size.

**malloc vs calloc:** malloc is one argument and leaves memory uninitialized; calloc takes count and size and zero-initializes. calloc is slightly slower due to zeroing.

## 14. Memory Layout of a C Program (great to know)

From low to high addresses:
- **Text (code) segment** — the compiled machine instructions (read-only).
- **Initialized data segment** — global/static variables with an explicit initial value.
- **BSS** — global/static variables **without** an initializer (zero-initialized).
- **Heap** — grows **upward**; dynamic memory (`malloc`).
- **Stack** — grows **downward**; local variables, function call frames, return addresses.

The heap and stack grow toward each other. Local variables live on the stack (freed automatically on return); `malloc`'d memory lives on the heap (freed only by `free`).

---

## PART D — AGGREGATES

## 15. Structures, Unions, Enums

**Structure (`struct`)** — groups variables of different types; each member has its **own** memory.
```c
struct Point { int x; int y; };
struct Point p = {3, 4};
p.x;             // direct access
struct Point *pp = &p;
pp->y;           // arrow: access through a pointer (== (*pp).y)
```
**Structure padding** — the compiler inserts padding bytes so members are aligned, so `sizeof(struct)` may exceed the sum of members. (A common "why is sizeof bigger than expected?" question.)

**Union** — like a struct, but all members **share the same memory** (size = largest member). Only one member is meaningfully valid at a time. Used to save memory or interpret the same bytes different ways.
```c
union U { int i; char c; };   // sizeof = 4 (largest member)
```
**struct vs union:** struct members have separate storage (size = sum + padding); union members share storage (size = largest member).

**Enum** — named integer constants (default start at 0, incrementing).
```c
enum Color { RED, GREEN, BLUE };   // RED=0, GREEN=1, BLUE=2
```
**typedef** — creates a type alias: `typedef struct Point Point;` then just `Point p;`.

---

## PART E — TRICKY TOPICS & C vs C++

## 16. Things That Trip People Up

- **`sizeof` is compile-time** (except VLAs) and returns `size_t`.
- **Integer promotion & implicit conversion** — smaller types promote to `int` in expressions; mixing signed/unsigned can surprise you.
- **Operator precedence with pointers:** `*p++` vs `(*p)++` vs `*++p` (see output questions).
- **Dangling/wild pointers and undefined behavior (UB)** — UB means anything can happen; don't rely on "it worked on my machine."
- **`#define` macros are text substitution** — no type checking; wrap in parentheses: `#define SQ(x) ((x)*(x))` to avoid precedence bugs.
- **`i = i++ ;`** and similar are **undefined behavior** (modifying a variable twice between sequence points).

## 17. C vs C++ (useful since your project is C++)

- C is procedural; C++ adds object-oriented (classes, inheritance, polymorphism) plus generic programming (templates).
- **Memory:** C uses `malloc`/`free`; C++ adds `new`/`delete` (which call constructors/destructors).
- C++ adds references (`&`), function overloading, namespaces, the STL, exceptions, and `bool` as a built-in.
- C has no classes, no overloading, no references, no built-in `bool` (pre-C99).
- Most C code compiles as C++, but they're distinct languages.

---

## PART F — POINTER OUTPUT-PREDICTION QUESTIONS (trace these by hand!)

These are the exact style C interviews use. Cover the answer and predict first.

**Q1.**
```c
int a[] = {1, 2, 3, 4, 5};
int *p = a;
printf("%d %d", *(p + 2), *p + 2);
```
**Answer:** `3 3`. `*(p+2)` dereferences the 3rd element (3). `*p + 2` is `a[0] + 2` = 1+2 = 3.

**Q2.**
```c
int a[] = {10, 20, 30};
printf("%d", 2[a]);
```
**Answer:** `30`. `2[a]` == `*(2 + a)` == `a[2]` == 30 (commutative subscript).

**Q3.**
```c
int a = 5;
int *p = &a;
int **pp = &p;
printf("%d", **pp);
```
**Answer:** `5`. Double dereference gets back to `a`.

**Q4.** (pointer arithmetic on the array address)
```c
int a[5] = {1, 2, 3, 4, 5};
printf("%d %d", *(a + 1), *(&a + 1 - 1));
```
**Answer:** `2 1`. `a+1` → `&a[1]`, value 2. `&a` has type `int(*)[5]`, so `&a + 1` skips the whole array; subtracting 1 (array) returns to `&a`, and `*` decays to `&a[0]`... careful: `*(&a+1-1)` = `*(&a)` which is the array `a`, decaying to `&a[0]`, and the outer nothing dereferences it here — printed with `%d` it's `a[0]`=1. *(This one is deliberately nasty; the point is knowing `&a+1` jumps a full array.)*

**Q5.** `*p++` vs `(*p)++` vs `*++p`
```c
int a[] = {10, 20, 30};
int *p = a;
printf("%d ", *p++);   // prints 10, THEN p moves to a[1]
printf("%d ", *p);     // 20
```
- `*p++` → `*(p++)`: dereference current, then increment the pointer. Prints 10, p now points to 20.
- `(*p)++` → dereference, then increment the **value** pointed to (returns old value).
- `*++p` → increment the pointer first, then dereference (pre-increment).

**Q6.** (const value via pointer to literal)
```c
char *s = "hello";
s[0] = 'H';    // ?
```
**Answer:** **Undefined behavior** — usually a segmentation fault. String literals are read-only. (With `char s[] = "hello";` it would be fine.)

**Q7.** (array decay & sizeof)
```c
int a[10];
printf("%zu %zu", sizeof(a), sizeof(a)/sizeof(a[0]));
```
**Answer (typical 64-bit):** `40 10`. Whole array = 40 bytes; element count = 40/4 = 10. (Inside a function taking `int a[]`, `sizeof(a)` would be 8 — the pointer.)

**Q8.** (pointer difference)
```c
int a[] = {1,2,3,4,5};
int *p = &a[1], *q = &a[4];
printf("%ld", q - p);
```
**Answer:** `3` — pointer subtraction gives the number of *elements* between them, not bytes.

**Q9.** (dangling pointer)
```c
int* f() { int x = 42; return &x; }
// int *p = f(); printf("%d", *p);
```
**Answer:** Undefined behavior — `x` is a local on the stack and is gone after `f` returns. Might print 42, garbage, or crash.

**Q10.** (swap works because of addresses)
```c
void swap(int *a, int *b){int t=*a;*a=*b;*b=t;}
int x=1,y=2; swap(&x,&y);
printf("%d %d", x, y);
```
**Answer:** `2 1`. Passing addresses lets the function modify the originals.

---

## PART G — GENERAL C Q&A BANK

**Q: What is a pointer?**
A variable that stores the memory address of another variable; `&` gets an address, `*` dereferences to the value at that address.

**Q: Why does C use pointers?**
For call-by-reference behavior, dynamic memory management, efficient array/string handling, building data structures (linked lists, trees), and direct hardware/memory access.

**Q: Is C call by value or reference?**
Strictly call by value — arguments are copied. You simulate call by reference by passing pointers (addresses).

**Q: Difference between an array and a pointer.**
An array is a contiguous block whose name decays to a pointer to its first element; but `sizeof(array)` is the whole block while `sizeof(pointer)` is the pointer size, and `&array` has an array-pointer type that jumps the whole array. An array's address is fixed; a pointer can be reassigned.

**Q: What is pointer arithmetic scaling?**
Adding 1 to a pointer advances it by `sizeof` the pointed-to type, not by one byte.

**Q: NULL vs void vs dangling vs wild pointer?**
NULL points to nothing (check before use). void* is a generic pointer that must be cast before dereferencing. A dangling pointer points to freed/out-of-scope memory. A wild pointer is uninitialized with a garbage address.

**Q: malloc vs calloc vs realloc?**
malloc allocates uninitialized memory (one size argument). calloc allocates zero-initialized memory (count and size). realloc resizes an existing block, possibly moving it.

**Q: What is a memory leak and how do you avoid it?**
Allocated heap memory that's never freed, so usage grows over time. Avoid it by pairing every malloc/calloc with a free, and setting pointers to NULL after freeing.

**Q: Difference between the three const-pointer forms.**
`const int *p` = value is const (can't change *p). `int *const p` = pointer is const (can't repoint). `const int *const p` = both const.

**Q: struct vs union?**
Struct members have separate memory (size ≈ sum + padding); union members share the same memory (size = largest member), so only one is valid at a time.

**Q: What is structure padding?**
Compiler-inserted bytes to satisfy alignment requirements, so sizeof a struct can exceed the sum of its members.

**Q: `.` vs `->` for structs?**
`.` accesses a member on a struct object; `->` accesses a member through a pointer to a struct (`p->x` == `(*p).x`).

**Q: What are storage classes?**
auto (default local, stack), register (register hint, no address), static (retains value / file-scope linkage), extern (references a global defined elsewhere).

**Q: What does static do?**
For a local variable, it keeps its value between calls (lifetime = program). For a global variable/function, it limits visibility to the current file.

**Q: Explain the memory layout of a C program.**
Text (code), initialized data, BSS (uninitialized globals, zeroed), heap (grows up, dynamic), stack (grows down, locals and call frames).

**Q: Stack vs heap memory?**
Stack: automatic, fast, limited, freed on function return, stores locals. Heap: manual (malloc/free), larger, flexible lifetime, risk of leaks.

**Q: What is a function pointer used for?**
Storing a function's address to enable callbacks, jump tables, and passing behavior as an argument (e.g., a comparator to qsort).

**Q: What is a dangling pointer? Give an example.**
A pointer to memory that's been freed or gone out of scope, e.g., returning the address of a local variable from a function. Using it is undefined behavior.

**Q: What is undefined behavior?**
Code whose result the C standard doesn't define (e.g., dereferencing a freed pointer, `i = i++`, out-of-bounds access). The program may do anything — crash, produce garbage, or appear to work.

**Q: Why prefer sizeof(var) over a hardcoded size?**
Type sizes are platform-dependent; sizeof keeps code portable and correct if types change.

**Q: What is the difference between `char s[] = "hi"` and `char *s = "hi"`?**
The array is a modifiable copy on the stack; the pointer points to a read-only string literal, so modifying it is undefined behavior.

**Q: What are the steps of compilation in C?**
Preprocessing (macros/includes), compilation (to assembly), assembly (to object code), and linking (combine objects and libraries into an executable).

**Q: How do you pass an array to a function correctly?**
Pass the pointer plus the length separately, because the array decays to a pointer and sizeof won't give the array size inside the function.

---

## PART H — CHEAT SHEET
- Pointer stores an address. `&` = address-of, `*` = dereference.
- Pointer arithmetic scales by `sizeof(*p)`. `arr[i]` == `*(arr+i)` == `i[arr]`.
- Array ≠ pointer: `sizeof(arr)` = whole array; `&arr+1` jumps the whole array. Arrays decay to pointers in functions (pass length!).
- Pointer types: NULL (nothing), void* (generic, cast before deref), dangling (freed/out-of-scope), wild (uninitialized).
- const: before `*` protects value, after `*` protects the pointer. `const int *p`, `int *const p`, `const int *const p`.
- Function pointer: `ret (*fp)(args)`.
- `char *s="x"` = read-only literal (don't modify); `char s[]="x"` = modifiable copy.
- Heap: malloc (uninit), calloc (zeroed), realloc (resize), free (release, then NULL). Bugs: leak, double free, use-after-free, overflow.
- Memory layout: text, data, BSS, heap (up), stack (down).
- struct = separate members (+padding); union = shared memory (size of largest).
- Storage classes: auto, register, static, extern.
- `*p++` (deref then move ptr), `(*p)++` (increment value), `*++p` (move ptr then deref).
- C is call by value; pass pointers for call-by-reference. Compilation: preprocess → compile → assemble → link.
