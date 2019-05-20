# How to use `std::from_chars`

[Original post](https://www.bfilipek.com/2018/12/fromchars.html)

There's a number of string <-> number conversion utilities prior to C++17: `sprintf/sscanf` family, `atol`, `strtol`, `strstream`, `stringstream`, `stoi`, etc.

C++17 introduces 2 new mechanisms: `std::from_chars` and `std::to_chars`. In short they're the fastest.
- non-throwing
- non-allocating
- no locale support
- memory safety
- error reporting

#### From characters to numbers: `std::from_chars`
A set of overloaded functions for integral and floating types. Integral functions allow base to be in range [2, 36]; floating allow to set floating point format. Formats are *scientific*, *fixed*, *hex* and *general*.

All functions return a `std::from_chars_result` struct:
```cpp
struct from_chars_result {
    const char* ptr;
    std::errc ec;
};
```
Retun results are:
- Success. *ptr* at 1st char not matching the pattern or *last*, *ec* is default initialized
- Invalid conversion: *ptr* equals 1st, *ec* equals `std::errc::invalid_argument`. *value* unmodified
- Out of range: *ec* equals `std::errc::result_out_of_range*, *ptr* at 1st char not matching the pattern. *value* unmodified

#### Example
```cpp
#include <charconv>
/* ... */
std::string str { "1029384756" };
int value = 0;
const auto result = std::from_chars(str.data(), std.data() + std.size(), value);
if (result.ec == std::errc()) { /* ... */}
else if (result.ec == std::errc::invalid_argument) { /* ... */}
else if (result.ec == std::errc::result_out_of_range) { /* ... */}
```

#### Performance
These conversion utilities are made with performance in mind. They're supposed to be tools for a higher level API, a replacement for C-style old ones.

