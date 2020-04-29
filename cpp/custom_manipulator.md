# How to create custom manipulator?

So, we want to do something like this:

```
std::string s = "dasFEX";
std::cout << upper << s << std::endl;
```

And get as a result:

```
DASFEX
```

Solve:

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
    uos.os << std::toupper(c);
  }
  return uos.os;
}
```
