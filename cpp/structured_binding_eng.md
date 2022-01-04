# How to add structured bindings support to your classes(C++17)?

Simple classes like this can be decomposed with no additional code:

```сpp
struct A {
  int a;
  double b;
  std::string s;
};

A f();

auto [a, b, c] = f();
```

For decomposing more complex classes you should hint to compiler with 
```std::tuple_size, std::tuple_element, get``` 
utilities.

So, we want to decompose such class:

```сpp
class Config {
    std::string name;
    std::size_t id;
    std::vector<std::string> data;

    //constructors and such
};
```

The simplest specialization is std::tuple_size. Since there are three elements, we’ll just return 3.

```сpp
namespace std {
    template<>
    struct tuple_size<Config>
        : std::integral_constant<std::size_t, 3> {};
}
```

Next is get. We’ll use C++17’s if constexpr for brevity. I’ve just added this as a member function to avoid the headache of template friends, but you can also have it as a non-member function found through ADL.

```сpp
class Config {
    //...
public:
   template <std::size_t N>
   decltype(auto) get() const {
       if      constexpr (N == 0) return std::string_view{name};
       else if constexpr (N == 1) return id;
       else if constexpr (N == 2) return (data); //parens needed to get reference
   }
};
```

Finally we need to specialize std::tuple_element. For this we just need to return the type corresponding to the index passed in, so std::string_view for 0, std::size_t for 1, and const std::vector<std::string>& for 2. We’ll cheat and get the compiler to work out the types for us using the get function we wrote above. This way, we don’t need to touch this specialization if we want to change the types we return later, or want to add more variables to the class.

```сpp
namespace std {
    template<std::size_t N>
    struct tuple_element<N, Config> {
        using type = decltype(std::declval<Config>().get<N>());
    };
}
```

Or you could do this the long way if you aren’t comfortable with the decltype magic:

```сpp
namespace std {
    template<> struct tuple_element<0,Config> { using type = std::string_view; };
    template<> struct tuple_element<1,Config> { using type = std::size_t; };
    template<> struct tuple_element<2,Config> { using type = const std::vector<std::string>&; };
}
```

With all of that done, we can now decompose Config like so:

```сpp
Config get_config();

auto [name, id, data] = get_config();
```

### Use note

If you declare variables, you must specify ```auto```
(you can't specify type explicitly).

If you want to use with declared variables you should use ```std::tie```:
```сpp
std::pair<int, int> f() {
    return {1, 2};
}

int a, b;
std::tie(a, b) = f();
```
