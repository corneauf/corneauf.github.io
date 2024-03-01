---
layout: post
title: Delving into libc++ - part 1
categories: [libc++, C++]
---
I have been learning to write my own programming language through the fantastic blog https://craftinginterpreters.com/ only, instead of doing it in Java or C, I have decided to do it in C++ since I have been wanting to learn more about advanced stuff. So far I have used nice features like `std::optional` and `std::variant` that I have not had the chance to play with before. Unfortunately all of this came to a head when it came to parsing string to support floats.
<!--more-->
See, this language supports two fundamental types: strings and floats. 1? that's a float. 2.0? That's a float. Therefore, when parsing the code, you need to convert the string representation "1.0" into a float. So, there you are, coding away and you need to parse a string to a float, what is your first instinct? Yes, look at cppreference.com. Let us see what we have:
- First we got `std::stof`: https://en.cppreference.com/w/cpp/string/basic_string/stof. While this is nice, our code works with `std::string_view`s so making a copy to call `stof` seems subpar. But, we see that `stof` calls
- `std::strtof`. Coming straight from C, we get our string parsing with no copies since we can just use our `string_viee`'s data and size member functions. But then you see this "nonempty sequence of decimal digits optionally containing decimal-point character (as determined by the current C locale) " and this "any other expression that may be accepted by the currently installed C locale". Dealing with locale stuff does not seem super efficient... Maybe there is a better way
- Third: if you scroll down the page in the "See also" section, you will find `from_chars`. It is C++17, which is not a problem. Notes mention "std::from_chars is locale-independent, non-allocating, and non-throwing." which sounds excellent for speed.

I have never used `from_chars` and it has been in the standard for 7 years! So might as well give it a spin. We write the code and 

```
error: call to deleted function 'from_chars'
        std::from_chars(repr.data(), repr.data() + repr.size(), value);
```

Huh, what? That is weird. Well... Turns out that `from_chars` with floating point types has not been implemented. Not into GCC, not into Clang. After **7 years** the only compiler with support for floating point types `from_chars` is MSVC and that support was completed in 2019, so 5 years ago. As it stands right now, libc++ only has support for `to_chars` (from MSVC) and `from_chars` with integral types.

So, today is the start of my journey to implement floating point types support of `from_chars` into libc++. My plan is this:
- Gonna get accustomed to the codebase and will document my findings and my questions here.
- Will establish exactly the scope of the problems and the definitions we need to work with like, what is a float, how are they represented, things like that.
- Break down the problem into smaller PRs so that incremental change can be merged.
