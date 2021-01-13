# templates, type\_traits

### Non-type template parameters

Всем известно, что в качестве шаблонов можно использовать также примитивные
числовые типы: ```int```, ```size_t```, ```bool``` и т.д.(в противоположность
```double``` и ```float```).

Но малоизвестен тот факт, что также можно использовать указатели:
```cpp
template <typename T, T& Ptr>
class MyClass {};
```
Может возникнуть вопрос, где же взять указатель кроме ```nullptr``` в compile time.
Например вот:
```cpp
static const int values[] = {4, 8, 15, 16, 23, 42};
MyClass <const int, values> myClass;

static char magic = '*';
MyClass <char, &magic> myClass2;
```
Немного больше информации можно посмотреть вот 
[тут](https://stackoverflow.com/questions/65680367/using-pointer-as-a-template-parameter/65680874#comment116128380_65680874).

### Template template parameters

Можно задавать в шаблонах типы, которые сами являются шаблонными.
На примере стека:
```cpp
template <template <typename> class Container>
struct Stack {
  Container<int> cont;
};
...
Stack<std::vector> stack;
```
Но вообще ```std::stack``` реализован примерно вот так:
```cpp
template <typename T, template <typename> class Container = std::vector>
struct stack {
  Container<T> cont;
};
...
stack<int> s1;
stack<bool, std::deque> s2;
```
Обратим внимание, что до C++17 мы обязаны писать именно слово ```class```, а не ```typename```.
После уже не принципиально.

### Basic type traits

В C++11 появился заголовок \<type\_traits\> для работы
с типами на этапе компиляции.
Некоторые функции пишутся с помощью хитрых трюков и фич, но некоторые
можно написать и сейчас.

Например давайте напишем ```is_same```(проверку во время компиляции
равны ли два типа):
```
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
```
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

Рассмотрим такой код:
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

Рассмотрим ещё один пример:
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

И ещё один:
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

### Explicit instantiation

Как известно, компиляция шаблонов происходит в два этапа: 
проверка каких-то базовых моментов до подстановки типа и после подстановки.
Сама подстановка называется инстанцированием. 
Не будем рассматривать, что это такое, т.к. эта тема хорошо покрыта во множестве источников.
Отметим более редкие моменты.

Существует способ осуществить явную подстановку типа, чтобы проверить, компилируются ли все
компоненты класса(поля, методы) без их явного вызова.
```cpp
template <typename T, size_T N>
struct S {
  void f() {
    T a[N];
  }
};

// explicit instantiation
template struct S<int, -1>;
```
Обычно мы получили бы ошибку только при инстанцировании структуры, но так этого можно не делать.
А вот как проинстанцировать методы?
