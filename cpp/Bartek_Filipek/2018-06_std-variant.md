# `std::variant`

[Original post](https://www.bfilipek.com/2018/06/variant.html)

There's a limited use for `union`. Some low-level work. For instance, getting parts of a 64 bit integer.

```cpp
union IntegerParts
{
    int32_t m_Number;
    struct
    {
        uint8_t m_Part0;
        uint8_t m_Part1;
        uint8_t m_Part2;
        uint8_t m_Part3;
    };
};
```

Or in the same manor accessing `x, y, z` values of a vector, stored as an array.

For a similar high level usage unions are unsuitable, because they don't track what type they contain, they don't know about lifetime of the contents, don't call destructors. C++17 offers `std::variant`.

#### Example

```cpp
std::variant<IOError, FormatError, CalculationError> GetError() const;

struct ErrorParser
{
    void operator()(cosnt IOError& ioerror) { ... }
    void operator()(cosnt FormatError& fmtError) { ... }
    void operator()(cosnt CalculationError& calcError) { ... }
};

...
auto error = GetError();
std::visit(ErrorParser{}, error);

if (std::get_if<IOError>(&error)) { ... }
else if (std::holds_aleternative<FormatError>(error)) { ... }

try
{
    auto calcError = std::get<CalculationError>(error);
    ...
}
catch (std::bad_variant_access& bva) { ... }
```

#### Features

- Type of inner data can be checked via `.index()` member function or `holds_alternative`.

- Access via `get_if` or `get` (this one might throw). Or use a visitor.

- Type safety

- Non-initialized `variant` will try to assign default-constructed value of the first type

- No extra heap allocations happen

- Destructors/constructors are called on inner content

#### Use cases

- Any property which might have value of different types (settings files, language parsers, command line parser)
- Expressing several outcomes of a computation
- Returning value or error
- State machines
- Polymorphism without vtables, using the visitor

#### Nuances

- As an empty alternative for `std::variant` there's `std::monostate` type. The following `variant` is thus default-constructible: `std::variant<std::monostate, NotConstructibleType>>`

- For explicit choice of the initializing inner type there's a `std::in_place_index` and `std::in_place_type`: `std::variant<int, std::string> var { std::in_place_index<1>, "Hello" };`

- `std::in_place` allows to create more complex types and more parameters to ctor.

#### Assignment

- Assignment operator
- `emplace`
- Get reference to inner type through `get`
- Visitor

#### Visitor

- `std::visit` accepts any callable and any number of variants. The callable must have overloads for inner types of all the passed variants.

- Visitors can change inner values of the variant.

#### More properties

- Two variants of same type can be compared. If the inner types are same, their comparison operator is called. Otherwise inner type indices are compared.

- Variant can be moved

- `std::hash` works with variant

#### Exceptions

If exception happens while `emplace` is called in the ctor of the new value, then variant becomes "valueless by exception" and it's `.valueless_by_exception()` returns true. And `.index()` return `variant_npos`.

#### Memory

Variant uses as much as the biggest inner type plus a little something to keep the index of the type. With respect to alignment rules.