---
title: "C++ template: Trying to Make the Easy Way"
date:  2023-04-30T00:59:12Z
summary: "A gentle introduction into advanced template techniques on a practical example: a `makeString(...)` method to convert everything to a string"
# weight: 1
# aliases: ["/first"]
tags: ["c++", "cpp", "templates", "sfinae", "concepts"]
author: "Me"
draft: false
#description: "Desc Text. Trying **bold**, or even `code`"
#canonicalURL: "https://canonical.url/to/page"
---
Imagine a power to convert everything to a string: integers, vectors, booleans - all transformed into a message to output wherever we want. Seems handy and not too hard, so let's dive into it -- this was my first thought when writing a debug variable watcher for my pet project.

Although the expectation of making something easy may result in a few sleep-deprived nights, I encourage the reader to embark on this journey through basic template programming concepts.

Expected prerequisites are:
 - A compiler with decent support for C++20: MSVC 2022, clang 14, or gcc 11 would be good, as we aim for a future-oriented experience, not dwelling on legacy code.
 - A basic familiarity with C++ template syntax: we've all written something like `template <typename T> T min(T a, T b);`, haven't we?[^typename]

Throughout the journey, we'll progress from a trivial attempt to a satisfying solution, highlighting pitfalls and techniques along the way. So don't be alarmed if the code doesn't look great in the early iterations -- it's all part of the plan.

There would be a [References](#references) section with all the links from an article. I'd treat it as a recommended reading. 

[^typename]: Actually, some of use wrote `template <class T>` -- that doesn't matter, but I prefer a `template <typename T>` because the `int` is not a class name

## The problem

Having a variable of type T, make a function `makeString(T t)` that will produce a string representation of `t`. As we could not know in advance how to convert an object of an arbitrary type into a string, consider making it easily extensible for any user-defined class.

Regarding extensibility, I'd like `makeString` to convert an object to string by using object's `std::string to_string() const` member function if it's present. It's not always possible, so fall-back to overloading or specialization if needed.

## Something to start with

### Setup a project

We need an empty project to start with. It's at least 2023 outside, so I use [CMake](https://en.wikipedia.org/wiki/CMake). It’s okay to create a console application project using your favorite IDE instead, but in case you ever plan to write a cross-platform code during the long and happy C++ career path, CMake is definitely worth trying.

{{< highlight cmake >}}
cmake_minimum_required(VERSION 3.12)
project(makeString VERSION 0.1.0)
add_executable(makeString main.cpp)

set(CMAKE_CXX_EXTENSIONS OFF)      # no vendor-specific extensions
set_property(TARGET makeString PROPERTY CXX_STANDARD 20)

# Ask a compiler to be more demanding
if (MSVC)
    target_compile_options(makeString PUBLIC /W4 /WX /Za /permissive-)
else()
    target_compile_options(makeString PUBLIC -Wall -Wextra -pedantic -Werror)
endif()
{{< /highlight >}}

### A bit of test code to understand the goal

Let’s add a couple of use cases: assume we have structs A and B. A is a tag with no data, while B contains an integer. The only reason to use structs instead of classes here is to save some screen space on `public:` visibility specifiers.

{{< highlight cpp >}}
#include <iostream>
#include "makeString.hpp"

struct A
{
    std::string to_string() const { return "A"; }
};

struct B
{
    int m_i = 0;
    std::string to_string() const { return "B{" + std::to_string(m_i) + "}"; }
};

int main()
{
    A a;
    B b = {1};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b)
              << std::endl;
}
{{< /highlight >}}

### A trivial template in makeString.hpp

And finally, a trivial template.

{{< highlight cpp >}}
#pragma once
#include <string>

template <typename Object>
std::string makeString(const Object& object)
{
    return object.to_string();
}
{{< /highlight >}}
Now we can build it, run, and enjoy the output: `a: A; b: B{1}`. 

This code has some problems, but let's pretend we don't see any until they arise during practical usage in the following sections of the article.

## Function template specialization

But what if we'd like to convert an `int` to a string?

{{< highlight cpp>}}int main()
{
    A a;
    B b = {1};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b)
              << "; pi: " << makeString(3)
              << std::endl;
}{{< /highlight >}}
Of course, the code above will result in a compilation error because there is no `to_string()` method for integers. Fortunately, we could provide a [template specialization](https://en.cppreference.com/w/cpp/language/template_specialization) specifying that `makeString<int>` should have a special implementation. Let's try:
{{< highlight cpp>}}// rather an attempt, than a solution
template <> std::string makeString(int i)
{
    return std::to_string(i);
}{{< /highlight >}}

Correct? Nope. The compiler does not find a matching template to specialize, because the only part of the template signature that could be specialized is the `Object` parameter: {{< highlight cpp>}}template <typename Object> std::string makeString(const Object& object){{< /highlight >}}
It's possible to substitute an `Object` with `int`, but we can't drop the const and reference qualifiers. Thus, the correct approach could be:
{{< highlight cpp>}}// template specialization: Object = int
template <> std::string makeString(const int& i)
{
    return std::to_string(i);
}{{< /highlight >}}

Well, although the function above works, it violates the ["pass cheaply-copied types by value"](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f16-for-in-parameters-pass-cheaply-copied-types-by-value-and-others-by-reference-to-const) guideline. Passing by-value is not only cheaper but also easier to optimize. Check out [Arthur O’Dwyer's blog](https://quuxplusone.github.io/blog/2021/11/09/pass-string-view-by-value/) for additional insight on the example of `string_view`.

## Template function overloading

Both template and non-template functions participate in overloading. [Function template](https://en.cppreference.com/w/cpp/language/function_template) reference has a lot of insight, [Overload resolution of function template calls](https://learn.microsoft.com/en-us/cpp/cpp/overload-resolution-of-function-template-calls) article has a couple of quick examples, but a summary is enough for now: _both template and non-template functions are considered during an overload resolution to find the best match possible_.

{{< highlight cpp>}}// not a template. This will be the best match for makeString(3),
// unless we explicitly specify that we want a template: makeString<int>(3)
std::string makeString(int i)
{
    return std::to_string(i);
}{{< /highlight >}}

Now we can drop the `const int&` specialization, compile, run, and enjoy overloaded `makeString(int)` and the approximate of pi: `a: A; b: B{1}; pi: 3`

For those who need precision, we could add more overloads:
{{< highlight cpp>}}
// main.cpp
// ...
    std::cout << "a: " << makeString(a) << "; b: " << makeString(b)
              << "; pi: " << makeString(3) << "; pi(double): " << makeString(3.1415926)
              << std::endl;
{{< /highlight >}}
{{< highlight cpp>}}
// makeString.hpp
//...
std::string makeString(float f)       { return std::to_string(f); }
std::string makeString(double d)      { return std::to_string(d); }
std::string makeString(long double d) { return std::to_string(d); }
// ... 5 more specializations ...
{{< /highlight >}}

## Selecting a single template from multiple implementations

### The problem

In the middle of writing the copy-paste above, a reader might come across the idea of creating a single template instead. However, a problem arises when the compiler is unsure which template to instantiate for a call like `makeString(3)`, leading to a compilation failure.

{{< highlight cpp>}}
template <typename Object>
std::string makeString(const Object& object)
{
    return object.to_string();
}

template <typename Numeric>
std::string makeString(Numeric value)
{
    return std::to_string(value);
}

/*
main.cpp:21:40: error: call of overloaded ‘makeString(int)’ is ambiguous
 |               << "; pi: " << makeString(3)
 |                              ~~~~~~~~~~^~~
  note: candidate: std::string makeString(const Object&) [with Object = int;]
  note: candidate: std::string makeString(Numeric) [with Numeric = int;]
*/
{{< /highlight >}}

During the overload resolution, the compiler examines only the function declarations, requiring the developer to provide a means of distinguishing overloads using function declarations only.

As a general rule, I'd consider restricting all template functions to prevent the compiler from instantiating them for incompatible types. Following this rule prevents extensibility issues when adding new template function overloads.

<details>
<summary>Spoiler: we could have used the code above almost as it is if we had read the Concepts section in advance</summary>
{{< highlight cpp>}}

// concepts HasToString, IsNumeric, and IsString should be defined above

template <HasToString Object>
std::string makeString(Object&& object)
{
    return std::forward<Object>(object).to_string();
}

std::string makeString(IsNumeric auto value)
{
    return std::to_string(value);
}

template <IsString String>
std::string makeString(String&& s)
{
    return std::string(std::forward<String>(s));
}
{{< /highlight >}}
</details><br>

For a deeper grasp of templates, we'll proceed with <abbr title="Substitution Failure Is Not An Error">SFINAE</abbr>: a more verbose solution that was prevalent before the concepts appeared in C++20. However, readers who are eager to move ahead may skip this section and proceed directly to the [perfect forwarding chapter](#perfect-forwarding-avoid-unnecessary-copying-and-allow-use-of-non-copyable-types) and then to [concepts](#concepts-a-modern-approach-to-set-requirements-on-a-template-type).

### Substitution Failure Is Not An Error: a harder, pre-C++20 approach

During overload resolution, the compiler will ignore the declaration if impossible to substitute the template parameter into it. That concept is known as [Substitution Failure Is Not An Error](https://en.cppreference.com/w/cpp/language/sfinae).

Using wit and a little imagination, come up with a solution:
 * Incorporate an `object.to_string()` member function call into the declaration of `std::string makeString(const Object& object)` to prevent substitution of `[Object = int]`.
 * Ensure that the declaration of `std::string makeString(Numeric value)` involves `std::to_string(value)` to prevent substitution in case of no such `std::to_string` overload before the compiler fails to compile the function body.

There are a few possible solutions:

#### 1. Depend on a result of the `to_string()` member function call in the type parameter.
{{< highlight cpp>}}
// will match only types with to_string() member function
template <typename Object,
          typename DummyType = decltype(std::declval<Object>().to_string())>
std::string makeString(const Object& object)
{
    return object.to_string();
}
{{< /highlight >}}

The code above uses a `DummyType` that is the same as the type of `Object::to_string()` member function. Substitution will fail if there is no `Object::to_string()` thus excluding the template from the overloading resolution.

#### 2. Trailing return type: access the function parameter name during the substitution

One could wonder, why don't we use parameter variable names to enforce SFINAE? That's becasue syntaxically we're enforcing it before the function parameters definition. Using <abbr title="auto foo(int x) -> trailing-return-type;">trailing return type</abbr> we could defer the SFINAE to the point where the function parameters are declared.

That's my favorite way to use SFINAE using C++ 17, prior to concepts introduction.

Let's take a look:
{{< highlight cpp>}}// makeString.hpp
#pragma once
#include <string>

template <typename Object>
auto makeString(const Object& object) -> decltype(object.to_string())
{
    return object.to_string();
}

template <typename Numeric>
auto makeString(Numeric value) -> decltype(std::to_string(value))
{
    return std::to_string(value);
}{{< /highlight >}}

Here, an `auto` return type froces the compiler to look at the trailing return type, so it will detect the type of expression on the right of `->`, for example `decltype(object.to_string())`. If the substituition fails, the function is omitted from the overloads candidate list. That's simple.

There is a `std::enable_if` template that could solve the same problem, but in my opinion, it’s more verbose. Therefore, I’d reserve its usage until it’s truly necessary.

Once again, compile, run, enjoy: `a: A; b: B{1}; pi: 3; pi(double): 3.141593`. We've spent some time on it, it seems working and I think, <abbr title="a polite way to say it's far from finished">it's a good start<abbr>.

## Collections support in makeString()

Let's level up makeString() with support for generic containers and amp up the coding fun. First, alter out test code with some cases:
{{< highlight cpp>}}
// main.cpp
#include <vector>
#include <set>
// ...

    const std::vector<int> xs = {1, 2, 3};
    const std::set<float> ys = {4, 5, 6};
    const double zs[] = {7, 8, 9};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b)
              << "; pi: " << makeString(3.1415926) << std::endl
              << "xs: " << makeString(xs) << "; ys: " << makeString(ys)
              << "; zs: " << makeString(zs)
              << std::endl;
{{< /highlight >}}
Of course, the code is missing the necessary template. I like diving right into coding without much thought, but in this case, let's be clever. A collection could be a vector, a set, or a C-array, but in a generic case, it is something _iterable_. We could use `std::begin` on something iterable. Thus we need a template that will accept something compatible with `std::begin` and iterate on it.

Now, start typing:
{{< highlight cpp>}}
// makeString.hpp
// ...
template <typename Iterable>
auto makeString(const Iterable& iterable)
    -> decltype(makeString(*std::begin(iterable))) // (1)
{
    std::string result;
    for (const auto& i : iterable)
    {
        if (!result.empty())
            result += ';';
        result += makeString(i);
    }

    return result;
}{{< /highlight >}}
Just in case, a quick note (1): that's a template that can accept the something that could be passed to `std::begin`, and return the same type as the `makeString` for the first element of that collection.

Compile, run, so far so good:

    a: A; b: B{1}; pi: 3.141593
    xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000

However, that code has a pitfall we're about to discover in the next section.

## String parameter support in makeString()

Just in case, why don't? It might streamline the makeString() usage in other templates. Let's start by adding some test cases:
{{< highlight cpp>}}
// main.cpp
// ...

    std::cout << makeString("Hello, ")
              << makeString(std::string_view("world"))
              << makeString(std::string("!!1"))
              << std::endl;
{{< /highlight >}}
Compile, run, and brace yourself for a surprise: it works! But not quite as intended...
{{< highlight cpp "hl_lines=3">}}
a: A; b: B{1}; pi: 3.141593
xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000
72;101;108;108;111;44;32;0119;111;114;108;10033;33;49
{{< /highlight >}}

Well, strings are iterable containers of chars. We also learned from the [Integer Promotions](https://en.cppreference.com/w/c/language/conversion) chapter on the cppreference that compiler promotes chars to integers when `char` overloading is absent. To address the issue, exclude the template for containers from matching string types.

Also, it could be reasonable to mark the `makeString(char)` as the deleted function to secure the pitfall forewer:
{{< highlight cpp>}}std::string makeString(char c) = delete;{{< /highlight >}}
{{< highlight cpp>}}
[build] makeString.hpp:62:6: note: candidate template ignored: substitution failure
          [with Iterable = const char (&)[8]]: call to deleted function 'makeString'
[build] auto makeString(Iterable&& iterable) -> decltype(makeString(*std::begin(iterable)))
[build]      ^                                           ~~~~~~~~~~
{{< /highlight >}}

That's a good time to remember about the type traits and `std::enable_if`.

### Type traits and enable_if: a way to specialize template on a trait or a condition

A solution is to restrict container's `makeString` so it will fail substitution on strings, and write a new one. Consider that "string" is `std::string`, `std::string_view`, `char*`, `const char*`. Let's explress this using C++:

{{< highlight cpp>}}
namespace traits
{
// generic type is not a string...
template <typename AnyType> inline constexpr bool isString = false;
// ... but these types are strings
template <> inline constexpr bool isString<std::string>      = true;
template <> inline constexpr bool isString<std::string_view> = true;
template <> inline constexpr bool isString<char*>            = true;
template <> inline constexpr bool isString<const char*>      = true;
}
{{< /highlight >}} But wait, what's about `const string`, `const string_view`, `const char* const`? And there is a `volatile` keyword as well. Fortunately, a lot of copy-paste could be avoided by using a kind of types mapping: `T` -> `T`, `const T` -> `T`. For example, we could have written something like the code below:
{{< highlight cpp>}}
namespace traits
{
namespace impl  // a "private" implementation
{
    // generic type is not a string...
    template <typename AnyType>
    inline constexpr bool isString = false;
    // ... but these types are strings
    template <> inline constexpr bool isString<std::string>      = true;
    template <> inline constexpr bool isString<std::string_view> = true;
    template <> inline constexpr bool isString<char*>            = true;
    template <> inline constexpr bool isString<const char*>      = true;
}

namespace Wheel
{
    template <typename T> struct remove_cv                   { using type = T; };
    template <typename T> struct remove_cv<const T>          { using type = T; };
    template <typename T> struct remove_cv<volatile T>       { using type = T; };
    template <typename T> struct remove_cv<const volatile T> { using type = T; };

    // shortcut to remove_cv<T>::type, saves a bit of stamina on fingers
    template <typename T> using remove_cv_t = typename remove_cv<T>::type;

    // enable_if only defined as a specialization enable_if<true, T>
    // thus failing substitution on any code that uses enable_if<false, T>;
    template <bool Condition, typename T> struct enable_if;
    template <typename T> struct enable_if<true, T> { using type = T; };

    // another shortcut
    template <bool B, class T> using enable_if_t = typename enable_if<B,T>::type;
}

// traits::isString<T> = traits::impl::isString<T>, but with remove_cv_t on T:
template <typename T>
inline constexpr bool isString = impl::isString<
                                    Wheel::remove_cv_t<T>
                                >;
}
{{< /highlight >}} However, we've already developed a habit of looking into a [cppreference.com](https://en.cppreference.com/w/cpp/header/type_traits) in advance, so we'll use `std::remove_cv_t` and `std::enable_if_t` instead if reinventing the wheel:
{{< highlight cpp>}}
// makeString.hpp
#pragma once
#include <string>
#include <type_traits>

namespace traits
{
namespace impl  // a "private" implementation
{
    // generic type is not a string...
    template <typename AnyType>
    inline constexpr bool isString = false;
    // ... but these types are strings
    template <> inline constexpr bool isString<std::string>      = true;
    template <> inline constexpr bool isString<std::string_view> = true;
    template <> inline constexpr bool isString<char *>           = true;
    template <> inline constexpr bool isString<const char *>     = true;
}

// isString<T> = impl::isString<T>, but with dropped const/volatile on T:
template <typename T>
inline constexpr bool isString = impl::isString<std::remove_cv_t<T>>;
}

template <typename Object>
auto makeString(const Object& object) -> decltype(object.to_string())
{
    return object.to_string();
}

template <typename Numeric>
auto makeString(Numeric value) -> decltype(std::to_string(value))
{
    return std::to_string(value);
}

// will fail substitution if `traits::isString<Iterable>`
// or when can't evaluate the type of makeString(*begin(container))
template <typename Iterable>
auto makeString(const Iterable& iterable)
    -> std::enable_if_t< !traits::isString<Iterable>,
                          decltype(makeString(*std::begin(iterable))) >
{
    std::string result;
    for (const auto& i : iterable)
    {
        if (!result.empty())
            result += ';';
        result += makeString(i);
    }

    return result;
}

// will fail substituition if `!traits::isString<String>`
template <typename String>
auto makeString(const String& s)
    -> std::enable_if_t< traits::isString<String>,
                         std::string >
{
    return std::string(s);
}
{{< /highlight >}}
Now compile, run, swear:
{{< highlight cpp "hl_lines=3">}}
a: A; b: B{1}; pi: 3.141593
xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000
72;101;108;108;111;44;32;0world!!1
{{< /highlight >}}

### Getting an insight of what does the template expand to

According to output, `"Hello, "` were treated as a container. It could be easier to understand what's happening using some tricks. There is at least a couple ways: upload a minimal __compilable__ example to a compiler analyzing tool, or produce an intentional failure with a descriptive error message.

#### C++ Insights

Once code compiles successfully, we could upload it to the cppinsights.io. There's [the link](https://cppinsights.io/lnk?code=I2luY2x1ZGUgPGlvc3RyZWFtPgojaW5jbHVkZSA8c3RyaW5nPgojaW5jbHVkZSA8dHlwZV90cmFpdHM+CgpuYW1lc3BhY2UgdHJhaXRzCnsKbmFtZXNwYWNlIGltcGwgIC8vIGEgInByaXZhdGUiIGltcGxlbWVudGF0aW9uCnsKICAgIC8vIGdlbmVyaWMgdHlwZSBpcyBub3QgYSBzdHJpbmcuLi4KICAgIHRlbXBsYXRlIDx0eXBlbmFtZSBBbnlUeXBlPgogICAgaW5saW5lIGNvbnN0ZXhwciBib29sIGlzU3RyaW5nID0gZmFsc2U7CiAgICAvLyAuLi4gYnV0IHRoZXNlIHR5cGVzIGFyZSBzdHJpbmdzCiAgICB0ZW1wbGF0ZSA8PiBpbmxpbmUgY29uc3RleHByIGJvb2wgaXNTdHJpbmc8c3RkOjpzdHJpbmc+ICAgICAgPSB0cnVlOwogICAgdGVtcGxhdGUgPD4gaW5saW5lIGNvbnN0ZXhwciBib29sIGlzU3RyaW5nPHN0ZDo6c3RyaW5nX3ZpZXc+ID0gdHJ1ZTsKICAgIHRlbXBsYXRlIDw+IGlubGluZSBjb25zdGV4cHIgYm9vbCBpc1N0cmluZzxjaGFyICo+ICAgICAgICAgICA9IHRydWU7CiAgICB0ZW1wbGF0ZSA8PiBpbmxpbmUgY29uc3RleHByIGJvb2wgaXNTdHJpbmc8Y29uc3QgY2hhciAqPiAgICAgPSB0cnVlOwp9CgovLyBpc1N0cmluZzxUPiA9IGltcGw6OmlzU3RyaW5nPFQ+LCBidXQgd2l0aCBkcm9wcGVkIGNvbnN0L3ZvbGF0aWxlIG9uIFQ6IAp0ZW1wbGF0ZSA8dHlwZW5hbWUgVD4gCmlubGluZSBjb25zdGV4cHIgYm9vbCBpc1N0cmluZyA9IGltcGw6OmlzU3RyaW5nPHN0ZDo6cmVtb3ZlX2N2X3Q8VD4+OyAKfQoKdGVtcGxhdGUgPHR5cGVuYW1lIE51bWVyaWM+CmF1dG8gbWFrZVN0cmluZyhOdW1lcmljIHZhbHVlKSAtPiBkZWNsdHlwZShzdGQ6OnRvX3N0cmluZyh2YWx1ZSkpCnsKICAgIHJldHVybiBzdGQ6OnRvX3N0cmluZyh2YWx1ZSk7Cn0KCi8vIHdpbGwgZmFpbCBzdWJzdGl0dXRpb24gaWYgYHRyYWl0czo6aXNTdHJpbmc8SXRlcmFibGU+YAovLyBvciB3aGVuIGNhbid0IGV2YWx1YXRlIHRoZSB0eXBlIG9mIG1ha2VTdHJpbmcoKmJlZ2luKGNvbnRhaW5lcikpCnRlbXBsYXRlIDx0eXBlbmFtZSBJdGVyYWJsZT4KYXV0byBtYWtlU3RyaW5nKGNvbnN0IEl0ZXJhYmxlJiBpdGVyYWJsZSkgCiAgICAtPiBzdGQ6OmVuYWJsZV9pZl90PCAhdHJhaXRzOjppc1N0cmluZzxJdGVyYWJsZT4sIAogICAgICAgICAgICAgICAgICAgICAgICAgIGRlY2x0eXBlKG1ha2VTdHJpbmcoKnN0ZDo6YmVnaW4oaXRlcmFibGUpKSkgPgp7CglzdGQ6OnN0cmluZyByZXN1bHQ7Cglmb3IgKGNvbnN0IGF1dG8mIGkgOiBpdGVyYWJsZSkKCXsKCQlpZiAoIXJlc3VsdC5lbXB0eSgpKQoJCQlyZXN1bHQgKz0gJzsnOwoJCXJlc3VsdCArPSBtYWtlU3RyaW5nKGkpOwoJfQoKCXJldHVybiByZXN1bHQ7Cn0KCi8vIHdpbGwgZmFpbCBzdWJzdGl0dWl0aW9uIGlmIGAhdHJhaXRzOjppc1N0cmluZzxTdHJpbmc+YAp0ZW1wbGF0ZSA8dHlwZW5hbWUgU3RyaW5nPgphdXRvIG1ha2VTdHJpbmcoY29uc3QgU3RyaW5nJiBzKSAKICAgIC0+IHN0ZDo6ZW5hYmxlX2lmX3Q8IHRyYWl0czo6aXNTdHJpbmc8U3RyaW5nPiwKICAgICAgICAgICAgICAgICAgICAgICAgIHN0ZDo6c3RyaW5nID4gCnsKICAgIHJldHVybiBzdGQ6OnN0cmluZyhzKTsKfQoKaW50IG1haW4oKQp7CiAgICBzdGQ6OmNvdXQgPDwgbWFrZVN0cmluZygiSGVsbG8sICIpIAogICAgICAgICAgICAgIDw8IG1ha2VTdHJpbmcoc3RkOjpzdHJpbmdfdmlldygid29ybGQiKSkgCiAgICAgICAgICAgICAgPDwgbWFrZVN0cmluZyhzdGQ6OnN0cmluZygiISExIikpIAogICAgICAgICAgICAgIDw8IHN0ZDo6ZW5kbDsKfQo=&insightsOptions=cpp17&std=cpp17&rev=1.0) on a slightly stripped code.

And here we go, a part of the produced output:
{{< highlight cpp>}}
namespace traits
{
  namespace impl
  {
    // ...
    template<>
    inline constexpr const bool isString<char *> = true;
    template<>
    inline constexpr const bool isString<const char *> = true;
    template<>
    inline constexpr const bool isString<char[8]> = false;
    template<>
    inline constexpr const bool isString<char> = false;

  }
  // ...
}

// ...

/* First instantiated from: insights.cpp:59 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> makeString<char[8]>(const char (&iterable)[8])
{
    // ...
}
#endif
{{< /highlight >}}
It's quite obvious from the output above that `makeString("Hello, ")` is actually a `makeString<char[8]>(/*reference-to-char[8]*/)` and we have no such overload for `impl::isString`, so generic one takes place: `impl::isString<char[8]> = false;`.

#### Intentional compilation failure

It could happen, however, that code should not be submitted elsewhere or is does not compile. In this case, an intentional compilation failure could make an insight on the deduced types and values. A deleted function will provide a good error context when called, while undefined struct will provide an insight when instantiated:
{{< highlight cpp>}}
// ...
template <typename... Args> bool fail_function(Args&&... args) = delete;
template <bool Value> struct fail_struct;    // never defined
// ...
static bool test1 = fail_function("Hello, ");                            // (1)
static fail_struct< traits::isString<decltype("Hello, ")> > test2 = {};  // (2)
{{< /highlight >}}

And here is the compilation output:
{{< highlight cpp "hl_lines=2 6">}}
error: use of deleted function ‘bool fail_function(Args&& ...)
       [with Args = {const char (&)[8]}]’
   40 | static bool test1 = fail_function("Hello, ");                            // (1)
      |                     ~~~~~~~~~~~~~^~~~~~~~~~~

error: variable ‘fail_struct<false> test2’ has initializer but incomplete type
   41 | static fail_struct< traits::isString<decltype("Hello, ")> > test2 = {};  // (2)
      |                                                             ^~~~~
{{< /highlight >}}Let's extract the insight from the error message:
* in `fail_function("Hello, ")`, argument type is a reference to a `const char[8]`. Despite it's eligible for implicit conversion to the `const char*` it is different from the `const char*`.
* in the `fail_struct< traits::isString<...> >` variable, argument is `false`, so the result of `traits::isString<...>` is false.

### Finally, a makeString that accepts a string and works as expected

<details>
<summary>Just in case someone wants a solution to <code>isString&lt;char[N]&gt;</code> trait</summary>
It's that simple:
{{< highlight cpp>}}
namespace impl
{
    // ...
    template <size_t N>
    inline constexpr bool isString<char[N]> = true;
    // ...
}
{{< /highlight >}}
</details>

In previous chapters we've invented some wheels and undertood type traits idea. Now it's time to re-think it. Given the [standard library](https://en.cppreference.com/w/cpp/meta) we could handle strings better. For example, let's consider a type to be a string if an `std::string` could be explicitly constructed from it. That's it.
{{< highlight cpp>}}namespace traits
{
template <typename T>
inline constexpr bool isString = std::is_constructible_v<std::string, T>;
}{{< /highlight >}}

Bring the pieces together, compile, and run.
{{< highlight cpp>}}// main.cpp
#include <iostream>
#include <vector>
#include <set>
#include "makeString.hpp"

struct A
{
    std::string to_string() const { return "A"; }
};

struct B
{
    int m_i = 0;
    std::string to_string() const { return "B{" + std::to_string(m_i) + "}"; }
};

int main()
{
    A a;
    B b = {1};

    const std::vector<int> xs = {1, 2, 3};
    const std::set<float> ys = {4, 5, 6};
    const double zs[] = {7, 8, 9};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b)
              << "; pi: " << makeString(3.1415926) << std::endl
              << "xs: " << makeString(xs) << "; ys: " << makeString(ys)
              << "; zs: " << makeString(zs)
              << std::endl;

    std::cout << makeString("Hello, ")
              << makeString(std::string_view("world"))
              << makeString(std::string("!!1"))
              << std::endl;
}{{< /highlight >}}

{{< highlight cpp>}}
// makeString.hpp
#pragma once
#include <string>
#include <type_traits>

namespace traits
{
template <typename T>
inline constexpr bool isString = std::is_constructible_v<std::string, T>;
}

template <typename Object>
auto makeString(const Object& object) -> decltype(object.to_string())
{
    return object.to_string();
}

template <typename Numeric>
auto makeString(Numeric value) -> decltype(std::to_string(value))
{
    return std::to_string(value);
}

// will fail substitution if `traits::isString<Iterable>`
// or when can't evaluate the type of makeString(*begin(container))
template <typename Iterable>
auto makeString(const Iterable& iterable)
    -> std::enable_if_t< !traits::isString<Iterable>,
                          decltype(makeString(*std::begin(iterable))) >
{
    std::string result;
    for (const auto& i : iterable)
    {
        if (!result.empty())
            result += ';';
        result += makeString(i);
    }

    return result;
}

// will fail substituition if `!traits::isString<String>`
template <typename String>
auto makeString(const String& s)
    -> std::enable_if_t< traits::isString<String>,
                         std::string >
{
    return std::string(s);
}
{{< /highlight >}}
{{< highlight cpp>}}
a: A; b: B{1}; pi: 3.141593
xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000
Hello, world!!1
{{< /highlight >}}

Now criticize: it's still _<abbr title="require iteration">a good start!</abbr>_ At least, the problem is that the `std::string` is also constructed from the parameter, even if there is a temporary that could be perfectly forwarded to a constructor. That brings us to the next chapter.

## Perfect forwarding: avoid unnecessary copying and allow use of non-copyable types

In the code below, a temporary is made by `getSomeString()` and bound to an [lvalue-reference](https://en.cppreference.com/w/cpp/language/value_category) parameter. The referenced string then copied to a return value. Thanks to a [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision), no further copies were made, but in the upshot, one unnecessary copy occurs.

{{< highlight cpp>}}
template <typename String>
auto makeString(const String& s)
    -> std::enable_if_t< traits::isString<String>,
                         std::string >
{
    return std::string(s);
}

// ...

std::string getSomeString();
std::cout << makeString(getSomeString());
{{< /highlight >}}

In addition to suboptimal performance, the previous approach does not allow non-copyable types, such as `std::unique_ptr`. However, by leveraging rvalue or forwarding references, we can omit unnecessary copy and avoid the requirement of copyable type. The `std::string` constructor can then efficiently steal its content without unnecessary copying.

While the only option for a non-template parameter is an rvalue overload, template parameters can utilize [_universal (or forwarding) references_](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers). In essence, a forwarding reference accepts the same reference type (lvalue, rvalue, const, or non-const) as passed by the caller. A forwarding reference syntax is `TemplateType&& parameter`. 

I strongly encourage looking at the great article above for additional details. 

Just in case, a quick summary and we're going to fix our templates.
| Parameter Type | Template               | Non-Template           |
|:--------------:|:----------------------:|:----------------------:|
| const &        | const lvalue reference | const lvalue reference |
|       &        |       lvalue reference |       lvalue reference |
|       &&       | the same as has been passed by the caller, <br>may be: (const-) lvalue or rvalue | rvalue reference |

This way, for every pass-by-reference in makeString.hpp we'll pass the parameter by a universal reference:
{{< highlight cpp>}}
// ...
template <typename Object>
auto makeString(Object&& object) -> decltype(object.to_string())
//...
template <typename Iterable>
auto makeString(Iterable&& iterable)
// ...
template <typename String>
auto makeString(String&& s)
// ...
{{< /highlight >}} Is that's it? Not yet. A function parameter is always an lvalue inside the function. But in order to allow `std::string` to steal temporary's content, we have to provide an rvalue _if possible_.

Thoroughful developer may consider all the possible cases:
 * an lvalue or (const-)lvalue reference is passed to makeString(): pass as an appropriate lvalue reference further, because we should not invalidate an lvalue by moving from it; 
 * a temporary or another type of rvalue is passed to makeString(): fine to pass it via rvalue-reference to allow moving from it.

It might sound complicated, but that's exactly what `std::forward<T>` does. It casts a function parameter to a whatever is was at the function invocation. So, a fast developer just uses `std::forward` for universal reference whenever a possible move is intended.

### A forwarding pitfall: a temporary may be 'consumed' and invalidated by the calee

However, a cautious developer would warn us that wherever `std::forward` is used, a move may occur. So a variable should never be used after forwarding:
{{< highlight cpp "hl_lines=10 12" >}}
#include <string>
#include <cassert>
#include <vector>

template <typename Arg>
void foo(Arg&& a)
{
    std::vector<std::string> v;
    for(int i = 0; i < 2; i++)
        v.push_back(std::forward<Arg>(a)); // surprise on the 2nd iteration

    assert(v.front() == v.back());         // fail
}

int main()
{
    auto getMessage = []() -> std::string { return "why does it fail?"; };
    foo(getMessage());
}
{{< /highlight >}}
The example above may seem synthetic, but it does occur in the wild. Consider the following situation: 
1. There is a callback that receives a parameter using perfect forwarding: `logCallback(std::forward<Event>(logEvent));`. Looks good and conventional.
2. Later, a developer forgot to remove `std::forward` while implementing multiple callbacks support: `for (auto& sink : logSinks) sink.logCallback(std::forward<Event>(event));`. This oversight creates a ticking time bomb.  
3. The issue goes unnoticed during basic testing with only a single callback or an lvalue reference.
4. Undefined behavior occurs in rare case where multiple callbacks involve rvalue reference, leading to unexpected behavior.

Thus a developer should have a strong intuition to remove `std::forward` once a possibility of multiple recipients of the forwarded value occurs.

### Temporary containers

A tricky case is a container that is passed as rvalue. Although the container may be a temporary, its elements are constructed in a usual way (for example, it's possible to get an address of the element) so they are treated like lvalues. Thus, in order to steal elements' content from a temporary container, a forcible `std::move` should be used:

{{< highlight cpp "hl_lines=17-21">}}
// will fail substitution if `traits::isString<Iterable>`
// or when can't evaluate the type of makeString(*begin(container))
template <typename Iterable>
auto makeString(Iterable&& iterable)
    -> std::enable_if_t< !traits::isString<Iterable>,
                          decltype(makeString(*std::begin(iterable)))
                        >
{
    std::string result;
    for (auto&& i : iterable)
    {
        if (!result.empty())
            result += ';';

        // a constexpr if, so the compiler can omit unused branch and
        // allow non-copyable types usage
        if constexpr (std::is_rvalue_reference_v<decltype(iterable)>)
            result += makeString(std::move(i));
        else
            result += makeString(i);
    }
    return result;
}
{{< /highlight >}}

'if constexpr' in this context enables selecting a strategy in the compile-time based on the container's <abbr title="lvalue or rvalue">value category</abbr>. Unlike the regular `if` statement, only one branch of constexpr `if/else` is compiled at template instantiation, allowing for the use of a type that is either movable or copyable.

The [MS STL implementation of std::vector::resize](https://github.com/microsoft/STL/blob/c8d1efb6d504f6392acf8f6d01fd703f7c8826c0/stl/inc/vector#L1529) utilizes this approach to choose between copy and move strategies. When resizing, it moves existing items to a new buffer if the value type has a 'noexcept' move constructor, and copies them otherwise[^noexcept-move].

Also, there are alternative approaches. For instance, introducing a separate template to encapsulate the copy-or-move decision into two overloads is demonstrated in the `std::move_if_noexcept` [example on cppreference](https://en.cppreference.com/w/cpp/utility/move_if_noexcept).

[^noexcept-move]: `std::vector` features strong exception safety guarantee: its remains unchanged if `resize()` throws an exception. In order to maintain that it has to keep initial buffer as a backup if element's type move constructor is not declated as `noexcept(true)`

Bringing all the pieces together, let's look at the perfect forwarding makeString below:
{{< highlight cpp "linenos=table,hl_lines=16">}}
// makeString.hpp
#pragma once
#include <string>
#include <type_traits>

namespace traits
{
template <typename T>
inline constexpr bool isString = std::is_constructible_v<std::string, T>;
}

template <typename Object>
auto makeString(Object&& object)
    -> decltype(std::forward<Object>(object).to_string())
{
    return std::forward<Object>(object).to_string(); // (see a note below)
}

template <typename Numeric>
auto makeString(Numeric value) -> decltype(std::to_string(value))
{
    return std::to_string(value);
}

// will fail substituition if `!traits::isString<String>`
template <typename String>
auto makeString(String&& s)
    -> std::enable_if_t< traits::isString<String>,
                         std::string >
{
    return std::string(std::forward<String>(s));
}

// will fail substitution if `traits::isString<Iterable>`
// or when can't evaluate the type of makeString(*begin(container))
template <typename Iterable>
auto makeString(Iterable&& iterable)
    -> std::enable_if_t< !traits::isString<Iterable>,
                          decltype(makeString(*std::begin(iterable)))
                        >
{
    std::string result;
    for (auto&& i : iterable)
    {
        if (!result.empty())
            result += ';';

        // a constexpr if, so the compiler can omit unused branch and
        // allow non-copyable types usage
        if constexpr (std::is_rvalue_reference_v<decltype(iterable)>)
            result += makeString(std::move(i));
        else
            result += makeString(i);
    }
    return result;
}
{{< /highlight >}}
And a couple of new tests:
{{< highlight cpp "linenos=table,hl_lines=24-25">}}
#include <iostream>
#include <vector>
#include <set>
#include "makeString.hpp"

struct A
{
    std::string to_string() const { return "A"; }
};

struct B
{
    int m_i = 0;
    std::string to_string() const { return "B{" + std::to_string(m_i) + "}"; }
};

struct NonCopyable
{
    std::string m_s;
    NonCopyable(const char* s) : m_s(s)  {}
    NonCopyable(NonCopyable&&) = default;
    NonCopyable(const NonCopyable&) = delete;

    std::string   to_string() const &  { return m_s; }
    std::string&& to_string() &&       { return std::move(m_s); }
};

int main()
{
    A a;
    B b = {1};

    const std::vector<int> xs = {1, 2, 3};
    const std::set<float> ys = {4, 5, 6};
    const double zs[] = {7, 8, 9};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b) 
              << "; pi: " << makeString(3.1415926) << std::endl
              << "xs: " << makeString(xs) << "; ys: " << makeString(ys) 
              << "; zs: " << makeString(zs)
              << std::endl;

    std::cout << makeString("Hello, ") 
              << makeString(std::string_view("world")) 
              << makeString(std::string("!!1")) 
              << std::endl;

    auto makeVector = []()
    { 
        std::vector<NonCopyable> v;
        v.emplace_back("two ");
        v.emplace_back(" non-copyables");
        return v; 
    };

    std::cout << makeString(makeVector())
              << std::endl;
}
{{< /highlight >}}
One might wonder, why did I forward an object here when calling `to_string()` on it. And what's a strange syntax on `NonCopyable::to_string`.
{{< highlight cpp >}}
template <typename Object>
auto makeString(Object&& object)
    -> decltype(std::forward<Object>(object).to_string())
{
    return std::forward<Object>(object).to_string(); // (see a note below)
}
// ...
struct NonCopyable
{
    // ...
    std::string to_string() const &  { return m_s; }
    std::string to_string() &&       { return std::move(m_s); }
};
{{< /highlight >}}

The `std::forward` here forwards the `object` to its `object.to_string()` member function, and corersponding `NonCopyable::to_string` methods feature the overloading on `*this` value category. This way we can allow an rvalue-overload of `NonCopyable::to_string` to steal the content of temporary avoiding unnecessary copying.

### Overloading member functions on reference qualifiers

A ref-quilified member function allows compiler to choose a specific overloading based on refevence type of `*this`. A simple example from the corresponding chapter of the [cppreference](https://en.cppreference.com/w/cpp/language/member_functions):
{{< highlight cpp>}}#include <iostream>

struct S
{
    void f() &  { std::cout << "lvalue\n"; }
    void f() && { std::cout << "rvalue\n"; }
};

int main()
{
    S s;
    s.f();            // prints "lvalue"
    std::move(s).f(); // prints "rvalue"
    S().f();          // prints "rvalue"
}{{< /highlight >}}
Regarding the `NonCopyable::to_string()`, in case of calling it on a temporary, it's safe to assume that the temporary will be obsolete after the call, so we can steal its content by moving the `NonCopyable::m_s` into a return value. Of course, we still don't need `&&` at the return type, because the object returned by value is rvalue itself.

Perhaps, that was the only time since 2011 that I've seen justified use of return `std::move`, because nowadays we may rely on guaranteed [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision) that usually works better that `std::move` for the return value[^return-move].

[^return-move]: a copy elision is better that moving a return value because the object will be constructed already in the scope of the caller without move construction. It also donsn't take any references of the return value thus making the optimizer's work easier.

There is an article on another interesting use case of ref-quilified member function on this blog: [Practical usage of ref-qualified member function overloading](../ref-qualifiers)

## Concepts: a modern approach to set requirements on a template type

Now it's time for a pivot: with C++20, we can avoid writing SFINAE code and embrace concepts. 

A concept is a named set of requirements. There's a detailed [article on cppreference](https://en.cppreference.com/w/cpp/language/constraints), but here's the gist: it allows distinguishing between multiple template overloads based on the template parameter, similar to SFINAE. However, concepts offer enhanced clarity by separating the declaration from usage and enabling easier combinations, resulting in a more concise code.

### A trivial trait-based concept and a requirement

Let's make minimal modifications to our `makeString(String&& s)` and `makeString(Numeric value)` overloads and take a look at the first attempt:
{{< highlight cpp "linenos=table,hl_lines=9 11-12 19 21">}}
namespace traits
{
template <typename T> 
inline constexpr bool isString = std::is_constructible_v<std::string, T>;
}

// a concepts that uses a constexpr bool trait to check its applicability
template <typename T>
concept IsString = traits::isString<T>;

template <IsString String>
std::string makeString(String&& s) 
{
    return std::string(std::forward<String>(s));
}

// a concept uses requires clause: code in braces should compile
template <typename T>
concept HasStdConversion = requires (T number) { std::to_string(number); };

std::string makeString(HasStdConversion auto number)
{
    return std::to_string(number);
}
{{< /highlight >}}

A few explanations below:
* Line #9: a trivial concept is created to demonstrate the usage of a constexpr bool as a requirement. Although the `<concepts>` provides `std::constructible_from`, we'll use a custom trait-based concept as a showcase. 
* Line #11-12: `IsString` concept is used instead of the `typename` keyword to keep typing to a minimum. Please note that there is no more need for SFINAE or trailing return type. 
* In line #19, a requirement is introduced: `HasStdConversion` is a concept that requires the compilation of `std::to_string(number);` with `T number` to ensure that the std::to_string conversion exists. Still not rocket science.
* Line #21: `HasStdConversion` is utilized to constrain the `auto` type of the parameter. This approach is equivalent to the one from line #11-12 and requires even less tying. However, recarding the code clarity, my personal preference is to use the `template` keyword to clearly indicate that the following method is a template. 

### Concepts: conjunctions and disjunctions

Now the code can be further cleaned up using `<concepts>` instead of hand-written traits. Also, drop the `traits` namespace and rewrite every function declaration to use concepts instead of declarations. Here we go:

* Assume that type `T` is string, if `std::string` is `std::constructible_from` it: 
{{< highlight cpp>}}
template <typename T>
concept IsString = std::constructible_from<std::string, T>;

template <IsString String>
std::string makeString(String&& s) 
{
    return std::string(std::forward<String>(s));
}
{{< /highlight >}}

* `HasStdConversion` will require that `std::to_string` could be called on its argument:
{{< highlight cpp>}}
template <typename T>
concept HasStdConversion = requires (T number) { std::to_string(number); };

template <HasStdConversion Numeric>
std::string makeString(Numeric number)
{
    return std::to_string(number);
}
{{< /highlight >}}

* `HasToString` is a concept that requires `object.to_string()` with `Object object` parameter to be valid and return something that is `std::convertible_to<std::string>`. In this case a requirement will check the expression type using another concept, in addition to a base requirement to compile the code in the braces:
{{< highlight cpp "hl_lines=4">}}
template <typename T>
concept HasToString = requires (T&& object) 
{ 
    {object.to_string()} -> std::convertible_to<std::string>; 
};

template <HasToString Object>
std::string makeString(Object&& object) 
{
    return std::forward<Object>(object).to_string();
}
{{< /highlight >}}

* Finaly, the tricky part. Similarly to SFINAE approach, `IsContainer` is a requirement on type to be iterable: `requires (T&& container) { std::begin(container); }`. However, a string is a container, too, and this will lead to ambiguity when calling `makeString("hello")` because both overloads will match. There is a point in out code where [_requires clause_](https://en.cppreference.com/w/cpp/language/constraints#Requires_clauses) is useful to require a boolean constraint.
{{< highlight cpp "hl_lines=5">}}
template <typename T>
concept IsContainer = requires (T&& container) { std::begin(container); };

template <typename Iterable>
    requires (IsContainer<Iterable> && !IsString<Iterable>)
std::string makeString(Iterable&& iterable) 
{
    //...
}
{{< /highlight >}}

I'd prefer, however, to define `IsContainer` just as something iterable, and `IsString` as a `IsContainer` that can be used to construct an `std::string`. This way when something is both `IsString` and `IsContainer`, the first one will have a priority because the more constrained concept is preferred by the compiler. 

That are some great articles within-depht explanation of [conjunctions, disjunctions](https://akrzemi1.wordpress.com/2020/03/26/requires-clause/) and [ordering by constraints](https://akrzemi1.wordpress.com/2020/05/07/ordering-by-constraints/) that I highly recommend to look at. 

Perhaps, it's a good point to summarize the code and sync-up with a reader's imagination by providing the full source:
{{< highlight cpp >}}
// makeString.hpp
#pragma once
#include <string>
#include <type_traits>
#include <concepts>

template <typename T>
concept IsContainer = requires (T&& container) { std::begin(container); };

// IsString is more constrained than IsContainer, 
// so it will have a priority wherever possible
template <typename T>
concept IsString = IsContainer<T> && std::constructible_from<std::string, T>;

template <typename T>
concept HasStdConversion = requires (T number) { std::to_string(number); };

template <typename T>
concept HasToString = requires (T&& object) 
{ 
    {object.to_string()} -> std::convertible_to<std::string>; 
};


template <HasStdConversion Numeric>
std::string makeString(Numeric number)
{
    return std::to_string(number);
}

template <HasToString Object>
std::string makeString(Object&& object) 
{
    return std::forward<Object>(object).to_string();
}

template <IsString String>
std::string makeString(String&& s) 
{
    return std::string(std::forward<String>(s));
}

template <IsContainer Iterable>
std::string makeString(Iterable&& iterable) 
{
    std::string result;
    for (auto&& i : iterable)
    {
        if (!result.empty())
            result += ';';

        // a constexpr if, so the compiler can omit unused branch and
        // allow non-copyable types usage
        if constexpr (std::is_rvalue_reference_v<decltype(iterable)>)
            result += makeString(std::move(i));
        else 
            result += makeString(i);
    }
    return result;
}
{{< /highlight >}}

Compile, run: it works!

### Ordering overloads by constraints 

Speaking of extensibility, let's imagine that our `makeString` is a library header, and we're unwilling to modyfy it. In such a scenario, consider the class that is both `IsContainer` and `HasToString`:

{{< highlight cpp "hl_lines=5-10 18">}}
struct C
{
    std::string m_string;
  
    auto begin() const { return std::begin(m_string); }
    auto begin()       { return std::begin(m_string); }
    auto end() const   { return std::end(m_string); }
    auto end()         { return std::end(m_string); }

    std::string to_string() const { return "C{\"" + m_string + "\"}"; }
};

int main()
{
    // ...
    std::cout << makeString(makeVector())
              << std::endl
              << makeString( C { "a container with its own to_string()" } )
              << std::endl;

}
{{< /highlight >}}

<details>
<summary>There is an ambiguity in the <code>makeString(C{...})</code> call because the compiler cannot determine whether the <code>IsContainer</code> or <code>HasToString</code> overload is better to apply.</summary>
{{< highlight cpp >}}
[build] main.cpp:58:18: error: call to 'makeString' is ambiguous
[build]               << makeString( C { "a container with its own to_string()" } )
[build]                  ^~~~~~~~~~
[build] makeString.hpp:37:13: note: candidate function [with Object = C]
[build] std::string makeString(Object&& object) 
[build]             ^
[build] makeString.hpp:49:13: note: candidate function [with Iterable = C]
[build] std::string makeString(Iterable&& iterable) 
[build]             ^
{{< /highlight >}}
</details><br>

Those familiar with the era of SFINAE can imagine the magnitude of the tragedy, and those who have already seen Andrzej's article on [ordering by constraints](https://akrzemi1.wordpress.com/2020/05/07/ordering-by-constraints/) can imagine the solution.

The ambiguity arises because two constrained methods have the same priority, with neither constraint <abbr title="P subsumes Q if P contain all the elements of Q">subsuming</abbr> the other. The solution is straightforward: introduce a third overload that is more restrictive than the conflicting ones.  

{{< highlight cpp "hl_lines=13-18 25">}}
struct C
{
    std::string m_string;
  
    auto begin() const { return std::begin(m_string); }
    auto begin()       { return std::begin(m_string); }
    auto end() const   { return std::end(m_string); }
    auto end()         { return std::end(m_string); }

    std::string to_string() const { return "C{\"" + m_string + "\"}"; }
};

template <typename Container>
    requires IsContainer<Container> && HasToString<Container>
std::string makeString(Container&& c)
{
    return std::forward<Container>(c).to_string();
}

int main()
{
    // ...
    std::cout << makeString(makeVector())
              << std::endl
              << makeString( C { "a container with its own to_string()" } )
              << std::endl;
}
{{< /highlight >}}

This way, the ambiguity is resolved by introducing a new rule, preserving the library implementation.

# Variadic templates

There is another topic the developer could greatly benefit when using carefully. Back in the days, we had a functions with varied arguments count. Still there are printf-like set of functions, where developer feeds variable amount of parameter, and the compiler tries to save him from a numerous pitfalls. 

For instance, Unreal Engine incorporates the format string checks into a custom preprocessor, while `std::format` or [`fmt`](https://github.com/fmtlib/fmt) provides us an error message that is [a bit cryptic](https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahUAUgBMAIUtXyKxqiIEh1ZpgDC6egFc2TA3cAGQImbAA5PwAjbFIQAGZyAAd0JWIXJi9ffwMUtOchELDIthi4xIdsJwyRIhZSIiy/AJ57bEcCplr6oiKI6NiE%2BzqGppzWpRHe0P7SwfiASnt0H1JUTi5LeNDUXxwAajN4j0FSNhYiI9wzLQBBG9vQon3z0Ign/frgVHJ91CR6gAqQELB5mADsNju%2Bxh%2B0mmBAIFO5yIEEsFghVi0EIAIodITxcejfl9UKD4lD7uCcVwlvRuABWfgBLg6cjobgeWy2OErNbYQ4WeJ8chEbS0pYAaxADIZADpwRZpBYAJwqrQANgZyq0DI1hm40mZ4vZ3H4ShAWlF4qWcFgKAwbCSDFilGojudjDiu2MwAA%2BkRSD4mJLyDgAG4EdYANQI2AA7gB5JLMbgiuj0IixC0QKImqKheoATzT/ALrFIRcTUV0VTFvH4jo4wkTTHoJdZ/BwUR8wA8EnoFobYew5xMkk7YYIpDrBHD2CHbOw6iqPizpcownaJvoBCipGLXhwG8DBDYpaWNCMwCUsYTydTw8EwjEEk4MjkwmUak0k/0iSMEw0G5axDD3C1ICWdAkk6IcAFo4PhI4cXMaxbCFfh0HnUhSAIHAIIgJZKmqVwIHcMYWnIYIZhKMpclSdIhAo%2Bj8gyPpaMGCZ2lnIRulGbxmgMYjOj46ZigGOIJimZipJ6diJKkIi%2BXWRSDS4JlyBZNkOS4fZ1AADg1OCNWkP5AOAfYIEDYNJQWSz8GIMhBWFBZ%2BHrHRbQddAnRdCgqAgD1fJAcNUCSJI/XDHgVT9Iws0mP11BMkdIxjOMkxTFl0wYLNSBzPNJ3LYsN0Kytq1rJwNybZgiFbdsTW7Xt%2B3oQcNxwMdgAnNlCBnap50Xfhl1XddhyebdJ13fdDywDYRVPc8G0va9bzSh9Mv4Z9RHESQPw278NBNfRWnM4C0NAiaCKgmCMng%2BETpsaxYoXIhMOw3D8PgIjuJIgIyKYTwBPGKi/vkuZJOSBjOhk8HWMKGiFK4joamkgHKOEpG5Lh0GhOR7JUamEG6J4JTVhUom1I0rTMO4PSTJeJQQv2SKVTlR7JnswgSFIZyibcm0liQbAWBwOJCPIaVpGkOULHBBlwS0LQVXiaRZZVCW1KNchzx4eXNJNHTzQMdyJTUix%2BHPBkrUp00uF5ztbUQCAUBWIgkjXN0Au8z1YnCdgNkS0y2Hp1BGailmLiegaObIPCDA219ttkXaVH2v8DHjA8kgvcnjUnHTEzXV3nnQGgaYDoOQ%2BZ1nnggLwfK9bnXOtO3yEdkBncL93Aq9H2OG4AyjNpn0TEs6yQ0b7Ao7e2P5Hj99E/kPbfzZf9yHTlhM%2Bb%2Bl1Jz7TuHzl2132Yu9MM4zTKHiyrKDMfLNrr2ua2RZbY8/nBeF6g6UNM2ZUtvWzXsS0TcX5qXiDvKmNsgHG2wmkVw0ggA%3D%3D%3D) but much better than an Undefined Behavior in runtime. 

Having that, why don't provide a variaric template for `makeString("xs: ", xs, "; and the double is: ", PI)`? At least, that's a nice excercise. 

## Basic syntax and considerations

{{< highlight cpp "linenos=table">}}
template <typename... Args>
constexpr int uselessCount(Args&&... args)
{
    return sizeof...(args);
}

int main(int argc, char**)
{
    return uselessCount(1,2,3) 
          + uselessCount();
}
{{< /highlight >}}

Let me describe this masterpiece line-by-line:
* Line #1: a [parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack) is declared with an obvious syntax. A parameter pack is _is a template parameter that accepts __zero or more__ template arguments_.
* Line #2: 'Args&&... args' is a forwarding reference to a pack.
* Line #4: a special `sizeof...` that is evaluated at a compile-time to a number of pack's arguments.
* Line #9: an obvious way to use.
* Line #10 may be unobvious, but still correct: a pack of zero arguments is provided. 

### Parameter pack expansion and recursive functions

Function arguments may contain a regular parameter alongside a parameter pack. For recursive functions, it provides a convenient way to break the recursion once the parameter pack is empty. Consider the `makeString(T&& first, Rest&&... rest)` in pseudocode:
 * `Rest` has parameters: `return makeString(first) + makeString(rest...);`.
 * `Rest` is an empty parameter pack: `return makeString(first)`; 

While the pseudocode above demonstrates the idea, it does not apply to our code as-is: `makeString(T&& first, Rest&&... rest)` accepts one or more parameters. Since we already have a set of single-parameter implementations, an empty-pack call will be ambiguous. The solution is constraining the variadic overload to accept at least two parameters.

Here is an approach from the past: a recursive function `makeString(First&& first, Second&& second, Rest&&... rest)` accepts at least two parameters plus an optional variadic pack. It converts the first parameter to a string and calls `makeString(second, rest...)` recursively. Once the `rest...` pack is empty, it expands to a non-recursive `makeString(second)` call.

{{< highlight cpp "linenos=table,hl_lines=8">}}
// makeString.hpp
// ... single-parameter implementations cut from the listing  ...

template <typename First, typename Second, typename... Rest>
std::string makeString(First&& first, Second&& second, Rest&&... rest)
{
    return makeString(std::forward<First>(first)) 
         + makeString(std::forward<Second>(second), std::forward<Rest>(rest)...); 
}

// main.cpp
// ... stripped ...
std::cout << makeString("a ", 
                        std::string_view("variadic " ), 
                        std::string("with a double: "), 
                        3.14)
          << std::endl;
{{< /highlight >}}

Line #8 introduces a parameter pack expansion to the reader: compiler expands `expression(pack)...` to a comma-separated list of zero or more `expression(argument)`patterns. For example, `std::forward<Rest>(rest)...` expands to:
* if `Rest` has no parameters: expands to nothing, ignoring the `std::forward<Rest>(rest)...` expression on line #8.
* if `Rest` has one parameter: `std::forward<Rest_0>(rest_0)` is added, resulting in a call like `makeString(std::forward<Second>(second), std::forward<Rest_0>(rest_0));`.
* if `Rest` has N parameters, each parameter is individually expanded using the enclosing expression: `std::forward<Rest_0>(rest_0), std::forward<Rest_1>(rest_1), ..., std::forward<Rest_N>(rest_N)`. 

In summary, amount of parameters to `makeString` call on line #8 depends on the size of the parameter pack. The whole `std::forward<Rest_#>(rest_#)` expression is "copy-pasted" for each added parameter.  It's important to note that this behavior applies to any kind of packed expression, not just `std::forward`, so `f(g(pack)...)` would be expanded as `f(g(pack_0), g(pack_1), ..., g(pack_N))` accordingly.

Given that, compile, run, enjoy!

{{< highlight cpp>}}
a: A; b: B{1}; pi: 3.141593
xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000
Hello, world!!1
two ; non-copyables
C{"a container with its own to_string()"}
a variadic with a double: 3.140000
{{< /highlight >}}

### Constrained version

However, we can improve further. Instead of using the `Second&& second` parameter to avoid overloading conflicts, we can leverage constraints by requiring `sizeof...(Rest) > 0`:

{{< highlight cpp "hl_line=2">}}
template <typename First, typename... Rest>
    requires (sizeof...(Rest) > 0)
std::string makeString(First&& first, Rest&&... rest)
{
    return makeString(std::forward<First>(first)) 
         + makeString(std::forward<Rest>(rest)...); 
}
{{< /highlight >}}

Less code is less of a mental load on the reading developer, so it's a bit better. Can it be even better? 

### Fold expressions

Well, since C++ 17 we have [fold expressions](https://en.cppreference.com/w/cpp/language/fold). Having a parameter pack `pack` and an unary or binary operation 'x' we can expand `(pack x ...)` to `(pack_0 x (pack_1 x pack_2))` and so on. There are several options including a left-associative fold `(... pack x)` with dots on the left, a right-associative with dots on the right, an option for unary or binary operations, an `init` variable for an empty pack, and so on. 

To keep this article size reasonable, I dare to forward the reader to the Fluent C++ blog for in-depth details: [part 1 - basics](https://www.fluentcpp.com/2021/03/12/cpp-fold-expressions/) and [part 2 - advanced usage](https://www.fluentcpp.com/2021/03/19/what-c-fold-expressions-can-bring-to-your-code/). 

As we're dealing with strings, I expect the left-associative fold over `operator+=` to be more performant than using `operator+` because the latter will produce temporaries for holding intermediate results in case of multiple strings involved: `return ((a + b) + c) + d` result in 3 temporaries besides `a, b, c`, while `return ((a += b) += c) += d;` avoids them.

{{< highlight cpp>}}
template <typename... Pack>
    requires (sizeof...(Pack) > 1)
std::string makeString(Pack&&... pack)
{
    return (... += makeString(std::forward<Pack>(pack)));
}
{{< /highlight >}}

Finally, I'd call it a day and summarize:
 * A template function can (and probably should) be constarined to let the compiler fail early and provide a concise error context in case of misuse. Even if it has no overloads yet.
 * SFINAE is a reasonable fallback when concepts are unavailable.
 * Use static_assert and intentional compilation failures to get an insight on template expansion in case of misunderstanding.
 * Perfect forwarding is our friend: it may offer better performance and relax 'copyable' requirement on the arguments.
    * ... but not for trivial types. Some times are cheaper to pass by-copy than by-reference. 
 * Variadic templates are useful way to process multiple arguments in a row. Fold expression might be even better.

# The code I argee with

Let's sync up on the result of this journey:
{{< highlight cpp>}}
// makeString.hpp
#pragma once
#include <string>
#include <type_traits>
#include <concepts>

template <typename T>
concept IsContainer = requires (T&& container) { std::begin(container); };

// IsString is more constrained than IsContainer, so it will have a priority wherever possible
template <typename T>
concept IsString = IsContainer<T> && std::constructible_from<std::string, T>;

template <typename T>
concept HasStdConversion = requires (T number) { std::to_string(number); };

template <typename T>
concept HasToString = requires (T&& object) 
{ 
    {object.to_string()} -> std::convertible_to<std::string>; 
};


template <HasStdConversion Numeric>
std::string makeString(Numeric number)
{
    return std::to_string(number);
}

template <HasToString Object>
std::string makeString(Object&& object) 
{
    return std::forward<Object>(object).to_string();
}

template <IsString String>
std::string makeString(String&& s) 
{
    return std::string(std::forward<String>(s));
}

template <IsContainer Iterable>
std::string makeString(Iterable&& iterable) 
{
    std::string result;
    for (auto&& i : iterable)
    {
        if (!result.empty())
            result += ';';

        // a constexpr if, so the compiler can omit unused branch and
        // allow non-copyable types usage
        if constexpr (std::is_rvalue_reference_v<decltype(iterable)>)
            result += makeString(std::move(i));
        else 
            result += makeString(i);
    }
    return result;
}

template <typename... Pack>
    requires (sizeof...(Pack) > 1)
std::string makeString(Pack&&... pack)
{
    return (... += makeString(std::forward<Pack>(pack)));
}
{{< /highlight >}}
{{< highlight cpp>}}
// main.cpp
#include <iostream>
#include <vector>
#include <set>
#include "makeString.hpp"

struct A
{
    std::string to_string() const { return "A"; }
};

struct B
{
    int m_i = 0;
    std::string to_string() const { return "B{" + std::to_string(m_i) + "}"; }
};

struct NonCopyable
{
    std::string m_s;
    NonCopyable(const char* s) : m_s(s)  {}
    NonCopyable(NonCopyable&&) = default;
    NonCopyable(const NonCopyable&) = delete;

    std::string   to_string() const &  { return m_s; }
    std::string&& to_string() &&       { return std::move(m_s); }
};

struct C
{
    std::string m_string;
  
    auto begin() const { return std::begin(m_string); }
    auto begin()       { return std::begin(m_string); }
    auto end() const   { return std::end(m_string); }
    auto end()         { return std::end(m_string); }

    std::string to_string() const { return "C{\"" + m_string + "\"}"; }
};

template <typename Container>
    requires IsContainer<Container> && HasToString<Container>
std::string makeString(Container&& c)
{
    return std::forward<Container>(c).to_string();
}

int main()
{
    A a;
    B b = {1};

    const std::vector<int> xs = {1, 2, 3};
    const std::set<float> ys = {4, 5, 6};
    const double zs[] = {7, 8, 9};

    std::cout << "a: " << makeString(a) << "; b: " << makeString(b) 
              << "; pi: " << makeString(3.1415926) << std::endl
              << "xs: " << makeString(xs) << "; ys: " << makeString(ys) 
              << "; zs: " << makeString(zs)
              << std::endl;

    std::cout << makeString("Hello, ") 
              << makeString(std::string_view("world")) 
              << makeString(std::string("!!1")) 
              << std::endl;

    auto makeVector = []()
    { 
        std::vector<NonCopyable> v;
        v.emplace_back("two ");
        v.emplace_back(" non-copyables");
        return v; 
    };

    std::cout << makeString(makeVector())
              << std::endl
              << makeString( C { "a container with its own to_string()" } )
              << std::endl;

    std::cout << makeString("a ", std::string_view("variadic "), std::string("with a double: "), 3.14)
              << std::endl;
}
{{< /highlight >}}
And the output is:
{{< highlight cpp>}}
a: A; b: B{1}; pi: 3.141593
xs: 1;2;3; ys: 4.000000;5.000000;6.000000; zs: 7.000000;8.000000;9.000000
Hello, world!!1
two ; non-copyables
C{"a container with its own to_string()"}
a variadic with a double: 3.140000
{{< /highlight >}}

# References
A section also known as "I saw an interesting link somewhere in a text wall above":
* [CMake (Wikipedia)](https://en.wikipedia.org/wiki/CMake)
* [cppreference: Template specialization](https://en.cppreference.com/w/cpp/language/template_specialization)
* [Cpp Core Guidelines: pass cheaply-copied types by value](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f16-for-in-parameters-pass-cheaply-copied-types-by-value-and-others-by-reference-to-const)
* [Arthur O’Dwyer's blog - pass string_view by value](https://quuxplusone.github.io/blog/2021/11/09/pass-string-view-by-value/)
* [cppreference: Function template](https://en.cppreference.com/w/cpp/language/function_template)
* [Overload resolution of function template calls](https://learn.microsoft.com/en-us/cpp/cpp/overload-resolution-of-function-template-calls)
* [cppreference: Substitution Failure Is Not An Error](https://en.cppreference.com/w/cpp/language/sfinae)
* [cppreference: Integer Promotions](https://en.cppreference.com/w/c/language/conversion)
* [cppreference: Type Traits](https://en.cppreference.com/w/cpp/header/type_traits)
* [cppreference: Metaprogramming library](https://en.cppreference.com/w/cpp/meta)
* [isocpp blog on _universal (or forwarding) references_](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)
* [cppreference: member functions](https://en.cppreference.com/w/cpp/language/member_functions)
* [cppreference: copy elision](https://en.cppreference.com/w/cpp/language/copy_elision)
* [me: Practical usage of ref-qualified member function overloading](../ref-qualifiers)
* [cppreference: Constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints)
* [cppreference: requires clause](https://en.cppreference.com/w/cpp/language/constraints#Requires_clauses)
* [Andrzej's C++ blog - ordering by constraints](https://akrzemi1.wordpress.com/2020/05/07/ordering-by-constraints/)
* [Andrzej's C++ blog - conjunctions, disjunctions (Requires-clause)](https://akrzemi1.wordpress.com/2020/03/26/requires-clause/)
* [`fmt` on Github, in case your standard library has no `std::format`](https://github.com/fmtlib/fmt)
* [cppreference: parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack)
* [cppreference: fold expressions](https://en.cppreference.com/w/cpp/language/fold)
* [Fluent C++: C++ Fold Expressions 101](https://www.fluentcpp.com/2021/03/12/cpp-fold-expressions/)
* [Fluent C++: What C++ Fold Expressions Can Bring to Your Code](https://www.fluentcpp.com/2021/03/19/what-c-fold-expressions-can-bring-to-your-code/)