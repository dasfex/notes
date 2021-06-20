## type traits

### Basics

В C++11 появился заголовок \<type\_traits\> для работы
с типами на этапе компиляции.
Некоторые функции пишутся с помощью хитрых трюков и фич, но некоторые
можно написать и сейчас.

Например давайте напишем ```is_same```(проверку во время компиляции
равны ли два типа):
```cpp
template <typename T, typename U>
struct is_same {
  static constexpr bool value = false;
};

template<typename T>
struct is_same<T, T> {
  static constexpr bool value = true;
};
```
Ещё для примера напишем ```remove_reference```:
```cpp
template <typename T>
struct remove_reference {
  using type = T;
};

template <typename T>
struct remove_reference<T&> {
  using type = T;
};
```
При понимании механизмов выведения типов у шаблонов понять,
как работает такая конструкция, несложно.

Аналогично пишутся ```remove_pointer, remove_const,
remove_extent```(последнее позволяет убрать одно измерение у массива,
т.е. ```array[][][][] -> array[][][]```(вместо ```T&``` пишется
```T[]```)).

Также есть структура ```std::decay<T>```, торая позволяет снять с 
типа ```T``` все "навешенные штуки".

Аналогично можно написать ```add_const, add_reference, add_pointer``` и т.д.

Вспомнив variadic templates можно написать кастомный ```is_homogeneous```:
```cpp
template <typename Head, typename... Tail>
struct is_homogeneous {
  static const bool value = (std::is_same_v<Head, Tail> && ...);
};
``` 

### std::common\_type
```cpp
namespace std {

// напомним про declval
template <typename T>
std::add_rvalue_reference_t<T> declval() noexcept;

template <typename Head, typename... Tail>
struct common_type {
  using type = typename common_type<Head, typename common_type<Tail...>::type>::type;
};

template <typename T, typename U>
struct common_type<T, U> {
  using type = std::remove_reference_t<decltype(true ? declval<T>() : declval<U>())>;
};

} // std
```

https://stackoverflow.com/questions/67959239/what-is-complexity-of-stdcommon-type

### Алиасы и шаблонные переменные

Чтобы не писать каждый раз ```std::remove_reference<T>::type``` существуют подобные алиасы:
```cpp
template <typename T>
using remove_reference_t = typename remove_reference<T>::type;
```
Однако в C++17 также появились шаблонные переменные, 
которые удобно использовать для некоторых ситуаций вроде ```std::is_same```:
```cpp
template <typename T, typename U>
const bool is_same_v = is_same<T, U>::value;
```

### Dependent names

Разберёмся, почему в примере выше с ```remove_reference_t``` мы использовали слово ```typename```.

Рассмотрим несколько примеров.

1.
```cpp
template <typename T>
struct S {
  using x = T;
};

template <>
struct S<int> {
  static int x;
};

template <typename T>
void f() {
  S<T>::x * a;
}
...
f<int>();
```
Как компилятор должен распарсить выражение ```S<T>::x * a```? 

Есть два варианта: объявление указателя ```a``` на тип ```S<T>::x``` и произведение двух переменных.

По дефолту считается, что в таких неоднозначных случаях будет иметься в виду переменная. 
Чтобы указать, что это именно тип, нужно использовать ```typename```:
```cpp
typename S<T>::x* a;
```

2.
```cpp
template <typename T>
struct S {
  template <size_t N>
  struct A {};
};

template <>
struct S<int> {
  static const int A = 5;
};

template <typename T>
void g() {
  S<T>::A<10> a;
}
```
При компиляции этого кода мы получим ошибку:
```
'a' was not declared in this scope
```
Тут опять же ```S<T>::A``` парсится как переменная, а далее компилятор разбирает это выражение как
оператор меньше(```<```), 10, оператор больше(```>```) и некоторая переменная ```a```.

Однако если написать 
```cpp
typename S<T>::A<10> a;
```
мы также получим ошибку компиляции, ведь компилятор понял, что ```S<T>::A``` - это тип, 
а дальше он ожидает идентификатор(имя переменной, которую мы объявляем), а не оператор меньше.
Вот тут недостаточно сказать компилятору, что это имя типа.
Нужно указать, что это имя шаблонного типа.
Делается это вот так:
```cpp
typename S<T>::template A<10> a;
```

3.
```cpp
template <typename T>
struct B {
  int x;
};

template <typename T>
struct D : B<T> {
  void f() {
    x = 1;
  }
};
```
При компиляции мы получим
```
'x' was not declared in this scope
```
Однако понятно, что если бы наследование происходило не от шаблонного родителя, то переменная была бы видна.
Почему же с шаблонами это не работает?

Тут ```x``` - зависимое имя. 
И компилятор не будет ходить в шаблонного родителя, чтобы понять, чем конкретно является ```x```.
Потому нужно явно указывать, откуда он берётся:
```cpp
Base<T>::x = 1;
// or
this->x = 1;
```
При этом, если ```x``` нигде нет, то мы получим ошибку компиляции на этапе инстанцирования.

## SFINAE

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

### Проверка наличия метода в классе

Стоит понимать, что название не удастся реализовать метафункцию, которая будет принимать
название метода как параметр.
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

Реализуем его:
```cpp
template <bool B, typename T>
struct enable_if {};

template <typename T>
struct enable_f<true, T> {
  using type = T;
};

template <bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;
```
Т.е. в случае ```true``` мы попадём в специализацаю, в которой есть ```type```, 
а иначе его не будет и произойдёт неудачная шаблонная подстановка.

Второй шаблонный аргумент ```T``` иногда бывает нужен(пока хз зачем:( ).

## Ещё немного type trait'ов

Не все type traits можно реализовать самому, т.к. иногда нужны знания о том,
как компилятор представляет информацию(например ```std::is_union```).

### is_constructible

```cpp
template <typename T, typename... Args>
struct is_constructible {
 private:
  template <typename TT, typename... Aargs,
        typename = decltype(TT(std::declval<Aargs>()...))>
  static true_type f(int);
  
  template <typename...>
  static false_type f(...);

 public:
  static const bool type = decltype(f<T, Args...>(0))::value;
};

template <typename T, typename... Args>
bool is_constructible_v = is_constructible<T, Args...>::value;
```

### is_copy_constructible

```cpp
template <typename T>
struct is_copy_constructible {
 private:
  template <typename TT,
        typename = decltype(TT(std::declval<const TT&>()))>
  static true_type f(int);
  
  template <typename...>
  static false_type f(...);

 public:
  static const bool type = decltype(f<T>(0))::value;
};

template <typename T>
bool is_copy_constructible_v = is_copy_constructible<T>::value;
```

### is_nothrow_move_constructible

```cpp
template <typename T>
struct is_nothrow_move_constructible {
 private:
  template <typename TT,
        typename = std::enable_if_t<noexcept(TT(std::declval<TT>()))>>
  static true_type f(int);
  
  template <typename...>
  static false_type f(...);

 public:
  static const bool type = decltype(f<T>(0))::value;
};

template <typename T>
bool is_nothrow_move_constructible_v = is_nothrow_move_constructible<T>::value;
```
