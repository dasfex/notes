# type\_traits

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
