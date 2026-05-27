# The Non-Throwing Swap idiom

A learning doc for someone new to templates. We'll build the idiom from scratch, see what it does for you, and learn why both halves of "the swap idiom" (member + non-member friend) exist.

---

## 1. What it is — in one paragraph

The **Non-Throwing Swap** idiom is a pattern for giving a C++ class a `swap` operation that (a) never throws, (b) does the minimum possible work — usually a few pointer swaps rather than copying any data — and (c) is reachable by **generic code** through Argument-Dependent Lookup (ADL). The pattern has two halves: a `noexcept` *member* `swap` that does the actual work, and a *non-member friend* `swap` in the same namespace that just forwards to it. Together they make your class behave correctly with the standard library (sorting, container reallocation), make exception-safe assignment via copy-and-swap possible, and avoid the silent O(n²) trap where `std::vector` falls back to copying because your move/swap "might throw."

This is C++98-flavoured idiom that became *much* more important from C++11 onwards, once move semantics and `noexcept` arrived.

---

## 2. The problem it solves

You have a class that owns a resource — a heap buffer, an OS handle, anything where copying is expensive or copying twice would be a bug:

```cpp
class Buffer {
    int* data_;
    std::size_t size_;
    // ... ctor/dtor that new[] and delete[] ...
};
```

You want three things to work cheaply and safely:

1. **Assignment.** `b1 = b2;` should be exception-safe — if anything throws midway, `b1` must still be in a valid state.
2. **Sorting / reordering.** `std::sort(buffers.begin(), buffers.end())` should not copy the heap buffers around; it should swap their internal pointers.
3. **Container growth.** `std::vector<Buffer>` should *move* its elements when it reallocates, not copy them.

Without an explicit swap idiom, all three are either slow, fragile, or both — and the language won't tell you. Especially #3 fails silently.

---

## 3. Naive solutions and why they fall short

### 3.1 Naive: rely on the default `std::swap`

`std::swap(a, b)` is defined roughly as:

```cpp
template <class T>
void swap(T& a, T& b) {
    T tmp = std::move(a);
    a     = std::move(b);
    b     = std::move(tmp);
}
```

For your `Buffer`, that's three moves. **What's wrong:**
- If your move constructor or move assignment is not `noexcept`, the default `swap` can throw — and any code that needs the *strong exception guarantee* (like `std::vector::reserve`) will refuse to call moves, and **copy your buffers instead**, silently turning O(n) into O(n²) plus extra allocations.
- Even when it works, three moves are more work than swapping two pointers.

### 3.2 Naive: a member `swap` and nothing else

```cpp
class Buffer {
public:
    void swap(Buffer& other) noexcept { /* ... */ }
};

void f(Buffer& a, Buffer& b) {
    a.swap(b);              // OK — but this only works if you know the type is Buffer.
}
```

**What's wrong:** generic algorithms can't reach it. The standard library's `std::sort` doesn't write `obj.swap(other)` — it writes `swap(a, b)` (with a `using std::swap;` first), and finds the right overload via ADL. A member-only `swap` is invisible to that lookup.

### 3.3 Naive: specialize `std::swap`

```cpp
namespace std {
    template <> void swap(Buffer& a, Buffer& b) noexcept { a.swap(b); }
}
```

**What's wrong:** adding things to `namespace std` is dangerous and largely forbidden. You're only allowed to specialize a handful of standard templates (like `std::hash`), and `std::swap` isn't one of them in a clean way. The next-better thing is *partial* specialization, but functions can't be partially specialized at all, so for class templates you can't reach this even illegally. The friend-function-in-the-same-namespace approach (§4) is the universally-recommended fix.

---

## 4. The canonical implementation

A minimal, complete `Buffer` showing the full pattern. C++17.

```cpp
#include <algorithm>     // std::swap (for primitive members)
#include <cstddef>
#include <type_traits>
#include <utility>

class Buffer {
public:
    explicit Buffer(std::size_t n = 0)
      : size_(n),
        data_(n ? new int[n]() : nullptr) {}

    ~Buffer() { delete[] data_; }

    // Copy ctor — allocates and copies elements.
    Buffer(const Buffer& other)
      : size_(other.size_),
        data_(other.size_ ? new int[other.size_] : nullptr)
    {
        std::copy(other.data_, other.data_ + other.size_, data_);
    }

    // Move ctor — steal the pointer. Noexcept!
    Buffer(Buffer&& other) noexcept
      : size_(other.size_), data_(other.data_)
    {
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // (A) Copy-and-swap assignment — one operator for both copy and move.
    //     Takes by value: caller's copy/move happens at the call site.
    Buffer& operator=(Buffer other) noexcept {       // <-- relies on swap
        swap(other);
        return *this;
    }

    // (B) Member swap — does the real work. Pointer swaps only, so noexcept.
    void swap(Buffer& other) noexcept {
        using std::swap;                              // <-- the "std two-step"
        swap(size_, other.size_);
        swap(data_, other.data_);
    }

    // (C) Non-member friend swap — the ADL hook for generic code.
    friend void swap(Buffer& a, Buffer& b) noexcept { a.swap(b); }

private:
    std::size_t size_ = 0;
    int*        data_ = nullptr;
};

// Make the contract checkable — if a future refactor accidentally drops
// the noexcept, this static_assert fires before any test runs.
static_assert(std::is_nothrow_swappable_v<Buffer>);
static_assert(std::is_nothrow_move_constructible_v<Buffer>);
```

Three pieces to notice before we dissect:

- **(B)** does the actual swap — pointer-only, no allocation, `noexcept`.
- **(C)** is a one-line wrapper that lets ADL find your version. The class needs *both*.
- **(A)** uses **(B)** to give you exception-safe assignment for free.

---

## 5. Idiom-by-idiom deep dive

### 5.1 The member `swap(T&) noexcept` (where the work happens)

**What it is:** a member function that exchanges the *internal state* of `*this` with another instance, without ever copying or allocating. For a resource-owning class this means swapping the pointers / handles, not the data they refer to.

**In our example:**
```cpp
void swap(Buffer& other) noexcept {
    using std::swap;
    swap(size_, other.size_);
    swap(data_, other.data_);
}
```

**Without it,** you'd be at the mercy of `std::swap`'s default move/move/move dance — which works only if your move members are themselves `noexcept` and which costs three moves instead of two pointer exchanges.

```cpp
// Naive: no swap member; rely on std::swap.
// Buffer a, b;
// std::swap(a, b);   // expands to: Buffer tmp = std::move(a);
//                    //             a = std::move(b);
//                    //             b = std::move(tmp);
//                    // three move-constructors + three null-out operations.
```

**Why here:** swapping two `int`s and two pointers is unconditionally cheap and cannot throw. The compiler will reduce this to a handful of register moves. The fact that it is *unconditionally* `noexcept` is what unlocks safe use inside `operator=` and inside `std::vector` reallocation. (C++11 for `noexcept`; the idiom predates C++11 but became sharper with it.)

📖 [Wikibooks — *More C++ Idioms / Non-throwing swap*](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap)

### 5.2 The non-member friend `swap` (the ADL hook)

**What it is:** a free function `swap(T&, T&)` declared inside the class as `friend`. It does no work — it just calls the member. Its job is to live in the class's namespace so that **Argument-Dependent Lookup** finds it. ADL is the rule by which `swap(a, b)` looks for `swap` not only in the current scope but also in the namespace(s) of `a` and `b`'s types.

**In our example:**
```cpp
friend void swap(Buffer& a, Buffer& b) noexcept { a.swap(b); }
```

**Without it,** generic code that writes the canonical `using std::swap; swap(a, b);` will *only* find `std::swap` (because `swap` isn't a member and there is no `swap(Buffer&, Buffer&)` in `Buffer`'s namespace). It will fall back to the default move-based implementation — see 3.1 for what that costs.

```cpp
// Without the friend version: generic code falls back to std::swap.
template <class T>
void rotate_pair(T& a, T& b) {
    using std::swap;       // bring std::swap into overload set
    swap(a, b);            // ADL also looks in T's namespace
                           // - finds your friend swap if you provided one
                           // - else only std::swap, which uses move/move/move
}
```

**Why here:** the entire standard library uses the "using std::swap; swap(x, y);" two-step. To plug your class into it, you must put a `swap` in your class's namespace. Doing so as a `friend` inside the class definition has two bonuses: it can touch privates (in case you don't want a member `swap`), and it's only visible via ADL (it won't pollute name lookup elsewhere).

📖 [cppreference — Argument-dependent lookup](https://en.cppreference.com/w/cpp/language/adl)

### 5.3 `noexcept` on both versions

**What it is:** an exception-specification promising the function won't throw. It is checkable by `noexcept(expr)` and by `std::is_nothrow_swappable_v<T>`. The C++ standard library uses these traits to decide whether to use moves (during `vector` reallocation, for example) or fall back to copies.

**In our example:**
```cpp
void swap(Buffer&)             noexcept;            // member
friend void swap(Buffer&, Buffer&) noexcept;        // friend

static_assert(std::is_nothrow_swappable_v<Buffer>);
```

**Without it,** the trait `std::is_nothrow_swappable_v<Buffer>` is `false`. `std::vector<Buffer>::reserve` and friends will see this and use the **copy** path during reallocation rather than moving — silently. Your O(n) reallocation becomes O(n²).

```cpp
// Naive: forgot noexcept
void swap(Buffer& other) { /* same body */ }
// std::is_nothrow_swappable_v<Buffer>  ==  false
// std::vector<Buffer>::push_back triggering a reallocation copies elements.
```

**Why here:** if the body genuinely cannot throw — and a pointer-swap *cannot* throw — then telling the compiler so is free. Forgetting it is the single most common silent perf bug in resource-owning classes. (`noexcept` is C++11.)

📖 [cppreference — `noexcept` specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec) · [`std::is_nothrow_swappable`](https://en.cppreference.com/w/cpp/types/is_swappable)

### 5.4 The "`using std::swap; swap(...)`" pattern (the std two-step)

**What it is:** the canonical way to invoke swap from generic code. You bring `std::swap` into scope as a *fallback* with `using std::swap;`, then make the call as an unqualified `swap(a, b);`. Unqualified lookup will combine the using-declared `std::swap` with any ADL-found `swap` in `a`'s/`b`'s namespace, and overload resolution picks the more specific one. Result: your class's `swap` wins if present; otherwise `std::swap` is used.

**In our example:**
```cpp
void swap(Buffer& other) noexcept {
    using std::swap;
    swap(size_, other.size_);       // size_ is std::size_t -> picks std::swap
    swap(data_, other.data_);       // data_ is int*       -> picks std::swap
}
```

**Without it,** you have to choose between `std::swap` (works for built-ins, but skips any user-defined `swap` in scope) or unqualified `swap` (which would find an ADL `swap` if one exists, but won't find `std::swap` for built-ins). The two-step does both.

```cpp
// Naive: call std::swap directly
std::swap(data_, other.data_);   // OK for int*, but if data_ were of a user type
                                 // it would ignore that type's specialized swap.

// Naive: call swap unqualified, no using-declaration
swap(data_, other.data_);        // For int*, no ADL associated namespace, so
                                 // overload resolution fails — compile error.
```

**Why here:** the bodies of generic functions in the standard library are full of this two-step. Doing it yourself, even in a leaf class, costs nothing and makes your implementation future-proof against members switching to user-defined types. (C++98.)

📖 [cppreference — `std::swap`](https://en.cppreference.com/w/cpp/algorithm/swap)

### 5.5 Connection: copy-and-swap assignment built on top

**What it is:** the by-value assignment operator that takes its rhs by value and then swaps. The "copy or move into the parameter" step is delegated to overload resolution at the call site — if the caller passes an lvalue, the parameter is copy-constructed; if it passes an rvalue, the parameter is move-constructed. Either way, you body-swap.

**In our example:**
```cpp
Buffer& operator=(Buffer other) noexcept {   // 'other' is constructed from rhs
    swap(other);                              // pointers swapped in
    return *this;
}                                             // 'other' destroyed -> frees the *old* data
```

**Without it,** you'd write separate `operator=(const Buffer&)` and `operator=(Buffer&&)` and have to think about: self-assignment, exception safety during allocation, and code duplication.

```cpp
// Naive: explicit copy assignment without swap
Buffer& operator=(const Buffer& other) {
    if (this == &other) return *this;       // self-assignment guard
    int* tmp = other.size_ ? new int[other.size_] : nullptr;   // can throw
    std::copy(other.data_, other.data_ + other.size_, tmp);
    delete[] data_;                          // commit
    data_ = tmp;
    size_ = other.size_;
    return *this;
}
```

That works but is fiddly. The copy-and-swap version is three lines and provides the *strong exception guarantee* (`*this` is untouched until the swap, which can't throw) for free.

**Why here:** this is the headline payoff of having `swap` at all. Without `swap`, exception-safe assignment is a chore; with `swap`, it's a one-liner. (C++98 idiom; C++11+ enables move-into-parameter, so the same operator also handles moves.)

📖 [Wikibooks — *More C++ Idioms / Copy-and-swap*](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-and-swap)

---

## 6. Variations

### 6.1 Friend free function vs. namespace-scope free function

The friend version (inside the class) and a non-friend free function in the same namespace are *functionally equivalent* for ADL. The friend form has the edge:

- It can access privates (useful if you forgo the member `swap` and put the body in the friend directly).
- It's "hidden friend" — only findable via ADL, not via normal name lookup, so it doesn't show up in unrelated overload resolution and won't slow compilation by participating in lookups it has no business in.

```cpp
// Equivalent for ADL, but the namespace version pollutes ordinary lookup:
namespace mylib {
    class Buffer { ... };
    void swap(Buffer& a, Buffer& b) noexcept { /* ... */ }
}
```

The friend-in-class form is the modern recommendation.

### 6.2 Pure-friend with no member swap

If you don't expect users to ever call `obj.swap(other)`, you can drop the member version and put the body in the friend:

```cpp
class Buffer {
    // ... no member swap ...
    friend void swap(Buffer& a, Buffer& b) noexcept {
        using std::swap;
        swap(a.size_, b.size_);
        swap(a.data_, b.data_);
    }
};
```

Pro: one function instead of two. Con: `b.swap(other)` doesn't compile, which some style guides require for parity with the standard containers (every standard container has both forms).

### 6.3 Member-only, no friend

Some classes provide only a member `swap`. This is legal and not always wrong — see §8. The downside is that generic code that uses the std two-step has to fall back to `std::swap`'s move/move/move, which for a resource type might still be acceptable if the move members are themselves `noexcept`. But it's strictly worse than providing both halves, with no compile-time benefit.

---

## 7. Common pitfalls

- **Forgetting `noexcept`.** This is the big one. Without `noexcept` on swap *and* move, `std::vector` quietly copies your objects during reallocation. Always add `static_assert(std::is_nothrow_swappable_v<T>);` next to the class to enforce the contract.
- **Providing only the member `swap`.** Generic code finds it via ADL only if you also have a free function in the class's namespace.
- **Specializing `std::swap`.** Don't. Put a `swap` in your own namespace as a friend; the std two-step will find it.
- **Calling `std::swap(a, b)` qualified inside generic code.** That bypasses ADL and forces the default. Use `using std::swap; swap(a, b);`.
- **Swapping by copying.** If your `swap` body does anything that allocates or can throw, you've defeated the entire point. The non-throwing swap is about exchanging *pointers*, not data.
- **Inheritance.** `swap` is not virtual. Swapping through a base reference will slice — usually you want swap only on leaf types, never on polymorphic bases.
- **Self-swap.** If your member swap is `swap(*this)`, the pointer-swap dance does nothing weird. But if you're swapping members that don't tolerate self-aliasing (rare for primitives), guard with `if (&other == this) return;`.

---

## 8. Where this appears in the wild

In the standard library — practically every value-semantic class:
- **`std::vector`, `std::string`, `std::map`, `std::set`, …** — all have both a member `swap` and a friend `swap`. All are `noexcept(/* member-swap-is-noexcept */)`.
- **`std::unique_ptr`, `std::shared_ptr`** — same pattern.
- **`std::optional`, `std::variant`, `std::any`** — same pattern, with `noexcept` conditioned on the wrapped type.

---

## 9. Suggested further reading

Easiest → hardest:

1. **Wikibooks — *More C++ Idioms / Non-throwing swap*** — the canonical write-up of this exact idiom. <https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap>
2. **Wikibooks — *More C++ Idioms / Copy-and-swap*** — the assignment operator on top of swap. <https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-and-swap>
3. **cppreference — `std::swap`** — the algorithm-level overview, including the default implementation. <https://en.cppreference.com/w/cpp/algorithm/swap>
4. **cppreference — `std::is_swappable` / `std::is_nothrow_swappable`** — the traits that the standard library uses to decide between moving and copying. <https://en.cppreference.com/w/cpp/types/is_swappable>
5. **cppreference — Argument-dependent lookup** — the full rules for the lookup that makes the friend swap visible. <https://en.cppreference.com/w/cpp/language/adl>

---

## 10. Quick-reference table

| Idiom                                    | Where in the example (§4)                                         |
|------------------------------------------|-------------------------------------------------------------------|
| Member `swap(T&) noexcept`               | `void swap(Buffer& other) noexcept`                               |
| Non-member friend `swap` (ADL hook)      | `friend void swap(Buffer& a, Buffer& b) noexcept`                 |
| `noexcept` on both                       | both bodies above, plus the `static_assert` at the bottom         |
| Std two-step (`using std::swap; swap()`) | inside the member `swap` body                                     |
| Copy-and-swap assignment                 | `Buffer& operator=(Buffer other) noexcept { swap(other); ... }`   |
| `is_nothrow_swappable_v` static contract | `static_assert(std::is_nothrow_swappable_v<Buffer>);`             |
| Move ctor (`noexcept`) supporting move-into-parameter | `Buffer(Buffer&& other) noexcept`                    |
