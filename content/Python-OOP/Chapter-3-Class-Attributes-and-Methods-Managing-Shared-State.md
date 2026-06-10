> [!abstract] What We'll Cover
> In this chapter, we explore the core concepts and mechanics of the topic.


Welcome back! Up until now, we've focused heavily on the individual objects we create. We've built classes, instantiated unique objects from them, and used `self` to assign and modify variables specific to those objects. 

But what if we want to share data across *all* instances of a class? Or what if we want a method that belongs to the class itself, rather than to any single object? 

In this chapter, we are going to look beyond the individual instance. We'll explore class attributes, class methods, and static methods, learning how they differ from the instance-level tools we've used so far.

---

## Instance Attributes vs. Class Attributes

In Python, an attribute is simply a variable bound to an object or a class. Understanding where and how these variables are bound is crucial for managing the state of your application effectively.

### Instance Attributes

If you've written an `__init__` method, you are already intimately familiar with instance attributes. 

> [!info] Definition: Instance Attribute
> Variables that belong to a specific instance of a class. They are unique to each object.

When you modify an instance attribute on one object, it has absolutely zero effect on any other objects created from the same class. They live entirely in their own silos.

```python
class Employee:
    def __init__(self, name, salary):
        # These are instance attributes
        self.name = name      
        self.salary = salary  

emp1 = Employee("Alice", 75000)
emp2 = Employee("Bob", 80000)

# We are only modifying emp1's state
emp1.salary = 77000  

print(emp1.salary)   # Output: 77000
print(emp2.salary)   # Output: 80000
```

### Class Attributes

Now, let's zoom out. What if we have a constant that every employee shares, or a counter that tracks how many employees exist in total? That's where class attributes shine.

> [!info] Definition: Class Attribute
> Variables that belong to the class itself, rather than any specific instance. They are shared across all instances of the class.

Class attributes are defined directly inside the class body, usually right at the top, outside of any methods.

```python
class Employee:
    # These are CLASS attributes
    company_name = "TechCorp"
    raise_amount = 1.05
    num_of_emps = 0

    def __init__(self, name, salary):
        self.name = name
        self.salary = salary
        
        # Accessing the class attribute directly via the class name
        Employee.num_of_emps += 1

    def apply_raise(self):
        # Accessing the class attribute via self
        self.salary = int(self.salary * self.raise_amount)
```

> [!note] How to Access Class Attributes
> You can access a class attribute in two ways:
> 1. Via the class name: `Employee.company_name`
> 2. Via an instance: `self.company_name` (or `emp1.company_name`)

Wait, why does `self.raise_amount` work if `raise_amount` is a class attribute? When you access an attribute via an instance, Python first checks if the instance has that attribute. If it doesn't, Python looks up the chain and checks if the *class* has it.

#### The "Shadowing" Gotcha

This lookup order (checking the instance first, then the class) leads to one of the most common pitfalls for beginners. 

What happens if we try to *modify* a class attribute through an instance?

```python
emp1 = Employee("Alice", 70000)
emp2 = Employee("Bob", 80000)

# Attempting to change the raise amount for just emp1
emp1.raise_amount = 1.10 

print(Employee.raise_amount) # 1.05
print(emp1.raise_amount)     # 1.10
print(emp2.raise_amount)     # 1.05
```

> [!warning] The Shadowing Gotcha
> By assigning `emp1.raise_amount = 1.10`, we did **not** modify the class attribute. Instead, we created a brand new *instance* attribute on `emp1` named `raise_amount`. Because Python checks the instance first, this new instance attribute "shadows" (or hides) the class attribute for `emp1`.

We can prove this by looking at the `__dict__` magic attribute, which stores the namespace of an object as a dictionary:

```python
# emp1 now has its own 'raise_amount'
print(emp1.__dict__) 
# {'name': 'Alice', 'salary': 70000, 'raise_amount': 1.1}

# emp2 only has name and salary. It falls back to the class for 'raise_amount'
print(emp2.__dict__) 
# {'name': 'Bob', 'salary': 80000}
```

---

## Class Methods (`@classmethod`)

Just as we have instance variables and class variables, we have different types of methods.

Regular instance methods (like `apply_raise` above) automatically take `self` as their first parameter. This gives the method access to the state of the specific object calling it.

Class methods, however, automatically take the *class itself* as their first parameter. By convention, we name this parameter `cls` (since `class` is a reserved keyword in Python).

> [!abstract] Class Methods Overview
> - **Decorator**: `@classmethod`
> - **First Parameter**: `cls`
> - **Purpose**: To access or modify class-level state, or to provide alternative constructors.

### Use Case 1: Modifying Class State

If you want a method that modifies a class attribute across the board, you should use a class method.

```python
class Employee:
    raise_amount = 1.05

    @classmethod
    def set_raise_amount(cls, amount):
        # cls acts just like 'Employee' here
        cls.raise_amount = amount

# Changes the raise amount for the entire class and all instances
Employee.set_raise_amount(1.07) 
```

### Use Case 2: Alternative Constructors (Factory Methods)

This is arguably the most powerful and common use of `@classmethod` in Python. 

Imagine you are receiving employee data from various sources: sometimes it's a comma-separated string, other times it's a JSON-like dictionary. Because Python doesn't support method overloading (you can't have multiple `__init__` methods in a single class), we use class methods to build "factories" that construct our objects in different ways.

> [!example] Factory Methods
> Look out for methods named `from_something`. This is a standard Python convention used extensively in standard libraries (like `datetime.date.fromtimestamp`).

```python
class Employee:
    def __init__(self, first, last, salary):
        self.first = first
        self.last = last
        self.salary = salary

    @classmethod
    def from_string(cls, emp_str):
        first, last, salary = emp_str.split('-')
        # cls(...) is exactly the same as calling Employee(...)
        # It creates and returns a new instance of the class
        return cls(first, last, int(salary))

    @classmethod
    def from_dict(cls, emp_dict):
        return cls(emp_dict['first'], emp_dict['last'], emp_dict['salary'])

# Usage: Constructing objects from different data formats
emp_str_1 = "John-Doe-70000"
emp1 = Employee.from_string(emp_str_1)

emp_dict_1 = {"first": "Jane", "last": "Smith", "salary": 80000}
emp2 = Employee.from_dict(emp_dict_1)
```

---

## Static Methods (`@staticmethod`)

Finally, we have static methods. Static methods are essentially regular Python functions that have been placed inside a class's namespace because they logically belong there.

> [!abstract] Static Methods Overview
> - **Decorator**: `@staticmethod`
> - **Parameters**: Neither `self` nor `cls` is passed implicitly.
> - **Purpose**: Utility functions that are related to the class conceptually, but don't need access to instance or class data to do their job.

Because they don't take `self` or `cls`, static methods cannot modify object state or class state. They are entirely self-contained.

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

    @staticmethod
    def is_workday(day):
        # Python's weekday(): 0 = Monday, 5 = Saturday, 6 = Sunday
        if day.weekday() == 5 or day.weekday() == 6:
            return False
        return True

import datetime
my_date = datetime.date(2023, 10, 15) # This is a Sunday

# We can call it directly on the class
print(Employee.is_workday(my_date))   # Output: False
```

> [!tip] How to spot a static method
> If you are writing a method inside a class, and you notice that you are *never* using `self` and *never* using `cls` inside the method body, it is a prime candidate to be decorated with `@staticmethod`.

---

## When to Use Which: A Quick Reference

Understanding when to use each type of method is a critical milestone in mastering OOP in Python. Keep this cheat sheet handy:

> [!note] Method Type Cheat Sheet
> 1. **Instance Methods**: 
>    - **Implicit Parameter**: `self`
>    - **Use when**: You need to access or modify data specific to an individual object (e.g., updating a specific employee's salary). *This is your default go-to.*
> 2. **Class Methods**: 
>    - **Implicit Parameter**: `cls`
>    - **Use when**: You need to access or modify data shared across all objects (class attributes), OR when you are writing a factory method to create instances from different data formats.
> 3. **Static Methods**: 
>    - **Implicit Parameter**: None
>    - **Use when**: You have a utility function that makes logical sense to bundle with the class, but doesn't actually depend on any instance or class data.

By organizing your code with these distinct method types, you communicate your intent clearly to other developers. An `@classmethod` signals "I am changing the class or building a new object," while a `@staticmethod` signals "I am just a helper function living in this neighborhood."
