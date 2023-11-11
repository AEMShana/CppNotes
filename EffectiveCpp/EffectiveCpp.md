# Effective C++ (改善程序与设计的55个具体做法)

## 条款01: 视C++为一个语言联邦

C++同时支持各种编程范式：过程式、面向对象形式、函数式、泛型形式、元编程形式。

可以将C++视为一个由相关语言组成的语言联邦而非单一语言：
- C
- Object-Oriented C++ (C with Classes)
- Template C++
- STL

## 条款02: 尽量以 const, enum, inline 替换 #define

## 条款03: 尽可能使用 const

### 指针的 const
对于指针的const，有以下4种情况：

```cpp
char greeting[] = "Hello";
char* p = greeting;                 // non-const pointer, non-const data
const char* p = greeting;           // non-const pointer, const data
char* const p = greeting;           // const pointer, non-const data
const char* const p = greeting;     // const pointer, const data
```

`*` 左边的 const 表示指针所指向的数据是常量。  
`*` 右边的 const 表示指针本身是常量。

### 迭代器的 const
```cpp
std::vector<int> vec;
const std::vector<int>::iterator iter = vec.begin();  // 迭代器本身是const
*iter = 10;                                           // OK
++iter;                                               // 错误, iter本身是const
std::vector<int>::const_iterator cIter = vec.begin(); // 迭代器所指向的数据是const
*cIter = 10;                                          // 错误
++cIter;                                              // OK
```

### 成员函数可根据 const 重载

```cpp
class App {
    int func() const {}
    int func() {}
};
```

### mutable
对于允许修改而不破坏逻辑上的常量性的成员，可以用 `mutable` 加以修饰。

```cpp
class App {
public:
    mutable int something = 233;
};

int main() {
    const App app{};
    std::cout << app.something << std::endl; // 233
    app.something = 666;
    std::cout << app.something << std::endl; // 666

    return 0;
}
```

### 当 const 和 non-const 成员函数有实质等价的实现时，令 non-const 版本调用 const 版本可避免代码重复

```cpp
class TextBook {
public:
    ...
    const char& operator[](std::size_t position) const {
        ...
        return text[position];
    }

    char& operator[](std::size_t position) {
        return const_cast<char&>(                         // 2.`然后擦除 const
            static_cast<const TextBook&>(*this)[position] // 1.调用 const operator[]
            );  
    }


};

```

## 条款04: 确定对象被使用前已被初始化

### 对内置类型对象进行手工初始化，C++并不保证初始化他们

### 使用成员初始化列表进行初始化

类成员的初始化发生在进入构造函数体之前。

```cpp
class App {
public:
    int x, y;
    App(int _x, int _y) {
        this->x = _x;       // 这是赋值
        this->y = _y;
    }
}

class App {
public:
    int x, y;
    App(int _x, int _y) : x(_x), y(_y) {} // 这是初始化
}

```

### 注意初始化顺序

C++ 有着十分固定的成员初始化次序：基类早于派生类被初始化，class 的成员变量总是以其声明次序被初始化，**即使它们在成员初始值列中以不同次序出现**。因此在书写成员初始化列表时，最好以其声明次序为次序。

```cpp
class App {
public:
    int x, y;
    App() : x(233), y(x) {}
};

int main() {
    App app{};
    std::cout << app.x << " " << app.y << std::endl;    // 233 233

    return 0;
}
```

```cpp
class App {
public:
    int x, y;
    App() : y(233), x(x) {} // 错误！
};

int main() {
    App app{};
    std::cout << app.x << " " << app.y << std::endl;    // -858993460 233

    return 0;
}
```

C++ 对定义于不同编译单元内的 non-local static 对象的初始化次序并无明确定义。

为免除“跨编译单元之初始化次序”问题，应以 local static 对象替换 non-local static 对象。

## 条款05: 了解 C++ 默默编写并调用哪些函数

C++拒绝为持有reference或const成员的类自动生成拷贝赋值运算符。

若基类将拷贝赋值运算符声明为 private，则编译器将拒绝为派生类生成拷贝赋值运算符。

## 条款06: 若不想使用编译器自动生成的函数，就该明确拒绝

```cpp
class App {
    App(const App&) = delete;
    App& operator=(const App&) = delete;
};
```

## 条款07: 为多态基类声明 virtual 析构函数

如果class带有任何virtual函数，那么它就该拥有一个virtual析构函数。

如果一个class不会被用作基类，就不要令其析构函数为virtual，因为这样class还要多保存一个指向虚函数表的指针，使类对象的体积增加。

不要继承标准容器或其他任何带有非virtual析构函数的class。

必须为纯虚析构函数提供定义。析构函数的运作方式是，最深层派生的那个class的析构函数最先被调用，然后是每一个base class的析构函数被调用，所以即使析构函数是纯虚函数，也必须为这个函数提供一个定义。如果不这样做，链接器会报错。

```cpp
class App {
public:
    int x, y;
    virtual ~App() = 0;
};

// App::~App() {} 必须为纯虚析构函数提供定义，否则报错

class App2 : public App {
public:
    virtual ~App2() {}
};

int main() {
    App2 app2{};

    return 0;
}
```

## 条款08: 别让异常逃离析构函数

析构函数绝对不要抛出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕获任何异常，然后吞下它们（不传播）或结束程序：

- 抛出异常就结束程序。
```cpp
class DBConn {
public:
    ...
    ~DBConn();
private:
    DBConnection db;
};

DBConn::~DBconn() {
    try { db.close(); }
    catch(...) {
        // ...
        // 记录下对close的调用失败
        std::abort();
    }
}
```

- 吞下异常。
```cpp
DBConn::~DBconn() {
    try { db.close(); }
    catch(...) {
        // ...
        // 记录下对close的调用失败
    }
}
```

如果客户需要对某个操作函数运行期间抛出的异常作出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作:

```cpp
class DBConn {
public:
    ...
    void close() {  // 供客户使用的新函数
        db.close();
        closed = true;
    }

    ~DBConn() {
        if (!closed) {  // 若客户没有自行关闭，则关闭连接
            try { db.close(); }
        }
        catch (...) {
            // 记录下对close的调用失败
        }
    }
private:
    DBConnection db;
    bool closed;
};
```

## 条款09: 绝不在构造和析构过程中调用 virtual 函数

在基类构造期间，virtual 函数绝不会下降到派生类层次，此时的 virtual 函数仿佛不是 virtual 函数：

```cpp
#include <iostream>

class A {
public:
    A() { func(); }

    virtual void func() { std::cout << "A::func()" << std::endl; }
};

class B : public A {
public:
    virtual void func() override { std::cout << "B::func()" << std::endl; }
};

int main() {
    B b{}; // 输出: A::func()

    return 0;
}
```

## 条款10: 令 operator= 返回一个 reference to *this

为了支持连锁赋值:
```cpp
int x, y, z;
x = y = z = 15;
// 等价于 x = (y = (z = 15));
```

赋值运算符必须返回一个 reference 指向操作符的左侧实参：
```cpp
class Widget {
public:
    Widget& operator=(const Widget& rhs) {
        return *this;
    }
};
```

## 条款11: 在 operator= 中处理自我赋值

自我赋值可能的出现情况：
```cpp
class Widget{...};
Widget w;
...
w = w;          // 赋值给自己
a[i] = a[j];    // 潜在的自我赋值
*px = *py;      // 潜在的自我赋值

class Base;
class Derived : public Base;

void do_something(const Base& rb, Derived* pd); // rb 和 pd 可能是同一对象
```

处理自我赋值一般有以下几种解决方案：
- 证同测试
```cpp
Widget& Widget::operator=(const Widget& rhs) {
    if (this == &rhs) return *this;  // 证同测试，如果是自我赋值，则不做任何事
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
- 精心考虑语句顺序，让 operator= 具备异常安全性往往自动获得自我赋值安全的回报
```cpp
Widget& Widget::operator=(const Widget& rhs) {
    Bitmap* pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;
    return *this;
}
```

- 使用 copy and swap 技术
```cpp
void swap(Widget& rhs);

Widget& Widget::operator=(const Widget& rhs) {
    Widget temp{rhs};
    swap(temp);
    return *this;
}
```

## 条款12: 复制对象时勿忘其每一个成分

copying 函数应该确保复制对象内的所有成员变量及所有基类部分。对于派生类的基类部分，应该让派生类的 copying 函数调用相应的基类函数：

```cpp
class A {
public:
    ...
    A(const A& rhs) {...}
    A& operator=(const A& rhs) {...}
};

class B : public A {
public:
    ...
    B(const B& rhs) : A(rhs) {...}
    B& operator=(const B& rhs) {
        A::operator=(rhs);
        ...
    }
};
```

不要尝试以某个 copying 函数实现另一个 copying 函数（如令拷贝构造函数调用赋值运算符，或令赋值运算符调用拷贝构造函数）。应该将共同机能放进第三个函数中，并由两个 copying 函数共同调用。

## 条款13: 以对象管理资源

为防止资源泄漏，请使用RAII对象，它们在构造函数中获得资源，并在析构函数中释放资源。

## 条款14：在资源管理类中小心 copying 行为

注意考虑RAII类的复制行为，是需要禁止复制，还是使用引用计数，还是进行深\浅拷贝，还是转移资源的所有权。

## 条款15: 在资源管理类中提供对原始资源的访问

API 往往要求访问原始资源(raw resources)，所以每一个RAII class应该提供一个取得其所管理之资源的办法。

对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对客户比较方便。因此需要根据具体情况进行取舍。

## 条款16: 成对使用 new 和 delete 时要采取相同形式

```cpp
int* x = new int{};
delete x;

int* y = new int[5] {};
delete[] y;
```

## 条款17: 以独立语句将 newed 对象置入智能指针

```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);

processWidget(std::shared_ptr<Widget>(new Widget), priority()); // 可能出现内存泄漏！
```

在调用 `processWidget` 前，需要执行以下3个步骤：
- `new Widget`
- 调用 `std::shared_ptr<Widget>` 的构造函数
- 调用 `priority`

`new Widget` 保证在调用 `std::shared_ptr<Widget>` 的构造函数前被执行，然而 C++ 并没有规定 `priority` 的执行顺序，`priority` 在`new Widget`语句之前、`new Widget` 之后、最后执行都有可能。

如果调用顺序是: 
- `new Widget`
- 调用 `priority`
- 调用 `std::shared_ptr<Widget>` 的构造函数

并且调用 `priority` 时抛出异常，那么 `new Widget` 返回的指针会遗失，造成内存泄漏。

因此以上代码应该改成:
```cpp
std::shared_ptr<Widget> pw(new Widget); // 以独立语句将 newed 对象置入智能指针
processWidget(pw, priority());
```

## 条款18: 让接口容易被正确使用，不易被误用

“促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。

“阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。

## 条款19: 设计 class 犹如设计 type

略。

## 条款20: 宁以 pass-by-reference-to-const 替换 pass-by-value

尽量以 pass-by-reference-to-const 替换 pass-by-value，前者通常比较高效，并可避免切割问题：

```cpp
class Window {
public:
    ...
    std::string name() const;
    virtual void display() const;
};

class WindowWithScrollBars: public Window {
public:
    ...
    virtual void display() const;
};

void printNameAndDisplay(Window w) { // pass-by-value
    std::cout << w.name() << std::endl;
    w.display();
}

WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb); // 把派生类赋值给基类，造成切割问题，将调用基类的 display 函数
```

对于内置类型、STL的迭代器和函数对象， pass-by-value 往往比较适当。

## 条款21: 必须返回对象时，别妄想返回其 reference

绝不要返回 pointer 或 reference 指向一个 local stack 对象，或返回 reference 指向一个 heap-allocated 对象，或返回 pointer 或 reference 指向一个 local staic 对象，而有可能同时需要多个这样的对象。

```cpp
int& foo() {
    static int value = 0;
    ++value;
    return value;
}

int main() {
    assert(foo() == foo()); // 返回的是引用自同一个static对象，断言一直成立

    return 0;
}
```

按值返回局部对象本身可以进行返回值优化 (return value optimization, RVO):

```cpp
class App {
public:
    App() {
        std::cout << "default constructor" << std::endl;
    }

    App(const App&) {
        std::cout << "copy constructor" << std::endl;
    }

    App(App&&) {
        std::cout << "move constructor" << std::endl;
    }
};

App func() {
    return App{};
}

int main() {
    App app = func();

    return 0;
}

// 输出:
// default constructor
// 只调用了一次构造函数！
```

## 条款22: 将成员变量声明为 private

将成员变量声明为 private，可赋予客户访问数据的一致性，可细微划分访问控制（如用函数实现只读、只写等）。

protected 并不比 public 更具备封装性。

## 条款23: 宁以 non-member, non-friend 替换 member 函数

对于为某个类提供便利的函数，让它成为 non-member、non-friend 函数比让它成为该类的 member 函数更具封装性、可扩展性。

```cpp
namespace WebBrowserStuff {
    class WebBrowser {
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();
        // void clearBrowser(); 不适合作为 member function
    };

    void clearBrowser(WebBrowser& wb) {
        wb.clearCache();
        wb.clearHistory();
        wb.removeCookies();
    }
}
```

## 条款24: 若所有参数皆需类型转换，请为此采用 non-member 函数

如果所有参数皆需类型转换，请为此采用 non-member 函数。

只有当参数被列于 parameter list 内，这个参数才是隐式转换的合格参与者：
```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    const Rational operator*(const Rational& rhs) const;
    int numerator() const;
    int denominator() const;
private:
    ...
};

int main() {
    Rational oneEight{ 1, 8 };
    auto result = oneEight * 2;   // OK
    result = 2 * oneEight;        // ERROR

    return 0;
}
```

使用 non-member 函数即可解决这个问题：
```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
private:
    ...
};

const Rational operator*(const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * lhs.numerator(), rhs.denominator() * rhs.denominator());
}

int main() {
    Rational oneEight{ 1, 8 };
    auto result = oneEight * 2;   // OK
    result = 2 * oneEight;          // ERROR

    return 0;
}
```

上面的例子中 `operator*` 只访问了 `Rational` 的公有部分，所以没有必要是 `Rational` 的 friend。如果可以避免 friend 函数，就应该避免，不能够只因函数不该成为 member，就自动让它成为 friend。