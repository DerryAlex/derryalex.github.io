---
title: C++23 Range Formatting 实用指南
date: 2026-02-10
slug: cpp-range-format
tags:
  - Programming
  - C++
---

随着近期 GCC 15.2 的发布，C++23 的 range formatting 已在所有主流编译器上获得支持。这意味着你现在可以直接使用 `std::format` 来格式化像 `std::vector` 和 `std::map` 这样的容器，无需手动编写循环。尽管网上有一些简单的例子，但我还没有找到能与 `std::format` 指南相媲美的介绍。本文旨在提供一个相对全面的指南。

> [!NOTE]
> 本文中文版由英文版用 LLM 翻译而来，并人工作了少许润色。

## 背景

### std::format

C++20 中引入了 `std::format`，这提供了一种现代化的格式化方法。

```cpp
#include <format>
#include <iostream>

int main() {
    /* 输出 `Hello, world!` */
    std::cout << std::format("Hello, {}!", "world") << std::endl;
    return 0;
}
```

可以通过特化 `std::formatter` 来支持自定义类型。

{{< details summary="std::formatter 特化示例" >}}

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
    /* 输出 `(1, 2)` */
    std::cout << std::format("{}", p) << std::endl;
    return 0;
}
```

{{< /details >}}

### Range formatting

C++23 引入了 range formatting，允许你轻松格式化 `vector`、`map` 等容器。在 C\+\+23 中，我们可以使用 `std::print` 进行输出。

```cpp
#include <format>
#include <iostream>
#include <print>
#include <set>
#include <version>

int main() {
    const std::set<int> deadbeef{0xDE, 0xAD, 0xBE, 0xEF};

    /* C++23 之前 */
    bool first = true;
    std::cout << "{";
    for (const auto& x: deadbeef) {
        if (!first) std::cout << ", "; else first = false;
        std::cout << std::format("{:#x}", x);
    }
    std::cout << "}" << std::endl;

    /* C++23 之后 */
#ifdef __cpp_lib_format_ranges
    /* 输出 `{0xad, 0xbe, 0xde, 0xef}` */
    std::println("{::#x}", deadbeef);
#endif

    return 0;
}
```

第二个 `:` 之后的格式说明符（`#x`，带前缀的十六进制）将应用于每个元素。

## 带自定义元素类型的 range formatting

将 range formatting 和自定义类型格式化结合起来并不困难。

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
    /* 输出 `[(1, 2), (3, 4), (5, 6)]` */
    std::println("{}", ps);
    return 0;
}
```

### 一个小挑战

为什么需要 `template<typename Out>`？如果将 `basic_format_context<Out, char>` 替换为 `format_context`，编译将失败并报错 `std::formatter must be specialized for each type being formatted`。

**TL;DR:** `formattable<Point>` 要求 `format` 函数接受的 context 类型不仅仅是 `std::format_context`。使用 `template<typename Out>` 可以让它同时适用于两种 context。

{{< details summary="详情" >}}

以下是来自 libstdc++ 的实现。

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

`formatter<range>` 委托给底层的 `range_formatter`。之前的错误信息表明替换失败了。让我们尝试实例化一个 `range_formatter<Point>` 看看会发生什么。编译器现在提示 `__formattable_with<Point, basic_format_context<_Iter_for_t<char>, char> >` 的条件不满足，而这是 `formattable<Point>` 所要求的。*注意：任何以下划线 `_` 开头的内容都是与实现相关的。* 然而，`format_context` 是 `basic_format_context<_Sink_iter<char>, char>` 的别名，它不能满足条件。

{{< /details >}}

**Takeaway**：为了兼容性，将 `format` 函数模板化。

## 自定义 range 类型的格式化

如果我们的类满足 `input_range` 概念，那么 range formatting 应该就可以生效。以下示例模拟了 Python 的 `range` 类。（为简洁起见，仅展示部分代码。）

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
    /* 输出 `xrange(3) = [0, 1, 2]` */
    std::println("xrange({}) = {}", n, xrange(n));
    return 0;
}
```

{{< details summary="完整代码">}}

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
    /* 输出 `xrange(3) = [0, 1, 2]` */
    std::println("xrange({}) = {}", n, xrange(n));
    return 0;
}
```

{{< /details >}}

## Range formatting 修改样式

`range_formatter` 有三个成员函数可用于修改输出的格式。

```cpp
constexpr void set_separator( std::basic_string_view<CharT> sep ) noexcept;
constexpr void set_brackets( std::basic_string_view<CharT> opening,
                             std::basic_string_view<CharT> closing ) noexcept;
constexpr std::formatter<T, CharT>& underlying() noexcept; /* 以及 const 变体 */
```

在下面的例子中，我们假设 `vector<vector<T>>` 代表一个矩阵，应该以不同的方式格式化。（在任何严肃的项目中，*都不要*为 `std::vector` 这样的标准库类型特化 `std::formatter`。）

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
    /* 输出
     * ```
     * [1 2
     *  3 4]
     * ```
     * 而不是 `[[1, 2], [3, 4]]`
     */
    std::println("{}", mat);
    return 0;
}
```

## 结论

C++23 的范围格式化是对 `std::format` 功能的一个强大补充，显著减少了处理容器时的代码。要点：

1.  为了兼容性，始终模板化 `format` 函数。
2.  如果类型满足 `input_range` 概念，会自动获得范围格式化功能，无需特化 `std::formatter`。
3.  使用 `range_formatter::set_separator()`、`set_brackets()` 和 `underlying()` 来修改特定类型的格式化输出。

虽然由于不太友好的错误信息，可能会想为像 `vector<MyType>` 这样的容器实例特化格式化器，但实现一个正确的 `formatter<MyType>` 可以更优雅地解决问题。

此外，与 SFINAE 相比，C++20 概念带来的改进的错误信息让调试变得容易得多。
