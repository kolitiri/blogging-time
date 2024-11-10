---
title: Custom parametrization scheme with Pytest
description: >-
  Improving the parametrization functionality in Pytest
date: 2024-10-16 18:00:00 +0100
categories: [Python]
tags: [python,pytest]
tok: true
image:
  path: /assets/img/illustrations/parametrisation.jpg
  alt: Parametrisation
---

---
[Pytest](https://docs.pytest.org/en/7.1.x/contents.html) is a great tool as it offers tones of functionality for writing tests quickly and reliably.

One of the features I love the most is that it allows you to easily parametrize your tests and run them against multiple scenarios.

Nevertheless, I can't say I am a huge fan of the default syntax required to achieve that and I will explain the reason in a bit.

## Parametrizing tests

Parametrizing tests is really easy. The [documentation](https://docs.pytest.org/en/7.1.x/example/parametrize.html) provides a nice example to demonstrate it in action.

```python
from datetime import datetime, timedelta

@pytest.mark.parametrize(
    "a,b,expected",
    [
        (datetime(2001, 12, 12), datetime(2001, 12, 11), timedelta(1)),
        (datetime(2001, 12, 11), datetime(2001, 12, 12), timedelta(-1)),
    ]
)
def test_timedistance_v0(self, a, b, expected):
    diff = a - b
    assert diff == expected
```

Running this snippet using the `-v` option gives us a nice breakdown of the scenarios that were ran for this single test.

```bash
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[a0-b0-expected0] PASSED                                        [ 50%]
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[a1-b1-expected1] PASSED                                        [100%]

========================================================= 2 passed in 0.02s =========================================================
```

We can clearly see that the same test `test_timedistance_v0` was ran twice, against two different sets of parameters.

An interesting thing to notice is that Pytest attaches some sort of ids (`a0-b0-expected0`, `a1-b1-expected1`) to each scenario, which don't make much sense..

But you can work around it by simply adding your own ids, which could very much be a string describing each scenario.

```python
from datetime import datetime, timedelta


class TestSampleWithScenarios:

    @pytest.mark.parametrize(
        "a,b,expected",
        [
            (datetime(2001, 12, 12), datetime(2001, 12, 11), timedelta(1)),
            (datetime(2001, 12, 11), datetime(2001, 12, 12), timedelta(-1)),
        ],
        ids=["forward", "backward"]
    )
    def test_timedistance_v0(self, a, b, expected):
        diff = a - b
        assert diff == expected
```

Now the result is much better!

```bash
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[forward]  PASSED                                               [ 50%]
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[backward] PASSED                                               [100%]

========================================================= 2 passed in 0.02s =========================================================
```

## When the OCD kicks in

This default parametrization is a great way to reuse your tests, however, one particular thing that I personally dislike is the fact that it makes it hard to figure out which value corresponds to which argument and which id relates to which scenario.

Oof that's a mouthful.. All I am trying to say is, imagine how this syntax would look if you had 20 different scenarios, each with 10 arguments.

I'll show you..

```python
@pytest.mark.parametrize(
    "a,b,c,d,e,f,g,h,i,j",
    [
        (1, 2, 3, 4, 5, 6, 7, 8, 9, 10),
        (11, 12, 13, 14, 15, 16, 17, 18, 19, 20),
        (21, 22, 23, 24, 25, 26, 27, 28, 29, 30),
        (31, 32, 33, 34, 35, 36, 37, 38, 39, 40),
        ...
    ],
    ids=["Test1", "Test2", "Test3", "Test4", ...]
)
def test_timedistance_v0(a, b, c, d, e, f, g, h, i, j):
    ...
```

Obviously that's a bad example, but you get it, right?

If I want to find the parameters used in scenario "Test4", I have to count the 4th line of the list.

Furthermore, If I need to check say argument "g", I have to count the 7th element of the tuple.. Yikes!

At that point, I might as well conveniently "forget" to write the test..

## The solution

Luckily, a while ago, the guys from [@pytestdotorg](https://x.com/pytestdotorg) suggested to me a neat solution.

And that was to implement my own [pytest_generate_tests](https://docs.pytest.org/en/stable/how-to/parametrize.html#basic-pytest-generate-tests-example) hook.

You can play around with the hook, but my implementation looks something like the one below.

```python
import pytest


def pytest_generate_tests(metafunc):
	""" Custom parametrisation scheme """
	function_name = metafunc.function.__name__
	function_scenarios = getattr(metafunc.cls, f"{function_name}_scenarios")
	function_params = [key for key in function_scenarios[0]]
	function_values = [scenario.values() for scenario in function_scenarios]
	ids_list=[sc.get("description") for sc in function_scenarios]

	metafunc.parametrize(function_params, function_values, ids=ids_list, scope="class")
```

The idea is that for every test named `test_x_functionality`, we can define a list named `test_x_functionality_scenarios` to provide the parameters for the test.

Not a huge fan of the idea of relying on naming conventions, but oh well, a small price to pay.

Let's see it in action.

```python
class TestSampleWithScenarios:

    test_timedistance_v0_scenarios = [
        dict(
            a=datetime(2001, 12, 12),
            b=datetime(2001, 12, 11),
            expected=timedelta(1),
            description="forward",
        ),
        dict(
            a=datetime(2001, 12, 11),
            b=datetime(2001, 12, 12),
            expected=timedelta(-1),
            description="backward",
        ),
    ]

    def test_timedistance_v0(self, a, b, expected, description):
        diff = a - b
        assert diff == expected
```

Now, all the parameters for a particular scenario are grouped under the same `dict` as key/value pairs and we also get to add our description!

And most importantly, the result is still the same!

```bash
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[forward]  PASSED                                               [ 50%]
parametrised.py::TestSampleWithScenarios::test_timedistance_v0[backward] PASSED                                               [100%]

========================================================= 2 passed in 0.02s =========================================================
```

## Conclusion

Pytest can work wonders and in this case the `pytest_generate_tests` hook can help you improve the readability of your tests massively.

It can get funny if you accidentally mess up with the hook implementation or if you don't adhere to the naming conventions, but as I said earlier, I think it is a small price to pay.

I usually group my tests in classes to narrow down this possibility even further and never had issues so far.

Do you have any better suggestions? Let me know :wink:
