---
title: Different Ways To Define Binary Flags
author: Marin Peko
date: 2020-11-16 08:00:00 +0200
categories: [C++, Flags]
tags: [c++, flags]
---

Using binary flags in software seems to be a common thing no matter what software domain is nor what programming language is being used in implementation. Of course, this is not a coincidence. There are indeed several hard reasons why is that so...

![Flag]({{ "/assets/img/2020-11-16-different-ways-to-define-binary-flags/flags.png" | relative_url }})

In this article, I will give my best to explain what are the reasons to use binary flags in your software design and what are some ways to define them. At the end, we will see advantages and disadvantages of each of these approaches... So, stay tuned <span>&#128526;</span>!

## Why Use Binary Flags?

Two things come to my mind when I mention binary flags. Those are: **efficiency** and **scalability**.

Even though single binary flag does not tell you more than *yes* or *no*, collection of such flags holds a lot of information in a very small storage space. E.g. single byte can represent 8 flags and, thus, we can make use of 8 states, options or attributes in our software.

Furthermore, using binary flags is ultra-fast because compiler is dealing with two things it is really good with, primitive data types (such as `std::uint8_t`, `std::uint16_t` etc.) and basic bitwise operations. Both of these contribute to pretty good binary flags performance in runtime.

Also, when talking about scalability, new state, option or attribute is easily added without "breaking" the old code.

Now that we have reasoned about why we should use binary flags at all, let's explain different ways on how to actually use them in C++...

## Bare Enumerations

One of the very first associations C++ developers make with binary flags is `enum` (or `enum class` in case of C++11).

So let's imagine we are developing autonomous driving software and we want to have flags representing all the states that car can be associated with (e.g. engine can be turned on/off, lights can be turned on/off etc.). In this case, our enumeration would look like the following:

```c++
enum class CarState : std::uint8_t {
    engine_on = 0b00000001,
    lights_on = 0b00000010,
    wipers_on = 0b00000100,
    // ...
};
```

Once our flags are specified, we just need to write overloaded bitwise operators like:

```c++
CarState operator|(CarState lhs, CarState rhs) {
    return static_cast<CarState>(
        static_cast<std::underlying_type_t<CarState>>(lhs) |
        static_cast<std::underlying_type_t<CarState>>(rhs)
    );
}

CarState operator&(CarState lhs, CarState rhs) {
    return static_cast<CarState>(
        static_cast<std::underlying_type_t<CarState>>(lhs) &
        static_cast<std::underlying_type_t<CarState>>(rhs)
    );
}

// ...
```

and, finally, the usage of binary flags is pretty simple:

```c++
CarState current = CarState::engine_on | CarState::lights_on;

if (current & CarState::engine_on) {
    std::cout << "We are ready to go!" << std::endl;
}

if (current & CarState::wipers_on) {
    std::cout << "Oh, it's raining!" << std::endl;
}

// ...
```

Now that we have seen how easy it is to use bare `enum` to define binary flags, let's see what could go wrong with this approach...

### Pitfalls When Using Bare Enumerations

Even though example mentioned in the previous section is pretty simple, it makes me feel dizzy while looking at all those 0s and 1s in the `enum class` definition. So how would the definition look like if the underlying type is `std::uint64_t` and developer needs to manually write all 64 binary literals like:

```c++
enum class HugeFlags : std::uint64_t {
    // ...
    huge_flag_a = 0b0000000000000001000000000000000000000000000000000000000000000000,
    huge_flag_b = 0b0000000000000010000000000000000000000000000000000000000000000000,
    // ...
};
```

I think this looks pretty chaotic already... But wait... This is not the end. Imagine several developers updating these flags over time. Even if those changes are not happening that often, updating flags requires lots of brainwork and additional effort which we all want to avoid.

Furthermore, we might specify a bit larger type than we actually need. E.g. if we need only 28 bit flags and we specify `std::uint64_t` as an underlying type, we waste some precious space there which, on some low-memory systems, might be critical.

At the end, I think we can agree that implementing binary flags by using bare `enum class` is manual and error-prone process that can get pretty nasty in some cases.

That being said, in the next few sections, we will discover some alternatives to it...

## Using `std::bitset`

`std::bitset`, introduced with C++11 standard, represents a fixed-size sequence of N bits. When using it, we don't need to overload all the necessary bitwise operators because those are already supported by the C++ standard. The only thing we need to do is to create a small wrapper class that would cross the bridge between the enumerated flags and `std::bitset`.

Let's take a look how such wrapper class could look like:

```c++
template <typename EnumT>
class Flags {
    static_assert(std::is_enum_v<EnumT>, "Flags can only be specialized for enum types");

    using UnderlyingT = typename std::make_unsigned_t<typename std::underlying_type_t<E>>

public:
    Flags& set(EnumT e, bool value = true) noexcept {
        bits_.set(underlying(e), value);
    }

    Flags& reset(EnumT e) noexcept {
        set(e, false);
    }

    Flags& reset() noexcept {
        bits_.reset();
    }

    [[nodiscard]] bool all() const noexcept {
        return bits_.all();
    }

    [[nodiscard]] bool any() const noexcept {
        return bits_.any();
    }

    [[nodiscard]] bool none() const noexcept {
        return bits_.none();
    }

    [[nodiscard]] constexpr std::size_t size() const noexcept {
        return bits_.size();
    }

    [[nodiscard]] std::size_t count() const noexcept {
        return bits_.count();
    }

    constexpr bool operator[](EnumT e) const {
        return bits_[underlying(e)];
    }

private:
    static constexpr UnderlyingT underlying(EnumT e) {
        return static_cast<UnderlyingT>(e);
    }

private:
    std::bitset<underlying(EnumT::size)> bits_;
};
```

and now we can create enumeration for our autonomous driving software:

```c++
enum class CarState : std::uint8_t {
    engine_on, // 0
    lights_on, // 1
    wipers_on, // 2
    // ...
    size
};
```

Please note that we don't have binary literals anymore. We got rid of that visual noise and, now, each flag represents an index in `std::bitset`.

![Binary Representation]({{ "/assets/img/2020-11-16-different-ways-to-define-binary-flags/binary-representation.png" | relative_url }})

Last enumerator, named `size`, represents the size of `std::bitset` data member of `Flags` wrapper class:

```c++
std::bitset<underlying(EnumT::size)> bits_;
```

Final usage of `Flags` wrapper class would look like:

```c++
using CarStates = Flags<CarState>;

CarStates car_states;
car_states.set(CarState::engine_on);

if (car_states[CarState::engine_on]) {
    std::cout << "We are ready to go!" << std::endl;
}

if (car_states[CarState::wipers_on]) {
    std::cout << "Oh, it's raining!" << std::endl;
}

// ...
```

### So `std::bitset` Is Perfect? Or Not?

`std::bitset` approach seems pretty good as it covers all the cases C++ developer would need when it comes to handling binary flags and what's the best thing - it is part of the C++ standard!

Other pros of `std::bitset` are:

- support for as many flags as you wish

Even though the approach looks nice, it has some cons too:

- end user might forget to specify `size` enum value and run into compilation error too often
- end user can set enum values on his/her own and possibly break the functionality of wrapper class
- depending on its implementation, size of `std::bitset` can be bigger than what you actually need (e.g. `sizeof(std::bitset<8>)` can result in 8 bytes but you can have 8 flags in only one byte)

Now, let's see if we can do better or at least as good as `std::bitset`...

## `bitflags` Library

Few months ago, I started writing [bitflags](https://github.com/m-peko/bitflags) library which purpose is to make life easier when it comes to defining binary flags. It gives you an opportunity to choose between two types of flags: raw flags and ordinary flags. The difference between those two is that raw flag comes with minimum overhead while ordinary flag contains its string representation.

So, how to define raw binary flags and ordinary flags? It is as easy as:

```c++
// raw binary flags
BEGIN_RAW_BITFLAGS(CarState)
    RAW_FLAG(engine_on)
    RAW_FLAG(lights_on)
    RAW_FLAG(wipers_on)
    // ...
END_RAW_BITFLAGS(CarState)

// ordinary binary flags
BEGIN_BITFLAGS(CarState)
    FLAG(engine_on)
    FLAG(lights_on)
    FLAG(wipers_on)
    // ...
END_BITFLAGS(CarState)
```

As you might have noticed, there is no assigning binary literals because library itself generates flag values automatically. Furthermore, there is no need to specify the underlying type of our binary flags. Library will detect the most suitable underlying type based on the number of the flags specified. So, if the number of flags is 5 then the underlying type would be `std::uint8_t`, if the number of flags is 16 then the underlying type would be `std::uint16_t` etc. End user does not need to care about these things that might be the most error-prone part of the process of defining binary flags.

### It Has Its Weaknesses Too

Here, I brought an overview of all pros and cons regarding `bitflags` library. If you feel like something is missing, please add up in the comment section below!

Pros:

- no need to specify underlying type
- no need to specify enumeration values
- small size (in case of 8 flags, `sizeof(CarState)` results in only 1 byte)
- associated string representation (does not come cheap though)

Cons:

- macro usage (which most of C++ developers don't like)
- max number of flags is 64

In the following section, we will see a performance comparison between [bitflags](https://github.com/m-peko/bitflags) library and `std::bitset`'s approach explained in the previous section...

## Benchmark

Now, the last thing that remains is to compare the performance of `bitflags` library and `std::bitset`. Since ordinary flags come with string representation that's not cheap, I used raw flags instead.

As you might see on the following figure, there is no much difference between raw flags and `std::bitset`. Full benchmark with ordinary flags included is available [here](https://github.com/m-peko/bitflags).

![Benchmark]({{ "/assets/img/2020-11-16-different-ways-to-define-binary-flags/raw-bitflags-bitset-benchmark.png" | relative_url }})

## Conclusion

Now that we have discussed several ways to define binary flags, I want you to let me know your thoughts and suggestions as well as how you solve this problem in everyday life. Feel free to write comments below... <span>⬇️</span><span>👇</span>
