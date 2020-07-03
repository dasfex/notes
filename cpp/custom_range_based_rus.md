# range based loop для своего класса

Для того, чтобы range based loop работал с нашим классом, 
необходимо, чтобы были класс имел методы
```begin```, ```end```, которые возвращают итератор.
У итератора должны быть перегружены операторы
```++```(префиксный), ```==```, ```!=```, ```*```. 

Давайте напишем свой аналог ```range``` из ```python3```:
```
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
	 
    bool operator==(const iterator& t) {
      return x == t.x;
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
```
for (auto x : range(0, 10)) {
  cout << x << ' ';
}
```
получим:
```
0 1 2 3 4 5 6 7 8 9 
```
