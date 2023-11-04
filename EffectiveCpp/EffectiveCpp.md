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

为免除“跨编译单元之初始化次序”问题，应以 local static 对象替换 non-local static 对象。98