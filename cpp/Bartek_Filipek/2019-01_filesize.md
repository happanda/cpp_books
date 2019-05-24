# Getting file size with `std::filesystem`

[Original post](https://www.bfilipek.com/2019/01/filesize.html)

Old-school ways to get file size included using 3rd party libraries, system-specific APIs or even opening a file and getting the size from position pointer. C++17 simlifies working with file system through `std::filesystem`.

#### File size
There are 2 ways to get a file size:
- `std::uintmax_t std::filesystem::file_size( const std::filesystem::path& p );`
- `std::uintmax_t std::filesystem::directory_entry::file_size() const;`
And their respective `noexcept` versions, which return an error through inout parameter. First one is a free function, second is a method in `directory_entry`.
The member function utilizes caching of `directory_entry` class, hence may be faster in certain scenarios. To refresh the cache one can call `directory_entry::refresh()`.



```cpp
try {
	std::filesystem::file_size("file.ext");
} catch (std::filesystem::filesystem_error& ex) {
	/* ... */
}

std::error_code ec;
auto size = std::filesystem::file_size("file.ext", ec);
 // can also use operator bool on ec
if (ec == std::error_code{})
	/* all's fine */
else
	/* error happened, check ec.message(), ec.value(), ec.category */
```