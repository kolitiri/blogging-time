---
title: Demystifying FastApi's dependency injection
description: >-
  Exploring how FastAPI's dependency injection works under the hood
date: 2024-11-20 07:10:00 +0100
categories: [Python]
tags: [python,fastapi]
tok: true
-image:
  path: /assets/img/illustrations/dependency-injection.jpg
  alt: Dependency Injection
---

---
[FastApi](https://fastapi.tiangolo.com/) has an interesting way of achieving dependency injection using the [fastapi/params.py::Depends](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/params.py#L764) class in the type annotations of a path function.

Here's a simple example taken from FastApi's [Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/#first-steps) documentation.

```python
async def common_parameters(q: str | None = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}


@app.get("/items/")
async def read_items(commons: Annotated[dict, Depends(common_parameters)]):
    return commons
```
When I first stumbled upon this syntax I was very confused..

I couldn't quite understand how a type annotation can be used to execute a callable and inject arguments into the scope of a path function.

That was an interesting approach and I felt I wanted to know how it was done.

## The route decorator
My very first thought was that the `app.get` decorator was doing some sort of magic to achieve this functionality, so I had to investigate deeper.

The decorator is part of the [fastapi/applications.py::FastAPI](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/applications.py#L48) class and it is essentially a function that returns a callable.

The callable is returned by another `get` function that belongs to the [fastapi/routing.py::APIRoute](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/routing.py#L428).

There, [fastapi/routing.py::APIRouter.api_route](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/routing.py#L963) is called, which finally returns the actual the decorator.

Now, as we know, a decorator is a wrapper function that:
- takes another function as an argument, in this case the path function `read_items`
- performs some logic, if necessary
- returns the original function back 

In this case, the logic that is performed is to create a new api route by calling [fastapi/routing.py::APIRouter.add_api_route](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/routing.py#L881) and append it to the routes of object.

```python
self.routes.append(route)
```

Fine.. At that point I still didn't have much.

## Dependency discovery (at import time)
An interesting thing was that `dependencies` were passed all the way down to the route class, so I knew I had to look deeper into the instantiation of that class.

The signature of the [fastapi/routing.py::APIRouter.\_\_init\_\_](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/routing.py#L429) function was indeed a confirmation as it clearly states that the dependencies is a sequence of optional `params.Depends`.

```python
class APIRoute(routing.Route):
    def __init__(
        self,
        path: str,
        ...
        dependencies: Optional[Sequence[params.Depends]] = None,
        ...
    )
```

Bingo! Turned out that upon initialization of the router class, a loop over the dependencies occurs, that after a series of nested calls (including some potential recursion), it assigns the dependency callables in the route object.

```python
for depends in self.dependencies[::-1]:
    self.dependant.dependencies.insert(
        0,
        get_parameterless_sub_dependant(depends=depends, path=self.path_format),
    )
```

The logic is fairly complicated, but the key function I was looking for was the [fastapi/dependencies/utils.py::get_typed_signature](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/dependencies/utils.py#L231).

```python
def get_typed_signature(call: Callable[..., Any]) -> inspect.Signature:
    signature = inspect.signature(call)
    globalns = getattr(call, "__globals__", {})
    typed_params = [
        inspect.Parameter(
            name=param.name,
            kind=param.kind,
            default=param.default,
            annotation=get_typed_annotation(param.annotation, globalns),
        )
        for param in signature.parameters.values()
    ]
    typed_signature = inspect.Signature(typed_params)
    return typed_signature
```

This was exactly the point where FastAPI was interpreting the type annotations from the signature of the `read_items` function, using Python's standard [inspect](https://docs.python.org/3/library/inspect.html) library.

Bear in mind, however, that all this was happening at import time.. Thus, there was a final step to investigate.

Where are the dependency callables actually executed?

## Dependency execution (at runtime)
Turns out, this is happening at runtime in [fastapi/routing.py::get_request_handler](https://github.com/fastapi/fastapi/blob/1f7629d3e421493ef9539ea7288564584c0182be/fastapi/routing.py#L217), presumably when a new request is made to the "/items/" endpoint.

Somewhere in there, the dependency callables that have already been assigned to the route object are executed (`solve_dependencies`) and their results are passed down to the `read_items` function as arguments (`run_endpoint_function`).

```python
solved_result = await solve_dependencies(
    request=request,
    dependant=dependant,
    body=body,
    dependency_overrides_provider=dependency_overrides_provider,
    async_exit_stack=async_exit_stack,
    embed_body_fields=embed_body_fields,
)
errors = solved_result.errors
if not errors:
    raw_response = await run_endpoint_function(
        dependant=dependant,
        values=solved_result.values,
        is_coroutine=is_coroutine,
    )
```

## Getting rid of all the noise
I figured out that sometimes it is a lot easier for me to understand something if I start stripping down pieces of code to reduce the complexity.

So, if all the above sound Greek to you, I've created an overly simplified example to demonstrate how it works under the hood.

In a nutshell, it is just a decorator that uses the `inspect` library to extract the callable from the annotation.

```python
import functools
import inspect
from typing import Annotated, Any, Callable, Optional
from typing_extensions import Annotated, get_args


class Depends:
    def __init__(self, dependency: Callable[..., Any]):
        self.dependency = dependency


def get_typed_signature(call: Callable[..., Any]) -> inspect.Signature:
    """Returns the Signature of a callable"""
    return inspect.signature(call)


def analyze_param(param: inspect.Parameter) -> Depends:
    """Extracts the dependency object from the parameter annotation"""
    annotated_args = get_args(param.annotation)
    return [
        arg
        for arg in annotated_args[1:]
        if isinstance(arg, Depends)
    ][-1]


def solve_dependencies(dependency: Depends) -> Any:
    """Executes the dependency function"""
    return dependency.dependency()


def get(path):
    def api_route(func: Callable) -> Callable:
        @functools.wraps(func)
        def decorator(*args, **kwargs) -> Any:
            signature = get_typed_signature(func)

            dependency_kwargs = {}
            for param_name, param in signature.parameters.items():
                dependency = analyze_param(param)
                dependency_results = solve_dependencies(dependency)
                dependency_kwargs[param_name] = dependency_results

            kwargs.update(dependency_kwargs)

            return func(*args, **kwargs)
        return decorator
    return api_route


def common_parameters(q: Optional[str] = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}


@get("/items/")
def read_items(commons: Annotated[dict, Depends(common_parameters)]):
    return commons


if __name__ == "__main__":
    print(read_items())
```

Again, this is way too simplified compared to FastAPI's source code, but in reality, this is how its dependency injection works..

**There is a decorator that extracts callables from type annotations, executes them and then passes the results to the decorated function.**

That's all it really is!

If we execute the script above, we get the expected result.

```bash
{'q': None, 'skip': 0, 'limit': 100}
```

## Conclusion
The way FastAPI is achieving dependency injection is indeed an interesting approach which I'd personally never seen before.

The fact that it relies on type annotations does feel a bit strange at first glance but, at the end of the day, I think it is what makes it look so elegant.

The source code of FastAPI can be found in the [fastapi](https://github.com/fastapi/fastapi/) github repo.

If you'd like to explore in more detail, I'd suggest that you fork the repo and run it locally with a few breakpoints here and there.
