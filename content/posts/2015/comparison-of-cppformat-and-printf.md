---
title: Comparison of C++ Format and C library's printf
date: 2015-02-26
aliases: ['/2015/02/26/comparison-of-cppformat-and-printf.html']
---

<div style="clear:right; float:right; margin-left:1em; margin-bottom:1em">
  <img src="/img/printf.jpg" width="280">
</div>

I was recently asked on GitHub why would someone use [`fmt::printf`](
https://fmt.dev/latest/api.html#printf-api), which is a part of the [C++ Format
library](https://github.com/cppformat/cppformat), over the standard C library
`printf`. This is a fair question and so I decided to write this blog post
comparing the two.

Disclaimer: I'm the author of C++ Format, so this is all very biased =).

Whenever I mention `printf` without a namespace in this post I mean the whole
family of `printf` functions from the C standard library, including `sprintf`
and `fprintf`. The C++ Format library includes its own implementation of
`printf` which resides in the `fmt` namespace, so I'll use qualified name
`fmt::printf` to distinguish it from the standard one.

## Safety

The main difference between `printf` and C++ Format is the way they accept
formatting arguments.

`printf` uses the mechanism provided by the C language sometimes referred to
as [varargs](https://en.wikipedia.org/wiki/Variadic_function). This mechanism is
inherently unsafe because it relies on the user to handle the type information
which, in the case of `printf`, is encoded in the format string together with
formatting options. GCC provides an extension, [_\_attribute__((format (printf,
...))][1], that partially addresses safety issues. However, this only works with
format strings that are string literals which is not always the case especially
when strings are localized. Also not all C and C++ compilers support this
attribute.

[1]: http://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html

C++ Format uses [variadic templates][2] introduces in C++11 and emulates them on
older compilers. This ensures complete type safety. Mismatch between the format
specification and the actual type causes an exception while in `printf` it often
leads to a segfault.

[2]: https://en.wikipedia.org/wiki/Variadic_template

Here's a small artificial example that illustrates the problem:

```c++
#include <stdio.h>

int main() {
  const char *format_str = "%s";
  printf(format_str, format_str[0]);
}
```

Let's compile and run it:

```
$ g++ -Wall -Wextra -pedantic test.cc
$ ./a.out 
Segmentation fault (core dumped)
```

Note that GCC doesn't even give a warning with `-Wall -Wextra -pedantic`.

As was pointed out in one of the comments on Reddit, you can get a warning with
`-Wformat-nonliteral`. As the name suggests, this will only warn you that
the format string is nonliteral.

Here's the C++ Format version:

```c++
#include "format.h"

int main() {
  const char *format_str = "%s";
  try {
    fmt::printf(format_str, format_str[0]);
  } catch (const std::exception &e) {
    fmt::printf("%s\n", e.what());
  }
}
```

Let's compile and run it:

```
$ g++ test.cc
$ ./a.out 
unknown format code 's' for integer
```

As you can see the error is reported as an exception that can be safely catched
and handled. C++ Format doesn't do compile-time checking of literal format
strings though.

## Extensibility

It is often said that `printf` is not extensible and doesn't support
user-defined types. This is not entirely true as the GNU C Library provides some
[extension mechanisms][3]. However, as with attributes, they are not standard.

[3]: http://www.gnu.org/software/libc/manual/html_node/Customizing-Printf.html

The current version of C++ Format can format any type that provides overloaded
`operator<<` in addition to all built-in types, for example:

```c++
class Date {
  int year_, month_, day_;
 public:
  Date(int year, int month, int day) : year_(year), month_(month), day_(day) {}

  friend std::ostream &operator<<(std::ostream &os, const Date &d) {
    return os << d.year_ << '-' << d.month_ << '-' << d.day_;
  }
};

std::string s = fmt::format("The date is {}", Date(2012, 12, 9));
// s == "The date is 2012-12-9"
```

Note that the above example uses Python-like [format string syntax][4] which is
also supported (and recommended) by C++ Format.

[4]: https://fmt.dev/latest/syntax.html

Since C++ Format manages type information automatically, you don't have to
encode it in the format string as with `printf`, so the same default format
specifier `{}` can be used with objects of any type. This can also result in
shorter format strings as format specifiers like `hh` or `ll` are not required
any more.

## Performance

The current version of C++ Format library is [much faster on integer
formatting][5], but not as fast on floating-point formatting [compared to
EGLIBC](https://github.com/cppformat/cppformat#speed-tests). The library relies
on `snprintf` for floating-point formatting and does some additional processing
to ensure consistent output across platforms. This will be improved in the next
version.

[5]: http://zverovich.net/2013/09/07/integer-to-string-conversion-in-cplusplus.html

## Portability

Being part of the C standard library, `printf` is available pretty much
everywhere, but the quality of implementations widely vary and there are
inconsistencies between output on different platforms especially with
floating-point numbers.

C++ Format has been tested on all major platforms and compilers. It provides
consistent output across platforms and implements POSIX extensions for
positional arguments that are not available in some `printf` implementations.

## Compiled code size

According to the [code bloat benchmark][6], C++ Format adds about 10% to the
optimized code size compared to `printf`. The additional size comes from the
type information that is passed alongside the formatting arguments.

[6]: https://github.com/cppformat/cppformat#compile-time-and-code-bloat

In practice, increase in binary size when switching from `printf` to C++
Format is small and, of course, depends on amount of formatting done by the
application. For example, when [0 A.D.](http://play0ad.com/) switched to C++
Format, they reported only 3% increase in binary size on older version of the
library (statically linked) which was more affected by code bloat. On the
current version it would be even smaller.

## Memory management

Memory management is related to safety, but I think it deserves a separate
section not to mix with type safety issues.

[`sprintf`](http://en.cppreference.com/w/cpp/io/c/sprintf) relies on the user
to provide large enough buffer which may easily result in buffer overflow.
This issue is addressed in `snprintf` where you can pass buffer size.
Unfortunately, it is rather awkward to use if you want to grow your buffer
dynamicaly to accommodate large output as the following example taken from
the [`snprintf` manpage](http://linux.die.net/man/3/snprintf) nicely
illustrates:

```c++
char *
make_message(const char *fmt, ...)
{
    int n;
    int size = 100;     /* Guess we need no more than 100 bytes */
    char *p, *np;
    va_list ap;
    if ((p = malloc(size)) == NULL)
        return NULL;
    while (1) {
       /* Try to print in the allocated space */
       va_start(ap, fmt);
        n = vsnprintf(p, size, fmt, ap);
        va_end(ap);

       /* Check error code */
       if (n < 0)
            return NULL;

       /* If that worked, return the string */
       if (n < size)
            return p;

       /* Else try again with more space */
       size = n + 1;       /* Precisely what is needed */
       if ((np = realloc (p, size)) == NULL) {
            free(p);
            return NULL;
        } else {
            p = np;
        }
    }
}
```

C++ Format allocates buffer on stack if the output is small enough and uses
dynamic memory allocation for larger output. This is all done automatically
without user intervention. For example, in the following code everything is
allocated on stack:

```c++
fmt::MemoryWriter w;
w.write("Look, a {} string", 'C');
w.c_str(); // returns a C string (const char*)
```

If necessary you can specify a [custom allocator][7] to be used for large
output.

[7]: http://fmtlib.net/latest/api.html#custom-allocators

## Conclusion

The C++ Format library provides safe extensible alternative to the C `printf`
family of functions with comparable (better on integer formatting) performance
and compiled code size.

**Update**: Added the "Memory management" section.
