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
















