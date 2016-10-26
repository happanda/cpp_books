# Preface
This document is my personal attempt to write a  series of notes following **"Effective modern C++"** by *Scott Meyers*. If you came across this document somewhere, I strongly recommend you to read the original text. Firstly, it is more detailed and contains less errors. Second, there are interesting examples and small wise tips, that I might have missed. Also some items from the book didn't make it here at all.

And one more thing: no grammar check has been made yet.


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




