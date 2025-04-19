---
categories: [cpp]
date: '2025-02-26T23:53:49-05:00'
layout: post
permalink: /c/const-correctness/
tags: [cpp, Software Architecture]
title: Const Correctness
---

Const correctness refers to the correct use of `const` to indicate whether a variable or parameter shall be modified or not. Other languages may inherently make variables const (“immutable”) by default, with a mutable values explicitly defined as such with a separate keyword: sometimes I wonder whether this is a better approach. When tracing some data over many function calls, having those functions respect const correctness allows the reader to skip reading those function calls, knowing the data cannot change.

In regards to performance, there are significant runtime performance benefits from using `constexpr` or `consteval` where appropriate. There are fewer (if any) performance benefits from using a runtime constant (plain `const`). However, my belief is that the improved readability will indirectly result in more time that can be dedicated to performance when and where it matters.

Before going further, I recommend reading the [ISO C++ section](https://isocpp.org/wiki/faq/const-correctness) on const correctness. It is a more official source of information and provides great examples.

There are various scenarios in which a value may be const. I attempt to summarize the most common scenarios below.

## Function Parameters

Even in a language-agnostic setup without a keyword indicating the constness of a variable, there is still a notion of constness as it relates to function parameters as they get categorized as `in`, `out`, or `in-out` parameters. As an example, Doxygen, a documentation generator, supports these categories, calling it [the direction](https://www.doxygen.nl/manual/commands.html#cmdparam) of the parameter. Clearly, the direction of a parameter is an important attribute that affects the readability of code.

I have seen support for these categories in the code documented in both the Doxygen comments and in the function signature as follows:

```cpp
#define IN
#define OUT
#define INOUT

struct LargeStruct { /* some large member variables */ };

/**
 * @param input[in]          Some input parameter
 * @param in_and_out[inout]  Some input and output parameter
 * @param output[out]        Some output parameter
 */
void foo(IN LargeStruct& input, INOUT LargeStruct& in_and_out, OUT LargeStruct& output);
```

While the Doxygen comments are helpful, the function signature could use some improvement. Specifically, what if I told you there was a way to have compile-time versions of the preprocessor `IN`, `OUT`, and `INOUT` macros? Yes, the keyword we’re looking for is `const`. If a parameter is only an input parameter, then it is logically const, and can be passed by value or by `const&` instead. If there is no `const`, then it must be either an output, or an input-output parameter. If this contract is broken by the function then it will refuse to compile. The compiler is your friend! Rewrite the above as follows:

```cpp
struct LargeStruct { /* some large member variables */ };

/**
 * @param input[in]          Some input parameter
 * @param in_and_out[inout]  Some input and output parameter
 * @param output[out]        Some output parameter
 */
void foo(LargeStruct const& input, LargeStruct& in_and_out, LargeStruct& output);
```

Better yet, if a parameter is only an output parameter then it can be returned from the function:

```cpp
struct LargeStruct { /* some large member variables */ };

/**
 * @param input[in]          Some input parameter
 * @param in_and_out[inout]  Some input and output parameter
 * @return Some output parameter
 */
LargeStruct foo(LargeStruct const& input, LargeStruct& in_and_out);
```

Most legacy C-based codebases use the return value to indicate the Status of the API call.

With modern C++, the introduction of `std::optional<T>`, `std::expected<T, Errcode>` (or an `absl::StatusOr<T>`) are much more readable alternatives – and they have other benefits, too.

If a function provides output via a non-const reference to an object, the caller is forced to add yet another line of code to declare that object as a variable in their scope. Not only is this more code to read (and write), it requires the object to be [default-constructible](https://en.cppreference.com/w/cpp/named_req/DefaultConstructible). This may not be an issue for trivial POD types, but it is often an issue for more complex functions, such as those returning an object representing a new NetworkConnection or the like.

Furthermore, the state of a passed-by-reference object in the case of an API failure is unknown: while it is expected for the caller to not access that object, it is yet another implicit contract that adds mental overhead.

I strongly recommend using std::optional to selectively return an object in a boolean success/failure cases, or a `std::expected<T, Errcode>` for cases where there is an explicit error code. If those are not available in your version of C++, you can implement them yourself, or use open source implementations ([tl::optional](https://github.com/TartanLlama/optional), [tl::expected](https://github.com/TartanLlama/expected)).

## Member Variables

One of my most interesting discoveries in C++ was that const class member variables prevents the default copy and move assignment constructors ([see example](https://godbolt.org/z/qzPY78hcT)). I was annoyed when I saw it for the first time, but I then realized it may be a good thing. I had attempted to make an `id` member variable constant, so each of my objects would have a unique identifier, and I quickly noticed that some of my code that used a copy assignment did not compile.

While frustrating in the moment, I realized this may have been a good thing: perhaps if an object is given a unique identifier, it should retain that identifier even if moved-from or copied-from, and perhaps the moved-to or copied-to object should get a new identifier? That can be done by manually defining the two assignment operators. At worst, if you’d like to copy the same identifier, you can still explicitly define those operators to do exactly what you’d like.

With that said, I do sometimes lament that I cannot have member variables that are only const in member functions, and not in constructors. Oh well.

## Function Return Values

Recall: const is used to improve the readability of a system and for some compile-time validation.

The return type of a function is part of its documented behaviour. The way a returned value is used by a caller is outside of the scope of the function, and does not need to be enforced, documented, or even suggested by the function signature.

In fact, even if a function does return a constant value, the caller can still store it as non-const.

```cpp
struct MyStruct {};

MyStruct create() { return MyStruct {}; }
const MyStruct create_const() { return MyStruct {}; }

int main() {
    MyStruct s = create();
    MyStruct s2 = create_const();
}
```

The only thing they cannot do is to use the returned rvalue directly in a non-const way, such as by calling a non-const member function or forwarding it as an rvalue reference.

```cpp
struct MyStruct {
    void do_something() {}
};

const MyStruct create_const() { return MyStruct {}; }

int main() {
    // error: passing 'const MyStruct' as 'this' argument discards qualifiers
    // create_const().do_something();

    void bar(MyStruct&& );
    // error: binding reference of type 'MyStruct&&' to 'const MyStruct' discards qualifiers
    // bar(create_const()); 
}
```

However, I don’t think that’s a common use case. All in all, I don’t know of good reason to return values by const.

## Conclusion

Runtime `const` is useful to formally guarantee and document the behaviour of an entity or API. It’s used as a tool for communication and compile-time validation. Most importantly, it improves the readability of your code. Use it where appropriate!
