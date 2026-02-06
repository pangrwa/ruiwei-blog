---
author: Rui Wei
pubDatetime: 2026-01-29T17:45:48+08:00
modDatetime: 2026-02-06T18:09:22+08:00
title: The C++ Phases of Translation
slug:
featured: false
draft: false 
tags: [c++]
description: The C++ Phases of Translation
imageNameKey: the_c++_phases_of_translation
csl: vancouver
---

## Table of Contents

## Introduction
When writing code with an interpreted language such as Python or JavaScript, we often overlook how our code looks like when it is fed into the CPU. Most of us treat the runtime engine for these languages like a magic box - write code, run and if we are lucky enough, a program comes out. Sometimes, a structural error that exists within our code would never be uncovered until we actually run our code. Compiled languages, such as C/C++, on the other hand helps the developer uncover most violation of the programming language before we actually release our code out.

This article goes through the phases of translation of how a source file transforms into a working program. We first take a look at how the preprocessor tidies up our source file before it gets compiled into machine code. We then take a look at some C++ compilation features. Finally, we take a look at how the linker combines the relevant source files together to produce our program.

## Phases of Translation
The image below summarizes the entire translation process a typical C++ compiler would follow. We go through each of the step in greater detail.
![](../../assets/excalidraw/The%20C++%20Phases%20of%20Translation%202026-02-06%2016.48.16.excalidraw.png)

Before we go through each phase, I'll share a brief introduction on the proper way to write C++  programs. Typically, a developer would declare entities such as classes and functions in a header file and define the functionality of those in a source file. This is primarily because each C++ source file is compiled independently and linked afterwards. One key benefit to this is that if a source file, `math.cpp`, changes, only that file has to be recompiled - everything else just re-links. This matters hugely for larger code bases which affects build time.

## Preprocessor 
Before our source files get compiled, all `.cpp` files go through the preprocessing phase. The most important takeaway from this phase, is the handling of *directives* - lines that start with the `#` character. For example, we would typically include a header file through the syntax `#include <iostream>` which brings the I/O header file into our source file (`.cpp`). The preprocessor will quite literally just paste the content of that header file and dump it into your source file. Suppose that we have the files below, we run `g++ -E -P main.cpp > main.i` to generate the preprocessed file - a translation unit.

```cpp
// a.hpp
int a;

// b.hpp
#include "a.hpp"

// main.cpp
#include "a.hpp"
#include "b.hpp"

// main.i - after preprocessing main.cpp
int a;
int a;
int main() {}
```

Our source file would have two definitions of `a` and hence will result in an error.
```zsh
✦ ❯ g++ main.cpp -o main a.hpp b.hpp
In file included from b.hpp:1,
                 from main.cpp:3:
a.hpp:2:5: error: redefinition of ‘int a’
    2 | int a;
      |     ^
```

This is where we utilize other directives to help prevent the redefinition issue above. The two most popular way, and also hotly debated, is to use either the non-standard `#pragma once` or the traditional header guards `#ifndef`. 

```cpp
// a.hpp
#ifndef A_HPP
#define A_HPP

// V.S
#pragma once
```

Most of the time, `#pragma once` is almost impossible to get wrong but there can still be instances where issues can occur. I give a brief summary on how both directives work under the hood with respect to  `g++`. To summarize, `#ifndef`  works by tracking an independent macro table for each translation unit. It is mostly reliable, if you are able to ensure that your guard macros are unique - this comes from adhering to a standard. On the other hand, `#pragma once` ensures uniqueness based on the physical file identify - file path, *inode*. I give an example where this might fail. 

```cpp
// include/a.hpp
int a;

// a.hpp
int a;

// main.cpp
#include "include/a.hpp"
#include "a.hpp"
```

If we duplicate two header files with the same base file name and content but in different location, `#pragma once` would not work and cause a compilation error.


## Compilation
After the preprocessing step, we take each translation unit and compile to the corresponding assembly code through this command `g++ -S main.cpp -o main.s`. Instead of focusing on how our source code gets translated into assembly - which is the focus on general concepts of compiler design - we focus instead on the features of C++ and its corresponding generated assembly code.

Before we begin, it might be useful to clear up the common misconception between *declarations* and *definitions*.
- A *declaration* introduces the name and type of a variable, function or a class without necessarily allocating memory or implementing it.
- A *definition* actually creates the entity, allocating memory or providing an implementation to it.

Simply put, a declaration implies that there exists and a definition shows where it is. All definitions are declarations, but not all declarations are definitions. I list some examples.
```cpp
// Declarations
int f(int); // function by default comes with external linkage
extern int a; 
class s;

// Definitions
int f(int x) { return x; } // defines both f and x
int a;
```
Because a declaration is just a contract that tells the compiler that this declarator exists, that would mean that it is totally fine to have duplicate declarations in the same translation unit, but it's imperative to only have one definition of it.

With that, I provide definitions to some keywords that could be useful along the way.
- The `extern` keyword specifies external linkage allowing different translation units (source files) to refer to the same declarator. It's basically a promise to the compiler that the entity will exist.
- The `static` keyword when use globally is quite the opposite, instead it means that the declarator is visible within that translation unit only - Internal Linkage. `const` and `constexpr` by default have internal linkage.

### Forward Declaration
 A forward declaration is basically a promise to the compiler that a certain entity (variable, function, class) will exist and will be defined later.  There are other reasons to  use forward declaration, such as reducing compile time, but let's focus on cyclic dependencies. Suppose you have two classes that depend on each other.

```cpp
// a.hpp
#include <b.hpp>
class A {
public:
	B b;
};

// b.hpp
#include <a.hpp>
class B {
public:
	A a;
};

// main.cpp
#include <a.hpp>
```

This is impossible for the compiler to compile the translation unit, `main.cpp`. Since before it can define `A` , it needs to know the size of `B` but this is impossible because `B` needs to know the size of `A` too. This is the classic cyclic dependency that can occur.  To fix this, we can forward declare one of the class, but this comes with the constraint that you can't define that member field by value.

```cpp
// a.hpp
class B;
class A {
public:
	B* b;
	void do_something(B& b);
};

// b.hpp
#include <a.hpp>
class B {
public:
	A a;
};

// main.cpp
#include <a.hpp>
```

In this example, because of `B*`, `A` can be defined since the size of a pointer is fixed and known in compile time. Since forward declaration provides an incomplete type, we can't really use that type in scenarios where we need to know the type's compile size such as creating an instance of the object. Because of this, we are also unable to define the method `A::do_something(B&)` in `a.hpp`, since we have no information on `B` at all. This would force us to define the method in a source file that includes `b.hpp` instead.

### Language Linkage
This feature provides linkage between executables generated in different programming language. Let's focus on how to declare functions in C++ that can be called by C executables. In our C++ header file, we have to wrap the functions and variables that we wish to expose to the C executable with `extern "C"`. This means those functions:
- Won't have C++ name mangling
- Uses your platform/architecture C ABI (calling convention)
- C-compatible symbol names

For example, the following code below in a `.hpp` file, a `.c` source file can include this header file and expect to find an `add(int, int)` function when linked because that function will be compiled as if it was a function written in `C`. Therefore, it is important that the function definition of `add` defined in a `.cpp` source file to use compatible features of the C programming language. I list some examples of C++ features that are not compatible with C:
- Function overloading
- Templates
- `std::string`, etc.
```cpp
// functions.hpp
extern "C" {
	int add(int a, int b);
}
```

Before we continue, let's make use of a Linux tool, `nm`, to help list symbols from object files or binaries. Now suppose, we use the `add` function that is not `extern "C"` in a source file in cpp as shown below and run the following command to generate the corresponding object file: `g++ -c main.cpp`

```cpp
// main.cpp
int add(int a, int b);

int main() {
  int x = add(1, 2);
  return x;
}
```

By using `nm` to inspect the `main.o` object file. We get the following output.
```zsh
✦ ❯ nm main.o
                 U _Z3addii
0000000000000000 T main
```

As you can tell from the output, the C++ compiler compiled `main.o` with the expectation that we provide a reference to the `add` function as denoted by the `U` symbol which means undefined. The name of the function generated by the C++ compiler is mangled by default. Name mangling primary purpose is to distinguish entities, such as functions, that share the same name but have different signatures. In short, name mangling is one of the reason why function overloading is supported in C++.

Instead, if we use the include `functions.hpp` into `main.cpp` to use the `extern "C"` specifier and compile the above file and by using `nm` to inspect the `main.o` object file. We get the following output:

```zsh
✦ ❯ nm main.o
                 U add
0000000000000000 T main
```

This clearly shows the difference in how our compiler generate object files expecting the kind of symbol reference that will be passed during the linker phase. Therefore, by using the `extern "C"` wrapper in the header file in C++, we essentially tell the compiler to treat this function as if it's usable by C source file.

### Templates
Templates are like generics in Java with the exception that there is no type erasure and automatic type casting, instead templates generate unique code based on each template instantiation with different types. A template in C++ is quite literally what it means, a blueprint to generate code. That means that templates are compiled (code generation) on demand.

```cpp
// bar.hpp
template <typename T>
int Bar(T a, T b) {
  return a + b;
}

// main.cpp
int main() {
	int x = Bar<int>(5, 5);
}

```

For example, in our source file we called `Bar<int>` and because we used it in a translation unit, the compiler will look at the template and generate the corresponding code in the translation unit.
```cpp
// main.cpp
template<int>
int Bar(int a, int b) {
	return a + b;
}
```

Another thing to take note, typically template definitions must be visible when the translation unit calls for it - that means the definition must exist in the header file.

You may have already noticed a possible flaw - what if multiple translation units (C++ source file) use `int Bar<int>(int, int)`, wouldn't that lead to unnecessary code generation across all the translation unit? This is where explicit instantiation happens. Suppose we have the code below.

```cpp
// bar.hpp
template <typename T>
int Bar(T a, T b);

// bar.cpp
template <typename T>
int Bar(T a, T b) {
	return a + b;
}

// main.cpp
int main() {
	int x = Bar<int>(5, 5);
	return x;
}
```

If we try to compile `g++ -o main main.cpp bar.cpp`.  Despite having the definition of `Bar<T>` in `bar.cpp`, we will still get an undefined reference of `int Bar<int>(int, int)` simply because: 
1. There is no definition in the `bar.hpp`
2. Despite linking to `bar.cpp` there is no instantiation of `Bar<int>`


Therefore in `bar.cpp` we will have to explicitly instantiate a template generation.
```cpp
// bar.cpp
template <typename T>
int Bar(T a, T b) {
	return a + b;
}

template int Bar<int>(int a, int b);
```
Now if we try to link `main.cpp` and `bar.cpp`, `main.cpp` will now have a resolved reference to `int Bar<int>(int, int)`. Now consider the case below where we also have a definition defined in `bar.hpp`.

```cpp
// bar.hpp
template <typename T>
int Bar(T a, T b) {
	return a + b;
};

// bar.cpp
template <typename T>
int Bar(T a, T b) {
	return 0;
}
template int Bar<int>(int a, int b);

// main.cpp
extern template int Bar<int>(int a, int b);
int main() {
  int x = Bar<int>(5, 5);
  return x;
}
```

Because of `extern template`, we essentially tell `main.cpp` to not generate a template instantiation of `Bar<int>` and we promise that we are able to resolve the reference during the linking phase. Therefore the output in this case would be `0` as the explicit instantiation in `bar.cpp` will take precedence. 

The benefits to explicit instantiation is that we are able to reduce code bloat during compile time generation and is useful for template heavy libraries. However, we restrict ourselves to limited type flexibility as we are only able to use those template types that have been instantiated explicitly, if we want to add a new type, we would have to add that additional line in the source file which brings a maintenance overhead.


## Linker
Once each of our source file, translation unit, gets compiled we can link all of them together to resolve any promises each source file made to the compiler to generate an executable. We first look at the one of the most important rule that our linker checks.

### One Definition Rule (ODR)
Certain C++ entities, such as variables, functions and classes, can only have one definition in any one of the translation unit. If two definitions of an entity such as a class or an inline function exist, then the following conditions must be met for *ODR* to be satisfied:
1. Both definitions appear in different translation unit
2. They are token-for-token identical
3. The meanings of those tokens are the same in both translation units

```cpp
// file1.cpp
struct S { int a; char b; };
// file 2.cpp
struct S { int a; char bb; };
```

The two definitions of `S` do not satisfy the above condition and hence *ODR* is violated and this will lead to a linker error if both source files are linked together.

There are workarounds to this particular rule. One of them is through the use of the `inline` specifier. The original intent of this keyword specifier was to serve as an indicator to the compiler to possibly optimize this function call by copying the function instructions to the caller. However, since *C++98*, `inline` meant that both functions and variables were both allowed to have multiple definitions without causing a linker error. There are some caveats to this though, the standard requires the function or variable declared as `inline` to follow:
1. The definition of the inline function must be present in the header file
2. All definitions of that named entity must be identical and inline in all translation units

If these properties are not met, then the program is ill-formed and may not necessarily have a linker error. Why is this feature present? Templates for example are inline by default. Recall that templates are compiled on demand by each source file. Let's take a look at an example below.

```cpp
// foo.hpp
template <typename T>
int foo(T a, T b) { return a + b; }

// a.cpp
int y = foo<int>(1, 2);

// b.cpp
int x = foo<int>(1, 2);
```

In both `a.cpp` and `b.cpp`, because `foo<int>` is required, both translation unit generated its own corresponding definition of `foo<int>`. Therefore, if we were to link both `a.cpp` and `b.cpp` there should be no reason to have any linker errors. This explains why templates are `inline` by default and is one example of why multiple definitions can co-exist across translation units.

I list some entities that are `inline` by default and give a short explanation:
- A member definition inside a `class` or `struct` is `inline` by default because multiple translation unit may include those member definitions and they have to be safe when linked together.
- `constexpr` functions or variables are compile-time constants and are meant to be used in headers and included across different translation units.

Let's take a look on how the linker would resolve the multiple definition issue during the linking phase. Let's look back at the code example above on templates, if we compile `a.cpp` into its corresponding object file and observe the symbols in it.
```zsh
> nm a.o
0000000000000000 W _Z3fooIiEiT_S0_
```
We can see that the generated `foo<int>` symbol in the object file has a  `W` notation which means that the function is a weak symbol in this object file. When the linker links multiple object files together, if it finds a strong symbol, `T`, the linker would prefer that definition. If not, it would stick to its own definition. Therefore, if the linker comes across a symbol with the same name, it doesn't throw a linker error but instead uses it as a candidate for a possible replacement.

### Static and Dynamic Linking
Static linking happens during the linking phase when the linker receives a bunch of object files and static libraries. The linker first builds a global symbol table from all the object files that are given and try to resolve all the unresolved symbols. If there are still unresolved symbols, it slowly looks through the libraries that are given in order to resolve it. Take note that the linker only pulls the necessary symbol definition from the library and its dependencies as it builds the executable. Therefore, if the last library that pass in depends on a symbol from a library that was given earlier, we would have to pass that library in again as the last argument as the linker scans the libraries given from left to right. Let's build our own static library that expose our `add` function.

```cpp
// add.cpp
#include "add.hpp"
#include <iostream>

void add(int a, int b) {
  std::cout << "MY adder has been called: " << a + b << std::endl;
}

// main.cpp
#include "add.hpp"

int main() { add(1, 2); }
```

1. Compile `add.cpp` into an object file: `g++ -c add.cpp`
2. Archive object file to create a static library: `ar rcs libadd.a add.o`
3. Build our program: `g++ main.cpp -o main -L. -ladd`
	- `-llibrary`: tries to find `library` to use for linking
	- `-Ldir`: provides a search path, `dir`, for `-l` to find the library

Dynamic linking happens right before the program starts as the OS calls the linker to resolve any unresolved symbols with the shared libraries that are embedded as metadata into the executable when we compiled our code. Shared libraries are the default go-to libraries and were created primarily to save disk space and memory. For example,`libstdc++`, every time we use something from the standard library in our program, our program will contain metadata on how to retrieve the shared library just before the program starts. The shared library then gets pulled from memory and the corresponding symbols will get linked. So essentially, the benefit of shared libraries is that we don't need to load the entire library into our executable hence keeping its size minimal. Let's build our own shared library that expose our `add` function using the same code as above.

1. Compile `add.cpp` into an object file: `g++ -fPIC -c add.cpp`
	- `-fPIC`: creates position independent code because shared library gets loaded on runtime
2. Create shared library: `g++ -shared -o libadd.so add.o`
3. Build our program: `g++ main.cpp -o main -L. -ladd`
4. Include the current directory as a possible search path for the shared library: `export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH`

Now if we take a look at the shared libraries `main` depends on, you would find `libadd.so`

```zsh
✦ ❯ ldd main
		...
        libadd.so => ./libadd.so (0x00007158be03a000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007158bde00000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007158bda00000)
        ...
```



## Conclusion
In this article, we first discussed how a source file gets preprocessed by the preprocessor and how directives are used to guard our headers so that we don't accidentally include the same header file we can lead to redefinition error. We then take a look at some features of C++ and how it gets compiled into machine code. We discussed how header files that depend on each other can be resolved with forward declaration and also how to expose our functions such that C programs are able to link with it. We also discussed how templates are actually compiled on demand and the template itself is just a blueprint. Finally, we tied things together by discussing how the linker combines all the translation units together and gave a small introduction to static and shared libraries.

<div id="refs" class="references">
1. IncludeGuardian. Pragma-once vs ifndef. <a href="https://includeguardian.io/article/pragma-once-vs-ifndef" target="_blank">https://includeguardian.io/article/pragma-once-vs-ifndef</a>
<br>
2. cppreference. C++ language definition. <a href="https://en.cppreference.com/w/cpp/language/definition.html" target="_blank">https://en.cppreference.com/w/cpp/language/definition.html</a>
<br>
3. Toptal. Understanding C++ compilation. <a href="https://www.toptal.com/developers/c-plus-plus/c-plus-plus-understanding-compilation" target="_blank">https://www.toptal.com/developers/c-plus-plus/c-plus-plus-understanding-compilation</a>
<br>
4. Lurklurk. Linkers: overview and object files. <a href="https://www.lurklurk.org/linkers/linkers.html#cfilelisting" target="_blank">https://www.lurklurk.org/linkers/linkers.html#cfilelisting</a>
<br>
5. cppreference. Inline functions in C++. <a href="https://en.cppreference.com/w/cpp/language/inline.html" target="_blank">https://en.cppreference.com/w/cpp/language/inline.html</a>
<br>
6. Stroustrup B. *The C++ Programming Language* (4th ed). Addison-Wesley Professional; 2013. <a href="https://www.google.com/search?q=The+C%2B%2B+Programming+Language+Bjarne+Stroustrup+4th+Edition" target="_blank">Google Books</a>
</div>