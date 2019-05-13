# Pattern Searching with Boyer-Moore algorithm

C++17 allows pattern searching applying different algorithms.

#### std::search

The algorithm is used to search for the first occurence of a sequence of elements withing another sequence of elements.
C++17 offers updated api:

- Use execution policy to run the default algorithm search in parallel
- Provide _searcher_ object that handles the search

Available searchers are

- default_searcher
- boyer_moore_searcher
- boyer_moore_horspool_searcher

#### Preprocessing

Both Boyer-Moore algorithms use preprocessing on the searched pattern, the complexity of that step depends on the size of the alphabet of the string.
Horsepool is a simplified version of Boyer-Moore. Average complexity is *O(n)*, but worst case is *O(nm)*.

#### Example

```cpp
std::string searchIn = "A very long and various characters containing string";
std::string searchThis = "var";

auto it = std::search(searchIn.begin(), searchIn.end(), std::boyer_moore_searcher(searchThis.begin(), searchThis.end()));
if (it != searchIn.end())
{ /*we found it!!*/ }
```

#### Notice

As the algorithm and searchers are templated and accept iterators, they can work not only with strings, but with arbitrary types that have equality operator defined. Or a binary comparison predicate if that version of search is used.