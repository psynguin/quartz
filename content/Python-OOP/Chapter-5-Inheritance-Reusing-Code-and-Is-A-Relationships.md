> [!abstract] What We'll Cover
> In this chapter, we explore the core concepts and mechanics of the topic.


Welcome back! So far, you've learned how to encapsulate data and behavior within a single class. You know how to build blueprints. But what happens when you need to build a *new* blueprint that's very similar to an existing one?

Imagine you're building a system for a zoo. You have an `Animal` class, but then you need specific classes for `Lion`, `Penguin`, and `Snake`. Do you copy and paste the `eat()` and `sleep()` methods into every single class?

> [!warning] The DRY Principle
> **Don't Repeat Yourself (DRY)** is a core tenet of software engineering. Copy-pasting code leads to maintenance nightmares. If you find a bug in the `sleep()` method, you don't want to fix it in 50 different places!

Inheritance is Object-Oriented Programming's solution to the DRY principle. It allows you to create new classes based on existing ones, inheriting their attributes and methods while adding or modifying specific behaviors. Let's dive in.

---

## The Concept of Inheritance: "Is-A" Relationships

Inheritance models an **"is-a"** relationship. A Dog *is an* Animal. A Manager *is an* Employee. A SavingsAccount *is a* BankAccount.

When you use inheritance, you define two types of classes:

> [!info] Key Terminology
> - **Base Class (Parent/Superclass):** The existing class whose features are being inherited. It provides the general blueprint.
> - **Derived Class (Child/Subclass):** The new class that inherits features from the base class. It specializes the general blueprint.

Let's see this in action. We'll define a generic `Animal` base class and a `Dog` derived class.

```python
class Animal:
    """Base class representing a generic animal."""
    def __init__(self, name: str):
        self.name = name

    def eat(self):
        print(f"{self.name} is eating.")

class Dog(Animal):
    """Derived class representing a dog."""
    def bark(self):
        print(f"{self.name} says woof!")

# Let's create a Dog instance
my_dog = Dog("Buddy")

# The dog can use methods inherited from Animal
my_dog.eat()  # Output: Buddy is eating.

# The dog can also use its own specific methods
my_dog.bark() # Output: Buddy says woof!
```

> [!note] Syntax Spotlight
> To inherit from a class in Python, simply place the base class name inside parentheses after the derived class name: `class DerivedClass(BaseClass):`.

By inheriting from `Animal`, the `Dog` class automatically gets the `__init__` method (which sets the `name` attribute) and the `eat` method. We didn't have to write them again! We only wrote the code that makes a `Dog` unique: the `bark` method.

---

## Subclassing and Overriding Methods

Inheriting methods is great, but what if the base class's behavior isn't *quite* right for the derived class? Subclassing isn't just about adding new methods; it's also about customizing existing ones.

When a derived class defines a method with the exact same name as a method in its base class, it is called **overriding**.

> [!abstract] Overriding
> Overriding allows a subclass to provide a specific implementation of a method that is already provided by its superclass. This is a crucial part of **polymorphism** (which we'll explore deeper in the next chapter).

Let's look at an `Employee` and `Manager` scenario.

```python
class Employee:
    def __init__(self, name: str, base_salary: float):
        self.name = name
        self.base_salary = base_salary
        
    def calculate_pay(self) -> float:
        """Calculates standard employee pay."""
        return self.base_salary

class Manager(Employee):
    def __init__(self, name: str, base_salary: float, bonus: float):
        self.name = name
        self.base_salary = base_salary
        self.bonus = bonus
        
    def calculate_pay(self) -> float:
        """Overrides Employee.calculate_pay to include a bonus."""
        return self.base_salary + self.bonus

# Let's test it out
emp = Employee("Alice", 50000)
mgr = Manager("Bob", 60000, 15000)

print(emp.calculate_pay()) # Output: 50000
print(mgr.calculate_pay()) # Output: 75000
```

Notice how `Manager` overrides `calculate_pay()`. When you call `calculate_pay()` on a `Manager` instance, Python looks for that method in the `Manager` class first. Finding it there, it executes the overridden version, completely ignoring the `Employee` version.

However, if you look closely at the `Manager.__init__` method, you might notice something annoying. We are repeating the assignment of `self.name` and `self.base_salary`. This violates the DRY principle we just talked about!

---

## Enter `super()`: The Hero of Hierarchy

How do we initialize the base class without copying and pasting its code into the derived class? How do we extend an overridden method instead of completely replacing it?

The answer is the built-in `super()` function.

> [!tip] The `super()` Function
> `super()` returns a proxy object that delegates method calls to a parent or sibling class. It allows you to call methods of the superclass from within the subclass.

Let's refactor our `Employee` and `Manager` classes to use `super()`. This is the **idiomatic** way to write Python.

```python
class Employee:
    def __init__(self, name: str, base_salary: float):
        self.name = name
        self.base_salary = base_salary
        
    def calculate_pay(self) -> float:
        return self.base_salary

class Manager(Employee):
    def __init__(self, name: str, base_salary: float, bonus: float):
        # Idiomatic way to initialize the base class
        super().__init__(name, base_salary)
        self.bonus = bonus
        
    def calculate_pay(self) -> float:
        # Reusing the base class's logic instead of repeating it
        base_pay = super().calculate_pay()
        return base_pay + self.bonus

mgr = Manager("Charlie", 70000, 20000)
print(f"Manager {mgr.name} earns ${mgr.calculate_pay()}")
# Output: Manager Charlie earns $90000
```

### Why is `super()` so important?

Using `super()` isn't just about saving a few keystrokes. It makes your code significantly more robust:

1. **Avoids Hardcoding:** By using `super()`, you don't have to explicitly name the parent class (`Employee`). If you ever rename `Employee` or change the class hierarchy, you don't have to hunt down and update all the hardcoded references in your subclasses.
2. **Reusability and Extension:** As seen in `calculate_pay()`, we used `super().calculate_pay()` to get the base salary, and then *added* the bonus to it. We extended the behavior rather than rewriting the core logic from scratch.
3. **Multiple Inheritance:** Python supports inheriting from more than one class at a time. In complex hierarchies, calling `super()` ensures that the correct parent methods are called in the right order exactly once, preventing nasty bugs like the "Diamond Problem".

> [!example] Advanced Topic: Method Resolution Order (MRO)
> When you call a method on an object, Python searches the object's class for the method. If it's not there, it searches the parent class, and so on. The exact order Python follows is called the **Method Resolution Order (MRO)**. 
> 
> Python uses an algorithm called C3 linearization to determine this order. You can actually peek under the hood and see a class's MRO using the special `__mro__` attribute!
> 
```python
print(Manager.__mro__)
# Output: (<class '__main__.Manager'>, <class '__main__.Employee'>, <class 'object'>)
```
> This output shows that Python checks `Manager` first, then `Employee`, and finally the built-in `object` base class (which every class in Python ultimately inherits from).

Inheritance is a powerful tool, but it should be used judiciously. A common pitfall for beginners is creating massive, deep inheritance trees that become impossible to manage. Remember the rule of thumb: favor inheritance strictly for "is-a" relationships.

## Custom Exceptions: A Practical Use Case

One of the most immediate and practical uses of inheritance for intermediate Python developers is creating custom exceptions. Instead of raising a generic `ValueError` or `Exception`, you can create descriptive, domain-specific errors by subclassing Python's built-in `Exception` class.

```python
class InsufficientFundsError(Exception):
    """Raised when an account has insufficient funds for a transaction."""
    pass
    
class BankAccount:
    def __init__(self, balance):
        self.balance = balance
        
    def withdraw(self, amount):
        if amount > self.balance:
            raise InsufficientFundsError(f"Cannot withdraw {amount}. Balance is {self.balance}.")
        self.balance -= amount
```

> [!tip] Clean Error Handling
> Creating custom exceptions allows you to `except` them specifically, making your error handling logic much cleaner and more robust!

In the next chapter, we'll explore what happens when you inherit from more than one class in Multiple Inheritance and Mixins!
