# Preface
This document is my personal attempt to write a  series of notes following **"Effective modern C++"** by *Scott Meyers*. If you came across this document somewhere, I strongly recommend you to read the original text. Firstly, it is more detailed and contains less errors. Second, there are interesting examples and small wise tips, that I might have missed. Also some items from the book didn't make it here at all.

And one more thing: no grammar check has been made yet.

---
# CHAPTER 1. Deducing types
---

# Item 1: Understand template type deduction.
We have a function
```cpp
template <typename T>
void f(ParamType param);
```
and a call of the form
```cpp
f(expr);
```
Compilator deduces 2 type: one for `T`, one for `ParamType`. They are often different due to qualifiers. There are 3 cases:
* `ParamType` is a pointer of reference, but not universal reference.
* `ParamType` is a universal reference.
* `ParamType` is neither a pointer nor a reference.

## Case 1 : `ParamType` is a Reference or Pointer, but not a Universal Reference
1. If `expr`’s type is a reference, ignore the reference part.
2. Pattern-match `expr`’s type against `ParamType` to determine `T`.

```cpp
template <typename T>
void f(T& param);

int        x = 12;
int const  cx = x;
int const& rx = x;

f(x);   // T - int;         param - int&
f(cx);  // T - int const;   param - int const&
f(rx);  // T - int const;   param - int const&
```
If we add *constness* to param, that changes some things.
```cpp
template <typename T>
void f(T const& param);

int        x = 12;
int const  cx = x;
int const& rx = x;

f(x);   // T - int;     param - int const&
f(cx);  // T - int;     param - int const&
f(rx);  // T - int;     param - int const&
```
If we change the reference to a pointer, things will be the same, except we change `&` with `*` where relevant.

## Case 2: `ParamType` is a Universal Reference
* If `expr` is an lvalue, `T` and `ParamType` are deduced as lvalues. It's the only case where `T` is a reference.
* If `expr` is an rvalue, apply Case 1 rules.

```cpp
template <typename T>
void f(T&& param);  // universal reference

int        x = 12;
int const  cx = x;
int const& rx = x;

f(x);   // T - int&;        param - int&
f(cx);  // T - int const&;  param - int const&
f(rx);  // T - int const&;  param - int const&
f(12);  // T - int;         param - int&&
```
The key point here is the universal reference parameters make difference for what kind of expressions they get - lvalue or rvalue.

## Case 3: `ParamType`  is Neither a Pointer nor a Reference
We're dealing with pass-by-value, so the `param` will hold a copy of whatever was passed in.
* If `expr`'s type is a reference, ignore the reference part.
* Remove *constness*, *volatileness* from `expr` too.

```cpp
template <typename T>
void f(T param);    // pass-by-value

int        x = 12;
int const  cx = x;
int const& rx = x;

f(x);   // T - int;     param - int
f(cx);  // T - int;     param - int
f(rx);  // T - int;     param - int
f(12);  // T - int;     param - int
```

## Array Arguments
There's no such thing as an array parameter, so when we pass an array to our template function, it's treated as if it were a pointer.

```cpp
char const greetings[] = "Hello and shut up";

template <typename T>
void f(T param);

f(greetings);   // T - char const*; param - char const*
```

BUT there is such thing as reference-to-array parameter, so the following case works the following way.

```cpp
template <typename T>
void f(T& param);

f(greetings);   // T - char const[18];
                // param - char const (&)[18]
```

## Function Arguments
Function types can decay into function pointers. So this case resembles the one with the arrays.

```cpp
void foo(int, double);

template <typename T>
void bar(T param);

template <typename T>
void baz(T& param);

bar(foo);   // T and param - void (*) (int, double)
baz(foo);   // T and param - void (&) (int, double)
```


# Item 2: Understand `auto` type deduction.
With one exception, `auto` type deduction is template type deduction. In an `auto` variable declaration, `auto` takes the role of `T` in the template, the type specifier for the variable is the `ParamType`.

```cpp
auto        x = 12;
auto const  cx = x;
auto const& rx = x;
```
These expressions correspond to the following (in terms of type deduction).

```cpp
template <typename T>
void foo(T param);

foo(12);

template <typename T>
void foo(T const param);

foo(x);

template <typename T>
void foo(T const& param);

foo(x);
```
All the rules, 3 cases and everything about array/function types, apply to the auto type deduction.

Now about the special case. In C++11 there are 4 ways to declare a variable:

```cpp
int x1 = 12;
int x2(12);
int x3 = { 12 };
int x4 { 12 };
```
If we are to replace `int` with `auto`, the last 2 expressions will give us a declaration of a `std::initializer_list<int>` type with a single 12 in it. Template function btw cannot deduce `std::initializer_list` type from braced parameters, they just fail to compile.

```cpp
auto x1 = 12;       // int
auto x2(12);        // int
auto x3 = { 12 };   // std::initializer_list<int>
auto x4 { 12 };     // std::initializer_list<int>
```
For C++14 there's one more thing to remember. `auto` can be used as a function's return type. In this case the usual *template type deduction* rules apply, so no initializer lists here. The same is true for `auto` lambda parameters.

```cpp
auto someFoo()
{
    return { 1, 2, 3 };     // compilation error
}

auto myLambda = [](auto const& value) { ... }
myLambda({ 1, 2, 3 });      // compilation error
```


# Item 3: Understand `decltype`.
`decltype` tells you the type of a name or an expression given.

```cpp
int const x = 12;               // decltype(i)      - int const
float pow(float val, float p);  // decltype(pow)    - float(float, float)
                                // decltype(pow(2.0f, 3.0f))    - float
Widget w;                       // decltype(w)      - Widget
std::vector<int> v;             // decltype(v)      - std::vector<int>
```
`decltype` can be used to write a function, whose return type depends on its parameters types.
Suppose we write a function that implements indexing operator [] and for instance, logs the access.

```cpp
// C++11 version
// it still needs some refinement
template<typename Container, typename Index>
auto logAndAccess(Container& c, Index i) -> decltype(c[i])
{
    logAndAccess();
    return c[i];
}
```
`auto` here means nothing but the *trailing return type* syntax. This way, we can use parameters in the return type specification.
C++11 permits to auto deduce the return type of a single-statement lambdas. As for C++14, it extends this functionality to any functions. So in C++14 this would look like this.

```cpp
// C++14 version
// it still needs some refinement
template<typename Container, typename Index>
auto logAndAccess(Container& c, Index i)
{
    logAndAccess();
    return c[i];
}
```
Here's a problem: return type `auto` deduction uses template deduction rules, so it will ignore any referenceness to the c[i]. And that would lead to compile errors when a user will do something like `logAndAccess(container, 12) = "Hello";`. So C++14 has a solution, which tells the compiler to apply `decltype` rules instead.

```cpp
// C++14 version
// still needs some refinement
template<typename Container, typename Index>
decltype(auto) logAndAccess(Container& c, Index i)
{
    logAndAccess();
    return c[i];
}
```
This can be also used to declare variables:

```cpp
Widget w;
Widget const& cw = w;
decltype(auto) cw2 = cw;    // cw2 type is Widget const&
```
One thing we can futher improve is working with rvalue containers. Though it may be dangerous to save references of the items in it, we may want to make a copy! `auto itemCopy = logAndAccess(loadStrings(), 12);`
The key to binding both lvalues and rvalues is using universal reference. Note though than we must use `std::forward` because we're ignorant of the type of the `Containter` whether it's lvalue or rvalue.

```cpp
// final C++11 version
template<typename Container, typename Index>
auto logAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i])
{
    logAndAccess();
    return std::forward<Container>(c)[i];
}
```

```cpp
// final C++14 version
template<typename Container, typename Index>
decltype(auto) logAndAccess(Container&& c, Index i)
{
    logAndAccess();
    return std::forward<Container>(c)[i];
}
```

And here's one final remark on the `decltype`. For lvalue expressions more complicated than names, `decltype` ensures that the type returned is always an lvalue reference. When we have a declared variable `int x = 12;`. Then this one `decltype(x)` gives us `int`, BUT this one `decltype((x))` is `int&`. So be careful when writing function like this:

```cpp
decltype(auto) foo()
{
    int x = 12;
    // ...
    return (x);     // you will get a dangling reference here
}
```

# Item 4: Know how to view deduced types.
Maybe I will write this one later.



---
# CHAPTER 2. `auto`
---
# Item 5: Prefer `auto` to explicit type declarations.
Consider 3 examples. A simple `int x;` declaration, which doesn't garantee you any initial values of the `x`. A correct way to declare a type of the value pointed by an iterator: `typename std::interator_traits<It>::value_type val = *it;` or even just declaring an iterator like `std::vector<int>::const_iterator = v.cbegin();`. And the last one is declaring a variable of a closure type, which is simply impossible, cause this type is known to the compiler only.
This cases are easily dealt with `auto`.

```cpp
auto x;         // compilation error, initializer required
auto y = 12;    // fine

template <typename It>
void foo(It it)
{
    auto derefIt = * it; // addition space after asterisk is for Atom to parse the Markdown text correctly, sorry
}
auto strLess = [](std::shared_ptr<Widget> const& lhs, std::shared_ptr<Widget> const& rhs)
{
    return * lhs < * rhs;
};
// in C++14 we can even declare lambda parameters auto
auto strLess14 = [](auto const& lhs, auto const& rhs)
{
    return * lhs < * rhs;
};
```
Why is it better to use an `auto`-declared lambda variable instead of a `std::function`? First, when you declare a `std::function`, you must type the correct type cause it's a template.

```cpp
std::function<std::shared_ptr<Widget> const&, std::shared_ptr<Widget> const&>
strFunc =
[](std::shared_ptr<Widget> const& lhs, std::shared_ptr<Widget> const& rhs)
{
    return * lhs < * rhs;
};
```
Second, `auto` variable has the exact same type as the closure, while a `std::function` is a more general concept, so it may in some cases allocate additional memory. And finally, calling a function through the `std::function` template is generally slower due to indirect function calls.

Another useful case for `auto` is when working with those dependent integral types like `std::vector<...>::size_type`. Some developers just use `int` or `size_t` like this:

```cpp
int isz = vctr.size();
size_t sz = vctr.size();
```
Both are incorrect and may lead to port errors. Declaring an `auto` variable on the other hand avoids such problems.

One more point for the `auto` is this code.

```cpp
std::map<std::string, int> mp;
for (std::pair<std::string, int> const& p : m) { ... }
```
The problem here is that in fact the type of an item inside the container is `std::pair<std::string const, int>`. That little constness of the key causes the code above to make a copy of every item while iterating over the map. We could avoid this if we used `auto`.

```cpp
for (auto const& p : m) { ... }
```


# Item 6: Use the explicitly typed initializer idiom when `auto` deduces undesired types.
There are some cases where you should use `static_cast`.



---
# CHAPTER 3. Moving to Modern C++
---
# Item 7: Distinguish between `()` and `{}` when creating objects.
C++11 introduces the concept of *uniform initialization*, which is brought to life by *braced initialization* syntactic construct.

```cpp
std::vector<int> v { 1, 3, 5 }; // initialize with values 1, 3, 5
struct Widget {
    int x { 0 };    // fine, x's default value is 0
    int y = 0;      // also fine
    int z(0);       // error!
};
```
Uncopyable objects may be initialized using braces.

```cpp
std::atomic<int> ai1 { 12 };    // fine
std::atomic<int> ai2(12);       // fine
std::atomic<int> ai3 = 12;      // error!
```
Braced initialization prohibits implicit *narrowing conversions* while old-school variants don't.

```cpp
float x, y, z;
int s1 { x + y + z };    // error
int s2(x + y + z);       // fine
int s3 = x + y + z;      // fine
```
Another advantage of braced initializers is it's immunity to *most vexing parse*. This is the effect where compiler interpretes a variable initialization as a function declaration.

```cpp
Widget w1();    // declares a function returning a Widget
                // though we wanted just to use a default constructor
Widget w2{};    // calls constructor with no arguments
```
However there's a drawback with braced initializers. It's related to constructor overload rules. When a class has a constructor with `std::initializer_list` argument, the compiler always tries to pick it when it sees braced initialization. It's a such strong desire that an error is generated when there's no way to convert the parameters to the mentioned list object.

```cpp
struct Widget {
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);
    
    operator float() const;     // convert to float
};

Widget w1(10, true);    // parens, calls first ctor
Widget w2 { 10, true }; // braces, calls std::initializer_list ctor
                        // (10 and true convert to long double)
Widget w3(10, 5.0);     // parens, calls second ctor
Widget w4 { 10, 5.0 };  // braces, calls std::initializer_list ctor
                        // (10 and 5.0 convert to long double)
```
This behaviour can even brake copy-ctors.

```cpp
Widget w5(w4);      // parens, calls copy ctor
Widget w6 { w4 };   // braces, calls std::initializer_list ctor
                    // (w4 converts to float, and float converts to long double)
```
Even if you use a move ctor with brace initialization, you will get the same behaviour as in `w6`.
And here's an example with an error.

```cpp
struct Widget {
    Widget(int i, double d);
    Widget(std::initializer_list<bool> il);
};

Widget w1 { 10, 5.0 };  // error, requires narrowing conversion
```
If the `initializer_list` contained `std::string` objects, than there would be no way to convert and the code would compile, successfully calling the first constructor.
One subtle detail worth mentioning. Empty braces means no arguments, not empty initializer list, so the following code works like this.

```cpp
struct Widget {
    Widget();
    Widget(std::initializer_list<int> il);
};

Widget w1{};    // calls default ctor
Widget w2({});  // calls second ctor with empty list
Widget w3({});  // calls second ctor with empty list
```
This topic is crucial for template class designers. In template code you can't know which classes will be used as a template parameter. So usually it's a "by design" decision to use braces or parens. For example, `std::make_unique` and `std::make_shared` use parentheses internally and document this decision.


# Item 8: Prefer `nullptr` to `0` and `NULL`.
Neither `0` nor `NULL` are integral values. `0` is an `int`, `NULL` is allowed to be other integral type than `int`. In certain contexts, compiler considers them as a null pointers though, but in C++11 we have a better option.
There is a guideline for C++98 programmers to avoid overloading on pointer and integral arguments.

```cpp
void f(int);
void f(bool);
void f(void*);
f(0);           // calls f(int), not f(void*)
f(NULL);        // might not compile, but typically calls
                // f(int). Never calls f(void*)
```
If `NULL` is defined as `0L`, than both `int` and `bool` versions are equal conversions, so there's even an ambiguity.
The actual type of `nullptr` is `std::nullptr_t`, which implicitly converts to all pointer types. So calling `f(nullptr)` gives us the desired invokation of the pointer version.
Another advantage is more obvious conditional statements, especially when `auto` is involved.

```cpp
auto res = findWhatIneed(...);
if (result == 0) { ... }        // not so clear what type findWhatIneed returns
if (result == nullptr) { ... }  // this one looks much better
```
Template functions are another case where `nullptr` is a better choice.

```cpp
template <typename FuncType, typename PtrType>
auto doSmthAndCall(FuncType func, PtrType ptr) -> decltype(func(ptr))
{
    // some code like locking a mutex
    return func(ptr);
}
// and then we have function like this
float calculate(std::shared_ptr<Widget> w);

doSmthAndCall(calculate, 0);        // compilation error
doSmthAndCall(calculate, NULL);     // compilation error
doSmthAndCall(calculate, nullptr);  // fine
```
The problem here is compiler deduces integral type for both `0` and `NULL`, which it can't convert to `std::shared_ptr` as is the case with `nullptr` .


# Item 9: Prefer `alias` declarations to `typedef`s.
`typedef`s are a convinient way to type less code. C++11 though offers a *alias declarations*.

```cpp
// C++98 way
typedef std::shared_ptr<std::map<std::string, std::string>>  DictPtr;
// C++11 way
using DictPtr = std::shared_ptr<std::map<std::string, std::string>>;
```
Here are some reasons to prefer the aliases instead of typedefs. First, it's more readable when used for function pointers.

```cpp
typedef void (*CopyFP)(int, std::string const&);

using CopyFP = void (*)(int, std::string const&);
```
A more significant reason is templates. It is not possible to templatise a `typedef` but you can do the following.

```cpp
template <typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> list;
```
Here's a C++98 way to do this.

```cpp
template<typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};
MyAllocList<Widget>::type list;
```
If you use this construct inside a template for a member, you will have to precede it with a `typename` word.

```cpp
template <typename T>
struct Widget {
    typename MyAllocList<T>::type list;     // more typing here
};
```


# Item 10: Prefer *scoped enums* to *unscoped enums*.
C++98 style `enum` declaration polutes the namespace, because the contained names all get to the `enum`'s scope. Hense the name *unscoped enums*.

```cpp
enum Color { black, white, red };
auto white = false;                 // error. white already declared
```
C++11 offers an alternative - *scoped enums*.

```cpp
enum class Color { black, white, red };
auto white = false;                     // okay now
Color color = Color::black;             // here's how you address those
color = white;                          // error, types differ
```
Scoped enums are much more strongly typed. You can do this with old-style enums:

```cpp
enum Color { black, white };
int power(int value, int pow);  // return the power of a value

Color c = black;
if (c > 0.1) {
    auto pow = power(c, 2);
    // ...
}
```
Writing a simple `enum class` will make the compiler yell at you madly. Although you can use `static_cast`s and this code will be legal with them.
Another property of the scoped enums is they have the default underlying integral type set to `int`. So you can forward declare them without explicitly writing the type.

```cpp
enum class Color;                   // fine
enum class Color : std::uint8_t;    // fine
enum Color98 : int;                 // fine
enum Color98e;                      // error
```
C++11 enums though are not very convinient when working with tuples.

```cpp
using UserInfo = std::tuple<std::string, std::string, float>;
// username, email and raiting
UserInfo uInfo;
// ...
auto val1 = std::get<1>(uInfo);  // not very readable

enum UserInfoFields { uiName, uiEmail, uiRating };
auto val2 = std::get<uiName>(uInfo);    // much more clear now

enum class UserInfoFieldsC11 { uiName, uiEmail, uiRating };
auto val2 = std::get<static_cast<std::size_t>(UserInfoFieldsC11::uiName)>(uInfo);
// is that the code you prefer to write?
```
There is a solution though. You can have a function to convert any enum variable to it's underlying type value. But this function must be `constexpr`.

```cpp
template <typename Enum>
constexpr typename std::underlying_type<Enum>::type conv(Enum enumval) noexcept
{
    return static_cast<typename std::underlying_type<Enum>::type(enumval);
}
```
In C++14 we can get rid of the `::type` by using `underlying_type_t` and replace the return value with `auto`.

```cpp
auto val = std::get<conv(UserInfoFieldsC11::uiEmail)>(uInfo);
```


# Item 11: Prefer deleted functions to private undefined ones.
You remember the C++98 way to prevent users of your class to call certain functions like copy constructors and assign operators? Usually they are just defined private.
Some code (member functions or class friends) still can call those, and you will get linker error in case the functions are not implemented. C++11 offers a better solution - *deleted funcitons*.

```cpp
struct Widget {
    Widget(Widget const&) = delete;
    Widget const& operator=(Widget const&) = delete;
};
```
Deleted functions cause compile time errors if they are called somewhere. For this to work they are usually declared public.

An interesting additional profit is that any funciton may be deleted. For example, you would like to avoid implicit type conversions. `delete` is your friend then.

```cpp
bool check(int number);
bool check(char) = delete;      // reject chars
bool check(bool) = delete;      // reject bools
bool check(double) = delete;    // reject doubles and floats
```
Another useful case is deleting certain template specifications.

```cpp
template <typename T>
void doPointerMath(T* ptr);

template <>
void doPointerMath<void>(void*) = delete;     // no math with void*
```
That can be also done inside class. While the old C++98 way doesn't work here, cause you can't give different visibility to template member function and it's instantiation.

```cpp
struct Widget {
    template <typename T>
    void doPointerMath(T* ptr) { ... }
};

template <>
void Widget::doPointerMath<void>(void*) = delete;
```


# Item 12: Declare overriding functions override.
*Overriding* is a mechanism that makes it possible to call derived class function through a base class interface. In order for it to work
* base class function must be virtual
* base and derived class functions' signature must be identical. That is function names, paremeter types, constness must be identical, return types and exception specifications must be compatible (?).
* functions' reference qualifiers must be identical.
Reference qualifiers are a C++11 feature. They are used to make a function callable for l-value or r-value object only.

```cpp
struct Widget {
    void doSmth() &;    // called on l-values only
    void doSmth() &&;   // called on r-values only
};

Widget newWidget();

Widget w;
w.doSmth();             // calls the & version
newWidget().doSmth();   // calls the && version
```
So consider the following example. Many things done wrong here.

```cpp
struct Foo {
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f4() &;
    void f4() const;
};

struct Bar : public Foo {
    virtual void f1();                  // incorrect constness
    virtual void f2(unsigned int x);    // wrong argument type
    virtual void f3() &&;               // wrong referenceness
    void f4() const;                    // base function isn't virtual
};
```
To avoid such errors just add the `override` keyword at the end of the overriding function signature. It's not necessary to use `virtual`. FYI, `override` is a *contextual keyword*. It means if old code used this word as a variable name, it won't break, cause the word becomes a keyword only at the end of a member function declaration.


# Item 14: Declare functions `noexcept` if they won’t emit exceptions.
C++98 exception specifications are deprecated. And instead of them C++11 offers a new way to declare function's exception garantees: the `noexcept` word. It is a part of the function's interface. The client code can query `noexcept` status of a function and use different paths of execution.
Another advantage of `noexcept` is that compilers can generate better code. This is because if an exception leaves `noexcept` function, the exception specification is violated, and the stack is (only) *possibly* unwound. Compilers need not to keep the stack in unwindable state, or ensure the correct inverse deletion order of the objects inside the function.

In C++98 such functions as `std::vector::push_back` have a strong exception garantee. If the vector needs resizing, it allocates new memory chunk and copies elements one by one. If exception is thrown during copying, the original vector still holds valid data. Replacing the copies with moves in C++11 is only possible when move constructors are declared `noexcept`. If it weren't so, exception thrown while moving the center element of a vector would leave the vector in a wrong state. There's also `std::swap` function, which is declared `noexcept` conditionaly.

```cpp
template <class T1, class T2>
struct pair {
    // ...
    void swap(pair& p) noexcept(noexcept(swap(first, p.first))
        && noexcept(swap(second, p.second)));
    // ...
};
```
One important thing about C++11 is that by default all memory deallocation functions and decstructors are `noexcept`, unless some member's destructor is declared `noexcept(false)`.

For the sake of back-compatibility the compilers usually allow `noexcept` functions to call functions that are not declared `noexcept`. The called functions may be from a C library for example and alas have no exception specification.


# Item 15: Use `constexpr` whenever possible.
The new keyword `constexpr` specifies that the value of a variable or function can appear in *constant expressions*. That is expressions that can be evaluated during compilation. Such values can be used as non-template arguments, array sizes, enumerator values, alignment specifiers, etc.
For objects it implies `const` and for functions - `inline`.

`constexpr` functions produce compile-time known values when they are gived compile-time constant expressions as arguments. Otherwise they just work as usual functions.

```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
constexpr auto N = 3;
std::array<bool, pow(2, N)> switches;   // use with compile-time constants

auto base = readBaseFromSocket();
auto exp = readExpFromSocket();

auto result = pow(base, exp);       // use with any value as a usual function
```
In C++11 `constexpr` functions must be one-liners, consisting of return statement only. `static_assert`s, `typedef`, `using` declarations and directives are also allowed. C++14 removes the requirement of one line. C++11 also doesn't count `void` as a literal type whilst C++14 does.

`constexpr` variables must have literal type (essentially type that can have values known during compilation) and immediately initialized with an expression must be a constant expression. User-defined types can be literal types too because constructors can be `constexpr`. For that to be true, every base class and every non-static member must be initialized, either in the constructors initialization list or by a member brace-or-equal initializer. In addition, every constructor involved must be a `constexpr` constructor and every clause of every brace-or-equal initializer must be a constant expression (C++11). It also may be deleted or defaulted.

```cpp
struct Point {
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
        : x(xVal), y(yVal)
        { }
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }

private:
    double x, y;
};

constexpr Point p1(3.14, 3.14);     // runs constexpr constructor
constexpr Point p2(2.1828, 2.1828); // this one too

constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
    // call constexpr member-functions
    return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 };
}
constexpr midP = midpoint(p1, p2);  // calculated during compilation
```
When using C++14 we can even declare `setX` and `setY` as `constexpr`, so functions like this are legit.

```cpp
// C++14 example
constexpr Point reflect(Point const& p) noexcept
{
    Point result;
    result.setX(-p.xValue());
    result.setY(-p.yValue());
    return result;
}
```
It is worth mentioning that `constexpr` variables can be placed in the constant memory region, so they are important for embeded programming.


# Item 16: Make `const` member functions thread safe.
The key point of this item is in the title. You know, that `const` functions are considered not to change the class members. But there is a `mutable` keyword, which allows some changes to occur even in `const` functions. From the point of view of the class user, `const` means no changes, so when you use `mutable` variables and the function may be called from different threads, use some synchronization mechanism.


# Item 17: Understand special member function generation.
Special member functions are the ones being generated by the compiler automatically. They are default constructor, destructor, copy construct, assignment operator, move constructor and move assignment operator. They are implicitly `public` and `inline` and non-virtual, unles it's a destructor in derived class, whose base class has a virtual destructor.

* Default constructor is generated if the class doesn't have any constructors (C++98 rules apply).
* Default destructor has same rules as in C++98 with the addition of `noexcept` word for them.

The 2 new special functions of C++11 - the move ones are generated when the following conditions apply.
* No copy operations are declared in the class.
* No move operations are declared in the class.
* No destructor is declared in the class.
C++11 deprecates the automatic generation of copy operations for classes declaring copy operations or a destructor, so at some point similar rules may be applied to the copy functions.
Currently though copy operations don't deny each other. Move operation also disable copy operations generation.

If the copy operations get new rules of generation later, you can fix old code easily by using the `= default` construct. It tells the compiler to generate the default version of a function.

```cpp
struct Widget {
    Widget();
    Widget(Widget const&) = default;
    Widget const& operator=(Widget const&) = default;
};
```
In fact you should always use `= default` to declare explicitly your intention to make the compiler generate special functions.

One interesting thing is that member function templates do not prevent compilers from generating specials.

```cpp
// copy and move operations still generated
struct Widget {
    template <typename T>
    Widget(T const& rhs);
    
    template <typename T>
    Widget& operator=(T const& rhs);
};
```


---
# CHAPTER 4. Smart pointers
---


















