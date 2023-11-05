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

