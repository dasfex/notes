# new, delete

## new

### Общая форма оператора new

У оператора ```new``` есть несколько версий. 
Стандартная выглядит вот так:
```cpp
T* p = new T(x, y, z); // i.e. some args
```

В таком виде он выделяет память в куче
(запрашивает у операционной системы память, а именно столько, 
сколько нужно для хранения одного объекта типа ```T```(```sizeof(T)```)), 
вызывает конструктор этого типа от аргументов на той памяти, 
которую дали, и возвращает указатель на эту самую память.

Однако иногда нам не нужно выделять память, а только вызвать конструктор, а иногда наоборот. 
Но обычный ```new``` в стандартной форме делает обе эти вещи. 
Причём вторую часть нельзя никак переопределить.

А что ещё может делать оператор ```new``` кроме этого?

На самом деле у него есть ещё другие синтаксические формы.

Наиболее полезной после стандартной является ```placement new```.

Чтобы понять, что это такое, давайте обратимся к примеру ```std::vector```.
Каким образом в нём работает ```push_back```? 

Пусть к нам приходит новый элемент и у нас переполняется буфер. 
Мы перевыделяем в 2 раза больше памяти, 
перемещаем уже имеющиеся элементы и конструируем на новом месте новый элемент. 
А что значит это "конструируем на новом месте новый элемент"? 
Т.е. по факту нам по указателю нового элемента нужно вызвать конструктор. 

Для этого существует другая форма оператора ```new``` - ```placement new```. 

```cpp
new(p) T(x, y, z); // type(p) = T* или reinterpret_cast<T*>
```

> Имеем ```arr``` - начало выделенной памяти, ```sz``` - размер текущего вектора.
> 
> В простейшем варианте ```push_back``` может выглядеть так:
> ```cpp
> new(arr + sz) T(x, y, z);
> ```

Чем это плохо?

Если бы контейнеры напрямую вызывали ```new```, то были бы проблемы, например, на серверах, 
где системная память распределяется по-особому.

Это две основных формы ```new```. 
Но на самом деле этот оператор может быть с любыми параметрами.

### Перегрузка new

Как уже говорилось выше, ```new``` состоит из 2х частей: выделение памяти и вызов конструктора.
На самом деле любая форма оператора ```new``` делает эти две вещи, просто бывает так, 
что первая часть переопределена таким образом, что ничего не происходит. 

Любая форма оператора ```new``` по стандарту вызывает глобальную функцию 
```operator new``` с параметрами, которые дали, и на месте, 
которое возвращает эта глобальная функция, вызывает конструктор. 
Причём 2я часть никак не подлежит переопределению. 

Т.е. теперь можно понять различие между стандартной формой и ```placement new```: 
первая вызывает глобальную функцию с единственным параметром количества байт, 
а вторая форма с двумя - количество байт и указатель на место для вызова конструктора. 
И 2й определён так, что он ничего не делает, 
в то время как 1й вариант вызывает некоторый системный вызов для выделения памяти.

Таким образом, когда речь идёт о перегрузке ```new``` надо понимать, 
что понимается под этим перегрузка функции с названием ```operator new```, 
и это не то же самое, что оператор ```new```. 
1е лишь часть 2ого. 
И перегрузить можно лишь функцию, но не весь оператор. 

Т.е. когда мы перегружаем оператор, на самом деле можем перегрузить глобальную функцию:
```cpp
void* operator new(size_t n) {
    std::clog << n << " bytes allocated" << std::endl;
    ...some code...
} // например хотим логировать все динамические выделения в программе
```

Стоит помнить, что внутри перегрузки нельзя вызывать оператор ```new```, иначе попадём в рекурсию.

Или например перегрузка ```placement new```:
```cpp
template <typename T>
T* operator new(size_t n, T* p) noexcept {
    return p; // именно так выглядит стандартная реализация такой формы
}
```

Как пример нестандартного ```new``` есть ```nothrow new```(```new```, который не бросает исключение):
```cpp
new(std::nothrow) T(x, y, z);
// реализация примерно такая
namespace std {
struct nothrow_t {} nothrow; // фиктивный тип
}
void* operator new(size_t n, std::nothrow_t nothrow) {
    ...
}
```

Если у него не получилось выделить память, он вернёт ```nullptr```
(конструктор соответственно не вызовется). 
Обычный ```new``` бросает ```std::bad_alloc```.

Ещё можно переопределить ```new``` внутри класса. 
Тогда все вызовы будут изменены только для конкретного класса. 
Ещё можно переопределить ```new []```
(тут параметр ```n``` не количество байт, а количество объектов типа ```T```).

## delete

Тут всё примерно также.

Оператор ```delete``` сначала вызывает деструктор, 
а потом вызывает глобальную функцию ```operator delete```. 
Первую часть переопределить нельзя, вторую можно.

> Однако в C++20 появилась версия ```delete```, которая не вызывает деструктор.
> 
> Например мы хотим запретить создавать экземпляры класса на стеке(т.е. только с помощию ```new```): 
> делаем деструктор приватным, а ```delete``` перегружаем так, 
> чтобы он вызывал некоторую публичную функцию, которая вызывает деструктор.

```cpp
void operator delete(void* p, size_t n) { // n - размер одного объекта
   ...clear memory...
}
```

Аналогично можно определять ```delete``` с пользовательскими параметрами или ```delete []```:
```
void operator delete[] (void* p, size_t n) { // n - некоторое число, которое запомнил компилятор
   ...clear memory...
}
```

> Замечения:
> 
> 1. Пусть есть два класса ```Base``` и ```Derived```(2й наследуется от 1ого). 
> Причём у каждого есть своя версия ```delete```. И мы пишем:
> ```cpp
> Base* p = new Derived();
> delete p;
> ```
> 
> Для того, чтобы был вызван правильный деструктор, нужно, чтобы он был виртуальным. 
> А что с оператором ```delete```?
> В стандарте это решено так: если есть виртуальный деструктор, 
> то вызывается и верная версия ```delete```.
> 
> 2. Если мы пишем кастомный оператор ```delete```, его нельзя вызывать ```delete(...) p```, 
> потому если мы написали собственный оператор ```delete```, 
> то вызывать придётся явно вызывать функцию освобождения памяти, 
> а перед этим явно вызвать деструктор: ```operator delete(...) p```.

Также стоит учитывать, что если мы пишем ```new``` со своими параметрами, 
то надо написать и ```delete``` с аналогичными, потому что в ситуации, когда
в вашем ```new``` конструктор выбросит исключение, компилятор непременно 
попытается вызвать ```delete``` с такими же параметрами.

## Нехватка памяти

Есть такая функция(это не её конкретное название, а скорее общее назначение) ```new_handler```
(обработчик ```new```).
Это функция, которая вызывается, если ```new``` не смог выделить память. 
Т.е. ```new``` не сразу бросает ```std::bad_alloc```.
Оператор сначала вызывает стандартный обработчик, который в свою очередь бросает исключение.

Для работы с обработчиком есть функции ```std::get_new_handler``` и ```std::set_new_handler```.
Т.е. подразумевается, что ```new_handler``` содержит некоторый код, 
который возможно поможет с выделением.
Причём оператор ```new``` не просто вызывает эту функцию, 
а вызывает её в цикле, пока что-то не произойдёт.