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
Это называется ```argument dependency lookup```.
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
