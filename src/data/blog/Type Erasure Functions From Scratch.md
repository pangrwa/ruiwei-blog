---
author: Rui Wei
pubDatetime: 2026-04-19T21:29:42+08:00
modDatetime: 2026-04-19T21:35:45+08:00
title: Type Erasure Functions From Scratch
slug:
featured: false
draft: false
tags: [cpp]
description: Type Erasure Functions From Scratch
imageNameKey: type_erasure_functions_from_scratch
csl: vancouver
---

## Table of Contents

## Introduction
Polymorphism refers to the ability of an entity to take on multiple forms. In programming, we often find ourselves the need to store a variety of "things that can be called"—lambdas, functors, and raw function pointers—in a single container, but their types are fundamentally incompatible.

In this article, I walkthrough the idea of *Type Erasure* functions - "forgetting" the specific type of an object while remembering how to interact with it. In this post, we’re going to peel back the layers and implement our own version of a polymorphic function wrapper from scratch, exploring how we can use manual virtual tables and small-buffer optimization to balance flexibility with performance.

## The Need For Type Erasure
In C++, a *callable* is defined as any entity - object, pointer or expression - that can be invoked using the function call syntax: `callable(args)`. I list some examples of such entities.
```cpp
// 1. Regular functions
int add(int a, int b);
function<int(int, int)> f = add;

// 2. Functors - class/struct that overloads the operator() with internal state
struct Adder {
	int operator()(int x, int y) { return y + x; }	
};
Adder add;
function<int(int, int)> f = add;

// 3. Lambda - essentially a simplified functor
auto add = [](int a, int b) { return a * b; };
function<int(int , int)> f = add;

// 4. Member Function Pointer
struct Person {
	int add(int x, int y) { return x + y; }
};
Person p;
function<int(int , int)> f = [&p](int x, int y) { return p.add(x, y); };
```

While these entities behave almost similarly - in a sense that they are all callable, they have different, incompatible types. C++ provides `std::function` that acts as a general-purpose polymorphic function wrapper - essentially type erasing the function types, allowing us to pass any callable with the defined function signature regardless of what the function does under the hood. However, the functions that are assigned under `std::function` during its lifetime must have *implicitly convertible* signatures. Here's what I mean by that - $return\_type(arg_1\_{type, ..})$.
- `int(int)` $<:$ `void(int)`: return type can be discarded
- `void(long)` $<:$ `void(int)`: argument implicitly converted 

Without `std::function`, we can't easily mix lambdas and function pointers in a container. The example below illustrates the significance of `std::function`.
```cpp
// Suppose we want to mix a couple of callbacks in the same container
auto foo = []() {...};
auto boo = []() {...};
Wrapper<decltype(foo)> w1{foo};
Wrapper<decltype(boo)> w2{boo};
// How do we create a parent class over those types such that
vector<Wrapper<??>> wrappers{w1, w2};

// With std::function
vector<std::function<void()> wrappers{w1, w2};
```

Let's attempt to dive deeper into how we can implement this particular behavior. Keep this question in mind: How do we store something of an unknown size and type in a single class?

## How to Store Anything?
In C++, we can represent a function type using this particular syntax: `return_type(arg1_type, arg2_type, ....)`. How do we create a template class that allows us to pass in that particular function signature such that we can create a `std::function` class for that particular function signature?
```cpp
// How can we decompose T (the function signature) into its return type and argument type?
template <typename T>
class Function;

template <typename R, typename... Args>
class Function<R(Args...)> {};

Function<int(int, int)> f;
```

We define a specialized template such that if the compiler sees the type `T: int(int, int)`, it will pattern match it against the specialized template, binding `R = int` and `Args... = {int}`.


### Dynamic Dispatch
The most obvious pattern we can think of to implement type erasure would be *Runtime Polymorphism*. By having multiple distinct types unified through an abstract interface, we can "technically" hide the specific details of the actual type at runtime.  
```cpp
//  suppose we have sub-classes of Shape
vector<Shape*> shapes;
for (auto& shape : shapes) {
	shape.draw();
}
```

Here's what the rough code would look like if we use the aforementioned technique.

```cpp
template <typename R, typename... Args>
class Function<R(Args...)> {
private:
    struct CallableBase {
        virtual R invoke(Args... args) = 0;
        virtual ~CallableBase() = default;
    };

    template <typename F>
    struct Callable : CallableBase {
        F callable;

        Callable(F&& f) : callable(std::forward<F>(f)) {}

        R invoke(Args... args) override {
            return callable(std::forward<Args>(args)...);
        }
    };

    CallableBase* ptr = nullptr;
};
```

Whenever we introduce a new callable into our `Function` wrapper, we simply erase the concrete type of the callable `F` behind a virtual interface.
```cpp
CallableBase* ptr = nullptr;
template <typename F>
Function(F&& f) {
	ptr = new Callable<std::decay_t<F>>(std::forward<F>(f));
}
```

At this point in time, our `Function` instance has no knowledge on the concrete type of the Callable `F` but only recognizes that we have a pointer to an abstracted class type. For our `Function` instance to call the obscured callable, hidden beneath the pointer, we rely on *Dynamic Dispatch* via a vtable.

```cpp
R operator()(Args... args) {
    return ptr->invoke(std::forward<Args>(args)...);
}
```

For every `Callable<F>` type that was instantiated whenever we pass a particular callable of type `F` into `Function`, the compiler generates a virtual table for that particular type which basically contains function pointers to where its actual methods are residing in memory. Every instance of a `Callable<F>` stores a pointer to its corresponding virtual table upon construction.

```bash
[ Function<R(Args...)> ]
         |
         | ptr
         v
+-------------------------+        +---------------------------+
|    Callable<F> object   |        |    vtable (Callable<F>)   |
|-------------------------|        |---------------------------|
|  callable (F)           |  vptr  | [0] invoke ──► Callable<F>::invoke()   
|  vptr ──────────────────┼──────► | [1] ~Callable ──► Callable<F>::~Callable()
+-------------------------+        +---------------------------+
```

The downside to this method is that we rely on heap allocation to store the instance of `Callable<F>`. That means for every new callable that we assign to our `Function` instance, we have to bear the additional cost of allocating memory to store that particular callable. While it's possible to inline some storage space within the `Function` instance - *Small Buffer Optimization*, another downside of runtime polymorphism is that dispatching the correct function call happens at runtime, not only does it introduce additional pointer chasing but it also restricts the opportunity for compilers to optimize our code.


### Manual Virtual Tables
I present another method that completely removes the need for inheritance. Instead of storing an actual object instance that hides the callable, we simply store a pointer to the callable instead. Because we do not rely on the implicit dynamic dispatch functionality to find the correct method to call, we need to manage and define our own "virtual tables" and explicitly dispatch the correct function pointers to process the hidden callable. Think of it this way - In the context of dynamic dispatching, every object has a fixed behavior. When you walk into a restaurant and order a plate of pasta, the menu simply tells you what you will expect to get. However, in the context of manual virtual table, we design and customize the menu ourselves such that for any given person that comes in and order a plate of pasta, the menu can showcase a different behavior for every different person that it comes across. In this case, the person represents the callable that we pass into `Function`, the menu represents the virtual table and what we receive is quite literally what the callable returns after invoking it.

I summarize some of the key differences between vtables and manual virtual tables below.


|Aspect|Virtual tables (OOP)|Manual Vtables (Type Erasure)|
|---|---|---|
|Model|Inheritance-based polymorphism|Behavior-based polymorphism|
|Object|Real derived object|Opaque storage (no real type)|
|Dispatch|Compiler vptr → vtable|User-defined function table|
|Type info|Preserved (class hierarchy)|Erased at runtime|
|Lifetime|Automatic via constructors/destructors|Manual via ops (destroy/move/copy)|
|Flexibility|Limited to base hierarchy|Any callable/type supported|
|SBO support|Not natural|Core feature|
|Performance|1 indirection|1–2 indirections + possible heap|
|Coupling|Tight (requires base class design)|Loose (fully decoupled)|

Let's now take a look at how can we customize our own virtual table.
```cpp
template <typename R, typename... Args>
class Function<R(Args...)> {
private:
	// static - all objects should share the same invoker for the same F signature
	template <typename F>
	static R invoker(void* erased_function, Args... args) {
        if constexpr (!std::is_void_v<R>) {
            return (*static_cast<F*>(erased_function))(args...);
        }
        // discard the return value in F
        (*static_cast<F*>(erased_function))(std::forward<Args>(args)...);
	}

	void* erased_ptr_to_callable;
	R(*invoker_ptr)(void*, Args...);

public:
	template <std::invocable F>
	Function(F&& f) {
		using CleanedF = std::decay_t<F>;
		erased_ptr_to_callable = new CleanedF(std::forward<F>(f));
		invoker_ptr = &invoker<CleanedF>;
	}
	
	R operator()(Args... args) const {
		return invoker_ptr(erased_ptr_to_callable, args...);
	}
};
```

You can think of `invoker` as part of an item in our menu (virtual table). So when a person/callable is passed into our `Function`, we simply store a raw pointer to the callable and adjust our menu to a particular behavior (`invoker_ptr = invoker<CleanedF>`) specifically for this callable type `F`. So in the future, if this person ever decides to order something from our menu (by calling `Function()` ), we simply have the corresponding `invoker_ptr` to provide a certain behavior for this person/callable.


I provide a small example to illustrate how this works under the hood.
```cpp
void foo(int x);
int boo(int x);

Function<void(int)> f = foo;
f = boo;

// When the code above compiles, the assembly code will store two template 
// instantiated invoker function to cater for foo and boo respectively.
static void invoker(void* erased_function, int x);
static int invoker(void* erased_function, int y);
```
When we run the code above at runtime, we write the code that is responsible for changing what `invoker_ptr` points to.

What we covered so far simply forms the basis for how we type erase our callable and how we store our manual virtual table such that we are able to guide our type erased pointer to produce a particular behavior. Here are some questions for you to ponder:
- How do we make a copy of `Function` if we don't know what's under the the type erased pointer?
	-  We simply define another behavior to add into our vtable/menu 
- What is the size of a function pointer?

"We can't know the type at class definition time — so we capture everything we need about the type at assignment time, then throw the type away." - This line perfectly summarizes the intention for this particular technique.

## SBO and  Alignment
Typically the size of a function pointer on a 64-bit system is 8 bytes. Since this particular callable is rather small, we don't actually **always** need to store our callables on the heap. Instead, we can consider inlining our callable with our `Function` instance. This forms the basis of *Small Object Optimization*.

```cpp

template <typename R, typename... Args>
class Function<R(Args...)> {
private:
	inline static constexpr size_t BUFFER_SIZE = 32;
	using uchar = unsigned char;
	uchar soo_buffer[BUFFER_SIZE];

private:
	template <typename F>
	static constexpr bool need_heap_memory() {
		if constexpr (sizeof(F) <= BUFFER_SIZE) {
			return false;
		}
		return true;
	}

	template <std::invocable F>
	Function(F&& f) {
		using CleanedF = std::decay_t<F>;
		if constexpr (need_heap_memory<CleanedF>()) {
			erased_ptr_to_callable = new CleanedF(std::forward<F>(f));
		} else {
			erased_ptr_to_callable = new (soo_buffer) CleanedF(std::forward<F>(f));
		}
		invoker_ptr = &invoker<CleanedF>;
	}
};
```

By inlining a small buffer to our Function object and checking the size of the callable being passed in, we either allocate new memory on the heap or we directly construct the object into the inline buffer. Keep in mind because of SBO, the implementer must remember to properly handle the destruction of the object in the destructor.

Another thing to take note, in C++, every type `T` has an alignment requirement. For a value, `v = alignof(T)`, any object of type `T` must be placed at an address that is a multiple of that value `v`. Otherwise, this can trigger undefined behaviors (On x86, it might work but may cause your program to be slower). I won't dive too deep on how the compiler generates the proper alignment value required by a class/struct, but what we need to know is that `Function` will not know the type it will receive at compile time so our abstraction over the inline buffer has to account for this.

```cpp
// char is 1-byte aligned
uchar soo_buffer[BUFFER_SIZE]
// What if CleanedF is not 1-byte aligned?
erased_ptr_to_callable = new (soo_buffer) CleanedF(std::forward<F>(f));

alignas(std::max_align_t) uchar soo_buffer[BUFFER_SIZE];
```

By using `std::max_align_t`, we force the compiler to make sure that this inline buffer is aligned enough for any standard type.


## Performance Trade-offs
Implementing your own `Function` reveals the hidden costs we pay for abstraction. When choosing between a raw template and a type-erased wrapper, consider the following.

**1. Type erasure overhead**

A raw lambda or a template function is a "transparent" entity to the compiler. Because the type is known at compile time, the compiler can often inline the entire call, effectively reducing the overhead to zero. `std::function` (and our implementation) erases this type. Because the call happens through a function pointer or a virtual table, the compiler usually cannot see "through" the indirection, preventing inlining and certain optimizations.


**2. The Hidden Cost of the Heap**

Our implementation relies on `new` for callables that exceed the `BUFFER_SIZE`. While 32 bytes covers most simple lambdas, any lambda that captures a large context (like a `std::vector` or a large struct by value) will trigger a heap allocation. This is why we use **Small Buffer Optimization (SBO)**—it keeps the "hot path" on the stack as long as the object remains small.


**3. Indirection and Cache Locality**

A manual vtable requires at least one extra pointer chase to find the `invoker` function. If SBO is used, the callable’s data is local to the `Function` object, which is great for the CPU cache. However, if the data is on the heap, you’re looking at two separate memory lookups: one for the `Function` object itself and another for the erased callable data.

**4. Size vs. Flexibility**

By increasing our `BUFFER_SIZE` to 64 or 128 bytes, we could avoid more heap allocations, but every single `Function` instance would then take up that much space on the stack. Choosing the right buffer size is a classic engineering trade-off: do you optimize for the 90% case of small lambdas, or do you pay the memory tax to accommodate the 10% of heavy ones?


## Conclusion

Type erasure is a powerful pattern that bridges the gap between the static nature of C++ templates and the dynamic requirements of modern application design. By moving away from standard OOP inheritance and toward manual virtual tables, we gain granular control over memory layout and dispatch logic.

We’ve seen how to decompose function signatures, manage opaque storage with SBO, and ensure architectural safety through alignment. While `std::function` is a versatile tool, understanding the "manual" way allows you to build specialized wrappers—perhaps a move-only `UniqueFunction` or a non-allocating `FixedFunction`—tailored to the specific constraints of your system.

You can find my trivial implementation of `std::function` over [here](https://godbolt.org/z/9W78G8fzY)



<div id="refs" class="references">
1. Meyers S. *Effective Modern C++: 42 Specific Ways to Improve Your Use of C++11 and C++14*. O'Reilly Media; 2014. <a href="https://www.google.com/search?q=Effective+Modern+C%2B%2B+Scott+Meyers" target="_blank">Google Books</a>
<br>
2. Stack Overflow. How is std::function implemented? <a href="https://stackoverflow.com/questions/18453145/how-is-stdfunction-implemented" target="_blank">https://stackoverflow.com/questions/18453145/how-is-stdfunction-implemented</a>
</div>
