# Как создать свой манипулятор?

Предположим, мы хотим делать что-то следующее:

```
std::string s = "dasFEX";
std::cout << upper << s << std::endl;
```

И как результат получить

```
DASFEX
```

Это делается следующим способом:

```
class upper_t {};
upper_t upper;

struct upper_ostream {
  std::ostream& os;
};

upper_ostream operator<<(std::ostream& os, upper_t) {
  return { os };
}

std::ostream& operator<<(upper_ostream uos, string& s) {
  for (char c : s) {
    uos.os << c;
  }
  return uos.os;
}
```
