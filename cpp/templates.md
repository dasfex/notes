# templates

---

## Type deduction notes

---

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

Первым(самым плохим вариантом) является использование RTTI:
```cpp
template <class T>
void f(T* const& t) {
  std::cout << typeid(t).name() << std::endl;
}
int x = 1;
f(&x); // на экране int* const
```
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

## templates notes

---

### Non-type template parameters

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