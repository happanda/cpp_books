# How to iterate through directories in C++

There're several "3rd Party" solutions to file system access in C++: operating system API (WinAPI/POSIX API), Qt, Boost, Poco, etc.

In C++17 however there's a built-in library functionality `std::filesystem` for that.

```cpp
#include <filesystem>

namespace fs = std::filesystem;

const fs::path basePath = fs::current_path();

for (const auto& entry : fs::directory_iterator(basePath)) {
    const auto filenameStr = entry.path().filename().string();
    if (entry.is_directory()) {
        // work with directory
    }
    else if (entry.is_regular_file()) {
        // work with file
    }
}
```

Note that iteration order is not specified, but every entry is visited once. Special dot and double dot entries are skipped.

There's also a `recursive_directory_iterator` if you need one. It even has `depth()` function to check how deep you are.

Both iterators are input iterators. Remember that when trying to use them in algorithms.

