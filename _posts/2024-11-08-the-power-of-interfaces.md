---
title: The power of interfaces
description: >-
  Using interfaces to achieve dependency inversion in our code
date: 2024-11-08 07:00:00 +0100
categories: [Software Design Principles]
tags: [python]
tok: true
image:
  path: /assets/img/illustrations/interfaces.jpg
  alt: Interfaces
---

---
Programming interfaces are essentially abstractions that define a set of method signatures without their actual implementation.

They are a form of contracts that define what kind of functionality is offered by a component, without revealing the way this functionality is achieved.

Interfaces are a great way to decouple parts of your code and make them depend on abstractions instead of implementation details.

This might not be the best way to describe them, but hopefully an example can clear things up.

## Simple is not always the best
Let's assume that your boss asked you to write a program that connects to a MongoDB node and returns a report with information about a specific user.

A very naive implementation of this program would be something like the one below.

```python
from typing import Any, Dict


class MongoClient:
    def find_one(self, query: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}


class Report:
    def __init__(self, client):
        self.client = client

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.client.find_one({"userId": user_id})
        return user


def get_report(user_id: int) -> Dict[str, Any]:
    client = MongoClient()
    report = Report(client)
    user = report.get_user(user_id)
    return user


if __name__ == "__main__":
    user_id = 12345
    user = get_report(user_id)
    print(user)
```
Let's explain what's going on here.

In line 4 we define a `MongoClient` class that can connect to a Mongo database and retrieve data. In reality, we would use a proper library like `pymongo`, or `motor` etc, but this would complicate the code a bit more so we'll keep things simple since this is not what we are focusing on right now.

```python
class MongoClient:
    def find_one(self, query: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}
```

The `MongoClient` class is exposing a `find_one` method that returns the first document that matches the query provided. Again, the result here is mocked for the sake of simplicity.

In line 9 we define a `Report` class that will generate our actual report information.

```python
class Report:
    def __init__(self, client):
        self.client = client

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.client.find_one({"userId": user_id})
        return user
```

The `Report` class is instantiated by providing a DB client, in this case `MongoClient`, and is exposing a `get_user` function that uses the DB client to retrieve the information from the database.

Finally, in line 18 we define a function `get_report` that takes a user_id and returns the user data.

```python
def get_report(user_id: int) -> Dict[str, Any]:
    client = MongoClient()
    report = Report(client)
    user = report.get_user(user_id)
    return user
```

This all works fine and sort of makes sense, however, one might notice an interesting detail..

The `Report` class is very tightly coupled with the `MongoClient` class, since the `get_user` method is directly calling the `find_one` method from the latter.

```python
user = self.client.find_one({"userId": user_id})
```

This behavior is a violation of the [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle), one of the five [SOLID principles](https://en.wikipedia.org/wiki/SOLID), which states that high level components (`Report`) should not depend on concrete implementations of low level components (`MongoClient`), but rather on abstractions (interfaces).

But what does that mean exactly?

## The problem of tight coupling
Let's now imagine that your boss comes back to notify you that, due to some licensing issues, you have to switch from MongoDB to CasandraDB.

You go back thinking and you decide that the natural thing to do would be to:
- Create a new `CasandraClient` class
- Alter the `Report` class so that it can now support queries both to Mongo and Casandra (you don't want to scrape all the hard work you did to support Mongo queries)
- Alter the `get_report` method to use a Casandra client instead of a Mongo one

Note that in reality, you cannot expect different libraries to expose the same functions, thus for the purpose of this example, the `CasandraClient` will expose a `get_item` method rather than a `find_one`.

Again, a naive implementation would now look something like the one below.

```python
from typing import Any, Dict


class MongoClient:
    def find_one(self, query: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}


class CasandraClient:
    def get_item(self, key: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}


class Report:
    def __init__(self, client):
        self.client = client

    def get_user_mongo(self, user_id: int) -> Dict[str, Any]:
        user = self.client.find_one({"userId": user_id})
        return user

    def get_user_casandra(self, user_id: int) -> Dict[str, Any]:
        user = self.client.get_item({"userId": user_id})
        return user


def get_report(user_id: int) -> Dict[str, Any]:
    client = CasandraClient()
    report = Report(client)
    user = report.get_user_casandra(user_id)
    return user


if __name__ == "__main__":
    user_id = 12345
    user = get_report(user_id)
    print(user)
```

Just like our first example, this could work and would probably make your boss happy, but there are a few caveats following this approach.

First of all, in line 18, you've now renamed the initial `get_user` method of the `Report` class to `get_user_mongo`, in order to differentiate it from the new `get_user_casandra`.

This violates the [Open–closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) which states that a class should be open for extension, but closed for modification.

But, as if that was not enough, in lines 28 and 30, you also had to change the `get_report` function to instantiate a `CasandraClient` and now call a different function `report.get_user_casandra`.

```python
def get_report(user_id: int) -> Dict[str, Any]:
    client = CasandraClient()
    report = Report(client)
    user = report.get_user_casandra(user_id)
    return user
```

This is bad...

Imagine that somehow you had published the `Report` as a library internally in your company and now other departments are also using it in their own programs.

The problem is that:
- If they update to the latest version, their programs will break because you introduced a breaking change (`get_user` was renamed to `get_user_mongo`)
- If they don't update, their programs will not return anything since the data would be migrated from Mongo to Casandra

## Dependency Inversion using an Interface
The way to solve this problem is by introducing an interface between the high level `Report` and the low level `MongoClient`, `CasandraClient` components.

And this is rather simple, especially if you do it in the early stages of development.

It goes something like this below..

```python
from abc import ABC, abstractmethod
from typing import Any, Dict


class DBClientInterface(ABC):

    @classmethod
    def client_factory(cls, db_type: str):
        if db_type == "mongo":
            return MongoClient()
        elif db_type == "casandra":
            return CasandraClient()
        else:
            raise ValueError("Invalid db type")

    @abstractmethod
    def get_user(self, user_id: int) -> Dict[str, Any]:
        ...


class MongoClient(DBClientInterface):
    def find_one(self, query: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}
    
    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.find_one({"userId": user_id})
        return user


class CasandraClient(DBClientInterface):
    def get_item(self, key: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.get_item({"userId": user_id})
        return user


class Report:
    def __init__(self, client):
        self.client = client

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.client.get_user({"userId": user_id})
        return user


def get_report(user_id: int, db_type: str) -> Dict[str, Any]:
    client = DBClientInterface.client_factory(db_type)
    report = Report(client)
    user = report.get_user(user_id)
    return user


if __name__ == "__main__":
    user_id = 12345
    get_report(user_id, db_type="casandra")
    print(user)
```

Let's explain what's going on this time.

First, in line 5, we implement a `DBClientInterface` class that inherits from `ABC` and we create an abstract `get_user` method.

This will be our generic DB client interface, forcing any DB client (mongo, casandra, etc) to implement this method.

We can also create a factory method `client_factory` to dynamically choose the correct client that needs to be instantiated.

```python
class DBClientInterface(ABC):

    @classmethod
    def client_factory(cls, db_type: str):
        if db_type == "mongo":
            return MongoClient()
        elif db_type == "casandra":
            return CasandraClient()
        else:
            raise ValueError("Invalid db type")

    @abstractmethod
    def get_user(self, user_id: int) -> Dict[str, Any]:
        ...
```

Then,  in lines 21 and 30, we implement our low level components by inheriting from the Interface.

```python
class MongoClient(DBClientInterface):
    def _find_one(self, query: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}
    
    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self._find_one({"userId": user_id})
        return user


class CasandraClient(DBClientInterface):
    def _get_item(self, key: Dict[str, Any]) -> Dict[str, Any]:
        return {"userId": 12345, "Name": "John Doe"}

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self._get_item({"userId": user_id})
        return user
```

Each client can implement the `get_user` method however they like.

For example, the `MongoClient` chooses to call a `_find_one` method, whereas the `CasandraClient` a `_get_item`.

This is an implementation detail and doesn't really matter since the high level component `Report` will only be calling `get_user` directly, which is consistent in any client that implements the `DBClientInterface`.

Then, in line 39, we implement the `Report` class, which now calls the generic `get_user` function of the interface.

```python
class Report:
    def __init__(self, client):
        self.client = client

    def get_user(self, user_id: int) -> Dict[str, Any]:
        user = self.client.get_user({"userId": user_id})
        return user
```

And.. voila, the `Report` class is now completely client agnostic and decoupled from the intricate details of the individual DB clients.

Finally, in line 48, the `get_report` function doesn't care anymore about what specific client we want to use since:
- The factory method handles the client instantiation
- The `report.get_user` function is consistent across all clients that implement the interface

```python
def get_report(user_id: int, db_type: str) -> Dict[str, Any]:
    client = DBClientInterface.client_factory(db_type)
    report = Report(client)
    user = report.get_user(user_id)
    return user


if __name__ == "__main__":
    user_id = 12345
    user = get_report(user_id, db_type="casandra")
    print(user)
```

Thus, even if you now wanted to switch to a third database, you would only have to implement a new client class and ask your colleagues to change the `db_type`.

No Dependency Inversion, or Open-Closed principles violated!

## Conclusion
This is a very simple example that can demonstrate the power of programming interfaces.

Obviously, in reality it is a lot harder to achieve (especially if you are trying to swap databases..) but the more you practice, the more you will realize its potential and hopefully it will bring a lot of value to your code.

My personal opinion is that even if you don't immediately think of sticking an interface between every component (ending up over-engineering things) you should at least reconsider it the moment a second low level component appears.

Let me know your thoughts.
