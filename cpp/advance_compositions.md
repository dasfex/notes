# Advance compositions

Тут я собираю нетривиальные композиции различных возможностей C++ 
для прокачки навыка придумывать подобное самому.

### Ввод/вывод

```cpp
std::copy(std::istream_iterator<int>(std::cin), std::istream_iterator<int>(), std::back_inserter(a));
```
аналогично следующему коду
```cpp
for (int& x : result) {
  std::cin >> x;
}
```

Аналогично:
```cpp
std::copy(results.begin(), results.end(), std::ostream_iterator<int>(std::cout, " "));
```
равно
```cpp
for (int& x : result) {
  std::cout << x << ' ';
}
```

### Частичные суммы вместе с введением данных

```cpp
std::partial_sum(std::istream_iterator<int>(std::cin), 
                 std::istream_iterator<int>(), 
                 std::back_inserter(sum));
```
аналогично:
```cpp
size_t n;
cin » n;
vector<int> a(n);
for (int& x : a) {
  cin » x;
}
vector<int> sum(n + 1);
for (int i = 1; i <= n; ++i) {
 sum[i] = sum[i - 1] + a[i - 1];
}
```

### Заполнить контейнер случайными числами

```cpp
std::random_device rd;
std::mt19937 mt(rd());
std::uniform_int_distribution<> dis(0, 100);

std::vector<int> v1(10), v2(10);
std::generate(v1.begin(), v1.end(), std::bind(dis, std::ref(mt)));
std::generate(v2.begin(), v2.end(), std::bind(dis, std::ref(mt)));
```
