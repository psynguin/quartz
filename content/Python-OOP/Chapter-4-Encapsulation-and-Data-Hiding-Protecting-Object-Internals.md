> [!abstract] What You'll Learn
> In this chapter, we're diving into **encapsulation**, one of the fundamental pillars of Object-Oriented Programming (OOP). You'll learn how to bundle data and behavior, hide internal implementations, and write clean, robust code using Python's idiomatic features like the `@property` decorator. 

## The Philosophy of Encapsulation

If you've ever driven a car, you know how to use the steering wheel, the accelerator, and the brakes. You don't need to understand the exact fuel injection ratio or how the transmission shifts gears to get to the grocery store. The car's internal mechanics are **encapsulated**—hidden away from you, the driver, behind a clean, simple interface.

In OOP, encapsulation works exactly the same way. It's the practice of bundling data (attributes) and the methods that operate on that data into a single unit: the class. Furthermore, encapsulation involves restricting direct access to some of the object's internal components. This restriction—often called **data hiding**—prevents accidental interference and misuse.

> [!info] The Pythonic Way: "We Are All Consenting Adults Here"
> If you're coming from languages like Java or C++, you might be used to strict access modifiers (`public`, `private`, `protected`) that are enforced by the compiler. Python handles things differently. Python relies heavily on **conventions** rather than strict enforcement. The community philosophy is often summarized as: *"We are all consenting adults here."* This means developers are trusted not to mess with internal variables if they are marked as such.

---

## 1. Public, Protected, and Private Members

Let's break down how Python indicates the intended visibility of class members.

### Public Members

By default, every attribute and method you define in a Python class is **public**. They are meant to be accessed from anywhere—both inside the class and outside of it. 

> [!example] Public Attributes and Methods
```python
class Employee:
    def __init__(self, name, salary):
        self.name = name      # Public attribute
        self.salary = salary  # Public attribute

    def work(self):           # Public method
        print(f"{self.name} is working.")

emp = Employee("Alice", 50000)
print(emp.name)  # Perfectly fine!
emp.work()       # Accessible as expected
```

Public members form the standard API of your class—the "steering wheel and pedals" you expect others to use.

### Protected Members

When you want to signal that an attribute or method is for internal use only (specifically, within the class and its subclasses), you prefix its name with a single underscore `_`.

> [!warning] It's Just a Convention!
> Adding a single underscore does **not** physically prevent someone from accessing the member. Python won't throw an error if you access it from outside the class. It's purely a gentleman's agreement—a blinking neon sign that says, "Hey, this is an internal detail. Use at your own risk!"

> [!example] Protected Attributes in Action
```python
class BankAccount:
    def __init__(self, account_number, balance):
        self.account_number = account_number # Public
        self._balance = balance              # Protected (by convention)

    def _calculate_interest(self):           # Protected method
        return self._balance * 0.05

    def add_interest(self):
        self._balance += self._calculate_interest()

account = BankAccount("12345", 1000)

# Technically accessible, but a major faux pas in Python:
print(account._balance) 
```

### Private Members and Name Mangling

What if you really, *really* want to hide something? If you prefix a member with a double underscore `__` (and at most one trailing underscore), Python triggers a mechanism called **name mangling**.

Name mangling takes your attribute name and silently prefixes it with `_ClassName`. This makes it significantly harder to access from outside the class.

> [!example] Hiding with Name Mangling
```python
class SecureVault:
    def __init__(self, password):
        self.__password = password  # Private attribute

    def check_password(self, attempt):
        return self.__password == attempt

vault = SecureVault("super_secret")

# Trying to access it directly raises an AttributeError!
# print(vault.__password)  
# AttributeError: 'SecureVault' object has no attribute '__password'

# But it's not totally impenetrable...
print(vault._SecureVault__password)  # Output: super_secret
```

> [!note] Why does name mangling exist?
> *Note for the intermediate developer:* While it feels like a security feature, name mangling was actually designed to prevent **naming collisions** in subclasses. If a subclass happens to define its own `__password` attribute, it won't accidentally overwrite the parent class's `__password` because they will be mangled differently (`_SubClass__password` vs. `_ParentClass__password`).

---

## 2. Getters and Setters: The Traditional Approach

In many older OOP languages, it's considered bad practice to ever make an attribute public. Instead, you keep the attributes private and create `get_attribute()` and `set_attribute()` methods. This allows you to add validation logic before changing a value.

You *can* do this in Python, but it's rarely the right move.

> [!example] Traditional Getters and Setters (Un-Pythonic)
```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius  # Protected attribute

    # Getter
    def get_celsius(self):
        return self._celsius

    # Setter
    def set_celsius(self, value):
        if value < -273.15:
            raise ValueError("Temperature below absolute zero is not possible.")
        self._celsius = value

temp = Temperature(25)
print(temp.get_celsius())  # 25
temp.set_celsius(30)
print(temp.get_celsius())  # 30
```

> [!warning] Drawbacks of this Approach
> - **Clunky Syntax:** Accessing data requires method calls (`temp.get_celsius()`) instead of clean attribute access (`temp.celsius`).
> - **Refactoring Nightmare:** If you start with a public attribute `celsius` and later realize you need validation, converting it to `get_celsius()` means you have to go back and rewrite *every single line of code* in your entire project that interacted with the original attribute!

---

## 3. The `@property` Decorator: Pythonic Encapsulation

Python provides an incredibly elegant solution to the getter/setter dilemma: the `@property` decorator. It lets you define methods that can be accessed *as if they were simple attributes*.

This gives you the best of both worlds: the clean, readable syntax of attribute access, coupled with the under-the-hood control of getter and setter methods.

### Creating a Read-Only Property (The Getter)

By decorating a method with `@property`, you turn it into a "getter". The name of the method becomes the name of your new property.

> [!example] Using `@property`
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        """The radius property."""
        return self._radius

c = Circle(5)
print(c.radius)  # Look! No parentheses needed!

# c.radius = 10  
# AttributeError: can't set attribute (because there's no setter yet)
```

### Adding a Setter and Deleter

To allow modification of your new property, you define another method with the *exact same name*, but decorate it with `@<property_name>.setter`. You can even define what happens when someone uses the `del` keyword by using `@<property_name>.deleter`.

> [!example] Full Property Lifecycle
```python
class Employee:
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name

    @property
    def email(self):
        # This is a dynamic property, calculated on the fly!
        return f"{self.first_name.lower()}.{self.last_name.lower()}@company.com"

    @property
    def fullname(self):
        return f"{self.first_name} {self.last_name}"

    @fullname.setter
    def fullname(self, name):
        first, last = name.split(' ')
        self.first_name = first
        self.last_name = last

    @fullname.deleter
    def fullname(self):
        print("Deleting Name!")
        self.first_name = None
        self.last_name = None

emp = Employee("John", "Doe")
print(emp.fullname) # Output: John Doe
print(emp.email)    # Output: john.doe@company.com

# Using the setter seamlessly
emp.fullname = "Jane Smith"
print(emp.first_name) # Output: Jane
print(emp.email)      # Output: jane.smith@company.com

# Using the deleter
del emp.fullname
```

### The True Power of `@property`: Backward Compatibility

Let's revisit our `Temperature` class. Imagine you originally shipped version 1.0 of your code with a simple public attribute `self.celsius`. Months later, you realize you urgently need to prevent users from setting temperatures below absolute zero.

With `@property`, you can add this validation without breaking *any* existing code that uses `temp.celsius`.

> [!tip] Refactoring with `@property`
```python
class Temperature:
    def __init__(self, celsius):
        # We can use our property setter right inside __init__!
        self.celsius = celsius 

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Temperature below absolute zero is not possible.")
        self._celsius = value

temp = Temperature(25)
print(temp.celsius)  # 25

temp.celsius = 30    # Implicitly calls the setter
print(temp.celsius)  # 30

# temp.celsius = -300 
# Boom! Raises ValueError automatically.
```

> [!abstract] Chapter Summary
> - **Encapsulation** bundles data and methods while hiding internal state.
> - Python relies on the **"consenting adults"** convention. We use `_` for protected and `__` for private members (which triggers name mangling to prevent subclass collisions).
> - Avoid writing traditional Java-style getters and setters (`get_val()`, `set_val()`).
> - Use the **`@property`** decorator. It allows you to start with simple public attributes, and later seamlessly add logic, validation, or dynamic computation without breaking the public interface of your class.
