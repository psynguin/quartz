> [!abstract] What We'll Cover
> In this chapter, we explore the core concepts and mechanics of the topic.


Welcome to Chapter 9! You've come a long way in your object-oriented journey. Up until now, we've covered the fundamentals and intermediate mechanics of Python's OOP. Now, we are entering the realm of **advanced mechanics**. 

These tools and concepts allow developers to write cleaner, more efficient, and highly customizable code. While you won't need these features for every single application you write, understanding them unlocks the true power of Python's object model. If you ever plan on writing robust libraries, frameworks, or enterprise-grade software, mastering these concepts is absolutely essential!

---

## 1. Data Classes (`dataclasses`)

Let's start with something that will instantly make your life easier. Introduced in Python 3.7, the `dataclasses` module provides a decorator that automatically generates special methods (like `__init__`, `__repr__`, and `__eq__`) for you. 

> [!info] What is a Data Class?
> A data class is essentially a class designed primarily to store state or data. By using the `@dataclass` decorator, you significantly reduce the boilerplate code required to set up these classes.

### Concepts and Benefits

- **Boilerplate Reduction:** Eliminates the need to write repetitive `__init__` methods.
- **Readability:** Class definitions become declarative, making it obvious what data the class holds.
- **Type Hints:** It relies heavily on Python's type annotations, enforcing cleaner code.

> [!example] Basic Data Class
> Notice how we don't have to write an `__init__` or `__repr__` method here. Python handles it for us!
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
    z: float = 0.0  # Default value

# __init__ is automatically generated
p1 = Point(1.5, 2.5)
p2 = Point(1.5, 2.5)

# __repr__ is automatically generated
print(p1)  # Output: Point(x=1.5, y=2.5, z=0.0)

# __eq__ is automatically generated (compares fields)
print(p1 == p2)  # Output: True
```

### Advanced Features

#### `field()` for Customization
If you need more control over a specific field—such as excluding it from `__repr__` or creating a default factory for mutable types—you can use the `field()` function.

> [!warning] Mutable Defaults
> Using a list or dictionary directly as a default value in a class is dangerous because the state is shared across all instances. Always use `field(default_factory=...)` for mutable defaults!

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Team:
    name: str
    # Using a list directly as a default is dangerous (shared state).
    # Use field(default_factory=...) instead.
    members: List[str] = field(default_factory=list)
    
    # Exclude internal state from string representation
    _internal_id: int = field(default=0, repr=False, compare=False)

team = Team("Developers")
team.members.append("Alice")
print(team) # Output: Team(name='Developers', members=['Alice'])
```

#### Post-Initialization Processing
Sometimes you need to initialize fields that depend on other fields. You can do this by defining the `__post_init__` method, which is automatically called at the end of the generated `__init__`.

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False) # Tell dataclass not to put this in __init__

    def __post_init__(self):
        self.area = self.width * self.height

rect = Rectangle(10, 5)
print(rect.area)  # Output: 50.0
```

#### Immutability (`frozen=True`)
You can make data classes immutable (similar to tuples) by passing `frozen=True` to the decorator. This makes them hashable and safe to use as dictionary keys.

> [!tip] Hashable Data
> If you need your objects to act as keys in a dictionary or be stored in a set, `frozen=True` is the way to go.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ImmutableConfig:
    host: str
    port: int

config = ImmutableConfig("localhost", 8080)
# config.port = 9000  # Raises dataclasses.FrozenInstanceError
```

---

## 2. Memory Optimization with `__slots__`

By default, Python objects store their instance attributes in a dynamic dictionary (`__dict__`). This allows for incredible flexibility, such as adding attributes at runtime. However, dictionaries have a significant memory overhead. If you create millions of instances of a simple class, this overhead can become a massive bottleneck.

### How `__slots__` Works

You can define a special class attribute called `__slots__` to explicitly declare all instance attributes. When Python sees `__slots__`, it allocates space for exactly those attributes and completely suppresses the creation of `__dict__` for each instance.

> [!example] `__dict__` vs `__slots__` Memory Usage
```python
import sys

class RegularPoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class SlottedPoint:
    __slots__ = ['x', 'y']
    
    def __init__(self, x, y):
        self.x = x
        self.y = y

p_reg = RegularPoint(1, 2)
p_slot = SlottedPoint(1, 2)

print(sys.getsizeof(p_reg) + sys.getsizeof(p_reg.__dict__)) # Memory overhead of __dict__
print(sys.getsizeof(p_slot)) # Considerably smaller

# Dynamic assignment fails on slotted objects
# p_slot.z = 3  # Raises AttributeError: 'SlottedPoint' object has no attribute 'z'
```

> [!caution] Caveats and Considerations
> - **No Dynamic Attributes:** You cannot add attributes that are not listed in `__slots__`. 
> - **Inheritance Constraints:** Subclasses do not automatically inherit `__slots__`. A subclass will have a `__dict__` unless it also explicitly defines `__slots__`.
> - **Multiple Inheritance:** You cannot use multiple inheritance if more than one parent class has non-empty `__slots__`.

---

## 3. Descriptors (The Descriptor Protocol)

A **descriptor** is any object that defines the methods `__get__()`, `__set__()`, or `__delete__()`. When a class attribute is a descriptor, its attribute access behavior is overridden by these methods.

In fact, much of Python's "magic" behind the scenes (like properties, regular methods, `classmethod`, and `staticmethod`) is implemented using descriptors under the hood!

### The Protocol

- `__get__(self, obj, objtype=None)`: Called when the attribute is accessed.
- `__set__(self, obj, value)`: Called when the attribute is assigned a value.
- `__delete__(self, obj)`: Called when the attribute is deleted.

> [!note] Data vs Non-Data Descriptors
> If an object defines `__set__` or `__delete__`, it is considered a **data descriptor**. If it only defines `__get__`, it is a **non-data descriptor** (like a typical method). Data descriptors take precedence over instance dictionaries.

### Use Case: Reusable Validation
Descriptors are perfect for extracting reusable property logic out of a class. For example, you can build a descriptor to ensure an attribute is always a specific type and within a given range.

> [!example] Typed Range Validator
```python
class TypedRange:
    """A data descriptor that validates type and range."""
    
    def __init__(self, name, expected_type, min_val, max_val):
        self.name = name  # The name of the attribute
        self.expected_type = expected_type
        self.min_val = min_val
        self.max_val = max_val

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__[self.name]

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"Expected {self.expected_type.__name__}")
        if not (self.min_val <= value <= self.max_val):
            raise ValueError(f"Value must be between {self.min_val} and {self.max_val}")
        # Store in the instance's dictionary
        obj.__dict__[self.name] = value

class Player:
    # Descriptors are instantiated at the class level
    level = TypedRange("level", int, 1, 100)
    health = TypedRange("health", float, 0.0, 100.0)

    def __init__(self, name, level, health):
        self.name = name
        self.level = level    # Triggers TypedRange.__set__
        self.health = health  # Triggers TypedRange.__set__

player = Player("Hero", 1, 100.0)
# player.level = 150  # Raises ValueError
# player.health = "high" # Raises TypeError
```

---

## 4. Object Creation (`__new__` vs `__init__`)

While most developers think of `__init__` as the constructor, it is actually the **initializer**. The true **constructor** is a method called `__new__`.

### The Lifecycle of Instantiation

When you call `MyClass(args)`, Python implicitly does the following behind the scenes:
1. Calls `MyClass.__new__(MyClass, args)` to allocate memory and create a new instance.
2. If `__new__` returns an instance of `MyClass`, Python then calls `__init__(self, args)` on that instance to initialize its state.

> [!info] Properties of `__new__`
> - It is a static method (no need for the `@staticmethod` decorator).
> - Its first argument is the class (`cls`) being instantiated.
> - It must return an instance object (usually by calling `super().__new__(cls)`).

### When to Override `__new__`

#### 1. Subclassing Immutable Types
Because immutable types like `int`, `str`, or `tuple` cannot be changed after they are created, you cannot initialize them in `__init__`. You must customize them during creation in `__new__`.

```python
class PositiveInt(int):
    def __new__(cls, value):
        if value < 0:
            raise ValueError("Value cannot be negative")
        # Call the parent's __new__ to actually create the integer
        return super().__new__(cls, value)

p = PositiveInt(10)
# n = PositiveInt(-5)  # Raises ValueError
```

#### 2. The Singleton Pattern
Sometimes you need to ensure a class only ever has a single instance. Overriding `__new__` makes this possible.

```python
class DatabaseConnection:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            print("Creating the single instance...")
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        # Note: __init__ is called every time DatabaseConnection() is invoked
        # You might need logic to prevent re-initialization
        self.connected = True

db1 = DatabaseConnection()
db2 = DatabaseConnection()
print(db1 is db2)  # Output: True
```

---

## 5. Metaclasses

In Python, everything is an object, including classes themselves! If a class is an object, what class instantiated it? The answer is a **metaclass**.

By default, the metaclass of all classes is `type`. 
- An object is an instance of a class.
- A class is an instance of a metaclass.

### How Classes are Created

When Python executes a `class` block, it gathers the class name, its base classes, and a dictionary of its attributes/methods. It then calls the metaclass (usually `type`) to instantiate the class object:
`type(name, bases, attrs)`

### Defining a Custom Metaclass

You can intercept the creation of a class by defining a custom metaclass that inherits from `type` and overriding `__new__` or `__init__`. You then assign it to a class using the `metaclass` keyword argument.

> [!example] Use Case: Enforcing Coding Standards
> Imagine we want to write a framework that enforces that every method in a class must have a docstring.
```python
class RequireDocstringsMeta(type):
    def __new__(mcs, name, bases, namespace):
        # mcs: The metaclass itself
        # name: Name of the class being created
        # bases: Tuple of base classes
        # namespace: Dictionary of class attributes/methods
        
        # Don't apply rules to the metaclass creation itself or base objects
        if name != 'BaseWithDocstrings':
            for attr_name, attr_value in namespace.items():
                if callable(attr_value) and not attr_name.startswith('__'):
                    if not attr_value.__doc__:
                        raise TypeError(f"Method '{attr_name}' in class '{name}' is missing a docstring!")
        
        # Proceed with normal class creation
        return super().__new__(mcs, name, bases, namespace)

class BaseWithDocstrings(metaclass=RequireDocstringsMeta):
    pass

# This works perfectly
class GoodClass(BaseWithDocstrings):
    def calculate(self):
        """Calculates the result."""
        return 42

# Uncommenting the following class will raise a TypeError at import/compile time!
# class BadClass(BaseWithDocstrings):
#     def calculate(self):
#         return 42
```

> [!example] Use Case: Automatic Registration (Registry Pattern)
> Metaclasses are excellent for automatically keeping track of subclasses, which is a common pattern in plugins or serialization frameworks.
```python
class PluginRegistry(type):
    registry = {}

    def __new__(mcs, name, bases, namespace):
        # Create the class
        cls = super().__new__(mcs, name, bases, namespace)
        # Register the class
        if name != 'BasePlugin':
            PluginRegistry.registry[name] = cls
        return cls

class BasePlugin(metaclass=PluginRegistry):
    pass

class AudioPlugin(BasePlugin):
    pass

class VideoPlugin(BasePlugin):
    pass

print(PluginRegistry.registry) 
# Output: {'AudioPlugin': <class '__main__.AudioPlugin'>, 'VideoPlugin': <class '__main__.VideoPlugin'>}
```

> [!abstract] A Word of Caution
> Metaclasses are a powerful tool often referred to as "black magic". They should be used sparingly. If you can solve a problem using inheritance, decorators, or descriptors, those are generally preferred over metaclasses for the sake of simplicity and readability. 
> 
> *“Metaclasses are deeper magic than 99% of users should ever worry about. If you wonder whether you need them, you don't.” — Tim Peters*

---

## 6. Type Hinting for OOP

As an intermediate developer, you're likely familiar with basic type hints (`int`, `str`, `List`). However, typing object-oriented structures introduces specific challenges and advanced typing features.

### Returning `Self`
When a method returns the instance itself (common in fluent interfaces or builder patterns), you can use the `Self` type hint from the `typing` module.

```python
from typing import Self

class QueryBuilder:
    def select(self, fields: str) -> Self:
        self.fields = fields
        return self

    def where(self, condition: str) -> Self:
        self.condition = condition
        return self
```

### Structural Subtyping with `Protocol`
How do you type-hint for **Duck Typing**? If a function accepts *anything* with a `.read()` method, how do you express that without forcing inheritance? Enter `typing.Protocol`.

> [!info] Protocol (Structural Subtyping)
> A `Protocol` allows you to define an interface implicitly. If an object implements the methods defined in the protocol, type checkers consider it a match, even if it doesn't explicitly inherit from the Protocol!

```python
from typing import Protocol

# Define the expected behavior
class Readable(Protocol):
    def read(self) -> str:
        ...

# A function that expects ANY Readable object
def print_content(document: Readable) -> None:
    print(document.read())

class FakeFile:
    def read(self) -> str:
        return "Fake content"

# Type checkers will accept this, because FakeFile has a read() returning a string!
print_content(FakeFile())
```

---

Now that you've grasped these advanced concepts—from metaclasses to protocols—you're ready to write professional, highly optimized, and "Pythonic" code. Keep experimenting with these features in your own side projects to truly solidify your understanding!
