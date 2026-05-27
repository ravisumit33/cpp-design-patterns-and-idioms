# The Copy-and-Swap Idiom

A learning doc for someone new to C++ templates and resource-managing classes. We'll build the idiom by first writing the buggy/painful version, then the correct-but-tedious version, and finally arriving at the canonical one-operator-handles-everything form.

---

## 1. What it is — in one paragraph

**Copy-and-swap** is a way to write the assignment operator for a class that owns a resource (a heap buffer, a file handle, a socket, anything where copying matters). Instead of clearing the current object and rebuilding it in place, you make a fresh copy of the source first, then *swap* its guts with `*this`, then let the temporary die — taking the old guts with it. This buys you three things at once: (1) **strong exception safety** — if anything throws, `*this` is left untouched; (2) **no code duplication** — the copy constructor does the copy, the destructor does the cleanup, you write neither again; and (3) **a single operator that handles both copy- and move-assignment** (C++11+), because the parameter is passed *by value* and the caller's copy or move constructor decides which path runs.

This is C++98 in origin (Herb Sutter, GotW #59, 1999) but became more powerful with C++11, when move semantics let the same operator handle moves for free.

---

## 2. The problem it solves

You have a class that manages a raw resource — let's say a buffer of `char`:

```cpp
class String {
    std::size_t size_;
    char*       data_;
public:
    String(const char* s);
    String(const String&);
    ~String();
    String& operator=(const String& rhs);   // <-- this is the hard part
};
```

The destructor calls `delete[] data_`. The copy constructor allocates a new buffer and `memcpy`s. Easy. But the **copy-assignment operator** has to do three things in one go:

1. Throw away the resource currently held by `*this`.
2. Acquire a fresh copy of `rhs`'s resource.
3. End up in a sane state even if step 2 throws halfway through.

That last requirement — **strong exception safety**: either the operation succeeds completely, or `*this` is left exactly as it was — is what trips up most hand-written assignment operators. Let's see why.

---

## 3. Naive solutions and why they fall short

### 3.1 Naive: free old, then allocate new

```cpp
String& operator=(const String& rhs) {
    if (this != &rhs) {                       // self-assignment check
        delete[] data_;                       // (1) free old buffer
        size_ = rhs.size_;
        data_ = new char[size_ + 1];          // (2) might throw!
        std::memcpy(data_, rhs.data_, size_ + 1);
    }
    return *this;
}
```

This is broken. If `new char[...]` throws `std::bad_alloc`, we've already deleted the old buffer. `*this` now holds a dangling `data_` pointer to freed memory, and the destructor will double-delete it later. We violated strong exception safety: we destroyed state before we knew the operation would succeed.

### 3.2 Better: allocate first, then free

```cpp
String& operator=(const String& rhs) {
    if (this != &rhs) {
        char* new_data = new char[rhs.size_ + 1];  // (1) might throw — fine, *this untouched
        std::memcpy(new_data, rhs.data_, rhs.size_ + 1);
        delete[] data_;                            // (2) only happens if (1) succeeded
        data_ = new_data;
        size_ = rhs.size_;
    }
    return *this;
}
```

This is actually exception-safe and correct. So why isn't it the end of the story? Three reasons:

- **Code duplication.** The allocate-and-memcpy logic is the same as what the copy constructor already does. The cleanup logic is the same as what the destructor already does. You've now written each piece twice — and if the class gains a new member (say a `capacity_`), you have to remember to update both the copy constructor *and* the assignment operator. Stay vigilant or rot creeps in.
- **You still need a separate move-assignment operator** for C++11. That's another ~10 lines and another opportunity for the new-member-forgot-to-update bug.
- **Self-assignment check is a footgun.** You wrote it because step 1 was destructive. If step 1 isn't destructive, the check is wasted CPU on every assignment for a case that almost never happens.

### 3.3 Could we just `=default`?

If `String` held a `std::string` instead of `char*`, then yes — the compiler-generated copy-assignment would be correct, because `std::string` itself handles the resource. This is the **Rule of Zero**: prefer member types that manage their own resources, and you write no special members at all.

But the copy-and-swap idiom is for the case where you *can't* push the resource down into a member — you *are* the wrapper. Container authors, smart-pointer authors, OS-handle wrappers — they need this.

---

## 4. The canonical implementation

Here is the entire idiom in one self-contained file. Read the comments marked `// <--` carefully; those are the four moves that make this idiom what it is.

```cpp
#include <algorithm>   // (only if you swap containers; not used directly here)
#include <cstddef>
#include <cstring>
#include <utility>     // std::swap (since C++11)

class String {
    std::size_t size_;
    char*       data_;

public:
    // --- normal special members ----------------------------------------

    explicit String(const char* s = "")
      : size_(std::strlen(s)),
        data_(new char[size_ + 1]) {
        std::memcpy(data_, s, size_ + 1);
    }

    // Copy constructor — allocates fresh storage and copies bytes.
    String(const String& other)
      : size_(other.size_),
        data_(new char[size_ + 1]) {
        std::memcpy(data_, other.data_, size_ + 1);
    }

    // Move constructor — steals the source's buffer (C++11+).
    String(String&& other) noexcept
      : size_(other.size_),
        data_(other.data_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    ~String() { delete[] data_; }

    // --- the copy-and-swap assignment ----------------------------------

    // ONE assignment operator handles both copy and move.
    // The parameter is passed BY VALUE — the compiler picks copy- or
    // move-constructor for `other` based on the call site.
    String& operator=(String other) noexcept {   // <-- (1) by value
        swap(*this, other);                       // <-- (2) the swap is the assignment
        return *this;
    }                                             // <-- (3) `other` dies here, taking the OLD bytes with it

    // Non-throwing friend swap, ADL-discoverable.
    friend void swap(String& a, String& b) noexcept {   // <-- (4) noexcept is mandatory
        using std::swap;          // canonical form even when only built-ins are swapped
        swap(a.size_, b.size_);
        swap(a.data_, b.data_);
    }
};
```

That's it — about 30 lines, and it correctly handles:

- `s1 = s2;` (copy-assignment from lvalue)
- `s1 = make_string();` (move-assignment from rvalue)
- `s1 = s1;` (self-assignment — wasteful but correct, no crash)
- `s1 = String("hi");` (assignment from temporary — moves into `other`)

No `if (this != &rhs)`. No duplicate alloc/memcpy logic. No second function for move-assignment.

---

## 5. Walk through the implementation, idiom by idiom

### 5.1 Pass-by-value parameter

**What it is:** When you declare a function parameter by *value* (not by `const&`, not by `&&`), the language requires the caller to construct that parameter — by copy-constructing it from an lvalue, or by move-constructing it from an rvalue. Since C++11, the compiler picks whichever is appropriate for free.

**Where in the code:** (above)
```cpp
String& operator=(String other) noexcept {
    //          ^^^^^^ — by value
```

**Without it:** A pre-C++11 version of copy-and-swap had to take `const String& rhs` and construct the copy inside the body:
```cpp
String& operator=(const String& rhs) {
    String tmp(rhs);     // explicit copy
    swap(*this, tmp);
    return *this;
}
```
This works, but it forces every assignment — including from an rvalue — to make a full copy. The pass-by-value form gets move-assignment for free: when the caller writes `s = make_string()`, the move constructor builds `other` directly from the rvalue, no copy.

**Why here:** This is what unifies the two assignment operators into one. The compiler, not you, picks copy vs. move. And — crucially — if the *copy construction of `other`* throws, it throws *before* `swap` runs, so `*this` is untouched. That's where the strong exception safety comes from.

📖 [cppreference: copy elision and value categories](https://en.cppreference.com/w/cpp/language/copy_elision)

### 5.2 Non-throwing `swap`

**What it is:** A function that exchanges the contents of two objects of the same type without copying their resources — typically by swapping a few pointers and primitive members. Marked `noexcept` because it must not throw: half of the idiom's safety guarantee depends on this.

**Where in the code:**
```cpp
friend void swap(String& a, String& b) noexcept {
    using std::swap;
    swap(a.size_, b.size_);
    swap(a.data_, b.data_);
}
```

**Without it:** You could write a member `void swap(String& other)` instead:
```cpp
void swap(String& other) noexcept {
    std::swap(size_, other.size_);
    std::swap(data_, other.data_);
}
```
That works for direct calls (`a.swap(b)`), but generic library code reaches `swap` via *Argument-Dependent Lookup* — the "std two-step" — and a member function isn't found that way. A free function in the same namespace as `String` is.

**Why here:** Two reasons. (1) `friend` lets `swap` touch the private members `size_` and `data_`. (2) Putting `swap` at namespace scope makes it visible to ADL, which is how `std::sort`, `std::vector`'s internals, and other generic code find it. Worth its own deep dive — see the related doc on the non-throwing swap idiom (`non-throwing-swap.md` in this folder).

📖 [cppreference: std::swap](https://en.cppreference.com/w/cpp/algorithm/swap)
📖 [Wikibooks: Non-throwing swap](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap)

### 5.3 `using std::swap;` + unqualified call

**What it is:** A two-line dance that says "try ADL first, then fall back to `std::swap`." You write `using std::swap;` to bring the standard one into scope, then call `swap(x, y)` *unqualified* — the compiler picks whichever overload is the best match: a user-defined `swap` if argument-dependent lookup finds one, otherwise `std::swap`.

**Where in the code:**
```cpp
using std::swap;
swap(a.size_, b.size_);
swap(a.data_, b.data_);
```

**Without it:** You could write `std::swap(a.size_, b.size_)` directly. That works *here*, because `size_` and `data_` are built-in types and `std::swap` is the only option anyway. But if you ever swap a member that has its own `swap` (e.g. a `std::vector<int>`), `std::swap` falls back to three move-constructions, while the type's own `swap` is usually a three-pointer trick. Hard-coding `std::` denies you that optimisation.

**Why here:** It's the canonical, copy-pasteable form. Writing it the same way every time means your code is robust against the member types growing into ones that have their own `swap`. (Some style guides drop the `using` when all members are primitives — both are defensible; the `using` is the safer default.)

📖 [Wikibooks: Argument-Dependent Lookup ("Koenig lookup")](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap)

### 5.4 `noexcept` on `swap` (and on `operator=`)

**What it is:** `noexcept` is a compile-time promise — and partially a runtime guarantee — that a function will not throw an exception. Marked since C++11. The standard library checks `noexcept`-ness with `std::is_nothrow_swappable` / `std::is_nothrow_move_constructible` and uses cheaper code paths when it sees them.

**Where in the code:**
```cpp
String& operator=(String other) noexcept { ... }
friend void swap(String& a, String& b) noexcept { ... }
```

**Without it:** If `swap` isn't `noexcept`, the strong exception safety guarantee collapses — if `swap` could throw mid-exchange, `*this` could end up half-swapped. And containers like `std::vector` will refuse to use your move constructor when reallocating; they'll copy instead, silently turning O(n) into O(n) copies. The performance regression is invisible at the call site.

**Why here:** Swap and assignment are the two operations whose `noexcept`-ness has the biggest downstream impact on user code. The copy-and-swap idiom only works *as advertised* when both are marked.

📖 [cppreference: noexcept specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec)

### 5.5 The destructor of the by-value parameter

**What it is:** Not really an "idiom" you write — it's a consequence of pass-by-value. When `operator=` returns, the parameter `other` goes out of scope, and `~String()` runs on it. The bytes inside `other` at that point are the *old* bytes that used to belong to `*this` (because `swap` exchanged them).

**Where in the code:**
```cpp
String& operator=(String other) noexcept {
    swap(*this, other);
    return *this;
}   // <-- `other`'s destructor runs here, freeing the OLD buffer
```

**Without it:** If you wrote the body manually without the swap, you'd have to remember to `delete[]` the old `data_` after copying. That's the cleanup logic you don't want to write twice. The destructor does it for free.

**Why here:** This is the magic that makes the idiom DRY. The copy constructor does the allocation; the destructor does the cleanup; `operator=` does neither directly — it just borrows them. Add a new member to the class and you only update the copy constructor and destructor; the assignment operator keeps working.

📖 [cppreference: Destructor](https://en.cppreference.com/w/cpp/language/destructor)

### 5.6 Rule of Five (implicit)

**What it is:** Since C++11, if you write any one of {destructor, copy ctor, copy assign, move ctor, move assign}, you almost certainly need to write (or `=default`, or `=delete`) all five. They are deeply interrelated.

**Where in the code:** All five exist in the canonical implementation above — the copy and move constructors, the destructor, and the single `operator=` that covers both copy- and move-assignment.

**Without it:** Forgetting the move constructor while writing the copy constructor means C++ won't auto-generate a move constructor for you (the user-declared copy constructor suppresses it). Your class will silently fall back to copies wherever moves were possible — including in container reallocation. The compiler will not warn you.

**Why here:** Copy-and-swap fuses copy- and move-assignment, but you still need the copy *constructor* and move *constructor* as separate entities — the constructors do the work; the assignment operator just borrows from them.

📖 [cppreference: Rule of three/five/zero](https://en.cppreference.com/w/cpp/language/rule_of_three)

---

## 6. Variations

### 6.1 Pre-C++11 form (still seen in older code)

Before move semantics, the parameter is taken by `const&` and the copy happens explicitly inside the body:

```cpp
String& operator=(const String& rhs) {
    String tmp(rhs);
    swap(*this, tmp);
    return *this;
}
```

Functionally identical for lvalue assignment. The C++11 by-value form is strictly better because it also picks up move-assignment for free.

### 6.2 Explicit-copy form for clarity

Some authors prefer to spell out the copy at the call site of `swap` for readability:

```cpp
String& operator=(const String& rhs) & {
    String(rhs).swap(*this);    // construct tmp, then swap *this with it
    return *this;
}
```

This requires a member `swap`. It reads as "make a copy of `rhs`, then swap it with me." Same machine code, slightly different aesthetic — and gives up move-assignment fusion unless you add a separate `operator=(String&&)`.

### 6.3 Skip the swap if you have a `std::string`-style buffer that can be reused

Copy-and-swap **always** allocates new storage. For a class whose buffer could be reused — e.g. you're assigning a 5-byte string into a `String` whose existing buffer is already 100 bytes — a hand-written assignment that *reuses* the old buffer can be faster (no allocation). The std library's `std::string::operator=` does this. The cost is the strong exception safety guarantee — buffer-reusing assignment usually offers only the *basic* guarantee.

This is the trade-off Mathieu Ropert lays out in "Copy and Swap, 20 years later" — see further reading.

### 6.4 The "pass by rvalue reference" variant

You can write two operators explicitly instead of fusing them:

```cpp
String& operator=(const String& rhs) {
    String tmp(rhs);
    swap(*this, tmp);
    return *this;
}
String& operator=(String&& rhs) noexcept {
    swap(*this, rhs);
    return *this;
}
```

This avoids the (tiny) cost of move-constructing `other` when assigning from an rvalue in the by-value form. In practice the compiler usually elides it; in real code the unified form is almost always preferred.

---

## 7. Common pitfalls

### 7.1 Forgetting `noexcept` on swap

The biggest silent footgun. Without `noexcept` on swap and the move operations, `std::vector<String>` will *copy* instead of *move* during reallocation — and `std::sort` of large objects gets dramatically slower. No warning, no error, just bad performance.

### 7.2 Self-assignment performance (not correctness)

`s = s;` with the by-value form will copy `s` into `other`, swap `*this` with `other` (now identical to `*this`), and then destroy `other` (which holds the same bytes that `*this` started with). This works correctly but does a full copy for no reason. If self-assignment is on the hot path, copy-and-swap is the wrong tool.

### 7.3 Writing the swap as a member but not as a friend

A member `void swap(String&)` is fine for direct calls but invisible to ADL. Generic library code that does `using std::swap; swap(a, b);` won't find it; it'll fall back to `std::swap`, which does three move-constructions instead of your three pointer-swaps. Always provide the non-member friend.

### 7.4 Marking `operator=` `noexcept` when the copy ctor can throw

In the by-value form, the parameter `other` is constructed *before* `operator=`'s body runs. If `other` is *copy*-constructed (from an lvalue argument) and the copy ctor throws, an exception escapes the call site — even though `operator=` is marked `noexcept`. This is actually fine: the exception is thrown during *argument construction*, which is not "inside" the noexcept function. But it surprises some readers. Some style guides therefore mark `operator=` non-`noexcept` when fed by a throwing copy. In practice for the canonical form it's correct as written; just understand what's happening.

### 7.5 Forgetting move constructor → copy-and-swap silently degrades

If you only write the copy constructor and not the move constructor, then `operator=(String other)` will *copy* even when called with an rvalue. You lose the move optimisation without warning. Always write the move constructor (or `=default` it) when you write copy-and-swap.

---

## 8. Where this appears in the wild

- **`std::shared_ptr` and `std::unique_ptr`** — their assignment operators are not literally copy-and-swap, but their `swap` member functions follow the non-throwing-swap idiom that copy-and-swap depends on. (See `<memory>`.)
- **Smart-pointer-like wrappers in Boost** (`boost::shared_ptr`, `boost::scoped_ptr`, etc.) historically use the copy-and-swap form for assignment.
- **Many container-of-resources wrappers** in third-party libraries — Eigen, Qt's `QVector` and `QString`, Folly's `fbstring` — provide a non-throwing `swap` and use it internally, even when their actual `operator=` is hand-written for performance (variant 6.3).
- **Herb Sutter's original GotW #59 (1999)** is where the idiom was first systematised; it's referenced in his book *More Exceptional C++* (Addison-Wesley, 2002).

---

## 9. Suggested further reading

Ordered roughly easiest → hardest:

📖 [Wikibooks: More C++ Idioms — Copy-and-swap](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-and-swap) — the canonical short write-up.

📖 [cppreference: Rule of three/five/zero](https://en.cppreference.com/w/cpp/language/rule_of_three) — the broader context for which assignment operator you need to write at all.

📖 [cppreference: copy assignment operator](https://en.cppreference.com/w/cpp/language/copy_assignment) and [move assignment operator](https://en.cppreference.com/w/cpp/language/move_assignment) — language reference for the two operators copy-and-swap fuses.

📖 [Mathieu Ropert: "Copy and Swap, 20 years later" (2019)](https://mropert.github.io/2019/01/07/copy_swap_20_years/) — when the idiom is and isn't the right choice in modern C++.

📖 [Herb Sutter: GotW #59 — Exception-Safe Class Design, Part 1: Copy Assignment](http://www.gotw.ca/gotw/059.htm) — the original article where the idiom was named and popularised.

📖 [Wikibooks: More C++ Idioms — Non-throwing swap](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap) — the sister idiom; copy-and-swap depends on it.

---

## 10. Quick-reference table

| Idiom | Section | Where in the canonical example |
|---|---|---|
| Pass-by-value parameter | 5.1 | `operator=(String other)` |
| Non-throwing friend `swap` | 5.2 | `friend void swap(String&, String&) noexcept` |
| `using std::swap;` + unqualified call (the "std two-step") | 5.3 | Inside the `swap` body |
| `noexcept` on `swap` and `operator=` | 5.4 | Both signatures |
| Destructor of by-value param cleans up old state | 5.5 | The closing `}` of `operator=` |
| Rule of Five (copy ctor, move ctor, dtor, one `operator=`) | 5.6 | The class as a whole |
