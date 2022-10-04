- [enum class](#enum-class)
- [try-catch-finally和return的执行顺](#try-catch-finally和return的执行顺)
# enum class
在C++中，变量名字仅仅在一个作用域内生效，出了大括号作用域，那么变量名就不再生效了。但是传统C++的enum却特殊，只要有作用域包含这个枚举类型，那么在这个作用域内这个枚举的变量名就生效了。即枚举量的名字泄露到了包含这个枚举类型的作用域内。在这个作用域内就不能有其他实体取相同的名字。
```cpp
enum Color{black,white,red};	//black、white、red作用域和color作用域相同
auto white = false;	//错误，white已经被声明过了
```
C++11中新增了枚举类，也称作限定作用域的枚举类。  
使用枚举类的第一个优势就是为了解决传统枚举中作用域泄露的问题。在其他地方使用枚举中的变量就要声明命名空间。
```cpp
enum class Color{black,white,red}; //black、white、red作用域仅在大括号内生效
auto white = false;		//正确，这个white并不是Color中的white
Color c = white;	//错误，在作用域范围内没有white这个枚举量
Color c = Color::white;	//正确
auto c = Color::white;	//正确
```
# try-catch-finally和return的执行顺
任何执行try 或者catch中的return语句之前，都会先执行finally语句，如果finally存在的话。

如果finally中有return语句，那么程序就return了，所以finally中的return是一定会被return的。