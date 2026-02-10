---
title: A Practical Guide to C++23 Range Formatting
date: 2026-02-10
slug: cpp-range-format
tags:
  - Programming
  - C++
---

With recent release of GCC 15.2, C++23 range formatting has gained full support of all mainstream compilers. This means you can now format containers like `std::vector` and `std::map` directly with `std::format`, eliminating the need for manual loops. Though there are some simple examples on the Internet, I have not found an introduction comparable to those of `std::format`. This post aims to be a relatively comprehensive guide.

## Background

### std::format

`std::format` was introduced in C++20 to provide a modern way to format strings.

```cpp
#include <format>
#include <iostream>

int main() {
    /* Outputs `Hello, world!` */
    std::cout << std::format("Hello, {}!", "world") << std::endl;
    return 0;
}
```

You can specialize `std::formatter` to support custom types.

{{< details summary="Example for std::formatter specialization" >}}

```cpp
#include <format>
#include <iostream>

struct Point {
    int x, y;
};

template<>
struct std::formatter<Point> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }

    auto format(const Point& p, std::format_context& ctx) const {
        return std::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};

int main() {
    Point p{1, 2};
    /* Outputs `(1, 2)` */
    std::cout << std::format("{}", p) << std::endl;
    return 0;
}
```

{{< /details >}}

### Range Formatting

C++23 introduces range formatting, which allows you to easily format containers like `vector` and `map`. With C\+\+23, we can use `std::print` for output.

```cpp
#include <format>
#include <iostream>
#include <print>
#include <set>
#include <version>

int main() {
    const std::set<int> deadbeef{0xDE, 0xAD, 0xBE, 0xEF};

    /* Before C++23 */
    bool first = true;
    std::cout << "{";
    for (const auto& x: deadbeef) {
        if (!first) std::cout << ", "; else first = false;
        std::cout << std::format("{:#x}", x);
    }
    std::cout << "}" << std::endl;

    /* After C++23 */
#ifdef __cpp_lib_format_ranges
    /* Outputs `{0xad, 0xbe, 0xde, 0xef}` */
    std::println("{::#x}", deadbeef);
#endif

    return 0;
}
```

Format specification after the second `:` (`#x`, hex with prefix) will be applied to each element.

## Range Formatting with User-defined Element Type

It is not hard to combine range formatting and custom type formatting.

```cpp
#include <format>
#include <print>
#include <vector>

struct Point {
    int x, y;
};

template<>
struct std::formatter<Point> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }

    template<typename Out>
    auto format(const Point& p, std::basic_format_context<Out, char>& ctx) const {
        return std::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};

int main() {
    std::vector<Point> ps{{1, 2}, {3, 4}, {5, 6}};
    /* Outputs `[(1, 2), (3, 4), (5, 6)]` */
    std::println("{}", ps);
    return 0;
}
```

### A Small Challenge

Why is `template<typename Out>` needed? If you replace `basic_format_context<Out, char>` with `format_context`, the compilation will fail with the error `std::formatter must be specialized for each type being formatted`.

**TL;DR:** `formattable<Point>` requires `format` function to accept a different context than `std::format_context`. Using `template<typename Out>` makes it work with both contexts.

{{< details summary="Details" >}}

Here is the implementation from libstdc++.

```cpp
template<typename _Tp, __format::__char _CharT = char>
  requires same_as<remove_cvref_t<_Tp>, _Tp> && formattable<_Tp, _CharT>
class range_formatter;

template<ranges::input_range _Rg, __format::__char _CharT>
  requires (format_kind<_Rg> != range_format::disabled)
    && formattable<ranges::range_reference_t<_Rg>, _CharT>
struct formatter<_Rg, _CharT>
{
private:
  using _Vt = remove_cvref_t<ranges::range_reference_t<_Rg>>;
  using _Formatter_under = range_formatter<_Vt, _CharT>;
  _Formatter_under _M_under;

public:
  /* ... */
};
```

`formatter<range>` simply delegates to underlying `range_formatter`. Previous error message indicates the substitution is failed. Let's try to instantiate a `range_formatter<Point>` and see what happens. The compiler now complains `__formattable_with<Point, basic_format_context<_Iter_for_t<char>, char> >` is not satisfied, which is required by `formattable<Point>`. *Note that anything starts with `_` is implementation specific.* However, `format_context` is an alias for `basic_format_context<_Sink_iter<char>, char>`, which does not meet the condition.

{{< /details >}}

**Takeaway**: For compatibility, template your `format` function.

## Range Formatting with User-defined Range Type

If our class satisfies the `input_range` concept, range formatting should be available. The following example mimics Python's `range` . (For simplicity, only part of the code is shown.)

```cpp
class xrange{
public:
    xrange(int _start, int _stop, int _step = 1);
    explicit xrange(int _end);

    class Iterator {
    public:
        using difference_type = std::ptrdiff_t;
        using value_type = int;
    
        Iterator& operator ++();
        Iterator operator ++(int);
        value_type operator *() const;
    };
    
    class Sentinel {
    public:
        friend bool operator ==(const Iterator& it, const Sentinel& s);
    };
    
    Iterator begin() const;
    Sentinel end() const;
};

int main() {
    int n = 3;
    /* Outputs `xrange(3) = [0, 1, 2]` */
    std::println("xrange({}) = {}", n, xrange(n));
    return 0;
}
```

{{< details summary="Full code">}}

```cpp
#include <format>
#include <print>
#include <stdexcept>

class xrange{
private:
    int m_start, m_stop, m_step;

public:
    xrange(int _start, int _stop, int _step = 1)
      : m_start(_start), m_stop(_stop), m_step(_step) {
        if (m_step == 0) throw std::runtime_error("step must not be zero");
    }
    explicit xrange(int _end) : m_start(0), m_stop(_end), m_step(1) {}

    class Iterator;
    class Sentinel;

    Iterator begin() const {
        return Iterator{m_start, m_step};
    }

    Sentinel end() const {
        return Sentinel{m_stop};
    }

    class Iterator {
    private:
        int m_value, m_step;
        friend Sentinel;

    public:
        using difference_type = std::ptrdiff_t;
        using value_type = int;

        Iterator(int _value, int _step) : m_value(_value), m_step(_step) {}
        Iterator() = default;

        Iterator& operator ++() {
            m_value += m_step;
            return *this;
        }

        Iterator operator ++(int) {
            auto tmp = *this;
            ++*this;
            return tmp;
        }

        value_type operator *() const {
            return m_value;
        }
    };

    class Sentinel {
    private:
        int m_end;

    public:
        explicit Sentinel(int _end) : m_end(_end) {}
        Sentinel() = default;

        friend bool operator ==(const Iterator& it, const Sentinel& s) {
            return (it.m_step >= 0 && it.m_value >= s.m_end)
                     || (it.m_step < 0 && it.m_value <= s.m_end);
        }
    };
};

int main() {
    int n = 3;
    /* Outputs `xrange(3) = [0, 1, 2]` */
    std::println("xrange({}) = {}", n, xrange(n));
    return 0;
}
```

{{< /details >}}

## Range Formatting Customization

`range_formatter` has three member functions that can be used to customize formatting.

```cpp
constexpr void set_separator( std::basic_string_view<CharT> sep ) noexcept;
constexpr void set_brackets( std::basic_string_view<CharT> opening,
                             std::basic_string_view<CharT> closing ) noexcept;
constexpr std::formatter<T, CharT>& underlying() noexcept; /* and const variant */
```

In the following example, we assume that vector of vectors is a matrix and should be formatted differently. (Do *NOT* specialize `std::formatter` for standard library types in any serious project.)

```cpp
#include <format>
#include <print>
#include <vector>

template<typename T>
  requires std::formattable<T, char>
struct std::formatter<std::vector<std::vector<T>>>
  : std::range_formatter<std::vector<T>>
{
    constexpr formatter() noexcept {
        this->set_separator("\n ");
        this->underlying().set_separator(" ");
        this->underlying().set_brackets("", "");
    }
};

int main() {
    std::vector<std::vector<int>> mat{{1, 2}, {3, 4}};
    /* Outputs
     * ```
     * [1 2
     *  3 4]
     * ```
     * instead of `[[1, 2], [3, 4]]`
     */
    std::println("{}", mat);
    return 0;
}
```

## Conclusion

C++23 range formatting is a powerful complement to `std::format` that significantly reduces boilerplate when working with containers.  Takeaways:

1. Always template your `format` function for compatibility.
2. If your type satisfies `input_range`, you get range formatting automatically. No `std::formatter` specialization needed.
3. Use `range_formatter::set_separator()`, `set_brackets()`, and `underlying()` to customize output for specific use cases.

While you might be tempted to specialize formatters for container instantiations like `vector<MyType>` due to the unfriendly error message, a proper `formatter<MyType>` solves the problem more elegantly.

Besides, the improved error messages from C++20 concepts (compared to SFINAE) make debugging much easier. 
