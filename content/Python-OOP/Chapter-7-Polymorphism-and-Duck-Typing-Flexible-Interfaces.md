> [!abstract] What's in this Chapter?
> In this chapter, we'll demystify one of the biggest words in Object-Oriented Programming: **Polymorphism**. We will explore Python's famous "Duck Typing" philosophy, learn the difference between "Easier to Ask for Forgiveness than Permission" (EAFP) and "Look Before You Leap" (LBYL), and finally, discover how to make our custom objects act like built-in Python data types using "Dunder" methods and Operator Overloading.

## 1. Demystifying Polymorphism

If you've ever felt intimidated by the jargon of OOP, **polymorphism** is usually high on the list of scary-sounding words. But let's break it down: it comes from the Greek words *poly* (many) and *morphism* (forms). 

In Object-Oriented Programming, polymorphism simply means the ability of different objects to respond to the same method call or operation in their own unique way.

> [!info] A Secret You Already Know
> Here is a secret: if you have intermediate Python skills, you've been using polymorphism since day one! 
> - The built-in `len()` function is polymorphic: it counts the characters in strings, the items in lists, keys in dictionaries, and elements in sets.
> - The `+` operator is polymorphic: it mathematically adds integers, concatenates strings side-by-side, and merges lists together.

In Python, polymorphism allows us to write generic, flexible code that can operate on multiple types of objects, as long as those objects implement the expected interface (methods or properties). This drastically reduces the tight coupling in our code.

### Polymorphism with Class Hierarchies

While Python doesn't *force* you to use inheritance to achieve polymorphism, it is a very common and robust pattern. 

> [!example] A Polymorphic Function
> Notice how `make_it_speak` below doesn't care whether `entity` is a `Dog`, a `Cat`, or even a `Robot`. It only cares that the object has a `speak()` method!

```python
class Animal:
    def speak(self):
        raise NotImplementedError("Subclass must implement abstract method")

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

class Robot:
    # Notice: Doesn't inherit from Animal!
    def speak(self):
        return "Beep boop!"

# Polymorphic function
def make_it_speak(entity):
    print(entity.speak())

entities = [Dog(), Cat(), Robot()]
for entity in entities:
    make_it_speak(entity) 

# Output:
# Woof!
# Meow!
# Beep boop!
```

Wait, `Robot` doesn't inherit from `Animal`, yet it still works perfectly? Welcome to the wonderful world of **Duck Typing**.

---

## 2. Duck Typing

Because Python is a dynamically typed language, its underlying philosophy strongly embraces **Duck Typing**. 

> [!tip] The Duck Test
> *"If it walks like a duck, and quacks like a duck, then it must be a duck."*

In Python, we rarely check if an object is an exact instance of a specific class using `isinstance()`. Instead, we just assume the object has the necessary methods and attributes. We care exclusively about an object's *behavior*, not its *ancestry*.

### Duck Typing in Action: File-like Objects

A classic Pythonic example is file handling. Many functions in the Python standard library require a "file-like object." They don't rigorously check if you passed an actual `io.TextIOWrapper` (a real file). They just assume you passed something with a `.write()` or `.read()` method.

```python
class StringWriter:
    def __init__(self):
        self.content = []
    
    def write(self, text):
        self.content.append(text)
        
    def get_value(self):
        return "".join(self.content)

def write_greeting(file_like_object):
    # We just assume this object has a 'write' method!
    file_like_object.write("Hello, World!")

# Works with a real file
with open("test.txt", "w") as f:
    write_greeting(f)

# Works with our custom object because it quacks (has a 'write' method)
writer = StringWriter()
write_greeting(writer)
print(writer.get_value())  # Output: Hello, World!
```

### EAFP vs. LBYL

Duck typing goes hand-in-hand with one of Python's most defining idioms: **EAFP** ("Easier to Ask for Forgiveness than Permission"). This is often contrasted with the more traditional **LBYL** ("Look Before You Leap") approach seen in other languages.

> [!note] EAFP over LBYL
> Rather than checking if an object has a method before calling it (LBYL), Pythonistas prefer to just call the method and catch the resulting exception if it fails (EAFP).

```python
# LBYL: Look Before You Leap (Not very Pythonic)
def process_data(data):
    if hasattr(data, 'process'):
        data.process()
    else:
        print("Data cannot be processed.")

# EAFP: Easier to Ask for Forgiveness than Permission (Pythonic)
def process_data_eafp(data):
    try:
        data.process()
    except AttributeError:
        print("Data cannot be processed.")
```

---

## 3. Magic / Dunder Methods

How does Python know how to use `len()` on a list, but also on a dictionary? It uses "Dunder" methods. 

**Dunder** stands for "Double UNDERscore". These are special predefined methods in Python, like `__init__`. They are hooks that allow your custom classes to integrate seamlessly with Python's built-in functions and syntax. They form the backbone of Python's data model and are the primary way we implement polymorphism for built-in operations.

> [!info] Common Dunder Methods
> - `__str__(self)`: Called by `print()` and `str()`. Should return a human-readable string representation of your object.
> - `__repr__(self)`: Called by `repr()` and interactive console outputs. Should be an unambiguous string representation. Ideally, it should look like valid Python code that could recreate the object.
> - `__len__(self)`: Called by the `len()` function. Must return an integer. Allows you to treat your object like a collection!

> [!example] Creating a Custom Collection
> Let's make a `Team` class that feels native to Python.

```python
class Team:
    def __init__(self, name, members):
        self.name = name
        self.members = members
        
    def __str__(self):
        return f"Team {self.name} with {len(self)} members"
        
    def __repr__(self):
        return f"Team('{self.name}', {self.members})"
        
    def __len__(self):
        return len(self.members)

dev_team = Team("Backend", ["Alice", "Bob", "Charlie"])

# Python automatically maps these built-ins to your dunder methods!
print(dev_team)          # Output: Team Backend with 3 members
print(repr(dev_team))    # Output: Team('Backend', ['Alice', 'Bob', 'Charlie'])
print(len(dev_team))     # Output: 3
```

---

## 4. Operator Overloading

Operator overloading is a highly satisfying application of dunder methods. It allows us to redefine how standard mathematical and comparison operators (`+`, `-`, `*`, `==`, `<`, etc.) behave when interacting with our custom objects.

> [!warning] With Great Power...
> Only overload operators when it intuitively makes sense. Overloading `+` to mean "subtract" might be hilarious to you, but it will be a nightmare for whoever reads your code next!

Common Operator Dunder Methods:
- `__add__(self, other)` $\rightarrow$ `+`
- `__sub__(self, other)` $\rightarrow$ `-`
- `__mul__(self, other)` $\rightarrow$ `*`
- `__eq__(self, other)` $\rightarrow$ `==`
- `__lt__(self, other)` $\rightarrow$ `<` (Less than, incredibly useful for sorting!)
- `__le__(self, other)` $\rightarrow$ `<=`

### Example: A 2D Vector Class

This `Vector2D` class demonstrates both arithmetic operator overloading and comparison overloading.

```python
class Vector2D:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        
    def __str__(self):
        return f"({self.x}, {self.y})"
        
    def __add__(self, other):
        # We must check if 'other' is a Vector2D!
        if not isinstance(other, Vector2D):
            return NotImplemented 
        return Vector2D(self.x + other.x, self.y + other.y)
        
    def __eq__(self, other):
        if not isinstance(other, Vector2D):
            return False
        return self.x == other.x and self.y == other.y
        
    def __lt__(self, other):
        # Compare vectors by magnitude
        if not isinstance(other, Vector2D):
            return NotImplemented
        mag_self = (self.x**2 + self.y**2) ** 0.5
        mag_other = (other.x**2 + other.y**2) ** 0.5
        return mag_self < mag_other

v1 = Vector2D(1, 2)
v2 = Vector2D(3, 4)
v3 = Vector2D(1, 2)

# Overloaded +
print(v1 + v2)  # Output: (4, 6)

# Overloaded ==
print(v1 == v3) # Output: True

# Overloaded < (enables the built-in sort functionality!)
vectors = [v2, v1]
vectors.sort()
for v in vectors:
    print(v)
# Output:
# (1, 2)
# (3, 4)
```

> [!tip] The Importance of `NotImplemented`
> In the `__add__` and `__lt__` methods above, notice how we return `NotImplemented` if `other` is not a `Vector2D`. Returning `NotImplemented` is a signal to Python. It tells the Python interpreter, *"I don't know how to add these two things together, try checking if the OTHER object knows how, or throw a `TypeError`."*

---

## 5. Object Introspection

Because of Python's dynamic nature and duck typing, it's often highly beneficial to be able to inspect objects at runtime to see what methods and attributes they possess. This is called **Object Introspection**.

> [!tip] Built-in Introspection Functions
> - `hasattr(obj, name)`: Checks if an object has an attribute or method (e.g., `hasattr(duck, 'quack')`). This is a cornerstone of safely interacting with unknown objects!
> - `getattr(obj, name, default)`: Retrieves the attribute, optionally returning a default value if it doesn't exist.
> - `isinstance(obj, classinfo)`: Checks if an object is an instance of a specific class or a tuple of classes.
> - `issubclass(cls, classinfo)`: Checks if a class is a subclass of another.
> - `dir(obj)`: Returns a list of all attributes and methods (including dunders) available on the object.

```python
class MysteryBox:
    def open(self):
        return "Surprise!"

box = MysteryBox()

# Introspection in action
if hasattr(box, 'open'):
    action = getattr(box, 'open')
    print(action())  # Output: Surprise!
```

Introspection empowers you to write flexible code that adapts to the objects passed into it, further reinforcing the power of duck typing!

---

> [!abstract] Key Takeaways
> 1. You already use **polymorphism** every time you use `+` or `len()` on different data types.
> 2. OOP in Python is exceptionally flexible. You don't need strict abstract base classes to guarantee an interface; you just trust **Duck Typing**.
> 3. Overloading **dunder methods** is the true "Pythonic" way to make custom objects interact beautifully with standard Python built-ins (like `sorted()`, `len()`, and math operators).
> 4. Returning `NotImplemented` from comparison or math dunders is crucial. It lets Python's internal mechanisms try alternative execution paths if your class doesn't know how to handle the `other` data type.
