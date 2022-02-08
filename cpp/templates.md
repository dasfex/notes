# templates

## Type deduction notes

### User-defined deduction guides(since C++17)

В C++17 появилась возможность автовывода шаблонных типов, когда это возможно:
```cpp
std::vector v{1, 2, 3};
```
Рассмотрим пример:
```cpp
template <typename T>
struct S {
  S(T x) {}
};
...
S s("abs");
```
Тут тип ```T``` будет выведен как ```const char*```, но можно попросить компилятор
в таких случаях выводить его как ```std::string```.
Для этого после класса пишем:
```cpp
S(const char*) -> S<std::string>;
```
Можно также писать шаблонные подсказки:
```cpp
template <typename T>
S(const T&) -> S<T&>;
```

Или например следующий код:
```cpp
template <class T>
struct container {
  template <class Iter>
  container(Iter beg, Iter end);
};

template <class Iter>
container(Iter beg, Iter end) ->
containter<typename std::iterator_traits<Iter>::value_type>;

std::vector<double> v;
auto d = container(v.begin(), v.end()); // container<double>
```

### decltype

Приоритет для ```decltype``` это точный тип параметра:
```cpp
const int& x = 1;
decltype(x) y = 1; // -> const int& y
```
А что же тут?
```cpp
struct Point { int x, y; };
Point pp{1, 2};
const Point& p = pp;
decltype(p.x) x = 0; // int x or const int& x?
```
Получим вот такой результат:
```cpp
decltype(p.x) x = 0; // int x
decltype((p.x)) x = 0; // const int&
```
Важный кейс: если в ```decltype(expr)``` оказывается, что ```expr``` -
lvalue, то ```decltype(expr)``` добавляет lvalue reference 
к выведенному типу:
```cpp
decltype((pp)) x = pp; // Point& x = pp;
```
или
```cpp
int x[10];
x[5] = 4; // x[5] ведёт себя как ссылка
decltype(x[5]) y = x[5]; // int& y = x[5];
y = 4; // изменяет x[5]
```

### decltype(auto)

Вывод типов является точным, но при этом выводится из всей правой части:
```cpp
double x = 1.0;
decltype(x) tmp = x; // вместо повторения x
decltype(auto) tmp = x; // perfect!
```
Также стоит принимать во внимание:
```cpp
decltype(auto) tmp = x; // double
decltype(auto) tmp = (x); // double&
```

### Проблема гетерогенного минимума

Рассмотрим проблему примерно из 2012 года: мы не имеем ```auto``` 
для возвращаемого типа функций:
```cpp
template <class T>
auto /// Fail!
doWork(const T& builder) {
  auto val = builder.makeObject();
  // some code
  return val;
}
```
Заменить ```auto``` на ```decltype(builder.makeObject())```
не увенчается успехом, т.к. в этом месте ещё не существует никакого
```builder```. На помощь приходит ```null pointer dereference```:
```cpp
template <class T>
decltype(((*T)(0))->makeObject()) // but pain...
doWork(const T& builder) {
  auto val = builder.makeObject();
  // some code
  return val;
}
```
Т.к. выражение внутри ```decltype``` не вычисляется, всё будет корректно.

Чтобы облегчить жизнь программистам был введён ```trailing```:
```cpp
template <class T>
auto doWork(const T& builder) ->
    decltype(builder.makeObject()) {
  auto val = builder.makeObject();
  // some code
  return val;
}
```

Теперь взглянем на следующий код:
```cpp
template <class T, class S>
auto min(T x, S y) -> decltype(x < y ? x : y) {
  return x < y ? x : y;
}
```
Т.к. тернарный оператор возвращает lvalue, то в случае
совпадения типов возвращаемый тип будет ```type&```.
Учитывая, что параметры - копии, получим dangling reference.

Потому начиная с C++14 такая проблема решается просто: 
раз правила ```decltype``` не подходят, то подойдут
правила для ```auto```, т.е. все cv-квалификаторы отбросятся.

Хотя конечно и такое иногда ломается:
```cpp
auto bad_sum(int i) {
  return (i > 2) ? bad_sum(i - 1) + i : i;
}
```
Тут ```auto``` не сможет отработать, 
ведь до выведения типа он уже был использован.
Ошибка так и будет звучать: use before deduction.
Если поменять выражения в операторе местами, (по идее)
всё должно скомпилироваться, т.к. станет понятно, 
какой возвращаемый тип;

### Более точный вывод для C++14

Стоит помнить, что ```auto``` режет типы:
```cpp
string f1();
string& f2();

auto d1() { return f1(); } // -> string d1();
auto d2() { return f2(); } // -> string d2();
```
Тут можно использовать ```decltype(auto)```:
```cpp
decltype(auto) d1() { return f1(); } // -> string d1()
decltype(auto) d2() { return f2(); } // -> string& d2()
```

### Исследование выведенных типов

Существует оператор ```typeid(...)```, который для полиморфного типа
скажет, какой это тип на самом деле.
```typeid(...)``` возвращает объект ```std::type_info```, который содержит
информацию, что на самом деле лежит под данным именем.
У него есть метод ```name()```, который вернёт строковое представление
названия типа.
Оно может не совпадать с тем именем, которое дали вы в программе, но тем не менее
это какая-то полезная информация.

Первый (самым плохой) вариант:
```cpp
template <class T>
void f(T* const& t) {
  std::cout << typeid(t).name() << std::endl;
}
int x = 1;
f(&x); // на экране int* const
```
Получаем манглированное имя, которое можно обработать с помощью
```c++filt -t``` и получить результат. 
Минусы: код должен исполнится, часть информации обрезается.

Используем линкер:
```cpp
template <class T>
void f(T* const& t);
int x = 1;
f(&x); // error: undefined reference to `void foo<int>(int* const&)`
```
Однако это работает лишь для объектов с external linkage. 
А если хочется для локальной переменной?
```cpp
template <class T>
struct Type;
template <class T>
void f(T* const& t) {
  Type<T> t1; // error: `Type<int> t1` has incomplete type
  Type<t> t2; // error: `Type<int* const&> t2` has incomplete type  
}
```
Раз мы хотим вызвать ошибку компиляции, какая разница, 
что мы подставим в шаблон :)

Однако использовать такой способ для выяснения overkill.

Приемлемым способ может быть решение с помощью ```boost```:
```cpp
#include <boost/type_index.hpp>
using boost::typeindex::type_id_with_cvr;
std::cout << type_id_with_cvr<decltype(t)>().pretty_name();
```
Эта функция не отбрасывает (что понятно по названию) cv-квалификаторы и ссылку,
как это делает стандартный ```typeid```.

И всё же, если не хочется тянуть ```boost```, то можно применить следующий способ:
```cpp
template <class T>
class Type {};

template <class T>
void f(const T& t) {
  Type<decltype(t)> tt;
  std::cout << typeid(tt).name(); // c++filt -t
}
```
Шаблон вынуждает ничего не резать. 

Если требуется только проверка типов на равенство, можно сделать так:
```cpp
constexpr const int i = 0;
constexpr int j = 0;
using T = const int;
using T = decltype(i);
using T = decltype(j);
```
Если скомпилируется, то типы равны, иначе получаем redefenition error.

## templates notes

### Non-type template parameters(nntp)

Всем известно, что в качестве шаблонов можно использовать также примитивные
числовые типы: ```int```, ```size_t```, ```bool``` и т.д.(в противоположность
```double``` и ```float```).

Но малоизвестен тот факт, что также можно использовать указатели:
```cpp
template <typename T, T* Ptr>
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
Это может быть полезно, если вам нужно дать имя своему классу. 

Можно так же передать указатель на функцию:
```cpp
template <void (*callback_)()>
struct CBack {
  void use() {
    callback_();
  }
};

CBack<&foo> c;
c.use();
```

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
> ##### Ускорение адаптеров
>
> Известный факт, что ```std::stack, std::queue, std::priority_queue``` 
> являются адаптерами и по дефолту реализованы на деке.
> Однако часто можно использовать ```std::vector``` 
> и получить прирост эффективности. Например:
> ```cpp
> std::stack<int, std::vector<int>> st;
> ```

Обратим внимание, что до C++17 мы обязаны писать именно слово ```class```, а не ```typename```.
После уже не принципиально.

Понятно, что растить такие шаблоны можно как в ширину:
```cpp
template <template <typename, typename> class T>
```
так и в глубину:
```cpp
template <template <template <typename> class S> class T>
```

### Explicit instantiation

Как известно, компиляция шаблонов происходит в два этапа: 
проверка каких-то базовых моментов до подстановки типа и после подстановки.
Сама подстановка называется инстанцированием. 
Не будем рассматривать, что это такое, т.к. эта тема хорошо покрыта во множестве источников.
Отметим более редкие моменты.

Существует способ осуществить явную подстановку типа, чтобы проверить, компилируются ли все
компоненты класса(поля, методы) без их явного вызова.
```cpp
template <typename T, size_t N>
struct S {
  void f() {
    T a[N];
  }
};

// explicit instantiation
template struct S<int, -1>;
```
Обычно мы получили бы ошибку только при инстанцировании структуры, но так этого можно не делать.
  
Интересно, что более частый способ использования шаблонов:
```cpp
A<double> a; // implicit instantiation  
```
не обязывает компилятор инстанцировать тип. 
В данном случае необходимо, чтобы у переменной ```a``` был
полный тип, потому инстанцирование произойдёт, но в общем случае
не обязано. 

### Variadic templates(since C++11)

Важным шагом было введение шаблонов с переменным количеством аргументов:
```cpp
// отделяем первый аргумент от остальных
template <typename Head, typename... Args>
void print(const Head& head, const Args&... args) {
  std::cout << head << ' ';
  print(args...);
}
// делаем перегрузку на случай пустого пакета
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
  static const bool value = std::is_same_v<First, Second> && 
          is_homogeneous<Second, Tail...>::value;
};

template <typename T, typename U>
struct is_homogeneous<T, U> {
  static const bool value = std::is_same_v<T, U>;
};
```
Что делать, если мы хотим узнать кол-во типов в пакете?
Для этого существует специальный оператор:
```cpp
sizeof...(Args);
sizeof...(args);
```

### Fold expressions(since C++17)

> До C++17: [link](https://articles.emptycrate.com/2016/05/14/folds_in_cpp11_ish.html).

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

Ещё вот такой красивый пример:
```cpp
template <class... Args, int... N>
void g(Args (&...args)[N]) {}
```
Тут по троеточию раскрывается как пакет с аргументами, так и пакет с размерами,
потому как аргументы мы получим функцию, принимающую массивы разных размеров
от разных типов. 

Бывают совсем экзотические свёртки. 
Например пусть мы имеем бинарное дерево:
```
         1
      /     \
     2       6
   /  \       \
  3    5       8
      /      /   \
     4      7     9
```
И следующий код:
```cpp
template <class T>
struct Node {
  T data;
  Node* left;
  Node* right;
};
Node* top = getTop();
Node* seven = tree_get(top, right, right, left); // 7
Node* four = tree_get(top, left, right, left); // 4
```
Интересно, что такая свёртка пишется всего в одну строчку:
```cpp
template<class T, class... Args>
Node<T>* tree_get(Node<T>* top, Args... args) {
    return (top ->* ... ->* args);
}

Node<int> t;
auto left = &Node<int>::left;
auto right = &Node<int>::right;
```

Вообще variadic templates часто применяются при emplace-операциях 
в различных контейнерах. 

### Управление инстанцированием шаблона функций

Предположим мы имеем некоторый шаблон функции в хедере, который включаем в 2 файла,
которые используют этот шаблон функции с одним и тем же типом шаблона. 
В таком случае инстанцирование шаблона произойдёт в каждом использующем файле. 
А что делать, если у нас не одна функция, а 500. 
И файлов, использующих каждую, по 100.
Мы очень сильно увеличим время компиляции и размер конечного объектного файла. 
Что можно сделать?

Существует синтаксис для запрета инстанцирования шаблона функции конкретным типом(например ```int```):
```cpp
extern template int max<int, int>(int, int);
```

Напишем это сразу после определения шаблона функции в хедере. 
Соответственно в каждом исходнике, где используется хедер, инстанцирование будет запрещено. 
И заведём ещё один исходник, в котором явно инстанцируем шаблон:
```cpp
template int max<int, int>(int, int);
```
Теперь мы получили довольно симметричное для каждой исходника решение,
которое позволяет использовать инстанцированный шаблон функции. 
Особенно удобно, что не нужно никаких дополнительных действий при создании
новых файлов. 

### Проблемы специализации

```cpp
template <class T> My {};
My<int> my;
template <> My<int> {}; // compile error
```
Пример довольно странный, но тем не менее, последняя строка не скомпилируется в силу того, что 
для ```int``` шаблон уже инстанцирован. 

Рассмотрим такой код:
```cpp
template <class T> void f(T);
template <class T> void f(T*);
template <> void f(int*); // компилятор имеет достаточно информации, 
                          // чтобы правильно вывести тип шаблона

int x;
f(&x);
```
Очевидно, что выберется третий вариант функции ```f```, т.к. сначала выберется более частный случай(```T*```),
а потом его специализация. 

Но что если расположить следующим образом:
```cpp
template <class T> void f(T);
template <> void f(int*);
template <class T> void f(T*);
```
Тут специализация уже выбрала, что она специализирует первый шаблон, потому из двух шаблонов
будет выбран более частный, а потом выбор среди их перегрузок. 
Потому выберется 3 вариант. 

Рассмотрим ещё такой пример:
```cpp
template <class T, class U> void f(T, U);
template <class T, class U> void f(T*, U*);
template <> void f<int*, int*>(int*, int*);

int x;
f(&x, &x);
```
Будет вызвана функция с номером 2. 
3я является специализацией первого(т.к. типы в шаблоне и параметрах совпадают), 
а во время выбора шаблона предпочтение будет отдано второму. 

Специализации, как и перегрузки, можно явно запрещать:
```cpp
template <> void f<char>() = delete;
```

### template using

Можно писать вот так:
```cpp
template <typename T>
using mi = map<int, T>;
mi<string> map_from_int_to_string;
```
  
## Metaprogramming notes

### Заметка о сложности вычислений на компиляции

Интересно, что вычисления на компиляции в силу механизма работы шаблонов 
часто работают, как будто кешируя результат. 
Рассмотрим следующий код:
```cpp
// runtime
int fib(int n) {
  return n < 2 ? 1 : fib(n - 1) + fib(n - 2);
}
             
// compile time
template <int N>
struct Fib {
  static constexpr int value = Fib<N - 1>::value + Fib<N - 2>::value;
};
template <>
struct Fib<1> {
  static constexpr int value = 1;
};
template <>
struct Fib<0> : Fib<1> {
};
```
В первом случае каждый вызов функции будет приводить к вычислению,
что приводит к медленному вычислению результата. 

Во втором же, в силу того, что при наличии инстанцированного шаблона
повторное инстанцирование происходить не будет, возьмётся сразу вычисленный
результат, что сокращает время работы от O(n!) до O(n). 
  
  
## Ссылки
  
1. [Boost MPL](https://www.boost.org/doc/libs/1_63_0/libs/mpl/doc/index.html). 
2. [Boost fusion](https://www.boost.org/doc/libs/1_78_0/libs/fusion/doc/html/index.html). 
3. [Boost Hana](https://www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html).
4. CppCon 2017: Arthur O'Dwyer. 
[“A Soupçon of SFINAE”](
https://www.youtube.com/watch?v=ybaE9qlhHvw).
5. CppCon 2018. Walter E. Brown.
[“C++ Function Templates: How Do They Really Work?”](
https://www.youtube.com/watch?v=NIDEjY5ywqU&t=1960s).
6. CppCon 2021. Jason Turner.
[Your New Mental Model of constexpr](
https://www.youtube.com/watch?v=MdrfPSUtMVM). 
7. CppCon 2021. Jody Hagins.
[Template Metaprogramming: Practical Application](
https://www.youtube.com/watch?v=4YC6_77-iEY).
