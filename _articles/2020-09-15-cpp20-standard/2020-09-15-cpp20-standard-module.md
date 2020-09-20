---
# layout: page
layout: single
title: "C++模块"
title-en: "C++ Module"
author: drivextech
date: 2020-09-15 21:00:00 +0800
last_modified_at: 2020-09-20 21:00:00 +0800
categories: C++
excerpt_separator: <!--more-->
---

<!-- ## 摘要 -->

C++20标准中开始引入模块(module)的概念，用于解决传统的头文件包含机制在编译时间，工程代码组织等方面的问题，本文简要介绍C++20的模块机制及其使用说明或限制条件。

<!--more-->


## 简介

C++20标准引入模块(Module)作为现代化的C++库和程序组件化的解决方案，模块可简单的类比为头文件(Header File)+翻译单元(translation unit)，模块的源文件与导入该模块的翻译单元是独立编译的，因而编译过程中模块只需要编译一次。模块作为`#include`头文件的替代方案可带来以下几个方面的优化：

- 编译时间(Compile-time)：C++编译器编译处理源文件时，对其`#include`的头文件，编译器会预处理并解析该头文件的内容以积极递归处理该头文件中包括的其它头文件，导致当项目工程中的多个源文件包括某些相同的头文件时编译器会做大量的重复编译工作。
- 标识符隔离(Isolation)：C++预处理器通过`#include`指令把头文件的内容直接包括进当前的源文件中，使得在多个不同的头文件中的相同的标识符产生冲突，用户需要重排`#include`指令顺序或引入`#undef`指令来避免冲突。
- 重复编码(Deduplication)：使用头文件方式编写库或程序时，需要在头文件中提供声明(declaration)，而在实现源文件中提供定义(definition)，导致不必要的重复和冗余。


## 语法

**模块声明(module declaration)**：*Declares that the current translation unit is a module unit*

```c++
[export] module <module-name> [: <module-partition-name>]+ ;
```

> 特殊模块声明：
>
>```c++
>    module ; // starts a global module fragment
>    module : private ; // starts a private module fragment
>```

**导出声明(export declaration)**：*[导出模块内的变量、函数、类等]*

```c++
export <declaration>
---
export { <declaration-seq>... }
```

e.g.,

```c++
export Module M;
export int I;
export struct S;
export class C;
export void fn();
export typedef S T;
export using T = C;
export static_assert(true); // Wrong
```

**模块导入声明(module import declaration)**：*[导入模块单元(module unit)、模块分区(module partition)、头文件单元(header unit)]*

```c++
[export] import (<module-name> | <module-partition-name> | <header-file-name>)
```

e.g.,

```c++
import M;
import :P;
import M:P;
import <vector>
```

### `export`限制

- `export`不能导出具有内部链接(internal linkage)的C++实体(如静态变量或函数，定义在匿名命名空间中的变量或函数或类等)，如：

    ```c++
    export static int var = 28; // illegal, static variable
    export static void func() {} // illegal, static function

    namespace {
        export int var = 28; // illegal, variable defined in anonymous namespace
        export void func() {} // illegal, function defined in anonymous namespace
        export class cls {} // illegal, class defined in anonymous namespace
    }
    ```

- 被导出(exported)的C++实体在第一次声明(declaration)时必须是一个导出声明(exported declaration)，即声明中必须要有`export`关键字，而后续的声明或定义可以不需要`export`关键字。

    ```c++
    export class Thing;  // Good

    export class Thing;  // Okay, but redundant

    class Thing;  // Implicit `export` keyword

    class Thing { // Implicit `export` keyword
        int a;
        int b;
    };

    class SomethingElse; // Good. Not exported.

    export class SomethingElse; // Illegal! First declaration is not exported!
    ```

- 只能在命名空间作用域(namespace-scope)内(也包含全局命名空间)包括导出声明(exported declaration)，例如：

    ```c++
    export class C {
        int a;
        export int b; // Illegal
    };

    export void foo() {
        export int value = get_value(); // Illegal
    }

    template <export typename T>
    export class my_container {}; // Illegal
    ```

- `using`声明(`using`-declaration)中所指向的非内部(internal)链接或模块(module)链接的C++实体也可进行导出(exported)，但`using namespace`声明不能被导出，如：

    ```c++
    namespace Stuff {
        export class Widget {};
        class Gadget {};

        namespace {
            class Gizmo {};
        }
    }

    export using Stuff::Widget; // Okay
    export using Stuff::Gadget; // Illegal

    export using Stuff::Gizmo; // Illegal

    export using namespace Stuff; // Illegal
    ```

- 命名空间(namespace)定义(definition)或`linkage-specification block`也可以被导出，但该声明(declaration)块中所声明的(declared)C++实体都必须满足导出条件，如：

    ```c++
    export namespace foo {
        int eight = 8; // Okay. `eight` is exported as `foo::eight`.

        static int nine = 0; // Illegal

        namespace {
            void do_stuff() {} // Illegal
        }
    }
    ```

    ```c++
    export namespace foo {
        int two = 2; // Okay
    }

    namespace foo {
        static int six = 6; // Okay, as not within an exported namespace definition, even though the containing namespace `foo` itself is exported by another namespace definition
    }
    ```

### `import`限制

- `import`语句必须在任何声明语句之前，如：

    ```c++
    export module yo;

    import dogs;

    void pet(dog& d);

    import cats; // Not allowed! Move this import above `pet`
    ```

- `import`声明仅能出现在全局作用域中(global scope)，如：

    ```c++
    void foo() {
        import std; // Illegal
    }

    namespace {
        import std; // Illegal
    }
    ```

- 模块单元中的`import`不能具有对该模块单元自身的接口依赖(interface dependency)，即循环导入(cyclic import)。


## 模块单元(Module Unit)

C/C++的基本编译单元是翻译单元(translation unit)，即单个源文件和该源文件所直接或间接包括的头文件的内容(注：条件预处理指令所派排除的代码块除外)。而在C++20标准中通过模块机制引入一种新的翻译单元称为模块单元(module unit)，具体来说，模块单元就是包含模块声明的翻译单元，即在文件顶级(top level)包括有`module`声明的源文件即是模块单元。

**模块单元从接口和实现的角度可分为：**

- 模块接口单元(module interface unit)：在模块声明(`module-declaration`)中包含有`export`关键字的模块单元。
- 模块实现单元(module implementation unit)：非模块接口单元的模块单元，即在模块声明(`module-declaration`)中不包含`export`关键字的模块单元。模块实现单元中的C++定义不能从另一个文件中进行导入(importing)，除非在模块接口单元中进行声明。

**模块单元从组织结构的角度可分为：**

- (常规)模块单元(module unit)：模块声明(`module-declaration`)中不包含模块分区名(即`:<partition-name>`)的模块单元。提及模块单元时需要根据上下文来判断是否是指常规模块单元或是指概念性模块单元。
- 模块分区(module parition)：模块声明(`module-declaration`)中包含模块分区名(即`:<partition-name>`)的模块单元。

**组合这两个不同维度可把模块单元可划分为4类：**  

<!--
| | interface | implementation |
| --- | --- | --- |
| unit | module interface unit | module implementation unit |
| partition | module interface partition [unit] | module implementation partition [unit] |
-->

| | |
| --- | --- |
| 模块接口单元 <br/> module interface unit | 模块实现单元 <br/> module implementation unit |
| 模块接口分区 <br/> module interface partition | 模块实现分区 <br/> module implementation partition |

- (常规)模块接口单元(module interface unit)：即模块声明(`module-declaration`)中包含`export`关键字但不包含模块分区名的模块单元。
- (常规)模块实现单元(module implementation unit)：即模块声明(`module-declaration`)中不包含`export`关键字也不包含模块分区名的模块单元。
- 模块接口分区(module interface partition)：模块接口单元 + 模块分区，即在模块声明(`module-declaration`)中同时包含`export`关键字和模块分区名的模块单元。
- 模块实现分区(module implementation partition)：模块实现单元 + 模块分区，即在模块声明(`module-declaration`)中包含模块分区名但不包含`export`关键字的模块单元。

**主模块接口单元(primary module interface unit)：**

<!--
<div style="display:none">
The export module without a partition name is the primary interface unit, i.e., The module unit that contains `export module <module-name>` (with no partition name) is known as the primary module interface unit, and there can only be one primary module interface unit, but any number of module interface partitions.

The primary module interface unit is the non-partition module unit that is also a module interface unit. There must be exactly one primary module interface unit in a module. All other module interface units must be module partitions.

The primary interface unit for a module must export all of the interface partitions for the module (either directly or indirectly) via `export import :<part-name>`.
</div>
-->

模块接口单元分为常规模块接口单元和模块接口分区2类，C++20标准中规定模块中必须有且只能有一个常规模块接口单元，称为主模块接口单元(primary module interface unit)，其它的模块接口单元必须是模块接口分区。


### Module Fragment

- Global Module Fragment

    在C++20模块机制下，每个C++实体都须属于某个模块，这直接带来2个方面的问题：
        1. 对于现有的未采用模块机制的代码，其所对应的模块是什么？
        2. `main()`函数所归属的模块是什么？
    问题的答案是全局模块(global module)，这是一个特殊的隐式(implicit)模块，也是唯一的非命名模块(named module)，所有的未声明在命名模块(named module)中的代码都属于该模块。由于全局模块是未命名的(unnamed)，因而也不能进行`import`。

    对于非模块头文件的下面2种包括方式：

    ```c++
    // legency include preprocessor directive
    #define _UNICODE
    #include <windows.h>
    ```

    ```c++
    // `header-unit import` preprocessor directive
    #define _UNICODE
    import <windows.h>;
    ```

    对于第一种采用`#include`方式的头文件包括，尽管`_UNICODE`宏可以改变头文件`windows.h`中的条件编译，但该头文中的所有的可导出符号(exportable symbol)都会附加到相应导入模块(importing module)空间(既具有模块链接(module linkage))。而对于第二种采用`import`指令的头文件单元导入方式，`_UNICODE`宏不能影响头文件`windows.h`的条件编译。

    为解决以上问题，C++20提出采用全局模块段(global module fragment)来解决，具体的：

    ```c++
    module; // global module fragment

    #define _UNICODE
    #include <windows.h>

    module foo; // module `foo`
    // module purview... [2]
    ```

    其中位于`module;`之后，`module foo`声明之前的部分称为全局模块段(global module fragment)，声明或定义在该部分的代码属于全局模块(global module)，而不是`foo`模块。

- Private Module Fragment

    若模块中仅包括唯一的模块单元(注：则该模块单元同时也是主模块接口单元)，依据C++20标准中，该模块中可包括私有模块片段(private module fragment)。库的作者可在该私有模块片段中定义某些变量、函数、结构体、类等C++实体，而地对库的使用者来说定义在该部分的具体实现细节是不可见的，相对于传统做法上采用一个单独的C++实现文件来隐藏库的具体实现代码，通过私有模块片段来对实现细节的隐藏一方面减少源文件的数量，而一方面也有助于源码实现的连贯性和灵活性。

    下面给出采用私有模块片段实现PIMP模式的案例：

    ```c++
    module;
    #include <memory>

    export module m;
    struct Impl;

    export class S {
    public:
        S();
        ~S();
        void do_stuff();
        Impl* get() const { return impl.get(); } // from caller side, calling this function will get a pointer to incomplete type
    private:
        std::unique_ptr<Impl> impl_;
    };

    module :private; // Everything beyond this point is not available to importers of 'm'.

    struct Impl {
        void do_stuff() {}
    };

    S::S():
        impl_{ std::make_unique<Impl>() }
    {}

    S::~S() {}

    void S::do_stuff() {
        impl_->do_stuff();
    }
    ```


### Module Examples

- **imp1：模块接口(实现)单元**

    ```c++
    // mod.cppm
    export module mod;
    export const char* name() { return "m"; }
    ```

- **imp2：模块接口单元 + 模块实现单元**

    ```c++
    // mod.cppm
    export module mod;
    export const char* name();
    ```

    ```c++
    // mod_imp.cppm
    module mod;
    const char* name() { return "m"; }
    ```

- **imp3：模块接口单元 + 模块接口分区**

    ```c++
    // mod.cppm
    export module mod;
    export import :name;
    export const char* name();
    ```

    ```c++
    // mod_int_part.cppm
    export module mod:name;
    export const char* name() { return "m"; }
    ```

- **imp4：模块接口单元 + 模块实现分区**

    ```c++
    // mod.cppm
    export module mod;
    export import :name;
    export const char* name();
    ```

    ```c++
    // mod_imp_part.cppm
    module mod:name;
    const char* name() { return "m"; }
    ```

## Note

1. C++模块并非是一种库或程序的分发机制，即C++模块并不能用于实现对用户不提供头文件，也不能实现对用户隐藏定义在头文件中的C++模块实现。
2. C++模块并不是命名空间，而是独立于C++命名空间，例如，`nested::name`和`nested.name`并没有在命名空间或文件目录组织结构上的关联。
3. 一个模块必须且只能有一个主模块接口单元，主模块接口单元必须直接或间接包含该模块的所有导出声明。
    - 直接包含导出声明：`export <module-name>`
    - 简介包含导出声明：`export import <module-partition-name>`
4. 一个模块可以包括多个模块接口分区，也可以包括多个模块实现单元(模块实现单元或模块实现分区单元)。

### Refs

- [C++20](https://en.wikipedia.org/wiki/C%2B%2B20)
- [Modules / cppreference](https://en.cppreference.com/w/cpp/language/modules)
- [Modules / ModernesCpp](https://www.modernescpp.com/index.php/c-20-modules)
- [C++20: The Advantages of Modules / ModernesCpp](https://www.modernescpp.com/index.php/cpp20-modules)
- [Modules / Clang](http://clang.llvm.org/docs/Modules.html#problems-with-the-current-model)
- [Overview of modules in C++](https://docs.microsoft.com/en-us/cpp/cpp/modules-cpp?view=vs-2019)
- [Standard C++20 Modules support with MSVC in Visual Studio 2019 version 16.8](https://devblogs.microsoft.com/cppblog/standard-c20-modules-support-with-msvc-in-visual-studio-2019-version-16-8/)
- [Understanding C++ Modules: Part 1: Hello Modules, and Module Units](https://vector-of-bool.github.io/2019/03/10/modules-1.html)
- [Understanding C++ Modules: Part 2: export, import, visible, and reachable](https://vector-of-bool.github.io/2019/03/31/modules-2.html)
- [Understanding C++ Modules: Part 3: Linkage and Fragments](https://vector-of-bool.github.io/2019/10/07/modules-3.html)
- [Modules - The Beginner's Guide - Meeting C++](https://meetingcpp.com/mcpp/slides/2019/modules-the-beginners-guide-meetingcpp2019.pdf)
- [A Few Words on C++ Modules](https://medium.com/@dmitrygz/brief-article-on-c-modules-f58287a6c64)
- [What exactly are C++ modules?](https://stackoverflow.com/questions/22693950/what-exactly-are-c-modules)
- [Translation unit (programming)](https://en.wikipedia.org/wiki/Translation_unit_(programming))
- [What is a “translation unit” in C++](https://stackoverflow.com/questions/1106149/what-is-a-translation-unit-in-c)
- [C++20 Draft / module.private.frag](http://eel.is/c++draft/module.private.frag)
