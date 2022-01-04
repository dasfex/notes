# range based loop для своего класса

Для того, чтобы range based loop работал с нашим классом, 
необходимо, чтобы класс имел методы
```begin```, ```end```, которые возвращают итератор.
У итератора должны быть перегружены операторы
```++```(префиксный), ```!=```, ```*```. 

Давайте напишем свой аналог ```range``` из ```python3```:
```cpp
class range {
 public:
  range(int l, int r) : l(l), r(r) {}
 
  class iterator {
   public:
    iterator(int x) : x(x) {}
	 
    iterator operator++() {
      ++x;
      return *this;
    }
	 
    bool operator!=(const iterator& t) {
      return x != t.x;
    }
	 
    int operator*() const {
      return x;
    }
	 
   private:
    int x;
  };
	 
  iterator begin() const {
    return {l};
  }
	 
  iterator end() const {
    return {r};
  }
 
 private:
  int l;
  int r;
};
```
При запуске:
```cpp
for (auto x : range(0, 10)) {
  cout << x << ' ';
}
```
получим:
```
0 1 2 3 4 5 6 7 8 9 
```
Кстати с C++20 появляется возможность использовать
range based loop с инициализацией.

[Документация](https://en.cppreference.com/w/cpp/language/range-for).
