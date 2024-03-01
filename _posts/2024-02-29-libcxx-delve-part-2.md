---
layout: post
title: Delving into libc++ - part 2: `from_chars` and the task ahead.
categories: [libc++, C++]
---
Before we step into the details of what a floating point is or what the insides of the libc++ code look like, we should take a moment and ask ourselves: what is `from_chars`?

To answer this question we will look at two sources which all contain parts of the answer. First one on the docket:

## [cppreference](https://en.cppreference.com/w/cpp/utility/from_chars)

We find something interesting stuff already. For one, the signature of `from_chars`

```
std::from_chars_result
    from_chars( const char* first, const char* last,
                /* floating-point-type */& value,
                std::chars_format fmt = std::chars_format::general );
```
Seems to return some kind of result, takes in pointers to a first and last character. Interestingly, it takes in a floating point type by _reference_ which we do not see often in the standard. Finally it takes in some kind of format specifier. Let's break it point by point.

<!--more-->

`std::from_chars_result`: this class contains two things: a `const char*` to the first character _not_ matching the pattern (which we will talk about later) then an `std::errc` object which specifies whether an error happened or not. In the case it did, then the return `std::errc` will simply be `std::errc{}` and the class offers an overload of `==` to check this. Otherwise, the value is either `std::errc::invalid_argument` or `std::errc::result_out_of_range`.

`const char* first` and `const char* last`: As mentionned in the last blog, taking an `std::string` would incure unnecessary copies so we got with `const char*`. This allows us to use `std::string_view` and `std::from_chars` to obtain basically have zero copy. Understandably, `first` and `last` must be a valid range.

`/* floating-point-type */& value`: The wording changed in 2023 due to this proposal [P1467R4](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1467r4.html#motivation) which introduces _extended floating-point types_ and defines _floating-point types_ to be the standard floating-point types (i.e. `float`, `double` and `long double`) and these extended floating-point types. The proposal also mentions that overload should be resolved with value-preserving overloads (`double` to `double`) or to lower-rank overload (`double` to `float`). We also see here one good reason to have `from_chars` in the standard:

```No changes [...] - The strtod and stod families of functions: With different names for each floating-point type (which for strtod was inherited from C), that scheme doesn’t work well for extended floating-point types.```

Meaning that, in the future, conversion from string to extended floating-point types will need to go through `from_chars`.

> Windows's extended floating-point types are all 64 bits, but libc++'s implementation will have to support higher bits representation. This is a big difference between both implementations.
> The proposal also introduces `std::float16_t` which we will need to take care of. I will get in contact with algorithm writers to see if 32/64 bits algorithm work for this size.

So we need to support overloads for all these types. Which should be... Fine. We can start with standard types and go from there. We haven't discussed what _value_ the floating-point types can take but we will do this in another post when discussing them in more depth.

Finally `std::chars_format` which defaults to `std::chars_format::general`. This enum specifies the formatting of the floating-point value we are reading from. It has four values:
    - `std::chars_format::hex`: parse the string as if it was in hexadecimal. Does not permit hexadecimal prefixes like `0x` and `0X`.
    - `std::chars_format::scientific` but not `std::chars_format::fixed`: exponent part is required.
    - `std::chars_format::fixed` but not `std::chars_format::scientific`: exponent part is not permitted.
    - `std::chars_format::general` (which `std::chars_format::scientific` and `std::chars_format::fixed`): exponent part is optional.

There is some other formatting requirements, but those are intrinsic to `from_chars` and cannot be changed:
    - Only a minus sign is allowed at the beginning. (Early return condition)
    - Just a sign is consider not matching anything. (Early return condition)
    - Leading whitespaces are not ignored. (Early return condition)
    - Digits might contain a period. This provides the significand (We will talk later about what that is)
    - An optional pattern of one `e` or `E` and `+` or `-` followed by some digits. This is the exponent (in base 10).
    - The same is true for `std::chars_format::hex` but the exponent is `p` or `P`.
    - Case insensitive "inf" or "infinity".
    - Case insensitive "NAN".

And then we have the return value. When `from_chars` is successful, `value` is equal to `one of at most two floating-point values closest to the value of the string matching the pattern, after rounding according to std::round_to_nearest.` (Again, we will talk about rounding floating-point number in time) and the `ptr` value is either `end` or the first invalid character. Otherwise, `ptr` is first and we return the `invalid_argument` error code. For values too big, we set `ptr` to the first invalid character and return `out_of_range`.

## [P0067R5](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0067r5.html)
Where it all began: the proposal. This gives a history of string conversion in C and C++ and why `from_chars` came to be. As we can see, the shortcomings come from different areas:
    - Locale. Respecting local is one of the biggest issue since it requires so much work.
    - Memory allocation. Allocating memory is *slow* and we would like to avoid it when possible.
    - Error handling. Some of them do not handle errors, others throw, which, again, slows things down.

So we will have to ensure we do none of those things during `from_chars`. We also see the wording required for our implementation (note that this was before the extended floating-point type proposal):
```
struct from_chars_result {
    const char* ptr;
    error_code ec;
  };

  from_chars_result from_chars(const char* first, const char* last, see below& value, int base = 10);  

  from_chars_result from_chars(const char* first, const char* last, float& value, chars_format fmt = chars_format::general);  
  from_chars_result from_chars(const char* first, const char* last, double& value, chars_format fmt = chars_format::general);  
  from_chars_result from_chars(const char* first, const char* last, long double& value,  chars_format fmt = chars_format::general);
```

We also have some functional requirements to keep in mind:
    1. [first,last) is required to be a valid range. So we need to ensure it is.
    2. If no characters match the pattern, value is _unmodified_, the member ptr of the return value is first and the member ec is equal to errc::invalid_argument.
    3. The member ptr of the return value points to the first character not matching the pattern.

## List of useful links
- [Floating-point types definition on cppreference](https://en.cppreference.com/w/cpp/language/types)
- [Fixed width floating-point types](https://en.cppreference.com/w/cpp/types/floating-point)
- [Macro to check if `from_chars` is enabled](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_to_chars)

