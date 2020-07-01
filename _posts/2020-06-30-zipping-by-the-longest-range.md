---
title: Zipping By The Longest Range
author: Marin Peko
date: 2020-06-30 08:00:00 +0200
categories: [C++17, Ranges, Algorithm]
tags: [c++17, ranges, algorithm]
---

Long time now, ranges have become a large part of every C++ codebase and, thus, a part of C++ programmer's everyday life. And indeed, C++ offers many ways to operate on the elements of a range.

But sometimes software requirements go beyond the power of STL...

In this article, I will try to explain several ways on how to zip multiple ranges together as well as the downsides of those approaches, and what we can do to overcome those downsides...

## Standard way of zipping ranges

Let's say we have two `std::vector`s like:

```c++
std::vector<int> a{ 1, 2, 3 };
std::vector<float> b{ 4.1, 5.2, 6.3 };
```

and we want to have a third range with the following content: `{ (1, 4.1), (2, 5.2), (3, 6.3) }`. The standard and straightforward way to achieve this would be to use `std::transform` like:

```c++
std::vector<std::pair<int, float>> c;

std::transform(
    a.begin(), a.end(), b.begin(),
    std::back_inserter(c),
    [](int const x, float const y) {
        return std::make_pair(x, y);
    }
);
```

But this approach has its downsides...

First of all, by using C++11, we are limited to zipping only two ranges together. C++17 has improved that a bit and we can then zip three ranges. However, there might be some cases where we want to zip as many ranges as possible.

Furthermore, we need to create a new `std::vector` instance which unfortunately results in memory allocation and, hence, drop in performance. Consequently, we cannot use `std::transform` as a part of a range-based for loop like:

```c++
for (auto&& i : std::transform(...)) {
    std::cout << i.first << " " << i.second << std::endl;
}
```

Instead, we need to have an additional variable which IMO introduces a bit of noise to our code.

At the end, if we ever pass two (or, in case of C++17, maximum of three) ranges of different lengths, we would run into an **U**ndefined **B**ehavior as we would iterate off the end of one of the ranges.

Now that we have seen some drawbacks of `std::transform`, let's see other options we have...

## Using *Boost* library

Boost library has solved some of the issues we have addressed in the previous section. Its `boost::combine` function accepts variadic number of ranges to be zipped and returns `combined_range` (which is an `iterator_range` of a `zip_iterator`).

```c++
#include <boost/range/combine.hpp>

std::list<int> a{ 1, 2, 3 };
std::vector<float> b{ 4.1, 5.2, 6.3 };
std::array<std::string, 3> c{ "a", "b", "c" };

for (auto&& t : boost::combine(a, b, c)) {
    std::cout << t.get<0>() << " " << t.get<1>() << " " << t.get<2>() << std::endl;
}
```

Since `boost::combine` uses `iterator_range` as a return value, there is no more memory allocation and performance drop associated to it. Also, we got rid of that boring temporary variable.

So what's the problem with the Boost library then?

General problem with the Boost library is its massiveness and the fact that you may only need `boost::combine` function but what you get is the whole library with lots of additional features that you don't actually use. Also, using a 3rd party library is, by itself, not the best thing in the world. It's a dependency which can, especially in case of Boost, increase compilation time, complicate your deployment etc. Apart from this, there is another problem that's present with `std::transform` solution too... and that's the famous **U**ndefined **B**ehavior, i.e. if your input ranges are of the different lengths, then your program might crash or iterate beyond the end.

So... *Boost* is not perfect either...

Let's see some more...

## C++20 standard and *ranges-v3* library

C++20 standard has brought up lots of awesome features and one of them is *ranges* library. However, despite being proposed in [P1035R4](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1035r4.html), `zip_view` adaptor didn't make its way through C++20 <span>&#128546;</span> (at least as far as I know so please correct me if I am wrong). Therefore, we will be talking only about Eric Niebler's *ranges-v3* library here.

Now, let's start by explaining what actually a `view` in *ranges-v3* library is... By definition, *a view is a lightweight wrapper that presents a view of an underlying sequence of elements in some custom way without mutating or copying it*. Thus, views are cheap to create and copy, and have non-owning reference semantics.

Following example shows how we can use a `zip` function which, given *N* ranges, returns a new range where the *i<sup>th</sup>* element is a tuple of *i<sup>th</sup>* elements of all *N* ranges:

```c++
#include <range/v3/all.hpp>

std::list<int> a{ 1, 2, 3 };
std::vector<float> b{ 4.1, 5.2, 6.3 };
std::array<std::string, 3> c{ "a", "b", "c" };

for (auto&& [i, j, k] : ranges::v3::view::zip(a, b, c)) {
    std::cout << i << " " << j << " " << k << << std::endl;
}
```

As stated in proposal [P1035R4](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1035r4.html):

> The typical zip operation transforms several input ranges into a single input range containing a tuple with the ith element from each range, and terminates **when the smallest finite range is delimited**.

Therefore, unlike `std::transform` and *Boost* library, `ranges::v3::view::zip` function does not lead us into an **U**ndefined **B**ehavior if input ranges are of different lengths.

So, basically *ranges-v3* library solves pretty much all the problems we had with previous two approaches. But... there might be something missing here... What about zipping by the longest input range?

For example, Python's *itertools* module provides such functionality with `zip_longest` function which makes an iterator that aggregates elements from each of the ranges. If the ranges are of uneven length, missing values are filled-in with the value specified on a function call and iteration continues until the longest range is exhausted. Pretty cool <span>&#128526;</span>, right?

Let's try and make something similar in C++...

## Zip by the longest one!

First, we need to set the requirements for our `zip_longest` function:

1. zip as many ranges as possible
2. iterate up to the length of the longest range
3. if one of the ranges runs out early, set its values to default ones

So, the usage at the end would look something like:

```cpp
float a[]{ 1.2, 2.3, 3.4, 4.5 };
std::vector<int> b{ 1, 2, 3, 4, 5 };
std::array<double, 3> c{ 1.1, 2.2, 3.3 };
std::vector<std::string> d{ "a", "b", "c" };

for (auto&& [i, j, k, l] : zip_longest(a, b, c, d)) {
    std::cout << i << " - " << j << " - " << k << " - " << l << std::endl;
}
```

and the result output:

```
1.2 - 1 - 1.1 - a
2.3 - 2 - 2.2 - b
3.4 - 3 - 3.3 - c
4.5 - 4 - 0 -
0 - 5 - 0 -
```

Let's start with the implementation details...

`zip_longest` function is a some sort of a helper function which accepts parameter pack (thus, it supports as many ranges as possible), forwards it to the `zip_longest_range` constructor and returns the constructed instance:

```cpp
template <typename... Range>
[[nodiscard]] constexpr zip_longest_range<Range...> zip_longest(Range&&... ranges) {
    return zip_longest_range<Range...>(
        std::forward<Range>(ranges)...
    );
}
```

Now, `zip_longest_range` class contains functions that are common to any *range*-like class (e.g. `std::vector` or any other container). It basically does not hold any logic related to zipping but only provides begin and end zip-iterators as well as other helper functions like `size` and `empty`. Furthermore, if you take a closer look at the `begin` public function, you may notice that both begin and end iterators of all the ranges are being passed to the zip-iterator. You may ask yourself why is that so... But the reason is quite simple: in order to fill the missing values (in case of ranges of uneven length) with the default ones, we need to know if we have reached the end of a particular range.

```cpp
template <typename... Range>
class zip_longest_range {
    using indices = std::index_sequence_for<Range...>;

public:
    explicit constexpr zip_longest_range(Range&&... ranges)
        : ranges_{ std::forward<Range>(ranges)... },
          size_{ max_size(indices()) }
    {}

    [[nodiscard]] constexpr auto begin() const noexcept {
        return make_zip_iterator(begin_tuple(indices()), end_tuple(indices()));
    }

    [[nodiscard]] constexpr auto end() const noexcept {
        return make_zip_iterator(end_tuple(indices()));
    }

    [[nodiscard]] constexpr std::size_t size() const noexcept {
        return size_;
    }

    [[nodiscard]] constexpr bool empty() const noexcept {
        return 0 == size();
    }

private:
    template <std::size_t... Is>
    [[nodiscard]] auto begin_tuple(std::index_sequence<Is...>) const noexcept {
        return std::make_tuple(std::begin(std::get<Is>(ranges_))...);
    }

    template <std::size_t... Is>
    [[nodiscard]] auto end_tuple(std::index_sequence<Is...>) const noexcept {
        return std::make_tuple(std::end(std::get<Is>(ranges_))...);
    }

    template <std::size_t... Is>
    [[nodiscard]] constexpr std::size_t max_size(std::index_sequence<Is...>) const noexcept {
        if constexpr (0 == sizeof...(Range)) {
            return 0;
        } else {
            return std::max({ size(std::get<Is>(ranges_))... });
        }
    }

    template <typename T>
    [[nodiscard]] constexpr std::size_t size(T&& range) const noexcept {
        return std::distance(std::begin(range), std::end(range));
    }

private:
    std::tuple<Range...> ranges_;
    std::size_t size_;
};
```

As already mentioned, `begin` and `end` public functions create zip-iterators by using `make_zip_iterator` helper function with 2 overloads, first one used for creating the end iterator and second one used for creating the begin iterator:

```cpp
template <typename... Ts>
zip_iterator<Ts...> make_zip_iterator(std::tuple<Ts...>&& end_tuple) {
    return zip_iterator<Ts...>{ std::forward<std::tuple<Ts...>>(end_tuple) };
}

template <typename... Ts>
zip_iterator<Ts...> make_zip_iterator(std::tuple<Ts...>&& begin_tuple,
                                      std::tuple<Ts...>&& end_tuple) {
    return zip_iterator<Ts...>{
        std::forward<std::tuple<Ts...>>(begin_tuple),
        std::forward<std::tuple<Ts...>>(end_tuple)
    };
}
```

Finally, let's get to the implementation of the `zip_iterator` class which contains all the logic. Actually, to be precise, overloaded increment operator contains all the logic as it increments iterators to all the ranges *(1)* and constructs the value tuple *(2)*. While constructing the value tuple, default value is being used if the incremented iterator is the end iterator of a particular range. Otherwise, the real value is being used.

```cpp
constexpr zip_iterator& operator++() noexcept {
    next(indices());                  // (1)
    value_tuple_ = make(indices());   // (2)
    return *this;
}
```

`indices` is an alias for `std::index_sequence_for<Ts...>` and it is being used by both the `next` function and the `make` function. `next` function takes advantage of fold expressions in order to carefully increment the iterators of all the ranges. Why did I say carefully? Because, we must not pass the end of a particular range because we would run into an **U**ndefined **B**ehavior in that case.

```cpp
template <std::size_t... Is>
constexpr void next(std::index_sequence<Is...>) noexcept {
    ((
        std::get<Is>(iterator_tuple_) == std::get<Is>(end_iterator_tuple_) ?
        std::get<Is>(iterator_tuple_) :
        ++std::get<Is>(iterator_tuple_)
    ),...);
}
```

Furthermore, `make` function creates a tuple out of the values current iterators are pointing to. However, if the current iterator is the end iterator, then the value of the specific type is defaultly initialized.

```cpp
template <std::size_t... Is>
[[nodiscard]] constexpr auto make(std::index_sequence<Is...>) noexcept {
    return std::make_tuple(
        (
            std::get<Is>(iterator_tuple_) == std::get<Is>(end_iterator_tuple_) ?
            typename std::iterator_traits<decltype(std::prev(std::get<Is>(iterator_tuple_)))>::value_type{} :
            *std::get<Is>(iterator_tuple_)
        )...
    );
}
```

Take a look at the full implementation of `zip_iterator` class below:

```cpp
template <typename... Ts>
class zip_iterator {
    using indices = std::index_sequence_for<Ts...>;

public:
    using iterator_category = std::forward_iterator_tag;
    using difference_type   = std::ptrdiff_t;
    using iterator_type     = std::tuple<Ts...>;
    using value_type        = std::tuple<typename std::iterator_traits<Ts>::value_type...>;
    using pointer           = value_type*;
    using reference         = value_type&;

    constexpr zip_iterator(std::tuple<Ts...>&& begin_iterator_tuple,
                           std::tuple<Ts...>&& end_iterator_tuple)
        : iterator_tuple_{ std::forward<std::tuple<Ts...>>(begin_iterator_tuple) },
          end_iterator_tuple_{ std::forward<std::tuple<Ts...>>(end_iterator_tuple) },
          value_tuple_{ make(indices()) }
    {}

    constexpr zip_iterator(std::tuple<Ts...>&& end_iterator_tuple)
        : iterator_tuple_{ std::forward<std::tuple<Ts...>>(end_iterator_tuple) }
    {}

    [[nodiscard]] constexpr auto operator*() noexcept {
        return value_tuple_;
    }

    [[nodiscard]] constexpr auto operator->() noexcept {
        return &value_tuple_;
    }

    constexpr zip_iterator& operator++() noexcept {
        next(indices());
        value_tuple_ = make(indices());
        return *this;
    }

    constexpr zip_iterator operator++(int) noexcept {
        auto tmp = *this;
        ++(*this);
        return tmp;
    }

    [[nodiscard]] constexpr bool operator!=(zip_iterator const& rhs) const noexcept {
        return iterator_tuple_ != rhs.iterator_tuple_;
    }

    [[nodiscard]] constexpr bool operator==(zip_iterator const& rhs) const noexcept {
        return iterator_tuple_ == rhs.iterator_tuple_;
    }

private:
    template <std::size_t... Is>
    [[nodiscard]] constexpr auto make(std::index_sequence<Is...>) noexcept {
        return std::make_tuple(
            (
                std::get<Is>(iterator_tuple_) == std::get<Is>(end_iterator_tuple_) ?
                typename std::iterator_traits<decltype(std::prev(std::get<Is>(iterator_tuple_)))>::value_type{} :
                *std::get<Is>(iterator_tuple_)
            )...
        );
    }

    template <std::size_t... Is>
    constexpr void next(std::index_sequence<Is...>) noexcept {
        ((
            std::get<Is>(iterator_tuple_) == std::get<Is>(end_iterator_tuple_) ?
            std::get<Is>(iterator_tuple_) :
            ++std::get<Is>(iterator_tuple_)
        ),...);
    }

private:
    iterator_type iterator_tuple_;
    iterator_type end_iterator_tuple_;
    value_type value_tuple_;
};
```

If you wish to experiment a bit, play with the code on [godbolt] [1].

## Conclusion

Throughout this article, we have discussed about several approaches on how to zip multiple ranges together as well as the benefits and drawbacks of using those approaches. At the end, an example of how to zip the longest range, a feature which is missing from all the other approaches we have mentioned before, has been shown. Let me know your thoughts and suggestions in the comment section below... <span>⬇️</span><span>👇</span>

[1]: https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxLgDYADKQAOqBYXW0eoYmgj5%2BanRWNvZGTi6cHorKmKoBDATMxARBxqacSSoRtOmZBFF2js5ungoZWTkh%2BbWl5TFx1QCUiqgGxMgcAORSAMzWyIZYANTiwzoEBl7sM9ji7gCCI2MTmNOzAG4pRMTLqxuSo7TjBlMzOgZqrIQAnifrm5fbuzrqtcSYzEZXmcLlcbrNCM5mEcge9QTtbmxgCRCAhAcMVm91gRMEZFlD4bMCE8vFoAZgAHSUyYAFQUQPGzAUCkmAC88F4APoQ4hQkjTADssnWkxFkwMflEk2s%2BH6zJmABFJrV0CAQNLMAAPDlKACOBi0/Q5NGOs1plPJJ2GQo26y8Bgcj2QIFOorFEuAUuxPKOHJY2KRxCeu0VytVxoA7pl0FyvbziByMsAZtbXeLrB78FQqM4DZgE8SdiKFUqCCqQF4CMRM1QE8mXaK05LuXH8yTXcGS2X5otMLczZTLSmG%2B7Jns2PrW4X2x3QyBu0tCQXaGTO6rmz7K8xCHTTTvsKqx4Y80SSebB/WRY2PT5rF7p%2B3i4eJyfMAAqOvC4fpyZ/bN/S5TtOj7jseBZSK4H42msrpoLQtSal4xCsuyMaQkcECzvOva7me6LgeBkxOMA1iod6JAJgs7CkKuc6UdhOj9haeGSBBLGTOYpEtlhHQXu2ICemh5FYRy4iCjREZRrcmF0X2Ci4Ss6IQERJHrkJdEdAK8qkLx04cap8bCaJ0jiSQkZVlJpaqlhsnycsEB6bGG7qZp2mfveo4gRRPYiWJRjMAA1pgEDSngsoQB0GmifKvFGVFpy8bB8EaohyGcvpGGWbRPY2QOzGsa47HaJxTk9jxbl8QJZEGXRPnGbOEnmbM0nZThuUKdg9lFfpXnsJF/LReVsX9fFg0AKyyONtAYHgCgsOZo3RQtkyJdiyVIcw9yoJMqAklVr7hZMU2av0FYCkO05/PMxC0B5R49XmkGunFmLQaK4jje90hTfgs2SQt72KitCHrZt227XGAC0dkaUdGonQQZ06aKl29Dd4FPseNWPW9w0vTBdBJSlbJpY5JAETtgnHDIcgHbD8OI%2BVro2BqBDBdooWYAo4U8VaSMihj90ch2fmBWzMqc9z2MXZgV03a%2BBAIDNUuaSNr0ikDa2pcVfIU1Vcg07eMOoMdmCnbFjOihtRCTAQuIdvLis7udD7UzIEAOzNPPO66KPXTbuLK89UFPR9k3Tb982LYDBOrSlDioPoYOU2AYAKhAxPa0hK0EcQCAKBpK2HcbcOmwj5tq%2B2vs3d1wmTCnxa53JNdY7zg248Hb2h194dzegAMA8tMfA4RCesEnevDNFk/pyh%2BmD3BBA53nBcx0XJtm4KfM/jLqOVVxNXBg3efks33mB%2B3pyIXgY7Ys65XYriwjYl8s5%2BCyx7mpMACSe4xV330zV7v3JaGsUpWy2iLIKs51Ral1PqACtwf7ySNuvMum8LYiirjRSB90IBb1dHgjB7kaLABlogvcwVSbVW8pFSexZZykMXrMH%2BdkHKU0FrQgAYvg%2B8L5lxGB2NAqhCYeTbluFgcYL4MplkQpgPY0jVSMPIXZU%2B7AOQRR5vuEAAsXxDUVHfCuxDRSvgYWQ5hFDVF5jKoY%2B8HQzxEK9jFC%2B98cR4mfhZMsb8P5UhYXhcqoCkJ7FQHgdAh1NSs2gdoTU2pMB6lzOQ5Ba8S4b29qKKAPCRSmKYToXxHVLHqMPpPEhZickULYVVDhuxuFENdFk5Ril8kaQMUYosrtpB1PMSo4RwlrHuS6OaRxbcBpQSvjfDgvEa4Fj3iVNRytJltnKfvM%2BrcK46KmTolu1o4orNOA/NxBI5hLjJJ/WkQIM76RykxbAkwcHnKoQorKC4GJyTavhNiHFuIMwrlgu5lNLknDEvVUykkmqZWsq1K5rCiqfO2Vs5xaw9lPwOXw45VJTl%2BLWL8iezz5I3ICnmLFcYHngpxa8liBFlK0HutRZqTzGInHJe86Fzly4%2Bx3n7Ql0IIUAtSZkzKDU%2B6gq7DJbleFFKUvul0LeQLiBmUFToWl9F6Vio6h89STip5wuGbs1xSKvgooEZ/AASiIUh9JhBMi1qwOgpDagch5KIeE6CK5XilOzWUM5MowJiXEgCRoyazBNY62yOzbT2kdM00UCFHSEHnoTJCGdrWOrtQ60hEAg1msZa4T%2BqbOa9PvPxXNChaomVlSCnQGbsIdSLXYqkUVXI2NqXgd%2BJa/KwObUFEKYUIoqzbrIeFIcJrdx%2BkA/6ICh6a3AYRTAxFaAHULnTUuXz3JYNubPe5EqsJiw5lzCK1E1U9m3d2iK59tVjSHQAiOfcx3RwXsPKd5h52r0XSkreq78Uck5SQTq0Yt1dolho09qtB2fUvaOqOcbY5IVfh2hMSoO1PoXkk%2BmrL7xYK8SJFZT0B2dwvT3P6EGAkj0Tq4okiHajIaXah6WstJjuEKSGBDgyK5B0vsQa%2B%2BJI0ikRfiF%2BmUMMEE/rkv%2BeGR0EYHlOzddEHnergfE8xiSF3FxQ86ld7KbqzhwVu2clKHlKM6YpIt6ja3kmY9hs9FcePuKFaqATQnf7ntA/hyOEnQYHvYDJqJsDYnwP6AktqK8kMvrQby7etHNMfu05lR9HTSl2SMxFAZQG8aiiswcmDLbBM%2BIcyx/%2Bznr2EYnSlDLx48Xtvfp5rA3nfV%2BYUwFyDlHX1ELwFQSDw8ID0YVPQjtqAqDmnTaazAPbqPEKwe4ZW5n2KsCUMulp6HMptrwYChDsXckQASyZzSZn7xB3M8B1LureO3ANTsakQIQNhzEy58dd7NYlbg14iAZ2s0/kG4FijwXZs0d3rOH6GQAIPN07mroNFH3A%2B20WeFozOO8UVbcStuKjPK3uwjDDkFYWqzS/qo5hqqSVphHlq7BWB5EcTTazmBB7WDfh4N3FZPk2s3x1mnNg385fbC7ventrKe5pp8GtqhDG2lrlXzs1hnWcmd4sx1j6xbx4usAdEbVBrVQkmMwT6/cxKcHJJIaikhyTDGosMckvBJjcHJKNFWWHRSzgOKoAN3x6DLEIkZSY%2BRJi68mIbs31FLcY/KrOTIPIXizHQD0B0mAjfO4kFr8k7v9ee%2BN8MK3ztbeHAd6/Ss6Znd9zElISQ6vJCe/zw4fP1F88SCLyn/bIpjSTAgOAt5BVPp4Gonwai/lqKsAHvxLnFOG/UQcNRZA1F0DDdU/eWcaB7hfFuFKWfsxphV4hkvyQC%2BdCTD4Ov1fkwV/5%2B3/5bf%2B%2B99V9uHPseZ/F%2BznMN363kOLNsto%2BNnZ/UBhdFYCAAYo0BikFMAMdwP%2BqAn%2BOg%2BsMgSoPQfQ8I5wnAP%2BBAn%2BABe6/kIAo0ngH%2BAw3AP%2BRgXA7gngf%2BABpAQBAwP%2BCgIAngcB/%2Bb%2BpAcAsASAmoKQm0ZAFADexAwACgAACiIMoAwAgKgOGH/jAaQGgLiHgE/AEBwTYKwNwbwXgT/kIV4CIVUKwcuF4AoDwQQIIagMIewMQAAPL3BSF8HwE/50HIBrAsEkFEGkAmHpD4B/4/40D0BMBsAcA8D8AgC67CCiAoCgHyCPAOAkGwD8IcBFp7CG4HAuCVgGC0D%2BR7o7RFAWEQzKgKiV6yAyCcD8jEEQH9CCDKjWDiFcE8GGEDAwHv6f7f6/5GEEGf4agAAcrgEMrg3AkwwAyAyA9ekR0RGk2AGo9BRw9euAhAfIIwnAGkIBbSsBRhe6CA/wWALg4UpASBKBQgn%2BGBpAWBnAOBFRFBVRlhJBZBkxpRAwkgP%2BMhOxExFBe64RfgGg3AQAA%3D%3D%3D