# `std::optional`

[Original post](https://www.bfilipek.com/2018/04/refactoring-with-c17-stdoptional.html)

`std::optional` is useful for cases when you want to return a value or nothing. For instance, in case of an error or impossibility of getting an answer (say, an equation doesn't have a root).

Lets refactor the following function (consider `float3` to be a 3D vector:

```cpp
bool FindClosestAvailablePosition(const float3& anchor, float3& result, float* distance);
```

- At each calling site we have to create variables to store the result

- Output parameters are not user-friendly

- If out parameters are pointers, additional null checks are needed

- Extending the function is cumbersome

### Refactoring

#### `std::tuple`

First attempt to replace out params with a return tuple is not the best approach. It's hard to remember parameters sequence. Extending the signature is still a problem.

#### Separate structure

```cpp
struct ClosestPosition
{
    float3 m_Position;
    float  m_Distance;
};
```

Now the original function can be written as

```cpp
std::pair<ClosestPosition, bool> FindClosestAvailablePosition(const float3& anchor);
```

Still using the `std::pair` is not the most pleasant activity.

#### `std::optional`

This struct in fact represents a pair of bool and a template argument type.
```cpp
std::optional<ClosestPosition> FindClosestAvailablePosition(const float3& anchor);
...
if (auto result = FindClosesAvailablePosition(anchor))
{
    // use result->m_Position
    // or *result.m_Position
}
```

And btw, optional doesn't make any dynamic memory allocations.
