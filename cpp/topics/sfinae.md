# SFINAE

Иногда требуется делать что-то в зависимости от того, есть ли у класса некоторый метод(
например ```allocator_traits``` пытается понять про методы ```construct``` и ```destruct```).

### Идея и простейший пример

```cpp
template <typename T>
auto f(const T& x) -> decltype(T().size()) {
  std::cout << 1;
  return 1;
}

size_t f(...) {
  std::cout << 2;
  return 2;
}
...
std::vector<int> v{1, 2, 3};
f(v);
f(1);
```
Что будет при выполнении такой программы?
Выводом будет ```12```.

Давайте разберём по пунктам мною написанное.

При вызове ```f(1)``` компилятор очевидно предпочтёт первую версию, т.к. она более частная,
но споткнётся на выводе типа, ведь у ```int``` нет метода ```size()```.
Почему же просто не выкинуть compile error?
Логика следующая: если не получается инстанцировать функцию(не тело, а именно сигнатуру),
компилятор не выдаст ошибку, а просто выкинет её из кандидатов и будет искать заново.

### Некоторые интересные техники


### Проверка наличия метода в классе

```cpp
template <typename T, typename... Args>
struct has_f {
 private:
  template <typename TT, typename... Aargs>
  constexpr static auto f(int) -> decltype(std::declval<TT>().f(std::declval<Aargs>()...), int()) {
    return 1;
  }
  
  template <typename...>
  constexpr static char f(...) {
    return 0;
  }

 public:
  static const bool value = f<T, Args...>(0);
};

template <typename T, typename... Args>
bool has_f_v = has_f<T, Args...>::value;
...
struct T {
  void f(int);
};

std::cout << has_f_v<T, int> << std::endl;
std::cout << has_f_v<T, int, int> << std::endl;
```

Обратим внимание на несколько вещей:
1. Мы пишем ```decltype```, после чего через запятую указываем тип.
Это делается для того, чтобы был выведен нужный нам тип, но и требуемое выражение
было проверено.
Такой способ называется comma trick.
2. Если не указать отдельные шаблоныне аргументы для ```f```, то в случаях,
когда метода нет, мы получим ошибку компиляции, т.к. инстанцирование произойдёт
во время инстанцирования класса, а не функции.
3. ```std::declval``` используем для случая, 
если у типа не окажется конструктора по умолчанию.

Однако текущая форма не совсем является корректным примером метапрограммирования,
т.к. метапрограммирование происходит над типами.
Модернизируем:
```cpp
template <typename T, T value_>
struct integral_constant {
  static const T value = value_;
};

struct true_type : public integral_constant<bool, true> {};
struct false_type : public integral_constant<bool, false> {};

template <typename T, typename... Args>
struct has_f {
 private:
  template <typename TT, typename... Aargs,
        typename = decltype(std::declval<TT>().f(std::declval<Aargs>()...))>
  static true_type f(int);
  
  template <typename...>
  static false_type f(...);

 public:
  using type = decltype(f<T, Args...>(0));
};

template <typename T, typename... Args>
bool has_f_v = std::is_same_v<typename has_f<T, Args...>::type, true_type>;
```
Тут мы сделали несколько улучшений: 
все вычисления производятся над типами(более чистое метапрограммирование);
переход к значению происходит в самом конце; 
тела функций не нужны, т.к. всё решается только с помощью сигнатур.

Стоит понимать, что не удастся реализовать метафункцию, которая будет принимать
название метода как параметр.
Разве что только с помощью макросов:
```cpp
#define HAS_METHOD(NAME) \
template <typename T, typename... Args> \
struct has_##NAME { \
 private: \
  template <typename TT, typename... Aargs, \
        typename = decltype(std::declval<TT>().NAME(std::declval<Aargs>()...))> \
  static true_type f(int); \
  template <typename...> \
  static false_type f(...); \
 public: \
  using type = decltype(f<T, Args...>(0)); \
}; \
template <typename T, typename... Args> \
bool has_##NAME##_v = std::is_same_v<typename has_##NAME<T, Args...>::type, true_type>;
```

### ```std::enable_if```

Предположим, у нас есть две функции:
```cpp
template <typename T>
void f(const T&) {
  std::cout << 1;
}

void f(...) {
  std::cout << 2;
}
```
И мы хотим, чтобы в первую, например, мы попадали только тогда, когда тип ```T```
является классом или, например, он может быть сконструирован от некоторых аргументов.
Т.е. для случаев, когда тип удовлетворяет некоторому метапредикату.
```cpp
template <typename T, typename = std::enable_if_t<std::is_class_v<T>>>
void f(const T&) {
  std::cout << 1;
}
```
Т.е. если некоторое метаусловие не выполняется, то мы хотим убрать функцию
из кандидатов при выборе перегрузки.

Реализуем его c помощью следующего трейта:
```cpp
template <class T>
struct type_is { using type = T; }

template <bool, class T = void>
struct enable_if : type_is<T> {};

template <class T>
struct enable_if<false, T> {};

template <bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;
```
Т.е. в случае ```true``` мы попадём в специализацию, в которой есть ```type```, 
а иначе его не будет и произойдёт неудачная шаблонная подстановка.

Зачем может быть нужен тип в ```enable_if```?
Иногда мы хотим получить различные типы в зависимости от выполнения
некоторых условий:
```cpp
template <class T>
std::enable_if_t<std::is_integral_v<T>, long long int> f(T val) {...}

template <class T>
std::enable_if_t<std::is_floating_point_v<T>, long long int> f(T val) {}
```

### Заметки о том, как это было раньше

Во времена, когда не существовало ключевого слова ```constexpr```,
писали вот так:
```cpp
int const x = 1;
```
