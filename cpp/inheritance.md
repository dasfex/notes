# Заметки про наследование

### Видимость и доступность

При работе с переменными и функциями стоит различать понятия
"видимость" и "доступность".
Например когда мы помечаем некоторым модификатором _доступа_
переменную или функцию в классе, она не перестаёт быть видимой,
однако меняет свою доступность.

Давайте разберёмся, что при наследовании видимо(или нет), а что (не)доступно.

Рассмотрим такую ситуацию:
```cpp
struct A {
  void f(int);
};
struct B: A {
  void f(double);
};
...
B b;
b.f(1);
```
Какая ```f``` вызовется?
Вызовется из ```B```, хотя кажется, что есть другая, которая подходит лучше.
А она не видна.
```f``` из ```A``` как раз доступна, но не видима, потому что 
```f(double)``` перекрывает ```f(int)``` будто переменная из более локальной
области видимости перекрывает переменную из более глобальной.
Т.е. во время поиска имён рассматриваются сначала все имена в наследнике,
а потом в родителе.

Аналогично с полями.
Если в обоих классах мы заведём поле ```int a;``` и напишем ```b.a = 1;```, 
мы будем иметь дело с ```a``` из класса ```B```.
Хотя на самом деле класс будет хранить два разных поля с одинаковым именем
```a```, одно из них не будет видимым(но доступным).

А что делать, если мы хотим обратиться к конкретному полю или методу из наследника?
Надо явно указать область видимости:
```cpp
b.A::f(1); // f(int)
b.A::a = 2;
```

Вот этот "приём", когда мы явно указываем область видимости, называется
*qualified id*, соответственно *unqualified id* - когда мы пишем через точку, 
не указывая область видимости.

Теперь другой пример:
```cpp
struct A {
  void f(int);
  int a;
};
struct B: A {
 private:  
  void f(double);
  int a;
};
...
B b;
b.f(1);
```
Тут мы получим ошибку компиляции в силу того, что компилятор выбирает из видимых
имён наиболее подходящее, а уже потом, выбрав, смотрит, доступна ли эта функция.
Точно так же, если бы речь шла не о наследовании, а просто в классе было 2 функции:
одна от ```public int```, другая от ```private double```, - 
и в результате ```b.f(1.2)```, компилятор сначала выберет наиболее подходящую функцию,
а уже потом поймёт, что она недоступна, и бросит ошибку компиляции.
Т.е. он не будет выбирать менее подходящую, но более доступную.

Получили, что компилятор всегда сначала выбирает, что вызывать(смотрит видимость),
а уже потом проверяет, можно ли это вызвать(проверяет доступность).

### using в наследниках

А что если мы хотим, чтобы функции из класса ```А``` из  1ого примера участвовали в перегрузке
наравне с функциями из класса ```B```?

Тут можно использовать ```using```, чтобы добавить функции и поля в текущую область видимости:
```cpp
using A::f;
```

С ```C++11``` также можно делать ```using``` для конструкторов:
```cpp
struct A {
  A(int);
  A(double);
};
struct B: public A {
  using A::A;
};
```
Теперь мы умеем конструировать экземпляры ```B``` с помощью конструкторов ```A```,
причём все поля наследника будут инициализированы дефолтным значением.
> Стоит различать этот приём с делегирующими конструкторами.

### friend

Так же наследники или родители могут объявлять друг друга ```friend```, но стоит понимать, 
что это не наследуется и всё ещё работает в одну сторону.

```cpp
struct A {
  void f(int);
};
struct B: private A {
  void f(double);
};
struct C: B {
  void f() {
    B m; // так можно написать
    A g; // тут будет ошибка компиляции в силу того, что в контексте C
    // тип A означает C::A, но из-за такой цепочки наследования C::A 
    // нам недоступно, потому что у C есть часть A, 
    // которая является приватной с точки зрения C
    // если мы хотим создать объект A, надо явно указать, 
    // что мы рассматриваем не часть класса C, а глобальный тип:
    ::A g;
  }
};
```

Однако если в ```B``` написать ```friend struct C```, то теперь в ```C``` можно будет написать
```A g```, т.к. ```B``` открыл видимости для всего, что есть в себе.
Причём если написать ```friend struct C``` в ```A```, то это не поможет, потому что 
доступ перекрывает ```B```, а не ```A```.

### virtual, override

У виртуальной и перегруженной функции сигнатуры должны совпадать **полностью**.
```cpp
virtual void f() const;
void f();
```
Тут 2я функция не перегружает первую, потому что у них разные сигнатуры.
Причём самое обидное, что компилятор никак на такое не поругается.

В C++11 добавили ключевое слово ```override```, чтобы явно показать, что вы хотите
переопределить виртуальную функцию.
Причём можно не писать, но иначе компилятор подскажет, что, например, сигнатуры не совпадают.
Очень удобно.

Ещё есть слово ```final```. 
Такую функцию нельзя переопределить в наследниках.
От ```final```класса больше нельзя будет отнаследоваться.

### Использование приватного члена из родителя

```cpp
struct A {
  virtual void f() {}
};
class B: A {
 private:
  void f() override {}
};
...
B b;
A& a = b;
a.f();
```
В данном случае будет вызвана версия из ```B```, несмотря на её приватность.
Суть в том, что публичность/приватность - концепции compile-time, а выбор
версии виртуальной функции происходит в runtime(пойти по указателю в виртуальную таблицу 
и выбрать нужную версию).

### virtual destructor

Представим ситуацию:
```cpp
struct base {};
struct der : base {
  int* p;

  def(int n) {
    p = new int(n);
  }

  ~def() {
    delete p;
  }
};
```
А теперь сделаем вот так:
```cpp
base* pb = new der(5);
delete pb;
```
Тут будет утечка памяти. 
Почему же?

Ведь ```pb``` - указатель на ```base```, а что происходит, когда
мы вызываем ```delete```?
Мы вызываем деструктор у объекта, лежащего под указателем.
А деструктор чего будет вызван, ведь ```pb``` - указатель на ```base```?
И получается, что ```p``` не удаляется.

Для решения такой проблемы нужно использовать виртуальный деструктор:
```cpp
virtual ~base() {}
```
Как только в типе появляется хоть что-то виртуальное, 
тип сразу становится полиморфным.

### vtable

Давайте рассмотрим пример:
```cpp
struct A {
  void f() {}
};
struct B {
  virtual void f() {}
};
```
Сколько байт будет занимать экземпляр первой структуры?
Правильный ответ: 1(некоторый фиктивный байт).
Потому что существует требование, чтобы у двух любых объектов
отличались адреса, чего с весом в 0 байт не достичь.

Сколько байт будет занимать экземпляр второй структуры?
Внезапно: 8.
Потому что внутри хранится указатель.
Ведь внутри полиморфный объект хранит указатель на область памяти,
где хранится информация о том, что это за тип.

В случае, когда происходит вызов вида ```a.f()```, происходит
прыжок к месту исполнения данной функции.

Если ```b.f()```(где под ```b``` лежит некоторый наследник), сначала произойдёт 
прыжок в некоторое таблицу, где компилятор выяснит, какую версию функции 
```f``` вызывать, а уже потом из того места прыгает 
к фактическому месту исполнения кода.

Эта таблица, в которой для данного объекта, по какому адресу надо искать 
перегруженные функции, называется виртуальной таблицей.
Такая таблица содержится для каждого типа в единственном экземпляре.

### Виртуальное наследование

Предположим, что имеем следующую иерархию:
```
    A
   / \
  B   C
   \ /
    D
```
В таком случае в ```D``` будем иметь два экземпляра класса ```A```.

Однако такого можно избежать.

С++ позволяет построить такую иерархию наследования, чтобы 
каждый базовый класс встречался только единожды.

Если мы хотим, чтобы часть ```A``` не дублировалась, следует наследоваться
вот так:
```cpp
B : virtual A
C : virtual A
```
Не очень понятно, почему одно и то же слово используется и для функций,
и для методов, ведь из-за виртуального наследования тип полиморфным
не становится.

Что такое виртуальный предок?
Виртуальный предок - это такой предок, который создаётся где-то отдельно,
а ```B, C``` будут указывать на один и этот объект, т.е. в них
будет храниться указатель на место, где лежит экземпляр ```A```.
> Стоит понимать, что в стандарте по факту сказано, что
> при виртуальном наследовании должен хранится один экземпляр
> базового класса, а обращение к полям должно быть однозначным.
> То, как это реализовано, остаётся на усмотрение компилятора.
> Тут указан лишь возможный вариант реализации.

Виртуальное наследование может быть полезно при использовании исключений:
```cpp
struct ex1 : std::exception {
  const char* what() const; 
};

struct ex2 : std::exception {
  const char* what() const; 
};

struct ex3 : ex1, ex2 {};

int main() {
try { throw_ex3(); }
catch(std::exception const& e) { std::cout << e.what(); }
catch(...) {}
}
```
Т.к. у ```ex3``` несколько предков ```std::exception```, то компилятор
не сможет понять, к какому именно нужно привести объект ```ex3```,
потому это исключение будет отловлено в ```catch(...)```.
При виртуальном наследовании ```ex1```, ```ex2``` от ```std::exception``` 
всё исправится. 

### Inaccesible base class

Имеем вот такую иерархию:
```
A(a)
 \
  B(b)  A(a)
   \   /
     C(c)
```
Тут получится, что мы никак не сможем получить поле, которое принадлежит
```A```, от которого наследуется ```C``` напрямую, т.к. если писать:
```c.a```, то получим CE из-за неоднозначности(ambiguous).
Чтобы получить ```a``` из родителя ```B```, то можно написать ```c.B::a```, 
но к ```a``` из прямого родителя ```A``` нет никакого способа обратится
(ведь ```c.A::a``` тоже порождает неоднозначность).

Да, можно решить эту проблему с помощью указателей или 
```reinterpret_cast```, но это больше похоже на нелегальные читы.

### RTTI(runtime type information)

Существует оператор ```typeid(...)```, который для полиморфного типа
скажет, какой это тип на самом деле. 
```typeid(...)``` возвращает объект ```std::type_info```, который содержит
информацию, что на самом деле лежит под данным именем.
У него есть метод ```name()```, который вернёт строковое представление
названия типа.
Оно может не совпадать с тем именем, которое дали вы в программе, но тем не менее
это какая-то полезная информация.
