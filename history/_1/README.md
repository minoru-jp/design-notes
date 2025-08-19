# Verbose Pattern in Python

## Introduction

The term **"Verbose Pattern"** is an informal name used in this document solely for convenience when describing this structure. It does not correspond to any officially recognized design pattern.

This structure is related to the following design patterns and principles:

* **Factory Pattern**: Delegating object creation to dedicated functions
* **Interface Segregation Principle**: Separating interfaces by role
* **Facade Pattern**: Hiding complex internal structures behind a simplified external interface
* **Strategy Pattern**: Separating functionality based on roles

This pattern defines a two-layered factory function, aiming to make step-by-step operation checks and tests easier for each processing unit.
It is intended for *long-lived, low-volume, fundamental components*.

## Key Features

* Clearly separates external-facing interfaces from internal/testing-stage interfaces.
* Allows representation of a state where variables have been defined but **meaningful initialization has not yet occurred**, enabling incremental testing of initialization processes.
* Interfaces are defined by role, and each can be validated independently, making it easy to adjust the granularity of tests.
* Introducing a `_ *Bridge` interface enables structured delegation from one factory function to another.
* By adopting only the required interfaces, the code size scales in proportion to complexity.

## Notes

* This pattern does not introduce fundamentally new capabilities compared to standard class instances.
* The goals pursued by this pattern can also be achieved with regular class instances.
* However, a general property of closures is that internal state referenced by an external-facing interface is not easily accessible. This reduces the risk of accidental internal state access compared to class instances. This, however, is not unique to this pattern—it applies generally to any interface exposed through factory functions.
* The primary focus of this pattern is to prioritize structural clarity of the implementation, even at the cost of verbosity.

## Implementation Overview

The pattern is implemented by splitting the factory function into two layers.
The first layer, `_create_*_all()`, returns multiple interfaces structured by role (e.g., `_ *Constant` and `_ *State` for constants and internal state getters).
Testing can then be performed on each interface individually.

### Structure Overview (Pseudo-code):

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from enum import Enum, auto
from typing import Protocol
from dataclasses import dataclass

# * = Interface name

class *(ABC):
    __slots__ = ()
    # Public-facing interface. Defined with ABC for static type checking.
    @abstractmethod
    def greet(self) -> None:
        ...

class _*Constant(Protocol):
    # Defines constants used internally in the factory
    GREET_IN_MORNING: str
    GREET_IN_AFTERNOON: str

class _*State(Protocol):
    # Provides access to mutable internal state
    init_state: str
    meridiem: Meridiem

class _*Core(Protocol):
    # Internal-use-only methods.
    # The 'initialize()' method is called externally to trigger initialization.
    def init_process_a(self) -> None: ...
    def init_process_b(self) -> None: ...
    def init_process_c(self) -> None: ...
    def initialize(self) -> None: ...

# _*+Bridge(Protocol): # + = Name according to delegated role
# Interface for delegating part of the functionality to related entities.
# Splitting by role (permissions) classifies the derived entities.

class _*ReaderBridge(Protocol):
    # Reader can check initialization status and the current Meridiem.
    def is_initialized(self) -> bool: ...
    def current_meridiem(self) -> Meridiem: ...

class _*UpdaterBridge(_*ReaderBridge, Protocol):
    # Updater can toggle between AM and PM.
    # Updater also inherits the functions of _*ReaderBridge.
    def toggle_meridiem(self) -> None: ...

class Meridiem(Enum):
    AM = auto()
    NOON = auto()
    PM = auto()
    MIDNIGHT = auto()

def _create_*_all() -> tuple[*, _*Constant, _*State, _*Core, _*Bridge]:  # 1
    # In this pseudo-code, _create_*_all() returns all interfaces.
    # _*Constant, _*State, _*Core, and _*Bridge may not exist.
    # The * interface is always present.
    # In particular, _*Core may be absent if initialization is considered complete
    # solely through factory function arguments.

    @dataclass(slots=True)  # 2, 3
    class _Constant(_*Constant):
        GREET_IN_MORNING: str = "good morning!!"
        GREET_IN_AFTERNOON: str = "good afternoon!!"
    constant = _Constant()

    @dataclass(slots=True)
    class _State(_*State):
        init_state: str = "not initialized"
        meridiem: Meridiem = Meridiem.NOON
    state = _State()  # 4

    class _Core(_*Core):
        __slots__ = ()
        def init_process_a(self) -> None:
            state.init_state = "start initialization"
        def init_process_b(self) -> None:
            state.init_state = "initialization is still in progress"
        def init_process_c(self) -> None:
            state.init_state = "initialized"
            state.meridiem = Meridiem.AM
        def initialize(self) -> None:
            self.init_process_a()
            self.init_process_b()
            self.init_process_c()
    core = _Core()

    class _ReaderBridge(_*ReaderBridge):
        __slots__ = ()
        def is_initialized(self) -> bool:
            return state.init_state == "initialized"
        def current_meridiem(self) -> Meridiem:
            return state.meridiem
    reader_bridge = _ReaderBridge()

    class _UpdaterBridge(_*UpdaterBridge):
        __slots__ = ()
        def is_initialized(self) -> bool:  # 5
            return reader_bridge.is_initialized()
        def current_meridiem(self) -> Meridiem:
            return reader_bridge.current_meridiem()
        def toggle_meridiem(self) -> None:
            meridiem = state.meridiem
            if meridiem is Meridiem.AM:
                state.meridiem = Meridiem.PM
            elif meridiem is Meridiem.PM:
                state.meridiem = Meridiem.AM
            else:
                raise RuntimeError("Bug")
    updater_bridge = _UpdaterBridge()

    class _Interface(*):
        __slots__ = ()
        def greet(self):
            meridiem = state.meridiem
            if meridiem is Meridiem.AM:
                print(constant.GREET_IN_MORNING)
            elif meridiem is Meridiem.PM:
                print(constant.GREET_IN_AFTERNOON)
            else:
                print("... (If you see this message, please contact support.)")
    interface = _Interface()

    return (interface, constant, state, core, reader_bridge, updater_bridge)

def create_*() -> *:
    roles = _create_*_all()
    core = roles[3]
    core.initialize()
    interface = roles[0]
    return interface
```

1. Factory function names (such as the *Interface name*) are written in snake\_case.
2. The `_Constant` class is instantiated for consistency with other interfaces, though this is not strictly required.
3. `_Constant` is defined as a dataclass without `frozen` so that constants can be modified during testing. If modification is unnecessary, setting `frozen` prevents accidental changes, improving safety.
4. The `state` is mutable, allowing any desired state to be created during testing.
5. `_ *+Bridge(Protocol)` is implemented here using inheritance for simplicity. It could also be implemented with composition, and either method may be chosen as needed.

### Additional Notes

* In the pseudo-code, `__slots__` is defined for all interface implementations, but this is optional. (This is to prevent unintended state injection into interfaces, but it is a matter of preference.)
* Related to the previous point, all interfaces could be implemented as classes without instantiation. However, doing so would require metaclasses for `__slots__` or `@classmethod` usage, so for simplicity, instantiation is used here. This is an implementation preference; either approach can be chosen if it expresses the interface properly. As noted earlier, while not the intended case, if adopting this pattern for lightweight, high-volume instances, using classes for all interfaces to avoid instantiation might be worth considering.
* The structure is intentionally verbose to preserve incremental testing and static type checking.
* All interfaces could be implemented with `ABC`, but `Protocol` is used for internal interfaces to avoid decorator boilerplate. These choices can be freely made based on developer preference and project requirements.

