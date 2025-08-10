# Verbose Pattern in Python

## Introduction

The term **"Verbose Pattern"** is an informal name used in this document to describe the structure. It does not correspond to any formal recognized design pattern.

This structure is related to the following design patterns and principles:

* **Factory Pattern**: Delegating object creation to dedicated functions
* **Interface Segregation Principle**: Separating interfaces by role
* **Facade Pattern**: Hiding internal complexity behind a simplified interface
* **Strategy Pattern**: Separating behavior based on roles

This pattern is defined by splitting the factory function into two layers, enabling step-by-step operation checks and testing for each processing unit. It is intended for *long-lived, low-volume, fundamental components* that require structured and maintainable implementation.

## Key Features

* Clearly separates external-facing interfaces from internal/test-time interfaces.
* Enables the representation of a *defined but not yet meaningfully initialized* state, making it easier to perform incremental testing of initialization processes.
* Interfaces are defined by role, allowing independent testing of each component, making it easy to set the granularity of tests.
* The introduction of a \_\*Bridge interface allows the structured delegation of functionality from one factory function to another.
* By adopting only the necessary interfaces, the code size grows in proportion to the complexity.

## Notes

* This pattern does not enable anything fundamentally new compared to standard class instances.
* The goals attempted with this pattern can also be achieved using regular class instances.
* However, a general characteristic of closures is that the internal state referenced by external interfaces is not easily accessible. As a result, the risk of unintended access to internal state is lower than with class instances. But this is not unique to this pattern; it applies to any interface exposed through factory functions.
* The primary focus of this pattern is to prioritize the structural clarity of the implementation, even at the cost of some redundancy.

## Implementation Overview

The pattern uses two layers of factory functions. The first layer, `_create_*_all()`, returns a tuple of interfaces organized by role. These include interfaces like `_Constant` and `_State`, representing constants and internal state accessors, respectively. Each interface can be independently tested.

### Structure Overview (Pseudo-code):

```python
from abc import ABC, abstractmethod
from typing import Protocol
from dataclasses import dataclass

# * = Interface name

class *(ABC):
    __slots__ = ()
    # Public-facing interface. Defined using ABC for static type checking.
    @abstractmethod
    def greet(self) -> None:
        ...

class _*Constant(Protocol):
    # Defines constants used internally in the factory
    CONSTANT: str

class _*State(Protocol):
    # Provides access to mutable internal state
    internal_state: str

class _*Core(Protocol):
    # Internal-use-only methods
    # 'initialize()' is callable from the outside to trigger initialization.
    def init_process_a(self) -> None: ...
    def init_process_b(self) -> None: ...
    def init_process_c(self) -> None: ...
    def initialize() -> None: ...

class _*Bridge(Protocol):
    # An interface for delegating functionality to related entities
    def is_initialized() -> bool: ...

def _create_*_all() -> tuple[*, _*Constant, _*State, _*Core, _*Bridge]:  # 1
    # In this pseudo-code, _create_*_all() returns all interfaces, 
    # but _*Constant, _*State, _*Core, and _*Bridge may not be present. 
    # The * interface is always present.
    # Specifically, _*Core might not exist if initialization is assumed to be done through function arguments.

    @dataclass(slots=True)  # 2, 3
    class _Constant(_*Constant):
        CONSTANT: str = "value"
    constant = _Constant()

    @dataclass(slots=True)
    class _State(_*State):
        internal_state: str = "not initialized"
    state = _State()  # 4

    class _Core(_*Core):
        __slots__ = ()
        def init_process_a(self) -> None:
            state.internal_state = "start initialization"
        def init_process_b(self) -> None:
            state.internal_state = "initialization is still in progress"
        def init_process_c(self) -> None:
            state.internal_state = "initialized"
        def initialize(self):
            self.init_process_a()
            self.init_process_b()
            self.init_process_c()

    core = _Core()

    class _Bridge(_*Bridge):
        __slots__ = ()
        def is_initialized(self):
            return state.internal_state == "initialized"
    bridge = _Bridge()

    class _Interface(*):
        __slots__ = ()
        def greet(self):
            message = 'good morning!' if state.internal_state == 'initialized' else 'zzz...'
            print(message)
    interface = _Interface()

    return (interface, constant, state, core, bridge)

def create_*() -> *:
    roles = _create_*_all()
    core = roles[3]
    core.initialize()
    interface = roles[0]
    return interface
```

1. Factory function names (such as *Interface name*) are written in snake\_case.
2. The `_Constant` class is instantiated to maintain consistency with other interfaces, though this is not strictly required.
3. The `_Constant` class is defined as a dataclass without the `frozen` attribute. This allows for modifications to constants during testing, but setting `frozen` would prevent accidental changes, increasing safety.
4. The `state` is mutable, allowing any desired state to be created during testing.

### Additional Notes

* In the pseudo-code, `__slots__` is defined for all interface implementations, but this is optional. (This is done to prevent accidental state injection into the interfaces, but it is a matter of preference.)
* Related to the previous point, all interfaces could be implemented as classes without the need for instantiation. However, doing so would require metaclasses to use `__slots__` or `@classmethod` methods, so for simplicity, all are instantiated. This choice is a matter of implementation preference, and either approach can be used as long as it accurately expresses the interface.
* When adopting this pattern for lightweight, high-volume instances, it might be worth considering using classes for all interfaces to eliminate the need for instantiating them.
* Overall, the structure might seem somewhat verbose, but this redundancy is aimed at preserving incremental testing and static type checking.
* Although all interfaces could be implemented with `ABC`, `Protocol` is used for internal interfaces to avoid boilerplate decorators. These choices can be adjusted according to the developer's preference or needs.
