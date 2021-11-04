# templates

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
template <typename T, template <typename> class Container = std::deque>
struct stack {
  Container<T> cont;
};
...
stack<int> s1;
stack<bool, std::vector> s2;
```
Обратим внимание, что до C++17 мы обязаны писать именно слово ```class```, а не ```typename```.
После уже не принципиально.

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

### Variadic templates(since C++11)

Следующим логичным шагом было бы введение шаблонов с переменным количеством аргументов:
```cpp
// отделяем первый аргумент от остальных
template <typename Head, typename... Args>
void print(const Head& head, const Args&... args) {
  std::cout << head << ' ';
  print(args...);
}
// делаем перегрузку на случай пустого пакетв
void print() {}
```
```Args``` - это некоторая абстрактная сущность, обозначащая список типов.
Над ним можно делать распаковку с помощью троеточия, т.е. компилятор развернёт
аргументы функции в ```const T1&, constT2&``` и т.д.
А ещё мы можем давать ему имя(например ```args```).

Напишем ещё один type_trait ```is_homogeneous```(равны ли все типы):
```cpp
template <typename First, typename Second, typename... Tail>
struct is_homogeneous {
  static const bool value = std::is_same_v<First, Second> && is_homogeneous<Second, Tail...>::value;
};

template <typename T, typename U>
struct is_homogeneous<T, U> {
  static const bool value = std::is_same_v<T, U>;
};
```
Что делаеть, если мы хотим узнать кол-во типов в пакете?
Для этого существует специальный оператор:
```cpp
sizeof...(Args);
sizeof...(args);
```
### Fold expressions(since C++17)

> Как это делаллось до C++17: [link](https://articles.emptycrate.com/2016/05/14/folds_in_cpp11_ish.html).

Это возможность языка сделать что-то для всех элементов пакета сразу
(при этом не нужно писать рекурсию).
```cpp
template <typename... Args>
void print(const Args&... args) {
  (std::cout << ... << args) << '\n';
}
```
По факту мы можем писать вот так:
```
(... binary_operator args)
```
и это превратится в
```
(args1 binary_operator args2 binary_operator ...)
```
Существует 4 формы fold expressions:
```
(... op args) -> (((args1 op args2) op args3) ... )
(args op ...) (args1 op (args2 op ... (argsn)...))
(x op ... op args)
(args op ... op x)
```
где ```x``` - некоторый аргумент. 

### Управление инстанцированием

В случае, когда некоторая функция является очень тяжёлой, можно явно её инстанцировать 
и запрещать в единицах трансляции:
```cpp
extern template int max<int, int>(int, int); // запретили

template int max<int, int>(int, int); // инстанцировали
```
/////// ПРИМЕР
