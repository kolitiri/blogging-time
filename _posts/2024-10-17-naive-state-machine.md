---
title: A naive State Machine
description: >-
  Implementing a naive state machine just for fun
date: 2024-10-17 07:30:00 +0100
categories: [Python]
tags: [python,statemachines]
tok: true
---

---
I am currently experimenting with state machines and I wanted to roll my own implementation just for fun!

There are a number of state machine libraries in python and I believe the most popular one is the [python-statemachine](https://python-statemachine.readthedocs.io/en/latest/) by Fernando Macedo (fgmacedo). It is in active development and has a decent amount of stars and a lot of contributors.

One thing I didn't quite like however was the fact that it uses naming conventions for certain functions.

For example, given a TrafficLight and a Cyclist, the cyclist class can implement the following functions `before_cycle`, `on_enter_red`, and `on_exit_red` where the `before_`, `on_enter_` and `on_exit_` are conventional prefixes that need to be respected.

Now, bear in mind that I might have very likely misunderstood the usage of the library, since I haven't actually used it, but oh well.. I guess I was just trying to find an excuse to write some code in my spare time.

## The plan

I wanted to somehow abstract the above functionality in a way that it is more declarative and also make it possible for certain state machines to automatically react to changes in the state of other state machines.

Some sort of intercommunication.

What I had in mind was something like the following

```python
class TrafficLightState(State):
    GREEN = "GREEN"
    YELLOW = "YELLOW"
    RED = "RED"


class CyclistState(State):
    CYCLING = "CYCLING"
    STOPPED = "STOPPED"


class TrafficLight(StateMachine):

    @action(TrafficLightState.GREEN)
    def green(self, **kwargs) -> None:
        ...

    @action(TrafficLightState.YELLOW)
    def yellow(self, **kwargs) -> None:
        ...

    @action(TrafficLightState.RED)
    def red(self, **kwargs) -> None:
        ...


class Cyclist(StateMachine):

    @action(CyclistState.CYCLING)
    def cycle(self, **kwargs) -> None:
        ...

    @action(CyclistState.STOPPED)
    def stop(self, **kwargs) -> None:
        ...

    @reaction(TrafficLightState.GREEN)
    def on_green_light(self, event: Event) -> None:
        self.cycle()

    @reaction(TrafficLightState.RED)
    def on_red_light(self, event: Event) -> None:
        self.stop()
```

This way, we could define the actions that a Cyclist can take, and also their reactions to changes in the state of a traffic light.

The `cycle` function is decorated with an `@action` transitioning to the state `CyclistState.CYCLING`.

On the other hand, the `on_red_light` function is decorated with a `@reaction` caused by the state of the TrafficLight transitioning to `TrafficLightState.RED`.

There should be no need to give these functions a conventional name since the decorators would take care of registering them appropriately through a Metaclass.

Furthermore, we could inject certain conditions straight into the function definition, and transform them into something like:

```python
@action(CyclistState.CYCLING, when={TrafficLightState.GREEN, TrafficLightState.YELLOW}, unless={})
def cycle_fast(self, **kwargs) -> None:
    ...

@action(CyclistState.CYCLING, when={TrafficLightState.GREEN}, unless={})
def cycle_slow(self, **kwargs) -> None:
    ...
```

## The implementation

Achieving the above meant my state machines should have three basic characteristics:
- All state machines would have to have a predefined set of Transitions
- State machines such as the `TrafficLight` would have to be able to "broadcast" Transitions of their internal state
- State machines such as the `Cyclist` would have to be able to "receive" and "process" external events, such as changes in the state of the `TrafficLight`

I started with some basic definitions

```python
import asyncio
from dataclasses import dataclass
from enum import Enum
from functools import wraps
from typing import Any, Callable, Dict, List, Optional, Set, Tuple


class TransitionError(Exception):
    pass


class State(Enum):
    pass


@dataclass
class Transition:
    source: State
    destination: State
    bidirectional: bool = False


@dataclass()
class Event:
    name: str
    source: "StateMachine"
    state: State
    meta: Optional[Dict[str, Any]] = None

```

Then, I defined a Metaclass that would be used to create the actual `StateMachine` class later on.

```python
class StateMachineMeta(type):
    def __new__(
        mcls: type, name: str, bases: Tuple[Any], namespace: Dict[Any, Any], /, **kwargs
    ) -> "StateMachineMeta":
        cls = super().__new__(mcls, name, bases, namespace, **kwargs)

        cls.broadcast_events = []
        cls.reactions_to_state = {}
        for name, value in namespace.items():
            action_name = name
            if getattr(value, "__is_action_function__", False):
                state = value.__transitions_to__
            
                if getattr(value, "__broadcast__", False):
                    cls.broadcast_events.append(action_name)

            if getattr(value, "__is_reaction_function__", False):
                state = value.__reacts_to__
                if state in cls.reactions_to_state:
                    raise Exception(f"Reaction to state {state} is already handled by function {action_name}")

                cls.reactions_to_state[state] = value

        return cls
```

The `StateMachineMeta` metaclass is responsible for:
- Registering the `@action` decorated functions
- Registering the `@reaction` decorated functions
- Registering the `@action` decorated functions that can broadcast their events

The way it's done is simply by reading some dunder attributes that are attached to the functions and storing them in a dataset (list or dict, etc)

These attributes should be attached to the functions by the `@action` and `@reaction` decorators at import time.

So, the next step was to define these two decorators.

```python
def action(state: State, broadcast: bool = False, when: Optional[Set[State]] = None, unless: Optional[Set[State]] = None) -> Callable:
    def decorator(function):
        @wraps(function)
        def wrapper(self, **kwargs) -> None:

            if getattr(self, "extra_meta", None):
                kwargs.update(self.extra_meta)

            states = set(list(self.publishers.values()) + [self.state])

            # If none of the when conditions are satisfied, do not process
            if when and not when.intersection(states):
                print(f"[{self}] cannot {function.__name__} unless {when}")
                return

            # If any of the unless conditions are satisfied, do not process
            if unless and unless.intersection(states):
                print(f"[{self}] cannot {function.__name__} when {unless}")
                return

            function(self, **kwargs)

            self.process_event(
                Event(name=function.__name__, source=self, state=state, meta=kwargs)
            )

        setattr(wrapper, "__broadcast__", broadcast)
        setattr(wrapper, "__transitions_to__", state)
        setattr(wrapper, "__is_action_function__", True)
        return wrapper

    return decorator
```

The `@action` decorator wraps a function and performs the following tasks:
- Validates the "when" and "unless" conditions (if any)
- Executes that wrapped function
- Processes the event that is associated with the wrapped function

```python
def reaction(state: State, when: Optional[Set[State]] = None, unless: Optional[Set[State]] = None) -> Callable:
    def decorator(function):
        @wraps(function)
        def wrapper(self, event: Event) -> None:

            states = set(list(self.publishers.values()) + [self.state])

            # If none of the when conditions are satisfied, do not process
            if when and not when.intersection(states):
                print(f"[{self}] cannot react {function.__name__} unless {when}")
                return

            # If any of the unless conditions are satisfied, do not process
            if unless and unless.intersection(states):
                print(f"[{self}] cannot react {function.__name__} when {unless}")
                return

            function(self, event)

        setattr(wrapper, "__reacts_to__", state)
        setattr(wrapper, "__is_reaction_function__", True)
        return wrapper

    return decorator
```

The `@reaction` decorator wraps a function and performs the following tasks:
- Validates the "when" and "unless" conditions (if any)
- Executes that wrapped function

Note that the `@reaction` decorator does not process any events since it is an external event itself that triggers the reaction.

Finally, the last step was to implement the actual `StateMachine` class that would glue all the pieces together.

```python
class StateMachine(metaclass=StateMachineMeta):
    transitions: List[Transition]
    broadcast_events: List[str]
    reactions_to_state: Dict[State, Callable]

    def __init__(self, initial_state: State) -> None:
        self.events = []
        self.state = initial_state

        self.transitions_map = {}
        self.consumers = set()
        self.publishers = {}
        self.name = self.__class__.__name__.lower()

        print(f"[{self.name}] starting in state {self.state}")

        for transition in self.transitions:
            self.transitions_map.setdefault(transition.source, [])
            self.transitions_map[transition.source].append(transition.destination)
            if transition.bidirectional:
                self.transitions_map.setdefault(transition.destination, [])
                self.transitions_map[transition.destination].append(transition.source)

    def register(self, consumers: List["StateMachine"]) -> None:
        for consumer in consumers:
            print(f"[{self.name}] registering consumer [{consumer}]")
            self.consumers.add(consumer)
    
    def subscribe(self, publishers: List["StateMachine"]) -> None:
        for publisher in publishers:
            self.publishers[publisher] = publisher.state
            publisher.register([self])

    def publish_event(self, event: Event) -> None:
        for consumer in self.consumers:
            print(f"[{self.name}] publishing {event.name} to [{consumer}]")
            consumer.process_event(event)

    def process_event(self, event: Event) -> None:
        self.events.append(event)

        if event.source == self:
            print(f"[{self.name}] processing event {event.name}")
            self._process_internal_event(event)
        else:
            print(f"[{self.name}] processing event {event.name} from [{event.source}]")
            self._process_external_event(event)

    def _process_internal_event(self, event: Event) -> None:
        target_state = event.state
        if self.state == target_state:
            return

        if target_state in self.transitions_map.get(self.state, []):
            print(f"[{self.name}] transitioning from {self.state} to {target_state}")
            self.state = target_state

            if event.name in self.broadcast_events:
                self.publish_event(
                    Event(event.name, self, self.state, meta=event.meta)
                )
        else:
            raise TransitionError(f"[{self.name}] cannot transition from {self.state} to {target_state}")

    def _process_external_event(self, event: Event) -> None:
        self.publishers[event.source] = event.state

        reaction_fn = self.reactions_to_state.get(event.state)
        if reaction_fn:
            reaction_fn(self, event)

    def __repr__(self) -> str:
        return self.name
```

This is were it got a bit more complicated, but in essence, the `StateMachine` class can perform the following tasks:
- Define the expected Transitions between the various states of the object
- Process internal events triggered by action functions
- Process external events triggered by other state machines
- Subscribe to external state machines and listen for events that are being broadcasted
- Register external state machines and broadcast to them events

The implementation might seem fairly straight forward, however I can't say I am particularly happy with it..

It does seem to me that the class is doing too many things, but I found myself going down the rabbit hole trying to establish a more granular separation of concerns.

I tried inheritance.. didn't work, I tried Mixins, didn't work.. I tried separating the actual publishing/subscribing functionality in order to use composition.. but still wasn't happy.

I am sure there has to be a more pythonic way of implementing it, but I just wasn't able to figure it out at that moment!

## The end result

Now it was time to put it to the test!

Let's assume that we have three state machines, a `TrafficLight`, a `Cyclist` and a `PoliceCar`.

We first define the state of each entity:
```python
class TrafficLightState(State):
    GREEN = "GREEN"
    YELLOW = "YELLOW"
    RED = "RED"


class CyclistState(State):
    CYCLING = "CYCLING"
    STOPPED = "STOPPED"


class PoliceCarState(State):
    ROLLING = "ROLLING"
    STOPPED = "STOPPED"
    CHASING = "CHASING"
```

The `TrafficLight` should be able to broadcast changes to its state, hence we pass `broadcast=True` to all its actions.

```python
class TrafficLight(StateMachine):
    
    transitions = [
        Transition(TrafficLightState.GREEN, TrafficLightState.YELLOW),
        Transition(TrafficLightState.YELLOW, TrafficLightState.RED),
        Transition(TrafficLightState.RED, TrafficLightState.GREEN),
    ]

    @action(TrafficLightState.GREEN, broadcast=True)
    def green(self, **kwargs) -> None:
        ...

    @action(TrafficLightState.YELLOW, broadcast=True)
    def yellow(self, **kwargs) -> None:
        ...

    @action(TrafficLightState.RED, broadcast=True)
    def red(self, **kwargs) -> None:
        ...
```

The `Cyclist` should be able to perform its regular actions (i.e cycle_slow, cycle_fast, stop) but under certain conditions.

For example, a cyclist should only cycle slow when the traffic light is green (and not yellow), and stop when the traffic light is red or when they're being chased by a police car.

```python
class Cyclist(StateMachine):

    transitions = [
        Transition(CyclistState.STOPPED, CyclistState.CYCLING),
        Transition(CyclistState.CYCLING, CyclistState.STOPPED),
    ]

    @action(CyclistState.CYCLING, when={TrafficLightState.GREEN, TrafficLightState.YELLOW}, unless={PoliceCarState.CHASING})
    def cycle_fast(self, **kwargs) -> None:
        ...

    @action(CyclistState.CYCLING, when={TrafficLightState.GREEN}, unless={PoliceCarState.CHASING})
    def cycle_slow(self, **kwargs) -> None:
        ...

    @action(CyclistState.STOPPED)
    def stop(self, **kwargs) -> None:
        ...

    @reaction(TrafficLightState.GREEN)
    def on_green_light(self, event: Event) -> None:
        self.cycle_slow()

    @reaction(TrafficLightState.RED)
    def on_red_light(self, event: Event) -> None:
        self.stop()

    @reaction(PoliceCarState.CHASING)
    def on_being_chased(self, event: Event) -> None:
        self.stop()
```

The `PoliceCar`, should be able to perform its regular actions (i.e roll, chase, stop), but just like the cyclist, under certain conditions.

For example, a police car could ignore a red light when it is chasing a suspect.

In addition, just like the `TrafficLight`, it should be able to broadcast to other entities when it begins chasing them (i.e engage the siren!).

```python
class PoliceCar(StateMachine):

    transitions = [
        Transition(PoliceCarState.ROLLING, PoliceCarState.STOPPED, bidirectional=True),
        Transition(PoliceCarState.ROLLING, PoliceCarState.CHASING, bidirectional=True),
        Transition(PoliceCarState.STOPPED, PoliceCarState.CHASING, bidirectional=True),
    ]

    @action(PoliceCarState.CHASING, broadcast=True)
    def chase(self, **kwargs) -> None:
        ...

    @action(PoliceCarState.ROLLING, broadcast=True)
    def roll(self, **kwargs) -> None:
        ...

    @action(PoliceCarState.STOPPED)
    def stop(self, **kwargs) -> None:
        ...

    @reaction(TrafficLightState.GREEN, unless={PoliceCarState.CHASING})
    def on_green_light(self, event: Event) -> None:
        self.roll()

    @reaction(TrafficLightState.RED, unless={PoliceCarState.CHASING})
    def on_red_light(self, event: Event) -> None:
        self.stop()
```

Let's see a few examples in practice.

### Scenario A

Given that:
- A traffic light is currently yellow
- A police car is chasing
- The traffic light switches to red
Then:
- The police car ignores the red light and carries on chasing

```python
async def run():
    traffic_light = TrafficLight(TrafficLightState.YELLOW)
    police_car = PoliceCar(PoliceCarState.ROLLING)
    
    police_car.subscribe([traffic_light])

    police_car.chase()
    traffic_light.red()
```

Looking at the logs, the result appears to be as expected (logs are slightly truncated to avoid much noise)
```bash
[trafficlight] starting in state TrafficLightState.YELLOW
[policecar] starting in state PoliceCarState.ROLLING
[policecar] transitioning from PoliceCarState.ROLLING to PoliceCarState.CHASING
[trafficlight] transitioning from TrafficLightState.YELLOW to TrafficLightState.RED
[trafficlight] publishing red to [policecar]
[policecar] processing event red from [trafficlight]
[policecar] cannot react on_red_light when {<PoliceCarState.CHASING: 'CHASING'>}
```

### Scenario B

Given that:
- A traffic light is currently yellow
- The traffic light switches to red, green and then yellow
Then:
- The cyclist and the police car stop at the red light
- The cyclist and the police car start moving at the green light
- The cyclist and the police car carry on moving at the yellow light

```python
async def run():
    traffic_light = TrafficLight(TrafficLightState.YELLOW)
    cyclist = Cyclist(CyclistState.CYCLING)
    police_car = PoliceCar(PoliceCarState.ROLLING)
    
    cyclist.subscribe([traffic_light])
    police_car.subscribe([traffic_light])
    police_car.register([cyclist])

    traffic_light.red()
    traffic_light.green()
    traffic_light.yellow()
```

Looking at the logs again, the result appears to be as expected, apart from a tiny bug that needs fixing..

We might notice that when the traffic light switches to yellow, the cyclist won't stop despite the fact that they are currently cycling slow!

```bash
[trafficlight] starting in state TrafficLightState.YELLOW
[cyclist] starting in state CyclistState.CYCLING
[policecar] starting in state PoliceCarState.ROLLING
[trafficlight] transitioning from TrafficLightState.YELLOW to TrafficLightState.RED
[cyclist] transitioning from CyclistState.CYCLING to CyclistState.STOPPED
[policecar] transitioning from PoliceCarState.ROLLING to PoliceCarState.STOPPED
[trafficlight] transitioning from TrafficLightState.RED to TrafficLightState.GREEN
[cyclist] transitioning from CyclistState.STOPPED to CyclistState.CYCLING
[policecar] transitioning from PoliceCarState.STOPPED to PoliceCarState.ROLLING
[trafficlight] transitioning from TrafficLightState.GREEN to TrafficLightState.YELLOW
[cyclist] transitioning from CyclistState.CYCLING to CyclistState.STOPPED
```

We could try a few more naughty scenarios such as having our cyclist ignore the fact that they're being chased by a police car.

This should be simple. Just change the `on_being_chased` reaction to call `cycle_fast` instead of `stop`

```python
@reaction(PoliceCarState.CHASING)
def on_being_chased(self, event: Event) -> None:
    self.cycle_fast()
```

But I'll leave that open to your imagination!

## Conclusion
You can find the source code in my github repository [naive-state-machine](https://github.com/kolitiri/naive-state-machine), along with a few examples to experiment with.

By no means, this version is buggy and requires some extra polishing.

I feel however, that this is a nice and more declarative way to define state machines, easy to read and easy to extend, thus I might fix the bugs and publish it on PyPI later on.

In the meantime, feel free to experiment!
