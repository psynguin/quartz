> [!abstract] What We'll Cover
> In this chapter, we explore the core concepts and mechanics of the topic.


Welcome to Chapter 8! By now, you've built solid foundations in inheritance, encapsulation, and polymorphism. But as your projects grow from a handful of scripts into large, modular systems, you will inevitably encounter situations where you need to lay down the law. You want to *force* subclasses to implement certain methods, establishing a strict contract that guarantees specific behaviors. 

This is exactly where **Abstract Base Classes (ABCs)** shine. 

---

## 1. Duck Typing vs. Formal Contracts

In Python, we traditionally rely on "duck typing"—the philosophy that "if it walks like a duck and quacks like a duck, it must be a duck." You informally expect certain methods to exist on an object (like `__iter__` or `__len__`).

> [!info] Interfaces vs. Abstract Classes
> * **Interface**: A blueprint or a contract specifying *what* methods a class must have, without dictating *how* they should be implemented.
> * **Abstract Class**: A class that cannot be instantiated on its own and is meant to be subclassed. It often serves as an interface but can also contain concrete (implemented) methods.

Duck typing is beautifully flexible, but informal protocols have a dark side: they can lead to unpredictable runtime errors deep in your call stack. Abstract Base Classes provide a formal way to define and enforce these interfaces, keeping your large codebases robust and predictable.

---

## 2. The Problem with `NotImplementedError`

Historically, Python developers tried to enforce class contracts by raising a `NotImplementedError` inside the base class methods. Let's see why this approach is flawed.

> [!example] The Old Way: Raising NotImplementedError
```python
class PaymentProcessor:
    def process_payment(self, amount):
        raise NotImplementedError("Subclasses must implement process_payment")

class CreditCardProcessor(PaymentProcessor):
    pass # Oops! We forgot to implement process_payment

# Instantiation succeeds. Wait, this is bad!
processor = CreditCardProcessor() 

# The error only occurs at RUNTIME when the method is actually called.
processor.process_payment(100) # Fails here with NotImplementedError
```

> [!warning] The Danger of Late Failure
> The major downside to the `NotImplementedError` approach is that instantiation actually *succeeds*. The missing implementation isn't caught until the specific method is executed. If that method is rarely called, the bug might lie dormant until it reaches production!

---

## 3. The Solution: Class Contracts with the `abc` Module

To solve this, Python provides the `abc` (Abstract Base Classes) module in the standard library. ABCs enforce contracts at **instantiation time**. If a subclass fails to implement all required abstract methods, Python raises a `TypeError` the moment you try to create an object.

Let's build a formal contract using `ABC` and `@abstractmethod`.

> [!example] The Modern Way: Using the `abc` Module
```python
from abc import ABC, abstractmethod

class DataReader(ABC):
    
    @abstractmethod
    def read(self, path: str) -> dict:
        """Reads data from a source and returns a dictionary."""
        pass
    
    @abstractmethod
    def close(self) -> None:
        """Closes the reader."""
        pass

class JSONReader(DataReader):
    
    def read(self, path: str) -> dict:
        import json
        with open(path, 'r') as f:
            return json.load(f)
            
    # We forgot to implement 'close' again!

# Instantiation fails immediately!
# TypeError: Can't instantiate abstract class JSONReader with abstract method close
reader = JSONReader() 
```

> [!tip] Document Your Abstract Methods
> While it is conventional to use `pass` or `...` inside an abstract method, it's actually an excellent place to include a docstring detailing the expected input and output of the contract.

**Key Behaviors of ABCs:**
1. **Cannot Instantiate**: Trying to instantiate the base class directly (`reader = DataReader()`) raises a `TypeError`.
2. **Incomplete Subclasses are Abstract**: A subclass like `JSONReader` remains abstract and cannot be instantiated until *all* abstract methods (like `close()`) are implemented.

---

## 4. Combining `@abstractmethod` with Other Decorators

As an intermediate Python developer, you can leverage a very powerful feature: combining `@abstractmethod` with other decorators like `@property`, `@classmethod`, or `@staticmethod`.

> [!warning] The Golden Rule of Decorator Stacking
> When stacking decorators with `@abstractmethod`, it must generally be the innermost decorator (the one closest to the `def` statement).

### Abstract Properties

You can force subclasses to provide a specific attribute by defining an abstract property.

> [!example] Enforcing Attributes with Abstract Properties
```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    
    @property
    @abstractmethod
    def wheel_count(self) -> int:
        pass

class Car(Vehicle):
    # Implemented as a property
    @property
    def wheel_count(self) -> int:
        return 4

class Motorcycle(Vehicle):
    # Implemented as a class attribute (Python allows this to satisfy the property contract!)
    wheel_count = 2 

class BrokenVehicle(Vehicle):
    pass

car = Car()
# broken = BrokenVehicle() # Raises TypeError: Can't instantiate abstract class with abstract method wheel_count
```

### Abstract Class Methods

Similarly, you can enforce the implementation of class methods, which is incredibly useful for factory patterns.

> [!example] Abstract Class Methods
```python
from abc import ABC, abstractmethod

class BaseFactory(ABC):
    
    @classmethod
    @abstractmethod
    def create(cls):
        pass

class UserFactory(BaseFactory):
    @classmethod
    def create(cls):
        return "New User"
```

---

## 5. Concrete Methods in ABCs

Unlike pure interfaces in some strictly typed languages (like Java), Python's abstract classes are not strictly limited to abstract methods. They can provide default behaviors or utility methods right alongside their abstract contracts.

This naturally leads us to the **Template Method Design Pattern**, where a concrete method outlines the skeleton of an algorithm, relying on abstract methods for the specific steps.

> [!example] Mixing Concrete and Abstract Methods
```python
from abc import ABC, abstractmethod

class ReportGenerator(ABC):
    
    @abstractmethod
    def gather_data(self):
        pass
        
    @abstractmethod
    def format_data(self, data):
        pass
        
    # Concrete method using abstract methods (Template Method Pattern)
    def generate(self):
        data = self.gather_data()
        formatted_report = self.format_data(data)
        self._save_report(formatted_report)
        return formatted_report
        
    # Concrete utility method
    def _save_report(self, report):
        print("Saving report to database...")
```

By designing your classes this way, you eliminate code duplication. The base class handles the orchestration (`generate`), while subclasses only need to focus on the unique details (`gather_data` and `format_data`).

---

## 6. ABCs in the Standard Library

> [!note] Did you know?
> Python uses ABCs extensively within its standard library, specifically in the `collections.abc` module, to define built-in container interfaces.

If you explore `collections.abc`, you'll find interfaces like:
* `Sequence`: Requires subclasses to implement `__getitem__` and `__len__`.
* `Iterable`: Requires subclasses to implement `__iter__`.
* `Mapping`: Requires subclasses to implement `__getitem__`, `__iter__`, and `__len__`.

By explicitly inheriting from these built-in ABCs, you hook directly into Python's native ecosystem, making your custom objects behave predictably and seamlessly with Python's typing and iteration structures.

---

## 7. Best Practices Summary

> [!abstract] Chapter 8 Recap
> Keep these principles in mind as you start defining contracts in your applications:
> * **Fail Fast**: Prefer `abc.ABC` over raising `NotImplementedError`. It catches contract violations at instantiation time rather than execution time.
> * **Interface Segregation**: Keep your interfaces small, focused, and specific. Don't force subclasses to implement methods they don't need (this is the 'I' in the SOLID principles).
> * **Abstract Attributes**: Use abstract properties (`@property` combined with `@abstractmethod`) to guarantee that specific state or attributes exist on your subclasses.
> * **Template Methods**: Utilize concrete methods inside your ABCs to reduce code duplication, orchestrating high-level logic while delegating specifics to the subclass abstract methods.
