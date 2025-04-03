---
title: Does syntax matter?
description: >-
  Exploring how the Python syntax can potentially affect performance
date: 2025-04-03 06:30:00 +0100
categories: [Python]
tags: [python,performance]
tok: true
image:
  path: /assets/img/illustrations/does-syntax-matter.jpg
---

---
It's well known that Python comes batteries included.

Actually, sometimes it might seem that it comes with more batteries than needed, meaning that it offers multiple alternatives for performing the same task.

For example, one could initialize a dictionary using either the ***dict()*** function, or the ***curly brackets {}***.

So what is the difference between such options and how do we choose which one to use?

# Does syntax matter?
In a nutshell, the differences between these various options usually boil down to:
- **Appearance**: How intuitive, compact or readable is the syntax
- **Implementation**: How performant is the underlying implementation; that is either in Python or in the C-extensions

Personally, whenever I face such a dilemma, I tend to use the "prettiest" option, because when it comes to Python, I value readability slightly more than performance.

After all, if I needed a blazing fast runtime, I wouldn't be using Python at the first place..

However occasionally, depending on the task, it might be deemed necessary to squeeze every little ounce of power out of the language.

Therefore, during these sad times, being aware of the nuances could prove handy.

# Let's see some examples
I have collected a few examples that showcase how a seemingly insignificant syntactic change could potentially improve the performance of your codebase.

Before jumping in, make sure you familiarize yourself with the [timeit](https://docs.python.org/3/library/timeit.html) module, to understand how it works.

These examples have been timed using C-Python 3.12.1 on a regular Linux machine, but keep in mind that some of the comparisons might not yield the same results in different Python versions, Python implementations or Operating Systems.

## Searching whether a string starts with a substring
If you need to search wether a string starts with a particular substring, use the ***startswith*** function instead of the ***in*** operator.

In the worst case scenario, when the substring will not exist in the original string, the ***in*** operator will still have to traverse the whole string, wasting extra time.

```python
import functools
import timeit


string = (
    f"Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor"
    f"minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex eas"
    f"oluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint oas"
    f"Sed ut perspiciatis unde omnis iste natus error sit voluptatem accusantium dol"
    f"m ad minima veniam, quis nostrum exercitationem ullam corporis suscipitas TEST"
)
substring = "TEST"


def search_with_startswith(string: str, substring: str) -> bool:
    if string.startswith(substring):
        return True
    return False


def search_with_in(string: str, substring: str) -> bool:
    if substring in string:
        return True
    return False


print(timeit.timeit(functools.partial(search_with_startswith, string, substring), number=10_000_000))
print(timeit.timeit(functools.partial(search_with_in, string, substring), number=10_000_000))
```

```bash
1.8934730000000854
3.051734000000579
```

## Creating an iterable for membership checks
If you are creating an iterable with the sole purpose of checking whether certain values are present within it, favor ***sets*** instead of ***lists***.

Due to their semantics, sets are faster than lists when looking up values.

```python
import functools
import timeit


my_list = [i for i in range(100)]
my_set = set(my_list)
number = 539


def search_in_set(_set: set, number: int) -> bool:
    if number in _set:
        return True
    return False


def search_in_list(_list: list, number: int) -> bool:
    if number in _list:
        return True
    return False


print(timeit.timeit(functools.partial(search_in_set, my_set, number)))
print(timeit.timeit(functools.partial(search_in_list, my_list, number)))
```

```bash
0.10908959999869694
1.4462419999981648
```

## Creating an iterable for iterating it
If you are creating an iterable with the purpose of iterating through all its values favor ***lists*** instead of ***sets***.

Again, due to their semantics, sets are slower than lists when iterating over their values (although in reality the improvement will be minor).

```python
import functools
import timeit


my_list = [i for i in range(100)]
my_set = set(my_list)
number = 539


def iterate_whole_list(_list: list):
    for i in _list:
        pass


def iterate_whole_set(_set: set):
    for i in _set:
        pass


print(timeit.timeit(functools.partial(iterate_whole_list, my_list), number=1_000_000))
print(timeit.timeit(functools.partial(iterate_whole_set, my_set), number=1_000_000))
```

```bash
0.8423409999995783
1.8966035000012198
```

## Creating a list
If you are creating a new list, use ***list comprehension*** instead of a ***for loop***.

```python
import timeit


def with_comprehension() -> None:
    my_list = [i for i in range(200) if i % 2 == 0]


def with_loop() -> None:
    my_list = []
    for i in range(200):
        if i % 2 == 0:
            my_list.append(i)


print(timeit.timeit(with_comprehension))
print(timeit.timeit(with_loop))
```

```bash
9.910756899997068
10.443588199999795
```

## Creating a dictionary
If you are creating a new dictionary, use ***dict comprehension*** instead of a ***for loop***.

```python
import timeit


def with_comprehension() -> None:
    my_dict = {i: i**2 for i in range(200)}


def with_loop() -> None:
    my_dict = {}
    for i in range(200):
        my_dict[i] = i**2


print(timeit.timeit(with_comprehension))
print(timeit.timeit(with_loop))
```

```bash
21.48162709999815
22.22446790000322
```

However, if you are relying on conditions, choose a ***for loop*** instead.

This might not always be true, but in case of a relatively complicated condition like below, a dictionary comprehension can significantly slow down the process.

```python
import timeit


def with_loop() -> None:
    my_dict = {}
    for i in range(100):
        if i % 2 == 0:
            my_dict[i] = "even"
        else:
            my_dict[i] = "odd"


def with_comprehension() -> None:
    my_dict = {(i, "even") if i % 2 == 0 else (i, "odd") for i in range(100)}


print(timeit.timeit(with_loop))
print(timeit.timeit(with_comprehension))
```

```bash
8.440671301999942
14.471410706000029
```

Further more, when creating a dictionary with value assignment, favor the ***dict()*** function instead of the ***curly braces***.

```python
import timeit


def with_dict() -> None:
    test = {'a': 1, 'b': 2, 'c': 3}


def with_curly_brackets() -> None:
    test = dict(a=1, b=2, c=3)


print(timeit.timeit(with_dict, number=100_000_000))
print(timeit.timeit(with_curly_brackets, number=100_000_000))
```

```bash
20.3870786999978
27.104615200001717
```

This is most likely due to the overhead of calling the ***dict()*** function and loading it into memory.

```python
import dis


def with_curly_brackets() -> None:
    test = dict(a=1, b=2, c=3)


dis.dis(with_curly_brackets)
```

```
4           0 RESUME                   0

5           2 LOAD_GLOBAL              1 (NULL + dict)
            14 LOAD_CONST               1 (1)
            16 LOAD_CONST               2 (2)
            18 LOAD_CONST               3 (3)
            20 KW_NAMES                 4
            22 PRECALL                  3
            26 CALL                     3
            36 STORE_FAST               0 (test)
            38 LOAD_CONST               0 (None)
            40 RETURN_VALUE
```

## Pattern matching
Use ***pattern matching*** instead of ***if-statements*** when you are interested in the pattern of the data.

Note, that in this case, I am not referring to a simple 1-1 value comparison, like in a traditional ***switch*** statement, but to actual pattern matching between two objects.

```python
import functools
import timeit


def with_pattern_matching(_list) -> str:
    match _list:
        case [_]: return "one"
        case [_, _]: return "two"
        case [_, _, _]: return "three"
        case _: return "more than three"


def with_if_statements(_list) -> str:
    if len(_list) == 1:
        return "one"
    elif len(_list) == 2:
        return "two"
    elif len(_list) == 3:
        return "three"
    return "more than three"


print(timeit.timeit(functools.partial(with_pattern_matching, ['a', 'b', 'c']), number=10_000_000))
print(timeit.timeit(functools.partial(with_if_statements, ['a', 'b', 'c']), number=10_000_000))
```

```bash
1.4475586999978987
1.8106488999983412
```

## Concatenating a list of strings
Use the ***join()*** function to concatenate string elements in a list, instead of the ***+*** operator.

```python
import functools
import timeit


_list = ["a", "b", "c", "d", "e"]

def use_join(_list: list) -> None:
    concat_list = "".join(_list)


def use_addition_operator(_list: list) -> None:
    concat_list = ""
    for i in _list:
        concat_list += i


print(timeit.timeit(functools.partial(use_join, _list), number=10_000_000))
print(timeit.timeit(functools.partial(use_addition_operator, _list), number=10_000_000))
```

```bash
2.0323625999990327
4.058531399998174
```

## Iterating over a list in reverse order
Finally, there are a number of ways to iterate a list in reverse order, but it appears that the fastest one is using the list's ***.reverse()*** function.

However, given that this will mutate the existing list instead of creating a new one, it might not be an option.

In this case, I would probably use either the ***reversed()*** function, or ***list slicing***, since I couldn't get concrete results when comparing the two..

I would certainly not use the java-ish approach since it's not only slow but also un-readable and un-pythonic.

```python
import timeit


def iterate_using_reverse() -> None:
    my_list = ['a', 'b', 'c', 'd']
    my_list.reverse()
    for i in my_list:
        value = i


def iterate_using_reversed() -> None:
    my_list = ['a', 'b', 'c', 'd']
    for i in reversed(my_list):
        value = i


def iterate_using_slice() -> None:
    my_list = ['a', 'b', 'c', 'd']
    for i in my_list[::-1]:
        value = i


def iterate_using_java() -> None:
    my_list = ['a', 'b', 'c', 'd']
    for i in range(len(my_list)):
        value = my_list[len(my_list)-1-i]


print(timeit.timeit(iterate_using_reverse, number=10_000_000))
print(timeit.timeit(iterate_using_reversed, number=10_000_000))
print(timeit.timeit(iterate_using_slice, number=10_000_000))
print(timeit.timeit(iterate_using_java, number=10_000_000))
```

```bash
2.320015199999034
3.6644386000007216
3.577267700002267
4.843100199999753
```

# Wrapping up
We have just demonstrated that simply altering our Python syntax could potentially award us with a performance boost.

However, I do believe that the examples above are not definitive enough to justify using one option over another.

Most often, the performance difference is going to be so small that choosing readability over performance would be wiser.

Lastly, I would like to point out that the purpose of this post is not to provide some concrete results, but to simply make you aware that things aren't always what they seem.

Thus, if the moment comes and you are called to "micro optimize" your code, please run your own tests in your own environment to make a solid decision.

And don't forget to use the **[dis](https://docs.python.org/3/library/dis.html)** package to dig deeper if necessary.

You can find the snippets above in this [Jupiter Notebook](https://github.com/kolitiri/blogging-time/blob/main/assets/notebooks/2025-04-03-does-syntax-matter.ipynb).
