---
title: Obscure Fact About References
author: Marin Peko
date: 2020-03-01 01:00:00 +0200
categories: [C++, Reference]
tags: [c++, const, reference, structured-binding]
---

We have all been told that marking variables `const` is a good practice and that we should all use it whenever we can. This is all true but sometimes `const` is not doing what it should, or at least, what we think it should be doing. Today, I am going to talk about one of these strange cases where marking variables `const` could be a bit misleading…

## Const References Behave As Any Other Const Type...

We would expect that applying `const` keyword to the reference has the same behavior or the same outcome as applying it to any other type of variable, that compilers should help us diagnose when we unintentionally modify it and possibly generate better code (though this is pretty rare). And this is really the case…

Following example:

```c++
int         foo{ 1 };
int const & bar{ foo };

bar = 2;
```

will obviously raise a compiler error saying something like:

> error: cannot assign to variable 'bar' with const-qualified type 'const int &' bar = 2;
> ~~~ ^
> note: variable 'bar' declared const here int const & bar{ foo };

Another obvious example:

```c++
auto p = std::make_pair(1, 2);

auto const & [ foo, bar ] = p;

foo = 3;
```

will cause pretty much the same compiler error as above.

At this stage, you might ask yourselves where’s the catch and what’s this post all about. Well, the next section will give you an answer to that...

## Well, Not Always ...

Let’s say we don’t want to use `std::pair` but instead we want to `std::tie` two variables together like:

```c++
int foo{ 0 };
int bar{ 1 };

auto const t = std::tie( foo, bar );
auto const & [ f, b ] = t;

f = 3;
std::cout << f << " " << b; // output: 3 1
```

What do you think the compiler error message would look like in the above example? We are trying to modify the value of something that should be a const reference, so logically, we expect the compiler to raise pretty much the same error message as with the other two examples above, right? Well… actually… This code compiles successfully, no error messages, no warnings. It simply outputs `3 1`.

What happened here? Is it a compiler bug or ...?

## Pure Standard Behaviour

If we had taken a closer look at how `std::tie` works, we would have noticed that it returns `std::tuple` of references.

```c++
template< class... Types >
constexpr tuple< Types&... > tie( Types&... args ) noexcept;
```

Now, when we do `auto const & [ f, b ] = t;` reference itself will be const-qualified, not the value referenced by it. If we take a peek into the [standard][1] saying:


> Cv-qualified references are ill-formed except when the cv-qualifiers are introduced through the use of a typedef-name or decltype-specifier, in which case the cv-qualifiers are ignored.

we will realize that, in this particular context, the const qualifier is actually being ignored.

KUDOS to those knowing this before reading this article.

To make the standard more clear, here are the examples. First, if we want to make a constant reference like:

```c++
int         i  { 0 };
int & const ref{ i };
```

the compiler will raise an error which is totally fine and logical because we can’t reassign references anyway. However, if we declare an alias for a reference type, then the cv-qualifiers are ignored, per standard:

```c++
int i{ 0 };

using int_ref = int &;
int_ref const & ref = i; // same behavior if we omit ’&’ here
++ref;

std::cout << ref << " " << i; // output: 1 1
```

Why is this so? Why are the cv-qualifiers ignored in this case?

Two words: generic programming. Without cv-collapsing rules generic programming life would be much harder because the following simple example would not compile:

```c++
template< typename T >
void fun( T const & ref )
{
    std::cout << "Ref: " << ref << std::endl;
}

int main()
{
    int   i  { 0 };
    int & ref{ i };

    fun( ref ); // calls fun( int & const & ref )

    return 0;
}
```

but, fortunately, cv-collapsing rules exist and this compiles (check on [Godbolt][2]).

Now, let’s check how to actually deal with those cases when const-qualifier is being ignored.

## How To Deal With These Cases?

There is not much help in recognizing above strange cases other than being familiar with the standard. GCC and CLANG provide us with the -Wignored-qualifiers flag but, in some cases, it’s not of great help either.

In case of the CLANG compiler, the code snippet with specifying alias for the reference type results in a warning saying:

> warning: 'const' qualifier on reference type 'int_ref' (aka 'int &') has no effect [-Wignored-qualifiers]
>    int_ref const & ref = i;
>            ^~~~~~

However, the example with structured bindings, we went through a few moments ago, compiles with no warning at all.

GCC, on the other hand, compiles both examples with no warnings. Check it yourself on [Godbolt][3].

Not much help for us there 😕.

## Conclusion

Throughout this article, we have seen a behavior that, I believe, not many of us have thought about before but it could lead us to some really obscure bugs in everyday life. Except for the fact that we should know the standard, compilers should help us diagnose these kinds of situations, and, effectively, help us put more focus on software logic than on implementation details such as this.

As always, let me know your thoughts in the comments section below... <span>⬇️</span><span>👇</span>

[1]: http://eel.is/c++draft/dcl.meaning#dcl.ref-1.sentence-3

[2]: https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxQQAdUCwutp7DJtx5edFY29kZOLpyKypiq3gwEzMQEvsamUUoqanQJSQQhdo7OggqJyan%2BGWX51oXhxZwAlIqoBsTIHADkUgDM1siGWADU4j066qXEmMxGo9jiAAwAgotLBJhGrsLrozpDBACerlozmEMAKiM988sAbqh46ENUBrQQF0NotKUjkgBsQymVCGjVW4gA7LJlkMYUNSugQChWgQrjpdr9JAAlTBUEAY1HooEEsZwggIkDmVijKErcEAETBy2sKKMzGsEFBywhNNhQ2ZsLwMO5QwWI3p1NWvP5UgBQOFgohDJ6NMlsJeb0BOJB1KGAHpdZ82KwFM9Xu9pf9PnQfjLNcDOStobCpgQ2rQRRKueLlp1mqwQJ0AKydUimToLEOoANomRyOGtdpnXqcEMEAMRxrNADWIG4gYAdH8egsAJwADh63CLZbLnCLvH9nW4IbDEdIUc6IYUIAWpDT4d9pDgsBgiApAA9YgYiGQKBAksAFAAFETKBgIVAAdzDKdIaE2eG23hXNlY663rZD%2B9ch%2BKwE4C0ke9QB/YxAA8tPz9v0yHMJPkCWYhFwDP8AISfAwxDGh6CYNgOB4fhBGEUQUDkOQhDwBxu0gZpUFcbJvgDABaeFRjpCRYxkThwSGYiAHUjTo%2Bj/wIYhmGYhQEGYdAtzo450BENRkGYvBgFoEhMHQYiAEcDDYPAqDwZwFC7BMOhKMlrBPNcNx/Tpd03djXADFM/QDYNQ1/dsA3HMs/mIv5uCGYBkBEh980kIZsAA6cSCGCBcEIfzk0aIYY1kGRU1/TNSBzSRAyEANm1IIwQB6ThPMSy8bM7RQez7GKh1HCAkH/KcZ3ISgF2XVczz0ncrxfG8jzoHT6ovazr1vFwBhEe8Hyibq30/RgGus8rAOA7s8smiDrFA0gYMYFh2C4XgBCfFCxHQmRMOw%2BA8II7wZtIslyMoyLpBo5imIY1j2M47jeM3fipKEvARIYsSJKmaS5IUpSVLUtoNKqSD2u/RrSCM5gTIMwdG0snKOzshynM%2Bbahk4TKFnzThvN8mcAqC4nQvC3bpGigdYoQaYsBcDkkqbEM0oyrKrIHXKuwK/sM2K%2BBSqRV9nCqiBhuKW5kFcVwAH1bk4EtZckMtZfHJylsPdZiG7CAHGshxrCSA5TKaowjC0Ah31oVgTa5rBWVENb7bwKY4luTAZrbSbpy6XdmWUazWCw9jiAOPQsGsti8DShHmmWuC1sQgQom2tCqPkYODtw9tjutANdXhdOrpu3V6IJ3VkC7GI4g0CBzAqdJSHMAowgiAJPEIxuO6CWhW6KSJoiyeJqm7jIa8I3Jkn7%2BpB9KPIx8UaoZ/bpoWlBhDzKDFtrNRjGjAUKWhgVktPLLEn8DJyQMtIcLmp634MrCiK5GpjNmjpnjiiZxsUrZ8F8zZV3otbsvY%2Ba%2BmzCAQM3B8w9D%2BCWBYdZAxwJxhWP4fxmY9B3lzDsb8IHM0kNgtsuDCo02aB7HW3hcxAA%3D%3D

[3]: https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEADZJpLegAyeWpgByxgEaZiISbwAOqBYXVaPUMTc0tffzU6OwdnIzcPL0VlTFVAhgJmYgJg41MLZJUo2gysghinV3dPXgVM7NzQgrqyiriEmoBKRVQDYmQOAHIpAGZ7ZEMsAGpxEZ0CA292WexxAAYAQVHxycwZufU64kxmIxX1rc37AimjZnsITovxAHZZTanPmbeLr7%2Bp64A17SKZrb4AEVm70uG3%2BXwM/lEAPoAH1jlR9uDkTcpGYob84di0ZgMWhaHUZpIzFN0ZigSMQQB6RlTBSnPZuBDMABu6mIAIxAHc9qgjIQZthJOIABxrcQATnluPEkplcsVUwQ7kwBLhcjk6Pxm11/zq6BAKF6OLmsx0NJJ%2Bx0tspkhdjudeHdc1ZBHNIGsrChU2ZUyt3gMBBAU040ZNr0hxo%2BX2BJr%2BgJoqGBoIhRthhMBLiyWZj8dzqa%2BzAjqCmZIp1qxZotakwECmGdIU0L/KeDPLn0rRBrdApuJmAFYQVQOy5x1jZliCGWk3CMfOpiNc4SfX60BGvXbVzbvVJXSf953z43/dpAwzgyywxGoyNY8uc4m859jgtiLRQWWXnBQZulYEBBjHQZSFMQY1kg1AwKdGQ5FZXp%2Bj2UZOEgyMYOA7oAGsQG4McADozBGNZ5WlEZuDI6VpU4MjeFAwZuEg6DYNIeDBkghQQDWUhsNg7o4FgGBEH9AAPVIqzICgICyYAFAABREZQGAQVBBWgzDSDQIxvDwYRihUhxWHUzT2MgvSDPYDxgE4NZLGswz3AAeQjcytLAjjMCk5ANmIRSwMg3zUgyfBoMgmh6CYNgOB4fhBGEUQUH1GQhDwFxeMgbpUG8YpeMGABaM15wkJCZE4F4piKgB1NhWBq2rfIIYhmCahQuXQTSau8TB0BENRkCavBgFoEh%2BqKgBHAw2DwKg8HcBQeNQgZBDNewTLUjSvMGHTBTa7wwMwkCwIgqDvLgsCJOlMwirMbgpmAZBhoc4jXWwPyZKmCBcEIEhKRGThOimRDZBkLDLs6fDPDHIQwNY0gjBAIH3rhyzOOCxQ%2BIEqHSBE8TQuQGTyEoBTlNUsydu0qzRRs5hjMpzyMec2yUGS%2ByHM4XS6Zc4h3MYanLqsPyAqC7iRbC317Cx6LGBYdguF4ARLA51KKvkVhMuyx5OPywJCpK30yrS6QqqahqmpatqOq6nqir6gb6DwYa6tG8bjnQabZq1halpWvo1u5jbaC2qmLOOyCDuYI69uA%2BHwLY4WuJuu6HprDno04Yi1mImNPukoh%2BV%2B/Ai8B4HQdNyGcOh0gtWYLAPF15jEeR1HJHR5Osd4/jBPjgmICQVn3FJiBh48blkG8bwUW5Th5RRSRpRRCSHtIBbWAIJbKBcYWXHsLIAE9I55owjC0AhXNoVhj5w0gsDuURFbv/BjjSblMEKnyvq3k/rmUYWWsXBtWIIfPQWBhatTwMjOO3Q5axUVglAQ3M1blXBprbW8Bcr62HGBRkjY0FyHNoyWqMZGTIB4ikNIGgIDWEaKYbm1g2hVA8NzCIAQ6D0MEOw4ozD4jVGDlQ4opQGj6DyOtIR6R6jlHsJUfhrDFDSK4cHaRfCOjAx6IHeKp1E4XTvindORgFCTymHPeU71pQ/T%2BmXDCHY9D6T5uXEGYM5DVyEt0eujdKA6NbijF4xFO76O7jjPutcCJjm4MREYZh5RrAYmOaJnBOBUTMGYBOIwk5BIlqEnRkhMkcS4m43CpAP7EH8BobgQA
