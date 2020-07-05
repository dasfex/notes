### Деление

Давайте рассмотрим, как работает деление.
```
>>> print(3 / 2)
1.5
>>> print(-3 / 2)
-1.5
>>> print(3 // 2)
1
>>> print(-3 // 2)
-2
```
Три первых результата дают именно то, что мы и ожидаем.
Однако последний пример немного необычен? 
Почему же ```3 // 2 = 1```, а ```-3 // 2 = -2```?

Ответ в том, что python всегда округляет результат деления чисел
типа ```integer``` в сторону минус бесконечности(т.е. к самому
маленькому возможному отрицательному числу), 
а ```-2``` как раз ближайшее меньшее для ```-1.5```.

### for loop with else

```for loop``` может иметь опциональный ```else``` блок.
```else``` блок запустится тогда и только тогда, когда все элементы 
заданной последовательности будут обработаны.

Давайте рассмотрим пример:
```
print('Only print code if all iterations completed')
num = int(input('Enter a number to check for: '))
for i in range(0, 6):
    if i == num:
        break
    print(i, ' ', end='')
else:
    print()
    print('All iterations successful')
```

Примеры запуска с разными входными данными:
```
Only print code if all iterations completed
Enter a number to check for: 7
0 1 2 3 4 5
All iterations successful
```
```
Only print code if all iterations completed
Enter a number to check for: 3
0 1 2
```
