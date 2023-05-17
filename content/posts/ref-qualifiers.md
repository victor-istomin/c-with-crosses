---
title: "Practical usage of ref-qualified member function overloading"
date:  2023-05-16T00:59:12Z
summary: "Recently, I discovered that <code>std::ranges</code> prohibits the creation of dangling iterators and provides an <code>owning_view</code> to take ownership of temporaries. Digging into the details led me to a valuable feature that can be used to make code safer."
tags: ["cpp"]
author: "Me"
draft: false
---
Recently, I discovered that <code>std::ranges</code> prohibits the creation of dangling iterators and provides an <code>owning_view</code> to take ownership of temporaries. Digging into the details led me to a valuable feature that can be used to make code safer.

<details><summary>Article updates (2023-05-17)</summary>
<ul>
  <li>mistakenly said that lvalue-overload can't match rvalue object: <a href="#a-ref-qualified-member-function">it can</a>.</li>
 <ul> 
</details>

## Intro (the source of my excitement)

Consider the following code that creates a temporary array using `get_array_by_value`, then uses the temporary when safe, rejects compilation when not safe, and takes ownership of the temporary when needed.

Although `std::ranges` is not the primary focus of this article, it's worth to briefly touch on it before moving on to the main topic. For those who are new to `std::ranges`, a brief explanation will follow below.

{{< highlight cpp>}}
#include <algorithm>
#include <array>
#include <ranges>
#include <type_traits>
#include <iostream>
#include <numeric>

int main()
{
    auto get_array_by_value = [] { return std::array{0, 1, 0, 1}; };

    // (1) a search for the max element in the temporary 
    auto dangling_iter = std::ranges::max_element(get_array_by_value());
    static_assert(std::is_same_v<std::ranges::dangling, decltype(dangling_iter)>);
    // Compilation error below: 
    // no match for 'operator*' (operand type is 'std::ranges::dangling')
    // std::cout << *dangling_iter << '\n';

    // (2) However, the code below works fine:
    std::array<int, std::size(get_array_by_value())> copied;
    auto copy_result = std::ranges::copy(get_array_by_value(), std::begin(copied));
    static_assert(std::is_same_v<std::ranges::dangling, decltype(copy_result.in)>);
    
    // (3) and this is fine, too:
    auto isOdd = [](int i){ return 0 != (i % 2);};
    auto filtered = get_array_by_value() | std::views::filter(isOdd);
    return std::accumulate(std::begin(filtered), std::end(filtered), 0);
}
{{< /highlight >}}

1. Here, we create a temporary array and search for its maximum element. Since the temporary is destroyed at the end of the _full-expression_[^at-the-semicolon], a trivial implementation might have returned an iterator to an element of the destroyed array. However, `std::ranges` detects this and returns a special type called `std::ranges::dangling`. This approach accomplishes two things:
- `std::ranges::max_element` is executed, any side effects are preserved (e.g. stdout output);
- `dangling_iter` is impossible to dereference, preventing access to the destroyed array.

2. A temporary is valid to be copied from, but the iterator to the end of the temporary range is `std::ranges::dangling` so the developer can't reuse it. 

3. The temporary is filtered and should be destroyed at the end of the _full-expression_, but the view takes ownership over it using the [owning_view](https://en.cppreference.com/w/cpp/ranges/owning_view). This means that it is perfectly valid to iterate over the filtered range after the original temporary has been destroyed.

[^at-the-semicolon]: Here, at the semicolon

All the magic above may suggest that when accessing the internal state of a large object like `RawImageBytes`, or when creating a most-recent-N subrange of the `RingBuffer`, we can do better than simply returning a pointer or reference to the internal data buffer and hoping that nobody will accidentally use a dangling pointer.

## The problem

Consider the `MyRawImage` class that holds large image data. Some computational algorithms may need image's data as a buffer, so `MyRawImage::data()` method is needed to access the whole buffer. Here is the first attempt to implement it:
{{< highlight cpp>}}
struct Pixel
{
    int r = 0;
    int g = 0;
    int b = 0;

    friend auto operator<=>(const Pixel& a, const Pixel& b) = default;
};

struct Metadata { /* some metadata here, width, height, etc. */ }; 

class MyRawImage
{
    std::vector<Pixel> m_buffer;
    Metadata           m_metadata;
public:
    MyRawImage(std::vector<Pixel> src) : m_buffer(std::move(src)) {}

    const Pixel& operator[](int index) const { return m_buffer[index]; }
    Pixel&       operator[](int index)       { return m_buffer[index]; }

    const std::vector<Pixel>& data() const   { return m_buffer; }
    const Metadata& information() const      { return m_metadata; }
};
{{< /highlight >}} 
Up to this point, the code appears to be following standard and conventional practices. However, I believe there is room for improvement, and that's why:
{{< highlight cpp>}}
MyRawImage loadImage(int i)
{
    return std::vector<Pixel>(i * 100, Pixel {i, i, i});
}

MyRawImage problematic(int i)
{
    std::vector<Pixel> filtered;
    auto filter = [](Pixel p) 
    { 
        p.r = std::min(p.r, 0xFF); 
        p.g = std::min(p.g, 0xFF); 
        p.b = std::min(p.b, 0xFF);
        return p; 
    };

    // oops: 
    // equivalent of `auto&& ps =  loadImage(i).data(); for (Pixel p : ps) { ... }`
    // loadImage() returns a temporary, temporary.data() reference is stored, then
    // for-loop iterates over a stored reference to a property of deleted temporary 
    for(Pixel p : loadImage(i).data())
        filtered.push_back(filter(p));

    return filtered;
}

Pixel fine(int i)
{
    auto max = [](auto&& range) -> Pixel 
    { 
        return *std::max_element(std::begin(range), std::end(range)); 
    };

    // this one is fine: a temporary will be destroyed after the max() calcualtion
    return max(loadImage(i).data()); 
}

int main(int, char**)
{
    constexpr static int pattern = 0x12;
    constexpr static Pixel pixelPattern = Pixel { pattern, pattern, pattern };

    Pixel maxPixel = fine(pattern);
    assert(maxPixel == pixelPattern);

    MyRawImage img = problematic(pattern);
    auto isGood = [](const Pixel& p) { return p == pixelPattern; };
    assert(img.data().end() == std::ranges::find_if_not(img.data(), isGood));
}
{{< /highlight >}} There is a [full source](https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahUAUgBMAIUtXyKxqiIEh1ZpgDC6egFc2TEABOLXJ3ABkCJmwAOT8AI2xSEABWEIAHdCViFyYvX38g9MzsoQio2LYEpNT7bEdnIREiFlIiPL8A4Nr6nKaWojKY%2BMSUkKVm1vaCrvH%2BwYqq0YBKe3QfUlROLksAZkjUXxwAajMdj1JjYGwlU9wzLQBBXf3D7BOz8SUVVtv7p4s9kwDj5jqcPAA3bBOMi/R7PIGvd4eCTAMjEJBsWFPR7jUg%2BJxHVQEdR1P5mADsNkeRxpR0iRCOpHeABEjlpTlSHrS6cIjsAWWyOX9ufSjnEBeydpzhbSaKQCO4jiwfCQjug0okWCRSGDTszbhAMExxoTiaSLAA2JXkI5Gk1Ekn0SxWuJLAU4GjK%2BhEIVw8n6qVknFEPEEgCy2GamC1LBOlKOAHoAFRHJToDhHDhRmNHJCJbA2gDuBEwRCQNrzBGASCINsjqAAdEckwm4wGrCc4Y8DiwvkcwwBPABKLELAEk2CwrmTKTKaeNMCAQJDoTqzg7STtcJmAPpxHw0GiJX1c2kR7PNblXmlsHdZljR5ontI%2BOL0AioEBz/vD0cTqfYBAC5Liu2pghuTpbqmGxuiAu77oeiRAUQi4gGw6CQkBMFuhSNj%2BkGp40naDIQc6aoahcYHJDYyTMhAoqRDg6husRcYdqQkbrEw8EHkeOrUYx2DqGYtEcm236kZa17kZqVE0XRDFMExbrXrhjKcaQ3G3ghfEiVYgnCaJUriV2hG2kIJrAcuUJgeuZqQXcUmPiwEAsRZDI0mpHFEFxPGIWuHYUvq1K0qx54PjGZGRIIpCTg0TCueZxoedyXkaVpd6RhFT7GUFM7tgRg4juOk5XEc9DoA%2B/5XPRvIEEsM6cty3m%2BVZoEwnZjoGgQzZHDwWghKajpsQQNqjXSQUNYGfrBdiDxFX%2BpVvGkpDoG%2B2BxR%2BtUMvVjXfm1NkdR4pFQXQ3r5pgJ7csqqpnUQiQCnpIl0RBRxpDhIWefG37cmkDZMnqqYoUubCRBAf0UGy6gAGLQ1NgWfVef38oDVmgwlyM2loMNw2JP20n94qo8DaFg4TWM4/D%2BM0i1mlvXjiN5dNc3cgmrboOqShwd%2BbNHNgACOPgEOCEjMAy6A0CcFpaDd6DOmRaRKAK5WVZg1WAfVDbOa5YkxUcECvWkRxwYrOHxg2FvidLPOthVVVLYltPGkqRz3WwGQXKQA42m7HstAOWsxo72B8cwmx0kr4xkNgmDkDbrt5txMUALQVeqdL3ZR1xqpCTKxlHHGYOpodAm8qqxit6qJEQA5qpLOCMPdRe%2B2Q/udmZMUG/Zb3Gyr9sAfRSyB80rkNYj3J3RdDYvkoSB7iwqAANa0Awmfg0sVOmc16VHJPhcnnlpmvXQUTbXSY9PLOiOy5mLDqI91HPRAsvy1JFwmNgbrJ7cQ11O3qXfXHrSJ2zY0Z3x3HUDaYtkKoQSMAMG78rgrCBqhdwEBEGf3hv/WkTNpSI15mWAgSshBvCIbvSInAXYt09rXYs9B6BijeDgXE6ABwxyVDQTOCc3iTnUIlcQ9BUA%2BAkPFb8IDeEQDtmrB2mttYbwZpfWafxRSTjBvSG0qAkAtCTEmC%2BuFvzESEitIGWoPw8gZGkLUmduKA2xjwCwV1QruSMUyWYzhUC/wYWkeyqgrGJBsTsVkr01KWKINYm0oTwlvT8XTXBBFuSvV4cEwJ5DT6RP8ZvMyvZvhEAgEk7uepAbeMdL4sJGSD6IwWiVACdI2AoxSZXdam1UDgxiUwTJ10VToAjgAcQ5kXQGT1aKGncp4hWZt2I7yNoUhpPi2l43wsza6Xxq70TqcPFyQ80E4UCcTVCGCuYgBPpgHcBAaA7iYOgXJBB1lyLGkoPpWB5FLNlBQiAdiLCZMPg8LgKx6DcGSPwAIXAdDkHQNwDwthbCpjWBsN4uw%2BDkCINoX5KxF6jEMNwaQQKUVgu4PwLmIRkUgt%2BeQOAsAUAYHdgwRIlBqBUu8YwJIwAlCsEVkgK55AcDgg/NgAAagqQsAB5DUwLEV7y5hAOIuK4iRH9twRFsrWBeyFXEXQNkFX8CpRwYQQqmD0AHLinAk4TCSBJVyggHEnDC2uLioSUIVRbFBfSOouL3xxBoV4HAuKQw3M1SsGgRgWUCuwMK0Vmq5DCDEKLKQshBDCGUGoTQ5r9A8EMJcNAULrCGAIHELmkAVjqnilzLgycFx6nMNYWwAJ%2BAYUSPKZh8AVgOBsjkNwSkpgBDTeEChCwRhpoyFkeKnaDCDpKEweYwwkhppbdaxoEw2jeA6AYWd8U%2BitEnZUft9gF0jpnQuzdiweDNthZsKQfyAU4vNeCrgRx1AAA4LTJwtNIPkqAPEQBDD4Jgi83QQHwMQMgJwATHv4MSnQG9yDopqP8rg2LyDAtBTeglIAiUosg7Biw/A2AgHJBaBsFpkjkmkMke9yQLQ7GSJRwIabEO1vxUi9DZLEAQBQGsIgL5axUBGdSpl0R2BbAfU%2Bl9b6P1fp/UsfgMdAMNoMPG0QAjOAyEjYoFQGhcWpvIIWC4aR/WYq4IChDuKb1CpVJxuud7H3PtfcAd9%2BtxO/v1l4XjD0EWScYySyDeYHwjFcvp%2BDOGAQNh2NICwyRpAyB2C%2Bl9WhAgWCM9ehjhKPMQYvVwHYV6kMMfA6i8gucsiuGkEAA%3D%3D%3D) just in case some of us don't enjoy guessing missed includes.

The code above aims to accomplish the following:
 - `fine()` method 'loads' image that consists of one hundred pixels with a value of `{0x12,0x12,0x12}` and finds the maximum among them.   
 - `problematic()` method loads similar image and clamps every pixel's component to the maximum value of 0xFF and ensures that no pixels deviate from the 0x12 pattern.
 
I believe that the reader can solve both mathematical problems mentally, but the thing is the provided code can't until the C++23:[^fixed-in-cpp23]
[^fixed-in-cpp23]: Temporaries in the range-based for will be fixed in C++23, see the _Temporary range expression_ chapter [in the range-based for documentation](https://en.cppreference.com/w/cpp/language/range-for). However, `const Bar& lvalue = foo().bar();` still has the same pitfall. 

{{< highlight shell >}}Program returned: 139
output.s: /app/example.cpp:77: int main(int, char**): 
          Assertion `img.data().end() == std::ranges::find_if_not(img.data(), isGood)' failed.
{{< /highlight >}}

The problem arises when obtaining a reference or pointer to a property of a temporary object. Often, it is permissible, but there is a risk of misuse by developers, leading to dangling references and undefined behavior.

A short STL example:
{{< highlight cpp>}}std::string getMessage() { return "I'm a long enough message to avoid SSO"; }
// fine, the string is destroyed at the semicolon
int length = strlen(getMessage().c_str());  

// bad, the string has been destroyed at the semicolon below
const char* dangling = getMessage().c_str();
int oops = strlen(dangling);    
{{< /highlight >}} Although it might seem syntetic example, there are some practical use cases like sending the `std::string` content using WinAPI `SendMessage` or `PostMessage`. While `SendMessage` is synchronous and fine, `PostMessage` will result in a dangling pointer being stored and used later.  

I believe we can do better: provide an access to `data()` in a way that significantly reduces the likelihood of misuse and the occurrence of dangling references.

## A ref-qualified member function

It could be reasonable to prevent certain member functions from being called on a temporary object. That's what we could achieve with _ref-qualified member functions_. This feature is not broadly used, so here is a short description: a member function might have a reference qualifier in the same familiar way as the _const qualifier_. The correct overload will be chosen based on the type of `*this` reference. While member functions can also be const-qualified, it’s important to note that constness is an orthogonal property that does not affect the reference qualifier. 

An overload resolution on ref-qualifier is done in the same way as for cv-qualifiers. To better understand how the member function overload works, let’s remember that the member function [_is considered to have an extra first parameter, called the implicit object parameter, which represents the object for which the member function has been called_](https://eel.is/c++draft/over.match.funcs.general#2).

Thus, `Foo foo; foo.f()` is similar to `Foo::f(foo);`[^without-temporaries], where the type of implicit argument is determined by present cv- and ref- qualifiers on the member function declaration. I think the [execution result](https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahUAUgBMAIUtXyKxqiIEh1ZpgDC6egFc2TEAstcncAGQImbAA5PwAjbFIQaQA2cgAHdCViFyYvX39A4Iys5yFwyJi2eMSU%2B2xHUqYRIhZSIjy/AKC6hpzm1qJy6LiEpNSlFraOgu6JgaHK6rGASnt0H1JUTi5LAGYI1F8cAGozXY8XCdJsdjPcMy0AQQfHq58nY5EXswB2GyfjoDjhgmBNgUhWgAqY6CdAQZbAoRgyzJQG/KzHa5EDZMU4WCywkCI0FEPGoswAVg8TEsFjOGN%2BABEXkDiWDUBDSNDYfCyazWejMdhsaRcbTCcd6AA3CQ%2BbB8ynU2n004/ZkAoEg9mc7noOEIrWklEo1UYrE4vEEvVEw1kk2Kmn4lVMlmapGkjlQmF63nGiyo/mmoUisX4iV%2B8lUx103YMtXfDWAubOVBsj0646xVoQW1ff0IoPm0WWrOJNMKqPK2Oq9WPVnJgip22ermZ7N55IFwOCouhiylonS2Xy%2B2Vp3Vl2J44Npvu8Fe0s5ucdlEInvCi20gfliOnMcxuO1%2BstFPllvQxcr/OBwsb4tb1pE3cOquHhN1oFS9AETDA640XlbXtP4XVjV1AS/H9JQAfX/X1/VNUD/g/CDv1/UhYOwAC139YCbHjMCniQ98IlJNgWAiCBSPIecuUhZZvhAqcWB8EhjmAYURFOXZGT3GwKV4gBaO5PhvW8Q0%2BeFnQI5DWRYtiOPaLizl4yl%2BKEkTc3Ei0RCkicZPfVkuKUbjeMU3SGMIlDyy41ATJU9jOL05DwOnIhMBANB1iNc4zg8RzRHhAA6Hku2OAB6cLvXQJ9cIQ04pzEtFfPOAKPAskKfS7SLoptOdn0SpK/L86dMv1JKcolIdfBHBDXKS7iPBKuyyt5G9KutHc6sKsTitS2lBMGobhuE/d6qKlL/JETzF3M%2BECxy7cCushq%2Bqmmbs0U9L5oiqLtyAuLyR6m81s%2BDbSAgJQwqBRbH0lGUaoVY7u0ms6QEXOzrt2tsywOo6VomprUomDyQHcegVSBQygSUILUDgyyMTEnL0AAa1cubljhhHIcBFHUZosAuFzE0HBoUyAos%2BkzGSLRXNc2H6Ew7DcfaqK0dcnLMaCpmcerb6ElIMgiSJjtp3qcmHMxonjmE3ZcGOUWZZYEyiaIJACCUZXSGAPxmFJQglHEUhMBMgBHHwJAIOgEnsikrEEmg0gSNhNayKVsEpWtOai2GML5pH%2BRywXhcVrgr1RMmKaUekZeuC2CGuEyWGBFWjSnbn/aw5zGNrLhVnobgKX4AIuB0cgYq4JrrFsad1k2Wrdj4cgiG0fPVlRkAKWCQuuGkEu24r7h%2BCUEBglbsv8/IOBYBQDA2DSBgEkoah58XxhEmAJRWDSJQkHQIhyBwKVG2wAA1AhsAAdwAeWd0vm7oegiFt6hYkH2IIlaABPbhm8/1gpBv431iLobATg/78HnhwYQN8mD0F/pPI%2B2ByImEkEgwg1wnAEA9qPJB2B1DgNYtscupF6iD3oAQWIpAf5eBwIPIgpACBsEgasGgRgt4X2vnfZgkC5DCDEBITgMh%2BGKBUBoQe%2BgeCGGMKYWwthDBUNHpAVY6A0iNDwYJEGKlzA12sBYXY/B0Ae1IEwnAyj4Q9HAY0NwTBPDeE6AYMIERhhVFGNI4o2QhDTACB4zIXimALBGIkaRDhrF9EmO0BxBRQn1HCUIfobQgluJCfYSJPiDBzCSS4xY7jVhKHrlsKQBci4DyQZXY46gAAcyRBLJGkOxVAqYICMJ8EwVGCIID4GIGQPETdlj8AnjoZYHcu4924P3cgpdy6VxHmPFubcRmGG4BYfgLDki7CCgATgMVoH4yQdm7D2Qc3YUzB6zIWZPVYM9kAgG8mkViK8lwLyXqQKI7BtjVNqfUxpzTWntIGSEQgJAzEGEEAI8QkgRHgrEWoTQSCpHkCvjQtIrDllcGLmc8p3Ab6sQeaSdA5Mvl1IacAJpxwWmkDaR0ilXgXkbz6TwQFQz27kCQDcHAiRLG90mSw3YyQgrHMOUK3ZPwsUzOHvYeZLKlm90MeKoxkqZWrBMVkVw0ggA) of the sample below will illustrate this much better than I can:
{{< highlight cpp>}}#include <iostream>

struct S
{
    const char* foo() const &  { return "foo: const & \n"; }
    const char* foo() &        { return "foo: lvalue & \n"; }
    const char* foo() const && { return "foo: const && \n"; }
    const char* foo() &&       { return "foo: && \n"; }

    static const char* bar(const S&)  { return "bar: const & \n"; }
    static const char* bar(S&)        { return "bar: lvalue & \n"; }
    static const char* bar(const S&&) { return "bar: const && \n"; }
    static const char* bar(S&&)       { return "bar: && \n"; }

    void cref() const & {};
    void l_ref() & {};
    void r_ref() && {};
};

int main(int, char**)
{
    auto getS = [] -> S        { return S(); };
    auto getCS = [] -> const S { return S(); };

    S s = getS();
    const S cs = getS();

    std::cout << getS().foo()    // foo: &&  
              << getCS().foo()   // foo: const && 
              << s.foo()         // foo: lvalue & 
              << cs.foo()        // foo: const & 
              << "-----------\n"
              << S::bar(getS())  // bar: && 
              << S::bar(getCS()) // bar: const && 
              << S::bar(s)       // bar: lvalue & 
              << S::bar(cs)      // bar: const & 
              << std::endl;    

    s.cref();         // ok
    getS().cref();    // ok, 'const S& self = getS();`
    
    s.l_ref();          // ok
    // getS().l_ref();  // error: 'S& self = getS()' -> 'S' as 'this' argument discards qualifiers [-fpermissive]

    // s.r_ref();       // error: 'S&& self = s;' requires a cast 
    getS().r_ref();
}
{{< /highlight >}}

It's worth highlighting that it's rare to encounter a `const &&` reference in the wild, so typically, rvalue-overloads should be non-const.

[^without-temporaries]: but without the ability to use an implocit cast or to introduce a temporary, like `auto s = "char* " + std::string("string")`. 

### Disable a member function call for rvalues

Having read the cppreference, let's delete undesired overloads from the `MyRawImage` with almost no effort:
{{< highlight cpp "hl_lines=11-15" >}}
class MyRawImage
{
    std::vector<Pixel> m_buffer;
    Metadata           m_metadata;
public:
    MyRawImage(std::vector<Pixel> src) : m_buffer(std::move(src)) {}

    const Pixel& operator[](int index) const { return m_buffer[index]; }
    Pixel&       operator[](int index)       { return m_buffer[index]; }

    const std::vector<Pixel>& data() const & { return m_buffer; }
    const std::vector<Pixel>& data() && = delete;

    const Metadata& information() const &    { return m_metadata; }
    const Metadata& information() && = delete;
};{{< /highlight >}}
Well, now we can't accidentally access the property of the temporary via reference because the rvalue-this overloading is explicitly deleted. 
{{< highlight cpp>}}
<source>: In function 'MyRawImage problematic(int)':
<source>:54:36: error: use of deleted function 'const std::vector<Pixel>& MyRawImage::data() &&'
   54 |     for(Pixel p : loadImage(i).data())
      |                   ~~~~~~~~~~~~~~~~~^~
{{< /highlight >}}

Now, let's address the compiler's complaints and examine the result. Currently, the only way to access `MyRawImage::data()` is by declaring a MyRawImage variable or using a const reference to [ extend the temporary lifetime](https://en.cppreference.com/w/cpp/language/lifetime). Therefore, we need to introduce a named variable or a const reference for every call to `data` on the temporary.

{{< highlight cpp>}}
#include <ranges>
#include <cassert>
#include <vector>

struct Pixel
{
    int r = 0;
    int g = 0;
    int b = 0;

    friend auto operator<=>(const Pixel& a, const Pixel& b) = default;
};

struct Metadata { /* some metadata here, width, height, etc. */ }; 

class MyRawImage
{
    std::vector<Pixel> m_buffer;
    Metadata           m_metadata;
public:
    MyRawImage(std::vector<Pixel> src) : m_buffer(std::move(src)) {}

    const Pixel& operator[](int index) const { return m_buffer[index]; }
    Pixel&       operator[](int index)       { return m_buffer[index]; }

    const std::vector<Pixel>& data() const & { return m_buffer; }
    const Metadata& information() const &    { return m_metadata; }
    const std::vector<Pixel>& data() && = delete;
    const Metadata& information() && = delete;
};

MyRawImage loadImage(int i)
{
    return std::vector<Pixel>(i * 100, Pixel {i, i, i});
}

MyRawImage was_problematic(int i)
{
    std::vector<Pixel> filtered;
    auto filter = [](Pixel p) 
    { 
        p.r = std::min(p.r, 0xFF); 
        p.g = std::min(p.g, 0xFF); 
        p.b = std::min(p.b, 0xFF);
        return p; 
    };

    // const lvalue reference extends temporary object lifetime
    const MyRawImage& image = loadImage(i); 
    for(Pixel p : image.data())
        filtered.push_back(filter(p));

    return filtered;
}

Pixel was_fine(int i)
{
    auto max = [](auto&& range) -> Pixel 
    { 
        return *std::max_element(std::begin(range), std::end(range)); 
    };

    // this one was fine: a temporary will be destroyed after the max() calcualtion
    // return max(loadImage(i).data()); 
    MyRawImage image = loadImage(i); 
    return max(image.data());         // <-- would be nice to avoid creating a named variable here
}

int main(int, char**)
{
    constexpr static int pattern = 0x12;
    constexpr static Pixel pixelPattern = Pixel { pattern, pattern, pattern };

    Pixel maxPixel = was_fine(pattern);
    assert(maxPixel == pixelPattern);

    MyRawImage img = was_problematic(pattern);
    auto isGood = [](const Pixel& p) { return p == pixelPattern; };
    assert(img.data().end() == std::ranges::find_if_not(img.data(), isGood));
}
{{< /highlight >}}
Just a few comments, as usual:
 - `was_problematic` method is now good, since there is no temporary access. It utilizes _const lvalue reference lifetime extension_: the return value is bound to a `const MyRawImage& image`, thus extending its lifetime to match that of the reference. So the `data()` call is safe;
 - on the other hand, the `was_fine` method had to introduce an unnecessary lvalue variable for the temporary. While it may not seem like a significant problem, it violates the principle of keeping the scope of variables as small as possible. Ideally, it would have been preferable to maintain the temporary's lifetime within a single full-expression, but unfortunately, it didn't work out that way.

 ### Explicitly enable the member function call on rvalues

The problem is that compiler can't detect whether the reference to a property of the temporary will outlive the parent object. Technically there are two choices: allow the call on rvalue always or never.

But what if the developer is certain that an rvalue could be used in the specific expression? Well, we can give the ability to state this intention and forcibly allow an rvalue overload call with a little trick.

Let's go back for a second to `std::range`. It does not prohibit creating a view from a temporary but performs a view type selection instead:
 - for lvalue, a non-owning `std::ranges::ref_view` is used;
 - for rvalue, an owning `std::ranges::owning_view` is used to take the ownership of the temporary and convert it to an lvalue;
 - when returning a possible-dangling iterator to a content of the temporary, return a `std::ranges::dangling` to indicate this. 

We could adopt the last approach to our needs: the rvalue overload of `data()` could return a special wrapper that should not be implicitly-convertible to an underlying reference type. This approach will require the programmer's attention and make sure that there is no dangerous access made by oversight. 

Here we go:
{{< highlight cpp "hl_lines=47 26" >}}
class MyRawImage
{
    std::vector<Pixel> m_buffer;
    Metadata           m_metadata;
public:
    class UnsafeReference 
    {  
        std::vector<Pixel>& m_buffer;

    public:
        UnsafeReference(std::vector<Pixel>& buffer) : m_buffer(buffer) {}

        // I would like it to be a free-function rather than a member function,
        // to lower the chance that Intellisence will provide a disservice
        // to the developer by slipping an unsafe getter by auto-suggestions. 
        // it's good to require a fair bit of attention here
        friend std::vector<Pixel>& allowUnsafe(UnsafeReference&&);
    };

    MyRawImage(std::vector<Pixel> src) : m_buffer(std::move(src)) {}

    const Pixel& operator[](int index) const { return m_buffer[index]; }
    Pixel&       operator[](int index)       { return m_buffer[index]; }

    const std::vector<Pixel>& data() const & { return m_buffer; }
    UnsafeReference data() &&                { return m_buffer; }

    const Metadata& information() const &    { return m_metadata; }
    const Metadata& information() && = delete;
};

std::vector<Pixel>& allowUnsafe(MyRawImage::UnsafeReference&& unsafe)
{
    return unsafe.m_buffer;
}

// ...

Pixel fine_again(int i)
{
    auto max = [](auto&& range) -> Pixel 
    { 
        return *std::max_element(std::begin(range), std::end(range)); 
    };

    // this one was fine: a temporary will be destroyed after the max() calcualtion
    return max(allowUnsafe(loadImage(i).data()));
}
{{< /highlight >}}
Now a similar approach could be implemented for the Metadata, but for now I hope the idea is clear[^honestly-sold] to the reader, so further experiments are up to you.

[^honestly-sold]: Honestly, I hope the idea is _sold_

### The result I could live with

The complete code below:
{{< highlight cpp>}}
#include <ranges>
#include <cassert>
#include <vector>

struct Pixel
{
    int r = 0;
    int g = 0;
    int b = 0;

    friend auto operator<=>(const Pixel& a, const Pixel& b) = default;
};

struct Metadata { /* some metadata here, width, height, etc. */ }; 

class MyRawImage
{
    std::vector<Pixel> m_buffer;
    Metadata           m_metadata;
public:
    class UnsafeReference 
    {  
        std::vector<Pixel>& m_buffer;

    public:
        UnsafeReference(std::vector<Pixel>& buffer) : m_buffer(buffer) {}

        // I would like it to be a free-function rather than a member function,
        // to lower the chance that Intellisence will provide a disservice
        // to the developer by slipping an unsafe getter by auto-suggestions. 
        // it's good to require a fair bit of attention here
        friend std::vector<Pixel>& allowUnsafe(UnsafeReference&&);
    };

    MyRawImage(std::vector<Pixel> src) : m_buffer(std::move(src)) {}

    const Pixel& operator[](int index) const { return m_buffer[index]; }
    Pixel&       operator[](int index)       { return m_buffer[index]; }

    const std::vector<Pixel>& data() const & { return m_buffer; }
    UnsafeReference data() &&                { return m_buffer; }

    const Metadata& information() const &    { return m_metadata; }
    const Metadata& information() && = delete;
};

std::vector<Pixel>& allowUnsafe(MyRawImage::UnsafeReference&& unsafe)
{
    return unsafe.m_buffer;
}

MyRawImage loadImage(int i)
{
    return std::vector<Pixel>(i * 100, Pixel {i, i, i});
}

MyRawImage was_problematic(int i)
{
    std::vector<Pixel> filtered;
    auto filter = [](Pixel p) 
    { 
        p.r = std::min(p.r, 0xFF); 
        p.g = std::min(p.g, 0xFF); 
        p.b = std::min(p.b, 0xFF);
        return p; 
    };

    const MyRawImage& image = loadImage(i);  // const lvalue reference extends temporary object lifetime 
    for(Pixel p : image.data())
        filtered.push_back(filter(p));

    return filtered;
}

Pixel fine_again(int i)
{
    auto max = [](auto&& range) -> Pixel 
    { 
        return *std::max_element(std::begin(range), std::end(range)); 
    };

    // this one is fine: a temporary will be destroyed after the max() calcualtion
    return max(allowUnsafe(loadImage(i).data()));
}

int main(int, char**)
{
    constexpr static int pattern = 0x12;
    constexpr static Pixel pixelPattern = Pixel { pattern, pattern, pattern };

    Pixel maxPixel = fine_again(pattern);
    assert(maxPixel == pixelPattern);

    MyRawImage img = was_problematic(pattern);
    auto isGood = [](const Pixel& p) { return p == pixelPattern; };
    assert(img.data().end() == std::ranges::find_if_not(img.data(), isGood));
}
{{< /highlight >}}
Compile, run, enjoy. 