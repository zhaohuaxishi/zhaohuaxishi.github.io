title: ISO C 和 ISO C++ 之间的不兼容性
tags:
 - C/CPP
 - 翻译
---

**本文是[Incompatibilities Between ISO C and ISO
C++](http://david.tribble.com/text/cdiffs.htm)一文的翻译，文章的版权归作者所有**

# 目录

<H1 id="introducation">引言</H1>

`C`语言大概在1985年的某个时候由`ANSI X3J9`委员会开始进行标准化。经过多年的努力之
后，1989年`ANSI`颁布了新的标准。一年之后（1990），一个`ISO`委员会在添加了一个处
理国际化问题的修正案之后批准了这一标准。1989年的C标准官方名称是`ANSI/ISO
9899-1989, Programming Languages - C`，这份文档中把1989年的C标准称为`C89`。
1990年的ISO标准修订本的官方名称是`ISO/IEC 9899-1990, Programming Languages - C`
，这份文档中把该标准称为`C90`。

在1999年，ISO批准了C的下一个标准。新标准的官方名称是`ISO/IEC 9899-1999,
Programming Languages - C`，在该文档中它被称为`C99`

在`ANSI C`标准化工作开始后不久，基于c程序设计语言的C++程序设计语言出现了。大概在
1995，一个ISO委员会被通知要标准化C++，并且在1998年批准了新的C++标准。该标准的官
方名称是`ISO/IEC 14882-1998, Programming Languages - C++`，它在这份文档中被称为
`C++98`或者简单的称为`C++`。

虽然这两门语言有相同的发展历程，而且参与标准化过程的语言设计者们已经尽力让让这两
种语言互相兼容，它们之间还是无可避免的产生了一些不兼容性。一旦程序员了解了这些可
能出错的地方，他们可以在大多数的C代码中避免它们。

当我们说C在某个语言特性上和C++不兼容的时候，我们想要表达的是使用了该特性的C代码
要么不是合法的C++代码所以无法编译为C++程序，要么就是可以编译为一个C++程序但是它
的行为和对于的C版本不同。换句话说，不兼容的C特性指的是在C中合法但在C++中不合法的
代码。避免这一类的不兼容性是这份文档主要想要处理的问题。避免这种不兼容性可以让程
序员正确的写出那种能与C++代码交互或者能编译成C++代码的C代码。

另一类不兼容性的语言特性是那种在C++程序中合法但是在C程序中不合法的特性。我们把它
们称为不兼容的C++特性。c++语言中的大部分内容都落入此类（比如：类、模板、异常、引
用、成员函数、匿名联合体等等），所以该文档只是处理了部分这类的兼容性。

还有一种不兼容性的语言特性是C++使用了C90中的某个同名特性但是在C中有不同的使用方
式或含义。这份文档也处理这一类的不兼容性。

这份文档只列举了C99和C++98之间的不兼容性。（C90和C++之间的不兼容性已经在其他地方
进行了归档；比如 Strostrup [STR] 的 Appendix B）

这份文档不处理C99中新增加的特性，除非它引入了C++不兼容性。

<H1 id="cppvsc">C++ VS C</H1>

正如引言一节中所有，这份文档不打算处理不兼容的C++特性，C语言中不支持的也就是C++
语言或者库。大量的C++特性和库都可归到此类。下面是这些特性的一份不完全列表：

- 匿名联合体 (anonymous unions)
- 类 (classes)
- 构造函数和析构函数 (constructors and destructors)
- 异常和 try/catch 语句块 (exceptions and try/catch blocks)
- 外部函数连接（比如：extern "C"）(external function linkages)
- 函数重载 (function overloading)
- 成员函数 (member functions)
- 命名空间 (namespaces)
- new 和 delete 操作符和函数 (new and delete operators and functions)
- 操作符重载 (operator overloading)
- 引用类型 (reference types)
- 标准模板库（STL） (standard template library)
- 模板类 (template classes)
- 模板方法 (template functions )


<H1 id="c99vscpp98">C99的变更 VS C++98</H1>

The following items are incompatibilities between C90 and C++98, but have since
been changed in C99 so that they no longer cause problems between the two
languages.

下面这些条目在C90和C++98中不兼容，但是在C99中有了变化，所以在两种语言中不再有不
兼容的问题。

- 组合类型的初值（initializers）

  C90要求组合类型（struct、array、union）的自变量（automatic）和`register`变
  量的初值必须只包含常量表达式。（不过有很多编译器都没有遵从这一限制）

  C99 移除了这一限制，允许在给这种变量提供初值时使用非常量表达式。

  C++ 允许使用非常量表达式来初始化自变量（automatic）和`register`变量。（它也允
  许在初始化静态变量和`external`变量的时候使用任意的非常量表达式）

  比如：

```
// C and C++ code
void foo(int i)
{
    float   x = (float)i;           // Valid C90, C99, and C++
    int     m[3] = { 1, 2, 3 };     // Valid C90, C99, and C++
    int     g[2] = { 0, i };        // Invalid C90
}
```

  [C99: §6.7.8]
  [C++98: §3.7.2, 8.5, 8.5.1]

- 注释

  C++ 可以识别 `//...` 式的注释和 `/*...*/` 式的注释.

  C90 只能识别 `/*...*/` 式的注释。`//...`式的注释再C90中通常会导致一个语法错误
  ，但是一些极端的情况下会在没有编译警告的情况下错误的通过编译。

```
i = (x//*y*/z++
   , w);
```

  C99 可以识别这两种类型的注释。

  [C99: §5.1.1.2, 6.4.9]
  [C++98: §2.1, 2.7]

- 条件表达式中的声明

  C++允许在条件表达式（出现在 for，if，while，switch 语句中）中声明局部变量。该
  变量的作用域持续到包含该条件表达式的语句中。比如：

```
for (int i = 0; i < SIZE; i++)
    a[i] = i + 1;
```

  C90 不允许该特性。

  C99 允许该特性，不过只能在 for 语句中使用。

  [C99: §6.8.5]
  [C++98: §3.3.2, 6.4, 6.5]

- 双字符组标点符号（Digraph punctuation tokens）

  C++可以识别两个字符的标点符号，称为双字符组，这种字符组无法被C90识别。双字符
  组和它们对于的符号如下：

```
<:      [
:>      ]
<%      {
%>      }
%:      #
%:%:    ##
```

   C99 可以识别同一组双字符组。

   下面的程序在 C99 和 C++ 中都是合法的

```
%:include <stdio.h>

%:ifndef BUFSIZE
 %:define BUFSIZE  512
%:endif

void copy(char d<::>, const char s<::>, int len)
<%
    while (len-- >= 0)
    <%
        d<:len:> = s<:len:>;
    %>
%>
```

  [C99: §6.4.6]
  [C++98: §2.5, 2.12]

- 隐式函数声明（Implicit function declarations）

  C90 允许一个函数在它第一次使用（调用）的地方隐式声明，函数默认的返回值是`int`
  。比如：

```
/* 作用域中没有提前声明 bar() */

void foo(void)
{
    bar();  /* 隐式声明: extern int bar() */
}
```

  C++ 不允许隐式函数声明。调用没有提前在作用域中声明的函数是非法行为。

  C99 不再允许隐式的函数声明。上面的代码在 C99 和 C++ 中都是不合法的。

  [C99: §6.5.2.2]
  [C++98: §5.2.2]

- 隐式变量声明（Implicit variable declarations）

  C90 允许声明一个变量、函数参数或者结构体成员的时候省去它的类型修饰符，它默认
  的类型是`int`。

  C99 不再允许这种省略，C++ 同样也不允许。

  下面的代码在C90中是合法的，但是在C99和C++中是不合法的：

```
static  sizes = 0;         /* Implicit int, error */

struct info
{
    const char *  name;
    const         sz;      /* Implicit int, error */
};

static foo(register i)     /* Implicit ints, error */
{
    auto  j = 3;           /* Implicit int, error */

    return (i + j);
}
```

  [C99: §6.7, 6.7.2]
  [C++98: §7, 7.1.5]

- 混合语句和声明（Intermixed declarations and statements）

  C90 的语法要求语句块中的所有声明必须出现在该语句块第一条语句之前。

  C++ 没有这一限制，它允许语句和声明在语句块中以任意顺序出现。

  C99 移除了这一限制，允许语句和声明的混合。

```
void prefind(void)
{
    int     i;

    for (i = 0; i < SZ; i++)
        if (find(arr[i]))
            break;

    const char *  s;   /* Invalid C90, valid C99 and C++ */

    s = arr[i];
    prepend(s);
}
```

  [C99: §6.8.2]
  [C++98: §6, 6.3, 6.7]

---

<H1 id="c99vscpp98">C99 VS C++98</H1>

The following items comprise the differences between C99 and C++98. Some of
these incompatibilities existed between C89 and C++98 and remain unchanged
between C99 and C++98, while others are new features that were introduced into
C99 that are incompatible with C++98.

Note that features that are specific to C++ and which are not legal C (e.g.,
class member function declarations) are not included in this section; only
language features that are common to both C and C++ are discussed. Most of the
features are valid as C but invalid as C++.

Some of these features are likely to be implemented as extensions by many C++
compilers in order to be more compatible with C compilers.

    Alternate punctuation token spellings

    C++ provides the following keywords as synonyms for punctuation tokens:

        and     &&
        and_eq  &=
        bitand  &
        bitor   |
        compl   ~
        not     !
        not_eq  !=
        or  ||
        or_eq   |=
        xor     ^
        xor_eq  ^=

    These keywords are also recognized by the C++ preprocessor.

    C90 does not have these built-in keywords, but it does provide a standard
    <iso646.h> header file that contains definitions for the same words as
    macros, behaving almost like built-in keywords.

    C++ requires implementations to provide an empty <iso646.h> header.
    Including it in a C++ program has no effect on the program. However, C code
    that does not include the <iso646.h> header is free to use these words as
    identifiers and macro names, which may cause incompatibilities when such
    code is compiled as C++.

        enum oper { nop, and, or, eq, ne };

        extern int  instr(enum oper op, struct compl *c);

    The recommended practice for code intended to be compiled as both C and C++
    is to use these identifiers only for these special meanings, and only after
    including <iso646.h>.

        // Proper header inclusion allows for the use of 'and' et al

        #ifndef __cplusplus
         #include <iso646.h>
        #endif

        int foo(float a, float b, float c)
        {
            return (a > b  and  b <= c);
        }

    [C99: §7.9]
    [C++98: §2.5, 2.11]

    Array parameter qualifiers

    C99 provides new declaration syntax for function parameters of array types,
    allowing type qualifiers (the cv-qualifiers const and volatile, and
    restrict) to be included within the first set of brackets of an array
    declarator. The qualifier modifies the type of the array parameter itself.
    For example, the following declarations are semantically identical:

        extern void  foo(int str[const]); extern void  foo(int *const str);

    In both declarations, parameter str is a const pointer to an int object.

    C99 also allows the static specifier to be placed within the brackets of an
    array declaration immediately preceding the expression specifying the size
    of the array. The presence of such a specifer indicates that the array is
    composed of at least the number of contiguous elements indicated by the size
    expression. (Presumably this is a hint to the compiler for optimizing access
    to elements of the array.) For example:

        void baz(char s[static 10]) { // s[0] thru s[9] exist and are contiguous
            ...
        }

    None of these new syntactic features are recognized by C++.

    (These features might be provided as an extension by some C++ compilers.)

    [C99: §6.7.5, 6.7.5.2, 6.7.5.3]
    [C++98: §7.1.1, 7.1.5.1, 8.3.4, 8.3.5, 8.4]

    Boolean type

    C99 supports the _Bool keyword, which declares a two-valued integer type
    (capable of representing the values true and false). It also provides a
    standard <stdbool.h> header that contains definitions for the following
    macros:

        bool    Same as _Bool false   Equal to (_Bool)0 true    Equal to
        (_Bool)1

    C++ provides bool, false, and true as reserved keywords and implements bool
    as a true built-in boolean type.

    C programs that do not include the <stdbool.h> header are free to use these
    keywords as identifiers and macro names, which may cause compatibility
    problems when such code is compiled as C++. For example:

        typedef short   bool;       // Different

        #define false   ('\0')      // Different
        #define true    (!false)    // Different

        bool  flag =    false;

    The recommended practice is therefore to use these identifiers in C only for
    these special meanings, and only after including <stdbool.h>.

    (It is likely that an empty <stdbool.h> header will be provided by most C++
    implementations as an extension.)

    [C99: §6.2.5, 6.3.1.1, 6.3.1.2, 7.16, 7.26.7]
    [C++98: §2.11, 2.13.5, 3.9.1]

    Character literals

    In C, character literals such as 'a' have type int, and thus sizeof('a') is
    equal to sizeof(int).

    In C++, character literals have type char, and thus sizeof('a') is equal to
    sizeof(char).

    This difference can lead to inconsistent behavior in some code that is
    compiled as both C and C++.

        memset(&i, 'a', sizeof('a'));   // Questionable code

    In practice, this is probably not much of a problem, since character
    constants are implicitly converted to type int when they appear within
    expressions in both C and C++.

    [C99: §6.4.4.4] [C++98: §2.13.2]

    clog identifier

    C99 declares clog() in <math.h> as the complex natural logarithm function.

    C++ declares std::clog in <iostream> as the name of the standard error
    logging output stream (analogous to the stderr stream). This name is placed
    into the global namespace if the <math.h> header is included, and refers to
    the logarithm function. If <math.h> defines clog as a preprocessor macro
    name, it can cause problems with other C++ code.

        // C++ code

        #include <iostream>
        using std::clog;

        #include <math.h>               // Possible conflict

        void foo(void)
        {
            clog << clog(2.718281828) << endl;
                                        // Possible conflict
        }

    Including both the <iostream> and the <cmath> headers in C++ code places
    both clog names into the std:: namespace, one being a variable and the other
    being a function, which should not cause any conflicts.

        // C++ code

        #include <iostream>
        #include <cmath>

        void foo(void)
        {
            std::clog << std::clog(2.718281828) << endl;
                                        // No conflict; different types
        }

        void bar(void)
        {
            complex double  (* fp)(complex double);

            fp = &std::clog;            // Unambiguous
        }

    It would appear that the safest approach to this potential conflict would be
    to avoid using both forms of clog within the same source file.

    [C99: §7.3.7.2] [C++98: §27.3.1]

    Comma operator results

    The comma operator in C always results in an r-value even if its right
    operand is an l-value, while in C++ the comma operator will result in an
    l-value if its right operand is an l-value. This means that certain
    expressions are valid in C++ but not in C:

        int     i; int     j;

        (i, j) = 1;     // Valid C++, invalid C

    [C99: §6.5.3.4, 6.5.17]
    [C++98: §5.3.3, 5.18]

    Complex floating-point type

    C99 provides built-in complex and imaginary floating point types, which are
    declared using the _Complex and _Imaginary keywords.

    There are exactly three complex types and three imaginary types in C99:

        _Complex float
        _Complex double
        _Complex long double

        _Imaginary long double
        _Imaginary double
        _Imaginary long double

    C99 also provides a standard <complex.h> header that contains definitions of
    complex floating point types, macros, and constants. In particular, this
    header defines the following macros:

        complex     Same as _Complex imaginary   Same as _Imaginary I   i  (the
        complex identity)

    C code that does not include this header is free to use these words as
    identifiers and macro names. This was an intentional part of the design of
    the _Complex and _Imaginary keywords, since this allows existing code that
    employs the new words to continue working as it did before under C89.

    Implicit widening conversions between the complex and imaginary types are
    provided, which parallel the implicit widening conversions between the
    non-complex floating point types.

        // C99 code

        #include <complex.h>

        complex double square_d(complex double a)
        {
            return (a * a);
        }

        complex float square_f(complex float a)
        {
            complex double  d = a;      // Implicit conversion

            return square_d(a);         // Implicit conversion
        }

    C++ provides a template class named complex, declared in the <complex>
    standard header file. This type is incompatible with the C99 complex types.

    C++ supports more complex types than C99, in theory, since complex is a
    template class.

        // C++ code

        #include <complex>

        complex<float> square(complex<float> a)
        {
            return (a * a);
        }

        complex<int> square(complex<int> a)
        {
            return (a * a);
        }

    It is possible to define typedefs that will work in both C99 and C++, albeit
    with some limitations:

        #ifdef __cplusplus

         #include <complex>

         typedef complex<float>           complex_float;
         typedef complex<double>          complex_double;
         typedef complex<long double>     complex_long_double;

        #else

         #include <complex.h>

         typedef complex float            complex_float;
         typedef complex double           complex_double;
         typedef complex long double      complex_long_double;

         typedef imaginary float          imaginary_float;
         typedef imaginary double         imaginary_double;
         typedef imaginary long double    imaginary_long_double;

        #endif

    Including these definitions allows for portable code that will compile as
    both C and C++ code, such as:

        complex_double square_cd(complex_double a) { return (a * a);
        }

    [C99: §6.2.5, 6.3.1.6, 6.3.1.7, 6.3.1.8]
    [C++98: §26.2]

    Compound literals

    C99 allows literals having types other than primitive types (e.g.,
    user-defined structure or array types) to be specified in constant
    expressions; these are called compound literals. For example:

        struct info { char    name[8+1];
            int     type;
        };

        extern void  add(struct info s);
        extern void  move(float coord[2]);

        void predef(void)
        {
            add((struct info){ "e", 0 });      // A struct literal
            move((float[2]){ +0.5, -2.7 });    // An array literal
        }

    C++ does not support this feature.

    C++ does provides a similar capability through the use of non-default class
    constructors, but which is not quite as flexible as the C feature:

        void predef2() { add(info("e", 0));      // Call constructor
        info::info()
        }

    (This C feature might be provided as an extension by some C++ compilers, but
    would probably be valid only for POD structure types and arrays of POD
    types.)

    [C99: §6.5.2, 6.5.2.5] [C++98: §5.2.3, 8.5, 12.1, 12.2]

    const linkage

    C specifies that a variable declared with a const qualifier is not a
    modifiable object. In all other regards, though, it is treated the same as
    any other variable. Specifically, if a const object with file scope is not
    explicitly declared static, its name has external linkage and is visible to
    other source modules.

        const int           i = 1;  // External linkage

        extern const int    j = 2;  // 'extern' optional
        static const int    k = 3;  // 'static' required

    C++ specifies that a const object with file scope has internal linkage by
    default, meaning that the object's name is not visible outside the source
    file in which it is declared. A const object must be declared with an
    explicit extern specifier in order to be visible to other source modules.

        const int           i = 1;  // Internal linkage

        extern const int    j = 2;  // 'extern' required
        static const int    k = 3;  // 'static' optional

    The recommended practice is therefore to define constants with an explicit
    static or extern specifier.

    [C99: §6.2.2, 6.7.3] [C++98: §7.1.5.1]

    Designated initializers

    C99 introduces the feature of designated initializers, which allows specific
    members of structures, unions, or arrays to be initialized explicitly by
    name or subscript. For example:

        struct info { char    name[8+1];
            int     sz;
            int     typ;
        };

        struct info  arr[] =
        {
            [0] = { .sz = 20, .name = "abc" },
            [9] = { .sz = -1, .name = "" }
        };

    Unspecified members are default-initialized.

    C++ does not support this feature.

    (This feature might be provided as an extension by some C++ compilers, but
    would probably be valid only for POD structure types and arrays of POD
    types. However, C++ already provides a similar capability through the use of
    non-default class constructors.)

    [C99: §6.7.8] [C++98: §8.5.1, 12.1]

    Duplicate typedefs

    C does not allow a given typedef to appear more than once in the same scope.

    C++ handles typedefs and type names differently than C, and allows redundant
    occurrences of a given typedef within the same scope.

    Thus the following code is valid in C++ but invalid in C:

        typedef int  MyInt;
        typedef int  MyInt;     // Valid C++, invalid C

    This means that typedefs that might be included more than once in a program
    (e.g., common typedefs that occur in multiple header files) should be
    guarded by preprocessing directives if such source code is meant to be
    compiled as both C and C++. For example:

        //======================================== // one.h

        #ifndef MYINT_T
         #define MYINT_T
         typedef int  MyInt;
        #endif
        ...

        //========================================
        // two.h

        #ifndef MYINT_T
         #define MYINT_T
         typedef int  MyInt;
        #endif
        ...

    Thus code can include multiple header files without causing an error in C:

        // Include multiple headers that define typedef MyInt
        #include "one.h"
        #include "two.h"

        MyInt   my_counter = 0;

    [C99: §6.7, 6.7.7]
    [C++98: §7.1.3]

    Dynamic sizeof evaluation

    Because C99 supports variable-length arrays (VLAs), the sizeof operator does
    not necessarily evaluate to a constant (compile-time) value. Any expression
    that involves applying the sizeof operator to a VLA operand must be
    evaluated at runtime (any other use of sizeof can be evaluated at compile
    time). For example:

        size_t dsize(int sz) { float   arr[sz];          // VLA, dynamically
        allocated

            if (sz <= 0)
                return sizeof(sz);    // Evaluated at compile time
            else
                return sizeof(arr);   // Evaluated at runtime
        }

    C++ does not support VLAs, so C code that applies the sizeof operator to VLA
    operands will cause problems when compiled as C++.

    [C99: §6.5.3.4, 6.7.5, 6.7.5.2] [C++98: §5.3, 5.3.3]

    Empty parameter lists

    C distinguishes between a function declared with an empty parameter list and
    a function declared with a parameter list consisting of only void. The
    former is an unprototyped function taking an unspecified number of
    arguments, while the latter is a prototyped function taking no arguments.

        // C code

        extern int  foo();          // Unspecified parameters
        extern int  bar(void);      // No parameters

        void baz()
        {
            foo(0);         // Valid C, invalid C++
            foo(1, 2);      // Valid C, invalid C++

            bar();          // Okay in both C and C++
            bar(1);         // Error in both C and C++
        }

    C++, on the other hand, makes no distinction between the two declarations
    and considers them both to mean a function taking no arguments.

        // C++ code

        extern int  xyz();

        extern int  xyz(void);  // Same as 'xyz()' in C++,
                                // Different and invalid in C

    For code that is intended to be compiled as either C or C++, the best
    solution to this problem is to always declare functions taking no parameters
    with an explicit void prototype. For example:

        // Compiles as both C and C++ int bosho(void) {
            ...
        }

    Empty function prototypes are a deprecated feature in C99 (as they were in C89).

    [C99: §6.7.5.3]
    [C++98: §8.3.5, C.1.6.8.3.5]

    Empty preprocessor function macro arguments

    C99 allows preprocessor function macros to be specified with empty (missing)
    arguments.

        #define ADD3(a,b,c)  (+ a + b + c + 0)

        ADD3(1, 2, 3)   => (+ 1 + 2 + 3 + 0)
        ADD3(1, 2, )    => (+ 1 + 2 + + 0)
        ADD3(1, , 3)    => (+ 1 + + 3 + 0)
        ADD3(1,,)       => (+ 1 + + + 0)
        ADD3(,,)        => (+ + + + 0)

    C++ does not support empty preprocessor function macros arguments.

    (This feature is likely to be provided as an extension by many C++ compilers.)

    [C99: §6.10.3, 6.10.3.1]
    [C++98: §16.3., 16.3.1]

    Enumeration constants

    Enumeration constants in C are essentially just named constants of type
    signed int. As such, they are constrained to having an initialization value
    that falls within the range [INT_MIN,INT_MAX]. This also means that for any
    given enumeration constant RED, the values of sizeof(RED) and sizeof(int)
    are always the same.

    C++ enumeration constants have the same type as their enumeration type,
    which means that they have the same size and alignment as their underlying
    integer type. This means that the values of sizeof(RED) and sizeof(int) are
    not necessarily the same for any given enumeration constant RED. Enumeration
    constants also have a wider range of possible underlying types in C++ than
    in C: signed int, unsigned int, signed long, and unsigned long. As such,
    they also have a wider range of valid initialization values.

    This may cause incompatibilities for C code compiled as C++, if the C++
    compiler chooses to implement an enumeration type as a different size than
    it would be in C, or if the program relies on the results of expressions
    such as sizeof(RED).

        enum ControlBits
        {
            CB_LOAD =   0x0001,
            CB_STORE =  0x0002,
            ...
            CB_TRACE =  LONG_MAX+1,       // (Undefined behavior)
            CB_ALL =    ULONG_MAX
        };

    [C99: §6.4.4.3, 6.7.2.2]
    [C++98: §4.5, 7.2]

    Enumeration declarations with trailing comma

    C99 allows a trailing comma to follow the last enumeration constant
    initializer within an enumeration type declaration, similar to structure
    member initialization lists. For example:

        enum Color { RED = 0, GREEN, BLUE, };

    C++ does not allow this.

    (This feature is likely to be provided as an extension by many C++ compilers.)

    [C99: §6.7.2.2]
    [C++98: §7.2]

    Enumeration types

    C specifies that each enumerated type is a unique type, distinct from all
    other enumerated types within the same program. The implementation is free
    to use a different underlying primitive integer type for each enumerated
    type. This means that sizeof(enum A) and sizeof(enum B) are not necessarily
    the same. This also means, given that RED is an enumeration constant of type
    enum Color, that sizeof(RED) and sizeof(enum Color) are not necessarily the
    same (since all enumeration constants are of type signed int).

    All enumeration constants, though, convert to values of type signed int when
    they appear in expressions. Since enumeration constants cannot portably be
    wider than int, it might appear that int is the widest enumeration type;
    however, implementations are free to support wider enumeration integer
    types. Such extended types may be different than the types used by a C++
    compiler, however.

    In C, objects of enumeration types may be assigned integer values without
    the need for a explicit cast. For example:

        // C code

        enum Color { RED, BLUE, GREEN };

        int         c = RED;    // Cast not needed
        enum Color  col = 1;    // Cast not needed

    C++ also specifies that all enumerated types are unique and distinct types,
    but it goes further than C to enforce this. In particular, a function name
    can be overloaded to take an argument of different enumerated types. While
    objects of enumerated types implicitly convert to integer values, integer
    values require an explicit cast to be converted into enumerated types.
    Implicitly converted enumeration values are converted to their underlying
    integer type, which is not necessarily signed int. For example:

        // C++ code

        enum Color { ... };

        enum Color setColor(int h)
        {
            enum Color  c;

            c = h;             // Error, no implicit conversion
            return c;
        }

        int hue(enum Color c)
        {
            return (c + 128);  // Implicit conversion,
                               // but might not be signed int
        }

    Since a C++ enumeration constant has the same type and size as its
    enumeration type, this means, given that RED is an enumeration constant of
    type enum Color, that the values of sizeof(RED) and sizeof(enum Color) are
    exactly the same, which differs from the rules in C.

    There is no guarantee that a given enumeration type is implemented as the
    same underlying type in both C and C++, or even in different C
    implementations. This affects the calling interface between C and C++
    functions. This may also cause incompatibilities for C code compiled as C++,
    if the C++ compiler chooses to implement an enumeration type as a different
    size that it would be in C, or if the program relies on the results of
    expressions such as sizeof(RED).

        // C++ code

        enum Color { ... };

        extern "C" void  foo(Color c); // Parameter types might not match

        void bar(Color c) { foo(c);         // Enum types might be different
        sizes
        }

    [C99: §6.4.4.3, 6.7.2.2]
    [C++98: §4.5, 7.2]

    Flexible array members (FAMs)

    This is also known as the struct hack. This specifies a conforming way to
    declare a structure containing a set of fixed-sized members followed by a
    flexible array member that can hold an unspecified number of elements. Such
    a structure is typically allocated by calling malloc(), passing it the
    number of bytes beyond the fixed portion of the structure to add to the
    allocation size. For example:

        struct Hack { int     count;    // Fixed member(s) int     fam[];    //
        Flexible array member };

        struct Hack * vmake(int sz) { struct Hack *  p;

            p = malloc(sizeof(struct Hack) + sz*sizeof(int)); // Allocate a
            variable-sized structure

            p->count = sz;
            for (int i = 0; i < sz; i++)
                p->fam[i] = i;

            return p;
        }

    C++ does not support flexible array members.

    (This feature might be provided as an extension by some C++ compilers, but
    would probably be valid only for POD structure types.)

    [C99: §6.7.2.1]
    [C++98: §8.3.4]

    Function name mangling

    In order to implement overloaded functions and member functions, C++
    compilers must have a means of mapping the source names of functions into
    unique symbols in the object code resulting from the compile. For example,
    the functions ::foo(int), ::foo(float), and Mine::foo() all have identical
    names (foo) but different calling signatures. In order for the linker to
    distinguish between the functions during program link time, they must be
    mangled into different symbolic names.

    This differs from the way functions names are mapped into symbolic object
    names in C, which allows for certain cases of type punning (between signed
    and unsigned integer types) and non-prototyped extern functions. Therefore C
    programs compiled as C++ will produce different symbolic names, unless the
    functions are explicitly declared as having extern "C" linkage. For example:

        int  foo(int i);   // Different symbolic names in C and C++

        #ifdef __cplusplus
        extern "C"
        #endif
        int  bar(int i);   // Same symbolic name in both C and C++

    C++ functions are implicitly declared with extern "C++" linkage.

    Another consequence of C++ function name mangling is that identifiers in C++
    are not allowed to contain two or more consecutive underscores (e.g., the
    name foo__bar is invalid). Such names are reserved for the implementation,
    ostensibly so that it may have a guaranteed means of mangling source
    function names into unique object symbolic names. (For example, an
    implementation might choose to mangle the member function Mine::foo(int)
    into something like foo__4Mine_Fi, which is a symbolic name containing
    consecutive underscores.)

    C does not reserve such names, so a C program is free to use such names in
    any manner. For example:

        void foo__bar(int i)  // Improper C++ name
        { ... }

    [C99: §5.2.4.1, 6.2.2, 6.4.2.1]
    [C++98: §2.10, 3.5, 17.4.2.2, 17.4.3.1.2, 17.4.3.1.3]

    Function pointers

    C++ functions have extern "C++" linkage by default. In order to call C
    functions from C++, the functions must be declared with extern "C" linkage.
    This is typically accomplished by placing C function declarations within an
    extern "C" block:

        extern "C"
        {
            extern int  api_read(int f, char *b);
            extern int  api_write(int f, const char *b);
        }

        extern "C"
        {
            #include "api.h"
        }

    But simply declaring functions with extern "C" linkage is not enough to
    ensure that C++ functions can call C functions properly. Specifically,
    pointers to extern "C" functions and pointers to extern "C++" functions are
    not compatible. When compiled as C++ code, function pointer declarations are
    implicitly defined as having extern "C++" linkage, so they cannot be
    assigned addresses of extern "C" functions. (Function pointers can thus be a
    source of problems when dealing with C API libraries and C callback
    functions.)

        extern int      mish(int i);    // extern "C++" function

        extern "C" int  mash(int i);

        void foo(int a)
        {
            int  (*pf)(int i);          // C++ function pointer

            pf = &mish;                 // Okay, C++ function address
            (*pf)(a);

            pf = &mash;                 // Error, C function address
            (*pf)(a);
        }

    To make the combination of function pointers and extern "C" functions work
    correctly in C++, function pointers that are assigned addresses of C
    functions must be changed to have extern "C" linkage.

    One solution is to use a typedef with the proper linkage:

        extern "C"
        {
            typedef int  (*Pcf)(int);   // C function pointer
        }

        void bar(int a)
        {
            int  (*pf)(int i);          // C++ function pointer

            pf = &mish;                 // Okay, C++ function address
            (*pf)(a);

            Pcf  pc;                    // C function pointer

            pc = &mash;                 // Okay, C function address
            (*pc)(a);
        }

    [C99: §6.2.5, 6.3.2.3, 6.5.2.2]
    [C++98: §5.2.2, 17.4.2.2, 17.4.3.1.3]

    Hexadecimal floating-point literals

    C99 recognizes hexadecimal floating-point literals, having a "0x" prefix and
    a "p" exponent specifier. For example:

        float  pi = 0x3.243F6A88p+03;

    C99 also provides additional format specifiers for the printf() and scanf()
    family of standard library functions:

        printf("%9.3a", f);
        printf("%12.6lA", d);

    (These features are likely to be provided as extensions by many C++ compilers.)

    [C99: §6.4.4.2, 6.4.8]
    [C++98: §2.9, 2.13.3]

    IEC 60559 arithmetic support

    C99 allows an implementation to pre-define the __STD_IEC_559 preprocessor
    macro, indicating that it conforms to certain required behavior of the IEC
    60559 (a.k.a. IEEE 599) specification regarding floating-point arithmetic
    and library functions. Implementations that do not pre-define this macro are
    not require to provide conforming floating-point behavior.

    C++ does not make any special provisions for implementations that explicitly
    support the IEC 60559 floating-point specification.

    Conformance to IEC 60559 floating-point arithmetic, and the pre-definition
    of the __STD_IEC_559 macro, is likely to be provided as an extension by many
    C++ compilers.

    C99 also allows an implementation to pre-define the __STD_IEC_559_COMPLEX
    preprocessor macro to indicate that it conforms to the behavior specified by
    IEC 60559 for complex floating-point arithmetic and library functions. This
    affects the way the _Complex and _Imaginary types are implemented.

    C++ provides library functions for complex floating-point arithmetic by
    providing the complex<> template class, declared in the standard <complex>
    header file. This type is incompatible with the C99 complex types.

    Conformance to the complex arithmetic specification, and the pre-definition
    of the __STD_IEC_559 macro, might also be provided by many C++ compilers,
    and this would indicate how the complex<> template class is implemented.

    [C99: §6.10.8, F, G]
    [C++98: §16.8]

    Inline functions

    Both C99 and C++ allow functions to be defined as inline, which is a hint to
    the compiler that invocations of such functions can be replaced with inline
    code expansions rather than actual function calls. Inline functions are not
    supposed to be a source of incompatibilities between C99 and C++ in
    practice, but there is a small difference in the semantics of the two
    languages.

    C++ requires all of the definitions for a given inline function to be
    composed of exactly the same token sequence.

    C99, however, allows multiple definitions of a given inline function to be
    different, and does not require the compiler to detect such differences or
    issue a diagnostic.

    Thus the following two example source files, which define two slightly
    different versions of the same inline function, constitute acceptable C99
    code but invalid C++ code:

        //========================================
        // one.c

        inline int twice(int i)         // One definition
        {
            return i * i;
        }

        int foo(int j)
        {
            return twice(j);
        }

        //========================================
        // two.c

        typedef int  integer;

        inline integer twice(integer a) // Another definition
        {
            return (a * a);
        }

        int bar(int b)
        {
            return twice(b);
        }

    This should not be a problem in practice, provided that multiple inline
    function definitions occur only in shared header files (which ensures that
    the multiple function definitions are composed of the same token sequences).

    [C99: §6.7.4]
    [C++98: §7.1.2]

    Integer types headers

    C99 provides the header file <stdint.h>, which contains declarations and
    macro definitions for standard integer types. For example:

        int  height(int_least32_t x);
        int  width(uint16_t x);

    C++ does not provide these types or header files.

    (This feature is likely to be provided as an extension by many C++
    compilers. Some C++ compilers might also provide a <cstdint> header file as
    an extension.)

    [C99: §7.1.2, 7.18]
    [C++98: §17.4.1.2, D.5]

    Library function prototypes

    The C++ standard library header files amend some of the standard C library
    function declarations so as to be more type-safe when used in C++. For
    example, the standard C library function declaration:

        // <string.h>
        extern char *   strchr(const char *s, int c);

    is replaced with these near-equivalent declarations in the C++ library:

        // <cstring>
        extern const char * strchr(const char *s, int c);
        extern char *       strchr(char *s, int c);

    These slightly different declarations can cause problems when C code is
    compiled as C++ code, such as:

        // C code
        const char * s = ...;
        char *       p;

        p = strchr(s, 'a');             // Valid C, invalid C++

    This kind of code results in an attempt to assign a const pointer returned
    from a function to a non-const variable. A simple cast corrects the code,
    making it valid as both C++ and C code, as in:

        // Corrected for C++
        p = (char *) strchr(s, 'a');    // Valid C and C++

    [C99: §7.21.5, 7.24.4.5]
    [C++98: §17.4.1.2, 21.4]

    Library header files

    C++ provides the standard C89 library as part of its library.

    C99 adds a few header files that are not included as part of the standard
    C++ library, though:

        <complex.h>
        <fenv.h>
        <inttypes.h>
        <stdbool.h>
        <stdint.h>
        <tgmath.h>

    Even though C++ provides the C89 standard C library headers as part of its
    library, it deems their use as deprecated. Instead, it encourages
    programmers to prefer the equivalent set of C++ header files which provide
    the same functionality as the C header files:

        <math.h>    replaced by     <cmath>
        <stddef.h>  replaced by     <cstddef>
        <stdio.h>   replaced by     <cstdio>
        <stdlib.h>  replaced by     <cstdlib>
        etc.        etc.

    Deprecating the use of the C header files thus makes the following valid
    C++98 program possibly invalid under a future revision of standard C++:

        #include <stdio.h>     // Deprecated in C++

        int main(void)
        {
            printf("Hello, world\n");
            return 0;
        }

    The program can be modified by removing the use of deprecated features in
    order to make it portable to future implementations of standard C++:

        #ifdef __cplusplus
         #include <cstdio>     // C++ only
         using std::printf;
        #else
         #include <stdio.h>    // C only
        #endif

        int main(void)
        {
            printf("Hello, world\n");
            return 0;
        }

    [C99: §7.1.2]
    [C++98: §17.4.1.2, D.5]

    long long integer type

    C99 provides signed long long and unsigned long long integer types to its
    repertoire of primitive types, which are binary integer types at least 64
    bits wide.

    C99 also has enhanced lexical rules to allow for integer constants of these
    types. For example:

        long long int           i = -9000000000000000000LL;
        unsigned long long int  u = 18000000000000000000LLU;

    C99 also provides several new macros in <limits.h>, new format specifiers
    for the printf() and scanf() family of standard library functions, and
    additional standard library functions that support these types. For example:

        void pr(long long i)
        {
            printf("%lld", i);
        }

    C++ does not recognize these integer types.

    (These features are likely to be provided as extensions by many C++
    compilers, especially those that provide the same runtime library for both C
    and C++ environments.)

    [C99: §5.2.4.2.1, 6.2.5, 6.3.1.1, 6.4.4.1, 6.7.2, 7.12.9, 7.18.4, 7.19.6.1,
    7.19.6.2, 7.20.1, 7.20.6, 7.24.2.1, 7.24.2.2, 7.24.4, A.1.5, B.11, B.19,
    B.23, F.3, H.2]
    [C++98: §2.13.1, 3.9.1, 21.4, 22.2.2.2.2, 27.8.2, C.2]

    Nested structure tags

    Nested structure types may be declared within other structures. The scope of
    the inner structure tag extends outside the scope of the outer structure in
    C, but does not do so in C++. Structure declarations possess their own scope
    in C++, but do not in C. This applies to any struct, union, and enumerated
    types declared within a structure declaration. For example:

        struct Outer
        {
            struct Inner        // Nested structure declaration
            {
                int         a;
                float       f;
            }           in;

            enum E              // Nested enum type declaration
            {
                UKNOWN, OFF, ON
            }           state;
        };

        struct Inner    si;     // Nested type is visible in C,
                                // Not visible in C++

        enum E          et;     // Nested type is visible in C,
                                // Not visible in C++

    In order to be visible in C++, the inner declarations must be explicitly
    named using its outer class prefix, or they must be declared outside the
    outer structure so that they have file scope. The former case, for example:

        // C++ code

        Outer::Inner     si;    // Explicit type name
        Outer::E         et;    // Explicit type name

    And the latter case:

        // C and C++ code

        struct Inner            // Declaration is no longer nested
        {
            int         a;
            float       f;
        };

        enum E                  // Declaration is no longer nested
        {
            UKNOWN, OFF, ON
        };

        struct Outer
        {
            struct Inner    in;
            enum E          state;
        };

    [C99: §6.2.1, 6.2.3, 6.7.2.1, 6.7.2.3]
    [C++98: §9.9, C.1.2.3.3]

    Non-prototype function declarations

    C supports non-prototype (a.k.a. K&R-style) function definitions. (Like C90,
    C99 deems this as deprecated practice.) For example:

        int foo(a, b)     // Deprecated syntax
            int  a;
            int  b;
        {
            return (a + b);
        }

    C++ allows only prototyped function definitions. So in order to compile the
    example above as C++ code, it must be rewritten in function prototype form:

        int foo(int a, int b)
        {
            return (a + b);
        }

    [C99: §6.2.7, 6.5.2.2, 6.7.5.3, 6.9.1, 6.11.6, 6.11.7]
    [C++98: §5.2.2, 8.3.5, 8.4, C.1.6]

    Old-style casts

    C++ provides four typecast operators:

        const_cast
        dynamic_cast
        reinterpret_cast
        static_cast

    While the following C code is also valid C++98 code, it may not be
    considered valid code in a future revision of the C++ standard:

        char *        p;
        const char *  s = (const char *) p;

    One possible work-around is to use macros in C that simulate the C++
    typecast operators:

        #ifdef __cplusplus
         #define const_cast(t,e)        const_cast<t>(e)
         #define dynamic_cast(t,e)      dynamic_cast<t>(e)
         #define reinterpret_cast(t,e)  reinterpret_cast<t>(e)
         #define static_cast(t,e)       static_cast<t>(e)
        #else
         #define const_cast(t,e)        ((t)(e))
         #define dynamic_cast(t,e)      ((t)(e))
         #define reinterpret_cast(t,e)  ((t)(e))
         #define static_cast(t,e)       ((t)(e))
        #endif

        const char *  s = const_cast(const char *, p);

    All four casts are included above even though dynamic_cast is not really
    useful in C code. Perhaps a better definition for dynamic_cast in C would
    be:

         #define dynamic_cast(t,e)      _Do_not_use_dynamic_cast
                                            // Produces a compile-time error

    C++ also provides functional typecasts, which are not recognized in C:

        f = float(i);   // C++ cast to float; invalid C

    These kinds of typecasts cannot be used in code that is compiled as both C
    and C++.

    [C99: §6.3, 6.54]
    [C++98: §5.2, 5.2.3, 5.2.7, 5.2.9, 5.2.10, 5.2.11, 5.4, 14.6.2.2, 14.6.2.3]

    One definition rule

    C allows tentative definitions for variables, e.g.:

        int  i;        // Tentative definition
        int  i = 1;    // Explicit definition

    C++ does not allow this. Only one definition of any given variable is
    allowed within a program.

    C also allows, or at least does not require a diagnostic for, different
    source files containing conflicting definitions. For example:

        //========================================
        // one.c

        struct Capri                // A declaration
        {
            int     a;
            int     b;
        };

        //========================================
        // two.c

        struct Capri                // Conflicting declaration
        {
            float   x;
        };

    C++ deems this invalid, requiring both definitions to consist of the same
    sequence of tokens.

    C allows definitions of the same function or object in different source
    files to be composed of different token sequences, provided that they are
    semantically identical.

    The C++ rules are more strict, requiring the multiple definitions to be
    composed of identical token sequences. Thus the following code, which
    contains multiple definitions that are semantically equivalent but
    syntactically (token-wise) different, is valid in C but invalid in C++:

        //========================================
        // file1.c

        struct Waffle               // A declaration
        {
            int     a;
        };

        int syrup(int amt)          // A definition
        {
            return amt*2;
        }

        //========================================
        // file2.c - Valid C, but invalid C++

        typedef int     IType;

        struct Waffle               // Equivalent declaration,
        {                           // but a different token sequence
            IType   a;
        };

        IType syrup(IType quant)    // Equivalent definition,
        {                           // but a different token sequence
            return (quant*2);
        }

    [C99: §6.9.2, J.2]
    [C++98: §3.2, C.1.2.3.1]

    _Pragma keyword

    C99 provides the _Pragma keyword, which operates in a similar fashion to the
    #pragma preprocessor directive. For example, these two constructs are
    equivalent:

        #pragma FLT_ROUND_INF   // Preprocessor pragma

        _Pragma(FLT_ROUND_INF)  // Pragma statement

    C++ does not support the _Pragma keyword.

    (This feature is likely to be provided as an extension by many C++ compilers.)

    [C99: §5.1.1.2, 6.10.6, 6.10.9]
    [C++98: §16.6]

    Predefined identifiers

    C99 provides a predefined identifier, __func__, which acts like a string
    literal containing the name of the enclosing function. For example:

        int incr(int a)
        {
            fprintf(dbgf, "%s(%d)\n", __func__, a);
            return ++a;
        }

    (While this feature is likely to be provided as an extension by many C++
    compilers, it is unclear what its value would be, especially for member
    functions within nested template classes declared within nested namespaces.)

    [C99: §6.4.2.2, 7.2.1.1, J.2]

    Reserved keywords in C99

    C99 has a few reserved keywords that are not recognized by C++:

        restrict
        _Bool
        _Complex
        _Imaginary
        _Pragma

    This will cause problems when C code containing these tokens is compiled as
    C++. For example:

        extern int   set_name(char *restrict n);

    [C99: §6.4.1, 6.7.2, 6.7.3, 6.7.3.1, 6.10.9, 7.3.1, 7.16, A.1.2]
    [C++98: §2.11]

    Reserved keywords in C++

    C++ has a few keywords that are not recognized by C99:

        bool    mutable     this
        catch   namespace   throw
        class   new     true
        const_cast  operator    try
        delete  private     typeid
        dynamic_cast    protected   typename
        explicit    public  using
        export  reinterpret_cast    virtual
        false   static_cast     wchar_t
        friend  template

    C++ also specifically reserves the asm keyword, which may or may not be
    reserved in C implementations.

    C code is free to use these keywords as identifiers and macro names. This
    will cause problems when C code containing these tokens is compiled as C++.
    For example:

        extern int   try(int attempt);
        extern void  frob(struct template *t, bool delete);

    [C99: §6.4.1]
    [C++98: §2.11]

    restrict keyword

    C99 supports the restrict keyword, which allows for certain optimizations
    involving pointers. For example:

        void copy(int *restrict d, const int *restrict s, int n)
        {
            while (n-- > 0)
                *d++ = *s++;
        }

    C++ does not recognize this keyword.

    A simple work-around for code that is meant to be compiled as either C or
    C++ is to use a macro for the restrict keyword:

        #ifdef __cplusplus
         #define restrict    /* nothing */
        #endif

    (This feature is likely to be provided as an extension by many C++
    compilers. If it is, it is also likely to be allowed as a reference modifier
    as well as a pointer modifier.)

    [C99: §6.2.5, 6.4.1, 6.7.3, 6.7.3.1, 7, A.1.2, J.2]
    [C++98: §2.11]

    Returning void

    C++ allows functions of return type void to explicitly return expressions of
    type void. C does not allow void functions to return any kind of expression.

    For example:

        void foo(someType expr)
        {
            ...
            return (void)expr;      // Valid C++, invalid C
        }

    This is allowed in C++ primarily to allow template functions to accept any
    function return type, including void, as a template parameter. For example:

        // C++ code
        template <typename T>
        T bar(someType expr)
        {
            ...
            return (T)expr;         // Valid even if T is void
        }

    [C99: §6.8.6.4]
    [C++98: §3.9.1, 6.6.3]

    static linkage

    Both C and C++ allow objects and functions to have static file linkage, also
    known as internal linkage. C++, however, deems this as deprecated practice,
    preferring the use of unnamed namespaces instead. (C++ objects and functions
    declared within unnamed namespaces have external linkage unless they are
    explicitly declared static. C++ deems the use of static specifiers on
    objects or function declarations within namespace scope as deprecated.)

    While it is not a problem for C code compiled under C++98 rules, it may
    become a problem in a future revision of the C++ language. For example, the
    following fragment uses the deprecated static feature:

        // C and C++ code

        static int  bufsize = 1024;
        static int  counter = 0;

        static long square(long x)
        {
            return (x * x);
        }

    The preferred way of doing this in C++ is:

        // C++ code

        namespace /*unnamed*/
        {
            static int  bufsize = 1024;
            static int  counter = 0;

            static long square(long x)
            {
                return (x * x);
            }

    }

    (Note that the use of the static specifiers is unnecessary.)

    A possible work-around is to use preprocessor macros and wrappers:

        // C and C++ code

        #ifdef __cplusplus
         #define STATIC  static
        #endif

        #ifdef __cplusplus
        namespace /*unnamed*/
        {
        #endif

        STATIC int  bufsize = 1024;
        STATIC int  counter = 0;

        STATIC long square(long x)
        {
            return (x * x);
        }

        #ifdef __cplusplus
        }
        #endif

    [C99: §6.2.2, 6.2.4, 6.7.1, 6.9, 6.9.1, 6.9.2, 6.11.2]
    [C++98: §3.3.5, 3.5, 7.3.1, 7.3.1.1, D.2]

    String initializers

    C allows character arrays to be initialized with string constants. It also
    allows a string constant initializer to contain exactly one more character
    than the array it initializes, i.e., the implicit terminating null character
    of the string may be ignored. For example:

        char  name1[] =  "Harry";   // Array of 6 char

        char  name2[6] = "Harry";   // Array of 6 char

        char  name3[] =  { 'H', 'a', 'r', 'r', 'y', '\0' };
                                    // Same as 'name1' initialization

        char  name4[5] = "Harry";   // Array of 5 char, no null char

    C++ also allows character arrays to be initialized with string constants,
    but always includes the terminating null character in the initialization.
    Thus the last initializer (name4) in the example above is invalid in C++.

    [C99: §6.7.8]
    [C++98: §8.5, 8.5.2]

    String literals are const

    In C, string literals have type char[n], but are not modifiable (i.e.,
    attempting to modify the contents of a string literal is undefined
    behavior).

    In C++, string literals have type const char[n] and are also not modifiable.

    When a string literal is used in an expression (or passed to a function),
    both C and C++ implicit convert it into a pointer of type char *. (The C++
    conversion is considered to be two conversions, the first being an
    array-to-pointer conversion from type const char[n] to type const char *,
    and the second being a qualification conversion to type char *.)

    The following code is valid in both C and C++.

        extern void  frob(char *s);
                            // Argument is not const char *

        void foo(void)
        {
            frob("abc");    // Valid in both C and C++,
                            // since literal converts to char *
        }

    This language feature does not present an incompatibility between C99 and
    C++98. However, the implicit conversion has been deprecated in C++
    (presumably to be replaced by a single implicit conversion to type const
    char *), which means that a future revision of C++ may no longer accept the
    code above as valid code.

    [C99: §6.3.2.1, 6.4.5, 6.5.1, 6.7.8]
    [C++98: §2.13.4, 4.2, D.4]

    Structures declared in function prototypes

    C allows struct, union, and enum types to be declared within function
    prototype scope, e.g.:

        extern void  foo(const struct info { int typ; int sz; } *s);

        int bar(struct point { int x, y; } pt)
        { ... }

    C also allows structure types to be declared as function return types, as in:

        extern struct pt { int x; }  pos(void);

    C++ does not allow either of these, since the scope of the structure
    declared in this fashion does not extend outside the function declaration or
    definition, making it impossible to define objects of that structure type
    which could be passed as arguments to the function or to assign function
    return values into objects of that type.

    Both C and C++ allow declarations of incomplete structure types within
    function prototypes and as function return types, though:

        void  frob(struct sym *s);  // Okay, pointer to incomplete type
        struct typ *  fraz(void);   // Ditto

    [C99: §6.2.1, 6.7.2.3, 6.7.5.3, I]
    [C++98: §3.3.1, 3.3.3, 8.3.5, C.1.6.8.3.5]

    Type-generic math functions

    C99 supports type-generic mathematical functions. These are functions that
    are essentially overloaded on the three floating-point types (float, double,
    and long double) and the three complex floating-point types (complex float,
    complex double, and complex long double). To use them, the header file
    <tgmath.h> must be included; the functions are defined as macros, presumably
    replaced by implementation-defined names.

    For example, the following is one possible implementation of the
    type-generic functions:

        /* Equivalent <tgmath.h> contents:
        * extern float                sin(float x);
        * extern double               sin(double x);
        * extern long double          sin(long double x);
        * extern float complex        sin(float complex x);
        * extern double complex       sin(double complex x);
        * extern long double complex  sin(long double complex x);
        * etc...
        */

        // Macro definitions
        #define sin  __tg_sin       // Built-in compiler symbol
        #define cos  __tg_cos       // Built-in compiler symbol
        #define tan  __tg_tan       // Built-in compiler symbol
        etc...

    C++ can also provide type-generic functions, since it is quite capable of
    providing multiple overloaded function definitions.

    (Support for type-generic mathematical functions might be provided by many
    C++ implementations as an extension, although the exact nature of such
    generic/overloaded functions would most likely differ substantially from the
    corresponding C99 implementation. In particular, pointers to type-generic
    functions would probably behave differently.)

    [C99: §7.22]
    [C++98: §13, 13.1, 13.3.1, 13.3.2, 13.3.3]

    Typedefs versus type tags

    C requires type tags to be preceded by the struct, union, or enum keyword.

    C++ treats type tags as implicit typedef names.

    Thus the following code is valid C but invalid C++:

        // Valid C, invalid C++

        typedef int  type;

        struct type
        {
            type            memb;   // int
            struct type *   next;   // struct pointer
        };

        void foo(type t, int i)
        {
            int          type;
            struct type  s;

            type = i + t + sizeof(type);
            s.memb = type;
        }

    This difference in the treatment of typedefs can also lead to code that is
    valid as both C and C++, but which has different semantic behavior. For
    example:

        int  sz = 80;

        int size(void)
        {
            struct sz
            { ... };

            return sizeof(sz);      // sizeof(int) in C,
                                    // sizeof(struct sz) in C++
        }

    [C99: §6.2.1, 6.2.3, 6.7, 6.7.2.1, 6.7.2.2, 6.7.2.3]
    [C++98: §3.1, 3.3.1, 3.3.7, 3.4, 3.4.4, 3.9, 7.1.3, 7.1.5, 7.1.5.2, 9.1]

    Variable-argument function declarators

    C90 syntax allows a trailing ellipsis in the parameter list of a function
    declarator, which specifies that the function can take zero or more
    additional arguments after the last named parameter.

    C++ also allows variable function argument lists, but provides two
    syntactical forms for this feature.

        /* Variable-argument function declarations */
        int  foo(int a, int b, ...);      // Valid C++ and C
        int  bar(int a, int b ...);       // Valid C++, invalid C

    [C99: §6.7.5]
    [C++98: §8.3.5]

    Variable-argument preprocessor function macros

    C99 supports preprocessor function macros that may take a variable number of
    arguments. Such macros are defined with a trailing '...' token in their
    parameter lists, and may use the __VA_ARGS__ reserved identifier in their
    replacement text.

    For example:

        #define DEBUGF(f,...) \
            (fprintf(dbgf, "%s(): ", f), fprintf(dbgf, __VA_ARGS__))

        #define DEBUGL(...) \
            fprintf(dbgf, __VA_ARGS__)

        int incr(int *a)
        {
            DEBUGF("incr", "before: a=%d\n", *a);
            (*a)++;
            DEBUGL("after: a=%d\n", *a);
            return (*a);
        }

    C++ does not provide this feature.

    (This feature is likely to be provided as an extension by many C++ compilers.)

    [C99: §6.10.3, 6.10.3.1, 6.10.3.4, 6.10.3.5]
    [C++98: §16.3, 16.3.1]

    Variable-length arrays (VLAs)

    C99 supports variable-length arrays, which are arrays of automatic storage
    whose size is determined dynamically at program execution time. For example:

        size_t sum(int sz)
        {
            float   arr[sz];      // VLA, dynamically allocated

            while (sz-- > 0)
                arr[sz] = sz;
            return sizeof(arr);   // Evaluated at runtime
        }

    C99 also provides new declaration syntax for function parameters of VLA
    types, allowing a variable identifier or a '*' to occur within the brackets
    of an array function parameter declaration in place of a constant integer
    size expression. The following example illustrates the syntax involved in
    passing VLAs to a function:

        extern float  sum_square(int n, float a[*]);

        float sum_cube(int n, float a[m])
        {
            ...
        }

        void add_seq(int n)
        {
            float   x[n];       // VLA
            float   s;

            ...
            s = sum_square(n, x) + sum_cube(n, x);
            ...
        }

    VLA function parameter declarations using a '*' can only appear in function
    declarations (with prototypes) and not in function definitions. Note that
    this capability also affects the way sizeof expressions are evaluated.

    C++ does not support VLAs.

    [C99: §6.7.5, 6.7.5.2, 6.7.5.3, 6.7.6]
    [C++98: §8.3.4, 8.3.5, 8.4]

    Void pointer assignments

    C allows a pointer to void (void *) value to be assigned to an object of any
    other pointer type without requiring a cast. This allows such things as
    assigning the return value of malloc() to a pointer variable without the
    need for an explicit cast.

    C++ does not allow assigning a pointer to void directly to an object of any
    other pointer type without an explicit cast. This is considered a breach of
    type safety, so an explicit cast is required. Thus the following code is
    valid C but invalid C++:

        extern void *  malloc(size_t n);

        struct object * allocate(void)
        {
            struct object *  p;

            p = malloc(sizeof(struct object));
                                // Direct assignment without a cast,
                                // valid C, invalid C++
            return p;
        }

    (Both languages allow values of any pointer type to be assigned to objects
    of type pointer to void without requiring an explicit cast.

        void *  vp;
        Type *  tp;

        vp = tp;    // No explicit cast needed,
                    // valid C and C++

    Such usage is considered type safe.)

    (Note that there are situations in C++ where pointers are implicitly
    converted to type pointer to void, such as when comparing a pointer of type
    pointer to void to another pointer of a different type, but such situations
    are considered type safe since no pointer objects are modified in the
    process.)

    [C99: §6.2.5, 6.3.2.3, 6.5.15, 6.5.16, 6.5.16.1]
    [C++98: §3.8, 3.9.2, 4.10, 5.4, 5.9, 5.10, 5.16, 5.17, 13.3.3.2]

    Wide character type

    C provides a wide character type, wchar_t, that is capable of holding a
    single wide character from an extended character set. This type is defined
    in the standard header files <stddef.h>, <stdlib.h>, and <wchar.h>.

    C++ also provides a wchar_t type, but it is a reserved keyword just like
    int. No header file is required to enable its definition.

    This means that C code that does not include any of the standard header
    files listed above is free to use wchar_t as an identifier or macro name;
    such code will not compile as C++ code.

        // Does not #include <stddef.h>, <stddef.h>, or <wchar.h>

        typedef unsigned short  wchar_t;

        wchar_t readwc(void)
        {
            ...
        }

    The recommended practice is therefore to use the wchar_t type only for its
    special meaning, and only after including <stddef.h>, <stdlib.h>, or
    <wchar.h>.

    (It is likely that a <wchar.h> header will be provided by most C++
    implementations as an extension. Some C++ compilers might also provide an
    empty <cwchar> header as an extension.)

    [C99: §3.7.3, 6.4.4.4, 6.4.5, 7.1.2, 7.17, 7.19.1, 7.20, 7.24]
    [C++98: §2.11, 2.13.2, 2.13.4, 3.9.1, 4.5, 7.1.5.2]
