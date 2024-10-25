---
title: Functional VS Unit tests
description: >-
  A short demonstration of the differences between functional and unit tests
date: 2024-10-24 21:00:00 +0100
categories: [Python]
tags: [python]
tok: true
---

---
Functional and unit tests are used widely in software development and arguably, they should be mandatory..

They are both a great way to lock your functionality and assess that it is still working as expected when changes are introduced.

However, the scope of a functional test is usually wider than the scope of a unit test.

Let's explore this using an example.

Assume that we have a program that takes a list of raw account balances from an external source and does the following before adding them into our internal system:
- Converts the balances to USD
- Converts the epoch timestamp to a readable date-time

```python
from datetime import datetime
from typing import Any, List, Dict


CURRENCY_EXCHANGE = {
    "USD": 1,
    "EUR": 0.9,
    "GBP": 0.8,
}


def convert_to_usd(amount: float, currency: str) -> float:
    return amount * CURRENCY_EXCHANGE[currency]


def convert_to_date(timestamp: int) -> str:
    return str(datetime.fromtimestamp(timestamp))


def cleanup_transactions(transactions: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    for transaction in transactions:
        transaction["balance"] = convert_to_usd(
            transaction["balance"], transaction["currency"]
        )
        transaction["currency"] = "USD"
        transaction["timestamp"] = convert_to_date(transaction["timestamp"])

    return transactions


if __name__ == "__main__":
    transactions = [
        {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": 1729774487},
        {"name": "Clark Kent", "balance": 2000, "currency": "EUR", "timestamp": 1729788887},
        {"name": "Peter Parker", "balance": 500, "currency": "GBP", "timestamp": 1729789007},
    ]

    print(cleanup_transactions(transactions))
```

If we run the `cleanup_transactions` function with some test data we get the following output:
```bash
[
  {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": "2024-10-24 13:54:47"},
  {"name": "Clark Kent", "balance": 1800.0, "currency": "USD", "timestamp": "2024-10-24 17:54:47"},
  {"name": "Peter Parker", "balance": 400.0, "currency": "USD", "timestamp": "2024-10-24 17:56:47"}
]
```

The output looks as expected, so naturally, we'd like to write some tests to protect ourselves against future changes.

## Functional tests
A functional test, as the name suggests, is supposed to be testing a specific functionality of the system.

It usually shouldn't care about low implementation details meaning that in the background it could be testing multiple components.

In our example above, we can write a functional test that checks the overall functionality of the `cleanup_transactions` function.

```python
from example import cleanup_transactions


def test_cleanup_transactions():
    transactions = [
        {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": 1729774487},
        {"name": "Clark Kent", "balance": 2000, "currency": "EUR", "timestamp": 1729788887},
        {"name": "Peter Parker", "balance": 500, "currency": "GBP", "timestamp": 1729789007},
    ]

    expected_transactions = [
        {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": "2024-10-24 13:54:47"},
        {"name": "Clark Kent", "balance": 1800.0, "currency": "USD", "timestamp": "2024-10-24 17:54:47"},
        {"name": "Peter Parker", "balance": 400.0, "currency": "USD", "timestamp": "2024-10-24 17:56:47"}
    ]

    actual_transactions = cleanup_transactions(transactions)

    assert actual_transactions == expected_transactions
```

If you notice, a test like this will not only test the `cleanup_transactions` but also the functions that are called within it, `convert_to_usd` and `convert_to_date`.

And this is the main reason why it is called a functional test.

## Unit tests
A unit test, on the other hand, is supposed to be testing the smallest units of your system, such as a method.

These tests should completely isolate the function to be tested by stubbing or mocking any external calls.

In our example above, we can write a unit test that strictly checks the functionality of the `cleanup_transactions`, assuming that `convert_to_usd` and `convert_to_date` work as expected.

```python
from example import cleanup_transactions

from unittest.mock import patch


def test_cleanup_transactions():
    transactions = [
        {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": 1729774487},
        {"name": "Clark Kent", "balance": 2000, "currency": "EUR", "timestamp": 1729788887},
        {"name": "Peter Parker", "balance": 500, "currency": "GBP", "timestamp": 1729789007},
    ]

    expected_transactions = [
        {"name": "Bruce Wayne", "balance": 1000, "currency": "USD", "timestamp": "2024-10-24 13:54:47"},
        {"name": "Clark Kent", "balance": 1800.0, "currency": "USD", "timestamp": "2024-10-24 17:54:47"},
        {"name": "Peter Parker", "balance": 400.0, "currency": "USD", "timestamp": "2024-10-24 17:56:47"}
    ]

    with (
        patch("example.convert_to_usd", side_effect=[1000, 1800.0, 400.0]),
        patch("example.convert_to_date", side_effect=["2024-10-24 13:54:47", "2024-10-24 17:54:47", "2024-10-24 17:56:47"])
    ):
        actual_transactions = cleanup_transactions(transactions)

    assert actual_transactions == expected_transactions
```

This test might appear to be very similar to the functional test above but there is actually a huge difference.

The functions `convert_to_usd` and `convert_to_date` are not called anymore, but rather mocked to return some expected values.

You can easily confirm that by adding a breakpoint or a print statement inside these functions, and you will see that they are not executed at all.

## Functional VS Unit tests
The biggest drawback I've noticed when writing functional tests is that the deeper they get the harder it is to identify the issue when your test is breaking.

Looking at the same example above, if we were to change the CURRENCY_EXCHANGE for GBP from `0.8` to `0.9` and run our functional test again, we would get a nasty error like this:

```bash
>       assert actual_transactions == expected_transactions
E       AssertionError: assert [{'balance': ...24 17:56:47'}] == [{'balance': ...24 17:56:47'}]
E
E         At index 1 diff: {'name': 'Clark Kent', 'balance': 1800.0, 'currency': 'USD', 'timestamp': '2024-10-24 17:54:47'} != {'name': 'Clark Kent', 'balance': 1800.0, 'currency': 'EUR', 'timestamp': '2024-10-24 17:54:47'}
E         Use -v to get more diff
```

It's fairly easy to understand that the `actual_transactions` don't match the `expected_transactions`, but there is no easy way to identify that the `convert_to_usd` is the function responsible.

You would have to compare the two values, notice that the balances don't match, figure out who is responsible for converting the balances and finally conclude that it has to be something wrong with the `convert_to_usd` function.

Now, obviously this is a tiny example, but you can easily see how frustrating this can be if you have a much deeper and more complex functional test.

On the other hand, the unit test we wrote above will not be affected because it doesn't really execute `convert_to_usd` at all.

It simply assumes that it works as expected, because in theory it should have its own unit test responsible for checking that.

## Conclusion
Both functional and unit tests have their own advantages and disadvantages.

Functional tests are easier to write but harder to debug when they break.

Unit tests are easier to debug, but harder to write due to all the stubbings/mockings involved.

In my opinion, every function should have unit test coverage, ideally testing the most critical paths (if not all).

Functional tests should be kept to a minimum and when used they should be written in a clear and documented way to help future developers debug them when the time comes.

Let me know what you think!
