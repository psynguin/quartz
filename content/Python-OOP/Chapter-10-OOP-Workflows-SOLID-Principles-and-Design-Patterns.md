> [!abstract] Introduction
> As your Python projects grow from small scripts into larger applications, how you organize and structure your Object-Oriented code becomes just as important as the code itself. This chapter explores how to transition from writing all your code in a single file to building modular, maintainable, and scalable OOP architectures. We will cover the **SOLID principles**—a set of design guidelines that help you write robust object-oriented software—and deeply contrast practical workflows for structuring Python projects.

## The SOLID Principles in Python

The SOLID principles are five design guidelines intended to make software designs more understandable, flexible, and maintainable. While they originated in languages like C++ and Java, they apply perfectly to Python, especially when combined with Pythonic twists.

### 1. Single Responsibility Principle (SRP)

> [!info] Principle
> **A class should have one, and only one, reason to change.** This means a class should only have one job or responsibility.

> [!warning] Anti-Pattern
> Mixing data management with file I/O operations breaks SRP.
```python
class Report:
    def __init__(self, data):
        self.data = data
        
    def generate_report(self):
        # Generates report data
        return f"Report Data: {self.data}"
        
    def save_to_file(self, filename):
        # Handles file I/O - a separate responsibility!
        with open(filename, 'w') as f:
            f.write(self.generate_report())
```

> [!example] Pythonic Approach
> Separate the concerns into different classes.
```python
class Report:
    def __init__(self, data):
        self.data = data
        
    def generate(self):
        return f"Report Data: {self.data}"

class ReportSaver:
    @staticmethod
    def save_to_file(report, filename):
        with open(filename, 'w') as f:
            f.write(report.generate())
```

### 2. Open/Closed Principle (OCP)

> [!info] Principle
> **Software entities (classes, modules, functions) should be open for extension, but closed for modification.** You should be able to add new functionality without changing existing code.

> [!tip] Pythonic Approach using Polymorphism
> Use Abstract Base Classes (`ABC`) to define a common interface. Notice how we can add new discount types without modifying the `Order` class.
```python
from abc import ABC, abstractmethod

class Discount(ABC):
    @abstractmethod
    def calculate(self, price):
        pass

class NoDiscount(Discount):
    def calculate(self, price):
        return price

class PercentageDiscount(Discount):
    def __init__(self, percentage):
        self.percentage = percentage
        
    def calculate(self, price):
        return price - (price * self.percentage / 100)

class Order:
    def __init__(self, price, discount_strategy: Discount):
        self.price = price
        self.discount_strategy = discount_strategy
        
    def get_total(self):
        return self.discount_strategy.calculate(self.price)
```

### 3. Liskov Substitution Principle (LSP)

> [!info] Principle
> **Objects of a superclass shall be replaceable with objects of its subclasses without breaking the application.** Subclasses must behave in a way that clients expect.

> [!warning] Anti-Pattern
> If a function expects a `Bird` and calls `fly()`, passing a `Penguin` breaks the program.
```python
class Bird:
    def fly(self):
        print("Flying")

class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("Penguins can't fly!")
```

> [!example] Pythonic Approach
> Redefine the class hierarchy so that subclasses genuinely adhere to the behavior of their parent.
```python
class Bird:
    pass

class FlyingBird(Bird):
    def fly(self):
        print("Flying")

class Penguin(Bird):
    def swim(self):
        print("Swimming")
```

### 4. Interface Segregation Principle (ISP)

> [!info] Principle
> **No client should be forced to depend on methods it does not use.** Instead of one fat interface, create many small, specific interfaces. 

Python doesn't have strict "interfaces" like Java or C#, but we use Abstract Base Classes (ABCs) or simple duck typing to implement this concept.

> [!example] Pythonic Approach
```python
from abc import ABC, abstractmethod

class Printer(ABC):
    @abstractmethod
    def print_document(self): pass

class Scanner(ABC):
    @abstractmethod
    def scan_document(self): pass

# A simple printer only needs to implement Printer
class BasicPrinter(Printer):
    def print_document(self):
        print("Printing...")

# A multi-function device implements both
class MultiFunctionCopier(Printer, Scanner):
    def print_document(self):
        print("Printing...")
        
    def scan_document(self):
        print("Scanning...")
```

### 5. Dependency Inversion Principle (DIP)

> [!info] Principle
> **High-level modules should not depend on low-level modules. Both should depend on abstractions.** Rely on abstract classes or protocols rather than concrete implementations.

> [!tip] Pythonic Approach (Dependency Injection)
> We depend on the abstraction `Database`, not a concrete class like `MySQLDatabase`.
```python
class Database(ABC):
    @abstractmethod
    def save(self, data): pass

class MySQLDatabase(Database):
    def save(self, data):
        print(f"Saving {data} to MySQL")

class Application:
    # Application depends on the abstraction
    def __init__(self, db: Database):
        self.db = db
        
    def process(self, data):
        self.db.save(data)

# Inject the concrete dependency at runtime
db = MySQLDatabase()
app = Application(db)
app.process("User Info")
```

---

## OOP Workflows and Project Architecture

When transitioning to OOP, how you organize your files is crucial. The jump from a single file to a modular project changes the way you think about code organization. Let's highlight the contrast between different architectural workflows.

### Workflow 1: The Single Document (Scripting / Prototyping)

When starting out or building a small script, it's common to place all classes, functions, and execution logic in a single file (e.g., `app.py`).

> [!note] Single-File Workflow
> **Pros:** 
> - Easy to navigate for small projects.
> - Simple to run (`python app.py`).
> 
> **Cons:** 
> - Quickly becomes unmaintainable. 
> - Difficult to navigate beyond 300-500 lines. 
> - Causes severe merge conflicts in team environments.

> [!tip] Best Practice for Single Files
> Always separate your class definitions from your execution logic using the `__name__ == "__main__"` idiom.
```python
# app.py
class DataProcessor:
    def process(self):
        pass

def main():
    processor = DataProcessor()
    processor.process()

if __name__ == "__main__":
    main()
```

### Workflow 2: Module-Based Structure

As the project grows, apply the **Single Responsibility Principle** to your file structure. Group related classes into modules (files) and packages (directories). This is where the contrast becomes stark: instead of scrolling through hundreds of lines, you navigate a file tree.

> [!example] Directory Structure
```text
my_project/
│
├── main.py              # Entry point
├── models/              # Package for data classes
│   ├── __init__.py
│   ├── user.py          # Contains User class
│   └── product.py       # Contains Product class
│
└── services/            # Package for business logic
    ├── __init__.py
    ├── payment.py       # PaymentProcessor class
    └── notification.py  # EmailSender class
```

> [!tip] Modular Benefits
> - **High Cohesion, Low Coupling:** Files do exactly what their name implies.
> - **Scalability:** Easy to find specific code.
> - **Collaboration:** Teams can work on different files without constant merge conflicts.

### Workflow 3: Import Strategies in OOP Packages

When structuring modular projects, how you import classes dramatically affects the developer experience. Let's look at the contrast between explicit imports and package-level facades.

#### Approach A: Explicit Imports (The Pythonic Way)
Modules import exactly what they need from where it lives.

```python
# main.py
from models.user import User
from services.payment import PaymentProcessor
```
> [!note] Pros & Cons
> **Pros:** Explicit, clear dependencies.  
> **Cons:** Import paths can become long and repetitive.

#### Approach B: Facade Pattern via `__init__.py`
You can use `__init__.py` to expose specific classes at the package level, creating a clean API for your modules.

```python
# models/__init__.py
from .user import User
from .product import Product

__all__ = ['User', 'Product']
```

Now, the client code can import from the package directly:
```python
# main.py
from models import User, Product
```

> [!warning] Beware Circular Imports
> **Pros:** Hides internal package structure and keeps imports much cleaner.  
> **Cons:** Can cause circular imports if modules within the package try to import from the `__init__.py` prematurely.

---

## Applying SOLID Principles to File Structure

> [!abstract] Architectural SOLID
> - **SRP in Files:** Just as a class should have one reason to change, a module (file) should ideally represent a single domain or concept. Don't put Database connections and UI logic in the same module.
> - **OCP in Architecture:** Use plugins or strategy patterns across files. A new feature should mean adding a new file, not heavily modifying an existing core file.
> - **DIP in Packages:** Create an `interfaces.py` or `protocols.py` module to define your ABCs or `typing.Protocol`s. High-level modules import these interfaces, and low-level modules implement them, avoiding tight coupling between concrete modules.
