---
layout: post
title: "Serialization with no efforts"
categories: [c++]
tags:
  serialization
  meta-programming
  concepts
  reflection
  metaclasses
---

In this post, I want to share my way how I usually deal with serialization using meta-programming.

## Motivation

Once you need to send or receive your data, you have to solve the serialization problem. Alternatively, imagine you want to print your data in your terminal. Commonly, you write a bunch of functions that would iterate through the members in your objects and apply logic code (sending, receiving, or printing.) This approach has many problems:
- it is very wordy, repetitive, and error-prone,
- it is not generic, and every type, you want to work with, has to be done individually,
- it is very tedious and tiresome.

I invite readers to address those problems with C++ meta-programming.

To reduce wordiness, we abstract our logic code from the procedure iterating through members.
The iteration procedure itself is simplified dramatically by exposing your members with a tuple.
Furthermore, once you have exposed it, you don't have to write anything else, except a special case when a type differs from normal behavior.
In the end, you get the compiler doing the job for you in a type-safe manner, avoiding repetitive work and making fewer mistakes — good motivation to feel content.

## Members exposure

Let's begin with exposing our members so that they are available for iterating. Imagine we have a type:

```c++
struct employee {
  std::string name;
  int salary;
};
```

We want to have `name` and `salary` members available so that the callers, performing serialization, don't depend on the concrete type, `employee`. That is, for a given `T` we should be able to access exposed members. To achieve it we define `members` member function.

```c++
struct employee {
  std::string name;
  int salary;

  auto members() noexcept {
    return std::forward_as_tuple(name, salary);
  }
  auto members() const noexcept {
    return std::forward_as_tuple(name, salary);
  }
};
```

Having those "elemental" functions defined, we can do whatever we want.

## Logic code abstraction

Once you have a tuple, you can use the standard library to perform an iteration and apply your logic code on its elements.
Let's write a simple serializer into a stream:


```c++
template <typename T>
std::ostream& operator << (std::ostream& os, T const& obj) {
  using std::operator <<;

  std::apply([&os](auto const& fst, auto const&... rest) {
    os << fst;
    ((os << ", " << rest), ...);
  }, obj.members());
  return os;
}
```
    
Now, we can instantiate an `employee` and easily print it:

```c++
employee e{"Steve Jobs", 1};
std::cout << e << std::endl;
```

The code above prints

```
Steve Jobs, 1
```

A complete and working example can be found at [coliru](https://coliru.stacked-crooked.com/a/747d34b510dd8c46) (or [gist](https://gist.github.com/deni64k/6077048dba21f92b7b70d3f1c614462d) if unreachable).

## Simplification and Systemizing

There is, definitely, some amount of boilerplate: we have to maintain two versions of `members` functions. The advanced use of preprocessor can help us to systemize our approach. Let's define a few macros. I'm using [Boost.Preprocessor](https://www.boost.org/doc/libs/1_67_0/libs/preprocessor/doc/index.html) for simplicity.

```c++
#define EXPOSE_MEMBERS(...)                                     \
    auto members() {                                            \
      return std::forward_as_tuple(__VA_ARGS__);                \
    }                                                           \
    auto members() const {                                      \
      return std::forward_as_tuple(__VA_ARGS__);                \
    }                                                           \
    static constexpr auto names() {                             \
      return std::make_array(                                   \
        BOOST_PP_LIST_ENUM(                                     \
          BOOST_PP_LIST_TRANSFORM(                              \
            EXPOSE_MEMBERS_Q, @,                                \
            BOOST_PP_VARIADIC_TO_LIST(__VA_ARGS__))));          \
    }
```

Now, we can reduce our example to:

```c++
struct employee {
  std::string name;
  int salary;

  EXPOSE_MEMBERS(name, salary);
};
```

You may notice we have added static function `names` that returns an array with member names in the same order as in `members`. It allows us to create, for example, a pretty-print function or json serializers.

Let's look at better versions of stream operators.

The `operator <<` prints the member name as well:

```c++
template <typename T>
std::ostream& operator << (std::ostream& os, T const& obj) {
  using std::operator <<;

  std::apply([&os](auto const& names,
                   auto const& fst,
                   auto const&... rest) {
    unsigned int i = 0;
    os << names[i] << '=' << fst;
    ((os << ", " << names[++i] << '=' << rest), ...);
  }, std::tuple_cat(std::make_tuple(obj.names()),
                    obj.members()));
  return os;
}
```

The `operator >>` allows us to deserealize an `employee` from a string.

```c++
template <typename T>
std::istream& operator >> (std::istream& is, T& obj) {
  using std::operator >>;

  std::apply([&is](auto&... members) {
    (is >> ... >> members);
  }, obj.members());
  return is;
}
```

Let's use them:

```c++
employee e{"Steve Jobs", 1};

std::cout << e << std::endl;
// Prints
// name=Steve Jobs, salary=1

std::string input = "Bill-Gates 100";
std::istringstream ss{input};
ss >> e;

std::cout << e << std::endl;
// Prints
// name=Bill-Gates, salary=100
```

And this approach is appliable for any type simply by using one macro `EXPOSE_MEMBERS`.

A complete and working example can be found at [coliru](https://coliru.stacked-crooked.com/a/037b4d0823e6ff16) (or [gist](https://gist.github.com/deni64k/2e118d8274df6d46d990ff2511152a16) if unreachable).

## Further improvements

### Constrained function templates

The operators (or any other serializers) have very wide input template parameters. It is pretty easy to come up with a SFINAE-based solution, but I will show it with [Concepts](https://en.cppreference.com/w/cpp/language/constraints). We need to make sure the serializable object contains methods `members` and `names`.

```c++
template <typename T>
concept bool Serializable = requires {
    std::declval<T>().members();
    std::declval<T>().names();
};
```

Now, to apply constraints `Serializable` on a template parameter, you simply write `template <Serializable T>` instead of `template <typename T>`.

Note: don't forget to pass `-fconcepts` to gcc while experimenting.

Example at [coliru](https://coliru.stacked-crooked.com/a/cc8cf550742ef845).

### Reflection

The use of preprocessor becomes unnecessary when [Reflection proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0194r3.html) gets approved and mainstreamed. But for now, we have to list members by hand. A good part, you get a compilation error if there is a typo in member names.

### Metaclass

With metaclasses you are able to define such a metaclass that runs exposing automatically. I highly recommend to read [Metaclasses: Generative C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0707r3.pdf).
