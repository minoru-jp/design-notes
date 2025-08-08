# Verbose Pattern in Python

## Introduction

The term **"Verbose Pattern"** is an informal name used in this document to describe the structure. It does not correspond to any formal recognized design pattern.

This structure shares some conceptual similarities with the following design patterns and principles:

* **Factory Pattern**: Delegating object creation to dedicated functions
* **Interface Segregation Principle**: Separating interfaces by role
* **Facade Pattern**: Hiding internal complexity behind a simplified interface
* **Strategy Pattern**: Separating behavior based on roles

This pattern is designed with a two-layer factory function structure to support step-by-step testing and modular development. It’s especially suited for *long-lived, low-volume, fundamental components* that require structured and maintainable implementation.

---

## Key Features

* Clearly separates external interfaces from internal/test-time interfaces.
* Enables expression of a *defined but not yet meaningfully initialized* state, making incremental testing of the initialization process easier.
* Interfaces are grouped by role, so each aspect of the system can be tested independently.
* Introducing a `_Bridge` interface allows for structured delegation between factory functions.
* Only the necessary interfaces are exposed, so code size tends to grow proportionally with complexity.

---

## Notes

* This pattern does not enable anything fundamentally new compared to standard class instances.
* The same goals can also be achieved using regular class-based implementations.
* However, one general characteristic of closures is that the internal state referenced by external interfaces is not easily accessible. As a result, the risk of unintended access to internal state tends to be lower than with class instances. That said, this is not unique to this pattern—it applies more broadly to any interface exposed through factory functions.
* What this pattern emphasizes is the prioritization of structural clarity over brevity, even at the cost of some redundancy in implementation.

---

## Implementation Overview

This pattern uses two layers of factory functions. The first layer, `_create_*_all()`, returns a tuple of interfaces organized by role.
These include interfaces like `_ * Constant` and `_ * State`, representing constants and internal state accessors respectively.
Each interface can be independently tested.

### Structure Overview (Pseudo-code):

```python
from abc import ABC, abstractmethod
from typing import Protocol, ClassVar

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
    # Internal-use-only methods. 
    # 'initialize()' is callable from the outside to trigger initialization.
    def init_process_a(self) -> None: ...
    def init_process_b(self) -> None: ...
    def init_process_c(self) -> None: ...
    def initialize() -> None:
        ...

class _*Bridge(Protocol):
    # An interface for delegating part of the functionality to related entities
    def is_initialized() -> bool: ...

def _create_*_all() -> tuple[*, _*Constant, _*State, _*Core, _*Bridge]:
    # This function returns all role-based interfaces.
    # Some roles (like _*Core or _*Bridge) may not be needed, depending on the use case.
    # The * interface (public) is always present.

    class _Constant(_*Constant):
        __slots__ = ('CONSTANT',)
        def __init__(self):
            self.CONSTANT = "value"
    constant = _Constant()
    
    class _State(_*State):
        __slots__ = ('internal_state',)
        def __init__(self):
            self.internal_state = "not initialized"
    state = _State()

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
    roles = _create_*_all(...)
    core = roles[3]
    core.initialize()
    interface = roles[0]
    return interface
```

* The actual factory function names use `snake_case`.
* Since the `state` interface allows direct mutation, it can be used in tests to simulate any internal condition.

---

## Additional Notes

* `__slots__` is used in each implementation to prevent unintended attribute injection, but it's optional and a matter of preference.
* All interfaces could technically be implemented as classes without instance creation. However, this would require metaclasses to use `__slots__`, or `@classmethod` methods. For simplicity, we use class instances here.
* If you're targeting lightweight, high-volume instances, it might be worth switching to class-based interfaces to reduce instantiation overhead.
* We use `Protocol` for internal interfaces to keep things lightweight and avoid boilerplate decorators, though `ABC` could also be used throughout.


