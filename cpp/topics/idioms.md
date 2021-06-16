## CRTP

CRTP - Curiously Recursive Template Pattern.

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

Давайте заодно разберём, как работает ```enable_shared_from_this```.

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
