---
title: Python Metaclasses to the rescue
description: >-
  Using Python metaclasses to add special logic in a class
date: 2024-10-13 18:00:00 +0100
categories: [Python, Tutorial]
tags: [python,metaclasses]
tok: true
---

# Python Metaclasses to the rescue

## How it started

I recently faced the challenge of building a test suite for stress testing certain components of an application.

The idea was to generate a huge load of fake data and feed them to an internal system in order to evaluate the performance and scalability of the application.

Now, generating fake data is nothing really hard these days, however, there were a few requirements that bugged me for a while..

Firstly, the tests would have to run on a daily basis, via a scheduled job in an automated pipeline.

Secondly, all the jobs would have to run against the same environment, meaning that certain attributes of the fake data which were considered unique (such as ids) had to be transformed to avoid integrity errors downstream.

Long story short, the first thing that came to my mind was to create some custom transformers that can programmatically alter certain fields of the incoming data every time the tests run.

Nothing fancy, right?

Sure, but I also wanted the code to be as declarative as possible and written in a bulletproof way so that these **transformers are actually enforced**.

Sounds familiar? It's more or less what the ABC library is doing to enforce the implementation of abstract methods by child classes.

This would ensure that if a future developer was ever required to expand the tests, they would not accidentally forget to add some of the required transformers.

## Decorators are awesome

Python decorators are one of the most beautiful features in Python and I like using them a lot.

They not only "supercharge" your functions, but can also visually signal that a function has something special!

One of the most well known decorators is the built-in `@abstractmethod` of the [ABC](https://docs.python.org/3/library/abc.html) library, which essentially enforces a function to be implemented by any child class that inherits from the parent class/interface.

And this functionality is exactly what I wanted for my transformers!

I just had to figure out how it was implemented.

## Welcome to Python metaclasses

Taking a quick look at the [abc.py](https://github.com/python/cpython/blob/main/Lib/abc.py) module, gave me an overall understanding of how that mechanism works under the hood.

In a nutshell, the library is using the [ABCMeta](https://github.com/python/cpython/blob/main/Lib/abc.py#L92C11-L92C18), which is a type of [Metaclass](https://docs.python.org/3/reference/datamodel.html#metaclasses).

Metaclasses is an advanced Python feature and are essentially used to define and create other classes.

In this case, the ABCMeta is used to create abstract classes (Let's call them interfaces for simplicity..)

When an interface inherits from the ABCMeta class, the ABCMeta will "register" all its abstract functions (decorated with `@abstractmethod`), prior to defining the interface (at import time).

Doing that is as simple as attaching an `__isabstractmethod__` flag to the function objects.

```python
def abstractmethod(funcobj):
    """A decorator indicating abstract methods.

    Requires that the metaclass is ABCMeta or derived from it.  A
    class that has a metaclass derived from ABCMeta cannot be
    instantiated unless all of its abstract methods are overridden.
    The abstract methods can be called using any of the normal
    'super' call mechanisms.  abstractmethod() may be used to declare
    abstract methods for properties and descriptors.

    Usage:

        class C(metaclass=ABCMeta):
            @abstractmethod
            def my_abstract_method(self, arg1, arg2, argN):
                ...
    """
    funcobj.__isabstractmethod__ = True
    return funcobj
```

You can easily verify that with a simple example:
```bash
>>> from abc import ABCMeta, abstractmethod
>>>
>>> class Interface(metaclass=ABCMeta):
...     @abstractmethod
...     def test(self):
...         pass
...
>>> print(Interface.test.__isabstractmethod__)
True
```

Later on, when a child class that inherits from the interface is instantiated (at runtime), an evaluation will be performed, to check that all the abstract methods are implemented.

## Back to the task

Now, I had a basic understanding of what I needed to do in order to "replicate" this behavior for my transformers.

I started by creating my decorator.
```python
def requiredtransformer(fn) -> Callable:
    """ Decorator used to mark a function as required.
        It works in a similar way to abc.abstractmethod.
    """
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    setattr(wrapper, '__isrequiredtransformer__', True)
    return wrapper
```

Then, I created a metaclass that would allow me to mark specific transformers as "required".
```python
class EnforceTransformersMeta(type):
    """ Metaclass used to create classes with the additional
        option of marking their functions as 'required'.
    """
    def __new__(
            mcls: type, name: str, bases: Tuple[Any], namespace: Dict[Any, Any], /, **kwargs
    ) -> 'EnforceTransformersMeta':
        cls = super().__new__(mcls, name, bases, namespace, **kwargs)

        setattr(cls, '__requiredtransformers__', set())
        for name, value in namespace.items():
            if getattr(value, "__isrequiredtransformer__", False):
                cls.__requiredtransformers__.add(name)

        return cls
```

Given that I would also need to use some abstraction, I created a Mixin metaclass that combines the `ABCMeta` and the new `EnforceTransformersMeta`.
```python
class MsgTransformerMeta(ABCMeta, EnforceTransformersMeta):
    """ Mixin metaclass that combines abstract class functionality with
        additional functionality of the EnforceTransformersMeta metaclass
        that allows to enforce transformers.
    """
    pass
```

And now, I was ready to create my actual class that would handle the logic to enforce the required transformers.
```python
class MsgTransformer(metaclass=MsgTransformerMeta):
    """ Transformer class used to transform a specific type of json """
    __requiredtransformers__: Set[str]

    @property
    @abstractmethod
    def json_type(self) -> str:
        ...

    @abstractmethod
    def _validate_json_type(self, message: dict):
        ...

    def _ensure_transform(
        self, message: dict, transformers: Optional[List[Callable]] = None
    ) -> None:
        """ Ensures the required message transformers were
            called upon the generation of a new message.
        """
        required_transformers = self.__requiredtransformers__

        missing_transformers = None
        if required_transformers and not transformers:
            missing_transformers = required_transformers

        called = set()
        if transformers:
            for func in transformers:
                if isinstance(func, functools.partial):
                    called.add(func.func.__name__)
                else:
                    called.add(func.__name__)

                func(message=message)

        if required_transformers != called:
            missing_transformers = required_transformers.difference(called)

        if missing_transformers:
            raise MissingTransformersError(self.__class__.__name__, missing_transformers)

    def transform(
        self, message: dict, transformers: Optional[List[Callable]] = None
    ) -> Dict[str, Any]:
        """ Validates that all the 'required' transformers
            were used and returns the transformed message.
        """
        self._validate_json_type(message)
        self._ensure_transform(message, transformers)
        return message
```

If you take a closer look, you will notice that the `MsgTransformer` class provides a `transform` function that takes a message and a number of optional callables.

Let's move on to glue all the pieces together with an example.

## Play time

Let's assume we have the following bank message and we want to use it as a template for generating fake data.

```python
example_json = {
    'correspondence': 'Fake Bank',
    'accountName': 'John Doe',
    'accountNumber': 'IDLBG431911934/2220123',
    'openingBalance': 3000,
}
```
Now, I am not working for a bank, but I'd assume that the unique identifier in this case would be the accountNumber. Hence, we'd need to alter this number to something unique every time we run the tests.

Let's create a class with two transformers, `replace_account_number` and `replace_account_name`, the former being decorated as `@requiredtransformer`

```python
class MyBankTransformer(MsgTransformer):
    """ Transformer class for messages """
    json_type: str = 'FAKE'

    def _validate_json_type(self, message: Dict[str, Any]):
        if message.get('correspondence') != 'Fake Bank':
            raise Exception(f"Invalid json type {self.json_type}")

    @requiredtransformer
    def replace_account_number(
        self, message: Dict[str, Any], account_number: Optional[str] = None
    ) -> Dict[str, Any]:
        message['accountNumber'] = account_number
        return message

    def replace_account_name(
        self, message: Dict[str, Any], account_name: Optional[str] = None
    ) -> Dict[str, Any]:
        message['accountName'] = account_name
        return message
```

We can call the `transform` function and pass to it the two transformers.
```python
msg_transformer = MyBankTransformer()

transformed_json = msg_transformer.transform(
    example_json,
    transformers=[
        functools.partial(msg_transformer.replace_account_number, account_number="ABC123"),
        functools.partial(msg_transformer.replace_account_name, account_name="Jane Doe")
    ]
)

pprint.pprint(transformed_json)
```

```bash
{
    'accountName': 'Jane Doe',
    'accountNumber': 'ABC123',
    'correspondence': 'Fake Bank',
    'openingBalance': 3000
}
```
The result is as expected. The accountNumber has changed to "ABC123" and the accountName has changed to "Jane Doe".

And now to the interesting part. Let's check that our transformers are enforced by removing the `replace_account_number` transformer.

```python
msg_transformer = MyBankTransformer()

transformed_json = msg_transformer.transform(
    example_json,
    transformers=[
        functools.partial(msg_transformer.replace_account_name, account_name="Jane Doe")
    ]
)

pprint.pprint(transformed_json)
```

And voila! Now we get an exception.

```bash
Traceback (most recent call last):
  File "/home/state_machine/metaclasses.py", line 148, in <module>
    transformed_json = msg_transformer.transform(
  File "/home/state_machine/metaclasses.py", line 110, in transform
    self._ensure_transform(message, transformers)
  File "/home/state_machine/metaclasses.py", line 101, in _ensure_transform
    raise MissingTransformersError(self.__class__.__name__, missing_transformers)
__main__.MissingTransformersError: The following mandatory 'MyBankTransformer' transformers were not applied: {'replace_account_number'}
```

## Conclusion

With a few lines of code, we created our own decorator that gives a special meaning in our classes and offers a decent form of validation.

However, I am still not convinced this is the best way to attack this problem, but time will tell..

Nevertheless, playing with metaclasses was certainly fun!

You can find the complete code in my github repository [here](https://github.com/kolitiri/python-utils-collection/tree/master/metaclasses)
