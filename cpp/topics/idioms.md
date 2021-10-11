## CRTP

CRTP - Curiously Recurring Template Pattern.

### enable_shared_from_this

Пусть имеется некоторый класс, из которого мы хотим возвращать ```std::shared_ptr``` на себя:
```cpp
class S {
  public:
    std::shared_ptr<S> get_ptr() const {
      return std::shared_ptr(this);
    }
};
```

Очевидно, в такой реализации будут проблемы, т.к. при вызове этого метода будет получено
несколько умных указателей, которые знают о некотором сыром указателе, но не знают
друг о друге, что приведёт к повторному удалению.

В стандартной библиотеке есть класс ```enable_shared_from_this<T>```, который 
помогает решить эту проблему.

Для начала разберёмся, как им пользоваться:
```cpp
class S : public enable_shared_from_this<S> {
  public:
    std::shared_ptr<S> get_ptr() const {
      return shared_from_this();
    }
};
```

Как видим, класс ```S``` является наследником шаблонного класса, параметризированного самим ```S```.
Это и есть CRTP.

Разберём, как работает ```enable_shared_from_this```.

Понятно, что этот класс не должен влиять на счётчик ```shared_ptr```, т.к. получилось бы,
что каждый объект ```S``` имеет указатель на самого себя.
Тогда будем хранить ```weak_ptr```!

```cpp
template <typename T>
class enable_shared_from_this {
 private:
  std::weak_ptr<T> wptr;
 
 public:
  shared_ptr<T> shared_from_this() const {
    return wptr.lock(); // если expired, то получим ub, например
  }
};
```

Существует нюанс, что это всё будет работать только в случае, если уже есть хотя бы один
```shared_ptr``` на класс.
Потому, возможно, перед вызовом ```shared_from_this``` стоит проверить, есть ли такой, 
и если нет, создать самому.

Для того, чтобы это работало, нужно в каждый конструктор ```shared_ptr``` дописать следующую конструкцию:
```cpp
// сделаем enable_shared_from_this другом shared_ptr
template <typename U>
friend class shared_ptr;
//
if constexpr (std::is_base_of_v<enable_shared_from_this<T>, T>) {
  ptr->wptr = /*...*/; // инициализация контрольным блоком, например
}
```

Ещё один пример использования это [boost::operators](https://www.boost.org/doc/libs/1_67_0/libs/utility/operators.htm).

[CRTP usecases](http://www.vishalchovatiya.com/crtp-c-examples/).

## [type erasure](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Type_Erasure)

## ADL

ADL - Argument dependent lookup(ADL).

Рассмотрим следующий код:
```cpp
namespace A {
  struct S {};
  
  void call(const S& x) {}
}
...
A::S x;
call(x);
```
Удивительно, но этот код скомпилируется, несмотря на то, что мы явно не указали
namespace функции ```call```.
Это называется ```argument dependency looking```.
Компилятор смотрит на пространства имён аргументов и в них ищет функцию.

Что же будет, если подходящих вариантов несколько?
```cpp
namespace A {
    struct S1;
}

namespace B {
    struct S2 {};
    
    void call(const A::S1& x, const S2& y) {
        cout << "1";
    }
}

namespace A {
    struct S1 {};
    
    void call(const S1& x, const B::S2& y) {
        cout << "2";
    }
}
...
A::S1 x;
B::S2 y;
call(x, y);
```
Как вариант, вызовется та версия, которая раньше будет найдена(какой аргумент раньше объявлен).
Однако получим ошибку компиляции об неоднозначности вызова:
```
error: call of overloaded ‘call(A::S1&, B::S2&)’ is ambiguous
```
Существуют также ситуации, когда ADL не работает:
```cpp
void foo();
namespace N {
  struct X {};
  void foo(X) {};
  void qux(X) {};
}

struct C {
  void foo() {}
  void bar () {
    foo(N::X{}); // будет обнаружена C::foo, которая не принимает аргументы
  }
};

void bar() {
  extern void foo(); // redeclare ::foo
  foo(N::X{}); // ::foo не требует аргументов
}

int qux;

void baz() {
  qux(N::X{}); // variable declaration disables ADL for "qux"
}
```
